关于BeanDefinition:

BeanDefinition定义了如下信息：

1、对对象元数据进行操作的方法（设置与获取属性）

2、描述对象画像的操作（一系列的get、set、is方法）

3、定义Bean的角色常量

BeanDefinition只是定义了一系列的操作，描述Bean画像的相关属性交给了子类AbstractBeanDefinition，这个子类实现了BeanDefinition定义的一系列操作。

![](E:\study\再出发\image\spring-1.jpg)

GenericBeanDefinition：除了指定类、可选的构造函数参数值和属性参数外，还可以通过parenetName属性灵活设置父类的类全限定名;

RootBeanDefinition：在注册Bean阶段最常使用的就是RootBeanDefinition，它对应<bean>元素标签，在配置文件中可以定义父<bean>和子<bean>，父bean可以用RootBeanDefinition，子Bean可以用ChildBeanDefinition，从spring2.5开始，对于子bean，更常使用GenericBeanDefinition;

AnnotatedGenericBeanDefinition：以@Configuration注解标记的类会解析为AnnotatedGenericBeanDefintion

ConfigurationClassBeanDefinition：以@Bean注解标记的方法对应的Bean会解析为ConfigurationClassBeanDefinition;

ScannedGenericBeanDefinition：以@Component、@Service、@Controller注解标记的类会解析为ScannedGenericBeanDefinition



关于AbstractBeanDefinition：

​	AbstractBeanDefinition定义了一系列描述Bean画像的属性，同时实现了BeanDefinition定义的方法，通过这个类，可以窥见Bean的某些默认设置（例如默认为单例），这个类的属性基本可以在xml配置文件中找到相应的属性或是标签（例如该类的scope属性对应xml配置文件中的scope属性）。

- Bean的描述信息（例如是否是抽象类、是否单例）
- depends-on属性（String类型，不是Class类型）
- 自动装配的相关信息
- init函数、destroy函数的名字（String类型）
- 工厂方法名、工厂类名（String类型，不是Class类型）
- 构造函数形参的值
- 被IOC容器覆盖的方法
- Bean的属性以及对应的值（在初始化后会进行填充）



关于RootBeanDefinition：(与AbstractBeanDefinition是互补关系，RootBeanDefinition在AbstractBeanDefinition的基础上定义了更多属性，初始化Bean需要的信息基本完善)

- 定义了id、别名与Bean的对应关系（BeanDefinitionHolder）
- Bean的注解（AnnotatedElement）
- 具体的工厂方法（Class类型），包括工厂方法的返回类型，工厂方法的Method对象
- 构造函数、构造函数形参类型
- Bean的class对象

关于GenericBeanDefinition：

​	GenericBeanDefinition的patentName属性指定了当前类的父类，最重要的是它实现了parentName属性的setter、getter函数，**RootBeanDefinition没有parentName属性**，对应的getter函数只是返回null，setter函数不提供赋值操作。

​	也就是说RootBeanDefinition不提供继承相关的操作，但是初始化时使用的是RootBeanDefinition，那父类的性质如何体现？这里要注意一点，子类会覆盖父类中相同的属性，所以Spring**会首先初始化父类的RootBeanDefinition，然后根据子类的GenericBeanDefinition覆盖父类中相应的属性，最终获得子类的RootBeanDefinition，**这个比较巧妙。



关于ConfigurationClassBeanDefinition：

​	在@Configuration注解的类中，使用@Bean注解实例化的Bean，其定义会用ConfigurationClassBeanDefinition存储。

 ConfigurationClassBeanDefinition的默认设置：

1、如果@Bean注解没有指定bean的名字，默认会用方法的名字命名bean

2、@**Configuration注解的类会成为一个工厂类，而所有的@Bean注解的方法会成为工厂方法，通过工厂方法实例化Bean，而不是直接通过构造函数初始化**

3、@Bean注解注释的类会使用构造函数自动装配。





spring生命周期的四个阶段：

- 实例化（Instantiation）**-> 主要体现在createBean()方法**
  	DefaultListableBeanFactory 的抽象⽗类 AbstractAutowireCapableBeanFactory （createBean方法）完成了 Bean 的实例化和初始化。

  ```java
  protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
  		if (logger.isTraceEnabled()) {
  			logger.trace("Creating instance of bean '" + beanName + "'");
  		}
  		RootBeanDefinition mbdToUse = mbd;
  
  		//类加载校验，并不是所有的BeanDefinition都有BeanClass属性值的，例如使用@Bean注解初始化的Bean，这种情况下，就需要通过beanName获得Bean的类名
  		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
  		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
  			mbdToUse = new RootBeanDefinition(mbd);
  			mbdToUse.setBeanClass(resolvedClass);
  		}
  
  		//⽅法重写校验和准备
  		try {
  			mbdToUse.prepareMethodOverrides();
  		}
  		catch (BeanDefinitionValidationException ex) {
  			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
  					beanName, "Validation of method overrides failed", ex);
  		}
  
  		try {
  			//如果 Bean 配置了实例化的前置处理器，⽅法返回⾮空时，会直接使⽤返回的实例化 Bean ,不再进⾏后续流程。否则，继续使⽤容器的实例化
  			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
  			if (bean != null) {
  				return bean;
  			}
  		}
  		catch (Throwable ex) {
  			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,"BeanPostProcessor before instantiation of bean failed", ex);
  		}
  
  		try {
              //创建bean本身实例；如果有切面应用于Bean，则会为其生成代理
  			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
  			if (logger.isTraceEnabled()) {
  				logger.trace("Finished creating instance of bean '" + beanName + "'");
  			}
  			return beanInstance;
  		}catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
  			// A previously detected exception with proper bean creation context already,
  			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
  			throw ex;
  		}
  		catch (Throwable ex) {
  			throw new BeanCreationException(
  					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
  		}
  	}
  ```

  发现

  ```java
  protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
  		Object bean = null;
  		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
  			// Make sure bean class is actually resolved at this point.
  			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
  				Class<?> targetType = determineTargetType(beanName, mbd);
  				if (targetType != null) {
                      // 执⾏实例化前置处理器的 postProcessBeforeInstantiation ⽅法
  					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);// 如有返回对象，执⾏初始化后置处理器的 postProcessAfterInstantiation⽅法，完善创建流程
  					if (bean != null) {
  						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
  					}
  				}
  			}
  			mbd.beforeInstantiationResolved = (bean != null);
  		}
  		return bean;
  	}
  ```

  这⾥会检查3个条件

  - Bean的属性中的 beforeInstantiationResolved 字段是否为true 。
  - 原⽣的 Bean 。
  - Bean的 hasInstantiationAwareBeanPostProcessors 属性为true，这个属性在 Spring 准备刷新容器
    前准备 BeanPostProcessors 的时候会设置，如果当前Bean实现InstantiationAwareBeanPostProcessor 则这个就会是true。当 applyBeanPostProcessorsBeforeInstantiation 返回⾮空时，会直接调⽤初始化的后置处理器，中间的实例化后置处理器和初始化前置处理器都不执⾏。

此处引出**InstantiationAwareBeanPostProcessor**：

**BeanPostProcessor** :
	发生在 BeanDefiniton 加工Bean 阶段. 具有拦截器的含义. 可以拦截BeanDefinition创建Bean的过程, 然后插入拦截方法,做扩展工作.

- postProcessBeforeInitialization初始化前置处理 （对象已经生成）
- postProcessAfterInitialization初始化后置处理 （对象已经生成）

**InstantiationAwareBeanPostProcessor**: 继承于BeanPostProcessor ,所以他也是一种参与BeanDefinition加工Bean过程的BeanPostProcessor拦截器, 并且丰富了BeanPostProcessor的拦截.

- postProcessBeforeInstantiation 实例化前置处理 （对象未生成）
- postProcessAfterInstantiation 实例化后置处理 （对象已经生成）
- postProcessPropertyValues 修改属性值。（对象已经生成）

总的来说：

BeanPostProcessor定义的方法是在对象初始化过程中做处理。
InstantiationAwareBeanPostProcessor定义的方法是在对象实例化过程中做处理

spring创建对象会形成**两种执行流程**完成BeanDefinition 创建Bean.

1. postProcessBeforeInstantiation()--自定义对象-->postProcessAfterInitialization();《该模式需要当前Bean实现InstantiationAwareBeanPostProcessor 》
2. postProcessBeforeInstantiation() -->postProcessAfterInstantiation-->postProcessBeforeInitialization()-->postProcessAfterInitialization()《该模式为流水线模式即默认模式》

我们看出:**postProcessBeforeInstantiation一定执行, postProcessAfterInitialization一定执行**.



- 属性填充（Populate）**-> 主要体现在populateBean()方法**
- 初始化（Initialization）**-> 主要体现在initializeBean()方法**
  - Aware相关回调
  - 初始化前置处理
  - 初始化
  - 初始化后置处理
- 销毁（Destruction）





附录：

在开发中可能会有这样的情景。需要在容器启动的时候执行一些内容。比如读取配置文件，数据库连接之类的。SpringBoot给我们提供了两个接口来帮助我们实现这种需求。这两个接口分别为CommandLineRunner和ApplicationRunner。**他们的执行时机为容器启动完成的时候。**