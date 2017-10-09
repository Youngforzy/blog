---
title: JS特效
date: 2017-08-7 13:15:25
categories: 技术
---

<div id="map-wrap" style="height: 500px;width:800px;"></div>

<script type="text/javascript" src="/js/src/echarts.min.js"></script>
<script src="/js/src/china.js"></script>
<script src="/js/src/api.js"></script>
<script src="/js/src/bmap.js"></script>

<script>
var bmapChart=echarts.init(document.getElementById("map-wrap"));var data=[{name:"上海",value:299},{name:"厦门",value:245},{name:"丰城",value:120},{name:"南昌",value:160},{name:"张家界",value:128},{name:"长沙",value:75},{name:"杭州",value:90},{name:"福州",value:90},{name:"深圳",value:90},{name:"武汉",value:73}];var geoCoordMap={"厦门":[118.105,24.443],"上海":[121.399,31.321],"丰城":[115.801,28.201],"南昌":[115.856,28.691],"张家界":[110.489,29.118],"福州":[119.3,26.08],"长沙":[113,28.21],"杭州":[120.16,30.28],"深圳":[114.06,22.55],"武汉":[114.31,30.52]};var convertData=function(data){var res=[];for(var i=0;i<data.length;i++){var geoCoord=geoCoordMap[data[i].name];if(geoCoord){res.push({name:data[i].name,value:geoCoord.concat(data[i].value)})}}return res};option={title:{text:"我们的足迹 - Our footprints",subtext:"一步一个脚印，让时光见证",sublink:"#",left:"center"},tooltip:{trigger:"item"},bmap:{center:[106.320439,32.58783],zoom:5,roam:true,mapStyle:{styleJson:[{"featureType":"water","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"land","elementType":"all","stylers":{"color":"#f3f3f3"}},{"featureType":"railway","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"highway","elementType":"all","stylers":{"color":"#fdfdfd"}},{"featureType":"highway","elementType":"labels","stylers":{"visibility":"off"}},{"featureType":"arterial","elementType":"geometry","stylers":{"color":"#fefefe"}},{"featureType":"arterial","elementType":"geometry.fill","stylers":{"color":"#fefefe"}},{"featureType":"poi","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"green","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"subway","elementType":"all","stylers":{"visibility":"off"}},{"featureType":"manmade","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"local","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"arterial","elementType":"labels","stylers":{"visibility":"off"}},{"featureType":"boundary","elementType":"all","stylers":{"color":"#fefefe"}},{"featureType":"building","elementType":"all","stylers":{"color":"#d1d1d1"}},{"featureType":"label","elementType":"labels.text.fill","stylers":{"color":"#999999"}}]}},series:[{name:"footmark",type:"scatter",coordinateSystem:"bmap",data:convertData(data),symbolSize:function(val){return val[2]/10},label:{normal:{formatter:"{b}",position:"right",show:false},emphasis:{show:true}},itemStyle:{normal:{color:"#60C0DD"}}},{name:"I miss you",type:"effectScatter",coordinateSystem:"bmap",data:convertData(data.sort(function(a,b){return b.value-a.value}).slice(0,2)),symbolSize:function(val){return val[2]/10},showEffectOn:"render",rippleEffect:{brushType:"stroke"},hoverAnimation:true,label:{normal:{formatter:"{b}",position:"right",show:true}},itemStyle:{normal:{color:"purple",shadowBlur:10,shadowColor:"#333"}},zlevel:1}]};bmapChart.setOption(option);
</script>
