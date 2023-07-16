### springboot

### 简介

springboot的底层是spring框架，并且能够简化构建配置（引入启动器），内嵌web服务器，自动配置spring以及第三方功能

### 依赖管理机制

父项目依赖管理

```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.6.7</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>2.6.7</version>
    </parent>
<!--    几乎声明了常用开发中所有常用的依赖的版本号
        如果需要修改版本号 查看dependencies中的变量名 在pom文件里声明一个properties属性 修改版本
        starter会引入开发场景的所有需要的依赖-->
```

### 自动配置原理

+ 默认的包扫描路径 sba同目录下的所有子包

+ 自动配置好tomcat

+ 自动配置好springMVC 如视图解析器 分发器 编码过滤器等

+ 各种配置都拥有默认值

+ 按需加载所有的自动配置项
  
  - 有非常多的starter,引入了哪些场景这个场景的自动配置才会开启
    
    ### 与配置bean相关的注解
    
    #### @Configuration @Import @ImportResource @Bean @Conditional系列
    
    ```java
    /***
  * proxyBeanMethods 代理bean的方法 true的话是用cglib实现代理
  
  * 指定configuration的使用模式：full模式与Lite模式
  
  * 如果true 保证每个bean方法返回的都是单实例
  
  * 如果是false 每个bean方法返回的都是新创建的
  
  * 如果bean有依赖关系，则需要设置为true
  
  * 设置为FALSE可以减少启动时间
  
  * 
  
  * @Import({Class.class}) 导入一个组件并注册到容器中 id为全限定名
  
  * @ImportResource("classpath:...") 导入xml中的bean 
  
  * 
  
  * 条件装配 @Conditional 只有满足条件时才会装配bean
    */
    @Configuration(proxyBeanMethods = false) // 告诉springboot这是一个配置类 本身也是组件
    public class MyConfiguration {
    @Bean("pet1") // 将返回的对象注册到容器中 默认以方法名作为组件id 但也可以自己命名
    public Pet getPet(){
    
        return new Pet();
    
    }
    }
    @SpringBootApplication
    public class SpringbootlApplication {
  
  public static void main(String[] args) {
    //返回ioc容器
    ConfigurableApplicationContext run = SpringApplication.run(SpringbootlApplication.class, args);
    // 查看容器里面的bean
    String[] beanDefinitionNames = run.getBeanDefinitionNames();
  //        for (String beanDefinitionName : beanDefinitionNames) {
  //            System.out.println(beanDefinitionName);
  //        }
    MyConfiguration myConfiguration = run.getBean("myConfiguration", MyConfiguration.class);
    System.out.println(myConfiguration.getPet() == myConfiguration.getPet());
  }
  }
  ```
  
  #### 装配bean的属性的方式
1. @Component + @ConfigurationProperties(prefix = "")

2. @EnableConfigurationProperties({Pet.class})(用于配置类)+@ConfigurationProperties(prefix = "")

3. 使用@PropertySource("classpath:user.properties")获取对应的properties文件，再用@ConfigurationProperties(prefix = "user")进行属性映射
   
   ```java
   @Configuration(proxyBeanMethods = true) // 告诉springboot这是一个配置类 本身也是组件
   @EnableConfigurationProperties({Pet.class})
   public class MyConfiguration {
   // @Bean("pet1") // 将返回的对象注册到容器中 默认以方法名作为组件id 但也可以自己命名
    public Pet getPet(){
        return new Pet();
    }
   }
   ```

@Data
@ConfigurationProperties(prefix = "pet")
public class Pet {
  private String name;
}

Pet pet = run.getBean(Pet.class);
System.out.println(pet.getName());

```
###自动配置原理入门
```java
xxxxAutoConfiguration：帮我们给容器中自动配置组件；
xxxxProperties:配置类来封装配置文件的内容；
```

**总结：**

* springboot先加载所有的自动配置类 xxxAutoConfiguration
* 每个自动配置类按照条件（@Conditional）进行生效，默认会绑定配置文件指定的值。即加载xxxProperties里面的值，且改文件绑定了配置文件
* 生效的配置类才会给容器中加载Bean
* 只要容器中有bean 相应功能就有了
* 用户可以进行定制化：
  * 直接用自己的bean替换底层组件
  * 修改配置文件中的值

### 配置文件

1. properties
2. yml
- yml语法
  
  * 非常适合以数据为中心的配置文件
  
  * 以 key: value 表示
  
  * 缩进表示层级关系
  
  * 不允许tab 只允许space
  
  * \#表示注释
  
  * 字符串不需要双引号
  
  * 单双引号的意义
    
    * 单引号字符串不转义 双引号转义
  
  * 各种写法：
    
    * 数组（或set） k: [v1,v2] 或 k: -v1 -v2 -v3 需要回车
    * 对象(或map) k:{k: v} 或  k: k1: v1 需要回车
  
  * 绑定数据与properties一样的方式
    
    ```properties
    #常用配置
    server.port=8080
    spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
    spring.datasource.url=jdbc:mysql://localhost:3306/ssmbuild?characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
    ```

spring.datasource.password=ying1234
spring.datasource.username=root

pet.name = wang

# 静态资源位置

spring.web.resources.static-locations=[classpath:/name/]

# 映射路径

spring.mvc.static-path-pattern=/res/**

# 是否使用静态资源映射

spring.web.resources.add-mappings=true

# 缓存时间

spring.web.resources.cache.period=1000

#开启restful风格
spring.mvc.hiddenmethod.filter.enabled=true

# 开启基于请求参数的内容协商

spring.mvc.contentnegotiation.favor-parameter=true

# 路径前缀

server.servlet.context-path=/path

# 上传的文件最大大小

spring.servlet.multipart.max-file-size=10MB

```
### springWeb的定制化
> If you want to keep those Spring Boot MVC customizations and make more MVC customizations (interceptors, formatters, view controllers, and other features), you can add your own @Configuration class of type WebMvcConfigurer but without @EnableWebMvc.  \
  If you want to provide custom instances of RequestMappingHandlerMapping, RequestMappingHandlerAdapter, or ExceptionHandlerExceptionResolver, and still keep the Spring Boot MVC customizations, you can declare a bean of type WebMvcRegistrations and use it to provide custom instances of those components. \
  If you want to take complete control of Spring MVC, you can add your own @Configuration annotated with @EnableWebMvc, or alternatively add your own @Configuration-annotated DelegatingWebMvcConfiguration as described in the Javadoc of @EnableWebMvc.

```java
//springboot会最后加载用户的配置类，执行定义的方法，然后覆盖之前的设置，以达到定制的目的
@Configuration
public class MVCConfig implements WebMvcConfigurer {
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        UrlPathHelper urlPathHelper = new UrlPathHelper();
        urlPathHelper.setRemoveSemicolonContent(false);
        configurer.setUrlPathHelper(urlPathHelper);

    }
}
```

### 静态资源访问

* springboot的默认静态资源映射路径为 `/static` `/public /resources` `/META-INF/resources` 访问只需要/+资源名即可

* 所有 /webjars/**（主要是web的资源打的jar包） ，都去 classpath:/META-INF/resources/webjars/ 找资源

* 原理：映射为/**

* 也可以修改配置文件，已修改静态资源访问前缀（`spring.mvc.static-path-pattern=/resources/**`） 方便拦截请求

* 修改静态资源位置：`spring.web.resources.static-locations=[classpath:/name/]`

* 请求进入，先去controller看能不能处理 不能处理的请求都交给静态资源处理器，还找不到404
  
  #### 欢迎页
  
  > Spring Boot supports both static and templated welcome pages. It first looks for an index.html file in the configured static content locations. If one is not found, it then looks for an index template. If either is found, it is automatically used as the welcome page of the application.

---

***v2.6.7 欢迎页与icon的访问映射在设置了静态资源访问路径后仍然无法正确访问***

#### 静态资源配置原理

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnWebApplication(
    type = Type.SERVLET
)
@ConditionalOnClass({Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class})
@ConditionalOnMissingBean({WebMvcConfigurationSupport.class}) // 当配置了这个类时，失效
@AutoConfigureOrder(-2147483638)
@AutoConfigureAfter({DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class, ValidationAutoConfiguration.class})
public class WebMvcAutoConfiguration {}
```

```java
// 与spring.mvc spring.web的属性进行了绑定
    @Configuration(
        proxyBeanMethods = false
    )
    @Import({WebMvcAutoConfiguration.EnableWebMvcConfiguration.class})
    @EnableConfigurationProperties({WebMvcProperties.class, WebProperties.class}) // 配置文件中的属性与xx进行了绑定
    @Order(0)
    public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer, ServletContextAware {
        //添加资源处理器
        public void addResourceHandlers(ResourceHandlerRegistry registry) {
          if (!this.resourceProperties.isAddMappings()) {
            logger.debug("Default resource handling disabled");
          } else {
            this.addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
            this.addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
              registration.addResourceLocations(this.resourceProperties.getStaticLocations());
              if (this.servletContext != null) {
                ServletContextResource resource = new ServletContextResource(this.servletContext, "/");
                registration.addResourceLocations(new Resource[]{resource});
              }

            });
          }            
        }
  //欢迎页映射配置      
  @Bean
  public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext applicationContext, FormattingConversionService mvcConversionService, ResourceUrlProvider mvcResourceUrlProvider) {}


  //为什么静态资源映射配置之后欢迎页失效
  WelcomePageHandlerMapping(TemplateAvailabilityProviders templateAvailabilityProviders, ApplicationContext applicationContext, Resource welcomePage, String staticPathPattern) {
    if (welcomePage != null && "/**".equals(staticPathPattern)) {
      logger.info("Adding welcome page: " + welcomePage);
      this.setRootViewName("forward:index.html");
    } else if (this.welcomeTemplateExists(templateAvailabilityProviders, applicationContext)) {
      logger.info("Adding welcome page template: index");
      this.setRootViewName("index");
    }
  }
}
```

###请求路径原理

#### 请求映射原理

* 核心类 dispatcherServlet

* HandlerMapping 将每个请求路径对应到相应的controller

* 进来的每个请求 都会调用ds里的doDispatch方法分发
  
  * 调用getHandler遍历查找找到请求由哪个handler处理
    
    * springboot自动配置了以下handlerMapping
      
      * RequestHandlerMapping 
      
      * WelcomePageHandlerMapping
      
      * ...
        
        ```
        protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        mappedHandler = this.getHandler(processedRequest);
        }
        ```
        
        #### RSETFUL风格

* put delete get post

* 核心filter HiddenHttpMethodFilter

* 用法 表单post 隐藏域 _method

* 需要手动开启 ```spring.mvc.hiddenmethod.filter.enabled=true```
  
  ```java
  //源码
  public class HiddenHttpMethodFilter extends OncePerRequestFilter {
  public static final String DEFAULT_METHOD_PARAM = "_method";
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
    HttpServletRequest requestToUse = request;
    if ("POST".equals(request.getMethod()) && request.getAttribute("javax.servlet.error.exception") == null) {
      String paramValue = request.getParameter(this.methodParam);
      if (StringUtils.hasLength(paramValue)) {
        String method = paramValue.toUpperCase(Locale.ENGLISH);
        if (ALLOWED_METHODS.contains(method)) {
            //封装一下请求 put delete请求由此诞生
          requestToUse = new HiddenHttpMethodFilter.HttpMethodRequestWrapper(request, method);
        }
      }
    }
  
    filterChain.doFilter((ServletRequest)requestToUse, response);
  }  
  }
  ```

####常用请求参数注解使用

* @RequestParam

* @PathVariable 如果不指定参数 可以返回一个包含所有路径参数的map

* @RequestHeader 不指定参数即所有请求头

* @CookieValue

* @RequestBody 返回post请求的请求体 也包含请求参数

* @RequestAttribute  取出请求域属性值的方法

* ***注意：request.getAttribute得到浏览器发来的参数 getParameter得到一些固有的参数属性***
  
  #### 一些其他的请求参数的类型

* Model Map 相当于是request的所有属性，给它们添加内容相当于调用request.setAttribute方法

* RedirectAttributes 重定向携带数据

* HttpServletRequest

* HttpServletResponse

* 封装为一个对象传入
  
  #### urlPathHelper
  
  可以解析路径中的参数，保存到请求的attribute中
  
  #### 请求参数解析绑定原理
  
  - HandlerMapping中先找一个能够处理请求的handler
  
  - 为当前handler找一个适配器（handleAdapter）一个反射工具
    
    ```java
    public interface HandlerAdapter {
    boolean supports(Object handler);
    
    @Nullable
    ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
    }
    ```
    
    - 添加参数解析器
      
      - 作用：确定将要执行的方法的参数的值
      - MVC目标方法能支持多少种参数，取决于参数解析器
    
    - 添加返回值处理器
    
    - 执行请求方法（通过遍历参数解析器，获取支持的解析器，然后确定好参数的值后，调用controller中的方法）获得返回值
    
    - 完成方法后，更新ModelAndViewContainer
      
      ```java
      public interface HandlerMethodArgumentResolver { // 参数解析器
      //是否支持
      boolean supportsParameter(MethodParameter parameter);
      ```
    
    @Nullable
    //解析方法
    Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
    }
    ```
    
    #### 数据类型的转化
- 通过各种converter来实现文本类型到各种数据类型的转换

- 自定义converter 已实现自定义类型转换
  
  ```java
  @Configuration
  public class MVCConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        //添加自定义的参数类型转换器
        registry.addConverter(new Converter<String, Pet>() {
            @Override
            public Pet convert(String source) {
                //处理规则
                return null;
            }
        });
    }
  }
  ```

### 响应处理

#### @RestController 或者 @ResponseBody

- 使用该注解，可以把对象转换成浏览器支持的格式返回

- 因为springboot自动引入了json启动器，所以自动返回json数据

- 原理：
  
  - 返回值解析器 // 这也是springboot默认支持的返回值类型
    ![img.png](img.png)
  
  - 确定处理返回值的处理器
  
  - 调用特定的处理器来处理 RequestResponseBodyMethodMethodProcessor
    
    - 使用消息转换器来进行写出操作  HttpMessageConverter
    
    - 根据内容协商与服务器自身，觉得生成什么内容类型的数据
    
    - springMVC遍历messageConverter，确定哪一个能够处理
    
    - Java对象都能通过MappingJackson2HttpMessageConverter转化为json格式
      
      ```java
      public interface HttpMessageConverter<T> {
      //类型是否支持读入
      boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);
      //类型是否支持写出
      boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);
      ```
  
  // 设置支持的数据类型
  List<MediaType> getSupportedMediaTypes();
  
  //读入
  T read(Class<? extends T> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException;
  //写出
  void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException;
  }
  
  ```
  #### 内容协商
  ```

- 根据客户端接收能力不同，返回不同媒体类型的数据

- 通过改变浏览器中的accept的值，就可以让服务端返回不同类型的数据（如果要让服务器返回xml 需要导入 fastxml）
  
  ```xml
      <dependency>
          <groupId>com.fasterxml.jackson.dataformat</groupId>
          <artifactId>jackson-dataformat-xml</artifactId>
      </dependency>
  ```

- 可以开启基于请求参数的内容协商 ``spring.mvc.contentnegotiation.favor-parameter=true`` 请求参数中的format指定返回的类型
  
  #### 自定义内容协商
  
  **核心：messageConverter 处理各种不同数据**
1. 添加一个自定义的converter

2. 系统底层统计出所有converter能操作哪些类型
   
   ```java
   @Configuration
   public class MVCConfig implements WebMvcConfigurer {
   
   @Override
   public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
      converters.add(new MyFormConverter());
   }
   }
   ```

public class MyFormConverter implements HttpMessageConverter<Person> {
  @Override
  // 能否读成某个类型
  public boolean canRead(Class<?> clazz, MediaType mediaType) {
    return false;
  }

  @Override
  public boolean canWrite(Class<?> clazz, MediaType mediaType) {
    //如果是person类型的 则可以写入
    return clazz.isAssignableFrom(Person.class);
  }

  @Override
  public List<MediaType> getSupportedMediaTypes() {
    return MediaType.parseMediaTypes("application/myform");
  }

  @Override
  public Person read(Class<? extends Person> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
    return null;
  }

  @Override
  public void write(Person person, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
    String data = person.getName()+"/"+person.getAge()+"/"+person.getDate();
    OutputStream out = outputMessage.getBody();
    // 解决中文乱码
    out.write(data.getBytes("GBK"));
  }
}

```
3. 如果要支持基于自定义格式的参数的内容协商
```java
@Configuration
public class MVCConfig implements WebMvcConfigurer {
  @Override
  public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
      converters.add(new MyFormConverter());
  }
  @Override
  public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
      //  方式一 直接注册一个新的mediaType
//        configurer.mediaType("myform", MediaType.parseMediaType("application/myform"));
      // 方式二 覆盖策略 但是要注意 会覆盖掉原来的默认值 所以需要增加原来的默认
      Map<String,MediaType> map = new HashMap<>();
      map.put("json",MediaType.APPLICATION_JSON);
      map.put("xml",MediaType.APPLICATION_XML);
      map.put("myform",MediaType.parseMediaType("application/myform"));
      ParameterContentNegotiationStrategy negotiationStrategy = new ParameterContentNegotiationStrategy(map);
      // 这个默认的也要添加 否则会被覆盖而失效
      HeaderContentNegotiationStrategy headerContentNegotiationStrategy = new HeaderContentNegotiationStrategy();
      configurer.strategies(Arrays.asList(negotiationStrategy,headerContentNegotiationStrategy));
  }
}
```

### 视图解析

- 返回值通过ViewNameMethodReturnValueHandler处理
- 方法执行过程中，所有数据与视图地址都会放在modelAndViewContainer中存放(Model ViewAddress 如果controller方法参数是一个自定义类型对象，也会放在mv中)
- 任何方法执行完成后返回modelAndView对象（数据与视图地址）
- 处理派发结果 决定页面该如何相应 ```processDispatchResult()```
  - render(mv,request,response) 渲染页面
    - 先得到一个View对象(定义了解析视图的逻辑) 由视图解析器获得
    - view.render()
    - 模板引擎渲染

### 拦截器

#### 配置过程

- 实现HandlerInterceptor接口

- 注册拦截器
  
  - 重写WebMvcConfig的方法
    
    ```java
    @Configuration
    public class MVCConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new MyInterceptor()).addPathPatterns("/**").excludePathPatterns("/a");
    }
    }
    ```
    
    #### 基本原理
1. 根据当前请求，找到可以处理请求的handler的所有拦截器
2. 按照顺序执行所有拦截器的preHandle方法
   1. 如果当前拦截器的prehandle返回true 则执行下一个
   2. 否则倒序执行所有拦截器的afterCompletion
3. 如果任何一个拦截器执行失败，则不执行目标方法
4. 否则执行目标方法
5. 方法执行完成后，倒序执行所有拦截器的postHandle方法
6. 如果一切正常，进入页面渲染逻辑
7. 否则都会触发afterCompletion（倒序触发）
8. 页面渲染成功之后，也会执行afterCompletion

### 文件上传

处理前端通过post请求传过来的文件

- 前端需要的格式：
  
  ```html
  <form enctype="multipart/form-data">
    <input type="file" name="f1" multiple>
    <input type="file" name="f2">
    <button type="submit">提交</button>
  </form>
  ```

- 后端需要的格式：
  
  ```java
    @PostMapping("/ssdd")
    @ResponseBody
    public String get4(@RequestParam("f1")MultipartFile[] multipartFiles, @RequestParam("f2")MultipartFile multipartFile) throws IOException {
        multipartFile.transferTo(new File("path"));
        return "ok";
    }
  ```

### 错误处理

- 在请求发生错误时，springboot会返回给浏览器或者其他发送者错误信息
- 在static下error/ 下可以写几个关于错误码的html 替换白页

### 注入原生servlet组件

方式一： @ServletComponentScan+@WebServlet 
方式二： 配置类注入servlet

> 如果需要注册Spring Boot三大组件（Servlet、Filter、Listener）就必须依赖于Spring Boot为我们提供的对应的配置类。
>  ● ServletRegistrationBean                     Servlet注册Bean 
>  ● FilterRegistrationBean                        Filter注册Bean 
>  ● ServletListenerRegistrationBean        Listener注册Bean

```java
// 创建自定义MyServlet类，实现定制的Servlet功能
public class MyServlet extends HttpServlet {
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        // 定制返回消息
        resp.getWriter().write("Spring Boot, Hello MyServlet...");
    }
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        // get请求直接转发到post
        doPost(req, resp);
    }
}

// 创建配置类，注入ServletRegistrationBean
@Configuration
public class MyServerConfiguration {
    @Bean  // 必须注入到容器才能生效
    public ServletRegistrationBean myServlet(){
        // 引入自定义Servlet，并绑定请求映射uri
        return new ServletRegistrationBean(new MyServlet(), "/helloMyServlet");
    }
}
```

```java
// 创建自定义的Filter类，实现自定义过滤功能
public class MyFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, 
                         FilterChain filterChain) throws IOException, ServletException {
        System.out.println("Spring Boot, Hello MyFilter...");
        // 处理完成后，继续向下执行
        filterChain.doFilter(servletRequest, servletResponse);
    }
}

// 创建配置类，注入FilterRegistrantionBean
@Configuration
public class MyServerConfiguration {
    @Bean   // 必须注入到容器才能生效
    public FilterRegistrationBean myFiler(){
        FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
        // 引入自定义Filter
        filterRegistrationBean.setFilter(new MyFilter());
        // 指明要拦截过滤的请求
        filterRegistrationBean.setUrlPatterns(Arrays.asList("/myFilter", "/helloMyFilter"));
        return filterRegistrationBean;
    }
}
```

```java
// 创建自定义的Listener类，实现自定义监听功能
public class MyListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println("contextInitialized...  WEB应用启动了！");
    }
    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println("contextDestroyed...  当前WEB应用销毁了！");
    }
}

// 创建配置类，注入ServletListenerRegistrationBean
@Configuration
public class MyServerConfiguration {
    @Bean
    public ServletListenerRegistrationBean myListener(){
        ServletListenerRegistrationBean<MyListener> listener = 
            new ServletListenerRegistrationBean<>(new MyListener());
        return listener;
    }
}
```

### 切换web服务器与定制化

- springBoot应用启动时发现当前是Web应用，会导入web场景包-导入tomcat
- web应用会创建一个web版的IOC容器 ServletWebServerApplicationContext
- ServletWebServerApplicationContext 启动时寻找WebServer工厂 **TomcatServletWebServerFactory、JettyServletWebServerFactory..**
- 底层会有一个自动配置类ServletWebServerFactoryAutoConfiguration 动态判断导入的web包，创建出web服务器

#### 切换web服务器

- 导入对应的web服务器starter

- 在springboot-starter中排除tomcat
  
  #### 定制servlet容器
1. 修改配置文件
2. 直接自定义ConfigurableServletWebServerFactory
3. 实现WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>

### 定制化总结

- 修改配置文件
- 给容器中添加组件（@Component/@Configuration+@Bean）
- 定制化web功能：实现WebMvcConfigurer接口
- xxxCustomizer 定制化器
- 全面接管： @EnableWebMvc
- ...

### springboot 用法记录

#### Spring的@Transactional注解使用方法

声明式事务，对于RUNTIMEEXCEPTION 和 ERROR的异常会回滚 其他不会回滚 可以通过`rollbackFor 设置回滚的异常类型`

对于私有方法、同一个类中的方法调用事务也会失效

#### Spring的@ExceptionHandler注解使用方法

Spring的@ExceptionHandler可以用来统一处理方法抛出的异常，比如这样：

```
@ExceptionHandler()
public String handleExeption2(Exception ex) {
    System.out.println("抛异常了:" + ex);
    ex.printStackTrace();
    String resultStr = "异常：默认";
    return resultStr;
}
```

当我们使用这个@ExceptionHandler注解时，我们需要定义一个异常的处理方法，比如上面的handleExeption2()方法，给这个方法加上@ExceptionHandler注解，这个方法就会处理类中其他方法（被@RequestMapping注解）抛出的异常。

###### 注解的参数

@ExceptionHandler注解中可以添加参数，参数是某个异常类的class，代表这个方法专门处理该类异常，比如这样：

```
@ExceptionHandler(NumberFormatException.class)
public String handleExeption(Exception ex) {
    System.out.println("抛异常了:" + ex);
    ex.printStackTrace();
    String resultStr = "异常：NumberFormatException";
    return resultStr;
}
```

此时注解的参数是NumberFormatException.class，表示只有方法抛出NumberFormatException时，才会调用该方法。

###### 就近原则

当异常发生时，Spring会选择最接近抛出异常的处理方法。

比如之前提到的NumberFormatException，这个异常有父类RuntimeException，RuntimeException还有父类Exception，如果我们分别定义异常处理方法，@ExceptionHandler分别使用这三个异常作为参数，比如这样：

```
@ExceptionHandler(NumberFormatException.class)
public String handleExeption(Exception ex) {
    System.out.println("抛异常了:" + ex);
    ex.printStackTrace();
    String resultStr = "异常：NumberFormatException";
    return resultStr;
}

@ExceptionHandler()
public String handleExeption2(Exception ex) {
    System.out.println("抛异常了:" + ex);
    ex.printStackTrace();
    String resultStr = "异常：默认";
    return resultStr;
}

@ExceptionHandler(RuntimeException.class)
public String handleExeption3(Exception ex) {
    System.out.println("抛异常了:" + ex);
    ex.printStackTrace();
    String resultStr = "异常：RuntimeException";
    return resultStr;
}
```

那么，当代码抛出NumberFormatException时，调用的方法将是注解参数NumberFormatException.class的方法，也就是handleExeption()，而当代码抛出IndexOutOfBoundsException时，调用的方法将是注解参数RuntimeException的方法，也就是handleExeption3()。

###### 注解方法的返回值

标识了@ExceptionHandler注解的方法，返回值类型和标识了@RequestMapping的方法是统一的，可参见@RequestMapping的说明，比如默认返回Spring的ModelAndView对象，也可以返回String，这时的String是ModelAndView的路径，而不是字符串本身。

有些情况下我们会给标识了@RequestMapping的方法添加@ResponseBody，比如使用Ajax的场景，直接返回字符串，异常处理类也可以如此操作，添加@ResponseBody注解后，可以直接返回字符串，比如这样：

```
@ExceptionHandler(NumberFormatException.class)
@ResponseBody
public String handleExeption(Exception ex) {
    System.out.println("抛异常了:" + ex);
    ex.printStackTrace();
    String resultStr = "异常：NumberFormatException";
    return resultStr;
}
```

这样的操作可以在执行完方法后直接返回字符串本身。

###### 错误的操作

使用@ExceptionHandler时尽量不要使用相同的注解参数。

如果我们定义两个处理相同异常的处理方法：

```
@ExceptionHandler(NumberFormatException.class)
@ResponseBody
public String handleExeption(Exception ex) {
    System.out.println("抛异常了:" + ex);
    ex.printStackTrace();
    String resultStr = "异常：NumberFormatException";
    return resultStr;
}

@ExceptionHandler(NumberFormatException.class)
@ResponseBody
public String handleExeption2(Exception ex) {
    System.out.println("抛异常了:" + ex);
    ex.printStackTrace();
    String resultStr = "异常：默认";
    return resultStr;
}
```

#### 全局异常处理

```
@ControllerAdvice
public class GlobalExceptionController {
    /**
     * 全局异常捕捉处理
     * @param e
     * @return
     */
    @ResponseBody
    @ExceptionHandler(value = Exception.class)
    public Map errorHandler(Exception e) {
        System.out.println("用户异常调用测！");
        Map map = new HashMap();
        map.put("code", 100);
        map.put("msg", e.getMessage());
        return map;
    }

    /**
     * 拦截捕捉自定义异常 用户删除异常
     * @param ex
     * @return
     */
    @ResponseBody
    @ExceptionHandler(value = UserDeleteException.class)
    public AjaxResult UserDeleteException(UserDeleteException ex) {
        return AjaxResult.error(ex.getMessage());
    }

    /**
     * 拦截捕捉自定义异常 用户锁定异常
     * @param ex
     * @return
     */
    @ResponseBody
    @ExceptionHandler(value = UserBlockedException.class)
    public AjaxResult UserBlockedException(UserBlockedException ex) {
        System.out.println("用户锁定异常调用！");
        return AjaxResult.error(ex.getMessage());
    }
}
```

#### @InitBind注解的使用方法

标注在方法上，可以对请求参数进行预处理

* @InitBinder 标识的方法的参数通常是 WebDataBinder。
* @InitBinder 标识的方法,可以对 WebDataBinder 进行初始化。WebDataBinder 是 DataBinder 的一个子类,用于完成由表单字段到 JavaBean 属性的绑定。
* @InitBinder 标识的方法不能有返回值,必须声明为void。

##### 用途：

###### 参数处理

```
@ControllerAdvice
public class MyGlobalHandler {

    @InitBinder
    public void processParam(WebDataBinder dataBinder){

        /*
         * 创建一个字符串微调编辑器
         * 参数{boolean emptyAsNull}: 是否把空字符串("")视为 null
         */
        StringTrimmerEditor trimmerEditor = new StringTrimmerEditor(true);

        /*
         * 注册自定义编辑器 会对指定类型参数进行处理
         * 接受两个参数{Class<?> requiredType, PropertyEditor propertyEditor} 
         * requiredType：所需处理的类型
         * propertyEditor：属性编辑器，StringTrimmerEditor就是 propertyEditor的一个子类
         */
        dataBinder.registerCustomEditor(String.class, trimmerEditor);

        binder.registerCustomEditor(Date.class,
                new CustomDateEditor(new 
                // 这个也是propertyEditor的一个子类
                SimpleDateFormat("yyyy-MM-dd"), false));
    }
}
```

###### 参数绑定

对参数进行绑定，常用于封装对象

```
@ControllerAdvice
public class MyGlobalHandler {

    /*
     * @InitBinder("person") 对应找到@RequstMapping标识的方法参数中
     * 找参数名为person的参数。
     * 在进行参数绑定的时候，以‘p.’开头的都绑定到名为person的参数中。
     */
    @InitBinder("person")
    public void BindPerson(WebDataBinder dataBinder){
        dataBinder.setFieldDefaultPrefix("p.");
    }

    @InitBinder("book")
    public void BindBook(WebDataBinder dataBinder){
        dataBinder.setFieldDefaultPrefix("b.");
    }
}
```

### 拦截器

![img](https://img-blog.csdn.net/20170626173214101?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQwMTcxMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

```
@Slf4j
public class TokenInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String token = request.getParameter("token");
        if (token == null || JwtUtil.verify(token)){
            Response responses = new Response();
            responses.setStatus_code(400);
            responses.setStatus_msg("你还没有登录哦");
            response.getWriter().print(responses);
            return false;
        }
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
       log.info("postHandle执行");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        log.info("afterCompletion执行");
    }
}
```

```
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new TokenInterceptor()).addPathPatterns("/**").excludePathPatterns("/douyin/feed","/douyin/user/register","/douyin/user/login");
    }
```

#### 拦截器优先级

preHandle 方法是按注册顺序进行执行的，而 postHandle 和 afterCompletion 跟注册顺序是相反的。

### 数据访问

#### jdbc场景

![img_1.png](img_1.png)

#### 整合Druid数据源

方式一： 手动配置

```xml
<dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>druid</artifactId>
      <version>1.1.21</version>
</dependency>
```

1. 数据源配置文件 通过configurationProperties注解绑定到数据源上
   
   ```yaml
   spring:
   datasource:
    username: root
    password: root
    url: jdbc:mysql://127.0.0.1:3306/test
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource   ## 引入Druid数据源
   
    ## 数据源其他配置
    initialSize: 5
    minIdle: 5
    maxActive: 20
    maxWait: 60000
    timeBetweenEvictionRunsMillis: 60000
    minEvictableIdleTimeMillis: 300000
    validationQuery: SELECT 1 FROM DUAL
    testWhileIdle: true
    testOnBorrow: false
    testOnReturn: false
    poolPreparedStatements: true
    ## 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
    filters: stat,wall
    maxPoolPreparedStatementPerConnectionSize: 20
    useGlobalDataSourceStat: true
    connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=500
   ```
   
   2.写配置类
   ```java
   /**
* Druid数据配置类
  **/
  @Configuration
  public class MyDruidConfiguration {
  
    /**
  
  * 配置自定义数据源
  
  * @return
    */
    @Bean
    // 导入配置文件的配置
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource myDruidDataSource(){
     return new DruidDataSource();
    }
    
    /**
  
  * 配置Druid的监控管理后台 拦截器对其无效
  
  * @return
    */
    @Bean
    public ServletRegistrationBean druidServletRegistrationBean(){
     ServletRegistrationBean bean = new ServletRegistrationBean(new StatViewServlet(), "/druid/*");
    
     // 设置权限
     Map<String, String> initParams = new HashMap<>();
     initParams.put("loginUsername", "admin");
     initParams.put("loginPassword", "admin");
     initParams.put("allow", ""); // 默认允许所有访问
     initParams.put("deny", "");
    
     bean.setInitParameters(initParams);
     return bean;
    }
    
    /**
  
  * 配置一个web监控的filter 
  
  * @return
    */
    @Bean
    public FilterRegistrationBean webStatFilter(){
     FilterRegistrationBean bean = new FilterRegistrationBean();
     bean.setFilter(new WebStatFilter());
    
     Map initParams = new HashMap();
     initParams.put("exclusions", "*.js,*.css,/druid/*");
     bean.setInitParameters(initParams);
     bean.setUrlPatterns(Arrays.asList("/*"));
     return bean;
    }
    }
    
    ```
    方式二：自动配置
    使用starter
    常用配置
    ```yaml
    spring:
    datasource:
    url: jdbc:mysql://localhost:3306/db_account
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver
    
    druid:
    aop-patterns: com.atguigu.admin.*  #监控SpringBean
    filters: stat,wall     # 底层开启功能，stat（sql监控），wall（防火墙）
    
    stat-view-servlet:   # 配置监控页功能
     enabled: true
     login-username: admin
     login-password: admin
     resetEnable: false
    
    web-stat-filter:  # 监控web
     enabled: true
     urlPattern: /*
     exclusions: '*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*'
    ```
    
      filter:
    
        stat:    # 对上面filters里面的stat的详细配置
          slow-sql-millis: 1000
          logSlowSql: true
          enabled: true
        wall:
          enabled: true
          config:
            drop-table-allow: false

```
### 整合Mybatis
```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.4</version>
</dependency>
```

- 可以不写全局配置文件了 都可以通过yml方式配置
1. 编写实体类与mapper接口 +@Mapper注解 / 或者在启动类上加@MapperScan扫描包

2. 编写Sql映射文件并绑定mapper接口

3. 配置文件中知道mapper位置
- 注解实现 适合简单查询
  - @Option 可以设置如自增主键等选项

### 整合redis

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

![img_2.png](img_2.png)
自动配置：

- RedisAutoConfiguration 自动配置类。RedisProperties 属性类 --> spring.redis.xxx是对redis的配置

- 连接工厂是准备好的。LettuceConnectionConfiguration、JedisConnectionConfiguration

- 自动注入了RedisTemplate<Object, Object> ： xxxTemplate；

- 自动注入了StringRedisTemplate；k：v都是String

- key：value

- 底层只要我们使用 StringRedisTemplate、RedisTemplate就可以操作redis
  
  ```java
    @Test
    void testRedis(){
        ValueOperations<String, String> operations = redisTemplate.opsForValue();
  
        redisTemplate.opsForValue();//操作字符串 
        redisTemplate.opsForHash();//操作hash
        redisTemplate.opsForList();//操作list
        redisTemplate.opsForSet();//操作set
        redisTemplate.opsForZSet();//操作有序set
  
        operations.set("hello","world");
  
        String hello = operations.get("hello");
        System.out.println(hello);
    }
  ```

- 默认使用lettuce客户端 更改方式如下
  
  ```xml
  <!--        导入jedis-->
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>
  ```
  
  ```yaml
  spring:
  redis:
      host: r-bp1nc7reqesxisgxpipd.redis.rds.aliyuncs.com
      port: 6379
      password: lfy:Lfy123456
      client-type: jedis # 如果使用jedis客户端
      jedis:
        pool:
          max-active: 10
  ```

### 热更新 依赖(待补充)
