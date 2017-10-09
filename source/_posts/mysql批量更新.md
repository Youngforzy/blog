---
title: Mysql批量操作之更新及插入
date: 2017-07-12 15:12:12
tags: [Mysql,Mybatis]
categories: 技术
---
## 前言
   这个问题困扰了整整一天。
   当遇到多条记录需要插入或者更新的时候，往往会使用批量操作来提高效率，提高性能。然而在使用过程中确是出现了各种问题，真的是有些坑只有趟过才知道！！
   好了，话不多说，进入正题。
   注：数据库Mysql     持久层框架 Mybatis

## 批量更新
需求如下代码：		
```
Map<String,Object> ucmap = new HashMap<String, Object>();
		ucmap.put("updateUser","zyong");
		ucmap.put("updateTime", time);
		ucmap.put("list", updateClist);
		connectorInfoMapper.updateBatch(ucmap);
```
这里的需求是对设备的信息进行批量更新，利用Map来传参，Map中包含了一个List，这个List包含了需要更新的对象集合，也就是多条记录。
sql片段如下：
```
<update id="updateBatch" parameterType="java.util.Map">
      <foreach collection="list" item="item" index="index" separator=";">
 update connector_info
    set  
      Connector_Type = #{item.connectorType,jdbcType=INTEGER},
      Voltage_Upper_Limits = #{item.voltageUpperLimits,jdbcType=INTEGER},
      Voltage_Lower_Limits = #{item.voltageLowerLimits,jdbcType=INTEGER},
      Current = #{item.current,jdbcType=INTEGER},
      Power = #{item.power,jdbcType=DOUBLE},
      National_Standard = #{item.nationalStandard,jdbcType=INTEGER},
      Update_User = #{updateUser,jdbcType=VARCHAR},
	  Update_Time = FROM_UNIXTIME(#{updateTime,jdbcType=TIMESTAMP})   
    where Connector_ID = #{item.connectorId,jdbcType=VARCHAR}
    </foreach>
 </update>
```
   可以看到，这里使用了foreach标签进行迭代，item代表着每一个元素，item.power等代表的就是每个元素的属性，而没有加上item的参数如updateUser、updateTime等则是存放在Map中的参数，从需求代码中能看得更加明显。

当我检查了多遍后，感觉没问题之后，运行。
控制台的错误让我明白还是太年轻——报错了！！！仔细一看，是Sql语法错误，What？我把sql放入Navicat中美化，又检查了好几遍，这明明没有错啊！

于是开启了百度，各种查：
 - 有的说把separator=";"中的分号换成——separator="UNION ALL"，测试，还是报同样的错误。
 - 将几个属于Map的字段删除再测试，依然报错  

查了很多并没有什么实质性的进展，将多条数据改成一条数据进行更新，测试，竟然通过了！——原因很明显：批量操作的原因导致。
再百度，终于找到了答案：
并不是Sql的原因，而是数据库设置的原因————Mysql需要打开批量更新的设置。
<font size=5>在数据库JDBC链接中加入： **&allowMultiQueries=true**</font>
如下： 
jdbc.url=jdbc:mysql://139.224.35.81:3306/evshare?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true

<font size=5>在添加了这个“开关”之后，成功批量更新多条数据。</font>

## 批量插入

相比于批量更新，批量插入则没有类似的设置。
一个简单的示例Sql：
```
<insert id="insertBatch" parameterType="java.util.Map">
  insert into connector_info (Equipment_Seq, Connector_ID, 
      Connector_Type, Voltage_Upper_Limits, 
      Voltage_Lower_Limits, Current, Power, 
      National_Standard,Create_User,Create_Time,Update_Time)
      values
      <foreach collection="list" item="item" index="index" separator=",">
     (#{item.equipmentSeq,jdbcType=BIGINT}, #{item.connectorId,jdbcType=VARCHAR}, 
      #{item.connectorType,jdbcType=INTEGER}, #{item.voltageUpperLimits,jdbcType=INTEGER}, 
      #{item.voltageLowerLimits,jdbcType=INTEGER}, #{item.current,jdbcType=INTEGER}, #{item.power,jdbcType=DOUBLE}, 
      #{item.nationalStandard,jdbcType=INTEGER},
      #{createUser,jdbcType=VARCHAR},
      FROM_UNIXTIME(#{createTime,jdbcType=TIMESTAMP}),
      FROM_UNIXTIME(#{updateTime,jdbcType=TIMESTAMP}))
      </foreach>
  </insert>
```
## 总结： 
 1. 有时候问题并不是出现在代码上，可以往系统环境和配置方面考虑。  
 2. 对于批量更新Mysql需要设置，而Oracle则不需要设置，但sql可能要变化
