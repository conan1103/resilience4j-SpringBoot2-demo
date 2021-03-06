[toc]
# resilience4j-SpringBoot2-demo
    resilience4j version : 0.14.1
    Spring Boot2 version : 2.1.3.RELEASE

## circuitBreaker demo
* Spring AOP<br>
`BackendAOPController > AopBusinessService > AopConnector`
* without Spring AOP<br>
`BackendNoAOPController > NoAopBusinessService > NoAopConnector`
<br><br>demo分为使用Spring aop和不使用aop两种方式，看demo中对应代码即可
### Consume emitted CircuitBreakerEvents
`pull task`
```
@Bean(name = CRConstants.CIRCUITBREAKERAOP)
public CircuitBreaker circuitBreaker() {
    CircuitBreaker circuitBreaker = circuitBreakerFactory.create(CRConstants.CIRCUITBREAKERAOP, 10, 1 * 1000, 10, 10, BusinessException.class);
    circuitBreaker.getEventPublisher().onEvent(eventConsumerRegistry.createEventConsumer(CRConstants.CIRCUITBREAKERAOP, 100));
    return circuitBreaker;
}
  
//CircularEventConsumer 该种方式需要去pull task
//@PostConstruct  //若注解打开，/actuator/circuitbreakerevents看不到事件，该监控作用是看近期未被消费的事件
public void init() {
    scheduled.scheduleAtFixedRate(() -> {
    CircularEventConsumer<CircuitBreakerEvent> ringBuffer = new CircularEventConsumer<>(10);
    circuitBreaker.getEventPublisher().onEvent(ringBuffer);
    List<CircuitBreakerEvent> bufferedEvents = (List<CircuitBreakerEvent>) ringBuffer.getBufferedEvents();
    bufferedEvents.forEach(event -> circuitBreakerEventsProcessor.processEvent(event));
    }, 1000, 1000, TimeUnit.MILLISECONDS); 
}
```

`push task`

```
@PostConstruct
public void init() {
    circuitBreaker = circuitBreakerFactory.create(CRConstants.CIRCUITBREAKERNOAOP, 10, 5 * 1000, 5, 10, BusinessException.class);
    circuitBreaker.getEventPublisher().onEvent(event -> circuitBreakerEventsProcessor.processEvent(event));
}
```
        两种区别在于：pull task需要主动拉取事件并进行消费，push task是CircuitBreaker触发事件时主动push并立刻消费

推荐采用官网yml配置方式(Spring AOP), 配置简单，上手快，维护成本低：
```
resilience4j.circuitbreaker:
    backends:
        backendA:
            registerHealthIndicator: true
            ringBufferSizeInClosedState: 5
            ringBufferSizeInHalfOpenState: 3
            waitInterval: 5000
            failureRateThreshold: 50
            eventConsumerBufferSize: 10
            ignoreExceptions:
                - io.github.robwin.exception.BusinessException
        backendB:
            registerHealthIndicator: true
            ringBufferSizeInClosedState: 10
            ringBufferSizeInHalfOpenState: 5
            waitInterval: 5000
            failureRateThreshold: 50
            eventConsumerBufferSize: 10
            recordFailurePredicate: io.github.robwin.exception.RecordFailurePredicate
```
暂时研究yml配置除了registerHealthIndicator（健康检查），其它配置均可通过circuitBreakerConfig自定义编码配置
### Monitoring
The CircuitBreaker provides an interface to monitor the current metrics.
```
//这段比较简单，未在demo中体现
CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();
// Returns the current failure rate in percentage.
float failureRate = metrics.getFailureRate();
// Returns the current number of buffered calls.
int bufferedCalls = metrics.getNumberOfBufferedCalls();
// Returns the current number of failed calls.
int failedCalls = metrics.getNumberOfFailedCalls();
// Returns the current number of not permitted calls when the CircuitBreaker is open.
int failedCalls = metrics.getNumberOfNotPermittedCalls();
```
### actuator url
```
http://localhost:8080/actuator  查看所有监控url
http://localhost:8080/actuator/circuitbreakers  查询所有熔断器
http://localhost:8080/actuator/circuitbreakerevents 查询最近100（默认）未被消费的熔断事件
http://localhost:8080/actuator/circuitbreakerevents/{name}    根据熔断器名称查询熔断事件
http://localhost:8080/actuator/circuitbreakerevents/{name}/{eventType}   根据熔断器名称及事件类型查询熔断事件
```

# RateLimiter
  web module: RateLimiterController
# Bulkhead
  web module: BulkheadController
