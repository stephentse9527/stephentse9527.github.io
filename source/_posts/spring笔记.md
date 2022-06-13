---
title: Spring笔记
date: 2022-05-21 12:37:38
---

# Spring笔记

#### Spring核心要点

- **控制反转IOC**：使用Spring之前，对象的使用和创建是绑定在一起的，除了主要的逻辑代码外，还需要为依赖的其他对象做很多复杂的创建工作，**引入了Spring之后，就可以将创建和使用分离开**，把对象的创建工作交由Spring框架来进行处理，需要用到的时候从Spring的容器中获取即可
- **面向切面AOP**：不同业务模块的解耦

#### 控制反转IOC

##### 配置方式

- XML、JavaConfig
- 注解配置：@Component、@Controller、@Service、@Repository

##### 依赖注入的方式

- 构造方法注入（Construct注入），
- setter注入
- 基于注解的注入（接口注入）
  - @Autowired：根据Type类型，c出现多个就根据name来匹配

#### Spring MVC

一个基于MVC模式设计的轻量级web 框架

##### Spring MVC请求流程

![spring-springframework-mvc-5](spring笔记/spring-springframework-mvc-5.png)

- **首先用户发送请求——>DispatcherServlet**，前端控制器收到请求后自己不进行处理，而是委托给其他的解析器进行 处理，作为统一访问点，进行全局的流程控制；进入DispatcherServlet之前还会有Filter，可以做preFilter以及postFilter
- **DispatcherServlet——>HandlerMapping**， HandlerMapping 将会把请求映射为 HandlerExecutionChain 对象（包含一 个Handler 处理器（页面控制器）对象、多个HandlerInterceptor 拦截器）对象，通过这种策略模式，很容易添加新的映射策略；目的：获得请求映射到的handler处理器以及拦截器
- **DispatcherServlet——>HandlerAdapter**，HandlerAdapter 将会把处理器包装为适配器，从而支持多种类型的处理器， 即适配器设计模式的应用，从而很容易支持很多类型的处理器；
- **HandlerAdapter——>处理器功能处理方法的调用**，HandlerAdapter 将会根据适配的结果调用真正的处理器的功能处 理方法，完成功能处理；并返回一个ModelAndView 对象（包含模型数据、逻辑视图名）；
- **ModelAndView 的逻辑视图名——> ViewResolver**，ViewResolver 将把逻辑视图名解析为具体的View，通过这种策 略模式，很容易更换其他视图技术；
- **View——>渲染**，View 会根据传进来的Model 模型数据进行渲染，此处的Model 实际是一个Map 数据结构，因此 很容易支持其他视图技术；
- **返回控制权给DispatcherServlet**，由DispatcherServlet 返回响应给用户，到此一个流程结束。

#### Spring IOC实现原理

#### ![spring-framework-ioc-source-73](spring笔记/spring-framework-ioc-source-73.png)

Spring顶层设计围绕`BeanFactory`和`BeanRegistry`来进行

- **BeanFactory：工厂模式定义了IOC容器的基本功能规范**
- **BeanRegistry： 向IOC容器手工注册 BeanDefinition 对象的方法**

##### BeanDefinition：各种Bean对象及其相互的关系

Bean对象存在依赖嵌套等关系，所以设计者设计了BeanDefinition，它用来对Bean对象及关系定义；我们在理解时只需要抓住如下三个要点

- BeanDefinition 定义了各种Bean对象及其相互的关系

-  BeanDefinitionReader 这是BeanDefinition的解析器

-  BeanDefinitionHolder 这是BeanDefination的包装类，用来存储BeanDefinition，name以及aliases等

##### IOC初始化流程

![spring-framework-ioc-source-9](spring笔记/spring-framework-ioc-source-9.png)

- 首先定位资源文件，通过ResourceLoader来完成资源文件的定位，并将其抽象成Resource对象
- 接下来就是解析资源文件，通过 BeanDefinitionReader来完成定义信息的解析获取Bean的定义信息
- 获取到BeanDefinition后，需要将其注册到IOC容器中，这由 IOC 实现 BeanDefinitionRegistry 接口来实现。注册过程就是在 IOC 容器内部维护的一个HashMap 来保存得到的 BeanDefinition 的过程。这个 HashMap 是 IoC 容器持有 bean 信息的场所，以后对 bean 的操作都是围绕这个HashMap 来实现的.

##### Bean实例化

从定义信息中将BeanDefinition信息注册到IOC容器中后，只是保存了Bean信息，使用这些Bean的话还需要去进行Bean的实例化

#### Spring容器初始化

> 创建IOC容器也就是创建一个ApplicationContext接口的实现类，

- Spring容器初始化时，会调用 `refresh()` 刷新IOC容器
- refresh 里面先调用 `prepareRefresh()` ，记录容器启动时间，标记已启动状态
- 接下来会进入`obtainFreshBeanFactory()` 先重置BeanFactory，获得该BeanFactory对象
- 接下来使用loadBeanDefinitions，将Bean的元信息存放到BeanDefinition当中
- 获得BeanFactory后，进入prepareBeanFacotry为获得的BeanFacotory设置属性
- 这时候所有的Bean定义已经加载完成，但是都没实例化，这一步可以去修改bean 的定义或者增加Bean的定义
- 接下来会如果bean没有设置懒加载，就会加载这些单例bean



#### SpringBean的生命周期

- 当要获取一个bean时，会调用getBean方法，实际的逻辑在doGetBean当中，首先会调用getSingleton方法，尝试从一级二级三级缓存中获取bean

- **创建实例：**如果都没有，就会调用createBean方法来创建bean，通过beanDefinition获取bean 的信息，创建出一个原始对象，并将它放到三级缓存当中，三级缓存是以beanname做为key，可以执行函数作为value的map

- **属性注入**：接下来进行属性注入，初始化bean，值和Bean的引用注入进Bean对应的属性中

- **执行Aware方法**：执行Aware将在初始化前如果bean实现了Aware接口，就会执行这些Aware方法

- **Before-init**：如果Bean实现了BeanPostProcessor，执行`postProcessbeforeInitialization`，相当于初始化前的操作

- **init**：如果Bean实现了InitializingBean，执行InitializingBean的afterPropertiesSet方法，相当于初始化Bean

- **After-init：**同样，如果实现了BeanPostProcess，就执行BeanPostProcessor的`postProcessAfterInitialization`方法，而将原始对象变成代理对象是发生在BeanPostProcessor的postProcessAfterInitialization中的wrapIfNecessary方法中

- ```java
  protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
     if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
     }
     if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
     }
     if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
     }
  
     // Create proxy if we have advice.
     Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
     if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
              bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
     }
  
     this.advisedBeans.put(cacheKey, Boolean.FALSE);
     return bean;
  }
  ```

  

- 最后把完整的对象放进一级缓存中，同时删除其他缓存

#### Spring解决循环依赖的问题

- 当要获取一个bean时，会调用getBean方法，实际的逻辑在doGetBean当中，首先会调用getSingleton方法，尝试从一级二级三级缓存中获取bean
- 如果都没有，就会调用createBean方法来创建bean，通过beanDefinition获取bean 的信息，创建出一个原始对象，并将它放到三级缓存当中，三级缓存是以beanname做为key，可以执行函数作为value的map
- 当进行populaBean属性注入时，发现依赖B，就会去走getBean那一套，直到进行属性注入，这时候会发现A在三级缓存中已经有了，所以从三级缓存中拿出A的创建工厂，获得A的实例，B就顺利创建完成
- 而回到A这边B也创建好了，只需要进行后面的初始化就可以了，然后将bean从二级缓存中放到一级缓存

#### 为什么要三级缓存解决循环依赖

如果单纯解决循环依赖，不需要第三级缓存，只需要一个缓存（三级缓存）即可

而如果A进行了AOP操作的话，生成代理对象是在属性注入之后，如果发生循环依赖，在B的创建过程中获取A就需要提前获取到A的代理对象，而不是A的原始对象，所以要有第三个Map，来存放singletonFactory，在B中提前获得A的代理对象，这样B中注入的对象才会是和最后A的对象是一致的

#### 为什么只有一级缓存是ConcHashMap，其他是普通Map

因为二级缓存和三级缓存的put操作都会伴随着另一个的删除操作，要保证这两个操作的原子性，如果直接使用两个concmap，不能保证原子性，所以在对应的方法里面都是使用的sync关键字来直接加锁

