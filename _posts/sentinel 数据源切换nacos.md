---
title: sentinel 数据源切换nacos
date: 2020-06-08 23:56:36
tags: alibaba,sentinel

---

1. 后端修改

    新增nacos的Provider 和 Publisher 

   ~~~java
   /**
    * @author Eric Zhao
    * @since 1.4.0
    */
   @Component("flowRuleNacosProvider")
   public class FlowRuleNacosProvider implements DynamicRuleProvider<List<FlowRuleEntity>> {
   
       @Autowired
       private ConfigService configService;
       @Autowired
       private Converter<String, List<FlowRuleEntity>> converter;
   
       @Override
       public List<FlowRuleEntity> getRules(String appName) throws Exception {
           String rules = configService.getConfig(appName + NacosConfigUtil.FLOW_DATA_ID_POSTFIX,
               NacosConfigUtil.GROUP_ID, 3000);
           if (StringUtil.isEmpty(rules)) {
               return new ArrayList<>();
           }
           return converter.convert(rules);
       }
   }
   
   ~~~

   ~~~java
   /**
    * @author Eric Zhao
    * @since 1.4.0
    */
   @Component("flowRuleNacosPublisher")
   public class FlowRuleNacosPublisher implements DynamicRulePublisher<List<FlowRuleEntity>> {
   
       @Autowired
       private ConfigService configService;
       @Autowired
       private Converter<List<FlowRuleEntity>, String> converter;
   
       @Override
       public void publish(String app, List<FlowRuleEntity> rules) throws Exception {
           AssertUtil.notEmpty(app, "app name cannot be empty");
           if (rules == null) {
               return;
           }
           configService.publishConfig(app + NacosConfigUtil.FLOW_DATA_ID_POSTFIX,
               NacosConfigUtil.GROUP_ID, converter.convert(rules));
       }
   }
   ~~~

   ~~~java
   public class NacosConfig {
   
       @Value("${nacos.configService.server}")
       private String nacosAddr;
   
       @Bean
       public Converter<List<FlowRuleEntity>, String> flowRuleEntityEncoder() {
           return JSON::toJSONString;
       }
   
       @Bean
       public Converter<String, List<FlowRuleEntity>> flowRuleEntityDecoder() {
           return s -> JSON.parseArray(s, FlowRuleEntity.class);
       }
   
       @Bean
       public ConfigService nacosConfigService() throws Exception {
           Properties properties = new Properties();
           // tenant 为 namespace 对应的标识
           // namepace 为 命名空间的名称
           properties.setProperty(PropertyKeyConst.NAMESPACE, "77fdd731-c6d6-4e49-8820-cfb372f2b547");
           properties.setProperty(PropertyKeyConst.SERVER_ADDR, nacosAddr);
           return ConfigFactory.createConfigService(properties);
   //         ConfigFactory.createConfigService("localhost");
       }
   }
   ~~~

   > 这里的namespace 为namespace的id

   ~~~java
   /**
    * @author Eric Zhao
    * @since 1.4.0
    */
   public final class NacosConfigUtil {
   
       public static final String GROUP_ID = "SENTINEL_GROUP";
       
       public static final String FLOW_DATA_ID_POSTFIX = "-flow-rules";
       public static final String PARAM_FLOW_DATA_ID_POSTFIX = "-param-rules";
       public static final String CLUSTER_MAP_DATA_ID_POSTFIX = "-cluster-map";
   
       /**
        * cc for `cluster-client`
        */
       public static final String CLIENT_CONFIG_DATA_ID_POSTFIX = "-cc-config";
       /**
        * cs for `cluster-server`
        */
       public static final String SERVER_TRANSPORT_CONFIG_DATA_ID_POSTFIX = "-cs-transport-config";
       public static final String SERVER_FLOW_CONFIG_DATA_ID_POSTFIX = "-cs-flow-config";
       public static final String SERVER_NAMESPACE_SET_DATA_ID_POSTFIX = "-cs-namespace-set";
   
       private NacosConfigUtil() {}
   }
   
   ~~~

   修改controller `FlowControllerV2` 类

   ~~~java
       @Autowired
   -    @Qualifier("flowRuleDefaultProvider")
   +    @Qualifier("flowRuleNacosProvider")
       private DynamicRuleProvider<List<FlowRuleEntity>> ruleProvider;
       @Autowired
   -    @Qualifier("flowRuleDefaultPublisher")
   +    @Qualifier("flowRuleNacosPublisher")
       private DynamicRulePublisher<List<FlowRuleEntity>> rulePublisher;
   ~~~

   

2. 前端修改

   修改`src/main/webapp/resources/app/scripts/directives/sidebar/sidebar.html` 

   ~~~html
   56    	<li ui-sref-active="active" ng-if="!entry.isGateway">
   58   -     <a ui-sref="dashboard.flowV1({app: entry.app})">
   59   +     <a ui-sref="dashboard.flow({app: entry.app})">
   60         <i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;流控规则</a>
   61  	  </li>
   ~~~

   命令重新编译前端

   `gulp build`

3. 编译dashboard 

   `mvn install `

