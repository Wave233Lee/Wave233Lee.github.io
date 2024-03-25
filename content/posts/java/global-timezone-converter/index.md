---
date: '2024-03-25T15:11:24+08:00'
title: '国际时区转换'
summary: "海外部署服务前后端交互时区转换"
tags: ["Java"]
---

## 1，概述

### 1.1，需求场景
海外部署服务，用户界面展示当地时间，后端及数据库采用北京时间GMT+8时区，前端采用浏览器定位当地时区，前后端交互时需对时间进行时区转换。
### 1.2，实现原理
前端增加请求头 ISrmTimeZone，传入当前浏览器获取时区；
后端自定义消息转换器 HttpMessageConverter，修改 Jackson 序列化配置，根据请求头 ISrmTimeZone 设置时区。
## 2，代码实现
### 2.1，自定义 HttpMessageConverter
继承 MappingJackson2HttpMessageConverter，原逻辑基础上修改 objectMapper 部分，从请求头/响应头上获取 ISrmTimeZone 设置时区。
```java
package com.midea.srm.core.timezone;

import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.lang.reflect.Type;
import java.text.SimpleDateFormat;
import java.util.List;
import java.util.Objects;
import java.util.TimeZone;
import com.fasterxml.jackson.core.JsonEncoding;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.util.DefaultIndenter;
import com.fasterxml.jackson.core.util.DefaultPrettyPrinter;
import com.fasterxml.jackson.databind.JavaType;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.ObjectWriter;
import com.fasterxml.jackson.databind.SerializationConfig;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.exc.InvalidDefinitionException;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.http.HttpInputMessage;
import org.springframework.http.HttpOutputMessage;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConversionException;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.http.converter.HttpMessageNotWritableException;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.http.converter.json.MappingJacksonInputMessage;
import org.springframework.http.converter.json.MappingJacksonValue;
import org.springframework.lang.NonNull;
import org.springframework.lang.Nullable;
import org.springframework.util.TypeUtils;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

/**
 * 国际时区消息转换器
 * 请求头/响应头包含 ISrmTimeZone 时生效
 * 根据 ISrmTimeZone 动态设置 objectMapper 时区
 * {@link org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter}
 *
 * @author liwf106
 * @since 2023/12/7
 */
@Slf4j
@SuppressWarnings("all")
public class TimeZoneMessageConverter extends MappingJackson2HttpMessageConverter {
    public static final String TIME_ZONE_HEADER = "ISrmTimeZone";

    public TimeZoneMessageConverter() {
        super();
    }

    @NonNull
    @Override
    protected Object readInternal(@NonNull Class<?> clazz, HttpInputMessage inputMessage) throws IOException,
        HttpMessageNotReadableException {
        List<String> timeZone = inputMessage.getHeaders().get(TIME_ZONE_HEADER);
        ObjectMapper objectMapper = getObjectMapper(timeZone);
        JavaType javaType = getJavaType(clazz, null);
        return readJavaType(objectMapper, javaType, inputMessage);
    }

    @NonNull
    @Override
    public Object read(@NonNull Type type, Class<?> contextClass, HttpInputMessage inputMessage)
        throws IOException, HttpMessageNotReadableException {
        List<String> timeZone = inputMessage.getHeaders().get(TIME_ZONE_HEADER);
        ObjectMapper objectMapper = getObjectMapper(timeZone);
        JavaType javaType = getJavaType(type, contextClass);
        return readJavaType(objectMapper, javaType, inputMessage);
    }

    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        HttpServletResponse response =
            ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getResponse();
        String timezone = response.getHeader(TimeZoneMessageConverter.TIME_ZONE_HEADER);
        return super.canRead(clazz, mediaType) && StringUtils.isNotBlank(timezone);
    }

    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        HttpServletResponse response =
            ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getResponse();
        String timezone = response.getHeader(TimeZoneMessageConverter.TIME_ZONE_HEADER);
        return super.canWrite(clazz, mediaType) && StringUtils.isNotBlank(timezone);
    }

    @Override
    protected void writeInternal(@NonNull Object object, @Nullable Type type, HttpOutputMessage outputMessage)
        throws IOException, HttpMessageNotWritableException {
        List<String> timeZone = outputMessage.getHeaders().get(TIME_ZONE_HEADER);
        ObjectMapper objectMapper = getObjectMapper(timeZone);

        MediaType contentType = outputMessage.getHeaders().getContentType();
        JsonEncoding encoding = getJsonEncoding(contentType);
        JsonGenerator generator = objectMapper.getFactory().createGenerator(outputMessage.getBody(), encoding);
        try {

            Class<?> serializationView = null;
            FilterProvider filters = null;
            Object value = object;
            JavaType javaType = null;
            if (object instanceof MappingJacksonValue) {
                MappingJacksonValue container = (MappingJacksonValue) object;
                value = container.getValue();
                serializationView = container.getSerializationView();
                filters = container.getFilters();
            }
            if (type != null && TypeUtils.isAssignable(type, value.getClass())) {
                javaType = getJavaType(type, null);
            }
            ObjectWriter objectWriter;
            if (serializationView != null) {
                objectWriter = objectMapper.writerWithView(serializationView);
            } else if (filters != null) {
                objectWriter = objectMapper.writer(filters);
            } else {
                objectWriter = objectMapper.writer();
            }
            if (javaType != null && javaType.isContainerType()) {
                objectWriter = objectWriter.forType(javaType);
            }
            SerializationConfig config = objectWriter.getConfig();
            if (contentType != null && contentType.isCompatibleWith(MediaType.TEXT_EVENT_STREAM) &&
                config.isEnabled(SerializationFeature.INDENT_OUTPUT)) {
                DefaultPrettyPrinter prettyPrinter = new DefaultPrettyPrinter();
                prettyPrinter.indentObjectsWith(new DefaultIndenter("  ", "\ndata:"));
                objectWriter = objectWriter.with(prettyPrinter);
            }
            objectWriter.writeValue(generator, value);

            generator.flush();

        } catch (InvalidDefinitionException ex) {
            throw new HttpMessageConversionException("Type definition error: " + ex.getType(), ex);
        } catch (JsonProcessingException ex) {
            throw new HttpMessageNotWritableException("Could not write JSON: " + ex.getOriginalMessage(), ex);
        }

    }

    // 修改 objectMapper 时区
    private ObjectMapper getObjectMapper(List<String> timeZone) {
        ObjectMapper timeZoneObjectMapper = objectMapper.copy();
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        timeZoneObjectMapper.setDateFormat(dateFormat);

        if (Objects.nonNull(timeZone) && !timeZone.isEmpty()) {
            timeZoneObjectMapper.setTimeZone(TimeZone.getTimeZone(timeZone.get(0)));
            log.info("set srm timezone {}", TimeZone.getTimeZone(timeZone.get(0)));
        } else {
            timeZoneObjectMapper.setTimeZone(TimeZone.getTimeZone("Asia/Shanghai"));
        }
        return timeZoneObjectMapper;
    }

    private Object readJavaType(ObjectMapper objectMapper, JavaType javaType, HttpInputMessage inputMessage)
        throws IOException {
        try {
            if (inputMessage instanceof MappingJacksonInputMessage) {
                Class<?> deserializationView = ((MappingJacksonInputMessage) inputMessage).getDeserializationView();
                if (deserializationView != null) {
                    return objectMapper.readerWithView(deserializationView).forType(javaType).
                        readValue(inputMessage.getBody());
                }
            }
            return objectMapper.readValue(inputMessage.getBody(), javaType);
        } catch (InvalidDefinitionException ex) {
            throw new HttpMessageConversionException("Type definition error: " + ex.getType(), ex);
        } catch (JsonProcessingException ex) {
            throw new HttpMessageNotReadableException("JSON parse error: " + ex.getOriginalMessage(), ex);
        }
    }

}

```
### 2.2，自定义 ResponseBodyAdvice
为兼容 mcomponent 的 JsonResponseBodyAdvice，仅修改 supports 方法以适配自定义 MessageConverter。
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package com.midea.srm.core.timezone;

import java.util.Arrays;
import java.util.Iterator;
import java.util.List;
import com.midea.mcomponent.core.config.Settings;
import com.midea.mcomponent.core.response.Response;
import com.midea.mcomponent.core.response.SuccessResponse;
import com.midea.mcomponent.core.response.SuccessResponseData;
import com.midea.mcomponent.core.util.IPUtils;
import com.midea.mcomponent.core.util.TraceContext;
import org.apache.tomcat.util.buf.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.MethodParameter;
import org.springframework.core.annotation.Order;
import org.springframework.http.MediaType;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.util.AntPathMatcher;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

/**
 * {@link com.midea.mcomponent.web.common.advice.JsonResponseBodyAdvice}
 *
 * @author liwf106
 * @since 2023/12/7
 */
@Order(-1)
@ControllerAdvice
@SuppressWarnings("all")
public class TJsonResponseBodyAdvice implements ResponseBodyAdvice<Object> {
    protected final Logger logger = LoggerFactory.getLogger(TJsonResponseBodyAdvice.class);
    private static final List<String> DEFAULT_UN_WARP_PATHS =
        Arrays.asList("/**/configuration/ui", "/**/swagger-resources", "/**/v2/api-docs", "/**/configuration/security");
    @Autowired
    private Settings settings;

    public TJsonResponseBodyAdvice() {
    }

    public Object beforeBodyWrite(Object object, MethodParameter methodParameter, MediaType mediaType, Class clazz,
                                  ServerHttpRequest request, ServerHttpResponse response) {
        ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getResponse()
            .setHeader("Cache-Control", "no-cache,no-store");
        String uri = request.getURI().getPath();
        boolean isRecord = new Boolean(TraceContext.getLocaleWeb());
        if (this.isUnWarpPath(uri)) {
            return object;
        } else {
            if (object instanceof SuccessResponseData) {
                ((SuccessResponseData) object).setProviderSpans(
                    isRecord ? StringUtils.join(TraceContext.getSpans()) : null);
                ((SuccessResponseData) object).setWebIpAddress(IPUtils.getLocalIpAddress());
                ((SuccessResponseData) object).setTraceId(TraceContext.getTraceId());
            }
            if (object instanceof Response) {
                return object;
            } else if (object == null) {
                return SuccessResponse.newInstance();
            } else {
                SuccessResponseData data = SuccessResponseData.newInstance(object);
                data.setProviderSpans(isRecord ? StringUtils.join(TraceContext.getSpans()) : null);
                data.setWebIpAddress(IPUtils.getLocalIpAddress());
                data.setTraceId(TraceContext.getTraceId());
                return data;
            }
        }
    }

    private boolean isUnWarpPath(String path) {
        AntPathMatcher matcher = new AntPathMatcher();
        Iterator var4 = DEFAULT_UN_WARP_PATHS.iterator();
        String pattern;
        while (var4.hasNext()) {
            pattern = (String) var4.next();
            if (org.apache.commons.lang.StringUtils.isNotEmpty(pattern) && matcher.match(pattern, path)) {
                return true;
            }
        }
        if (this.settings.getUnwarpUrls() != null && this.settings.getUnwarpUrls().size() > 0) {
            var4 = this.settings.getUnwarpUrls().iterator();
            while (var4.hasNext()) {
                pattern = (String) var4.next();
                if (org.apache.commons.lang.StringUtils.isNotEmpty(pattern) && matcher.match(pattern, path)) {
                    return true;
                }
            }
        }
        return false;
    }

    public boolean supports(MethodParameter methodParameter, Class clazz) {
        return clazz.equals(TimeZoneMessageConverter.class);
    }
}

```
### 2.3，替换 MappingJackson2HttpMessageConverter
使用自定义 MessageConverter 替换掉项目原来默认处理 json 类型报文的 MappingJackson2HttpMessageConverter。
若不确定项目默认消息转换器可在此方法下断点判断：
```java
org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport#addDefaultHttpMessageConverters
```
```java
package com.midea.srm.core.timezone;

import java.util.List;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * 配置国际时区转换器
 *
 * @author liwf106
 * @since 2023/12/7
 */
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        int index = converters.size();
        for (int i = 0; i < converters.size(); i++) {
            if (converters.get(i) instanceof MappingJackson2HttpMessageConverter) {
                index = i;
                break;
            }
        }
        converters.add(index, new TimeZoneMessageConverter());
    }
}

```
### 2.4，响应头设置 ISrmTimeZone & 特殊传参处理
使用 AOP 拦截需要进行时区转换的接口，将请求头 ISrmTimeZone 设置到响应头中以供自定义 messageConverter 获取时区。
修改 objectMapper 时区仅适用于 Java 对象传参，若使用形如 Map<String, String>的方式传参则无法获取到 Date 类型而无法转换时区，因此可在 AOP 内酌情判断并手动转换时区。
```java
package com.midea.srm.core.timezone;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Map;
import java.util.TimeZone;
import cn.hutool.json.JSONUtil;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

/**
 * 国际时区转换处理
 * 处理 Map 等特殊入参形式
 *
 * @author liwf106
 * @since 2023/12/6
 */
@Slf4j
@Aspect
@Component
@SuppressWarnings("all")
public class TimeZoneConvertAspect {
    // 国内时区
    public static final SimpleDateFormat SH_SDF = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    // 国内时区-日期
    public static final SimpleDateFormat SH_SDF_DATE = new SimpleDateFormat("yyyy-MM-dd");
    // 国际时区
    public static final SimpleDateFormat GLOBAL_SDF = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    // 国际时区-日期
    public static final SimpleDateFormat GLOBAL_SDF_DATE = new SimpleDateFormat("yyyy-MM-dd");

    @Autowired
    private ObjectMapper objectMapper;

    public TimeZoneConvertAspect(ObjectMapper objectMapper) {
        SH_SDF.setTimeZone(TimeZone.getTimeZone("Asia/Shanghai"));
        SH_SDF_DATE.setTimeZone(TimeZone.getTimeZone("Asia/Shanghai"));
        this.objectMapper = objectMapper;
    }

    @Pointcut("@within(org.springframework.web.bind.annotation.RestController) && within(com.midea.srm..*)")
    public void point() {
    }

    @Around("point()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        try {
            HttpServletRequest request =
                ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
            String timeZone = request.getHeader(TimeZoneMessageConverter.TIME_ZONE_HEADER);
            if (StringUtils.isNotBlank(timeZone)) {
                HttpServletResponse response =
                    ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getResponse();
                response.addHeader(TimeZoneMessageConverter.TIME_ZONE_HEADER, timeZone);
                // 根据请求头设置国际时区
                GLOBAL_SDF.setTimeZone(TimeZone.getTimeZone(timeZone));
                GLOBAL_SDF_DATE.setTimeZone(TimeZone.getTimeZone(timeZone));
                // 入参
                Object[] args = joinPoint.getArgs();
                log.info("原始请求参数为{}", args);
                for (Object arg : args) {
                    log.info("参数:{}", JSONUtil.toJsonStr(arg));
                    // 仅处理 Map 形式入参
                    if (arg instanceof Map) {
                        ((Map) arg).forEach((k, v) -> {
                            if (v instanceof String && StringUtils.isNotBlank((String) v) && isDateFieldName((String) k)) {
                                try {
                                    log.info("传参含date字段:[{}],[{}]", k, v);
                                    Date date;
                                    String format;
                                    // 仅年月日
                                    if ((((String) v)).length() == 10) {
                                        date = GLOBAL_SDF_DATE.parse((String) v);
                                        format = SH_SDF_DATE.format(date);
                                    } else {
                                        date = GLOBAL_SDF.parse((String) v);
                                        format = SH_SDF.format(date);
                                    }
                                    ((Map) arg).put(k, format);
                                    log.info("传参含date字段转换结果:[{}],[{}]", k, format);
                                } catch (ParseException e) {
                                    log.error("解析失败", e);
                                }
                            }
                        });
                    }
                }
            }
        } catch (Exception e) {
            log.error("时区转换失败", e);
        }
        return joinPoint.proceed();
    }

    /**
     * 根据字段名猜测是否为日期
     *
     * @param fieldName 字段名
     * @return isDateFieldName
     */
    private boolean isDateFieldName(String fieldName) {
        return fieldName.contains("date") || fieldName.contains("Date");
    }
}

```

## 3，逻辑验证

- 前端验证可修改浏览器位置信息模拟海外场景
- 后端验证手动添加请求头，若不生效可关注所有已注册的 HttpMessageConverter 响应的 contentType 和注册顺序

## 附：注意事项

- 以上方案仅处理前后端交互报文，未处理 excel 导入导出场景
- 以上方案会替换处理 json 的默认 HttpMessageConverter，建议根据实际情况评估影响范围（例如本例中影响了 com.midea.mcomponent.web.common.advice.JsonResponseBodyAdvice）
- 2.4 中特殊传参处理仅考虑了传参为 Map 的场景
