---
title: RabbitMQ整合SpringBoot
date: 2021-05-03 21:03:59
tags: ['RabbitMQ', '使用心得', '中间件']
---

# RabbitMQ的使用



![image-20210503210600807](RabbitMQ%E6%95%B4%E5%90%88SpringBoot/image-20210503210600807.png)

在 `Rabbitmq` 的模型中，`Server` 中会包含很多个虚拟主机 `Virtual Host` ，这就类似于数据库中的库，用来和项目做一一的映射，不同的项目建立不同的主机，达到隔离每个项目的目的。

所以在创建项目前先创建一个主机

#### 在 `RabbitMQ` 中创建一个虚拟主机

![image-20201224095213586](RabbitMQ%E6%95%B4%E5%90%88SpringBoot/image-20201224095213586.png)

添加了一个虚拟主机，应该设置一个可以访问其的用户，`guest` 用户默认可以访问所有虚拟主机，这里创建一个用户只允许访问我们刚刚创建的主机

![image-20201224113706766](RabbitMQ%E6%95%B4%E5%90%88SpringBoot/image-20201224113706766.png)

新创建的用户默认是不能访问任何虚拟主机的，需要为其设置可以访问的虚拟主机

![image-20201224113836723](RabbitMQ%E6%95%B4%E5%90%88SpringBoot/image-20201224113836723.png)

![image-20201224113943657](RabbitMQ%E6%95%B4%E5%90%88SpringBoot/image-20201224113943657.png)



#### 添加依赖

```xml
<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
```

向 `application.yml` 中添加 `RabbitMQ` 的配置信息

```yml
spring:
  rabbitmq:
    host: 自己rabbit的路径
    port: 自己rabbit的端口
    virtual-host: /
    username: 自己rabbit的账号
    password: 自己rabbit的密码
    publisher-confirms: true
```

配置完成，整合`SpringBoot`需要完成下列步骤：

![image-20201203113907638](RabbitMQ%E6%95%B4%E5%90%88SpringBoot/image-20201203113907638.png)

`RabbitMQ` 的配置文件需要应用到Java中，所以我们增加获取配置文件信息的实体类

#### 增加获取配置文件信息的实体类

```java
@Configuration
public class RabbitProperties {

    /**
     * rabbitmq 服务器地址
     */
    @Value("${spring.rabbitmq.host}")
    private String host;

    /**
     * rabbitmq 服务器端口
     */
    @Value("${spring.rabbitmq.port}")
    private int port;

    /**
     * rabbitmq 账号
     */
    @Value("${spring.rabbitmq.username}")
    private String username;

    /**
     * rabbitmq 密码
     */
    @Value("${spring.rabbitmq.password}")
    private String password;
    
    // seter and getter
    
    // To-String menthod
    
}

```

获取到配置信息后，需要应用到我们的 `RabbitMQ` 程序中

#### 增加配置类

```java
@Configuration
public class RabbitConfiguration {

    @Autowired
    private RabbitProperties rabbitProperties;

    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory connectionFactory = new CachingConnectionFactory(rabbitProperties.getHost(), rabbitProperties.getPort());
        connectionFactory.setUsername(rabbitProperties.getUsername());
        connectionFactory.setPassword(rabbitProperties.getPassword());
        connectionFactory.setVirtualHost("/");
        connectionFactory.setPublisherConfirms(true);
        return connectionFactory;
    }

    /**
     * @return
     * @Scope(value=ConfigurableBeanFactory.SCOPE_PROTOTYPE)这个是说在每次注入的时候回自动创建一个新的bean实例
     * @Scope(value=ConfigurableBeanFactory.SCOPE_SINGLETON)单例模式，在整个应用中只能创建一个实例
     * @Scope(value=WebApplicationContext.SCOPE_GLOBAL_SESSION)全局session中的一般不常用
     * @Scope(value=WebApplicationContext.SCOPE_APPLICATION)在一个web应用中只创建一个实例
     * @Scope(value=WebApplicationContext.SCOPE_REQUEST)在一个请求中创建一个实例
     * @Scope(value=WebApplicationContext.SCOPE_SESSION)每次创建一个会话中创建一个实例
     * proxyMode=ScopedProxyMode.INTERFACES创建一个JDK代理模式
     * proxyMode=ScopedProxyMode.TARGET_CLASS基于类的代理模式
     * proxyMode=ScopedProxyMode.NO（默认）不进行代理
     */
    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_SINGLETON)
    public RabbitTemplate rabbitTemplate() {
        RabbitTemplate template = new RabbitTemplate(connectionFactory());
        // 消息发送失败返回到队列中, yml需要配置 publisher-returns: true
        // template.setMandatory(true);
        return template;
    }
}

```

`RabbitMQ` 中需要存在队列、交换机等组件，需要将其用Java的方式定义出来

#### 定义 `RabbitMQ` 各个组件

```java
public class RabbitMqKey {

    /**
     * 订单-队列
     */
    public static final String TRADE_ORDER_QUEUE = "trade-order-queue";

    /**
     * 订单-交换器
     */
    public static final String TRADE_ORDER_EXCHANGE = "trade-order-exchange";

}
```

每个组件都是单独存在的，需要将他们联系起来

#### 初始化队列、交换机等并绑定关系

> **由Exchange、Queue、Routing Key三个才能决定一个从Exchange到Queue的唯一的线路。**

```java
@Component
public class TradeOrderQueueConfig {


    private final static Logger logger = LoggerFactory.getLogger(TradeOrderQueueConfig.class);

    /**
     * 创建队列
     * Queue 可以有4个参数
     * String name: 队列名
     * boolean durable: 持久化消息队列，rabbitmq 重启的时候不需要创建新的队列，默认为 true
     * boolean exclusive: 表示该消息队列是否只在当前的connection生效，默认为 false
     * boolean autoDelete: 表示消息队列在没有使用时将自动被删除，默认为 false
     * Map<String, Object> arguments:
     *
     * @return
     */
    @Bean(name = "queue")
    public Queue queue() {
        logger.info("queue : {}", RabbitMqKey.TRADE_ORDER_QUEUE);
        // 队列持久化
        return new Queue(RabbitMqKey.TRADE_ORDER_QUEUE, true);
    }

    /**
     * 创建一个 Fanout 类型的交换器
     * <p>
     * rabbitmq中，Exchange 有4个类型：Direct，Topic，Fanout，Headers
     * Direct Exchange：将消息中的Routing key与该Exchange关联的所有Binding中的Routing key进行比较，如果相等，则发送到该Binding对应的Queue中；
     * Topic Exchange：将消息中的Routing key与该Exchange关联的所有Binding中的Routing key进行对比，如果匹配上了，则发送到该Binding对应的Queue中；
     * Fanout Exchange：直接将消息转发到所有binding的对应queue中，这种exchange在路由转发的时候，忽略Routing key；
     * Headers Exchange：将消息中的headers与该Exchange相关联的所有Binging中的参数进行匹配，如果匹配上了，则发送到该Binding对应的Queue中；
     *
     * @return
     */
    @Bean(name = "fanoutExchange")
    public FanoutExchange fanoutExchange() {
        logger.info("exchange : {}", RabbitMqKey.TRADE_ORDER_EXCHANGE);
        return new FanoutExchange(RabbitMqKey.TRADE_ORDER_EXCHANGE);
    }

    /**
     * 把队列（Queue）绑定到交换器（Exchange）
     * topic 使用路由键（routingKey）
     *
     * @return
     */
    @Bean
    Binding fanoutBinding(@Qualifier("queue") Queue queue,
                    @Qualifier("fanoutExchange") FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(queue).to(fanoutExchange);
    }
}
```

到此，`RabbitMQ` 的所有服务组件已经搭建和连接完成了，可以使用 `Sender` 发送消息和接收者接收消息了。

#### 新建发送消息类

```java
@Component
public class Sender {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    /**
     * 如果rabbitTemplate的scope属性设置为ConfigurableBeanFactory.SCOPE_PROTOTYPE，所以不能自动注入
     * 需手动注入
     */
    
    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 订单信息（发送至交换器）
     *
     * @param payload
     * @return
     */
    public String orderSendExchange(Object payload){
        return baseSend(RabbitMqKey.TRADE_ORDER_EXCHANGE, "", payload, null, null);
    }

    /**
     * 订单信息（发送至队列）
     *
     * @param payload
     * @return
     */
    public String orderSendQueue(Object payload){
        return baseSend("", RabbitMqKey.TRADE_ORDER_QUEUE, payload, null, null);
    }

    /**
     * MQ 发送数据基础方法
     *
     * @param exchange  交换器名
     * @param routingKey  队列名
     * @param payload 消息信息
     * @param uniqueMessageId  标示id，不传可自动生成
     * @param messageExpirationTime  持久化时间
     * @return 消息编号
     */
    public String baseSend(String exchange, String routingKey, Object payload, String uniqueMessageId, Long messageExpirationTime) {
        // 生成消息ID
        String finalUniqueMessageId = uniqueMessageId;
        if (StringUtils.isBlank(uniqueMessageId)) {
            uniqueMessageId = UUID.randomUUID().toString();
        }
        logger.info("SEND --- unique message id：{}", uniqueMessageId);

        // 消息属性
        MessagePostProcessor messagePostProcessor = new MessagePostProcessor() {
            @Override
            public Message postProcessMessage(Message message) throws AmqpException {
                // 消息属性中写入消息编号
                message.getMessageProperties().setMessageId(finalUniqueMessageId);
                // 消息持久化时间
                if (!StringUtils.isEmpty(String.valueOf(messageExpirationTime))) {
                    logger.info("设置消息持久化时间：{}", messageExpirationTime);
                    message.getMessageProperties().setExpiration(Long.toString(messageExpirationTime));
                }
                // 设置持久化模式
                message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                return message;
            }
        };

        logger.info("SEND --- messagePostProcessor：{}", messagePostProcessor);

        // 消息
        Message message = null;
        try {
            ObjectMapper objectMapper = new ObjectMapper();
            String json = objectMapper.writeValueAsString(payload);
            logger.info("发送消息：{}", payload.toString());
            // 转换数据格式
            MessageProperties messageProperties = new MessageProperties();
            messageProperties.setContentEncoding(MessageProperties.CONTENT_TYPE_JSON);
            message = new Message(json.getBytes(), messageProperties);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }

        // correlationData
        CorrelationData correlationData = new CorrelationData(uniqueMessageId);

        /**
         * convertAndSend(String exchange, String routingKey, Object message, MessagePostProcessor messagePostProcessor, CorrelationData correlationData)
         * exchange: 路由
         * routingKey: 绑定key
         * message: 发送消息
         * messagePostProcessor: 消息属性处理类
         * correlationData: 对象内部只有一个 id 属性，用来表示当前消息唯一性
         */
        rabbitTemplate.convertAndSend(exchange, routingKey, message, messagePostProcessor, correlationData);

        return finalUniqueMessageId;
    }
}

```

发送的消息不一定会成功送到队列中，所以要增加一个确认机制

#### 确认消息

```java
@Component
public class RabbitAck implements RabbitTemplate.ConfirmCallback {

    private final static Logger logger = LoggerFactory.getLogger(RabbitAck.class);

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        //指定 ConfirmCallback
        //rabbitTemplate如果为单例的话，那回调就是最后设置的内容
        rabbitTemplate.setConfirmCallback(this);
    }

    /**
     * @param correlationData
     * @param ack
     * @param cause
     */
    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
        logger.info("ACK --- MQ message id: {}" + correlationData);
        if (ack) {
            logger.info("ACK --- Message sent confirmation success！");
        } else {
            logger.info("ACK --- MQ message id: {}", correlationData.getId());
            logger.info("ACK --- MQ confirmetion: {}", ack);
            logger.info("ACK --- Message sending confirmation failed, reason for failure:" + cause);
        }
    }
}

```

#### 接收消息

```java
@Component
public class OrderQueueListener {

    private static final Logger logger = LoggerFactory.getLogger(OrderQueueListener.class);

    /**
     * 接收消息
     *
     * @param message
     */
    @RabbitListener(queues = RabbitMqKey.TRADE_ORDER_QUEUE)
    public void process(Message message) {
        try {
            String msg = new String(message.getBody());
            if (StringUtils.isBlank(msg)) {
                logger.warn("接收的数据为空");
                return;
            }
            System.out.println(msg);
        } catch (Exception e) {
            logger.warn("处理接收到数据，发生异常：{}", e.getMessage());
            e.printStackTrace();
        }
    }
}

```

