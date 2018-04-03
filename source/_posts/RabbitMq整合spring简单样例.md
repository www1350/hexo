---
title: RabbitMq整合spring简单样例
date: 2016-07-04 22:06:07
tags: [RabbitMq,spring]
categories: 消息队列
---

**生产者**
spring 配置如下：

``` xml
<rabbit:connection-factory id="connectionFactory" host="localhost" port="5672" publisher-confirms="true" virtual-host="/" username="absurd" password="absurd" />
<!--下面只有当声明了exchange和队列才可以使用->
<!-- <rabbit:queue id="queue" durable="true" auto-delete="false" exclusive="false" name="queue"/>
<rabbit:queue id="queue2" durable="true" auto-delete="false" exclusive="false" name="queue2"/>
        将队列绑定到交换路由同时与key绑定
    <rabbit:fanout-exchange name="absurd_exchange" durable="true" auto-delete="false" id="absurd_exchange">
        <rabbit:bindings>
            <rabbit:binding queue="queue" />
            <rabbit:binding queue="queue2" />
        </rabbit:bindings>
    </rabbit:fanout-exchange> 
 <rabbit:template id="rabbitTemplate" exchange="absurd_exchange" connection-factory="connectionFactory"/>  -->
<rabbit:template id="rabbitTemplate"  connection-factory="connectionFactory"/> 
```

service

``` java
@Service
public class ProducerServiceImpl implements ProducerService{

    @Autowired private RabbitTemplate rabbitTemplate;
    public void sendMessage(String msg, String routingKey,String exchange) {
        System.err.println("err"+msg+routingKey+exchange);
        RabbitAdmin admin = new RabbitAdmin(this.rabbitTemplate.getConnectionFactory());
        admin.declareExchange(new FanoutExchange(exchange,true,false));  
        admin.declareQueue(new Queue(routingKey,true,false,false) );
        admin.declareBinding(new Binding(routingKey, DestinationType.QUEUE, exchange, routingKey, null));//如果声明了队列、exchange、绑定后就无需使用RabbitAdmin 
        rabbitTemplate.convertAndSend(exchange,routingKey,msg);
        rabbitTemplate.convertAndSend(routingKey,msg);
    }

}
```

controller

``` java
    @RequestMapping(value="/publish/{msg}/{queue}/{exchange}",method=RequestMethod.GET)
    public ModelAndView publish(@PathVariable(value="msg")String msg,@PathVariable(value="queue")String queue,@PathVariable(value="exchange")String exchange){
        ModelAndView mv = new ModelAndView();
        producerService.sendMessage(msg, queue,exchange);
        System.out.println(msg);
        mv.setViewName("a");
        mv.addObject("msg", msg);
        return mv;

    }
```

**消费者**
spring配置

``` xml
    <!-- 连接工厂 -->
    <rabbit:connection-factory id="connectionFactory" host="localhost" publisher-confirms="true" virtual-host="/" username="absurd" password="absurd" />
    <!-- 监听器 -->
    <rabbit:listener-container connection-factory="connectionFactory" acknowledge="auto" task-executor="taskExecutor">
        <!-- queues是队列名称，可填多个，用逗号隔开， method是ref指定的Bean调用Invoke方法执行的方法名称 -->
        <rabbit:listener queues="queue" method="onMessage" ref="redQueueListener" />
    </rabbit:listener-container>
    <!-- 队列声明 -->
    <rabbit:queue name="queue" durable="true" />

       <!-- 配置线程池 -->  
<bean id ="taskExecutor"  class ="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor" >  
    <!-- 线程池维护线程的最少数量 -->  
<property name ="corePoolSize" value ="5" />  
    <!-- 线程池维护线程所允许的空闲时间 -->  
<property name ="keepAliveSeconds" value ="30000" />  
    <!-- 线程池维护线程的最大数量 -->  
<property name ="maxPoolSize" value ="1000" />  
    <!-- 线程池所使用的缓冲队列 -->  
<property name ="queueCapacity" value ="200" />  
    </bean>  
    <!-- 红色监听处理器 -->
    <bean id="redQueueListener" class="com.absurd.rabbitmqcustomer.RedQueueListener"  />
```

监听方法

``` java
public class RedQueueListener {
    private static Logger log = LoggerFactory.getLogger(RedQueueListener.class);
    /**
     * 处理消息
     * @param message
     * void
     */
    public void onMessage(String message) {
        log.info("RedQueueListener Receved:"  + message);
    }
}
```

rabbitmq引入：

```
        <dependency>
            <groupId>org.springframework.amqp</groupId>
            <artifactId>spring-rabbit</artifactId>
            <version>1.5.6.RELEASE</version>
        </dependency>
```

效果：
访问http://localhost:8080/rabbitmqproducer/hello/publish/bfdbdfbdfg/queue/absurd_exchange5
消费者就能监听到消息