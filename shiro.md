# shiro

## 核心组件

- UsernamePasswordToken  用来封装用户的登陆信息，使用用户的登陆信息来创建令牌Token
- SecurityManager 核心部分 负责安全认证和授权
- Suject 包含用户信息
- Realm 开发自定义的模块，根据项目需求，验证和授权逻辑全部写在realm中
- AuthenticationInfo 用户的角色信息集合，认证使用
- AuthorzationInfo 角色的权限集合 授权使用
- DefaultWebSecurityManger 安全管理器 自定义的Realm需要注入到这里才能生效
- ShiroFilterFactoryBean 过滤器工厂，shiro根据用户定义的规则生成一个个过滤器

## 角色、权限、用户

会给角色赋予权限，给用户赋予角色

## 基本实现

```
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring</artifactId>
    <version>1.9.0</version>
</dependency>
```

```
//先认证再授权
public class AccountRealm extends AuthorizingRealm {
    // 授权方法
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        // 获取当前登陆用户
        Subject subject = SecurityUtils.getSubject();
        Account account = (Account) subject.getPrincipal();
        // 进行授权
        Set<String> stringSet = new HashSet<>();
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo(stringSet);
        info.addStringPermission("admin");
        return info;
    }
    //认证方法
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        UsernamePasswordToken token = (UsernamePasswordToken) authenticationToken;
        Account account = new Account();
                                            //保存到对象中 通过getPrincipal获得
        return new SimpleAuthenticationInfo(account,account.getPassword(),getName());
    }
}
```

```
@Configuration
public class ShiroConfig {
    @Bean
    public AccountRealm accountRealm(){
        return new AccountRealm();
    }
    @Bean
    public DefaultWebSecurityManager webSecurityManager(){
        DefaultWebSecurityManager defaultWebSecurityManager = new DefaultWebSecurityManager();
        defaultWebSecurityManager.setRealm(accountRealm());
        return defaultWebSecurityManager;
    }
    @Bean
    public ShiroFilterFactoryBean factoryBean(){
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(webSecurityManager());

        //认证过滤器
        /**
         * anon 无需认证
         * authc 必须认证
         * authcBasic
         * user 只要曾经被shiro记录即可
         */
        Map<String,String> map = new HashMap<>();
        //必须登陆
        map.put("/index","anon");
        map.put("/login","anon");
        map.put("/test","anon");
        map.put("/**","authc");
        //登出
        map.put("/logout","logout");
        //必须有授权
        map.put("/admin","perms[manager]");
        map.put("/admin2","roles[admin]");
        //授权过滤器
        /***
         * perms 必须拥有某个权限
         * role 必须是某个角色
         * port 必须指定端口
         * ssl 必须是安全的 https
         */
        shiroFilterFactoryBean.setFilterChainDefinitionMap(map);


        return shiroFilterFactoryBean;
    }
}
```

```
@Controller
public class TestController {
    @GetMapping("/test")
    public String test(){
        Subject subject = SecurityUtils.getSubject();
        UsernamePasswordToken token = new UsernamePasswordToken("a","123");
        try {
            subject.login(token);
        }
        catch (UnknownAccountException e){
            System.out.println("无用户");
        }
        catch (IncorrectCredentialsException e){
            System.out.println("密码错误");
        }
        return "!";
    }
}
```

## JWT （json web token）

https://blog.csdn.net/weixin_45070175/article/details/118559272

三部分组成：

- Header 包含算法
- Payload 载荷 存放有效信息
- Sinature 签名