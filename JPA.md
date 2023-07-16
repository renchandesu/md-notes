## ORM

> `ORM`是面向对象程序设计语言和[关系型数据库](https://so.csdn.net/so/search?q=关系型数据库&spm=1001.2101.3001.7020)发展不同步时的解决方案，采用 `ORM`[框架](https://so.csdn.net/so/search?q=框架&spm=1001.2101.3001.7020)后，应用程序不再直接访问底层数据库，而是以面向对象的方式来操作持久化对象，而`ORM`框架则将这些面向对象的操作转换成底层的 SQL 操作。





# JPA

### 依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

```
//驱动 版本高了启动报错
<dependency>
    <groupId>com.microsoft.sqlserver</groupId>
    <artifactId>mssql-jdbc</artifactId>
    <version>8.4.1.jre8</version>
</dependency>
```

### yml配置

```
spring:
  datasource:
    url: jdbc:sqlserver://49.235.94.85:1433;DatabaseName=test
    username: sa
    password: <ying@123456>
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver

  jpa:
    hibernate:
    //启动时表如果不存在则创建表，存在则更新数据
      ddl-auto: update
    show-sql: true
```

### entity

```
@Data
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    @Column
    private String name;
    @Column
    private Integer age;
    @Column
    private Date birth;
}
```

@Transient表示忽略这一字段

主键生成策略: 

![image-20220705133809826](C:\Users\10441\AppData\Roaming\Typora\typora-user-images\image-20220705133809826.png)

​												序列用于oracle数据库 

主键使用uuid

```
@Id
@GeneratedValue(generator="system_uuid")
@GenericGenerator(name="system_uuid",strategy="uuid")
```

##### 表注释 参数类型

```
@Column(columnDefinition = "int(11) comment 'id'")
```

##### 索引

```
@Table(indexes = {
        @Index(columnList = "name"),
        @Index(columnList = "account")
})
```

```
//组合索引
@Table(indexes = {
        @Index(name = "name_account", columnList = "name"),
        @Index(name = "name_account", columnList = "account")
})
```

```
//唯一索引
@Table(uniqueConstraints = @UniqueConstraint(columnNames= {"name"}))
```

### repository

继承JpaRepository接口，通过@Query注解或者函数命名方式实现数据访问



### @Query注解

@Query ⽤法是使⽤ JPQL 为实体创建声明式查询⽅法。我们⼀般只需要关⼼ @Query ⾥⾯的 value 和 nativeQuery、countQuery 的值即可，因为其他的不常⽤。

语法结构有点类似 SQL，唯⼀的区别就是 JPQL FROM 后⾯跟的是对象，⽽ SQL ⾥⾯的字段对应的是对象⾥⾯的属性字段。

如果使用nativeQuery 则不支持Sort直接传入 **JPQL**可以直接使用Sort

```
@Query(value="from Customer  c where c.custId=:#{#customer.custId}")
Customer findCustomerByInfo(@Param("customer") Customer customer);```
```
```
@Query(value = "select * from user_info where first_name=?1 order by ?2",nativeQuery = true)
List<UserInfoEntity> findByFirstName(String firstName, String sort);
// 调⽤的地⽅写法 last_name 是数据⾥⾯的字段名，不是对象的字段名
repository.findByFirstName("jackzhang","last_name");
```





### 联合查询

##### 一对多

```
@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    @Column(nullable = false)
    private String name;
    @Column(nullable = false)
    private Integer age;
    @Column(nullable = false)
    private Date birth;
    // 重点是这个注解 target是多              表示关联的字段          // 非懒加载
    @OneToMany(targetEntity = Item.class,mappedBy = "sno", fetch = FetchType.EAGER)
    private List<Item> itemList;
}
```
