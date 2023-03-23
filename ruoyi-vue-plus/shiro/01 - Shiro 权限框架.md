# Shiro 权限框架

## 资料

- [若依文档介绍](https://doc.ruoyi.vip/ruoyi/document/xmjs.html#shiro%E5%AE%89%E5%85%A8%E6%8E%A7%E5%88%B6)
- [Shiro官方文档](https://shiro.apache.org/10-minute-tutorial.html)

↑ Shiro 文档的链接是快速开始的官方介绍，关于框架的集成其实看若依的文档会比较容易理解。

## 若依架构
#### 1. 安全管理器 SecurityManager

> Subject主体，代表了当前的“用户”，这个用户不一定是一个具体的人，与当前应用交互的任何东西都是Subject，如网络爬虫， 机器人等；即一个抽象概念；所有Subject都绑定到SercurityManager，与Subject的所有交互都会委托给SecurityManager；可以把Subject认为是一个门面；SecurityManager才是实际的执行者
SecurityManage安全管理器；即所有与安全有关的操作都会与SecurityManager交互；且它管理着所有Subject； 可以看出它是Shiro的核心，它负责与后边介绍的其他组件进行交互

安全管理器 SecurityManager (com.ruoyi.framework.config.ShiroConfig)

```java
/**
 * 安全管理器
 */
@Bean
public SecurityManager securityManager(UserRealm userRealm)
{
    DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
    // 设置realm.
    securityManager.setRealm(userRealm);
    // 记住我
    securityManager.setRememberMeManager(rememberMeManager());
    // 注入缓存管理器;
    securityManager.setCacheManager(getEhCacheManager());
    // session管理器
    securityManager.setSessionManager(sessionManager());
    return securityManager;
}
```
若依框架中 SecurityManager 主要参数有 Realm、RememberMe、CacheManager、SessionManager 等。

#### 2. Realm

> Realm域，Shiro从Realm获取安全数据（如用户，角色，权限），就是说SecurityManager要验证用户身份， 那么它需要从Realm获取相应的用户进行比较以确定用户身份是否合法；也需要从Realm得到用户相应的角色/权限进行验证用户是否能进行操作；可以有1个或多个Realm，我们一般在应用中都需要实现自己的Realm

若依框架中也实现了自己的 Realm (com.ruoyi.framework.shiro.realm.UserRealm) ，在自定义Realm中主要重写了两个方法 doGetAuthorizationInfo (授权) ，doGetAuthenticationInfo (登录认证) 。

并且将 Realm 加入了缓存管理器中 (com.ruoyi.framework.config.ShiroConfig) 。

```java
/**
 * 自定义Realm
 */
@Bean
public UserRealm userRealm(EhCacheManager cacheManager)
{
    UserRealm userRealm = new UserRealm();
    userRealm.setAuthorizationCacheName(Constants.SYS_AUTH_CACHE);
    userRealm.setCacheManager(cacheManager);
    return userRealm;
}
```
#### 3. SessionManager、SessionDAO、SessionFactory

> SessionManager如果写过Servlet就应该知道Session的概念，Session需要有人去管理它的生命周期，这个组件就是SessionManager
SessionDAODAO大家都用过，数据库访问对象，用于会话的CRUD，比如我们想把Session保存到数据库，那么可以实现自己的SessionDAO，也可以写入缓存，以提高性能

SessionManager (com.ruoyi.framework.config.ShiroConfig)

```java
/**
 * 会话管理器
 */
@Bean
public OnlineWebSessionManager sessionManager()
{
    OnlineWebSessionManager manager = new OnlineWebSessionManager();
    // 加入缓存管理器
    manager.setCacheManager(getEhCacheManager());
    // 删除过期的session
    manager.setDeleteInvalidSessions(true);
    // 设置全局session超时时间
    manager.setGlobalSessionTimeout(expireTime * 60 * 1000);
    // 去掉 JSESSIONID
    manager.setSessionIdUrlRewritingEnabled(false);
    // 定义要使用的无效的Session定时调度器
    manager.setSessionValidationScheduler(SpringUtils.getBean(SpringSessionValidationScheduler.class));
    // 是否定时检查session
    manager.setSessionValidationSchedulerEnabled(true);
    // 自定义SessionDao
    manager.setSessionDAO(sessionDAO());
    // 自定义sessionFactory
    manager.setSessionFactory(sessionFactory());
    return manager;
}
```
SessionFactory (com.ruoyi.framework.config.ShiroConfig)

```java
/**
 * 自定义sessionFactory会话
 */
@Bean
public OnlineSessionFactory sessionFactory()
{
    OnlineSessionFactory sessionFactory = new OnlineSessionFactory();
    return sessionFactory;
}
```
自定义sessionFactory会话 OnlineSessionFactory (`com.ruoyi.framework.shiro.session.OnlineSessionFactory`)
主要是从请求中获取需要的信息保存到自定义对象 OnlineSession 中。

SessionDAO (`com.ruoyi.framework.config.ShiroConfig`)

```java
/**
 * 自定义sessionDAO会话
 */
@Bean
public OnlineSessionDAO sessionDAO()
{
    OnlineSessionDAO sessionDAO = new OnlineSessionDAO();
    return sessionDAO;
}
```
OnlineSessionDAO (`com.ruoyi.framework.shiro.session.OnlineSessionDAO`) 针对自定义的ShiroSession的db操作。

#### 4. 缓存管理器 CacheManager

> CacheManager缓存控制器，来管理如用户，角色，权限等的缓存的；因为这些数据基本上很少去改变，放到缓存中后可以提高访问的性能

EhCacheManager (`com.ruoyi.framework.config.ShiroConfig`)
若依框架中缓存配置使用 EhCacheManager ，配置文件为 ehcache-shiro.xml 。

```java
/**
 * 缓存管理器 使用Ehcache实现
 */
@Bean
public EhCacheManager getEhCacheManager()
{
    net.sf.ehcache.CacheManager cacheManager = net.sf.ehcache.CacheManager.getCacheManager("ruoyi");
    EhCacheManager em = new EhCacheManager();
    if (StringUtils.isNull(cacheManager))
    {
        em.setCacheManager(new net.sf.ehcache.CacheManager(getCacheManagerConfigFileInputStream()));
        return em;
    }
    else
    {
        em.setCacheManager(cacheManager);
        return em;
    }
}

/**
 * 返回配置文件流 避免ehcache配置文件一直被占用，无法完全销毁项目重新部署
 */
protected InputStream getCacheManagerConfigFileInputStream()
{
    // shiro 缓存配置文件路径
    String configFile = "classpath:ehcache/ehcache-shiro.xml";
    InputStream inputStream = null;
    try
    {
        inputStream = ResourceUtils.getInputStreamForPath(configFile);
        byte[] b = IOUtils.toByteArray(inputStream);
        InputStream in = new ByteArrayInputStream(b);
        return in;
    }
    catch (IOException e)
    {
        throw new ConfigurationException(
                "Unable to obtain input stream for cacheManagerConfigFile [" + configFile + "]", e);
    }
    finally
    {
        IOUtils.closeQuietly(inputStream);
    }
}
```
#### 5. Shiro过滤器配置 ShiroFilterFactoryBean
ShiroFilterFactoryBean (com.ruoyi.framework.config.ShiroConfig)

```java
/**
 * Shiro过滤器配置
 */
@Bean
public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager)
{
    ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
    // Shiro的核心安全接口,这个属性是必须的
    shiroFilterFactoryBean.setSecurityManager(securityManager);
    // 身份认证失败，则跳转到登录页面的配置
    shiroFilterFactoryBean.setLoginUrl(loginUrl);
    // 权限认证失败，则跳转到指定页面
    shiroFilterFactoryBean.setUnauthorizedUrl(unauthorizedUrl);
    // Shiro连接约束配置，即过滤链的定义
    LinkedHashMap<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
    // 对静态资源设置匿名访问
    filterChainDefinitionMap.put("/favicon.ico**", "anon");
    filterChainDefinitionMap.put("/ruoyi.png**", "anon");
    filterChainDefinitionMap.put("/html/**", "anon");
    filterChainDefinitionMap.put("/css/**", "anon");
    filterChainDefinitionMap.put("/docs/**", "anon");
    filterChainDefinitionMap.put("/fonts/**", "anon");
    filterChainDefinitionMap.put("/img/**", "anon");
    filterChainDefinitionMap.put("/ajax/**", "anon");
    filterChainDefinitionMap.put("/js/**", "anon");
    filterChainDefinitionMap.put("/ruoyi/**", "anon");
    filterChainDefinitionMap.put("/captcha/captchaImage**", "anon");
    // 退出 logout地址，shiro去清除session
    filterChainDefinitionMap.put("/logout", "logout");
    // 不需要拦截的访问
    filterChainDefinitionMap.put("/login", "anon,captchaValidate");
    // 注册相关
    filterChainDefinitionMap.put("/register", "anon,captchaValidate");
    // 系统权限列表
    // filterChainDefinitionMap.putAll(SpringUtils.getBean(IMenuService.class).selectPermsAll());

    Map<String, Filter> filters = new LinkedHashMap<String, Filter>();
    // 自定义在线用户处理过滤器
    filters.put("onlineSession", onlineSessionFilter());
    // 自定义在线用户同步过滤器
    filters.put("syncOnlineSession", syncOnlineSessionFilter());
    // 自定义验证码过滤器
    filters.put("captchaValidate", captchaValidateFilter());
    // 同一个用户多设备登录限制
    filters.put("kickout", kickoutSessionFilter());
    // 注销成功，则跳转到指定页面
    filters.put("logout", logoutFilter());
    shiroFilterFactoryBean.setFilters(filters);

    // 所有请求需要认证
    filterChainDefinitionMap.put("/**", "user,kickout,onlineSession,syncOnlineSession");
    shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);

    return shiroFilterFactoryBean;
}
```
