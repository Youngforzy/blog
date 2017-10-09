---
title: 使用Redis的管道（Pipeline）进行批量操作
date: 2017-07-28 14:55:55 
tags: [Redis,Pipeline,批量，缓存]
categories: 技术
---
# Redis管道技术简介

----------

Reids是一个cs模式的Tcp服务，类似于http的请求。 当客户端发送一个请求时，服务器处理之后会将结果通过响应报文返回给客户端 。  

那么当需要发送多个请求时，难道每次都要等待请求响应，再发送下一个请求吗？
  
当然不是，这里就可以采用Redis的管道技术。  

举个例子，如果说jedis是：request response，request response，...；  

&emsp;&emsp;&emsp;&emsp;&emsp;那么pipeline则是：request request... response response的方式。  

下面，就简单测试一下使用管道的效果。

----------

# 单条插入与批量插入
这里采用逐条和批量的方式往Redis中写入一些数据。  
先从Mysql中查出需要的数据，这里大概是300条左右，数据量并不大，但是简单做个测试应该没问题。  
**单条插入—— Jedis：**

        Jedis jedis = jedisPool.getResource();
		long start = System.currentTimeMillis();
		List<VehicleInfo> vehicleInfos  = vehicleInfoMapper.selectByParam(param);
		for (VehicleInfo vehicleInfo : vehicleInfos) {		
			//遍历每个vehicleInfo
			TVehicleRealReportMsg real = new TVehicleRealReportMsg();
			Map<String, String> keysmap = new HashMap<String, String>();
			keysmap.put("vehicleStatus", real.getVehicleStatus() + "");
			keysmap.put("chargeStatus", real.getChargeStatus() + "");
			keysmap.put("longitude", "9");
			keysmap.put("latitude", "9");
			List<Long> list1 = new ArrayList<Long>();
			Long l = 1000L;
			Long l2 = 22222L;
			list1.add(l);
			list1.add(l2);
			real.setEngineFaultsList(list1);
			keysmap.put("engineFaultsList", JSON.toJSONString(list1));
			//单条插入
			jedis.hmset(vehicleInfo.getVehicleSeq()+"", keysmap);
		}	
		jedis.close();  		
		long end = System.currentTimeMillis(); System.out.println("耗时："+(end-start) +"ms");  
结果：467ms  

![1](http://osuskkx7k.bkt.clouddn.com/%E5%8D%95%E6%9D%A1%E6%8F%92%E5%85%A5.PNG)  

**批量插入—— Pipeline：** 
   
        Jedis jedis = jedisPool.getResource();
		Pipeline pip = jedis.pipelined();
		long start = System.currentTimeMillis();
		List<VehicleInfo> vehicleInfos  = vehicleInfoMapper.selectByParam(param);
		for (VehicleInfo vehicleInfo : vehicleInfos) {		
			//遍历每个vehicleInfo
			TVehicleRealReportMsg real = new TVehicleRealReportMsg();
			Map<String, String> keysmap = new HashMap<String, String>();
			keysmap.put("vehicleStatus", real.getVehicleStatus() + "");
			keysmap.put("chargeStatus", real.getChargeStatus() + "");
			keysmap.put("longitude", "9");
			keysmap.put("latitude", "9");
			List<Long> list1 = new ArrayList<Long>();
			Long l = 1000L;
			Long l2 = 22222L;
			list1.add(l);
			list1.add(l2);
			real.setEngineFaultsList(list1);
			keysmap.put("engineFaultsList", JSON.toJSONString(list1));
			//批量插入
			pip.hmset(vehicleInfo.getVehicleSeq()+"", keysmap);
		}
		pip.sync();//同步
		jedis.close();		
		long end = System.currentTimeMillis();
		System.out.println("耗时："+(end-start) +"ms");  


结果：175ms  

![2](http://osuskkx7k.bkt.clouddn.com/%E6%89%B9%E9%87%8F%E6%8F%92%E5%85%A5.PNG)  
  
可以看到使用管道之后的时间为，相比于单条插入的总时间大大减少，性能更优。

----------

# 单条读取和批量读取

**单条读取—— Jedis：**  

        Jedis jedis = jedisPool.getResource();
		long start = System.currentTimeMillis();
		//1.采用redis单条读取
		List<VehicleInfo> vehicleInfos = vehicleInfoMapper.selectByParam(param);
		List<Coordinate> list = new ArrayList<Coordinate>();
		for(VehicleInfo key: vehicleInfos){
			String hashkey = key.getVehicleSeq()+"";
			if(jedis.exists(hashkey+"")){							
			Coordinate coord = new Coordinate();
			coord.setVehicleSeq(key.getVehicleSeq());
			coord.setOrgId(key.getOrgId());	
			coord.setVehiclemodelseq(key.getVehiclemodelseq());	
			coord.setVin(jedis.hget(hashkey, "vin"));
			coord.setLongitude(Long.valueOf(jedis.hget(hashkey, "longitude")));
			coord.setLatitude(Long.valueOf(jedis.hget(hashkey, "latitude"))); 		
			list.add(coord);
			}
		}	
		jedis.close();
		long end = System.currentTimeMillis();
		System.out.println("耗时："+(end-start)+" ms");
		return list;  
结果： 第一次为1032ms，之后稳定在800~900ms  
 ![3](http://osuskkx7k.bkt.clouddn.com/%E5%8D%95%E4%BD%93%E8%AF%BB.PNG)  

**批量读取—— Pipeline**： 

        Jedis jedis = jedisPool.getResource();
		Pipeline pip = jedis.pipelined();
		long start = System.currentTimeMillis();
        //2.采用redis管道读取
		List<VehicleInfo> vehicleInfos = vehicleInfoMapper.selectByParam(param);
		List<Coordinate> list = new ArrayList<Coordinate>();
		Map<String,Object> map = new HashMap<String, Object>();//map用来暂存属性
		Map<String,List<Response<String>>> responses  = new HashMap<String, List<Response<String>>>(vehicleInfos.size());
		for(VehicleInfo info: vehicleInfos){
			List<Response<String>> resls = new ArrayList<Response<String>>();
			resls.add(pip.hget(info.getVehicleSeq()+"","longitude"));
			resls.add(pip.hget(info.getVehicleSeq()+"","latitude"));
			responses.put(info.getVehicleSeq() + "", resls);//得到了一辆车所有的实时数据--300辆车
			map.put(info.getVehicleSeq()+"orgId", info.getOrgId());
			map.put(info.getVehicleSeq()+"vin", info.getVin());
			map.put(info.getVehicleSeq()+"vehiclemodelseq", info.getVehiclemodelseq());
		}
		pip.sync(); 
		for(String k:responses.keySet()){
			Coordinate coord = new Coordinate();
			coord.setLongitude(Long.valueOf(responses.get(k).get(0).get()));//是get，不是toString
			coord.setLatitude(Long.valueOf(responses.get(k).get(1).get()));
			coord.setVehicleSeq(Long.valueOf(k));
			coord.setOrgId((String) map.get(k+"orgId"));
			coord.setVin((String) map.get(k+"vin"));
			coord.setVehiclemodelseq((Long) map.get(k+"vehiclemodelseq"));
			list.add(coord);
		}
		jedis.close();
		long end = System.currentTimeMillis();
		System.out.println("耗时："+(end-start)+" ms");
		return list;  
结果： 第一次为200ms，之后维持在30ms左右  
 ![4](http://osuskkx7k.bkt.clouddn.com/%E6%89%B9%E9%87%8F%E8%AF%BB.PNG)  

总时间大概是单条读取总时间的1/5甚至更低，可以看出管道大大提升了效率，具有更好的性能。  

<font size="3">注：使用管道所获取的值的类型是Response<String\>,因此需要转为String，如下代码片段： </font> 

    Map<String,List<Response<String>>> responses  = new HashMap<String, List<Response<String>>>  (vehicleInfos.size());  

	//转String
    responses.get(k).get(0).get();  

----------

# 总结


- <font size="3">这里仅仅测试了300条数据的操作，已经取得了相对明显的效果。  
- 对于大量数据的操作，使用Redis管道可以大大提升性能和效率。</font>