---
layout:  post
title:   Spring Boot Customize @Valid Logic
date:    2018-05-07
author:  抠脚大猩猩
catalog: true
tags:
    - Spring
    - Spring Boot 
---
# Spring Boot Customize @Valid logic
------
在使用Spring的时候我们通常使用@Valid标签对请求参数进行校验。但是***javax.validation.constraints***仅提供了几种简单的检验逻辑，很多时候这些简单的校验逻辑并不能满足我们在开发中的需求。比如说，我们需要校验用户传入的对象是否已经存在于数据库中。这个时候我们就需要自己定义我们的校验逻辑。这样我们仅需要在需要校验的地方加上我们自定义的标签就可以完成对于要校验对象的校验。
> * 修改pom.xml文件
> * 定义注解接口
> * 为注解接口提供实现类
> * 对要验证的对象提供注解

------

## 1. 修改pom.xml 文件

因为我们只实现几个基础的功能,所有我们尽需要如下几个dependency, 剩余的pom.xml所需模块可以自行添加。本文使用的Spring Boot 版本为 2.0.1

```xml
<dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>com.voodoodyne.jackson.jsog</groupId>
      <artifactId>jackson-jsog</artifactId>
      <version>1.1</version>
    </dependency>
    <dependency>
	  <groupId>org.springframework.boot</groupId>
	  <artifactId>spring-boot-starter-test</artifactId>
	  <scope>test</scope>
	</dependency>
</dependencies>	
```

## 2. 定义注解接口
在定义接口前我们需要知道我们的注解是要应用于哪一层的。是类级别还是方法级别，或者是变量级别。这个需要我们在@Target标签内注明。第二个我们需要注意的地方是，我们的注解作用的时间是编译时还是运行时。这个需要我们在@Retention标签内注明。第三个就是该注解的实现类，一个注解可以有多个实现类。实现类可以坐在@Constraint标签内注明。
以下是具体的代码实现。
```java
package com.example.validator.model.interf;

import com.example.validator.model.impl.UniqueUserIdValidator;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import javax.validation.Constraint;
import javax.validation.Payload;

@Target({ElementType.TYPE}) // 1. 
@Retention(RetentionPolicy.RUNTIME) // 2.
@Constraint(validatedBy = {UniqueUserIdValidator.class}) // 3.
public @interface UniqueId {

    String message() default "{User Id already existed}"; // 4.
    Class<?>[] groups() default {};
    Class<? extends Payload>[] Tarpayload() default {};
}
```
以上代码我们定义了一个名为UniqueId的注解。1.注解的作用范围是类级别。 2.注解作用的时间为运行时。3.注解的实现类为UniqueUserIdValidator.class。4.注解的默认信息为“User Id already existed”。

## 3.注解接口实现类
```java
package com.example.validator.model;

import com.example.validator.model.UserView;
import com.example.validator.model.interf.UniqueId;
import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

public class UniqueUserIdValidator implements ConstraintValidator<UniqueId, UserView> {

    int[] ids = {1, 2, 3, 4, 5}; // for test
    
    @Override
    public void initialize(UniqueId constraintAnnotation) {

    }

    @Override
    public boolean isValid(UserView userView, ConstraintValidatorContext constraintValidatorContext) {
        return !userView.getId().isEmpty() && !Arrays.asList(ids).contains(userView.getId());
    }
}
```
这段代码我只是想实现一个简单的功能，所以并没有实现与数据库交互的部分。我们简单的判断了一下rest请求的对象是否符合我们要校验的逻辑，如果不符合我们就返回一个false，这里的返回值，会被@Valid标签接收，如果返回值为false，@Valid标签会抛出一个MethodArgumentNotValidException错误

## 4. 对需要验证的对象提供注解
```java
package com.example.validator.model;

import com.example.validator.model.interf.UniqueId;

@UniqueId(message = "user id: ${validatedValue} already existed.")
public class UserView {

	private String id;

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}

    @Override
    public String toString() {
        return id;
    }
}
```
因为我们的注解作用级别为类，所以我们将注解写在了类名上， 其中message里的${validatedValue}的值为toString()方法的返回值, 这样我们就能返回用户的输入。最后我们只需要在controller中加上@Valid标签就可以实现我们需要的功能了。
```java
@RequestMapping(method = RequestMethod.POST)
	public void save(@Valid @RequestBody UserView userView) {
		// dummy
	}
```