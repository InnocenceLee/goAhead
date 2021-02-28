### BeanFactory和ApplicationContext区别

**BeanFactory：**

是Spring里面最低层的接口，提供了最简单的容器的功能，只提供了实例化对象和拿对象的功能；

 **ApplicationContext：**

应用上下文，继承BeanFactory接口，它是Spring的一各更高级的容器，提供了更多的有用的功能；

1) 国际化（MessageSource）

2) 访问资源，如URL和文件（ResourceLoader）

3) 载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的web层  

4) 消息发送、响应机制（ApplicationEventPublisher）

5) AOP（拦截器）

 **两者装载bean的区别**

 **BeanFactory：**

BeanFactory在启动的时候不会去实例化Bean，中有从容器中拿Bean的时候才会去实例化；

 **ApplicationContext：**

ApplicationContext在启动的时候就把所有的Bean全部实例化了。它还可以为Bean配置lazy-init=true来让Bean延迟实例化； 

### spring解决循环依赖



### @Autowired和@Resource

##### Spring注入的方式及场景

Spring常见的DI方式：构造器注入、Setter注入、字段注入。显然，我们经常使用的方式并不是官方最推荐的。

而上面三种注入方式所适用的场景也是有所区别的：1、构造器注入适用具有强依赖和不变性的依赖；2、Setter注入适用于具有可选性和可变性的依赖注入；3、Field注入，尽量少使用，如果需要则使用@Resource进行替代，以降低耦合性。

##### Field注入的缺点

Field注入的缺点很明显，比如不能像构造器注入那样注入不可变的对象，依赖对外部不可见（构造器和Setter可见，而private的属性不可见），会导致组件与IoC容器（比如Spring）紧密耦合，单元测试也需要使用IoC容器，依赖过多时相对构造器注入不能够明显的看出依赖过多（违反单一职责原则）。

既然Field注入这么多缺点，但为什么大家还是习惯使用呢？主要原因：太方便了，极大的缩减了代码。而且大多数业务并不需要用构造器强绑定，同时换IoC容器的可能性也极低。所以，虽然官方及IDE一直强调和提醒，但貌似并没有阻止程序员的使用。

##### 为什么只对@Autowired警告

最主要的原因是：@Autowired是Spring提供的，是特定IoC提供的特定注解，与框架形成了强绑定，一旦换用其他IoC框架，是无法支持注入的。而@Resource是JSR-250提供的，IoC容器应当去兼容它，即使更换容器，也可以正常工作。

另外可能还跟这两种注解的工作机制有关。默认情况下@Autowired是以类型（ByType）进行匹配的，@Resource是以名字（ByName）进行匹配的。也就是说当容器中存在两个相同类型的Bean时，使用@Autowired注入会报错，而使用@Resource会更精准。当然@Autowired也可以指定名称（还需配合@Qualifier注解）。

##### @Autowired和@Resource功能

就Spring而言，不但支持自定义的@Autowired注解，还支持几个由JSR-250规范定义的注解，分别为@Resource、@PostConstruct以及@PreDestroy。

而@Autowired和@Resource的功能基本一致，@Resource的作用相当于@Autowired，只不过@Autowired默认按byType自动注入，而@Resource默认按byName自动注入。

@Resource有两个核心属性：name和type。Spring将@Resource注解的name属性解析为bean的名字，type属性则解析为bean的类型。默认情况下会通过反射机制使用byName自动注入策略。

@Resource装配场景：

- 1、如果同时指定了name和type，则从Spring上下文中找到唯一匹配的bean进行装配，找不到则抛出异常；
- 2、如果指定了name，则根据名称进行装配，找不到则抛出异常；
- 3、如果指定了type，则根据类型进行装配，找不到或者找到多个，都会抛出异常；
- 4、没有任何指定（默认情况），则采用byName方式进行装配，如果没有匹配到，则回退为一个原始类型进行匹配；