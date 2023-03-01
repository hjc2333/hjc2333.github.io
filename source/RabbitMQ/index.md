title: RabbitMQ
date: 2023-03-01 10:04:16
---
# RabbitMQ

### exchange

- Direct Exchange

  根据queue绑定exchange时的routing key 与消息的routing key进行匹配，把消息路由到routing key 一致的队列，再从每个队列中轮询消费者进行负载均衡，每个队列找出一个消费者进行消费消息。

- Topic Exchange

  消息发送到Topic Exchange 时所带的routing key  必须是一连串用`.`分割的词表（例如“stock.usd.nyse”；限制在255字节内）。

  队列绑定topic exchange 有两种重要的特殊的key：

  - `*` 星号只能呢个匹配一个词。
  - `#`能匹配0个或者多个词。

- Fanout Exchange

  将消息路由到交换机上的每个队列，再从每个队列中轮询消费者进行负载均衡，每个队列找出一个消费者进行消费消息。

- Headers Exchange

  与Topic Exchange 相似，但使用header而不是routing key 进行匹配。队列可以使用多个header去绑定交换机，其中`x-match`绑定参数可以决定匹配消息的一个header还是多个。

  `x-match`属性有两个可能的值：`any` 与 `all`,其中`all`是省略值。如果值为`all`表明队列绑定中所有header 都要匹配消息的header；如果值为 `any`表示至少有一对键值对匹配。

  > 如果队列绑定时，header的键值对中值为null，则只需消息有对应的键即可，不关心值是否也为null。

  下面的示例中，`test1Queue`也会收到消息

  ```java
  @Bean("binding1")
  Binding binding(@Qualifier("test1Queue") Queue testQueue,@Qualifier("test1Exchange") HeadersExchange exchange) {
      Map<String,Object> map  = new HashMap<>();
      map.put("test",null);
      map.put("1",null);
      return BindingBuilder.bind(testQueue).to(exchange).whereAll(map).match();
  }
  
  @Test
      void contextLoads() {
          MessageProperties messageProperties = new MessageProperties();
          messageProperties.setHeader("test", "1");
          messageProperties.setHeader("1","abc");
          messageProperties.setHeader("test1213","2");
          MessageConverter messageConverter = new SimpleMessageConverter();
          Message message = messageConverter.toMessage("hello world", messageProperties);
          rabbitTemplate.send("test1Exchange","", message);
      }
  ```

### Message

消息的持久化需要三个方面配合才能实现MQ节点重启也不会丢失消息。

1. 队列的durable属性为true。
2. exchange的durable熟悉为true。
3. 消息发送时配置deliveryMod 为 persistent

> 消息持久化会影响性能。

### Transactions

在spring  Rabbit 框架支持自动事务管理同步与异步用例，且有两种方式声明需要事务。在 `RabbitTemplate`与 `SimpleMessageListenerContainer`这两者中，都有一个`channelTransacted` 的标识符，当设置为`true`时就会使用 事务 channel 来提交或回滚发送或接收的操作。

一般来说，`channelTransacted`这个标识符应该在创建 AMQP 组件时声明，更常见的应该是应用启动时声明。不过在实际上，当事务被分层声明在应用程序中，标识符也是常作为一个配置设定。

- 对于使用 `RabbitTemplate`的同步用例，外部事务通常由调用方根据不同情况（通常是Spring 事务模型）来提供。例子：

```java
@Transactional
public void doSomething() {
    String incoming = rabbitTemplate.receiveAndConvert();
    // do some more database processing...
    String outgoing = processInDatabaseAndExtractReply(incoming);
    rabbitTemplate.convertAndSend(outgoing);
}
```

上面的例子在一个用`@Transactional`的注解方法里面，接收一个字符串载荷，经过数据处理转化为一个消息的body来发送。如果数据处理产生异常，那接收的消息会返回给中间件，并且消息也不会发送。

- 对于使用 `SimpleMessageListenerContainer`的异步用例来说，外部事务只有在该容器设置监听器时才有被设置的可能。如果需要外部事务，用户需要在该容器配置时提供 `PlatformTransactionManager`的实现。下面例子：

  ```java
  @Configuration
  public class ExampleExternalTransactionAmqpConfiguration {
  
      @Bean
      public SimpleMessageListenerContainer messageListenerContainer() {
          SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
          container.setConnectionFactory(rabbitConnectionFactory());
          container.setTransactionManager(transactionManager());
          container.setChannelTransacted(true);
          container.setQueueName("some.queue");
          container.setMessageListener(exampleListener());
          return container;
      }
  
  }
  ```

  上面的例子中，事务管理器被作为一个依赖注入添加。并且`channelTransacted`的标识符被设置为 `true`。这样设置就可以导致，当监听器失败产生异常时，事务会被回滚并且返回中间件中。还有当外部事务提交失败时，AMQP的事务也会回滚，消息会返回中间件。如果 `channelTransacted` 标识符设置为 `false` ，上面例子中外部事务依然会提供给监听器，但所有的消息操作都是自动应答，导致即使发生了回滚，消息操作一样被提交。

  