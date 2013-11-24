spring mvc (3.2) 的REST风格异常处理
===================================

## 1. 需要spring 3.2 以上的版本，使用注解方式
spring 3.2之前提供的异常处理都没有很优雅的处理与rest风格相符错误处理的问题。

但是异常的基本处理过程还是类似，需要@ExceptionHandler，传递要处理的异常（或其父类）

## 2. @ControllerAdvice 注解
在异常处理类上注解上ControllerAdvice，这样spring就可以自动扫描到这个类并把其当做异常处理的bean。

## 3. 继承ResponseEntityExceptionHandler类
这个类提供了绝大多数spring mvc内置的异常处理以及自定义的异常处理，对于自定义的异常处理，只需重载即可。

## 4. 关于spring mvc内置异常只有error code没有相关错误提示解决办法
父类的handleException方法是final的，无法重载，但是查看源代码，发现都是通过比较exception的类型然后调用相关的异常处理方法，传递的response body是null。

所以我们只需要在子类中重载相关异常，传递我们想要的response body即可。

比如，MissingServletRequestParameterException，这个spring mvc内置的异常，是controller的方法中缺少了查询参数所触发的。

如果通过上树的配置，返回客户端的只有400(bad request)，什么提示也没有，非常不友好，所以我们要在子类中重载handleMissingServletRequestParameter()这个方法。

## 5. 完整的源代码
```java
/*
 * The MIT License (MIT)
 * Copyright (c) 2013 newgxu.cn <the original author or authors>.
 * The software shall be used for good, not evil.
 */
package com.github.longkai.web.exception;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.ConversionNotSupportedException;
import org.springframework.beans.TypeMismatchException;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.http.converter.HttpMessageNotWritableException;
import org.springframework.validation.BindException;
import org.springframework.web.HttpMediaTypeNotAcceptableException;
import org.springframework.web.HttpMediaTypeNotSupportedException;
import org.springframework.web.HttpRequestMethodNotSupportedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.MissingServletRequestParameterException;
import org.springframework.web.bind.ServletRequestBindingException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.context.request.WebRequest;
import org.springframework.web.multipart.support.MissingServletRequestPartException;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;
import org.springframework.web.servlet.mvc.multiaction.NoSuchRequestHandlingMethodException;


/**
 * 全局异常处理器
 *
 * @User longkai
 * @Date 13-3-28
 * @Mail im.longkai@gmail.com
 */
@ControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

	@Override
	protected ResponseEntity<Object> handleExceptionInternal(Exception ex, Object body, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return super.handleExceptionInternal(ex, body, headers, status, request);
	}

	// 处理实体无法找到的异常
	@ExceptionHandler({NullPointerException.class, SecurityException.class})
	public ResponseEntity<?> handleEntityNotFound(NullPointerException ex, WebRequest webRequest) {
		l.error(ex.getClass().getSimpleName(), ex.getLocalizedMessage());
		return super.handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), new HttpHeaders(), HttpStatus.UNAUTHORIZED, webRequest);
	}

	// 默认的异常处理
	@ExceptionHandler(Exception.class)
	public ResponseEntity<?> handException(Exception ex, WebRequest webRequest) {
		l.error(ex.getClass().getSimpleName(), ex.getLocalizedMessage());
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), new HttpHeaders(), HttpStatus.BAD_REQUEST, webRequest);
	}

	// 以下所有重载的方法都是为了在response body添加信息

	@Override
	protected ResponseEntity<Object> handleMissingServletRequestParameter(MissingServletRequestParameterException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleNoSuchRequestHandlingMethod(NoSuchRequestHandlingMethodException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleHttpRequestMethodNotSupported(HttpRequestMethodNotSupportedException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleHttpMediaTypeNotSupported(HttpMediaTypeNotSupportedException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleHttpMediaTypeNotAcceptable(HttpMediaTypeNotAcceptableException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleServletRequestBindingException(ServletRequestBindingException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleConversionNotSupported(ConversionNotSupportedException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleTypeMismatch(TypeMismatchException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleHttpMessageNotReadable(HttpMessageNotReadableException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleHttpMessageNotWritable(HttpMessageNotWritableException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleMissingServletRequestPart(MissingServletRequestPartException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	@Override
	protected ResponseEntity<Object> handleBindException(BindException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
		return handleExceptionInternal(ex, jsonError(ex.getLocalizedMessage()), headers, status, request);
	}

	private static String jsonError(String msg) {
		return "{\"success\":false,\"message\":\"" + msg + "\"}";
	}

	private static Logger l = LoggerFactory.getLogger(GlobalExceptionHandler.class);
}
```

## 6. 字符编码和content-type的问题是可能会出现乱码或者，或者json直接以text/html的content type写会会客户端，这不太符合REST的风格。。。
有时可能会出现乱码或者json直接以text/html的content type写会会客户端，这不太符合REST的风格。。。

这个就需要自己配置了，最好是配置内容协商协议（ContentNegotiation），或者在@RequestMapping的设置``produce="application/json;charset=utf-8"``

关于spring的ContentNegotiation，请自行google或者下回再写啦
