---
layout:     post
title:      SpringBoot-JPA数据库操作
subtitle:   
date:       2018-07-31
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: springboot
tags:
    - SpringBoot
    - JPA
---

![JPA关系图](https://ws2.sinaimg.cn/large/006tNc79ly1ftroylpdirj30m70auq2x.jpg)

### JPA 常用API

* CurdRepository 提供了增删改产方法。
* PagingAndSortingRepositroy 增加分页查询和排序方法。
* JpaRepositroy 增加了实例查询方法。
* 实际应用中可以有选择的继承上面任何一个接口都可以。

### JPA 查询语句关键字

JPA 提供的查询方式可以直接从方法名中派生成查询（需要遵循它的规范），都需要以findBy开头，且方法中的字段名必须与实体类中的属性名一致，并遵循驼峰式代码编写风格

| Keyword | Sample | JPQL snippet | |
| :------ | :------ | :------ | :------ |
| And | findByLastnameAndFirstname | where x.lastname = ?1 and x.firstname = ?2 | |
| Or | findByLastnameOrFirstname | where x.lastname = ?1 or x.firstname = ?2 | |
| Is, Equals | findByFirstname,findByFirstnameIs,</br>findByFirstnameEquals	 | where x.firstname = 1? | |
| Between | findByStartDateBetween | where x.startDate between 1? and ?2 | |
| LessThan | findByAgeLessThan | where x.age < ?1 | |
| LessThanEqual | findByAgeLessThanEqual | where x.age <= ?1 | |
| GreateThan | findByAgeGreaterThan | where x.age > ?1 | |
| GreateThanEqual | findByAgeGreaterThanEqual | where x.age >= ?1 | |
| After | findByStartDateAfter | where x.startDate > ?1 | |
| Before | findByStartDateBefore | where x.startDate < ?1 | |
| IsNull | findByAgeIsNull | where x.age is null | |
| IsNotNull, NotNull | findByAge(Is)NotNull | where x.age not null | |
| Like | findByFirstnameLike	 | where x.firstname like ?1 | 不推荐使用 |
| NotLike | findByFirstnameNotLike | where x.firstname not like ?1 | 不推荐使用 |
| StartingWith | findByFirstnameStartingWith | where x.firstname like ?1 (parameter bound with appended %) | |
| EndingWith | findByFirstnameEndingWith | where x.firstname like ?1 (parameter bound with prepended %) | 不推荐使用 |
| Containing | findByFirstnameContaining | where x.firstname like ?1 (parameter bound wrapped in %) | 不推荐使用 |
| OrderBy | findByAgeOrderByLastnameDesc | where x.age = ?1 order by x.lastname desc | |
| Not | findByLastnameNot | where x.lastname <> ?1 | |
| In | findByAgeIn(Collection<Age> ages) | where x.age in ?1 | |
| NotIn | findByAgeNotIn(Collection<Age> age) | where x.age not in ?1 | |
| True | findByActiveTrue() | where x.active = true | |
| False | findByActiveFalse() | where x.active = false | |
| IgnoreCase | findByFirstnameIgnoreCase | where UPPER(x.firstame) | 不推荐使用 |

> JPA查询方法，条件查询，list集合查询，分页查询，排序查询，sql语句查询，指定字段查询。

```
public interface PersonRepository extends JpaRepository<Person, Integer> {
	
    //自定义查询，And 为 JPA 关键字，相当于(where x.lastname = ?1 and x.firstname = ?2)
    Person findByNameAndPassword(String name, String password); 
        
    //排序查询，返回 list 集合
    List<Person> findByNameAndPassword(String name, String password, Sort sort);
         
    //分页查询 查询计算元素总个数总页数，数据多的情况下，性能不高
    Page<Person> findByNameAndPassword(String name, String password, Pageable pageable);
    //分页查询，返回的是一个片段，它只知道下一片段或者上一片段是否可用。
    Slice<Person> findByNameAndPassword(String name, Pageable pageable); 
 
    //sql 查询。?后面数字，对应方法中参数位置。使用原生 sql 语句
    @Query("select p from person as p where p.name = ?1 and p.password = ?2 ",nativeQuery = true) 
    Person myfind(String name, String password);
}
```

### JPA 的 CRUD 操作

#### 根据 ID 查询数据

实现方式有三种：

* `JpaRepository` 里的 `T getOne(ID var1);`
* `CrudRepository` 里的 `T findOne(ID var1);`
* 根据查询语句关键词命名规范自定义 `T findById(Long id);`

经过测试，三种方式在 `ID` 存在的情况下，都能返回正常的结果对象。但是如果参数是一个数据库中不存在的 `ID` 值时，`getOne` 的返回值有些不同：

![getOne 懒加载](https://ws3.sinaimg.cn/large/006tKfTcly1ftru4r3q8dj31kw0phabu.jpg)

从上图的信息可以看出 `getOne` 是懒加载的模式，在使用对象的时候才真正地去赋值。

***所以，根据我们以前的规范，如果返回值是单个对象时，存在就返回对象，否则返回 null；如果是 List 等集合类型时，返回一个包含0个对象的空集合。这样方便调用端统一判断，避免 NPE 。***

#### 更新操作（update/delete）

实现方式有两种：

* save()
* 自定义更新操作

> save 方法，从字面的意思是保存新数据，但是如果对象中有 ID 值，则执行更新操作，测试用例如下：

```
// 原数据库中的记录：UserPushLastLogin(id=3480, userCode=hpeihpeixxxxyyy, appCode=CP1501141, loginAt=1532077916498, createAt=1532077916498, updateAt=1532077916498)
@Test
public void testSaveForUpdate() {

    UserPushLastLogin userPushLastLogin = new UserPushLastLogin();
    userPushLastLogin.setId(3480L);
    userPushLastLogin.setUserCode("hpeihpeixxxx");

    userPushLastLogin = userPushLastLoginRepository.save(userPushLastLogin);
    System.out.println("==>" + userPushLastLogin);
}

// 通过设置显示 SQL 得到如下输出
Hibernate: select userpushla0_.id as id1_0_0_, userpushla0_.app_code as app_code2_0_0_, userpushla0_.create_at as create_a3_0_0_, userpushla0_.login_at as login_at4_0_0_, userpushla0_.update_at as update_a5_0_0_, userpushla0_.user_code as user_cod6_0_0_ from user_jpush_last_login userpushla0_ where userpushla0_.id=?
Hibernate: update user_jpush_last_login set app_code=?, create_at=?, login_at=?, update_at=?, user_code=? where id=?
==>UserPushLastLogin(id=3480, userCode=hpeihpeixxxx, appCode=null, loginAt=null, createAt=null, updateAt=null)
```

***通过输出发现，实体中未设置值的字段都被更新为null；通过显示的 SQL 可知，先根据 ID 执行查询，然后再执行更新，比自定更新操作多执行了一次查询操作。因此使用此方法要小心。***

> 自定义更新操作： 只要涉及修改或删除数据的操作都需要加上注释 `@Modifying` 和`@Transcational`（ Transcational 是org.springframework.transaction.annotation 包中的不要导错了，如果不加此注解，在服务启动的时候会报错）

```
@Modifying
@Query("UPDATE UserLogin u SET u.userCode=?1 WHERE u.id in (1, 2754, 3034, 3148)")
@Transactional
// 返回值为受影响的条数
int updateUserCodes(String userCode);

// 单元测试结果为(因为 ID 为1的记录不存在)： 
// Hibernate: update user_jpush_last_login set user_code=? where id in (1 , 2754 , 3034 , 3148)
// row==>3
```

#### 分页查询

```
//分页查询 查询计算元素总个数总页数，数据多的情况下，性能不高
Page<Person> findByNameAndPassword(String name, String password, Pageable pageable);
//分页查询，返回的是一个片段，它只知道下一片段或者上一片段是否可用。
Slice<Person> findByNameAndPassword(String name, Pageable pageable);
```

***通过设置显示 SQL 发现，Page 返回值的接口，会多执行一条 count 语句。所以，当数据量很大的时候性能不高。Slice 返回值的接口，通过 hasNext 来判断是否有下一页，与我们之前的用法相似。***

***













