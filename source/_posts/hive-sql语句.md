---
title: hive sql语句
date: 2017-08-22
categories: ['数据库']
tags:
---
需求：查询统计上报的数据（获取满足指定条件的字段的值）。

<!-- more -->

问题：custom的数据类型是map
``` javascript
select custom, count(1) as pv, count(DISTINCT union_id) as uv from xxx where partition_date = '2017-08-21' and page_identifier = 'c_dqihv0si' group by custom
```
以上sql语句的查询结果如下：
``` javascript
custom  pv  uv
{"fromTag":"home","web_appnm":"maoyan_i"} 4 3
{"fromTag":"activity","web_appnm":"maoyan_i"} 13  11
{"fromTag":"maoyantab"} 3 2
{"fromTag":"shop"}  549 43
{"fromTag":"moviehome","web_appnm":"maoyan_i"}  82  53
{"fromTag":"showindex"} 467 333
{"fromTag":"showdeallist"}  57  40
{"analyse_val":"{}"}  12433 2402
{"fromTag":"shop","web_appnm":"maoyan_i"} 6213  2752
{"web_appnm":"maoyan_i"}  18677 11357
{"fromTag":"showindex","web_appnm":"maoyan_i"}  6423  4119
{"fromTag":"maoyantab","web_appnm":"maoyan_i"}  1 1
{"fromTag":"showdeallist","web_appnm":"maoyan_i"} 1096  483
```
可以看到是有数据的。

但是，当sql语句如下时，结果却跟预想的不一致：
``` javascript
select partition_date,
       case when custom['fromtag'] = 'shop' then 'POI商户页'
         when custom['fromtag'] = 'showindex' then '演出频道首页'
         when custom['fromtag'] = 'showdeallist' then '演出团购列表'
         else '未知'
         end,
       count(1) as pv,
       count(DISTINCT union_id) as uv
  from xxx
where partition_date = '2017-08-21'
   and page_identifier =  'c_dqihv0si'
group by partition_date,custom['fromtag']
```
查询得到的结果是：
``` javascript
partition_date  c1  pv  uv
2017-08-21  未知  46018 21281
```
以上结果并不如意，但很容易知道，sql语句写错了~

正确的姿势：
``` javascript
select partition_date,
       case custom['fromTag']
         when 'shop' then 'POI商户页'
         when 'showindex' then '演出频道页'
         when 'showdeallist' then '演出团购列表'
         else '未知'
       end,
     count(1) as pv,
     count(DISTINCT union_id) as uv
 from xxx
where partition_date = '2017-08-21'
  and page_identifier =  'c_dqihv0si'
group by partition_date,
       case custom['fromTag']
         when 'shop' then 'POI商户页'
         when 'showindex' then '演出频道页'
         when 'showdeallist' then '演出团购列表'
         else '未知'
       end
```
查询结果如下：
``` javascript
partition_date  _col1 pv  uv
2017-08-21  演出团购列表  1153  523
2017-08-21  POI商户页  6762  2788
2017-08-21  未知  31213 13829
2017-08-21  演出频道页 6890  4450
```
完美！
原因：hive比较傻（据某位数据大神说），group by的字段必须和获取的字段一致~

尴尬，后来又发现，之前那个查询之所以不符合预期，是因为字段写错了~
``` javascript
select partition_date,
       case when custom['fromTag'] = 'shop' then 'POI商户页'
         when custom['fromTag'] = 'showindex' then '演出频道首页'
         when custom['fromTag'] = 'showdeallist' then '演出团购列表'
         else '未知'
         end,
       count(1) as pv,
       count(DISTINCT union_id) as uv
  from xxx
where partition_date = '2017-08-21'
   and page_identifier =  'c_dqihv0si'
group by partition_date,custom['fromTag']
```
结果：
``` javascript
partition_date  c1  pv  uv
2017-08-21  未知  82  53
2017-08-21  演出团购列表  1153  523
2017-08-21  未知  4 3
2017-08-21  未知  4 3
2017-08-21  未知  13  11
2017-08-21  演出频道首页  6890  4450
2017-08-21  POI商户页  6762  2788
2017-08-21  未知  31110 13759
```
不过，这里依然没有上一个查询结果好~看来group by确实需要跟获取的字段保持一致~





