# RuoYi-Vue-Plus 学习笔记（持续施工ing...）
- - -
## 简介
此工程为 RuoYi-Vue-Plus 框架学习笔记汇总。😊

## 博客
CSDN：[MichelleChung](https://blog.csdn.net/Michelle_Zhong?type=blog)<br>

## 代码地址

| 介绍       | 项目名              | 项目地址                                                                                                             | 注意事项                       |
|----------|:-----------------|------------------------------------------------------------------------------------------------------------------|----------------------------|
| 🔥 分布式集群 | RuoYi-Vue-Plus   | - [Gitee](https://gitee.com/dromara/RuoYi-Vue-Plus)<br> - [GitHub](https://github.com/dromara/RuoYi-Vue-Plus)    | 重写RuoYi-Vue全方位升级(不兼容原框架)   |
| 🔥 微服务分支 | RuoYi-Cloud-Plus | - [Gitee](https://gitee.com/dromara/RuoYi-Cloud-Plus)<br>- [GitHub](https://github.com/dromara/RuoYi-Cloud-Plus) | 重写RuoYi-Cloud全方位升级(不兼容原框架) |

## 模块说明
| 包名              | 内容                                             | 官方文档                                                                | 说明                                                                                                                  | 备注                                    |
|-----------------|------------------------------------------------|---------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| aop             | <a href="#aop">AOP</a>                         |                                                                     |                                                                                                                     |                                       |
| data-permission | <a href="#数据权限">数据权限</a>                       |                                                                     |                                                                                                                     |                                       |     
| easyexcel       | <a href="#easy-excel">Easy Excel</a>           | [传送门](https://easyexcel.opensource.alibaba.com/docs/current/)       | 一个基于Java的简单、省内存的读写Excel的开源项目。                                                                                       |
| extends         | 📘 <a href="#-扩展笔记">扩展笔记</a>                   |                                                                     |                                                                                                                     |
| filter          | <a href="#过滤器">过滤器</a>                         |                                                                     |                                                                                                                     |
| interceptor     | <a href="#拦截器">拦截器</a>                         |                                                                     |                                                                                                                     |
| issues          | 📕 <a href="#-问题笔记">问题笔记</a>                   |                                                                     |                                                                                                                     |
| jackson         | <a href="#jackson">Jackson</a>                 | [传送门](https://github.com/FasterXML/jackson)                         | Jackson has been known as "the Java JSON library" or "the best JSON parser for Java". Or simply as "JSON for Java". |
| jwt             | <a href="#jwt">JWT</a>                         | [传送门](https://jwt.io/)                                              | JSON Web Tokens                                                                                                     | 结合权限框架使用（如 Sa-Token、Shiro 等）          |
| listener        | <a href="#监听器">监听器</a>                         |                                                                     |                                                                                                                     |                                       |
| log             | <a href="#日志">日志</a>                           | - TLog：[传送门](https://tlog.yomahub.com/)                             | - TLog：轻量级的分布式日志标记追踪神器                                                                                              | - ⚠ TLog 已于 V4.4.0 版本移除               |                                
| mybatis         | <a href="#mybatis">Mybatis</a>                 | [传送门](https://mybatis.org/mybatis-3/zh/index.html)                  | MyBatis 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。                                                                          |
| mybatis-plus    | <a href="#mybatis-plus">Mybatis-Plus</a>       | [传送门](https://baomidou.com/pages/24112f/)                           | MyBatis-Plus (opens new window)（简称 MP）是一个 MyBatis (opens new window)的增强工具，在 MyBatis 的基础上只做增强不做改变，为简化开发、提高效率而生。      |
| oss             | <a href="#oss">OSS</a>                         | - MinIO：[传送门](http://docs.minio.org.cn/docs/)                       | - MinIO：MinIO 是一款高性能、分布式的对象存储系统。                                                                                    |
| redisson        | <a href="#redisson">Redisson</a>               | [传送门](https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95) | Redisson - Redis Java client with features of In-Memory Data Grid.                                                  |                                       |     
| role-permission | <a href="#角色权限">角色权限</a>                       |                                                                     |                                                                                                                     |
| sa-token        | <a href="#sa-token">Sa-Token</a>               | [传送门](https://sa-token.dev33.cn/doc/index.html#/)                   | 一个轻量级 Java 权限认证框架，让鉴权变得简单、优雅！                                                                                       |
| shiro           | <a href="#shiro">Shiro</a>                     | [传送门](https://shiro.apache.org/documentation.html)                  | Simple. Java. Security.                                                                                             | 轻量级权限框架， ⚠ 在原版若依中使用                   |     
| spring-security | <a href="#spring-security">Spring Security</a> | [传送门](https://docs.spring.io/spring-security/reference/index.html)  | Spring Security is a framework that provides authentication, authorization, and protection against common attacks.  | ⚠ 框架 V3.5.0 及以下使用，V4.0.0+ 改为 Sa-Token |
| validator       | <a href="#校验">校验</a>                           |                                                                     |                                                                                                                     |
| web             | <a href="#前端相关">前端相关</a>                       |                                                                     |                                                                                                                     |

