# Spring IOC源码分析

## 前言

​		这是我第一次写spring的源码分析，本人也是小白一个，知道spring源码的重要性，所以花时间，查了很多博客和资料想去了解spring的源码。很多地方都会加上自己的理解，如果理解有问题，我会再修改一次。

​		在我看来，spring的学习还是用配置式的好理解，所以本文就根据配置式的spring来分析。

​		本文对spring源码的分析仅在于方便理解，所以会有很多遗漏的地方，以后对spring理解加深了会再补上。

## spring例子

​		首先来看配置式的启动Spring的例子。

- 首先创建一个maven项目

- 然后在pom.xml文件里加上spring依赖

  ~~~xml
  <dependency>
  	<groupId>org.springframework</groupId>
  	<artifactId>spring-webmvc</artifactId>
  	<version>5.2.0.RELEASE</version>
  </dependency>
  ~~~

  spring-webmvc这一个依赖包括了aop，bean，spring-context等spring的6个依赖。不知道引入哪些依赖的，就引入这一个就行。

- 创建一个实体类，注意加上get set 方法

  ~~~java
  public class Person {
  
      private String name;
  
      private String age;
  
  
      public String getName() {
          return name;
      }
  
      public void setName(String name) {
          this.name = name;
      }
  
      public String getAge() {
          return age;
      }
  
      public void setAge(String age) {
          this.age = age;
      }
  }
  ~~~

- 在resouce文件夹中加入spring的xml文件，设置bean

  ~~~xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.springframework.org/schema/beans
          https://www.springframework.org/schema/beans/spring-beans.xsd">
  
          <bean id="Person1" class="com.zevin.Person" >
              <property name="name" value="人"/>
              <property name="age" value="18"/>
          </bean>
  </beans>
  ~~~

- 写一个测试的main

  ~~~java
  public class Hello {
      public static void main(String[] args) {
          // 创建ClassPathXmlApplicationContext类读取xml文件
          ApplicationContext context = new ClassPathXmlApplicationContext("myBean.xml");
          // 通过 getBean创建得到bean的实例
          Person person = context.getBean("Person1", Person.class);
          System.out.println(person.getName()+"   "+person.getAge());
      }
  }
  // 打印结果
  人   18
  ~~~

以上就是一个spring简单的例子。

## Spring bean

### Bean是什么

​	在spring中，最重要的就是IOC和AOP，而Bean就是IOC中最重要的概念，在spring官方文档中，对Bean的解释如下：

~~~text
In Spring, the objects that form the backbone of your application and that are managed by the Spring IoC container are called beans. A bean is an object that is instantiated, assembled, and otherwise managed by a Spring IoC container.

在 Spring 中，构成应用程序主干并由Spring IOC容器管理的对象称为bean。bean是一个由Spring IoC容器实例化、组装和管理的对象。
~~~

​	可以看出，bean其实就是对象，只不过这个对象的生命周期，由SpringIOC容器来管理。

### bean源码图分析

​	从目录2里spring的例子开始：先看下类关系图，以下从ClassPathXmlApplicationContext开始来对比该类图：

![ClassPathXmlApplicationContext](D:\ClassPathXmlApplicationContext.png)

~~~Java
ApplicationContext context = new ClassPathXmlApplicationContext("myBean.xml");
~~~

（1）这里获取xml的上下文，点进去ClassPathXmlApplicationContext:

~~~Java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
}
~~~

​	继续点击this：

~~~Java
public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
    	// 注意这个方法
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
}
~~~

​	（2）这里有个setConfigLocations方法，点击进去则是调用的AbstractRefreshableConfigApplicationContext里的方法。点击super，进到父类AbstractXmlApplicationContext：

~~~Java
public AbstractXmlApplicationContext(@Nullable ApplicationContext parent) {
		super(parent);
}
~~~

​	（3）点击super，再进到父类AbstractRefreshableConfigApplicationContext：

~~~Java
public abstract class AbstractRefreshableConfigApplicationContext extends AbstractRefreshableApplicationContext
		implements BeanNameAware, InitializingBean {
    
    ......
	public AbstractRefreshableConfigApplicationContext(@Nullable ApplicationContext parent) {
			super(parent);
	}
    ......
        
    public void setConfigLocations(@Nullable String... locations) {
		if (locations != null) {
			Assert.noNullElements(locations, "Config locations must not be null");
			this.configLocations = new String[locations.length];
			for (int i = 0; i < locations.length; i++) {
				this.configLocations[i] = resolvePath(locations[i]).trim();
			}
		}
		else {
			this.configLocations = null;
		}
	}
    ......
}
~~~

​	注意这里的setConfigLocations方法，ClassPathXmlApplicationContext调用的就是该方法。这里从类图上可以看到，该类继承和实现了一些类。先从实现入手，先看[InitializingBean](#InitializingBean)：

~~~Java
public interface InitializingBean {

	/**
	 * Invoked by the containing {@code BeanFactory} after it has set all bean properties
	 * and satisfied {@link BeanFactoryAware}, {@code ApplicationContextAware} etc.
	 * <p>This method allows the bean instance to perform validation of its overall
	 * configuration and final initialization when all bean properties have been set.
	 * @throws Exception in the event of misconfiguration (such as failure to set an
	 * essential property) or if initialization fails for any other reason
	 */
	void afterPropertiesSet() throws Exception;
}
~~~

​	InitializingBean中只有一个afterPropertiesSet()方法。

​	现在回到AbstractRefreshableConfigApplicationContext中，进入到BeanNameAware类中：

~~~Java
public interface BeanNameAware extends Aware {
	
    void setBeanName(String name);

}
~~~

​	BeanNameAware中也只有一个setBeanName方法，该方法的作用是让Bean获取自己在BeanFactory配置中的名字，只要实现该接口，就会在初始化bean之前执行这个方法，让Bean知道它的名字。

​	（4）现在再回到AbstractRefreshableConfigApplicationContext中，点击（3）中的super，进到父类AbstractRefreshableApplicationContext中：

~~~Java
public AbstractRefreshableApplicationContext(@Nullable ApplicationContext parent) {
		super(parent);
}
~~~

​	（5）再点击super，进到父类AbstractApplicationContext中

~~~Java

~~~



​	



## 一些基础类说明

### InitializingBean

​	InitializingBean接口为bean提供了初始化方法的方式，它只包括afterPropertiesSet方法，凡是继承该接口的类，在初始化bean的时候都会执行该方法。简单点说，只要实现了InitializingBean接口，就会在初始化Bean的时候执行afterPropertiesSet方法。下面是个例子：

~~~java
// 构建一个自定义的TestInitializingBean类实现InitializingBean
public class TestInitializingBean implements InitializingBean {
    public void afterPropertiesSet() throws Exception {
        System.out.println("执行afterPropertiesSet");
    }
    public void ourDefineFunc(){
        System.out.println("这是我们自己定义的方法");
    }
}
~~~

~~~XML
<!-- 配置自定义类的xml-->
<bean id="TestInitializingBean" class="com.demo2.TestInitializingBean"/>
~~~

~~~Java
public class InitializingBeansMain {
    public static void main(String[] args) {
        // 测试，当该类初始化的时候
        ApplicationContext context =new  ClassPathXmlApplicationContext(
                "myBean.xml");
    }
}
~~~

~~~text
// 打印结果
执行afterPropertiesSet
~~~

​	可以看出，在初始化bean的时候，就会自动调用该方法。

​	

























































