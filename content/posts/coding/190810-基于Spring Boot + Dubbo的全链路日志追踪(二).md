+++
date = '2019-08-10T12:09:00+08:00'
draft = false
title = '基于Spring Boot + Dubbo的全链路日志追踪(二)'
tags = ['SpringBoot', 'Dubbo', '链路追踪']
+++


# 一、概要

紧接上一篇，完成分析之后，就要具体的实现了。

`service-a`: 实现dubbo服务。

`service-b`: 实现web服务，并调用`service-a`实现的服务。

# 二、实现

## 2.1 日志采集及存储

本例直接使用【阿里云·日志服务】进行数据存储和检索，使用[`Aliyun Log Logback Appender`](https://github.com/aliyun/aliyun-log-logback-appender)进行日志收集及上传。

其实就是阿里自己实现了一个Logback Appender。当然我们也可以自己实现，比如上传至自建的ELK中。

## 2.2 项目中traceId生成、传递、销毁

### 2.2.1 traceId生成、销毁

#### 2.2.1.1 客户端请求等触发(外部)

外部类请求触发情况，使用拦截器处理。

请求过来之后，生成`traceId`，并写入`org.slf4j.MDC`。

请求完成之后，将`traceId`从`org.slf4j.MDC`中移除。

```java
package com.example.dubboserviceb.interceptor;

import com.example.dubboserviceb.constants.Constants;
import org.slf4j.MDC;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.UUID;

/**
 * @author lpe234
 * @since 2019/5/25 14:43
 */
public class TraceIdInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        // generate traceId
        String traceId = UUID.randomUUID().toString().replace("-", "");

        // put traceId
        MDC.put(Constants.TRACE_ID, traceId);

        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

        // clear traceId
        MDC.remove(Constants.TRACE_ID);
    }
}
```

#### 2.2.1.2 定时任务等触发(内部)

(略...)

### 2.2.1 traceId传递

#### 2.2.1.1 WEB类传递

简单的接口返回类，增加`traceId`字段。

```java
package com.example.dubboserviceb.utils;

import lombok.Data;

/**
 * @author lpe234
 * @since 2019/5/25 14:55
 */
@Data
public class RestResponse<T> {
    private Integer code;
    private String msg;
    private T data;
    private String traceId;

    public RestResponse() {
    }

    public RestResponse(Integer code, String msg, T data) {
        this.code = code;
        this.msg = msg;
        this.data = data;
    }

    public static <T> RestResponse<T> ok(T data) {
        return new RestResponse<>(200, "ok", data);
    }

    public static <T> RestResponse<T> error(T data) {
        return new RestResponse<>(400, "error", data);
    }
}

```

当请求响应结果生成前，获取当前`org.slf4j.MDC`中的`traceId`，设置到`RestResponse`中。

```java
package com.example.dubboserviceb.advice;

import com.example.dubboserviceb.constants.Constants;
import com.example.dubboserviceb.utils.RestResponse;
import org.slf4j.MDC;
import org.springframework.core.MethodParameter;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

/**
 * @author lpe234
 * @since 2019/5/25 15:03
 */
@ControllerAdvice
public class ResponseModifyAdvice implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(MethodParameter methodParameter, Class<? extends HttpMessageConverter<?>> aClass) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object o, MethodParameter methodParameter, MediaType mediaType, Class<? extends HttpMessageConverter<?>> aClass, ServerHttpRequest serverHttpRequest, ServerHttpResponse serverHttpResponse) {

        // put traceId to response
        ((RestResponse) o).setTraceId(MDC.get(Constants.TRACE_ID));

        return o;
    }
}
```

最终，接口响应数据例如如下：

```json
{
  "code": 200,
  "msg": "ok",
  "data": "Hello apple",
  "traceId": "6c25de3422374d51be58555ae9c380e8"
}
```

#### 2.2.1.2 Dubbo类传递

`traceId`的存储使用`org.apache.dubbo.rpc.RpcContext`(内部使用`InternalThreadLocal`实现)。

借助Dubbo的过滤器来实现，`traceId`在Dubbo服务间的读取、写入和清除。

```java
package com.example.dubboservicea.filter;

import com.example.dubboservicea.constants.Constants;
import org.apache.dubbo.common.extension.Activate;
import org.apache.dubbo.rpc.*;
import org.slf4j.MDC;

/**
 * @author lpe234
 * @since 2019/5/25 15:24
 */
@Activate(group = {org.apache.dubbo.common.Constants.PROVIDER, org.apache.dubbo.common.Constants.CONSUMER})
public class DubboTraceIdFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {

        RpcContext rpcContext = RpcContext.getContext();

        // before
        if (rpcContext.isProviderSide()) {
            // get traceId from dubbo consumer，and set traceId to MDC
            String traceId = rpcContext.getAttachment(Constants.TRACE_ID);
            MDC.put(Constants.TRACE_ID, traceId);
        }

        Result result = invoker.invoke(invocation);

        // after
        if (rpcContext.isProviderSide()) {
            // clear traceId from MDC
            MDC.remove(Constants.TRACE_ID);
        }

        return result;
    }

    @Override
    public Result onResponse(Result result, Invoker<?> invoker, Invocation invocation) {
        return result;
    }
}
```

另，需要在`resources/META-INF/dubbo/`文件夹下，创建`com.alibaba.dubbo.rpc.Filter`文本文件。内容为`dubboTraceIdFilter=com.example.dubboservicea.filter.DubboTraceIdFilter`。

而后，spring-boot配置文件中 配置dubbo的过滤器

```yaml
#dubbo
dubbo:
  scan:
    base-packages: com.example.dubboservicea.provider, com.example.dubboservicea.reference
  protocol:
    name: dubbo
    port: 12101
  registry:
    address: zookeeper://118.190.204.150:20084
  provider:
    filter: dubboTraceIdFilter
  consumer:
    filter: dubboTraceIdFilter
```

# 三、结果

## 3.1 HTTP请求结果返回

![HTTP请求结果返回](https://static.lpe234.xyz/pic/202410251728686.png)


## 3.2 日志服务查询

![日志服务查询](https://static.lpe234.xyz/pic/202410251729247.png)
