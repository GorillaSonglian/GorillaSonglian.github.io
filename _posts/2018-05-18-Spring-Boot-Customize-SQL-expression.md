---
layout:  post
title:   Spring Boot Customize SQL Expression
date:    2018-05-18
author:  抠脚大猩猩
catalog: true
tags:
    - Spring
    - Spring Boot 
---
# Spring Boot Customize SQL Expression
在使用Spring的时候我们可以通过继承JpaRepository或CrudRepository来实现与数据库的交互。 但是很多时候, 仅通过继承这两个接口并不能实现我们对于一些复杂的SQL命令的需求。这时候我们就需要自定义一个接口，然后写一个接口实现类用来实现我们所需要的SQL命令。然后再定义一个新的接口继承我们刚刚定义的接口和JpaRepository或CrudRepository。
> * 修改pom.xml文件
> * 定义数据模型
> * 定义查询接口
> * 实现查询接口
> * 定义接口继承查询接口和JpaRepository或CrudRepository接口

## 修改pom.xml文件
在这里我使用的是MySQL数据库，所以我添加了mysql的依赖，如果使用其他的数据库，只需将对应的数据库依赖添加到pom.xml文件中即可。剩余的pom.xml所需模块可以自行添加。本文是用的Spring Boot 版本为2.0.1
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<scope>runtime</scope>
</dependency>
```

## 定义数据模型
因为我们只是为了说明怎样实现功能, 说以我简化了数据模型。
```java
package com.example.customizesql.model;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;
import javax.persistence.Transient;
import javax.validation.constraints.Email;
import javax.validation.constraints.NotEmpty;
import javax.validation.constraints.Size;


@Entity
@Table(name = "user")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "user_id")
    private int id;
    @Column(name = "email")
    @Email(message = "*Please provide a valid Email")
    @NotEmpty(message = "*Please provide an email")
    private String email;
    @Column(name = "password")
    @Size(min = 5, message = "*Your password must have at least 5 characters")
    @NotEmpty(message = "*Please provide your password")
    @Transient
    private String password;
    @Column(name = "name")
    @NotEmpty(message = "*Please provide your name")
    private String name;
    @Column(name = "last_name")
    @NotEmpty(message = "*Please provide your last name")
    private String lastName;

    // ignore getters and setters
}

```

## 定义查询接口
在这里我们定义一个简单的实现接口，这个接口含有一个通过email查询用户的方法。当然这个方法我们直接继承JpaRepository或者CrudRepository就可以实现。但是我们在这里为了展示具体的实现方法，就不写一个复杂的查询方法了。
```java
package com.example.customizesql.repository;

public interface CustomizedUserRepository {

    void getUserByEmail(String email);
}

```

## 实现查询接口
在实现查询接口这里是有一个特别需要注意的地方。查询接口的实现类名的前缀应该与查询接口名完全一致，只在前缀后加上Impl。只有这样，Spring才能扫描到这个查询接口的实现类。
```java
package com.example.customizesql.repository;

import com.example.customizesql.model.User;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;

public class CustomizedUserRepositoryImpl implements CustomizedUserRepository {

    @Autowired
    private JdbcTemplate jdbcTemplate;
    private static final Logger LOGGER = LoggerFactory.getLogger(CustomizedUserRepository.class);
    @Override
    public void getUserByEmail(String email) {
        String sql = "SELECT * FROM `user` WHERE `email`= ?";
        ArrayList<User> user = (ArrayList<User>) jdbcTemplate.query(sql, new Object[]{email}, new RowMapper<User>() {
            @Override
            public User mapRow(ResultSet resultSet, int i) throws SQLException {
                User user1 = new User();
                user1.setId(resultSet.getInt("user_id"));
                user1.setName(resultSet.getString("name"));
                return user1;
            }
        });
        for (User t: user) {
            LOGGER.info(t.getName());
        }
    }

}

```

## 定义接口继承查询接口和JpaRepository或CrudRepository接口
```java
package com.example.customizesql.repository;

import ccom.example.customizesql.model.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository("userRepository")
public interface UserRepository extends JpaRepository<User, Long>, CustomizedUserRepository {

}

```
之后我们就可以按照如下方式使用我们定义好的查询接口了
```java
@Autowired
private UserService userService;
```