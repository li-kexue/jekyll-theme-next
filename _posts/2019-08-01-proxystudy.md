---
layout: post
title: 'proxy study'
date: 2019-08-01
author: likexue
photos:
- https://images.unsplash.com/photo-1562102049-03d1bd23c43e?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=750&q=80
categories: java
tags: [java spring]
---

> 想学习一下spring aop，但是看看有动态代理的知识，就先来学学动态代理.

# 代理模式

> 所谓的代理者是指一个类别可以作为其它东西的接口。代理者可以作任何东西的接口：网络连接、存储器中的大对象、文件或其它昂贵或无法复制的资源。--wike

既然学习代理了，就将动态代理和静态代理都学学.

## 静态代理

> 刚开始看静态代理的时候发现，我去，这不就是给原来的类再封装一下吗，然后给他起一个名字，叫静态代理。

+ 通过一个例子来看看静态代理


```java
//首先需要一个代理接口
public interface Animal {
    void eat();
    void play();
}

//然后实现一个目标类
public class Dog implements Animal {
    private String name;
    private String age;

    @Override
    public void eat(){
        System.out.println("狗吃骨头");
    }
    @Override
    public void play(){
        System.out.println("狗玩飞盘");
    }
}

//代理类
public class AnimalProxy implements Animal {

    Animal target;

    public AnimalProxy(Animal animal) {
        this.target = animal;
    }

    @Override
    public void eat() {
        if(target!=null){
            before();
            target.eat();
            after();
        }
    }
    public void before(){
        System.out.println("日志:"+this.getClass().getSimpleName()+"执行了eat()方法");
    }
    public void after(){
        System.out.println("日志:"+this.getClass().getSimpleName()+"的eat方法执行结束");
    }
}

public class ProxyDemo {

    public static void main(String[] args) {
        Animal animal = new Dog();
        AnimalProxy proxy = new AnimalProxy(animal);
        proxy.eat();
    }
}
```

> 代理加强以后显示效果

```
proxy start...
日志:AnimalProxy执行了eat()方法
狗吃骨头
日志:AnimalProxy的eat方法执行结束
```

如果只有几个代理接口，我们使用静态代理还是比较好用的，但是如果数量很多的话，可能就会出现很多重复的代码
