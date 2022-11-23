### 一、[循环依赖](https://so.csdn.net/so/search?q=循环依赖&spm=1001.2101.3001.7020)问题全景图



![img](https://img-blog.csdnimg.cn/img_convert/ceef00b5c21677e9c0ec4bfbe400a3c5.png)

 

### 二、什么是循环依赖问题？

**1、什么是循环依赖：**

类与类之间的[依赖关系](https://so.csdn.net/so/search?q=依赖关系&spm=1001.2101.3001.7020)形成了闭环，就会导致循环依赖问题的产生。

比如下图中A类依赖了B类，B类依赖了C类，而最后C类又依赖了A类，这样就形成了循环依赖问题。

![img](https://img-blog.csdnimg.cn/img_convert/7e0dff0f2f7c64f4ae88bb88a657eb93.png)

**2、循环依赖问题案例分析：**

（1）演示代码：

```java
public class ClassA {



	private ClassB classB;



 



	public ClassB getClassB() {



		return classB;



	}



 



	public void setClassB(ClassB classB) {



		this.classB = classB;



	}



}



public class ClassB {



	private ClassA classA;



 



	public ClassA getClassA() {



		return classA;



	}



 



	public void setClassA(ClassA classA) {



		this.classA = classA;



	}



}
```

（2）配置文件：

```XML
<?xml version="1.0" encoding="UTF-8"?>



<beans xmlns="http://www.springframework.org/schema/beans"



	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"



	xsi:schemaLocation="http://www.springframework.org/schema/beans



        http://www.springframework.org/schema/beans/spring-beans.xsd">



 



	<bean id="classA" class="ioc.cd.ClassA">



		<property name="classB" ref="classB"></property>



	</bean>



	<bean id="classB" class="ioc.cd.ClassB">



		<property name="classA" ref="classA"></property>



	</bean>



</beans>
```

（3）测试代码：

```java
	@Test



	public void test() throws Exception {



		// 创建IoC容器，并进行初始化



		String resource = "spring/spring-ioc-circular-dependency.xml";



		ApplicationContext context = new ClassPathXmlApplicationContext(resource);



		// 获取ClassA的实例（此时会发生循环依赖）



		ClassA classA = (ClassA) context.getBean(ClassA.class);



	}
```

**3、通过Spring IOC流程的源码分析循环依赖问题：**

![img](https://img-blog.csdnimg.cn/img_convert/bacbc47f948415cf49e1636644565551.png)

 

### 三、循环依赖问题的类型

循环依赖问题在Spring中主要有三种情况：

- （1）通过构造方法进行依赖注入时产生的循环依赖问题。
- （2）通过setter方法进行依赖注入且是在多例（原型）模式下产生的循环依赖问题。
- （3）通过setter方法进行依赖注入且是在单例模式下产生的循环依赖问题。

在Spring中，只有第（3）种方式的循环依赖问题被解决了，其他两种方式在遇到循环依赖问题时都会产生异常。其实也很好解释：

- 第（1）种构造方法注入的情况下，在new对象的时候就会堵塞住了，其实也就是”先有鸡还是先有蛋“的历史难题。
- 第（2）种setter方法（多例）的情况下，每一次getBean()时，都会产生一个新的Bean，如此反复下去就会有无穷无尽的Bean产生了，最终就会导致OOM问题的出现。

 

### 四、如何解决循环依赖问题？

1、Spring解决的[单例模式](https://so.csdn.net/so/search?q=单例模式&spm=1001.2101.3001.7020)下的setter方法依赖注入引起的循环依赖问题，主要是通过两个缓存来解决的，请看下图：

![img](https://img-blog.csdnimg.cn/img_convert/d6f63135eed5564efe166c7b472d8fab.png)

 

### 五、Spring三大缓存介绍

Spring中有三个缓存，用于存储单例的Bean实例，这三个缓存是彼此互斥的，不会针对同一个Bean的实例同时存储。如果调用getBean，则需要从三个缓存中依次获取指定的Bean实例。 读取顺序依次是一级缓存 ==> 二级缓存 ==> 三级缓存。

![img](https://img-blog.csdnimg.cn/img_convert/d9f570aa0dce404d7da35ecf42e9ddb5.png)

**1、一级缓存：Map<String, Object> singletonObjects：**

（1）第一级缓存的作用：

- 用于存储单例模式下创建的Bean实例（已经创建完毕）。
- 该缓存是对外使用的，指的就是使用Spring框架的程序员。

（2）存储什么数据？

- K：bean的名称
- V：bean的实例对象（有代理对象则指的是代理对象，已经创建完毕）

**2、第二级缓存：Map<String, Object> earlySingletonObjects：**

（1）第二级缓存的作用：

- 用于存储单例模式下创建的Bean实例（该Bean被提前暴露的引用，该Bean还在创建中）。
- 该缓存是对内使用的，指的就是Spring框架内部逻辑使用该缓存。
- 为了解决第一个classA引用最终如何替换为代理对象的问题（如果有代理对象）

**3、第三级缓存：Map<String, ObjectFactory<?>> singletonFactories：**

（1）第三级缓存的作用：

- 通过ObjectFactory对象来存储单例模式下提前暴露的Bean实例的引用（正在创建中）。
- 该缓存是对内使用的，指的就是Spring框架内部逻辑使用该缓存。
- 此缓存是解决循环依赖最大的功臣

（2）存储什么数据？

- K：bean的名称
- V：ObjectFactory，该对象持有提前暴露的bean的引用

![img](https://img-blog.csdnimg.cn/img_convert/449a13de9e3f9e05cd888ac5b27f1ad0.png)

（3）为什么第三级缓存要使用ObjectFactory？

如果仅仅是解决循环依赖问题，使用二级缓存就可以了，但是如果对象实现了AOP，那么注入到其他bean的时候，并不是最终的代理对象，而是原始的。这时就需要通过三级缓存的ObjectFactory才能提前产生最终的需要代理的对象。

![img](https://img-blog.csdnimg.cn/img_convert/5e0670114b787860bd190fab3ad91629.png)

（4）什么时候将Bean的引用提前暴露给第三级缓存的ObjectFactory持有？时机就是在第一步实例化之后，第二步依赖注入之前，完成此操作。

![img](https://img-blog.csdnimg.cn/img_convert/ccbd43c654f7bb2cc592082321566ea3.png)

 

### 六、解决构造函数相互注入造成的循环依赖：

前面说Spring可以自动解决单例模式下通过setter()方法进行依赖注入产生的循环依赖问题。而对于通过[构造方法](https://so.csdn.net/so/search?q=构造方法&spm=1001.2101.3001.7020)进行依赖注入时产生的循环依赖问题没办法自动解决，那针对这种情况，我们可以使用@Lazy注解来解决。

也就是说，对于类A和类B都是通过构造器注入的情况，可以在A或者B的构造函数的形参上加个@Lazy注解实现延迟加载。@Lazy实现原理是，当实例化对象时，如果发现参数或者属性有@Lazy注解修饰，那么就不直接创建所依赖的对象了，而是使用动态代理创建一个代理类。

比如，类A的创建：A a=new A(B)，需要依赖对象B，发现构造函数的形参上有@Lazy注解，那么就不直接创建B了，而是使用动态代理创建了一个代理类B1，此时A跟B就不是相互依赖了，变成了A依赖一个代理类B1，B依赖A。但因为在注入依赖时，类A并没有完全的初始化完，实际上注入的是一个代理对象，只有当他首次被使用的时候才会被完全的初始化。