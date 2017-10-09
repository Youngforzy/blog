---
title: Jersey—— 一个基于Rest风格的Web Service开发框架
date: 2017-07-10 21:54:27
tags: [Java,Jersey]
categories: 技术
---
一、什么是Jersey
-----------

   Jersey 是一个Java规范（JAX-RS）下的基于Rest风格的Web Service开发框架。
   
   说的直白一点，主要应用于移动项目，用来给移动终端和服务端传递数据。
   
   Rest则是一种目前主流的软件架构风格，它可以通过一套统一的接口为 Web，iOS和Android提供服务。因为有些平台不需要显式的前端，只需要一套提供服务的接口，于是就有了Rest风格的软件架构。

二、Jersey+Spring+Mybatis搭建一个简单的Web Service
-----------------------------------------

#### 1、在Eclipse下创建一个Maven工程
工程目录结构如下图：

![目录](http://img.blog.csdn.net/20170710212939408?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmFieWxvdmVfQmFMZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
   上图中——com.zy包下存放业务代码
              ——resources文件夹下存放资源文件
              ——其它主要有Web.xml和Pom.xml文件
#### 2、pom.xml

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">  
  <modelVersion>4.0.0</modelVersion>  
  <groupId>com.zy</groupId>  
  <artifactId>jersey</artifactId>  
  <packaging>war</packaging>  
  <version>0.0.1-SNAPSHOT</version>  
  <name>jersey Maven Webapp</name>  
  <url>http://maven.apache.org</url>  
    
  <properties>  
        <!-- 指明使用JDK8 -->  
        <java-version>1.8</java-version>  
        <!-- 指明使用utf-8编码 -->  
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>  
        <spring3.version>4.2.6.RELEASE</spring3.version>  
        <jersey.version>2.22.2</jersey.version>  
    </properties>  
      
      
  <dependencies>  
    <dependency>  
      <groupId>junit</groupId>  
      <artifactId>junit</artifactId>  
      <version>3.8.1</version>  
      <scope>test</scope>  
    </dependency>  
    <dependency>  
    <groupId>redis.clients</groupId>  
    <artifactId>jedis</artifactId>  
    <version>2.9.0</version>  
    </dependency>  
    <!-- Jersey依赖 -->  
    <dependency>  
            <groupId>org.glassfish.jersey.containers</groupId>  
            <artifactId>jersey-container-servlet</artifactId>  
            <version>${jersey.version}</version>  
        </dependency>  
  
        <dependency>  
            <groupId>org.springframework</groupId>  
            <artifactId>spring-web</artifactId>  
            <version>${spring3.version}</version>  
            <scope>compile</scope>  
        </dependency>  
  
        <dependency>  
            <groupId>org.glassfish.jersey.ext</groupId>  
            <artifactId>jersey-spring3</artifactId>  
            <version>${jersey.version}</version>  
        </dependency>  
  
        <dependency>  
            <groupId>org.glassfish.jersey.media</groupId>  
            <artifactId>jersey-media-json-jackson</artifactId>  
            <version>${jersey.version}</version>  
        </dependency>  
    <!-- 加入mysql驱动依赖包 -->  
    <dependency>  
            <groupId>mysql</groupId>  
            <artifactId>mysql-connector-java</artifactId>  
            <version>5.1.27</version>  
    </dependency>  
      
    <!-- 引入mybatis -->  
        <dependency>  
            <groupId>org.mybatis</groupId>  
            <artifactId>mybatis-spring</artifactId>  
            <version>1.1.1</version>  
        </dependency>  
        <dependency>  
            <groupId>org.mybatis</groupId>  
            <artifactId>mybatis</artifactId>  
            <version>3.2.8</version>  
        </dependency>  
    <!-- 引入数据源 -->  
        <dependency>  
            <groupId>com.alibaba</groupId>  
            <artifactId>druid</artifactId>  
            <version>1.0.1</version>  
        </dependency>  
        <dependency>  
            <groupId>org.aspectj</groupId>  
            <artifactId>aspectjweaver</artifactId>  
            <version>1.7.4</version>  
        </dependency>  
        <!-- 加入fastjson依赖包 -->  
        <dependency>  
            <groupId>com.alibaba</groupId>  
            <artifactId>fastjson</artifactId>  
            <version>1.1.37</version>  
        </dependency>  
  
        <dependency>  
            <groupId>com.github.pagehelper</groupId>  
            <artifactId>pagehelper</artifactId>  
            <version>3.7.6</version>  
        </dependency>  
        <dependency>  
            <groupId>cglib</groupId>  
            <artifactId>cglib</artifactId>  
            <version>2.2.2</version>  
        </dependency>  
        <dependency>  
            <groupId>commons-io</groupId>  
            <artifactId>commons-io</artifactId>  
            <version>2.4</version>  
        </dependency>  
        <dependency>  
            <groupId>org.glassfish.jersey.ext</groupId>  
            <artifactId>jersey-bean-validation</artifactId>  
            <version>2.22.2</version>  
            <exclusions>  
                <exclusion>  
                    <groupId>org.hibernate</groupId>  
                    <artifactId>hibernate-validator</artifactId>  
                </exclusion>  
            </exclusions>  
        </dependency>  
        <dependency>  
            <groupId>commons-beanutils</groupId>  
            <artifactId>commons-beanutils</artifactId>  
            <version>1.7.0</version>  
        </dependency>  
          
        <dependency>  
            <groupId>org.slf4j</groupId>  
            <artifactId>slf4j-log4j12</artifactId>  
            <version>1.7.5</version>  
        </dependency>  
        <!-- E起充解码包 -->  
        <dependency>  
            <groupId>com.extracme.evready</groupId>  
            <artifactId>decode</artifactId>  
            <version>1.1.6</version>  
        </dependency>  
        <dependency>  
            <groupId>com.extracme</groupId>  
            <artifactId>evreadyHelp</artifactId>  
            <version>0.0.1-SNAPSHOT</version>  
        </dependency>  
        <!-- disconf注解插件 -->  
        <dependency>  
            <groupId>com.baidu.disconf</groupId>  
            <artifactId>disconf-client</artifactId>  
            <version>2.6.36</version>  
        </dependency>  
        <dependency>  
            <groupId>org.apache.httpcomponents</groupId>  
            <artifactId>httpclient</artifactId>  
            <version>4.5.2</version>  
        </dependency>  
  </dependencies>  
   
  <build>  
    <finalName>jersey</finalName>  
        <plugins>  
            <plugin>  
                <groupId>org.apache.maven.plugins</groupId>  
                <artifactId>maven-surefire-plugin</artifactId>  
                <version>2.18.1</version>  
                <configuration>  
                    <skipTests>true</skipTests>  
                </configuration>  
            </plugin>  
            <plugin>  
                <groupId>org.apache.maven.plugins</groupId>  
                <artifactId>maven-compiler-plugin</artifactId>  
                <version>2.3.2</version>  
                <configuration>  
                    <skipTests>true</skipTests>  
                    <source>1.8</source>  
                    <target>1.8</target>  
                </configuration>  
            </plugin>  
        </plugins>  
  </build>  
</project>  
```

以上就是项目所依赖的jar包，其中有一些是项目需要的可以忽略，重点是Jersey和spring以及mybatis的依赖，上面有相关的注释。
#### 3、web.xml
```
<web-app>  
      
    <listener>  
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
    </listener>  
    <listener>  
        <listener-class>org.springframework.web.context.request.RequestContextListener</listener-class>  
    </listener>  
    <context-param>  
        <param-name>contextConfigLocation</param-name>  
        <param-value>classpath:spring.xml</param-value>  
    </context-param>  
    <servlet>  
        <servlet-name>jersey</servlet-name>  
        <servlet-class>org.glassfish.jersey.servlet.ServletContainer</servlet-class>  
        <init-param>  
            <param-name>javax.ws.rs.Application</param-name>  
            <param-value>com.zy.StartApplication</param-value>  
              
        </init-param>  
        <load-on-startup>1</load-on-startup>  
    </servlet>  
    <servlet-mapping>  
        <servlet-name>jersey</servlet-name>  
        <url-pattern>/*</url-pattern>  
    </servlet-mapping>  
</web-app>  
```
以上是web.xml的配置。
——listener定义了Spring框架中的Bean随着Web容器启动而被创建。
——context-param定义了Spring.xml的位置。
——servlet定义了org.glassfish.jersey.servlet.ServletContainer，相当于对客户端的请求（/*）进行了拦截，同时还有一个启动参数，它是Application类的实现，需要我们自己定义，利用它来注册资源，实现如下：

```
public class StartApplication extends ResourceConfig {  
  
    /**  
     * Register JAX-RS application components.  
     */  
    public StartApplication() {  
        //register(AuthRequestFilter.class);  
        packages("com.zy.resource");  
      
    }  
}  
```
#### 4、spring-mybatis.xml和mybatis-config.xml以及spring.xml
Spring-mybatis.xml如下：

```
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"  
    xmlns:c="http://www.springframework.org/schema/c" xmlns:cache="http://www.springframework.org/schema/cache"  
    xmlns:context="http://www.springframework.org/schema/context"  
    xmlns:jdbc="http://www.springframework.org/schema/jdbc" xmlns:jee="http://www.springframework.org/schema/jee"  
    xmlns:lang="http://www.springframework.org/schema/lang" xmlns:mvc="http://www.springframework.org/schema/mvc"  
    xmlns:p="http://www.springframework.org/schema/p" xmlns:task="http://www.springframework.org/schema/task"  
    xmlns:tx="http://www.springframework.org/schema/tx" xmlns:util="http://www.springframework.org/schema/util"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd  
        http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd  
        http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd  
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd  
        http://www.springframework.org/schema/jdbc http://www.springframework.org/schema/jdbc/spring-jdbc.xsd  
        http://www.springframework.org/schema/jee http://www.springframework.org/schema/jee/spring-jee.xsd  
        http://www.springframework.org/schema/lang http://www.springframework.org/schema/lang/spring-lang.xsd  
        http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd  
        http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task.xsd  
        http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd  
        http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd">  
          
      
          
          
    <!-- 配置测试环境数据源 -->  
    <bean name="test-dataSource" class="com.alibaba.druid.pool.DruidDataSource"  
        init-method="init" destroy-method="close">  
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />  
        <property name="url" value="jdbc:mysql://localhost:3306/jersey-test?useUnicode=true&characterEncoding=UTF-8" />  
        <property name="username" value="root" />  
        <property name="password" value="" />  
  
        <!-- 初始化连接大小 -->  
        <property name="initialSize" value="0" />  
        <!-- 连接池最大使用连接数量 -->  
        <property name="maxActive" value="20" />  
        <!-- 连接池最小空闲 -->  
        <property name="minIdle" value="0" />  
        <!-- 获取连接最大等待时间 -->  
        <property name="maxWait" value="60000" />  
  
        <property name="testOnBorrow" value="false" />  
        <property name="testOnReturn" value="false" />  
        <property name="testWhileIdle" value="true" />  
  
        <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->  
        <property name="timeBetweenEvictionRunsMillis" value="60000" />  
        <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->  
        <property name="minEvictableIdleTimeMillis" value="25200000" />  
  
        <!-- 打开removeAbandoned功能 -->  
        <property name="removeAbandoned" value="true" />  
        <!-- 1800秒，也就是30分钟 -->  
        <property name="removeAbandonedTimeout" value="1800" />  
        <!-- 关闭abanded连接时输出错误日志 -->  
        <property name="logAbandoned" value="true" />  
  
        <!-- 监控数据库 -->  
        <!-- <property name="filters" value="mergeStat" /> -->  
        <property name="filters" value="stat" />  
    </bean>  
      
          
          
    <!--根据dataSource和configLocation创建一个sqlSessionFactory -->  
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">  
        <property name="dataSource" ref="test-dataSource" />  
        <property name="configLocation" value="classpath:mybatis-config.xml"></property>  
    </bean>  
    <bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate"  
        scope="prototype">  
        <constructor-arg index="0" ref="sqlSessionFactory" />  
    </bean>  
     <!-- 配置事务管理器 -->  
    <bean name="transactionManager"  
        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">  
        <property name="dataSource" ref="test-dataSource"></property>  
    </bean>  
  
    <!-- 注解方式配置事物 -->  
     <tx:annotation-driven transaction-manager="transactionManager" />   
  
    <bean id="sqlSessionCache" class="com.zy.utils.SqlSessionCache"  
        init-method="initMapper">  
        <!-- 扫描的映射mapper.xml的文件路径 -->  
        <property name="packageSearchPath" value="classpath*:com/zy/*/sql/*.xml"></property>  
        <property name="sqlSessionFactory" ref="sqlSessionFactory"></property>  
    </bean>  
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">  
        <property name="basePackage" value="com.zy.*.mapper" />  
    </bean>  
      
    <bean id="framelnterceptor" class="com.zy.utils.Framelnterceptor" />   
    <aop:aspectj-autoproxy/>     
</beans>  
```
mybatis-config.xml如下：

```
<?xml version="1.0" encoding="UTF-8"?>  
<!DOCTYPE configuration PUBLIC    
    "-//mybatis.org//DTD Config 3.0//EN"    
    "http://mybatis.org/dtd/mybatis-3-config.dtd">    
<configuration>    
<!--     <properties resource="project.properties" /> -->  
    <settings>    
        <setting name="cacheEnabled" value="true" />  
        <setting name="lazyLoadingEnabled" value="true" />  
        <setting name="multipleResultSetsEnabled" value="true" />  
        <setting name="useColumnLabel" value="true" />  
        <setting name="useGeneratedKeys" value="false" />  
        <setting name="autoMappingBehavior" value="PARTIAL" />  
        <setting name="defaultExecutorType" value="SIMPLE" />  
        <setting name="defaultStatementTimeout" value="25" />  
        <setting name="safeRowBoundsEnabled" value="false" />  
        <setting name="mapUnderscoreToCamelCase" value="false" />  
        <setting name="localCacheScope" value="SESSION" />  
        <!-- <setting name="logImpl" value="STDOUT_LOGGING" /> -->  
        <setting name="jdbcTypeForNull" value="OTHER" />  
        <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString" />   
    </settings></configuration> 
```
Spring.xml的配置如下：

```
<?xml version="1.0" encoding="UTF-8"?>  
  
<beans xmlns="http://www.springframework.org/schema/beans"  
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
       xmlns:context="http://www.springframework.org/schema/context"  
       xsi:schemaLocation="http://www.springframework.org/schema/beans  
  http://www.springframework.org/schema/beans/spring-beans.xsd  
  http://www.springframework.org/schema/context  
  http://www.springframework.org/schema/context/spring-context.xsd"  
        >  
    <!-- 自动扫描dao和service包(自动注入) -->  
    <context:component-scan base-package="com.zy.*" />  
    <import resource="classpath:spring-mybatis.xml" />  
</beans> 

```

注意：在spring-mybatis.xml配置文件的底部配置了一个拦截器——
`<bean id="framelnterceptor" class="com.zy.utils.Framelnterceptor" />
`     ——作用是输出请求接口的信息和接口返回的信息，以及获取某些与Token相关的信息。

#### 5、请求过程

```
Resources
package com.zy.resource;

import javax.ws.rs.Consumes;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;


@Path("/helloworld")
public class RestHelloWorld {
	
	@GET
	@Consumes(MediaType.APPLICATION_JSON)
	@Produces("application/json;charset=UTF-8")
	public String sayHelloWorld(){
		return "Hello ZY!!!大苏打";
	}
}

```

————这是一个简单的获取资源，使用GET方式获取，屏幕输出     Hello ZY!!!大苏打 。

资源类是一个简单的 Java 对象 (POJO)，可以实现任何接口，简单、可重用性强。
资源类上的常用注解有：
@Path，标注资源类或者方法的相对路径
@GET，@PUT，@POST，@DELETE，标注方法是HTTP请求的类型。
@Produces，标注返回的MIME媒体类型
@Consumes，标注可接受请求的MIME媒体类型
@PathParam，@QueryParam，@HeaderParam，@CookieParam，@MatrixParam，@FormParam
分别标注方法的参数来自于HTTP请求的不同位置，例如
@PathParam来自于URL的路径，
@QueryParam来自于URL的查询参数，
@HeaderParam来自于HTTP请求的头信息，
@CookieParam来自于HTTP请求的Cookie。
##**总结：以上就是一个简单的Jersey框架搭建过程，其中涉及到的很多东西这里没有详细解释，日后深入理解再详谈。**
