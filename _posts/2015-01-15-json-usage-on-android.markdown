---
layout: post
title:  "Json 在 Android 中的使用"
date:   2015-01-15 18:00:00
tags:
    - Android
    - Json
    - Gson
---
# Json 的使用
Json 分为两种，Object 跟 Array，从名字上很容易就可以区分出来，一个是 Json 对象而另一个则是 Json 数组，其区别我们可以引用官方的一张[图][Json]来说明。

JsonObject 存储的是一个个键值对，而 JsonArray 则存储的是一个个对象或者基本类型，后者需要遍历取出，JsonObject 则可以很方便的通过 key 取到 value。
	
	JSONObject jsonObject = new JSONObject();
    jsonObject.put("id", 1);
    jsonObject.put("name", "ivanchou");
    jsonObject.put("age", "23");

运行结果：`{"id":1,"age":"23","name":"ivanchou"}`

    JSONArray jsonArray = new JSONArray();
    jsonArray.put("ivanchou");
    jsonArray.put(jsonObject);

运行结果：`["ivanchou",{"id":1,"age":"23","name":"ivanchou"}]`


# Gson 的使用
[Gson][Gson] 是 Google 推出用于解析 Json 的，使用到了 Java 的 Serialization & Deserialization。
	

# SharedPreferences 的使用
	SharedPreferences mSharedPreferences = context.getSharedPreferences(“NAME”, Context.MODE_PRIVATE);
	SharedPreferences.Editor editor = mSharedPreferences.edit();
	editor.put...();
	editor.remove(key);
	editor.commit();
	editor.get...();

Json 跟 SharedPreferences 同是健值对的形式存储数据，同时在开发中也可以结合用于存储轻量级的信息。

[Json]:http://www.json.org/json-zh.html
[Gson]:https://sites.google.com/site/gson/gson-user-guide