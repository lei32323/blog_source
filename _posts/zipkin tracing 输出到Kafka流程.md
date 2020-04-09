---
title: zipkin tracing 输出到Kafka流程
date: 2020-04-09 16:55:36
tags: zipkin

---



# zipkin tracing 输出到Kafka流程

主要3个类

**ByteBoundedQueue** 是一个既有长度限制，又有字节限制的阻塞队列



**BufferNextMessage** Queue的消费者 



**BoundedAsyncReporter** 创建span到队列中





1. **BoundedAsyncReporter** 的 `report` 方法主要把span 存储到队列中 

~~~java
 public void report(S next) {
        if (next == null) throw new NullPointerException("span == null");

        metrics.incrementSpans(1);
        int nextSizeInBytes = encoder.sizeInBytes(next);
        int messageSizeOfNextSpan = sender.messageSizeInBytes(nextSizeInBytes);
        metrics.incrementSpanBytes(nextSizeInBytes);
        if (closed.get() ||
                // don't enqueue something larger than we can drain
                messageSizeOfNextSpan > messageMaxBytes  || // (1)
                !pending.offer(next, nextSizeInBytes)) {  // (2)
            metrics.incrementSpansDropped(1);
        }
    }
~~~

​    (1) 当前span如果大于设置的messageMaxBytes ,则直接记录到metrics中，不增加到队列中

​	(2) 调用队列的 `offer` 函数添加span到队列中

> messageMaxBytes : 设置发往kafka的每条消息大小

2. **ByteBoundedQueue**  `offer` 增加span到队列

~~~java
public boolean offer(S next, int nextSizeInBytes) {
    lock.lock(); //(1)
    try {
      if (count == maxSize) return false ;  //(2)
      if (sizeInBytes + nextSizeInBytes > maxBytes) return false;//(3)

      elements[writePos] = next;
      sizesInBytes[writePos++] = nextSizeInBytes;

      if (writePos == maxSize) writePos = 0; // circle back to the front of the array  //(4)

      count++;
      sizeInBytes += nextSizeInBytes;

      available.signal(); // alert any drainers  //(5)
      return true;
    } finally {
      lock.unlock(); (6)
    }
}
~~~



​	(1) 先加一把锁，保证只有自己在写span。

 （2）队列已有的span个数(count)  和 队列的容量(maxSize)  相同，不添加span

​    (3)  队列已有的span的bytes大小(sizeInBytes)  + 当前span的byte大小(nextSizeInBytes) 大于 最大的byte大小(maxBytes)，不添加span

​    (4)  写完span的下一个下标 和 队列的容量(maxSize)  相同, 下一次要写的位置(writePos) 设置为0 ，从头开始写

  （5）唤醒其他阻塞线程(消费者)

  （6）释放当前锁



3. **BoundedAsyncReporter** 的 `public flush` 为消费者 消费的函数 

   ~~~java
   public final void flush() {
           flush(BufferNextMessage.create(encoder.encoding(), messageMaxBytes, 0)); // (1)
    }
   ~~~

   (1)  创建 BufferNextMessage 消费者 

   > encoder.encoding()：设置吐出到kafka的格式
   >
   > messageMaxBytes : 设置为 每条消息的的byte大小
   >
   > 0 : 等待时间 

   

4. **BoundedAsyncReporter** 的 `flush` 为消费者 消费的函数 (处理消费逻辑的函数)

~~~java
void flush(BufferNextMessage<S> bundler) {

        if (closed.get()) throw new IllegalStateException("closed");

        pending.drainTo(bundler, bundler.remainingNanos());  //(1)

        // record after flushing reduces the amount of gauge events vs on doing this on report
        metrics.updateQueuedSpans(pending.count);
        metrics.updateQueuedBytes(pending.sizeInBytes);

        // loop around if we are running, and the bundle isn't full
        // if we are closed, try to send what's pending
        if (!bundler.isReady() && !closed.get()) return;   //(2)

        // Signal that we are about to send a message of a known size in bytes
        metrics.incrementMessages();
        metrics.incrementMessageBytes(bundler.sizeInBytes());

        // Create the next message. Since we are outside the lock shared with writers, we can encode
        ArrayList<byte[]> nextMessage = new ArrayList<>(bundler.count());
        bundler.drain(new SpanWithSizeConsumer<S>() {
            @Override
            public boolean offer(S next, int nextSizeInBytes) {
                nextMessage.add(encoder.encode(next)); // speculatively add to the pending message
                if (sender.messageSizeInBytes(nextMessage) > messageMaxBytes) {
                    // if we overran the message size, remove the encoded message.
                    nextMessage.remove(nextMessage.size() - 1);
                    return false;
                }
                return true;
            }
        });

        try {
            sender.sendSpans(nextMessage).execute();
        } catch (IOException | RuntimeException | Error t) {}
    }
~~~

​    (1)  从队列 `ByteBoundedQueue` 获取span信息

 `pending.drainTo(bundler, bundler.remainingNanos())`

> bundler.remainingNanos() 参数 
>
> ~~~java
> long remainingNanos() {
>   if (spans.isEmpty()) {
>     deadlineNanoTime = System.nanoTime() + timeoutNanos; //(1)
>   }
>   return Math.max(deadlineNanoTime - System.nanoTime(), 0); //(2)
> }
> ~~~
>
> ( 1 ) 如果spans集合没有数据的话，当前传入的 0 秒 和 当前的时间 相加
>
> ( 2 ) 有数据的话，用设置的等待时间  -  当前时间时间 和 0 取最大值   
>
>  
>
> **ByteBoundedQueue** 的`drainTo`  获取span 
>
> ~~~java
> int drainTo(SpanWithSizeConsumer<S> consumer, long nanosTimeout) {
>     try {
>       // This may be called by multiple threads. If one is holding a lock, another is waiting. We
>       // use lockInterruptibly to ensure the one waiting can be interrupted.
>       lock.lockInterruptibly();
>       try {
>         long nanosLeft = nanosTimeout;
>         while (count == 0) {
>           if (nanosLeft <= 0) return 0;
>           nanosLeft = available.awaitNanos(nanosLeft); // (1)
>         }
>         return doDrain(consumer);  // (2)
>       } finally {
>         lock.unlock();
>       }
>     } catch (InterruptedException e) {
>       return 0;
>     }
>   }
> ~~~
>
> ( 1 ) 阻塞 设置的0秒时间  直接进入( 2  ) 步
>
> ( 2 ) 进入 具体获取span的函数 `doDrain`
>
> ~~~java
>  int doDrain(SpanWithSizeConsumer<S> consumer) {
>     int drainedCount = 0;
>     int drainedSizeInBytes = 0;
>     while (drainedCount < count) {  //(1)
>       S next = elements[readPos];
>       int nextSizeInBytes = sizesInBytes[readPos];
> 
>       if (next == null) break;   //(2)
>       if (consumer.offer(next, nextSizeInBytes)) {  //(3)
>         drainedCount++;
>         drainedSizeInBytes += nextSizeInBytes;
> 
>         elements[readPos] = null;
>         if (++readPos == elements.length) readPos = 0; // circle back to the front of the array  //(4)
>       } else {
>         break;
>       }
>     }
>     count -= drainedCount;
>     sizeInBytes -= drainedSizeInBytes;
>     return drainedCount;
>   }
> ~~~
>
> ( 1 )  循环条件：本次需要获取的span个数 小于 队列里已有的span数量 
>
> ( 2 )  当获取到的span为null， 退出 等待下一次 进入
>
> ( 3 )  调用consumer的`off` 函数，验证是否可以获取此span 
>
> ( 4 )  当前获取的span已经是队列的最后一个了，下次从下标0开始取
>
>  
>
> **BufferNextMessage** 的`off` 函数
>
> ~~~java
> public boolean offer(S next, int nextSizeInBytes) {
>     int x = messageSizeInBytes(nextSizeInBytes);
>     int y = maxBytes;
>     int includingNextVsMaxBytes = (x < y) ? -1 : ((x == y) ? 0 : 1); // Integer.compare, but JRE 6 // (1)
> 
>     if (includingNextVsMaxBytes > 0) {  // (2)
>       bufferFull = true;
>       return false; // can't fit the next message into this buffer
>     }
> 
>     addSpanToBuffer(next, nextSizeInBytes);  // (3)
>     messageSizeInBytes = x;
> 
>     if (includingNextVsMaxBytes == 0) bufferFull = true;  // (4)
>     return true;
>   }
> ~~~
>
> ( 1 ) 用 当前span的byteSize + 当前已有的 byteSizes 的大小  是否和设置的maxBytes(设置的1M) 对比
>
> ( 2 ) 大于 0 说明 大于1M ，设置bufferFull 为true ,说明已经满了，可以发了
>
> ( 3 )  没有的话，那么添加span到  span的集合中  
>
> ( 4 )  等于0的话，说明是最后一个span了，设置bufferFull 为true ,说明已经满了，可以发了

​    (2)  验证span的集合是否 已经满了   ,不满 直接 退出，进入下一次

` if (!bundler.isReady() && !closed.get()) return;`   

> `bundler.isReady()`
>
> ~~~java
>  boolean isReady() {
>     return bufferFull ||  //(1)
>       remainingNanos() <= 0;//(2) 
>   }
> ~~~
>
> ( 1 ) bufferFull   当前span集合已经满了 -> maxBytes == span集合的bytes   
>
> ( 2 ) remainingNanos() 再次调用一次`remainingNanos()` ，获取是否已经过了等待时间



​	 (3)  获取**BufferNextMessage** 的span集合，发往kafka

~~~java
ArrayList<byte[]> nextMessage = new ArrayList<>(bundler.count());
bundler.drain(new SpanWithSizeConsumer<S>() {  //(1)
  @Override
  public boolean offer(S next, int nextSizeInBytes) {
    nextMessage.add(encoder.encode(next)); // speculatively add to the pending message   //(2)
    if (sender.messageSizeInBytes(nextMessage) > messageMaxBytes) {  //(3)
      // if we overran the message size, remove the encoded message.
      nextMessage.remove(nextMessage.size() - 1);//(4)

      return false;
    }
    return true;
  }
});
~~~

   >​      ( 1 ) 调用**BufferNextMessage**的 drain方法，把span集合读取出来
   >
   >​      ( 2 )  添加span到nextMessage集合
   >
   >​      ( 3 )  nextMessage 大于 设置的messageMaxBytes(1M)
   >
   >​      ( 4 )  从nextMessage删除 刚才添加的span
   >
   >

​     (4)  `sender.sendSpans(nextMessage).execute();` 发送kafka





5. zipkin 还提供了一个 守护线程，一直调用flush函数 获取span

~~~java
 public <S> AsyncReporter<S> build(BytesEncoder<S> encoder) {
      if (encoder == null) throw new NullPointerException("encoder == null");

      if (encoder.encoding() != sender.encoding()) {
        throw new IllegalArgumentException(String.format(
            "Encoder doesn't match Sender: %s %s", encoder.encoding(), sender.encoding()));
      }

      final BoundedAsyncReporter<S> result = new BoundedAsyncReporter<>(this, encoder);

      if (messageTimeoutNanos > 0) { // Start a thread that flushes the queue in a loop.
        final BufferNextMessage<S> consumer =
            BufferNextMessage.create(encoder.encoding(), messageMaxBytes, messageTimeoutNanos);
        Thread flushThread = threadFactory.newThread(new Flusher<>(result, consumer));
        flushThread.setName("AsyncReporter{" + sender + "}");
        flushThread.setDaemon(true);
        flushThread.start();
      }
      return result;
    }
  }
~~~

在 **AsyncReporter** build函数中 创建了`flushThread` 线程，持续调用`flush`

~~~java
public void run() {
      try {
        while (!result.closed.get()) {
          result.flush(consumer);
        }
      } catch (RuntimeException | Error e) {
        logger.log(Level.WARNING, "Unexpected error flushing spans", e);
        throw e;
      } finally {
        int count = consumer.count();
        if (count > 0) {
          result.metrics.incrementSpansDropped(count);
          logger.warning("Dropped " + count + " spans due to AsyncReporter.close()");
        }
        result.close.countDown();
      }
    }
~~~



> 1. 当zipkin 在 **BufferNextMessage**的bufferFull 为true (就是span集合的byteSize 等于设置的1M)  会发送消息
>
> 2. 当**BufferNextMessage**的bufferFull不为false,但是过了等待时间，那么也会发消息到kafka
>
> 



配图 

![未命名文件](https://wanglei.club/zipkin1111.png)