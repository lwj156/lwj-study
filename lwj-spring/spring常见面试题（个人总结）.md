# spring常见面试题（个人总结）

>其实准备面试的最好方式就是自己`模拟面试`
>
>想象模拟面试官会提问的问题，自己把自己的回答总结下来，到时候组织语言可能会缺乏逻辑性



## 什么是spring IOC

ioc就是将bean的生命周期交给ioc容器，ioc容器有AnnotationApplicationContext、classpathAC等。通过ioc容器将配置的bean信息（xml配置或者注释标注）加载到BeanDefinition当中，然后通过反射初始化到ioc容器当中。此时的对象是BeanWrapper对象



## BeanWrapper和BeanDefinition分别存储什么

>BeanWrapper存储类的原始对象、代理对象、父类对象等信息
>
>BeanDefinition存储类的属性、构造参数和方法等信息



## 什么是spring DI

DI指的就是容器主动将依赖的类注入给它，一般非懒加载的类在ioc容器的refresh方法的时候就会统一调用BeanFactory实现类的doGetBean方法去初始化，懒加载在调用的时候进行初始化。在beanWrapper实现类中setValue方法通过反射注入

补充：单例对象在另外一个缓存的容器当中



## 什么是spring AOP

aop底层就是通过代理模式生成代理类，代理策略主要看配置，默认如果有接口有jdk动态代理，其他情况用cglib代理。声明的前置通知、后置通知等就是通过维护的一个责任链模式（List chain）去层层调用。多个切面可以用order参数控制

**补充**：aop配置的通知会被解析成MethodInterceptor，mybatis也有类似的实现`插件`，不过插件是多层代理，也是责任链+代理模式实现



## BeanFactory和ApplicationContext的区别

ApplicationContext是BeanFactory的子接口

1. ApplicationContext对bean的监控，支持国际化
2. ApplicationContext统一资源的读取方式



### spring Bean的生命周期

监控方式：

1. InitializingBean和DisposableBean用来回调

   xml需要声明：init-method="initMethod" destroy-method="destroyMethod"

   注解式：@PostConstruct和@PreDestroy

   >InitializingBean：bean设值后调用afterPropertiesSet()
   >
   >DisposableBean：在销毁bean前调用destroy()

2. Aware接口

3. BeanPostProcessor扩展

   

控制生命周期：

spring bean各作用域之间的区别



1. 定位加载：加载spring bean的配置并通过反射实例化，配置的值利用set赋值
2. 若实现*.Aware接口，调用对应的方法，在Bean初始化的时候可以获取对应的值，如BeanNameAware调用setBeanName可以获取beanName的值
3. 加载相关的BeanPostProcessor，执行postProcessBeforeInitialization方法
4. 若实现的InitializingBean，实现afterPropertiesSet方法
5. 调用配置文件的init-method
6. 调用postProcessAfterInitialization方法
7. 如果bean实现了DisposableBean，调用destroy方法
8. 如果配置了destroy-method，执行指定方法



![image-20200706134520811](spring%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98%EF%BC%88%E4%B8%AA%E4%BA%BA%E6%80%BB%E7%BB%93%EF%BC%89.assets/image-20200706134520811.png)

![image-20200706134833649](spring%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98%EF%BC%88%E4%B8%AA%E4%BA%BA%E6%80%BB%E7%BB%93%EF%BC%89.assets/image-20200706134833649.png)



### springmvc工作原理

1.web服务器解析http请求， 匹配DispatcherServlet的请求路径。

2.DispatcherServlet根据请求信息以及HandlerMapping配置信息找到处理器Handler。

3.通过HandlerAdapter对Handler进行调用。即controller处理业务逻辑。

4.返回一个ModelAndView对象。通过ViewResolver（视图解析器）转化为视图。

5.返回给客户端/浏览器。



### springbean的作用域

单例、多例、request、session
