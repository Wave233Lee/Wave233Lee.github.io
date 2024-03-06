---
date: '2024-03-06T11:04:24+08:00'
title: '异步场景日志链路追踪'
summary: "分布式架构下的日志全链路追踪"
tags: ["Java"]
---

## 1，概述
线上环境排查问题需要对某次请求的所有日志进行链路追踪。logback 的 MDC 提供的日志追踪方案仅支持单个线程内的追踪，这是因为 MDC 是基于 ThreadLocal 实现，无法跨线程传递 traceId。阿里开源的TransmittableThreadLocal 基于 InheritableThreadLocal 扩展，支持线程池跨线程传递变量。借此重写 MDCAdapter 即可实现异步场景下的日志链路追踪。
## 2，MDC 改造
### 2.0，依赖引入
``` html
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.14.2</version>
</dependency>
```
### 2.1，重写 MDCAdapter
MDC 存储线程变量的逻辑位于 MDCAdapter
``` java
package org.slf4j;


import com.alibaba.ttl.TransmittableThreadLocal;
import org.slf4j.spi.MDCAdapter;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

/**
 * 重写logback的LogbackMDCAdapter，用TransmittableThreadLocal替换ThreadLocal，解决异步线程traceId传递
 *
 * @author liwf106
 * @since 2024/1/25
 */
@SuppressWarnings("all")
public class TtlMDCAdapter implements MDCAdapter {

    private final ThreadLocal<Map<String, String>> copyOnInheritThreadLocal = new TransmittableThreadLocal<>();

    private static final int WRITE_OPERATION = 1;
    private static final int MAP_COPY_OPERATION = 2;

    static TtlMDCAdapter mtcMDCAdapter;

    static {
        mtcMDCAdapter = new TtlMDCAdapter();
        // 替换MDC的MDCAdapter
        MDC.mdcAdapter = mtcMDCAdapter;
    }

    public static MDCAdapter getInstance() {
        return mtcMDCAdapter;
    }

    final ThreadLocal<Integer> lastOperation = new ThreadLocal<Integer>();

    private Integer getAndSetLastOperation(int op) {
        Integer lastOp = lastOperation.get();
        lastOperation.set(op);
        return lastOp;
    }

    private boolean wasLastOpReadOrNull(Integer lastOp) {
        return lastOp == null || lastOp.intValue() == MAP_COPY_OPERATION;
    }

    private Map<String, String> duplicateAndInsertNewMap(Map<String, String> oldMap) {
        Map<String, String> newMap = Collections.synchronizedMap(new HashMap<String, String>());
        if (oldMap != null) {
            // we don't want the parent thread modifying oldMap while we are
            // iterating over it
            synchronized (oldMap) {
                newMap.putAll(oldMap);
            }
        }

        copyOnInheritThreadLocal.set(newMap);
        return newMap;
    }

    /**
     * Put a context value (the <code>val</code> parameter) as identified with the
     * <code>key</code> parameter into the current thread's context map. Note that
     * contrary to log4j, the <code>val</code> parameter can be null.
     * <p/>
     * <p/>
     * If the current thread does not have a context map it is created as a side
     * effect of this call.
     *
     * @throws IllegalArgumentException in case the "key" parameter is null
     */
    @Override
    public void put(String key, String val) throws IllegalArgumentException {
        if (key == null) {
            throw new IllegalArgumentException("key cannot be null");
        }

        Map<String, String> oldMap = copyOnInheritThreadLocal.get();
        Integer lastOp = getAndSetLastOperation(WRITE_OPERATION);

        if (wasLastOpReadOrNull(lastOp) || oldMap == null) {
            Map<String, String> newMap = duplicateAndInsertNewMap(oldMap);
            newMap.put(key, val);
        } else {
            oldMap.put(key, val);
        }
    }

    /**
     * Remove the the context identified by the <code>key</code> parameter.
     * <p/>
     */
    @Override
    public void remove(String key) {
        if (key == null) {
            return;
        }
        Map<String, String> oldMap = copyOnInheritThreadLocal.get();
        if (oldMap == null)
            return;

        Integer lastOp = getAndSetLastOperation(WRITE_OPERATION);

        if (wasLastOpReadOrNull(lastOp)) {
            Map<String, String> newMap = duplicateAndInsertNewMap(oldMap);
            newMap.remove(key);
        } else {
            oldMap.remove(key);
        }
    }

    /**
     * Clear all entries in the MDC.
     */
    @Override
    public void clear() {
        lastOperation.set(WRITE_OPERATION);
        copyOnInheritThreadLocal.remove();
    }

    /**
     * Get the context identified by the <code>key</code> parameter.
     * <p/>
     */
    @Override
    public String get(String key) {
        final Map<String, String> map = copyOnInheritThreadLocal.get();
        if ((map != null) && (key != null)) {
            return map.get(key);
        } else {
            return null;
        }
    }

    /**
     * Get the current thread's MDC as a map. This method is intended to be used
     * internally.
     */
    public Map<String, String> getPropertyMap() {
        lastOperation.set(MAP_COPY_OPERATION);
        return copyOnInheritThreadLocal.get();
    }

    /**
     * Returns the keys in the MDC as a {@link Set}. The returned value can be
     * null.
     */
    public Set<String> getKeys() {
        Map<String, String> map = getPropertyMap();

        if (map != null) {
            return map.keySet();
        } else {
            return null;
        }
    }

    /**
     * Return a copy of the current thread's context map. Returned value may be
     * null.
     */
    @Override
    public Map<String, String> getCopyOfContextMap() {
        Map<String, String> hashMap = copyOnInheritThreadLocal.get();
        if (hashMap == null) {
            return null;
        } else {
            return new HashMap<>(hashMap);
        }
    }

    @Override
    public void setContextMap(Map<String, String> contextMap) {
        lastOperation.set(WRITE_OPERATION);

        Map<String, String> newMap = Collections.synchronizedMap(new HashMap<String, String>());
        newMap.putAll(contextMap);

        // the newMap replaces the old one for serialisation's sake
        copyOnInheritThreadLocal.set(newMap);
    }
}
```
### 2.2，加载TtlMDCAdapter
定义初始化方法
``` java
package com.midea.escm.core.config;

import org.slf4j.TtlMDCAdapter;
import org.springframework.context.ApplicationContextInitializer;
import org.springframework.context.ConfigurableApplicationContext;

/**
 * 初始化自定义MDCAdapter
 *
 * @author liwf106
 * @since 2024/1/25
 */
@SuppressWarnings("all")
public class TtlMDCAdapterInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(ConfigurableApplicationContext configurableApplicationContext) {
        // 加载自定义的MDCAdapter
        TtlMDCAdapter.getInstance();
    }
}
```
启动类里增加 initializer _（ApplicationContextInitializer另有两种使用方法，在此不赘述）_
``` java
public static void main(String[] args) {
    SpringApplication springApplication = new SpringApplication(VbpApplication.class);

    springApplication.addInitializers(new TtlMDCAdapterInitializer());

    springApplication.run(args);
}
```
## 3，MDC 使用
### 3.1 添加 traceId
在web请求拦截器用调用 MDC.put 添加traceId
``` java
public void addInterceptors(InterceptorRegistry registry) {

    registry.addWebRequestInterceptor(new WebRequestInterceptor() {

        @Override
        public void preHandle(WebRequest request) throws Exception {
            String cookie = request.getHeader("Cookie");
            TokenUtil.setToken(cookie);
            MDC.put("traceId", TraceContext.getTraceId());
        }

        @Override
        public void postHandle(WebRequest request, ModelMap model) throws Exception {
        }

        @Override
        public void afterCompletion(WebRequest request, Exception ex) throws Exception {
            TokenUtil.removeToken();
        }
    });
}
```
### 3.2 日志打印
在 logback-spring.xml 的输出格式中使用 %X{traceId} 获取 traceId 的值
``` html
<property name="log.colorPattern"
          value="%date{yyyy-MM-dd HH:mm:ss} | %cyan(%-5level) | %yellow(%thread) | %red([TraceId:%X{traceId}] | %magenta(%file:%line) | %green(%logger{96}) | (%msg%n%n)"/>
```

附：
[https://github.com/alibaba/transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local)
