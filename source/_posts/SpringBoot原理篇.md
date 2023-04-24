---
title: SpringBoot原理篇
date: 2023-04-24 15:35:36
categories: 后端学习笔记
tags: SpringBoot
description: |
    **_SpringBoot原理_**
---

# 1.自动配置工作流程

​	在学习自动配置的时候，需要对spring容器如何进行bean管理的过程非常熟悉才行，所以这里需要先复习一下有关spring技术中bean加载相关的知识。这里列出的bean的加载方式仅仅应用于后面课程的学习，并不是所有的spring加载bean的方式。

## 1.1. Bean的加载方式

#### 方式一：配置文件+```<bean/>```标签

​		最初级的bean的加载方式其实可以直击spring管控bean的核心思想，就是提供类名，然后spring就可以管理了。所以第一种方式就是给出bean的类名，至于内部嘛就是反射机制加载成class，然后，就没有然后了，拿到了class你就可以搞定一切了。如果这句话听不太懂，请转战java基础高级部分复习一下反射相关知识。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!--xml方式声明自己开发的bean-->
    <bean id="cat" class="Cat"/>
    <bean class="Dog"/>

    <!--xml方式声明第三方开发的bean-->
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"/>
    <bean class="com.alibaba.druid.pool.DruidDataSource"/>
    <bean class="com.alibaba.druid.pool.DruidDataSource"/>
</beans>
```

​	配置了类名(id)，可以通过`getBean("id")`的方式获取Bean对象。如果xml文件中出现同名的`<bean>`段落，这种情况下就会导致Sping无法识别要加载哪一个，报错。因此也可以通过`getBean(Class.class)`方式通过类名获取对象。在不配置类名(id)的情况下，Spring会自动给配置的Bean以编号命名用以区分。

```xml
    <!--xml方式声明自己开发的bean-->
    <bean id="cat" class="com.cqupt.bean.Cat"/>
    <bean class="com.cqupt.bean.Dog"/>
    <bean class="com.cqupt.bean.Dog"/>
    <bean class="com.cqupt.bean.Dog"/>
    <bean class="com.cqupt.bean.Dog"/>
```

```java
//运行ctx.getBean(Dog.class);并打印
cat
com.cqupt.bean.Dog#0
com.cqupt.bean.Dog#1
com.cqupt.bean.Dog#2
com.cqupt.bean.Dog#3
dataSource
```





#### 方式二：配置文件扫描+注解定义bean

​		由于方式一种需要将spring管控的bean全部写在xml文件中，对于程序员来说非常不友好，所以就有了第二种方式。哪一个类要受到spring管控加载成bean，就在这个类的上面加一个注解，还可以顺带起一个bean的名字（id）。这里可以使用的注解有@Component以及三个衍生注解@Service、@Controller、@Repository。

```java
@Component("tom")
public class Cat {
}
```

```java
@Service
public class Mouse {
}
```

​	当然，由于**我们无法在第三方提供的技术源代码中去添加上述4个注解**，因此当你需要加载第三方开发的bean的时候可以使用下列方式定义注解式的bean。@Bean定义在一个方法上方，当前方法的返回值就可以交给spring管控，记得这个方法所在的类一定要定义在@Component修饰的类中，有人会说不是@Configuration吗？建议把spring注解开发相关课程学习一下，就不会有这个疑问了。

```java
@Component
public class DbConfig {
    @Bean
    public DruidDataSource dataSource(){
        DruidDataSource ds = new DruidDataSource();
        return ds;
    }
}
```

​	上面提供的仅仅是bean的声明，spring并没有感知到这些东西，像极了上课积极回答问题的你，手举的非常高，可惜老师都没有往你的方向看上一眼。想让spring感知到这些积极的小伙伴，必须设置spring去检查这些类，看他们是否贴标签，想当积极分子。可以通过下列xml配置设置spring去检查哪些包，发现定了对应注解，就将对应的类纳入spring管控范围，声明成bean。

​	像这样在类名上添加注解就能配置bean也太神奇了，总不能每次起名时Spring都把所有的类库都扫描一遍吧？为了避免Spring在类加载时扫描没有的包，引入了包扫描机制，只扫描程序员指定的包。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
       http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd">
    <!--指定加载bean的位置，component，就是扫描组件文件-->
    <context:component-scan base-package="com.cqupt.bean"/>
</beans>
```

​	尝试运行getBeanDefinitionNames()：

![image-20230424164545410](image-20230424164545410.png)

​	可以看到tom和jerry两个bean都加载上了，此外还多了一些东西，多的这些后面再讨论。但是这里面没有druidDataSource，因为Spring不知道去哪里找，因此配置文件中还要再添加：

```xml
<!--指定加载bean的位置，component，就是扫描组件文件-->
    <context:component-scan base-package="com.cqupt.bean, com.cqupt.config"/>
```

​	再次运行getBeanDefinitionNames()：
![image-20230424165626636](image-20230424165626636.png)	

​	发现已经加载到druid了。（这里也可以对配置类添加@Configuration注解，在企业级开发中专门用来声明配置类。产生的结果并不会有变化，本质与@Component是一样的）

​	方式二声明bean的方式是目前企业中较为常见的bean的声明方式，但是也有缺点。方式一中，通过一个配置文件，你可以查阅当前spring环境中定义了多少个或者说多少种bean，但是方式二没有任何一个地方可以查阅整体信息，只有当程序运行起来才能感知到加载了多少个bean。



#### 方式三：注解方式声明配置类

​		方式二已经完美的简化了bean的声明，以后再也不用写茫茫多的配置信息了。仔细观察xml配置文件，会发现这个文件中只剩了扫描包这句话，于是就有人提出，使用java类替换掉这种固定格式的配置，所以下面这种格式就出现了。严格意义上讲不能算全新的方式，但是由于此种开发形式是企业级开发中的主流形式，所以单独独立出来做成一种方式。嗯……，怎么说呢？方式二和方式三其实差别还是挺大的，番外篇找个时间再聊吧。

​		定义一个类并使用@ComponentScan替代原始xml配置中的包扫描这个动作，其实功能基本相同。为什么说基本，还是有差别的。先卖个关子吧，番外篇再聊。

```java
@ComponentScan({"com.cqupt.bean","com.cqupt.config"})
public class SpringConfig3 {
}
```

​		带上@ComponentScan注解，就已经能代替原applicationContext中除**<context>**之外的所有内容了。在@ComponentScan注解后添加字符串数组，写明包扫描的位置，这个配置类就可以完全代替xml文件了。

​	而是用带注解的配置类代替xml文件这种方法后，Spring上下文的获取就不能使用`ClassPathXmlApplicationContext()`来获取了，这种方式只能使用xml文件获取Spring上下文。要使用`AnnotationConfigApplicationContext()`

```java
public class App3 {
    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(SpringConfig3.class);
        String[] beanDefinitionNames = ctx.getBeanDefinitionNames();
        for (String beanDefinitionName : beanDefinitionNames) {
            System.out.println(beanDefinitionName);
        }
    }
}
```

​	再次运行`getBeanDefinitionNames()`:

![image-20230424173022197](image-20230424173022197.png)

​	说明使用`AnnotationConfigApplicationContext()`加载，里面填入的类会自动加载成Bean。

![image-20230424173644402](image-20230424173644402.png)

##### 使用FactroyBean接口

​		补充一个小知识，<span style="color:red">spring提供了一个接口FactoryBean，也可以用于声明bean，只不过实现了FactoryBean接口的类造出来的对象不是当前类的对象，而是FactoryBean接口泛型指定类型的对象</span>>。如下列，造出来的bean并不是DogFactoryBean，而是Dog。有什么用呢？可以在对象初始化前做一些事情，下例中的注释位置就是让你自己去扩展要做的其他事情的。

```java
public class DogFactoryBean implements FactoryBean<Dog> {
    @Override
    public Dog getObject() throws Exception {
        Dog d = new Dog();
        //.........
        return d;
    }
    @Override
    public Class<?> getObjectType() {
        return Dog.class;
    }
    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

​		有人说，注释中的代码写入Dog的构造方法不就行了吗？干嘛这么费劲转一圈，还写个类，还要实现接口，多麻烦啊。还真不一样，你可以理解为Dog是一个抽象后剥离的特别干净的模型，但是实际使用的时候必须进行一系列的初始化动作。只不过根据情况不同，初始化动作不同而已。如果写入Dog，或许初始化动作A当前并不能满足你的需要，这个时候你就要做一个DogB的方案了。然后，就没有然后了，你就要做两个Dog类。当时使用FactoryBean接口就可以完美解决这个问题。

​		通常实现了FactoryBean接口的类使用@Bean的形式进行加载，当然你也可以使用@Component去声明DogFactoryBean，只要被扫描加载到即可，但是这种格式加载总觉得怪怪的，指向性不是很明确。

```java
@ComponentScan({"com.cqupt.bean","com.cqupt.config"})
public class SpringConfig3 {
    @Bean
    public DogFactoryBean dog(){
        return new DogFactoryBean();
    }
}

```



##### 注解格式导入XML格式配置的bean

​		再补充一个小知识，由于早起开发的系统大部分都是采用xml的形式配置bean，现在的企业级开发基本上不用这种模式了。但是如果你特别幸运，需要基于之前的系统进行二次开发，这就尴尬了。新开发的用注解格式，之前开发的是xml格式。这个时候可不是让你选择用哪种模式的，而是两种要同时使用。spring提供了一个注解可以解决这个问题，@ImportResource，在配置类上直接写上要被融合的xml配置文件名即可，算的上一种兼容性解决方案，没啥实际意义。

```java
@Configuration
@ImportResource("applicationContext1.xml")
public class SpringConfig32 {
}
```



##### proxyBeanMethods属性

​		前面的例子中用到了@Configuration这个注解，当我们使用AnnotationConfigApplicationContext加载配置类的时候，配置类可以不添加这个注解。但是这个注解有一个更加强大的功能，它可以保障配置类中使用方法创建的bean的唯一性。为@Configuration注解设置proxyBeanMethods属性值为true即可，由于此属性默认值为true，所以很少看见明确书写的，除非想放弃此功能。

​		<span style="color:red">说人话就是：如果在当前的配置类中，有一个方法可以得到一个对象，并且这个对象被加载成了Bean，那么"proxyBeanMethods=true"可以保证这个方法只要是在这个类中调用，不管调用多少次，都是从容器中获取的Bean，而不是新创建的</span>

​		<span style="color:blue">注：也不是说在这个类之外就不能保证，但是要求通过getBean方法获取到这个配置类的Bean，然后通过调用配置类Bean的方法获取单例对象</span>

​		如果改为false，则每次调用`SpringConfig33.cat()`方法时，都会创建新的对象，而若是true，则可以在每次窗前对象前拦截创建对象的方法并从Spring容器中获取对应的对象并返回。

```java
@Configuration(proxyBeanMethods = true)
public class SpringConfig33 {
    @Bean
    public Cat cat(){
        return new Cat();
    }
}
```

​		下面通过容器再调用上面的cat方法时，得到的就是同一个对象了。注意，必须使用spring容器对象调用此方法才有保持bean唯一性的特性。此特性在很多底层源码中有应用，前面讲MQ时，也应用了此特性，只不过当前没有解释而已。这里算是填个坑吧。

```java
public class App33 {
    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(SpringConfig33.class);
        String[] names = ctx.getBeanDefinitionNames();
        for (String name : names) {
            System.out.println(name);
        }
        System.out.println("-------------------------");
        SpringConfig33 springConfig33 = ctx.getBean("springConfig33", SpringConfig33.class);
        System.out.println(springConfig33.cat());
        System.out.println(springConfig33.cat());
        System.out.println(springConfig33.cat());
    }
}
```





#### 方式四：使用@Import注解注入bean

​		使用扫描的方式加载bean是企业级开发中常见的bean的加载方式，但是由于扫描的时候不仅可以加载到你要的东西，还有可能加载到各种各样的乱七八糟的东西，万一没有控制好得不偿失了。

​		有人就会奇怪，会有什么问题呢？比如你扫描了com.itheima.service包，后来因为业务需要，又扫描了com.itheima.dao包，你发现com.itheima包下面只有service和dao这两个包，这就简单了，直接扫描com.itheima就行了。但是万万没想到，十天后你加入了一个外部依赖包，里面也有com.itheima包，这下就热闹了，该来的不该来的全来了。

​		所以我们需要一种精准制导的加载方式，使用@Import注解就可以解决你的问题。它可以加载所有的一切，只需要在注解的参数中写上加载的类对应的.class即可。有人就会觉得，还要自己手写，多麻烦，不如扫描好用。对呀，但是他可以指定加载啊，好的命名规范配合@ComponentScan可以解决很多问题，但是@Import注解拥有其重要的应用场景。有没有想过假如你要加载的bean没有使用@Component修饰呢？这下就无解了，而@Import就无需考虑这个问题。

​		<span style="color:red">这种形式可以有效的降低源码与Spring技术的耦合度，在Spring技术底层及诸多框架的整合中大量使用</span>

```java
@Import({Dog.class})
public class SpringConfig4 {
}
```

```java
//被导入的bean无需使用注解声明为bean
public class Dog {
}
```

使用`getBeanDefinitionNames()`查看：

![image-20230424195526344](image-20230424195526344.png)

##### 使用@Import注解注入配置类

​		除了加载bean，还可以使用@Import注解加载配置类。其实本质上是一样的，如果导入的是普通的类，这个类会直接加载为Bean，如果导入的是配置类，那么不仅这个配置类会被加载为Bean，配置类内部对应的声明也会被加载为Bean。**（如果将配置类的注解，如@Configuration删去，那么配置类本身仍然会加载为Bean，因为此时配置类已经与普通类一样了。而配置类内部的声明同样也会被加载。<span style="color:red">说明这种方式导入，配置类不需要注解来声明，但是此时获取的配置类已经不是由代理对象方式创建的bean了，获取的dog也不再是同一个对象</span>）**

```java
@Import(Dog.class,DogFactoryBean.class)
public class SpringConfig4 {
}
```



#### 方式五：编程形式注册bean

​		前面介绍的加载bean的方式都是在容器启动阶段完成bean的加载，下面这种方式就比较特殊了，可以在容器初始化完成后手动加载bean。通过这种方式可以实现编程式控制bean的加载。

```java
public class App5 {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(SpringConfig.class);
        //上下文容器对象已经初始化完毕后，手工加载bean
        ctx.registerBean("tom", Cat.class,0);//第三个参数是可变参数
        ctx.register(Mouse.class);//也可以直接用类的类型注册,此时类的名称是类名小写（前提是这个类没有使用注解声明为bean并且起别名(id="?")）
    }
}
```

​		其实这种方式坑还是挺多的，比如容器中已经有了某种类型的bean，再加载会不会覆盖呢？这都是要思考和关注的问题。新手慎用。

```java
public class App5 {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(SpringConfig.class);
        //上下文容器对象已经初始化完毕后，手工加载bean
        ctx.registerBean("tom", Cat.class,0);
        ctx.registerBean("tom", Cat.class,1);
        ctx.registerBean("tom", Cat.class,2);
        System.out.println(ctx.getBean(Cat.class));
    }
}
```

​		通过尝试每次注册时传入不同的参数，发现最终生效的是最后一次注册。说明手工注册方式类似于Map，只对最后一次修改生效。



#### 方式六：导入实现了ImportSelector接口的类

​		在方式五中，我们感受了bean的加载可以进行编程化的控制，添加if语句就可以实现bean的加载控制了。但是毕竟是在容器初始化后实现bean的加载控制，那是否可以在容器初始化过程中进行控制呢？答案是必须的。实现ImportSelector接口的类可以设置加载的bean的全路径类名，记得一点，只要能编程就能判定，能判定意味着可以控制程序的运行走向，进而控制一切。

​		现在又多了一种控制bean加载的方式，或者说是选择bean的方式。<span style="color:red">可以动态地控制加载bean</span>。

```java
public class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        System.out.println("===========================");
        //获取元数据的类名，发现元数据就是SpringConfig6，也就是@Import(MyImportSelector.class)所在位置
        System.out.println("提示： " + importingClassMetadata.getClassName());
        //判断元数据中是否带有该注解
        System.out.println(importingClassMetadata.hasAnnotation("org.springframework.context.annotation.Configuration"));
        //获取元数据中该注解的属性信息
        System.out.println(importingClassMetadata.getAnnotationAttributes("org.springframework.context.annotation.ComponentScan"));
        System.out.println("===========================");
        return new String[]{"com.cqupt.bean.Dog","com.cqupt.bean.Cat"};
    }
}
```

​		我们可以发现，通过这种方式，可以在容器初始化的过程进行大量的检查与判断。比如检查元数据（SpringConfig配置类）中是否加载了想要的注解？注解中是否带有希望添加的属性？还能根据判断结果决定是否要追加扫描其他的包或者类。

```java
public class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        //各种条件的判定，判定完毕后，决定是否装载指定的bean
        boolean flag = importingClassMetadata.hasAnnotation("org.springframework.context.annotation.Configuration");
        if(flag){//若是有Configuration注解，则追加扫描Mouse，否则追加扫描Dog和Cat
            return new String[]{"com.cqupt.bean.Mouse"};
        }
        return new String[]{"com.cqupt.bean.Dog","com.cqupt.bean.Cat"};
    }
}
```





#### 方式七：导入实现了ImportBeanDefinitionRegistrar接口的类

​		方式六中提供了给定类全路径类名控制bean加载的形式，如果对spring的bean的加载原理比较熟悉的小伙伴知道，其实bean的加载不是一个简简单单的对象，spring中定义了一个叫做BeanDefinition的东西，它才是控制bean初始化加载的核心。BeanDefinition接口中给出了若干种方法，可以控制bean的相关属性。说个最简单的，创建的对象是单例还是非单例，在BeanDefinition中定义了scope属性就可以控制这个。如果你感觉方式六没有给你开放出足够的对bean的控制操作，那么方式七你值得拥有。我们可以通过定义一个类，然后实现ImportBeanDefinitionRegistrar接口的方式定义bean，并且还可以让你对bean的初始化进行更加细粒度的控制，不过对于新手并不是很友好。忽然给你开放了若干个操作，还真不知道如何下手。

​		<span style="color:red">相比于方式六（将注册bean的过程封装了），这个方式将bean的注册方式开放了（甚至可以自定义bean的名称）</span>

```java
public class MyRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        BeanDefinition beanDefinition = 	
            BeanDefinitionBuilder.rootBeanDefinition(BookServiceImpl2.class).getBeanDefinition();
        registry.registerBeanDefinition("bookService",beanDefinition);
    }
}
```

![image-20230424211331554](image-20230424211331554.png)





## 1.2. bean的加载控制

​		前面复习bean的加载时，提出了有关加载控制的方式，其中手工注册bean，ImportSelector接口，ImportBeanDefinitionRegistrar接口，BeanDefinitionRegistryPostProcessor接口都可以控制bean的加载，这一节就来说说这些加载控制。

​		企业级开发中不可能在spring容器中进行bean的饱和式加载的。什么是饱和式加载，就是不管用不用，全部加载。比如jdk中有两万个类，那就加载两万个bean，显然是不合理的，因为你压根就不会使用其中大部分的bean。那合理的加载方式是什么？肯定是必要性加载，就是用什么加载什么。继续思考，加载哪些bean通常受什么影响呢？最容易想的就是你要用什么技术，就加载对应的bean。用什么技术意味着什么？就是加载对应技术的类。所以在spring容器中，通过判定是否加载了某个类来控制某些bean的加载是一种常见操作。下例给出了对应的代码实现，其实思想很简单，先判断一个类的全路径名是否能够成功加载，加载成功说明有这个类，那就干某项具体的工作，否则就干别的工作。

