---
layout: post
title:  "API 的设计"
date:   2015-01-05 17:00:00
tags:
    - API Document
    - 项目
---
#0.缘由
团队终于从开始的构想到了要正式搭建这个项目的阶段了，根据讨论的结果，整理出来的 API 文档，在这块也没有什么经验，不过看多了一些开放平台的 API 设计，或多或少也有了些了解，于是就有了这篇文章，也许设计的还是有漏洞，这需要在后面实现阶段渐渐完善吧。设计上参考了众多开放平台如**微博、V2EX**等的设计，同时也参考了阮老师的文章 [RESTful API 设计指南][restful_api]。



#1.登陆
用户登录

###URL
/api/authorize/login.json

* Method: POST

###请求参数

||必选|类型|说明|
---|---|---
username|true|string|用户名
password|true|string|密码


###返回字段说明

返回值字段|字段类型|字段说明|
---|---|---
session_id|int|用户的登录标识，用于后续的其他请求，服务器端需保存一个 session_id 与 user 的映射



#2.最新活动
获取最新活动的时间线

###URL
/api/activities/latest.json

* Method: GET
* Authentication: TRUE

###请求参数

||必选|类型|说明|
---|---|---
count|false|int|一次请求返回活动的数量，默认20
sort_type|false|int|返回活动的排序按照最新发布时间降序或活动开始时间升序
page|false|int|返回结果的页码，默认为1
session_id|true|int|用户的登录标识


###返回字段说明

返回值字段|字段类型|字段说明|
---|---|---
id|int|活动的 id
created_at|string|活动的创建时间
jointed|int|用户是否参与该活动标识，三个状态：无操作(0)、感兴趣(1)、参加(2)
author|object|创建该活动的用户
start_at|string|活动开始时间
title|string|活动的标题
text|string|活动的详细内容
tags|int|活动标签



#3.活动详情
根据活动 ID 获取活动详情

###URL
/api/activities/show.json

* Method: GET
* Authentication: TRUE

###请求参数

||必选|类型|说明|
---|---|---
id|true|int|活动的 id
session_id|true|int|用户的登录标识

###返回字段说明

返回值字段|字段类型|字段说明|
---|---|---
created_at|string|活动的创建时间
jointed|int|用户是否参与该活动标识，三个状态：无操作(0)、感兴趣(1)、参加(2)
author|object|创建该活动的用户
title|string|活动的标题
start_at|string|活动三要素之一：时间
place_at|string|活动三要素之一：地点
text|string|活动三要素之一：详情
thumbnail_pic|string|活动缩略图地址
original_pic|string|活动原始图片地址
tags|int|活动标签



#4.创建活动
用户创建自己的活动

###URL
/api/activities/create.json

* Method: POST
* Authentication: TRUE

###请求参数

||必选|类型|说明|
---|---|---
title|true|string|活动标题
start_at|true|string|活动三要素之一：时间
place_at|true|string|活动三要素之一：地点
text|true|string|活动三要素之一：详情
tags|true|int|活动标签
pic_path|false|string|活动配图
session_id|true|int|用户的登录标识


###返回字段说明

返回值字段|字段类型|字段说明|
---|---|---
|||


#5.对活动操作
用户对活动的操作，包含**参加**和**感兴趣**两种

###URL
/api/activities/change_state.json

* Method: POST
* Authentication: TRUE

###请求参数

||必选|类型|说明|
---|---|---
id|true|int|活动 id
joint|true|int|感兴趣(1)、参加(2)
session_id|true|int|用户的登录标识


###返回字段说明

返回值字段|字段类型|字段说明|
---|---|---
|||



#6.Ps
不了解 css 的悲剧就是，不知道怎么设置边框颜色以至于插入的表格看起来很怪异，当然有个简单的方法是`cmd + a`。

[restful_api]:http://www.ruanyifeng.com/blog/2014/05/restful_api.html
