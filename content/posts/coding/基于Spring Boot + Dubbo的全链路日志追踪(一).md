+++
date = '2019-05-25T18:24:00+08:00'
draft = false
title = '基于Spring Boot + Dubbo的全链路日志追踪(一)'
tags = ['SpringBoot', 'Dubbo', '链路追踪']
+++

# 一、 概要

当前公司后端整体架构为：Spring Boot + Dubbo。由于早期项目进度等原因，对日志这块没有统一的规范，基本上是每个项目自己管自己的日志。这也对后面的问题排查带来了很大的困难，特别是那些需要同时或者多级调用Dubbo的服务场景，排查起来更加的困难。

现在需要实现从请求开始，到请求结束的全程日志跟踪。需求很简单，实现思路也不难，只需要全局添加一个`traceId`即可。

当然只有日志的记录是不够的，还要有日志的统一存储和查询。

# 二、 思路

## 2.1 日志采集与存储

初步选择的方案是：`阿里云*日志服务`。可免落地，直接存储。日志服务支持Appender直接发送。

替代方案：基于`Logback appender` + `消息队列` + `ELK`来实现。不过，这样的话，成本并不一定会比阿里云服务低。

## 2.2 当前项目改造

### 2.2.1 API接口

当前项目返回数据格式: 
```json
{
    "code": 200,
    "data": "Hello world",
    "msg": "ok"
}
```
改造后，所有HTTP API响应体中增加`traceId`字段：
```
{
    "code": 200,
    "data": "Hello world",
    "msg": "ok",
    "traceId": "bd41aed8b2da4895a9d2b43d1ef12595"
}
```

### 2.2.2 Dubbo

对于服务调用方：每次调用服务时，都需要向服务提供方传递`traceId`。

对于服务提供方：每次服务响应时，都需要从服务调用方获取`traceId`,并将该`traceId`传递下去，直至该响应结束。

### 2.2.3 日志配置

需对当前日志格式及配置进行统一。

## 2.3 落地思路

### 2.3.1 API接口

项目内部使用`org.slf4j.MDC`传递`traceId`。

使用拦截器完成`traceId`的设置与清除。请求到来时，生成并设置`traceId`；请求结束时，清除`traceId`。

拦截器中无法修改HTTP响应体。可通过`ControllerAdvice`统一向Response Body中写入`traceId`。

### 2.3.2 Dubbo

使用Dubbo提供的`org.apache.dubbo.rpc.Filter`来完成`traceId`的设置与获取。

# 三、 备注

### Dubbo日志问题

Dubbo服务的调用，并不一定是HTTP Request引起的，所以会存在一些没有`traceId`的调用情况。这块需要单独的处理。

可对`traceId`的命名进行规范。

如：
- `req:xxa:xxx` 表示 前端请求，xxa模块，序号标识xxx
- `tim:xxc:xxx` 表示 定时器触发，xxc模块，序号标识xxx

命名规范需要根据具体业务来编写。示例仅供参考~

并且可对返回的数据进行适当的处理，使其避免暴露关键信息，同时还能方便问题的定位与处理。当然这个度还是需要根据具体业务和需求来把握。
