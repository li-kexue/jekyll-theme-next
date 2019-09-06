---
layout: post
title: 'spring boot 数据源配置'
date: 2019-08-20
author: likexue
categories: java
photos:
- https://images.unsplash.com/photo-1474291102916-622af5ff18bb?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=750&q=80
tags: [springboot]
---

#spring boot默认连接池
> spring boot2.x版本默认的连接池是HikariCP,官网[链接](https://github.com/brettwooldridge/HikariCP),里面有许多配置详情，这里我只做一些简单配置。
## application.properties配置

```properties
#这里注释掉是因为spring boot会尽可能判断驱动类型，所以当他不能确定驱动类型的时候在明确配置
#spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
#这里需要注意，mysql数据库版本如果在8或者之上的要加上serverTimezone这个配置，不然会报错，这里主要是时区问题
spring.datasource.url=jdbc:mysql://localhost:3306/test?useSSL=false&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
spring.datasource.username=******
spring.datasource.password=******
#设置客户端在等待池中等待的最大时间数，当超过这个时间就会报   
#SQLException异常,最小值为250毫秒，默认3000毫秒
spring.datasource.hikari.connection-timeout=3000
#设置连接空闲多长时间进行删除，但必须在minimumIdle小于maximum-pool-size时才有效，
#且当现连接数大于minimumIdle和空闲时间超出下面设置的这个时间，才会被删除
spring.datasource.hikari.idle-timeout=60000
#设置最大连接数
spring.datasource.hikari.maximum-pool-size=100
```

当然也可以通过：spring.datasource.type设置其他连接池的类型，但是要加入相应的依赖。
出于好奇，为什么我写了这些配置信息，为什么可以直接使用，然后网上搜了也没搜到，然后想了想应该是spring boot的自动配置，然后看了看源码，然后进行调试，发现真的是这样。
+ 首先我找到DataSourceProperties类，里面有很多信息，挑出一些

```java
@ConfigurationProperties(
    prefix = "spring.datasource"
)
public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean {
    private String driverClassName;
    private String url;
    private String username;
    private String password;
......
```

这里可以看出我们设置的spring.datasource.username、
spring.datasource.password等信息被上面的那些值给获取了。
但是我们配置的是hikari连接池啊，有点不对劲，然后找到DataSourceConfigration这了类

```java
 @Configuration
    @ConditionalOnClass({HikariDataSource.class})
    @ConditionalOnMissingBean({DataSource.class})
    @ConditionalOnProperty(
        name = {"spring.datasource.type"},
        havingValue = "com.zaxxer.hikari.HikariDataSource",
        matchIfMissing = true
    )
    static class Hikari {
        Hikari() {
        }

        @Bean
        @ConfigurationProperties(
            prefix = "spring.datasource.hikari"
        )
        public HikariDataSource dataSource(DataSourceProperties properties) {
            HikariDataSource dataSource = (HikariDataSource)DataSourceConfiguration.createDataSource(properties, HikariDataSource.class);
            if (StringUtils.hasText(properties.getName())) {
                dataSource.setPoolName(properties.getName());
            }

            return dataSource;
        }
    }
```

里面还有Dbcp2、Generic等连接池的配置，但我们设置的是Hikari，所以匹配到的应该是它，其中还有很多细节方面的东西，不过主要的是这两部分。 其中也有很多注解，之后再一一解决这些吧。

