---
title: Java(spring)实现Mysql的定时备份与还原
date: 2017-12-25 19:23:12
tags: [spring定时任务,Mysql备份还原]
categories: 技术
---
### 一、数据库的定时备份

#### 备份命令
Mysql的备份指令：

1. 指定数据库：
```
mysqldump -h localhost -uroot -proot  tuser>d:\user_2017-12-25_15-42-10.sql	
```
tuser：数据库名  
user_2017-12-25_15-42-10.sql：文件名

2. 指定数据库中的多个表：

```
mysqldump -h localhost -uroot -proot --databases tuser --tables t_user t_user2>d:\user_2017-12-25_15-42-two.sql 
```
在 --tables 之后加上所需备份的表名

#### 定时（Spring-Task）
了解了mysql的备份命令，那么如何实现定时呢？  
**这里采用Spring的定时任务来实现，基于注解的方式。**

主要有两点注意：

##### 1. Spring.xml中开启定时任务注解的配置：
```
 <!--开启定时任务注解-->
<task:annotation-driven />
```
注意在头部引入task的标签及描述

```
     xmlns:task="http://www.springframework.org/schema/task"
     http://www.springframework.org/schema/task
     http://www.springframework.org/schema/task/spring-task-4.0.xsd
```
##### 2.在相应的方法中添加注解@Scheduled

```
    @Scheduled(cron="0/5 * *  * * ? ")   //每5秒执行一次
    public void task1(){
        System.out.println("北京时间："+new Date());
    }

```
注意(cron="0/5 * *  * * ? ")  表达式

```
cron="0/5 * *  * * ? "   表示每隔5s执行一次
cron=" * * 0/1 * * ? "   表示每隔1小时执行一次

关于cronExpression的配置可以百度
```

对数据库  tuser  中的两张表 t_user 和 t_user2 进行备份:  
代码如下：

```
 //定时备份方案
    @Scheduled(cron="0/5 * *  * * ? ")   //每5秒执行一次  @Scheduled(cron=" * * 0/1 * * ? ") 每小时一次
    public void back(){
        System.out.println("现在时间是"+new Date());
        Runtime runtime = Runtime.getRuntime();  //获取Runtime实例
        String user = "root";
        String password = "root";
        String database1 = "tuser"; // 需要备份的数据库名
        String table1 = "t_user";
        String table2 = "t_user2";
        Date currentDate = new Date();
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd_HH-mm-ss");
        String sdfDate = sdf.format(currentDate);
        String filepath = "d:\\time_" + sdfDate + ".sql"; // 备份的路径地址
        //执行命令
        String stmt = "mysqldump  -h localhost -u "+user+" -p"+password+" --databases "+database1+" --tables "+table1+" "+table2 +" > "+filepath;   
        System.out.println(stmt);
        try {
            String[] command = { "cmd", "/c", stmt};
            Process process = runtime.exec(command);
            InputStream input = process.getInputStream();
            System.out.println(IOUtils.toString(input, "UTF-8"));
            //若有错误信息则输出
            InputStream errorStream = process.getErrorStream();
            System.out.println(IOUtils.toString(errorStream, "UTF-8"));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
### 二、数据库的还原
#### 还原命令
可以通过两种方式来进行还原操作。
##### 1. mysql 利用sql文件还原数据库
```
mysql -h localhost -uroot -proot tuser< D:\user_2017-12-25_15-42-10.sql
```
##### 2. source 命令  
这也是导入sql文件的方式，登录mysql之后，输入：
```
source d:/game_product2018-01-02_10-41-30.sql
```
注意反斜杠的方向，“source d:\ab.sql” 这样会执行失败。  
**注：在Navicat中无法使用 source 命令**

#### 还原

在代码中采用第一种方式实现还原操作


```
    public void restore() {
        String user = "root";
        String password = "root";
        String database = "tuser"; // 需要备份的数据库名
        System.out.println("现在时间是" + new Date());
        Runtime runtime = Runtime.getRuntime();

        String cmd = "mysql  -h localhost" + " -u " + user + " -p" + password + " " + database;
        System.out.println(cmd);
        try {
            String filePath =  "D:\\user_2017-12-25_15-42-10.sql"; // sql文件路径
            String stmt = cmd + " < " + filePath;
            String[] command = {"cmd", "/c", stmt};
            Process process = runtime.exec(command);
            //若有错误信息则输出
            InputStream errorStream = process.getErrorStream();
            System.out.println(IOUtils.toString(errorStream, "utf-8"));
            //等待操作
            int processComplete = process.waitFor();
            if (processComplete == 0) {
                System.out.println("还原成功.");
            } else {
                throw new RuntimeException("还原数据库失败.");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```
