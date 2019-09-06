---
layout: post
title: 'spring bean study'
date: 2019-7-31
author: likexue
categories: java
photos:
- https://images.unsplash.com/uploads/14110635637836178f553/dcc2ccd9?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=750&q=80
tags: [java springbean]
---

# spring bean

## 通过一个例子简单了解如何配置一个bean

+ 创建一个coffee类

```java
@Data //这里用了lombok插件
public class Coffee {
    private String name;
    private BigDecimal price;
}
```

### 方式一

+ 1.配置bean

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="coffee" class="com.li.springbeandemo.example.Coffee">
        <property name="name" value="latte"/>
        <property name="price" value="12.5"/>
    </bean>
</beans>
```

+ 2.如何使用

```java
@SpringBootApplication
public class SpringBeanDemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBeanDemoApplication.class, args);
		ApplicationContext context = new ClassPathXmlApplicationContext("bean.xml");
		Coffee latte = context.getBean("coffee",Coffee.class);
		System.out.println("咖啡："+latte.getName()+"\n价格："+latte.getPrice());
	}
}
```

我首先获得一个ApplicationCOntext实例对象,这个接口继承了Beanfactory,因为Application是一个接口，并且我是通过xml的方式配置bean的，
所以我用ClassPathXmlApplicationContext这个实现了ApplicationContext的类实例化对象，然后我们就可以通过getBean()方法获取我们所需要的实例对象。
(这里需要提一下，Spring Ioc容器默认情况下，Bean是单例模式)

### 方式二

+ 1.配置bean

```java
@Configuration//表示这是一个配置文件
public class CoffeeConfig {
    @Bean(name = "coffee")//设置bean的名称，默认的话，会将initCoffee作为bean的名字保存到Spring Ioc容器中.
    public Coffee initCoffee(){
        Coffee coffee = new Coffee();
        coffee.setName("mocha");
        coffee.setPrice(new BigDecimal(18.9));
        return coffee;
    }
}
```

+ 2.如何使用

```java
@SpringBootApplication
public class SpringBeanDemoAnnotationApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBeanDemoApplication.class, args);
        ApplicationContext context = new AnnotationConfigApplicationContext(CoffeeConfig.class);
        Coffee mocha = context.getBean(Coffee.class);
        System.out.println(mocha);
    }
}
```

这个方式配置有些不同，在实例化ApplicationContext时也不同，因为使用的时注解的方式，所以使用AnnotationConfigApplicationContext这个类获取上下文实例，
需要传入我们配置好的CoffeeConfig，之后的使用方式和之前的一样。

### 方式三

这个方式Coffee类需要加一个@Component的注解

```java
@Data 
@Component("coffee")//默认形式，将Coffee的第一个字母小写作为bean的名字
public class Coffee {
    private String name;
    private BigDecimal price;
}
```

+ 配置bean

```java
@Configuration//表示这是一个配置文件
@ComponentScan//这里需要注意，如果配置类没有和pojo类在同一包下，要设置包名，如@ComponentScan(basePackages ="com.li.example")，  
//默认情况下只会扫描当前包和其子包下的路径
public class CoffeeConfig {
}
```

下面使用方式和第二种方式一样.

## bean的生命周期

初始化-->依赖注入-->调用感知接口BeanNameAware.setBeanName-->调用感知接口BeanFactoryAware.setBeanFactoryName  
-->调用感知接口ApplicationContextAware.setApplicationContext(注：容器实现ApplicationContext接口才会被调用)  
-->Bean后置处理器调用postProcessBeforeInitialization方法-->自定义初始化方法(加@PostConstruct标注)  
-->调用InitializingBean的afterPropertiesSet-->BeanPostProcessor的postAfterInitialization方法-->生存期  
-->@PreDestroy标注的自定义销毁方法-->调用DisposableBean的destory方法

+ 这里解释一下感知接口
实现感知接口的Bean，自身能感知到容器的存在，容器在操作Bean的过程中，会调用感知接口中的方法。这些接口让bean更易定制。
BeanNameAware：这个接口可以让bean获取自身在容器中的id属性。
BeanFactoryAware：这个接口可以让bean获取BeanFactory实例。
ApplicationContextAware：这个接口可以让Bean获取上下文，在bean中可以使用上下文实例。

+ 查看源码
(后续再写)


## Bean的作用域

作用域类型|使用范围|作用域描述
--|--|--
singleton|所有spring应用|IoC容器只存在单例，为默认值
prototype|所有spring应用|每次从IoC容器创建一个新的实例
application|Spring web应用|Web工程生命周期
request|Spring web应用|Web工程单次请求
session|Spring Web|HTTP会话
globalSession|Spring Web应用|在一个全局的HTTP Session中，一个Bean定义对应一个实例，基本不使用。

### 测试singleton和prototype

+ 测试singleton

```java
@SpringBootApplication
public class SpringBeanDemoAnnotationApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBeanDemoApplication.class, args);
        ApplicationContext context = new AnnotationConfigApplicationContext(CoffeeConfig.class);
        Coffee mocha = context.getBean(Coffee.class);
	Coffee latte = context.getBean(Coffee.class);
        System.out.println(mocha==latte);
    }
}
```

其输出值为true,这个只把方式三的测试方法稍微改了一下，其他没又变，因为单例模式是默认的所以不需要修改。

+ 测试prototype

```java
@Data 
@Component("coffee")//默认形式，将Coffee的第一个字母小写作为bean的名字
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)//新加注解
public class Coffee {
    private String name;
    private BigDecimal price;
}
```

再用上面那段测试代码时就会显示false,这时getBean()获取的对象就是不同的了。
