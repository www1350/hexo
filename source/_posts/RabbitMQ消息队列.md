---
title: RabbitMQ消息队列
tags: RabbitMQ
categories: 消息队列
abbrlink: 26d4999d
date: 2016-07-04 21:50:32
---

关于详细介绍：http://blog.csdn.net/column/details/rabbitmq.html
RabbitMQ：基于AMQP协议（Advanced Message Queue Protocol）介绍：http://www.infoq.com/cn/articles/AMQP-RabbitMQ/
ActiveMQ：基于STOMP协议

所需环境：
1.Erlang
2.RabbitMQ
3.rabbit-client.jar  [api](http://www.rabbitmq.com/api-guide.html)

http://www.lxway.com/991402946.htm
Direct Exchange – 处理路由键。需要将一个队列绑定到交换机上，要求该消息与一个特定的路由键完全匹配。这是一个完整的匹配。如果一个队列绑定到该交换机上要求路由键 “dog”，则只有被标记为“dog”的消息才被转发，不会转发dog.puppy，也不会转发dog.guard，只会转发dog。 
![image](https://cloud.githubusercontent.com/assets/7789698/16560356/6d0cda70-4225-11e6-97bd-96f3a557078e.png)

Fanout Exchange – 不处理路由键。你只需要简单的将队列绑定到交换机上。一个发送到交换机的消息都会被转发到与该交换机绑定的所有队列上。很像子网广播，每台子网内的主机都获得了一份复制的消息。Fanout交换机转发消息是最快的。 
![image](https://cloud.githubusercontent.com/assets/7789698/16560357/764b5ce2-4225-11e6-91fe-9988a6a12df3.png)

Topic Exchange – 将路由键和某模式进行匹配。此时队列需要绑定要一个模式上。符号“#”匹配一个或多个词，符号“_”匹配不多不少一个词。因此“audit.#”能够匹配到“audit.irs.corporate”，但是“audit._” 只会匹配到“audit.irs”。我在RedHat的朋友做了一张不错的图，来表明topic交换机是如何工作的： 
![image](https://cloud.githubusercontent.com/assets/7789698/16560366/7ed92a88-4225-11e6-8334-e986473d9530.png)

```
 ConnectionFactory connFactory = new ConnectionFactory();//创建连接连接到MabbitMQ 
  connFactory.setUri(uri);//或 factory.setHost("localhost");  设置ip、uri或host
Connection connection = factory.newConnection();  //创建一个连接
  Channel channel = connection.createChannel();  //创建一个Channel 
channel.queueDeclare(queue, true, false, false, null);//指定队列
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());     //往队列中发出一条消息  
 //关闭频道和连接  
 channel.close();  
connection.close();  

```

Connecting to a broker

```
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(userName);
factory.setPassword(password);
factory.setVirtualHost(virtualHost);
factory.setHost(hostName);
factory.setPort(portNumber);
Connection conn = factory.newConnection();
```

[uri](http://www.rabbitmq.com/uri-spec.html)

```
ConnectionFactory factory = new ConnectionFactory();
factory.setUri("amqp://userName:password@hostName:portNumber/virtualHost");
Connection conn = factory.newConnection();
```

Using Exchanges and Queues
声明一个exchange然后把队列和exchange和队列绑定起来（只有绑定以后，往exchange投递才会跑到相应队列）

```
channel.exchangeDeclare(exchangeName, "direct", true);
String queueName = channel.queueDeclare().getQueue();
channel.queueBind(queueName, exchangeName, routingKey);
```

（完整的绑定过程）

```
channel.exchangeDeclare(exchangeName, "direct", true);
channel.queueDeclare(queueName, true, false, false, null);
channel.queueBind(queueName, exchangeName, routingKey);
```

Publishing messages

```
byte[] messageBodyBytes = "Hello, world!".getBytes();
channel.basicPublish(exchangeName, routingKey, null, messageBodyBytes);
```

```
channel.basicPublish(exchangeName, routingKey, mandatory,
                     MessageProperties.PERSISTENT_TEXT_PLAIN,
                     messageBodyBytes);
```

delivery mode 2 (persistent), priority 1 , content-type "text/plain".

```
channel.basicPublish(exchangeName, routingKey,
             new AMQP.BasicProperties.Builder()
               .contentType("text/plain")
               .deliveryMode(2)
               .priority(1)
               .userId("bob")
               .build()),
               messageBodyBytes);
```

自定义header

```
Map<String, Object> headers = new HashMap<String, Object>();
headers.put("latitude",  51.5252949);
headers.put("longitude", -0.0905493);

channel.basicPublish(exchangeName, routingKey,
             new AMQP.BasicProperties.Builder()
               .headers(headers)
               .build()),
               messageBodyBytes);
```

expiration

```
channel.basicPublish(exchangeName, routingKey,
             new AMQP.BasicProperties.Builder()
               .expiration("60000")
               .build()),
               messageBodyBytes);
```

在确认模式下发布大量的信息到一个通道,等待确认

```
package com.rabbitmq.examples;

import java.io.IOException;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.MessageProperties;
import com.rabbitmq.client.QueueingConsumer;

public class ConfirmDontLoseMessages {
    static int msgCount = 10000;
    final static String QUEUE_NAME = "confirm-test";
    static ConnectionFactory connectionFactory;

    public static void main(String[] args)
        throws IOException, InterruptedException
    {
        if (args.length > 0) {
                msgCount = Integer.parseInt(args[0]);
        }

        connectionFactory = new ConnectionFactory();

        // Consume msgCount messages.
        (new Thread(new Consumer())).start();
        // Publish msgCount messages and wait for confirms.
        (new Thread(new Publisher())).start();
    }

    @SuppressWarnings("ThrowablePrintedToSystemOut")
    static class Publisher implements Runnable {
        public void run() {
            try {
                long startTime = System.currentTimeMillis();

                // Setup
                Connection conn = connectionFactory.newConnection();
                Channel ch = conn.createChannel();
                ch.queueDeclare(QUEUE_NAME, true, false, false, null);
                ch.confirmSelect();

                // Publish
                for (long i = 0; i < msgCount; ++i) {
                    ch.basicPublish("", QUEUE_NAME,
                                    MessageProperties.PERSISTENT_BASIC,
                                    "nop".getBytes());
                }

                ch.waitForConfirmsOrDie();

                // Cleanup
                ch.queueDelete(QUEUE_NAME);
                ch.close();
                conn.close();

                long endTime = System.currentTimeMillis();
                System.out.printf("Test took %.3fs\n",
                                  (float)(endTime - startTime)/1000);
            } catch (Throwable e) {
                System.out.println("foobar :(");
                System.out.print(e);
            }
        }
    }

    static class Consumer implements Runnable {
        public void run() {
            try {
                // Setup
                Connection conn = connectionFactory.newConnection();
                Channel ch = conn.createChannel();
                ch.queueDeclare(QUEUE_NAME, true, false, false, null);

                // Consume
                QueueingConsumer qc = new QueueingConsumer(ch);
                ch.basicConsume(QUEUE_NAME, true, qc);
                for (int i = 0; i < msgCount; ++i) {
                    qc.nextDelivery();
                }

                // Cleanup
                ch.close();
                conn.close();
            } catch (Throwable e) {
                System.out.println("Whoosh!");
                System.out.print(e);
            }
        }
    }
}
```

*(the AMQP specification document)[http://www.amqp.org/]

接收消息的最有效的方法是建立一个订阅使用消费者接口。将自动被交付的消息到达,而不必显式地请求

```
boolean autoAck = false;
channel.basicConsume(queueName, autoAck, "myConsumerTag",
     new DefaultConsumer(channel) {
         @Override
         public void handleDelivery(String consumerTag,
                                    Envelope envelope,
                                    AMQP.BasicProperties properties,
                                    byte[] body)
             throws IOException
         {
             String routingKey = envelope.getRoutingKey();
             String contentType = properties.getContentType();
             long deliveryTag = envelope.getDeliveryTag();
             // (process the message components here ...)
             channel.basicAck(deliveryTag, false);
         }
     });
```

接收个别信息

```
boolean autoAck = false;
GetResponse response = channel.basicGet(queueName, autoAck);
if (response == null) {
    // No message retrieved.
} else {
    AMQP.BasicProperties props = response.getProps();
    byte[] body = response.getBody();
    long deliveryTag = response.getEnvelope().getDeliveryTag();
    ...
    channel.basicAck(method.deliveryTag, false); //  autoAck = false必须设置 Channel.basicAck来确认已经接受消息
```

处理不被路由的消息
假如一个信息被设置强制性（mandatory）的flag不被路由的话会被送到发送端。
如果客户端没有配置返回特定通道侦听器,将放弃返回的相关消息。
为了获取这个消息，客户端可以实现ReturnListener 接口还有调用 Channel.setReturnListener

```
channel.setReturnListener(new ReturnListener() {
    public void handleBasicReturn(int replyCode,
                                  String replyText,
                                  String exchange,
                                  String routingKey,
                                  AMQP.BasicProperties properties,
                                  byte[] body)
    throws IOException {
        ...
    }
});
```

关闭协议

The AMQP 0-9-1 connection and channel have the following lifecycle states:

open: the object is ready to use
closing: the object has been explicitly notified to shut down locally, has issued a shutdown request to any supporting lower-layer objects, and is waiting for their shutdown procedures to complete
closed: the object has received all shutdown-complete notification(s) from any lower-layer objects, and as a consequence has shut itself down

The AMQP connection and channel objects possess the following shutdown-related methods:

addShutdownListener(ShutdownListener listener) and removeShutdownListener(ShutdownListener listener), to manage any listeners, which will be fired when the object transitions to closed state. Note that, adding a ShutdownListener to an object that is already closed will fire the listener immediately
getCloseReason(), to allow the investigation of what was the reason of the object’s shutdown
isOpen(), useful for testing whether the object is in an open state
close(int closeCode, String closeMessage), to explictly notify the object to shut down.

```
import com.rabbitmq.client.ShutdownSignalException;
import com.rabbitmq.client.ShutdownListener;

connection.addShutdownListener(new ShutdownListener() {
    public void shutdownCompleted(ShutdownSignalException cause)
    {
        ...
    }
});
```

ShutdownSignalException包含了关闭时的错误异常

```
public void shutdownCompleted(ShutdownSignalException cause)
{
  if (cause.isHardError())
  {
    Connection conn = (Connection)cause.getReference();
    if (!cause.isInitiatedByApplication())
    {
      Method reason = cause.getReason();
      ...
    }
    ...
  } else {
    Channel ch = (Channel)cause.getReference();
    ...
  }
}
```

原子性的使用open

```
public void brokenMethod(Channel channel)
{
    if (channel.isOpen())
    {
        // The following code depends on the channel being in open state.
        // However there is a possibility of the change in the channel state
        // between isOpen() and basicQos(1) call
        ...
        channel.basicQos(1);//告诉RabbitMQ同一时间给一个消息给消费者  
    }
}
```

处于无效状态时应该抓取异常

```
public void validMethod(Channel channel)
{
    try {
        ...
        channel.basicQos(1);
    } catch (ShutdownSignalException sse) {
        // possibly check if channel was closed
        // by the time we started action and reasons for
        // closing it
        ...
    } catch (IOException ioe) {
        // check why connection was closed
        ...
    }
}
```

连接设置
设置pool数

```
ExecutorService es = Executors.newFixedThreadPool(20);
Connection conn = factory.newConnection(es);
```

使用地址列表

```
Address[] addrArr = new Address[]{ new Address(hostname1, portnumber1)
                                 , new Address(hostname2, portnumber2)};
Connection conn = factory.newConnection(addrArr);
```

心跳超时（Heartbeat Timeout）  [Heartbeats guide](http://www.rabbitmq.com/heartbeats.html)
自定义线程工厂

```
import com.google.appengine.api.ThreadManager;

ConnectionFactory cf = new ConnectionFactory();
cf.setThreadFactory(ThreadManager.backgroundThreadFactory());
```

Automatic Recovery From Network Failures

```
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(userName);
factory.setPassword(password);
factory.setVirtualHost(virtualHost);
factory.setHost(hostName);
factory.setPort(portNumber);
factory.setAutomaticRecoveryEnabled(true);
// connection that will recover automatically
Connection conn = factory.newConnection();


ConnectionFactory factory = new ConnectionFactory();
// attempt recovery every 10 seconds
factory.setNetworkRecoveryInterval(10000);


ConnectionFactory factory = new ConnectionFactory();

Address[] addresses = {new Address("192.168.1.4"), new Address("192.168.1.5")};
factory.newConnection(addresses);
```

The RPC (Request/Reply) Pattern

```
import com.rabbitmq.client.RpcClient;

RpcClient rpc = new RpcClient(channel, exchangeName, routingKey);



byte[] primitiveCall(byte[] message);
String stringCall(String message)
Map mapCall(Map message)
Map mapCall(Object[] keyValuePairs)
```