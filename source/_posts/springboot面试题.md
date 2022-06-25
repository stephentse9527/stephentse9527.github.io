### 1、什么是 Spring Boot？

- Spring Boot 主要是简化了使用 Spring 的难度，提供了各种启动器、引入JavaConfig来减省了繁重的xml配置，开发者能快速上手，并且内置了web服务，不需要而外启动tomcat服务器
- 

### 2、为什么要用 Spring Boot？

- 独立运行spring 项目。Springboot可以以jar包的形式直接运行，节省服务器资源
- 提供starter，简化maven配置
- 自动配置Spring：Spring Boot会根据项目依赖来自动配置Spring 框架，极大地减少项目要使用的配置
- 包含Spring所有特性和优点，包括IOC容器自动管理Bean生命周期、依赖注入等

### 3、Spring Boot 的核心配置文件有哪几个？它们的区别是什么？

**一共有**

- bootstrap (.yml 或者 .properties)
- application (.yml 或者 .properties)

**区别**

-  bootstrap 加载的优先级比 applicaton 高
- 应用场景：application 配置文件这个容易理解，主要用于 Spring Boot 项目的自动化配置，bootstrap 主要用于从额外的资源来加载配置信息，还可以在本地外部配置文件中解密属性

### 4、Spring Boot 的配置文件有哪几种格式？它们有什么区别？

- yml或者properties
- properties每个属性都要写全名，而yml可以共用前缀，严格按照缩进形式
- yml 不支持通过@PeopertySource注解来倒入配置

### 5、Spring Boot 的核心注解是哪个？它主要由哪几个注解组成的？

- **SpringBootApplication**注解

  通常在启动类上面，表明这是一个SpringBoot程序，并且会自动装配必要的配置，并且它是一个组合注解，组合中比较重要的有`SpringBootConfiguration`、`EnableAutoConfiguration`

- SpringBootConfiguration，本质上是一个Configuration注解，Configuration注解的作用搭配注解Bean来使用，是一个JavaConfig，会将带有Bean注解的方法返回的实例注入到IOC容器中

- **ComponentScan**：Bean自动扫描，会扫描制定路径下的被@Componet、@Controller @Servicer @Repatriy注解的类，生成实例注入IOC容器中

### 6、开启 Spring Boot 特性有哪几种方式？

- 继承spring-boot-starter-parent项目
- 导入spring-boot-dependencies项目依赖

### 7、Spring Boot 需要独立的容器运行吗？

- 不需要，其内置了tomcat，打包的时候是jar包的形式，可以直接运行

### 8、运行 Spring Boot 有哪几种方式？

- 打包jar包，打包war包

### 9、Spring Boot 自动配置原理是什么？

- Spring Boot的自动配置注解是@EnableAutoConfiguration，它被组合在@SpringBootApplication中，在EnableAutoConfiguration中，使用@Import注解注入了一个AutoConfigurationImportSelector，这个类实现了ImportSelector接口，而ImportSelector接口有一个方法selectImports，在AutoConfigurationImportSelector这个类实现的逻辑中，会去引入的每个starter包下面的`MATE-INF/spring.factories下，读取保存的配置类文件，然后进行Bean的去重、剔除@exclution注解包含的Bean，然后就可以把所有外部引入的bean信息返回，然后根据Bean的conditional注解来判断是否可以加载这个Bean到IOC容器中，这样就完成了自动装配

### 10、Spring Boot 的目录结构是怎样的？

### 11、你如何理解 Spring Boot 中的 Starters？

### 12、如何在 Spring Boot 启动的时候运行一些特定的代码？

- 实现applicationRuner或者commendLineRunner接口

### 13、Spring Boot 有哪几种读取配置的方式？

​	三种

- @Value注解

- @ConfigurationProperties

- 注入Environment类

### 14、Spring Boot 支持哪些日志框架？推荐和默认的日志框架是哪个？

### 15、SpringBoot 实现热部署有哪几种方式？

### 16、你如何理解 Spring Boot 配置加载顺序？

- 同目录层级下Perproties > yml
- 目录之间的关系
  - 根目录下的config > 根目录 > resource下的config > resource
- 也可以通过--spring.config.name 或者是sping.config.location来指定自定义的目录和文件名

### 17、Spring Boot 如何定义多套不同环境配置？

- Application.properties文件名中加上后缀prod prion dev 然后在启动参数添加spirng.profile.active来选择对应环境参数

### 18、Spring Boot 可以兼容老 Spring 项目吗，如何做？

### 19、保护 Spring Boot 应用有哪些方法？

### 20、Spring Boot 2.X 有什么新特性？与 1.X 有什么区别？

### 21、springboot和springcloud区别

- Spring Boot 是 Spring 的⼀套快速配置脚⼿架，可以基于Spring Boot 快速开发单个微服务，Spring Cloud是一个基于Spring Boot实现的云应⽤开发⼯具；
- **Spring Boot专注于快速、⽅便集成的单个微服务个体，Spring Cloud关注全局的服务治理框架**

### 22.**eureka和zookeeper都可以提供服务的注册与发现功能，他们的区别**

### 23.springcloud如何实现服务的注册和发现

- 服务再部署时，会将自己注册到eureka或者zookeeper注册中心上去，完成注册流程，同一个服务名如果部署多个实例，而服务在调用其他服务的方法通过远程调用的形式，从服务中心中获取可用的实例列表，然后通过负载均衡算法选出一个实例，将自己的参数序列化，然后传递给远程服务，远程服务则在响应请求时，反序列化参数，然后调用内部方法，获得结果后将结果序列化返回给客户端。

### 24.SpringBoot启动流程

1. 首先从main找到run()方法，在执行run()方法之前new一个`SpringApplication`对象
2. 进入run()方法，创建应用监听器`SpringApplicationRunListeners`开始监听
3. 然后加载SpringBoot配置环境(`ConfigurableEnvironment`)，然后把配置环境(Environment)加入监听对象中
4. 然后加载应用上下文(`ConfigurableApplicationContext`)，当做run方法的返回对象
5. 最后创建Spring容器，`refreshContext(context)`，实现starter自动化配置和bean的实例化等工作。

### 25.Spring 事务

Spring的事物隔离级别和数据库是一至的，同样有读未提交、读已提交，可重复读、可串行化这几个，

如果spring配置的事务和数据库不一致，则根据spirngwei

