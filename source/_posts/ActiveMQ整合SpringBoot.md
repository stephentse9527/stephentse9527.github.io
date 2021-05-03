---
title: ActiveMQ整合SpringBoot
date: 2021-05-03 21:16:08
---

# ActiveMQ 整合 SpringBoot

##### 引入 activemq 依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
```

配置 `yml` 文件

```yml
server:
  port: 7777

spring:
  activemq:
    broker-url: tcp://localhost:61617
    user: admin
    password: admin
  jms:
    # false = Queue
    # true = Topic 不写默认值 false
    pub-sub-domain: false

queue-name: boot-activemq-queue

```

> 因为有时候公司框架的原因，很多都无法使用，所以当配置文件无法使用时，可以使用`JavaConfig` 的方式配置

- 配置连接工厂
- 配置存放主题和队列的 Container
- 有了容器，自然就是创建`主题`或者是`队列`
- 最后要有一个 `JmsMessagingTemplate`  来发送消息
- 再设置一个接收消息的服务
- 因为有时候需要支持使用POJO来包装消息，默认的消息转换器不合适，所以要注入一个Json转换消息的转换器

```java
@Configuration
public class AmqConfig {
    // 1. 配置连接工厂
        @Bean
    public ConnectionFactory connectionFactory() {
        AmqServer amqServer = serverDiscovery.findServer();
        String brokerUrl = "tcp://" + amqServer.getIp() + ":7018";
        String username = amqServer.getUsername();
        String password = amqServer.getPassword();

        log.info("url：{}, 账号:{}, 密码: {}", brokerUrl, username, password);
        // 设置账号密码
        ActiveMQConnectionFactory factory = new ActiveMQConnectionFactory(username, password, brokerUrl);
        // 当需要发送接收序列化的POJO类时，在新版本中需要相信类所在包，所以打开此开关
        factory.setTrustAllPackages(true);
        return factory;
    }
    
    // 2. 配置 containerFactory
    // 在Topic模式中，对消息的监听需要对containerFactory进行配置
    @Bean("topicListener")
    public JmsListenerContainerFactory<?> topicJmsListenerContainerFactory(ConnectionFactory connectionFactory){
        SimpleJmsListenerContainerFactory factory = new SimpleJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        // 设置为true表示存放 topic
        factory.setPubSubDomain(true);
        return factory;
    }
    // 3. 创建一个收发消息的主题
    @Bean("topic")
    public Topic topic() {
        return new ActiveMQTopic(AmqDestination.SOPS_TOPIC);
    }
    
    // 4. 创建一个操作消息的JmsMessagingtemplate
        @Bean
    public JmsMessagingTemplate jmsMessagingTemplate() {
        return new JmsMessagingTemplate(connectionFactory());
    }  
    // 5. 注入消息转化器，使其可以转换序列化的类
        /**
     *
     *  Serialize message content to json using TextMessage
     *  使用 TextMessage 转化成 序列化后的 json
     *  注意要POJO类要序列化
     * */
    @Bean
    public MessageConverter jacksonJmsMessageConverter() {
        MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
        converter.setTargetType(MessageType.TEXT);
        converter.setTypeIdPropertyName("_type");
        return converter;
    }
}
```

需要 `Spring Boot` 支持消息队列，在启动类上加上 `@EnableJms` 注解就可以实现

### 生产者使用方法

```java
@RestController
public class AmqController {

    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;

    @Autowired
    private Topic topic;

    @PostMapping("/send")
    public TlncMessageDto send(@RequestBody TlncMessageDto message) {
        sendMessage(topic, message);
        return message;
    }
	
    // 因为已经注入了Json转化器，所以可以直接传入Object作为消息的payload
    protected void sendMessage(Destination destination, Object payload) {
        jmsMessagingTemplate.convertAndSend(destination, payload);
    }

}
```

### 对应消息对象

```java
// JsonInclude表示对null字段的序列化策略，ALways表示总是序列化null字段，NON_NULL表示不序列化
@JsonInclude(JsonInclude.Include.ALWAYS)
// JsonPropertyOrder 表示序列化和反序列化的时候，字段的顺序
@JsonPropertyOrder({
        "listType",
        "params"
})
@Data
// 需要实现Serializable接口来序列化
public class TlncMessageDto implements Serializable {

    // 表示Json字段名对应类的这个成员变量
    @JsonProperty("listType")
    public String listType;
    @JsonProperty("params")
    public ParamsDto params;
    // 表示序列化时如果没发现匹配的字段可以忽略
    @JsonIgnore
    private Map<String, Object> additionalProperties = new HashMap<String, Object>();
    private final static long serialVersionUID = -201428069813570775L;

    // 表示额外字段，可以获得成员变量没有的字段
    @JsonAnyGetter
    public Map<String, Object> getAdditionalProperties() {
        return this.additionalProperties;
    }

    // 表示添加额外的字段，配合@JsonAnyGetter使用
    @JsonAnySetter
    public void setAdditionalProperty(String name, Object value) {
        this.additionalProperties.put(name, value);
    }

}
```

##### 还可以定时投放消息

在方法上添加 @Scheduled 注解，启动主启动类即可，3000L指的是3秒钟

```java
@Service
public class MQService {
    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;

    @Autowired
    private Queue queue;

    @Scheduled(fixedDelay = 3000L)
    public void sendScheduledMessage() {
        jmsMessagingTemplate.convertAndSend(queue, "holy shit!!!!!" + UUID.randomUUID().toString());
    }
}
```

### 消费者使用方法

消费者只需要在方法上添加 `@JmsListener` 即可

```java
@Slf4j
@Service
public class   {
    @JmsListener(destination = "${queue-name}", containerFactory = "topicListener")
    // 参数直接写对应的类
    public void receive(TlncMessageDto message) {
		// TO-DO
    }
}
```

