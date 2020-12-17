---
title: 关于springCloud gateway
date: 2020-12-17 15:00:00 +0800
tags: [springcloud]
categories: [springcloud, gateway]
pin: true
---



### springcloud gateway 初体验

依赖：

```
   <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.5.RELEASE</version>
    </parent>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Finchley.SR1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
	<dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-gateway</artifactId>
	</dependency>
```

体验版：

```
/**
* 
*/
@SpringBootApplication
@RestController
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
    /**
    * 初始化路由器
    * 这里使用了一个RouteLocatorBuilder的Bean去创建路由，除了创建路由RouteLocatorBuilder可以让你
    * 添加各种predicates和filters，predicates断言的意思（JDK8版本开始引入），顾名思义就是根据
    * 具体的请求的规则，由具体的route去处理，filters是各种过滤器，用来对请求做各种判断和修改
    * <p>
    * 当请求路径为get时，会将请求转发到http://httpbin.org:80，并且header会添加 Hello：World
    *
    */
    @Bean
    public RouteLocator myRoutes(RouteLocatorBuilder builder) {
       return builder.routes()
        .route(p -> p
        	//请求路径适配
            .path("/get")
            //添加过滤器 header中追加  Hello：World
            .filters(f -> f.addRequestHeader("Hello", "World"))
            // 路由转发目标地址
            .uri("http://httpbin.org:80"))
        .build();
    }
    
}
```





### GatewayAutoConfiguration

GatewayAutoConfiguration 已经把 InMemoryRouteDefinitionRepository 注册成bean了，可以进行动态路由配置

```java
@Configuration
@ConditionalOnProperty(name = "spring.cloud.gateway.enabled", matchIfMissing = true)
@EnableConfigurationProperties
@AutoConfigureBefore({ HttpHandlerAutoConfiguration.class,
		WebFluxAutoConfiguration.class })
@AutoConfigureAfter({ GatewayLoadBalancerClientAutoConfiguration.class,
		GatewayClassPathWarningAutoConfiguration.class })
@ConditionalOnClass(DispatcherHandler.class)
public class GatewayAutoConfiguration {

	@Bean
	public StringToZonedDateTimeConverter stringToZonedDateTimeConverter() {
		return new StringToZonedDateTimeConverter();
	}

	@Bean
	public RouteLocatorBuilder routeLocatorBuilder(
			ConfigurableApplicationContext context) {
		return new RouteLocatorBuilder(context);
	}

	@Bean
	@ConditionalOnMissingBean
	public PropertiesRouteDefinitionLocator propertiesRouteDefinitionLocator(
			GatewayProperties properties) {
		return new PropertiesRouteDefinitionLocator(properties);
	}

	@Bean
	@ConditionalOnMissingBean(RouteDefinitionRepository.class)
	public InMemoryRouteDefinitionRepository inMemoryRouteDefinitionRepository() {
		return new InMemoryRouteDefinitionRepository();
	}

	@Bean
	@Primary
	public RouteDefinitionLocator routeDefinitionLocator(
			List<RouteDefinitionLocator> routeDefinitionLocators) {
		return new CompositeRouteDefinitionLocator(
				Flux.fromIterable(routeDefinitionLocators));
	}

	@Bean
	public RouteLocator routeDefinitionRouteLocator(GatewayProperties properties,
			List<GatewayFilterFactory> GatewayFilters,
			List<RoutePredicateFactory> predicates,
			RouteDefinitionLocator routeDefinitionLocator,
			@Qualifier("webFluxConversionService") ConversionService conversionService) {
		return new RouteDefinitionRouteLocator(routeDefinitionLocator, predicates,
				GatewayFilters, properties, conversionService);
	}
	
	……
}
```



####  InMemoryRouteDefinitionRepository

其提供了save和delete方法，这样就可以修改当前路由配置信息，路由配置信息修改完毕之后，再用spring的事件触发 <B>RefreshRoutesEvent</B> 事件来刷新路由就行了

```java

/**
 * @author Spencer Gibb
 */
public class InMemoryRouteDefinitionRepository implements RouteDefinitionRepository {

	private final Map<String, RouteDefinition> routes = synchronizedMap(
			new LinkedHashMap<String, RouteDefinition>());

     /**
     * 来自 RouteDefinitionWriter
     * <p>
     * 注：将路由信息保存到路由配置map中
     *
     * @param route
     */
	@Override
	public Mono<Void> save(Mono<RouteDefinition> route) {
		return route.flatMap(r -> {
			routes.put(r.getId(), r);
			return Mono.empty();
		});
	}

     /**
     * 来自 RouteDefinitionWriter
     * <p>
     * 注：将路由信息从路由配置map中移除
     *
     * @param routeId
     */
	@Override
	public Mono<Void> delete(Mono<String> routeId) {
		return routeId.flatMap(id -> {
			if (routes.containsKey(id)) {
				routes.remove(id);
				return Mono.empty();
			}
			return Mono.defer(() -> Mono.error(
					new NotFoundException("RouteDefinition not found: " + routeId)));
		});
	}

    
    /**
    * 实现RouteDefinitionLocator的方法
    * 将路由map所有values集合转换成Flux
    *
    * @return Flux路由器
    */
	@Override
	public Flux<RouteDefinition> getRouteDefinitions() {
		return Flux.fromIterable(routes.values());
	}

}
```



下次请求的时候就可以拿最新的路由配置了

顺序是：

1. RoutePredicateHandlerMapping的lookupRoute方法，由于路由刷新事件把路由缓存清了，所以重新获取
2. CompositeRouteLocator的getRoutes()方法遍历所有RouteLocator实现类的getRoutes方法
3. RouteDefinitionRouteLocator的getRoutes方法里重新获取了所有的路由定义，也就把我们刚刚用的事件添加的路由也获取了





#### RouteDefinitionRepository

```java
public interface RouteDefinitionRepository
		extends RouteDefinitionLocator, RouteDefinitionWriter {

}
```



#### RefreshRoutesEvent 事件刷新路由

```
/**
 * @author Spencer Gibb
 */
public class RefreshRoutesEvent extends ApplicationEvent {

	/**
	 * Create a new ApplicationEvent.
	 * @param source the object on which the event initially occurred (never {@code null})
	 */
	public RefreshRoutesEvent(Object source) {
		super(source);
	}

}
```



```

/**
 * @author Spencer Gibb
 */
public class CachingRouteLocator
		implements RouteLocator, ApplicationListener<RefreshRoutesEvent> {

	private final RouteLocator delegate;

	private final Flux<Route> routes;

	private final Map<String, List> cache = new HashMap<>();

	public CachingRouteLocator(RouteLocator delegate) {
		this.delegate = delegate;
		routes = CacheFlux.lookup(cache, "routes", Route.class)
				.onCacheMissResume(() -> this.delegate.getRoutes()
						.sort(AnnotationAwareOrderComparator.INSTANCE));
	}

	@Override
	public Flux<Route> getRoutes() {
		return this.routes;
	}

	/**
	 * Clears the routes cache.
	 * @return routes flux
	 */
	public Flux<Route> refresh() {
		this.cache.clear();
		return this.routes;
	}

	@Override
	public void onApplicationEvent(RefreshRoutesEvent event) {
		refresh();
	}

	@Deprecated
	/* for testing */ void handleRefresh() {
		refresh();
	}

}

```



### 读取requestBody

> 参考：https://www.cnblogs.com/cafebabe-yun/p/9328554.html