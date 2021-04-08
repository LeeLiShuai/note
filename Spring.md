**控制反转依赖注入**

IoC,控制反转这一种设计思想，将原本程序中手动创建对象的控制权，交由Spring框架来管理

理论上一个Map,保存各种对象

可以简化开发，把应用从负责的依赖关系中解放出来，IoC容器就像一个工厂，需要创建一个对象的时候，只需要配置文件或者注解即可。不用考虑对象是如何被创建出来的。，一个Service类可能以来很多底层类，如果不用IoC的话，需要手动创建很多个对象



依赖注入是在编译阶段不知道所需功能来自哪个类的情况下，讲其他独享所以来的功能对象实例化的模式。

依赖注入的几种实现方式：

1.构造器注入

2.属性注入

3.接口注入



IoC容器

BeanFactory:IoC的基本实现，Spring的基础设施，面向Spring本身

ApplicationContext:提供了更多的高级特性，面向开发者

​	主要实现类：

​		1.ClassPathXmlApplicationContext:从类路径下加载配置文件

​		2.FileSystemXmlApplicationContext:从文件系统中加载配置文件

​		3.ConfigurableApplicationContext

​		4.WebApplicationContext：为Web应用准备的，允许从相对于web根目录的路径中完成初始化工作中



实现原理

工厂模式和反射机制



**Spring Bean**

由IoC容器通过去实例化，配置，装配和管理的实例对象，基于用户提供的配置元数据创建

配置方式：

​	xml配置

​	注解配置

​	@Configuration,@Bean配置



**Spring Bean作用域**

singleton:单例模式，默认

prototype:每个bean请求一个实例

request：每个http请求一个实例

session：每个session一个实例

global-session:同session



**Spring Bean生命周期**

1.容器根据配置实例化bean

2.Spring使用历来注入填充所有属性

3.如果实现了BeanNameAware,则通过传递bean的ID来调用setBeanName

4.如果实现了BeanFactoryAware，则通过传递自身的实例来调用setBeanFactory

5其他xxxAware方法，调用xxx方法

6.如果存在于bean关联的BeanBeforeProcessors,调用preProcessBeforeInitizlization

7.如果执行了init方法，调用此方法

8.如果关联了BeanPostProcessors,调用preProcessPostInitizlization

9.如果实现了DisposableBean,容器关闭时，调用destory方法

10.如果bean指定destroy方法，调用此方法



配置Bean的方法

通过全类型反射，通过工厂方法，FactoryBean



**Spring装配**

bean在Spring容器中组合在一起时，称为bean装配，装配是创建应用对象之间协作关系的行为。

Spring容器需要知道什么bean以及容器如何使用依赖注入来将bean绑定在一起，同时装配bean

依赖注入的本质就是装配，装配是依赖注入的具体行为



**解决循环依赖**

解决循环依赖只能解决setter方法注入的情况，构造器注入无法解决，或者非单列模式无法解决。

setter方法注入情况可以解决是因为，可以先调用不需要参数的构造函数，然后在设置其他属性。

singletonObjects,并发map类型，一级缓存，存放完全初始化好的bean

earlySingletonObjects,hashmap,二级缓存，原始的bean,没有填充属性

singletonFactories，hashmap,三级缓存,存放bean工厂对象

getSingletion：首先从一级缓存中获取，如果一级缓存中没有且对应的bean正在创建中，则将一级缓存整个map使用synchronized锁住，然后从二级缓存中获取，如果二级缓存中也没有且允许从二级缓存中获取(一个参数控制，单例时可以)，就从三级缓存中获取，如果获取到了，就将缓存提升至二级缓存，并从三级缓存中删除。

getBean->doGetBean->createBean->doCreateBean->createBeanInstance->populateBean->initializeBean

createBeanInstance这一步创建一个原始引用，并放入三级缓存中。假设a依赖b，b依赖a.

a初始化时，到populateBean这一步是会调用getBean(b).b也会这一流程，在populateBean的时候调用getSingleton(a),此时返回a的初始引用，b就能完成初始化。然后返回给a的初始化流程，然后a也能完成初始化。



**AOP**

面向切面编程

切面：横切关注点，被模块化的特殊对象

连接点：程序执行的某个特殊位置，如调用某个方法前，后，抛出异常后等。在这个位置插入一个切面，即执行AOP的位置

通知：是在方法执行前或后要做的动作，即程序执行时要通过AOP触发的代码段

​	before：前置通知，在一个方法执行前调用

​	after：后置通知，在一个方法执行后调用，不论执行是否成功

​	after-returning:方法执行成功后执行

​	after-throwing:方法抛出异常后执行

​	around:执行之前和之后调用

目标Target:被通知的对象，代理对象

代理Proxy:向目标对象应用通知之后创建的对象



原理：动态代理，JDK动态代理代理接口，CGLIB动态代理代理类



**事务**

事务管理器，Spring不直接管理事务，提供多种事务管理器，将事务管理委托给其他持久层框架

提供PlatformTransactionManage接口，其他框架实现此接口管理事务

事务传播属性

| 传播行为                  | 具体                                                         |
| ------------------------- | ------------------------------------------------------------ |
| PROPAGATION_MANDATORY     | 方法必须运行于一个事务中，如果没有事务，则抛出异常           |
| PROPAGATION_NESTED        | 如果有实物，则方法运行在一个嵌套事务中，被嵌套的事务可以独立的提交或回滚，如果不存在事务，则和PROPAGATION_REQUIRES一样 |
| PROPAGATION_NEVER         | 从不运行在一个事务中，如果有事务，抛出异常                   |
| PROPAGATION_NOT_SUPPORTED | 不运行在十五中，如果有事务，事务则将该方法运行期间被挂起     |
| PROPAGATION_SUPPORTED     | 如果有事务，则加入事务                                       |
| PROPAGATION_REQUIRES_NEW  | 必须在自己的事务中运行，一个新事务启动，如果外层有事务,则这个方法运行期间被挂起 |
| PROPAGATINO_REQUIRES      | 必须运行在事务中，如果有事务郑爱运行，将加入事务，否则开启新事物 |

