# Redisson（七）集成 Spring Cache 缓存分析

## 前言
先说句题外话，框架发布了 [V4.3.0 公测版本](https://gitee.com/JavaLionLi/RuoYi-Vue-Plus) 了，有兴趣的朋友可以去尝鲜。

最近没有更新不是在摆烂，而是在梳理 AOP 的相关内容，因为这篇博客的内容属实有点难搞，所以就去找了很多书籍、博客和视频去恶补相关的内容（其中有些很好的我会在下面的参考目录里面一一列举出来）。

**这篇博客可能更多的是偏向于个人学习的笔记，所以如果有错漏的地方烦请指正。**

## 参考目录
- [Redisson 官方文档 -  14.2. Spring Cache整合](https://github.com/redisson/redisson/wiki/14.-%E7%AC%AC%E4%B8%89%E6%96%B9%E6%A1%86%E6%9E%B6%E6%95%B4%E5%90%88#142-spring-cache%E6%95%B4%E5%90%88)
  提供了最简单的 Demo 演示如何整合 Spring Cache。
- [Spring 官方文档 - 8. Cache Abstraction](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache)
  遇事不决，官方文档。
- [木子旭 - java-sping-cache 专栏](https://www.cnblogs.com/bjlhx/category/1233985.html)
  关于 Spring Cache 的专栏，写得很全面。
- [叔牙 - Spring cache原理详解](https://juejin.cn/post/6959002694539444231#heading-0)
  关于 Spring Cache 的博客，写得挺好的。
- [代码重工 - 第三章Spring AOP](https://www.wolai.com/t997N89dHDoHWqz9LpiNxY)
  一个博客网站，里面还有别的一些实用的知识总结。
- Spring 5 高级编程（第5版）
- Spring揭秘（第2版）
  两本书也是查看了关于 Spring AOP 的相关内容。

**Redisson 缓存是整合了 Spring Cache，而 Spring Cache 的实现又是基于 Spring AOP 的**，因此个人觉得对于这一部分的学习需要夯实基础才能更好地学习和理解。

## 框架集成
### 1、Maven
总工程 `pom.xml`，Redisson 版本 `V3.17.5`<br>
![总工程 pom.xml](img07/496925da37c54e6b91b4ca53a1f6f895.png)<br>

公共模块 `pom.xml`<br>
![公共模块 pom.xml](img07/c3b8ad9e7933450690efcb4a6c1b942b.png)<br>
### 2、配置文件缓存配置
配置文件 `application.yml`<br>
![配置文件 application.yml](img07/125914dd6e7747cb8956187f9fd1987c.png)<br>
### 3、Redis配置类
`com.ruoyi.framework.config.RedisConfig`<br>
![Redis配置类 RedisConfig](img07/b0570bcb3a1a4f48b3c4d913990ecaeb.png)<br>

其中，注解 `@EnableCaching` 开启缓存功能。<br>

Spring Cache 配置 `RedisConfig#cacheManager`<br>
![Redis配置类 Spring Cache 配置](img07/0cb8acd413b94b44919d29b8b9cb5e3a.png)<br>

（红线告警无视即可，在 `RedissonAutoConfiguration` 自动装配时已经在容器中注入了 `RedissonClient`）
### 4、缓存测试类
`com.ruoyi.demo.controller.RedisCacheController`<br>
![缓存测试类 RedisCacheController](img07/1f689a01b406469192e0fd06788ab1e4.png)

该类提供了三个主要缓存注解的演示 Demo 以及说明：

| 注解          | 解析                                                                                      | 使用             | 注意事项                                                      |
|-------------|-----------------------------------------------------------------------------------------|----------------|-----------------------------------------------------------|
| @Cacheable  | 表示这个方法有了缓存的功能，方法的返回值会被缓存下来。下一次调用该方法前，会去检查是否缓存中已经有值：如果有就直接返回，不调用方法；如果没有，就调用方法，然后把结果缓存起来。 | 一般用在查询方法上      | 缓存注解严禁与其他筛选数据功能一起使用。例如: 数据权限注解会造成 ==缓存击穿== 与 ==数据不一致== 问题 |
| @CachePut   | 会把方法的返回值put到缓存里面缓存起来，供其它地方使用                                                            | 通常用在新增方法上      |                                                           |
| @CacheEvict | 清空指定缓存                                                                                  | 一般用在更新或者删除的方法上 |                                                           |

### 5、测试
### 5.1、`@Cacheable`
![请求参数](img07/34a01a91cdde42858fe569b3533311f2.png)<br>

![Redis保存结果](img07/b0b6f186dc4547719d4ea7d7e9699175.png)<br>
### 5.2、`@CachePut`
![请求参数](img07/734cf6a1ddf447369de1444c14efaa56.png)<br>

![Redis保存结果](img07/6e56136965624cbf983d5f3718f21800.png)<br>

修改后 `@Cacheable` 接口请求结果：<br>
![请求结果](img07/9aa83db7609549ddbf771aa51af6b479.png)<br>
### 5.3、`@CacheEvict`
![请求参数](img07/3e761d58f74e4c21bea887c056bff95c.png)<br>

![Redis保存结果](img07/b553e8877a2f4865b3f7b952cc207bdf.png)<br>

`test` 已被删除。
## 源码分析
在开始之前，我想先声明一下，最好是对于 Spring AOP 有一定程度的了解，知道什么是 `Pointcut`，`PointcutAdvisor`，`Advice`，`MethodInterceptor` 等。除此之外，了解 Spring AOP 的基本实现原理，否则看起来可能会比较费解（本文不会太详细去阐述相关概念，可以通过参考目录或者其他资料自行补充理解）。

### 1、Redisson 整合 Spring Cache 核心 API

先放一张简单的脑图，只是列举出了 Spring Cache 比较核心的 API：<br>
![Spring Cache 核心 API](img07/95cd5e2c0d764da29bb9948c31716f74.png)
### 2、缓存自动装配
先来简单看下自动装配。<br>

在配置文件 `application.yml` 中，打开 debug 配置：<br>

```yml
debug: true
```
启动服务观察控制台输出。

### 2.1、`CacheAutoConfiguration`
![在这里插入图片描述](img07/e2003ffb06de44f8b354ee35a34a4e22.png)

![在这里插入图片描述](img07/8f47dc966645442d9ed601c5b1629689.png)

这里的 `CacheManager` 正是 `RedisConfig` 中配置的缓存管理器：<br>
![在这里插入图片描述](img07/f5c201970b3c4daaa5522754f72209ff.png)
### 2.2、`RedissonCacheStatisticsAutoConfiguration `
![在这里插入图片描述](img07/3ffc469e19304e9baaf1370175e69b11.png)

![在这里插入图片描述](img07/32a378a364224311bdea03bc663c4df0.png)
### 2.3、`CacheMetricsAutoConfiguration`
![在这里插入图片描述](img07/c4e9144c8f8c4578af371f5e805ae32d.png)

![在这里插入图片描述](img07/7f1f0eedc23e4126844033a31f5be05d.png)

这里引入了 `CacheMetricsRegistrarConfiguration`：<br>
![在这里插入图片描述](img07/55cf17744bdd41078786d7abb06e1a3d.png)

该方法会绑定缓存管理器到注册表中：<br>
![在这里插入图片描述](img07/6cb58d763f8940ef9b1c64bdfd06356f.png)

通过 `RedissonSpringCacheManager` 获取 Cache：<br>
![在这里插入图片描述](img07/c7df38e59a9d4bdb9550b87981da047c.png)

`RedissonSpringCacheManager#createMapCache`<br>
![在这里插入图片描述](img07/4b49bd1a3ed8415092d4991c5524558c.png)

`RedissonSpringCacheManager#getMapCache`<br>
![在这里插入图片描述](img07/211ec2c1846f44e8a1e266952075be10.png)

`Redisson#getMapCache`<br>
![在这里插入图片描述](img07/b9e87dcf6f3147c09f99e73ef4417325.png)

最终得到的结果：<br>
![在这里插入图片描述](img07/c39da17b386d43d280e6064bebdfbddd.png)
### 3、`@EnableCaching`
首先从入口开始入手，参考 [叔牙 - Spring cache原理详解](https://juejin.cn/post/6959002694539444231#heading-0) 中的时序图：

> ![Spring Cache时序图](img07/2851859207a846a1bd39d570f56e856b.png)<br>

`org.springframework.cache.annotation.EnableCaching`<br>
![@EnableCaching](img07/fcc73c26c7e44c189957fdb61d78a2d5.png)

要了解一个 `EnableXX` 注解干了啥，就看它导入 `@Import` 了什么内容。

### 3.1、`CachingConfigurationSelector`
`org.springframework.cache.annotation.CachingConfigurationSelector`<br>
![CachingConfigurationSelector](img07/db49870557294ab9a538e6e4a0e55726.png)

![在这里插入图片描述](img07/3b76ae88869b4623bf1e6b10847e67eb.png)
### 3.1.1、自动代理注册器 `AutoProxyRegistrar`
`org.springframework.context.annotation.AutoProxyRegistrar#registerBeanDefinitions`<br>
![在这里插入图片描述](img07/74591013941642178b870a57c8e385ea.png)

`org.springframework.aop.config.AopConfigUtils#registerOrEscalateApcAsRequired`<br>
![在这里插入图片描述](img07/17c157ed58064254a38cb57a70072fe5.png)
### 3.1.2*、代理缓存配置 `ProxyCachingConfiguration`
`org.springframework.cache.annotation.ProxyCachingConfiguration`<br>
![ProxyCachingConfiguration](img07/4d4aa67c1c4e47e99e507822a67ce092.png)

最重要的配置类之一，这里面配置了缓存相关的内容，也是基于 AOP 相关的实现。下面一个个来说明。

### 3.1.2.1、`PointcutAdvisor`：`BeanFactoryCacheOperationSourceAdvisor`
![在这里插入图片描述](img07/2794832dbb2045f09aa41a5b872d0c51.png)

首先来看下 UML 图：<br>
![在这里插入图片描述](img07/40ce38b599a4407cad810e0b54071825.png)<br>
`PointcutAdvisor` 相当于 `Pointcut` 与`Advice` 的连接器（ `PointcutAdvisor` 继承自 `Advisor`，`Advisor` 是 `Advice` 的容器接口，他们是一对一关联的）。

### 3.1.2.2、`CacheOperationSource`
![在这里插入图片描述](img07/49b6eb9177c14e4bb3d6dfbd7715c209.png)<br>

该方法 `new AnnotationCacheOperationSource()`。<br>
![在这里插入图片描述](img07/5e18e885f0d34786983a692cd845be6b.png)

初始化了 Spring 缓存解析器 `SpringCacheAnnotationParser`：<br>
![在这里插入图片描述](img07/a73db83ecede4c479e955b9930246ab2.png)

在方法 `SpringCacheAnnotationParser#parseCacheAnnotations` 进行缓存注解的解析操作：<br>
![在这里插入图片描述](img07/21f9ddd6b9574220818911b6dd302173.png)
### 3.1.2.3、`Advice`（`MethodInterceptor`）：`CacheInterceptor`
![在这里插入图片描述](img07/1db9d4c452c74e31a61de7b5064c9fdd.png)

熟悉 Spring AOP 的朋友都知道，Spring AOP 中提供了不同类型的 Advice，但是通过查看源码 `AdvisorAdapter` 可知，所有 Advice 底层实现都是用`MethodInterceptor` 来进行实现的。

查看 `CacheInterceptor` 的 UML 图：<br>
![在这里插入图片描述](img07/64931429d4314e3f86c1f1edfeb4a047.png)

AspectJ 实现是通过拦截器反射（invoke）来实现的，因此在该类中也必然有 `invoke` 方法：<br>
![在这里插入图片描述](img07/39196d611b1f4bcd80b26aaae286e45f.png)

`CacheInterceptor` 的能力来自于父类 `CacheAspectSupport`：<br>
![在这里插入图片描述](img07/0ec93a9ec24948058c8b764fe20d95c6.png)

稍后断点再进行说明。
### 4、缓存以及获取值（@Cacheable）
### 4.1、第一次请求，没有缓存值
`CacheAspectSupport#execute`<br>
![在这里插入图片描述](img07/5a6e18fa6bfe471cb6d0c87586bd0caf.png)
### 4.1.1、从缓存中查找是否有值
`CacheAspectSupport#execute`<br>
![在这里插入图片描述](img07/5daa47dedaed4a72bcae85794a7d48a9.png)

`CacheAspectSupport#findCachedItem`<br>
![在这里插入图片描述](img07/ba642447c5b64e4ab85efd504f439da1.png)

`CacheAspectSupport#findInCaches`<br>
![在这里插入图片描述](img07/c589df09f3a54e749d3bdd2e6587eaf6.png)

`AbstractCacheInvoker#doGet`<br>
![在这里插入图片描述](img07/58a234013c22496dbae88c630909ffd5.png)

`RedissonCache#get`<br>
![在这里插入图片描述](img07/623a443d42414251b5b019eebf2866fc.png)

第一次请求为空，回到主方法 `CacheAspectSupport#execute`，将结果进行缓存。
### 4.1.2、缓存
![在这里插入图片描述](img07/3a493f084ddd4fa5b4281146ba2b2e89.png)

`CachePutRequest#apply`<br>
![在这里插入图片描述](img07/a30a752636944ec5b0222ce93a5cfc81.png)

`AbstractCacheInvoker#doPut`<br>
![在这里插入图片描述](img07/56788fb3608d4a55b67af2cd6ce46373.png)

缓存完成回到主方法，返回最终执行结果：<br>
![在这里插入图片描述](img07/726de94cd94d47e2b76afcf411054dec.png)
### 4.2、第二次请求，有缓存值
`RedissonCache#get`<br>
![在这里插入图片描述](img07/6d0b81ffc7d942ac9295d196aba5623c.png)

获取到缓存值，直接返回到前端，不再进入请求方法体内。