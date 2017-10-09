---
title: 在Hexo发布博客的MarkDown文件中引入JS代码
date: 2017-08-7 15:05:05
tags: [Hexo,MarkDown]
categories: 技术
---
## 前言 ##
上周困扰了两天的问题终于得到解决，于是就打算写点东西，也当作一次记录。

从题目可以看出，问题就是——**如何在Hexo发布博客的Md文件中引入JS代码**，来实现你想要的特效。
之所以会提出这个问题，是源于一个简单的想法，单纯地想在某一篇博客中引入echarts（一个纯 Javascript 的图表库）特效，实质上就是引入它的JS代码。

那么就详细讲解一下引入echarts来实现特效的过程。
## 下载##
首先，在[echarts下载页面](http://echarts.baidu.com/download.html) 中下载相应的版本，我这里下载的是完整版（echarts.min.js）。
文件下载完成后，将其放入下图所示文件夹当中
![WithYou](http://osuskkx7k.bkt.clouddn.com/js%E5%AD%98%E5%82%A8.PNG)

我的博客使用的是next主题，打开themes文件夹中的next文件夹，再依次打开source、js、src文件夹，就可以看到许多js文件，将echarts.min.js放入即可。
## 使用##
在文件引入后（src就是指向刚刚存入js文件的目录），那么你就可以在你的博客中引用这个js文件来达到特定的效果。
引用的方式很简单，只需一行代码：

`<script type="text/javascript" src="/js/src/echarts.min.js"></script>`

在引用js文件后，那么你只要在md文件中添加相应的js代码片段即可，这里贴出我使用的js代码片段

```
<script>
var bmapChart=echarts.init(document.getElementById("map-wrap"));var data=[{name:"上海",value:299},{name:"厦门",value:245},{name:"丰城",value:120},{name:"南昌",value:160},{name:"张家界",value:128},{name:"长沙",value:75},{name:"杭州",value:90},{name:"福州",value:90},{name:"深圳",value:90},{name:"武汉",value:73}];var geoCoordMap={"厦门":[118.105,24.443],"上海":[121.399,31.321],"丰城":[115.801,28.201],"南昌":[115.856,28.691],"张家界":[110.489,29.118],"福州":[119.3,26.08],"长沙":[113,28.21],"杭州":[120.16,30.28],"深圳":[114.06,22.55],"武汉":[114.31,30.52]};var convertData=function(data){var res=[];for(var i=0;i<data.length;i++){var geoCoord=geoCoordMap[data[i].name];if(geoCoord){res.push({name:data[i].name,value:geoCoord.concat(data[i].value)})}}return res};option={title:{text:"我们的足迹 - Our footprints",subtext:"一步一个脚印，让时光见证",sublink:"#",left:"center"},tooltip:{trigger:"item"},bmap:{center:[106.320439,32.58783],zoom:5,roam:true,mapStyle:{styleJson:[{"featureType":"water","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"land","elementType":"all","stylers":{"color":"#f3f3f3"}},{"featureType":"railway","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"highway","elementType":"all","stylers":{"color":"#fdfdfd"}},{"featureType":"highway","elementType":"labels","stylers":{"visibility":"off"}},{"featureType":"arterial","elementType":"geometry","stylers":{"color":"#fefefe"}},{"featureType":"arterial","elementType":"geometry.fill","stylers":{"color":"#fefefe"}},{"featureType":"poi","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"green","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"subway","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"manmade","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"local","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"arterial","elementType":"labels","stylers":{"visibility":"off"}},{"featureType":"boundary","elementType":"all","stylers":{"color":"#fefefe"}},{"featureType":"building","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"label","elementType":"labels.text.fill","stylers":{"color":"#999999"}}]}},series:[{name:"footmark",type:"scatter",coordinateSystem:"bmap",data:convertData(data),symbolSize:function(val){return val[2]/10},label:{normal:{formatter:"{b}",position:"right",show:false},emphasis:{show:true}},itemStyle:{normal:{color:"#60C0DD"}}},{name:"I miss you",type:"effectScatter",coordinateSystem:"bmap",data:convertData(data.sort(function(a,b){return b.value-a.value}).slice(0,2)),symbolSize:function(val){return val[2]/10},showEffectOn:"render",rippleEffect:{brushType:"stroke"},hoverAnimation:true,label:{normal:{formatter:"{b}",position:"right",show:true}},itemStyle:{normal:{color:"purple",shadowBlur:10,shadowColor:"#333"}},zlevel:1}]};bmapChart.setOption(option);
</script>
```
 文章末尾有博客链接。
 在这个页面中所展现的特效中还引入了其他的js文件，如china.js、api.js、bmap.js等，例如api.js是百度地图的js代码，当然你可以引入任何你想要引用的js文件。  
## 详细##
至于bmap.js文件可以在[github](https://github.com/ecomfe/echarts/tree/master/extension/bmap)下载，下载之后打开echarts-master文件夹，找到extension中的bamp.js,如下图所示:
![WithYou](http://osuskkx7k.bkt.clouddn.com/1.PNG)


ECharts使用参考（其中有china.js的介绍）：

- [ECharts 实现地图散点图（上）](http://efe.baidu.com/blog/echarts-map-tutorial/)
- [ECharts 实现地图散点图（下）](http://efe.baidu.com/blog/echarts-map-tutorial-2/)

这里要注意一个问题，就是引入bmap.js后，地图并不显示，想要使用百度地图，还要去[百度地图开放平台](http://lbsyun.baidu.com/index.php?title=jspopular)申请一个密钥，申请成功后在页面中引入，但是经过多次尝试，直接在md文件中引入并不起作用，如下：

` <script src="http://api.map.baidu.com/api?v=2.0&ak=？（你的密钥）"></script>`

   由此想到在script标签中嵌入了网页链接，md可能不能识别（只是猜测），于是想着将这个链接所指向的js转为文件引入试试。直接点击链接，跳转到如下页面：
   
![WithYou](http://osuskkx7k.bkt.clouddn.com/%E7%99%BE%E5%BA%A61.PNG)

可以看到标记处同样为一个链接，再次从浏览器打开，出现如下：  

![WithYou](http://osuskkx7k.bkt.clouddn.com/%E7%99%BE%E5%BA%A62.PNG)

在你眼前呈现的是全屏的js代码，这就是我们所需要的js文件，即百度地图的js代码，将它全选保存为js文件，这里命名为api.js。
然后在md中引入它即可使用，这样我们就能取得和百度地图类似的效果。

下面贴出完整的文章代码，即md文件：
```
<div id="map-wrap" style="height: 500px;width:800px;"></div>

    <script type="text/javascript" src="/js/src/echarts.min.js"></script>
    <script src="/js/src/china.js"></script>
    <script src="/js/src/api.js"></script>
    <script src="/js/src/bmap.js"></script>
    
<script>
var bmapChart=echarts.init(document.getElementById("map-wrap"));var data=[{name:"上海",value:299},{name:"厦门",value:245},{name:"丰城",value:120},{name:"南昌",value:160},{name:"张家界",value:128},{name:"长沙",value:75},{name:"杭州",value:90},{name:"福州",value:90},{name:"深圳",value:90},{name:"武汉",value:73}];var geoCoordMap={"厦门":[118.105,24.443],"上海":[121.399,31.321],"丰城":[115.801,28.201],"南昌":[115.856,28.691],"张家界":[110.489,29.118],"福州":[119.3,26.08],"长沙":[113,28.21],"杭州":[120.16,30.28],"深圳":[114.06,22.55],"武汉":[114.31,30.52]};var convertData=function(data){var res=[];for(var i=0;i<data.length;i++){var geoCoord=geoCoordMap[data[i].name];if(geoCoord){res.push({name:data[i].name,value:geoCoord.concat(data[i].value)})}}return res};option={title:{text:"我们的足迹 - Our footprints",subtext:"一步一个脚印，让时光见证",sublink:"#",left:"center"},tooltip:{trigger:"item"},bmap:{center:[106.320439,32.58783],zoom:5,roam:true,mapStyle:{styleJson:[{"featureType":"water","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"land","elementType":"all","stylers":{"color":"#f3f3f3"}},{"featureType":"railway","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"highway","elementType":"all","stylers":{"color":"#fdfdfd"}},{"featureType":"highway","elementType":"labels","stylers":{"visibility":"off"}},{"featureType":"arterial","elementType":"geometry","stylers":{"color":"#fefefe"}},{"featureType":"arterial","elementType":"geometry.fill","stylers":{"color":"#fefefe"}},{"featureType":"poi","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"green","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"subway","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"manmade","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"local","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"arterial","elementType":"labels","stylers":{"visibility":"off"}},{"featureType":"boundary","elementType":"all","stylers":{"color":"#fefefe"}},{"featureType":"building","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"label","elementType":"labels.text.fill","stylers":{"color":"#999999"}}]}},series:[{name:"footmark",type:"scatter",coordinateSystem:"bmap",data:convertData(data),symbolSize:function(val){return val[2]/10},label:{normal:{formatter:"{b}",position:"right",show:false},emphasis:{show:true}},itemStyle:{normal:{color:"#60C0DD"}}},{name:"I miss you",type:"effectScatter",coordinateSystem:"bmap",data:convertData(data.sort(function(a,b){return b.value-a.value}).slice(0,2)),symbolSize:function(val){return val[2]/10},showEffectOn:"render",rippleEffect:{brushType:"stroke"},hoverAnimation:true,label:{normal:{formatter:"{b}",position:"right",show:true}},itemStyle:{normal:{color:"purple",shadowBlur:10,shadowColor:"#333"}},zlevel:1}]};bmapChart.setOption(option);
    </script>
```
特效如下图所示：

![WithYou](http://osuskkx7k.bkt.clouddn.com/%E7%89%B9%E6%95%882.PNG)

欢迎访问博客页面查看效果：[Youngforzy](https://youngforzy.github.io/2017/08/07/index/#more)
## 问题##
<font size="4">这里其实有个问题，就是特效虽然展现出来，但是在网页端还是无法实现地图的缩放，而在手机端和ipad中都可以进行地图的缩放，目前这个问题还未能得到解决，待日后解决再补充。</font>
## 总结##
以上就是在MarkDown中插入js代码的过程。