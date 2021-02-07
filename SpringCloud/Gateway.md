# Gateway

## 整合Nacos

```yaml
server:
  port: 8087
spring:
  application:
    name: nacos_gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    gateway:
      discovery:
        locator:
          enabled: true  #表明gateway开启服务注册和发现的功能，并且spring cloud gateway自动根据服务发现为每一个服务创建了一个router，这个router将以服务名开头的请求路径转发到对应的服务
          lower-case-service-id: true  #是将请求路径上的服务名配置为小写（因为服务注册的时候，向注册中心注册时将服务名转成大写的了
      routes:
          -id: apiuser
          #
          uri: lb://nacos-consumer-user
          predicates:
          # http://localhost:6601/user/user/users/2, 必须加上StripPrefix=1，否则访问服务时会带上user
          - Path=/user/** # 转发该路径
           #以下是配置例子
            # - id: 163                     #网关路由到网易官网
            #  uri: http://www.163.com/
            #  predicates:
                - Path=/163/**
        #      - id: ORDER-SERVICE           #网关路由到订单服务order-service
        #        uri: lb://ORDER-SERVICE
        #        predicates:
        #          - Path=/ORDER-SERVICE/**
        #      - id: USER-SERVICE            #网关路由到用户服务user-service
        #        uri: lb://USER-SERVICE
        #        predicates:
        #          - Pach=/USER-SERVICE/**
```



## Gateway 全局统一前缀

重写定义GatewayDiscoveryClientAutoConfiguration自动化配置类，改写原先的Predicate和Filter，添加统一的请求前缀“/api/v1”。

```java
@Slf4j
@Configuration
public class GatewayDiscoveryClientConfig<main> extends GatewayDiscoveryClientAutoConfiguration {

    @Value("${spring.cloud.gateway.api-prefix:/api/v1}")
    private  String prefix;

    @Bean
    @Override
    public DiscoveryLocatorProperties discoveryLocatorProperties() {
        DiscoveryLocatorProperties properties = new DiscoveryLocatorProperties();
        properties.setPredicates(myInitPredicates());
        properties.setFilters(myInitFilters());
        return properties;
    }

    public  List<PredicateDefinition> myInitPredicates() {
        ArrayList<PredicateDefinition> definitions = new ArrayList();
        PredicateDefinition predicate = new PredicateDefinition();
        //定义路由路径的匹配规则
        predicate.setName(NameUtils.normalizeRoutePredicateName(PathRoutePredicateFactory.class));
        String pattern ="'"+prefix+"/'+serviceId+'/**'";
        predicate.addArg("pattern", pattern);
        definitions.add(predicate);
        return definitions;
    }
    
    public  List<FilterDefinition> myInitFilters() {
        ArrayList<FilterDefinition> definitions = new ArrayList();
        FilterDefinition filter = new FilterDefinition();
        //重新请求路径
        filter.setName(NameUtils.normalizeFilterFactoryName(RewritePathGatewayFilterFactory.class));
        String regex = "'"+prefix+"/' + serviceId + '/(?<remaining>.*)'";
        String replacement = "'/${remaining}'";
        filter.addArg("regexp", regex);
        filter.addArg("replacement", replacement);
        definitions.add(filter);
        return definitions;
    }

}

```

启动类代码，排除gateway 自动化配置类

```java
@SpringBootApplication(exclude = {
        GatewayDiscoveryClientAutoConfiguration.class
})
@EnableDiscoveryClient
@Slf4j
public class WxswjGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(WxswjGatewayApplication.class, args);
        log.info("网关服务启动成功！");
    }
}
```

自定义自动化配置

```java
/**
 * @program: wxswj
 * @description: 自定义网关注册中心配置
 * @create: 2020-10-14 20:11
 **/
@Slf4j
@Configuration
public class GatewayDiscoveryClientConfig<main> extends GatewayDiscoveryClientAutoConfiguration {

    @Value("${spring.cloud.gateway.api-prefix:/api/v1}")
    private  String prefix;

    @Bean
    @Override
    public DiscoveryLocatorProperties discoveryLocatorProperties() {
        DiscoveryLocatorProperties properties = new DiscoveryLocatorProperties();
        properties.setPredicates(myInitPredicates());
        properties.setFilters(myInitFilters());
        return properties;
    }

    public  List<PredicateDefinition> myInitPredicates() {
        ArrayList<PredicateDefinition> definitions = new ArrayList();
        PredicateDefinition predicate = new PredicateDefinition();
        //定义路由路径的匹配规则
        predicate.setName(NameUtils.normalizeRoutePredicateName(PathRoutePredicateFactory.class));
        String pattern ="'"+prefix+"/'+serviceId+'/**'";
        predicate.addArg("pattern", pattern);
        definitions.add(predicate);
        return definitions;
    }

    public  List<FilterDefinition> myInitFilters() {
        ArrayList<FilterDefinition> definitions = new ArrayList();
        FilterDefinition filter = new FilterDefinition();
        //重新请求路径
        filter.setName(NameUtils.normalizeFilterFactoryName(RewritePathGatewayFilterFactory.class));
        String regex = "'"+prefix+"/' + serviceId + '/(?<remaining>.*)'";
        String replacement = "'/${remaining}'";
        filter.addArg("regexp", regex);
        filter.addArg("replacement", replacement);
        definitions.add(filter);
        return definitions;
    }
}


```



## Gateway基于nacos 配置实现动态路由

gateway没有适配nacos，一旦新增路由，gateway无法及时获取到，需要自定义监听器。

工程启动的时候拉取配置中心的配置，初始化路由信息。启动后监听配置中心的配置变化，有变化后则更新工程路由信息。

```java
@Service
@Slf4j
public class DynamicRouteByNacos implements CommandLineRunner {
    @Autowired
    private DynamicRouteUtil dynamicRouteService;

    @Autowired
    private NacosGatewayPropertiesConfig nacosGatewayProperties;

    /**
     * 监听Nacos Server下发的动态路由配置。初始化的时候先加载一次配置
     */
    public void dynamicRouteByNacosListener (){
        try {
            Properties properties = new Properties();
            properties.put(PropertyKeyConst.NAMESPACE, nacosGatewayProperties.getNameSpace());
            properties.put(PropertyKeyConst.SERVER_ADDR, nacosGatewayProperties.getAddress());
            ConfigService configService= NacosFactory.createConfigService(properties);
            String content = configService.getConfig(nacosGatewayProperties.getDataId(), nacosGatewayProperties.getGroupId(), nacosGatewayProperties.getTimeout());

            //初始话的时候也初始化路由
            if(StringUtils.isNotEmpty(content)){
                List<RouteDefinition> list = JSONObject.parseArray(content, RouteDefinition.class);
                list.forEach(definition->{
                    dynamicRouteService.update(definition);
                });
            }

            configService.addListener(nacosGatewayProperties.getDataId(), nacosGatewayProperties.getGroupId(), new Listener()  {
                @Override
                public void receiveConfigInfo(String configInfo) {
                    List<RouteDefinition> list = JSONObject.parseArray(configInfo, RouteDefinition.class);
                    log.warn("---------------------"+ JSON.toJSONString(list));
                    list.forEach(definition->{
                        dynamicRouteService.update(definition);
                    });
                }
                @Override
                public Executor getExecutor() {
                    return null;
                }
            });
        } catch (NacosException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run(String... args) throws Exception {
        dynamicRouteByNacosListener();

    }
}

```

### 网关中nacos的配置以及配置类

```java
/**
 * Created by Miracle ZhangYunHai on 2020/9/10 14:33
 */
@ConfigurationProperties(prefix="spring.cloud.nacos.route.config", ignoreUnknownFields = true)
@Configuration
@Data
public class NacosGatewayPropertiesConfig {

    /**
     * nacos的访问地址
     */
    private String address;

    /**
     * 配置的dataId
     */
    private String dataId;

    /**
     * 配置的groupId
     */
    private String groupId;

    /**
     * 连接超时时间
     */
    private Long timeout;

    /**
     * 命名空间
     */
    private String nameSpace;

}

```

### 更新路由方法

```java
/**
 * 
 *实现 ApplicationEventPublisherAware push RefreshRoutesEvent 来跟新路由
 */
@Service
@Slf4j
public class DynamicRouteUtil implements ApplicationEventPublisherAware {

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;

    private ApplicationEventPublisher publisher;


    /**
     * 增加路由
     * @param definition
     * @return
     */
    public String add(RouteDefinition definition) {
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        this.publisher.publishEvent(new RefreshRoutesEvent(this));
        return "success";
    }


    /**
     * 更新路由
     * @param definition
     * @return
     */
    public String update(RouteDefinition definition) {
        try {
            this.routeDefinitionWriter.delete(Mono.just(definition.getId()));
        } catch (Exception e) {
            return "update fail,not find route  routeId: "+definition.getId();
        }
        try {
            routeDefinitionWriter.save(Mono.just(definition)).subscribe();
            this.publisher.publishEvent(new RefreshRoutesEvent(this));
            return "success";
        } catch (Exception e) {
            return "update route  fail";
        }


    }
    /**
     * 删除路由
     * @param id
     * @return
     */
    public String delete(String id) {
        try {
            this.routeDefinitionWriter.delete(Mono.just(id));
            return "delete success";
        } catch (Exception e) {
            e.printStackTrace();
            return "delete fail";
        }

    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.publisher = applicationEventPublisher;
    }
}
```