## 基本介绍
![[Pasted image 20230901105611.png]]

Spring Security是Spring项目组提供的安全服务框架，提供了身份验证、授权和防范常见攻击的功能。它为系统提供了声明式安全访问控制功能，减少了 为系统安全而编写大量重复代码的工作。

## 基本结构
整体架构的介绍
有学习过，在客户端发起的请求进入servlet之前，会先经过过滤器链(FilterChain)，FilterChain包含了一系列的过滤器以及最终处理请求的servlet（基于请求的uri），在spring mvc应用中，这个servlet其实就是DispatcherServlet，它可以将请求分发给对应的Controller进行处理。过滤器链上的过滤器会依次对请求进行处理，过滤器在进行了自己的逻辑之后如果让请求通过，则会调用doFilter方法，放行请求，让下一个过滤器处理。如果不通过，则下游的过滤器不会继续处理请求
```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) { 
	// do something before the rest of the application 
	chain.doFilter(request, response); // invoke the rest of the application 
	// do something after the rest of the application 
}
```
![[Pasted image 20230831145355.png]]

在Spring中，如果想让一个Filter交由Spring容器管理，一种方法是加上@WebFilter@Componet注解。还可以将这个Filter的bean注册给DelegatingFilterProxy，DelegatingFilterProxy其实也是个Filter,但是它内部有一个delegate对象，这个对象是从Spring的容器中获取的Filter对象，而实际的过滤逻辑，也是委托给这个对象进行的。
![[Pasted image 20230901105704.png]]

在SpringSecurity中有非常多的过滤器，并且根据配置的不同，每个请求需要经过的过滤器也可能是不同的，那要怎么样才能将这些过滤器加入到整体的过滤器链中呢？

SpringSecurity使用了FilterChainProxy来代理所有SpringSecurity的过滤器。FilterChainProxy是一个特殊的过滤器，它包含了若干SecurityFilterChain，能够将请求委托给对应的SecurityFilterChain进行处理。由于FilterChainProxy是一个Bean，所有wrap在了DelegatingFilterProxy中
![[Pasted image 20230831150047.png]]

SecurityFilterChain是一个接口，被FilterChainProxy使用，来决定最终调用哪些filters
```java
public interface SecurityFilterChain {  
  
    boolean matches(HttpServletRequest request);  
  
    List<Filter> getFilters();  
  
}
```

![[Pasted image 20230831150804.png]]

![[Pasted image 20230831151709.png]]
为什么要使用FilterChainProxy来代理所有的Security Filters
- FilterChainProxy可以提供一个开始点
- 作为Security执行的中心，可以提取出一些必要的任务进行统一的执行，比如说，在每次请求结束之后清空SecurityContext来避免内存泄漏
- 可以让过滤器链的调用更加灵活。因为Servlet容器中Filters基于请求的Url进行调用，而这里，通过接口的match方法，可以实现更加丰富的匹配逻辑。

## Security Filters
下面是一个全面的SpringSecurity的过滤器的列表，并且过滤器的执行顺序也是如此

- ForceEagerSessionCreationFilter`
- ChannelProcessingFilter
- WebAsyncManagerIntegrationFilter
- SecurityContextPersistenceFilter
- HeaderWriterFilter
- CorsFilter
- CsrfFilter
- LogoutFilter
- OAuth2AuthorizationRequestRedirectFilter
- Saml2WebSsoAuthenticationRequestFilter
- X509AuthenticationFilter
- AbstractPreAuthenticatedProcessingFilter
- CasAuthenticationFilter
- OAuth2LoginAuthenticationFilter
- Saml2WebSsoAuthenticationFilter
- UsernamePasswordAuthenticationFilter
- OpenIDAuthenticationFilter
- DefaultLoginPageGeneratingFilter
- DefaultLogoutPageGeneratingFilter
- ConcurrentSessionFilter
- DigestAuthenticationFilter
- BearerTokenAuthenticationFilter
- BasicAuthenticationFilter
- RequestCacheAwareFilter
- SecurityContextHolderAwareRequestFilter
- JaasApiIntegrationFilter
- RememberMeAuthenticationFilter
- AnonymousAuthenticationFilter
- OAuth2AuthorizationCodeGrantFilter
- SessionManagementFilter
- ExceptionTranslationFilter
- FilterSecurityInterceptor
- SwitchUserFilter

可以看出共有30多个过滤器，相当的多，但是这些过滤器并不是全部都加入到过滤器链的，而是根据用户的配置与默认的配置，创建过滤器链，具体的逻辑...

- 扩展：csrf攻击
https://zhuanlan.zhihu.com/p/370796176

## 认证
认证就是去验证用户的身份。也就是对用户的请求中携带的身份信息进行核验，只有认证通过才能访问系统内的资源。
### SpringSecurity关于认证的几个组件
- [SecurityContextHolder](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html#servlet-authentication-securitycontextholder) - 存储认证用户的details
- [SecurityContext](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html#servlet-authentication-securitycontext) - 能够被SecurityContextHolder获取，存储了当前用户的认证信息Authentication
- [Authentication](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html#servlet-authentication-authentication) - 提供用户的凭证 
- [GrantedAuthority](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html#servlet-authentication-granted-authority) - 代表被授予Authentication中的principal的一种权限
- [AuthenticationManager](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html#servlet-authentication-authenticationmanager) - 定义了SpringSecurity的过滤器如何进行认证
- [ProviderManager](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html#servlet-authentication-providermanager) - the most common implementation of `AuthenticationManager`.
- [AuthenticationProvider](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html#servlet-authentication-authenticationprovider) - used by `ProviderManager` to perform a specific type of authentication.
- [Request Credentials with `AuthenticationEntryPoint`](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html#servlet-authentication-authenticationentrypoint) - used for requesting credentials from a client (i.e. redirecting to a login page, sending a `WWW-Authenticate` response, etc.)
- [AbstractAuthenticationProcessingFilter](https://docs.spring.io/spring-security/reference/5.7/servlet/authentication/architecture.html#servlet-authentication-abstractprocessingfilter) - a base `Filter` used for authentication. This also gives a good idea of the high level flow of authentication and how pieces work together.

### SecurityContextHolder
![[Pasted image 20230831095706.png]]

代表用户已经认证的方法：
```java
SecurityContext context = SecurityContextHolder.createEmptyContext(); Authentication authentication = new TestingAuthenticationToken("username", "password", "ROLE_USER"); context.setAuthentication(authentication); SecurityContextHolder.setContext(context);
```
获取用户信息的方法：
```java
SecurityContext context = SecurityContextHolder.getContext(); Authentication authentication = context.getAuthentication(); String username = authentication.getName(); Object principal = authentication.getPrincipal(); Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
```
默认SecurityContextHolder使用ThreadLocal变量来存储SecurityContext,所以每个线程获取到的context都是当前线程独有的。对于web应用来说，这样更好，因为会有很多个用户的请求。
对于一个桌面应用，比如swing开发的程序，我们希望每个线程获取到的都是同一个context，可以通过设置SecurityContextHolder的策略来实现

### Authentication

组成：
- principal: When authenticating with a username/password, often instance of userDetails 存储用户信息
- credentials：用户的凭据，一般来说是用户的密码，但是一般不设置
- authorities：用户的GrantedAuthority的集合

### GrantedAuthority

代表用户的权限(role scope ...)

### AuthenticationManager
是一个接口定义了Filters如何进行认证。返回的Authentication将被调用的Filter设置到SecurityContextHolder中。你也可以在过滤器中实现自己的认证方式，然后直接调用SecurityContextHolder.getContext.setAuth的方式，而不通过AuthenticationManager来进行认证。
![[Pasted image 20230831102419.png]]

###  ProviderManager

是AuthenticationManager的一个最常用到的实现类。

包含了一系列的AuthenticationProvider，每个provider用于进行不同类别的认证方式，比如username/password jwt... 但是对外只是暴露一个ProviderManager对象

![[Pasted image 20230831102717.png]]


### Request Credentials with `AuthenticationEntryPoint`
当需要用户进行凭证的输入时，AuthenticationEntryPoint可以用来发送一个response来请求用户输入凭证（重定向）


### AbstractAuthenticationProcessingFilter
是一个抽象的基类过滤器，用于对用户的凭证进行认证，子类过滤器重写
![[Pasted image 20230831110117.png]]

![[Pasted image 20230831110540.png]]



## 授权
不同用户可能具有不同的对系统的操作权限，而不同权限能够访问与操作的资源是不同的。因此在用户访问接口时，需要对用户的权限进行判断。

当进行认证时，`GrantedAuthority`也会被存入Authentication对象中，这将在之后被`AuthorizationManager`使用，从而做出授权决定

`GrantedAuthority`是一个接口，通过getAuthority返回的字符串来判断权限

###   The AuthorizationManager


AuthorizationManager被AuthorizationFilter调用，做出访问控制的最终决定

```java
@FunctionalInterface  
public interface AuthorizationManager<T> {  
	default void verify(Supplier<Authentication> authentication, T object) {  
	//...
	}  
	AuthorizationDecision check(Supplier<Authentication> authentication, T object);  
}
```
check方法定义了接口根据相关的信息进行access决定，做出的决定就是AuthorizationDecision对象，（true/false），object可以是传入包含了认证需要信息的对象。



## 如何处理Security Exceptions

`ExceptionTranslationFilter`处理了用户认证没有通过的情况
它的伪代码如下：
```java
try { 
	filterChain.doFilter(request, response); 
} catch (AccessDeniedException | AuthenticationException ex) { 
	if (!authenticated || ex instanceof AuthenticationException) { 
		startAuthentication(); 
	} 
	else { 
		accessDenied(); 
	} 
}
```
