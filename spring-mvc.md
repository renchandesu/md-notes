## spring 配置xml模板 
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd
        http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/mvc 
        http://www.springframework.org/schema/context/spring-mvc.xsd">

</beans>
```
###spring IOC
    ioc,是一种通过描述并通过第三方去生产获取对象的方式
#### 实现ioc的方式
    1.通过xml配置 -> 通过classpathXmlApplication 加载bean
```xml
    <bean id="cla1" class="com.ren.beans.Cla"> 
        <property name="cno" value="sno111"/>
        <property name="size" value="1"/>
    </bean>
```
```xml
    <bean id="stu" class="com.ren.beans.Student" scope="prototype">
        <property name="name" value="hello"/>
        <property name="age" value="2"/>
        <property name="cla" ref="cla"/>
    </bean>
```
```xml
<!--注入一些其他数据类型的方法-->
    <bean id="student" class="com.ren.beans.Student" >
<!--        <property name="cla">-->
<!--            <null/>-->
<!--        </property>-->
        <property name="sts">
            <array>
                <value>2</value>
                <value>3</value>
            </array>
        </property>
<!--        二维列表写法 在List中再嵌套list-->
        <property name="ttt">
            <list>
                <list>
                    <value>1</value>
                </list>
                <list>
                    <value>2</value>
                </list>
            </list>
        </property>
<!--        map写法-->
        <property name="map">
            <map>
                <entry key="hello" value="2"/>
            </map>
        </property>
<!--        set写法-->
        <property name="set">
            <set>
                <value>2</value>
                <value>3</value>
            </set>
        </property>
    </bean>
```
自动注入 xml实现 
```xml
    <bean id="test" class="com.ren.beans.Test1" autowire="byName"/>
```
自动注入 注解实现
```xml
<!--开启注解-->
    <context:annotation-config/>
```
```xml
<!--    包扫描 将包下的bean注册 效果等同于使用配置类配置的ComponentScan注解-->
    <context:component-scan base-package="com.ren.twobeans"/>
```

```java
import org.springframework.beans.factory.annotation.Qualifier;

@Data
@Component("teacher")
public class Teacher {
    @Value("zhangsan") // 注入值
    private String name;
    @Value(value = "#{'1,2,3,4,5'.split(',')}") // 注入数组
    private int[] cnos;
    @Autowired //按类型注入
    @Qualifier(value = "") //按照name注入
    private Art art;
}
```
    2.通过配置类配置 -> 通过AnnotationApplication 加载bean
```java
//这两个注解联合使用 可以向ioc容器中注入指定包下的bean
@Configuration
//@ComponentScan("com.ren.twobeans")
public class BeansConfig {

    //向ioc容器中装配Bean 与cs只需要一个存在即可
    @Bean("teacher")
    public Teacher getTeacher(){
        return new Teacher();
    }

    @Bean(value = "art")//起别名
    public Art getArt(){

        return new Art();
    }
}
```
### spring AOP
    基于动态代理实现面向切面编程 AOP中的关键单元是方面
1.基于注解的aop
```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {

}
```
```xml
<aop:aspectj-autoproxy/>
```

```java
import org.junit.Before;

//声明一个方面
@Aspect
public class AspectService {

    @Before("execution(* *.*.*(..))") // 设置建议与切入点
    public void me() {
        System.out.println("hello next is time!");
    }

    public void next() {
        System.out.println("执行后！");
    }

    public void around(ProceedingJoinPoint JP) throws Throwable {
        System.out.println("执行环绕前！");
        Object proceed = JP.proceed();
        System.out.println("执行环绕后！");
    }

    public void afterreturn() {
        System.out.println("返回值之后!");
    }
}
```
2.如果使用xml来配置aop
```xml
    <aop:config>
        <aop:pointcut id="p1" expression="execution(* com.ren.service.ServiceImp.*(..))"/>
<!--        adivsor 用于实现了springframework.aop下的methodBeforeAdivice等其他类-->
    <aop:advisor advice-ref="b" pointcut-ref="p1"/>
<!--    定义方面-->
        <aop:aspect id="asp1" ref="a">
<!--            根据书写顺序决定执行顺序-->
            <aop:after-returning method="afterreturn" pointcut-ref="p1"/>
<!--环绕-->
            <aop:around method="around" pointcut-ref="p1"/>

            <aop:before method="me" pointcut-ref="p1"/>
            <aop:after method="next" pointcut-ref="p1"/>
         </aop:aspect>

    </aop:config>
```

### mybatis-spring
```xml
<depencies>
    <dependency> // 整合包
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis-spring</artifactId>
        <version>2.0.7</version>
    </dependency>
    <dependency> // 驱动
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.27</version>
        <scope>test</scope>
    </dependency>
    <dependency> // 数据源
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>5.3.17</version>
    </dependency>
    <dependency> // mybatis
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.5.7</version>
    </dependency>
</depencies>
```
配置最主要的三个Bean
```xml
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/test"/>
    <property name="username" value="root"/>
    <property name="password" value="ying1234"/>
</bean>
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource" />
<!--        加载mybatis的配置文件-->
        <property name="configLocation" value="mybatis-config.xml"/>
    </bean>
    <bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
        <constructor-arg index="0" ref="sqlSessionFactory" />
    </bean>
```
多出的一步：实现Dao接口，注入sqlSessionTemplate来执行curd方法

### spring声明式事务
```xml
<!--    开启事务-->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <constructor-arg ref="dataSource" />
    </bean>
<!--    开启切面-->
    <aop:aspectj-autoproxy/>
    <!--    配置事务通知-->
    <tx:advice id="tx" transaction-manager="transactionManager">
<!--        給哪些方法配置事务-->
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED"/>
        </tx:attributes>
    </tx:advice>
    <aop:config>
        <aop:pointcut id="pt" expression="execution(* com.ren.service.DbServiceImp.*(..))"/>
        <aop:advisor advice-ref="tx" pointcut-ref="pt"/>
    </aop:config>
```
### 拦截器

​	拦截器类似于servlet中的过滤器，用于对处理器进行预处理和后处理，开发者可以自己定义一些拦截器来实现特定功能

​	是MVC框架自带的，只会拦截访问的控制器方法

**实现拦截器**

```java
public class MyInterceptor implements HandlerInterceptor {
    @Override 
    //return true  放行 执行下一个拦截器
    //return false 不放行 不执行下一个拦截器
    // 处理前
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        return HandlerInterceptor.super.preHandle(request, response, handler);
    }
	// 处理后 指controller 执行后
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
    }
	//清理
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
}
```

**配置拦截器**

```xml
    <mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/**"/>
            <bean class="com.ren.config.intercepter.MyInterceptor"/>
        </mvc:interceptor>
    </mvc:interceptors>
```

### 浏览器缓存 
304 表示请求发送给了服务器，但是服务器发现响应的内容与之前相同，所以让浏览器从缓存读取
响应有个字段为cache-control max-age 表示浏览器在一段时间内不需要向服务器发送请求而直接从硬盘读取
no-cache 表示每次请求都需要向服务器发送，但是是可以使用本地缓存的
no-store 不允许使用本地缓存