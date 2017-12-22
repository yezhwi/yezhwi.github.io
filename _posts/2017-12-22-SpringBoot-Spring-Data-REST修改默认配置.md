---
layout:     post
title:      SpringBoot-Spring-Data-REST修改默认配置
subtitle:   定制化的操作实现
date:       2017-12-22
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springboot
tags:
    - SpringBoot
    - RESTfulAPI
---

### 背景

在[上一篇](https://yezhwi.github.io/springboot/2017/12/17/SpringBoot-Spring-Data-REST%E8%BD%BB%E6%9D%BE%E6%90%9E%E5%AE%9ARESTfulAPI/)中除去配置类和实体类，写了两行代码，就实现了RESTful风格的接口，但在实际使用时，还需要一些额外的处理，比如在返回的数据中，password这类敏感字段是不应该返回的；删除操作，实际需求不是硬删除只是更新一个删除状态；保存对象操作之前需要做相应的数据校验和数据格式的转换等等，***自动转换成REST服务，是否支持自定义功能？***

### 分页+排序查询(这个与以往的习惯有点不同)

`http://localhost:8080/user?sort=id,desc&size=2`

> sort desc 这类关键字是否可以自定义？因为运维方面会对这些关键字做防SQL注入（waf）

```
{
  "_embedded": {
    "users": [
      {
        "userName": "Yezhwi4",
        "password": "123",
        "phone": "12345678901",
        "locked": 0,
        "_links": {
          "self": {
            "href": "http://localhost:8080/user/17"
          },
          "user": {
            "href": "http://localhost:8080/user/17"
          }
        }
      },
      {
        "userName": "Yezhwi5",
        "password": "123",
        "phone": "12345678901",
        "locked": 0,
        "_links": {
          "self": {
            "href": "http://localhost:8080/user/16"
          },
          "user": {
            "href": "http://localhost:8080/user/16"
          }
        }
      }
    ]
  },
  "_links": {
    "first": {
      "href": "http://localhost:8080/user?page=0&size=2&sort=id,desc"
    },
    "self": {
      "href": "http://localhost:8080/user"
    },
    "next": {
      "href": "http://localhost:8080/user?page=1&size=2&sort=id,desc"
    },
    "last": {
      "href": "http://localhost:8080/user?page=5&size=2&sort=id,desc"
    },
    "profile": {
      "href": "http://localhost:8080/profile/user"
    }
  },
  "page": {
    "size": 2,
    "totalElements": 11,
    "totalPages": 6,
    "number": 0
  }
}
```

### 自定义查询方法

根据给定的字段查找相应表中的数据对象。比如在上一篇中定义的User实体，需要一个按照userName(假如数据库中唯一)值查到与之对应的数据对象返回，只需要在UserRopository中定义如下代码：

```
import com.gemantic.model.User;
import com.google.common.base.Optional;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

/**
 * @author Yezhiwei
 * @date 17/12/16
 */
@RepositoryRestResource(path = "user")
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByUserName(@Param("userName") String userName);

}
```

所有的查询方法，默认转成search路径下，如：`http://localhost:8080/user/search/findByUserName?userName=Yezhiwei`

![查询方法](/img/spring-data-rest-5.png)

上图中自动转成的URL是按方法名，若要更改该查询方法所暴露的URL，在方法上使用`@RestResource`注释

```
@RestResource(path = "name", rel = "name")
Optional<User> findByUserName(@Param("userName") String userName);
```

输入 `http://localhost:8080/user/search/` 查看user下的所有查询方法有哪些：

![user下的所有查询方法](/img/spring-data-rest-6.png)

查看资源user下所有查询方法 /user/search

path = "name" 将路径参数findByUserName转为name /user/search/name

rel = "name" 更改参数findByUserName转为name


### 多条件查询

如果匹配多个条件查询，则用And和Or连接，比如：

```
import com.gemantic.model.User;
import com.google.common.base.Optional;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
import org.springframework.data.rest.core.annotation.RestResource;

import java.util.List;

/**
 * @author Yezhiwei
 * @date 17/12/16
 */
@RepositoryRestResource(path = "user")
public interface UserRepository extends JpaRepository<User, Long> {

    ...

    @RestResource(path = "nameAndPhone", rel = "nameAndPhone")
    List<User> findByUserNameAndPhone(@Param("userName") String userName, @Param("phone") String phone);

}
```

![多条件查询](/img/spring-data-rest-7.png)

### 排序

可以在方法名称的结尾处添加OrderBy，实现结果集排序。比如可以按照User的userName降序排列

```
import com.gemantic.model.User;
import com.google.common.base.Optional;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
import org.springframework.data.rest.core.annotation.RestResource;

import java.util.List;

/**
 * @author Yezhiwei
 * @date 17/12/16
 */
@RepositoryRestResource(path = "user")
public interface UserRepository extends JpaRepository<User, Long> {

    ...


    @RestResource(path = "nameStartsWith", rel = "nameStartsWith")
    List<User> findByUserNameStartsWithOrderByUserNameDesc(@Param("userName") String userName);

}
```

![排序](/img/spring-data-rest-8.png)

### 隐藏某个字段

比如在实体对象User中，不希望password序列化为JSON，Spring Data REST默认使用的是JackSon，则我们就可以在需要隐藏的字段添加@JsonIgnore即可 

```
@Data
@Entity
public class User {

    @Id
    @GeneratedValue(strategy= GenerationType.AUTO)
    private Long id;

    private String userName;

    @JsonIgnore
    private String password;

    private String phone;

    private Integer locked;
}
```

![隐藏某个字段](/img/spring-data-rest-9.png)

### 隐藏自定义的查询方法

只需要在方法上添加 `@RestResource(exported = false)` 即可，如：

```
/**
 * @author Yezhiwei
 * @date 17/12/16
 */
@RepositoryRestResource(path = "user")
public interface UserRepository extends JpaRepository<User, Long> {

//    @RestResource(path = "name", rel = "name")
    @RestResource(exported = false)
    Optional<User> findByUserName(@Param("userName") String userName);

    @RestResource(path = "nameAndPhone", rel = "nameAndPhone")
    List<User> findByUserNameAndPhone(@Param("userName") String userName, @Param("phone") String phone);


    @RestResource(path = "nameStartsWith", rel = "nameStartsWith")
    List<User> findByUserNameStartsWithOrderByUserNameDesc(@Param("userName") String userName);

}
```

![隐藏自定义的查询方法](/img/spring-data-rest-10.png)

### 屏蔽自动化方法(repository CRUD)`extends CrudRepository `

在实际生产环境中，不会真正删除用户数据，此时不希望DELETE的提交方式生效，可以添加@RestResource注解，并设置exported=false，即可屏蔽Spring Data REST的自动化方法 ，只需要写如下代码：

```
/**
 * @author Yezhiwei
 * @date 17/12/16
 */
@RepositoryRestResource(path = "user")
public interface UserRepository extends CrudRepository<User, Long> {

	...

    @RestResource(path = "nameStartsWith", rel = "nameStartsWith")
    List<User> findByUserNameStartsWithOrderByUserNameDesc(@Param("userName") String userName);

    @Override
    @RestResource(exported = false)
    void delete(Long id);

}
```

### 修改对象返回值json数据中的key

默认是User对象的属性，在一些特殊情况下，需要修改json返回值中的key，需要定义如下代码：

```
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.rest.core.config.Projection;

/**
 *
 */

@Projection(name = "vu",types = User.class)
public interface VirtualUser {

    @Value("#{target.userName}-#{target.phone}")
    String getFullInfo();

    @Value("#{target.locked}")
    Integer getStatus();
}
```

这里把User中的userName和phone合并成一列，locked重新命名为stauts，这里需要注意方法名前面一定要加get，不然无法序列化为JSON数据。启动服务，试一试

`http://localhost:8080/user/1`

![默认](/img/spring-data-rest-11.png)

`http://localhost:8080/user/1?projection=vu`

![修改属性](/img/spring-data-rest-12.png)



参考 [Customizing Spring Data REST](https://docs.spring.io/spring-data/rest/docs/current/reference/html/#customizing-sdr)













