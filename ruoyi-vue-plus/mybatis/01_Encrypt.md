# 数据加密功能 Encrypt 源码分析
- - -
## 前言
前段时间，在群里大佬们讨论了关于数据存储加密的相关需求，后面就有了关于这个功能的 [PR](https://gitee.com/dromara/RuoYi-Vue-Plus/pulls/274)，在 [框架 5.X 版本](https://gitee.com/dromara/RuoYi-Vue-Plus/tree/5.X/) 中，这个功能被抽取成了独立的组件，所以本文来分析一下这个功能的实现。

值得一提的是，基于这个功能，并且借助最近大火的 `ChatGPT`，对于 Mybatis 的自定义插件的实现过程，我有了更进一步的了解，并且最近也在结合书籍进行 Mybatis 源码的阅读，受益匪浅，有空的话会单独再对书中的笔记进行整理分享。

## 参考目录
- [数据加密](https://gitee.com/dromara/RuoYi-Vue-Plus/wikis/%E6%A1%86%E6%9E%B6%E5%8A%9F%E8%83%BD/%E6%89%A9%E5%B1%95%E5%8A%9F%E8%83%BD/%E6%95%B0%E6%8D%AE%E5%8A%A0%E5%AF%86)
  主要是关于该功能的使用说明。
- [通用源码阅读指导书：MyBatis源码详解](https://weread.qq.com/web/bookDetail/de732ba071f94a8ede7dc94)
  对于 Mybatis 源码（基于 `V3.5.2` 版本，目前框架版本为 `V3.5.11`）学习的资料，写得很详细并且易于上手。

## 功能实现的准备知识
### 1、目录结构说明
![在这里插入图片描述](img01/307c56edaf5e40fdb967ce4c87ebea5d.png)


| 类                             | 说明         | 功能                                            |
|-------------------------------|------------|-----------------------------------------------|
| EncryptField                  | 字段加密注解     | 标注需要加密的字段，用于实体类字段上                            |
| AlgorithmType                 | 算法枚举       | 加密注解的加密算法 `algorithm()`                       |
| EncodeType                    | 编码类型枚举     | 加密注解的编码类型 `encode()`                          |
| EncryptorAutoConfiguration    | 加解密模块配置类   | 配置初始化，注册加密解密拦截器以及加密管理类                        |
| EncryptorProperties           | 加解密属性配置类   | Yaml 配置                                       |
| EncryptorManager              | 加密管理类      | 加解密功能的缓存管理以及方法调用                              |
| EncryptContext                | 加密上下文对象    | 用于 encryptor 传递必要的参数                          |
| IEncryptor                    | 加解密接口      | 提供加解密接口用于自定义扩展                                |
| *Encryptor                    | 加解密现类      | 根据 `AlgorithmType` 以及 `EncodeType` 提供不同的加解密实现 |
| **MybatisEncryptInterceptor** | 入参加密拦截器    | 加密功能核心实现类                                     |
| **MybatisDecryptInterceptor** | 出参解密拦截器    | 解密功能核心实现类                                     |

### 2、一些准备知识
通过上面的目录结构，其实可以知道的是，要实现加解密功能，需要重点关注的是两个 Mybatis 拦截器。

关于这一部分，我看了源码以及书，也请教了一下 `ChatGPT`，下面是整理后的一些内容，了解了之后可以对这个功能先有一个大致的了解。

### 2.1、自定义插件如何实现？

> Mybatis的自定义插件是通过实现Mybatis提供的拦截器接口实现的。

这里的拦截器接口就是指 `org.apache.ibatis.plugin.Interceptor`，框架中 `MybatisEncryptInterceptor` 以及 `MybatisDecryptInterceptor` 也分别实现了这个接口。

### 2.2、Mybatis 拦截器的拦截点？
![在这里插入图片描述](img01/9355ea602d12425c8bf83a765fbfb90a.png)<br>

加密操作：对 `ParameterHandler` 进行拦截处理，拦截参数设置。<br>

![在这里插入图片描述](img01/83cc42277f1d46308e817df2fb5d6a3f.png)<br>

解密操作：对 `ResultSetHandler` 进行拦截处理，拦截结果集处理过程。<br>

![在这里插入图片描述](img01/91995ca623b9408ab3a40c5306742694.png)<br>

### 2.3、关于 `@Intercepts` 注解？
![在这里插入图片描述](img01/9073364e05714a4481d6b8b4dcf7ed69.png)<br>
### 2.4、关于拦截器中的 `Interceptor()` 方法和 `plugin()` 方法？
![在这里插入图片描述](img01/5bdaac8729b84859b2f1dd88575ff4ec.png)<br>
![在这里插入图片描述](img01/f948efc20f4f45ac841c0f510e951ffe.png)<br>

`MybatisEncryptInterceptor#plugin`<br>
![在这里插入图片描述](img01/76c60ab0ca8c4d2aaed8e90a488e7c34.png)<br>

`MybatisDecryptInterceptor#intercept`<br>
![在这里插入图片描述](img01/d8ad4e2e116b4be1aad87c4de5672a52.png)<br>

下面通过 Debug 结合框架中的代码来分析一下这个功能。

## 功能调用流程分析
### 1、说明
### 1.1、数据加密配置
`application.yml`<br>
![在这里插入图片描述](img01/2e3717c1f4534032b4e63c8f3fcb174a.png)<br>

本文使用默认配置进行说明，其他配置可以参考框架 wiki。

### 1.2、加密实体类
`com.ruoyi.demo.domain.TestDemoEncrypt`<br>
![在这里插入图片描述](img01/5009bc7dc7b148c2981bba81a0288f2f.png)

### 1.3、Mapper（非必须）
`com.ruoyi.demo.mapper.TestDemoEncryptMapper`<br>
![在这里插入图片描述](img01/2af05aa5b57045d19b56b7fb3f5c2747.png)

**由于我使用的是开发中的分支，加入了多租户插件，这里为了避免报错所以使用了插件忽略注解。**

### 1.4、测试方法
`com.ruoyi.demo.controller.TestEncryptController`<br>
![在这里插入图片描述](img01/42d534aae4c44ce9902e37e00a0b1920.png)<br>

这是框架内置的测试方法，如果想要更好地了解执行过程，也可以对此方法拆分再进行 Debug 分析。如下：<br>

![在这里插入图片描述](img01/c8264fbce4fe416b9995a17c4c31148a.png)<br>

加密解密操作可以理解成是一个相互的过程，有加密就有解密。理解了其中一个之后，另一个其实也是类似的过程。下面分别会对这两个过程进行分析。

### 2、加密过程的实现分析
### 2.1、拦截加密实现方法
`MybatisEncryptInterceptor#plugin`<br>
![在这里插入图片描述](img01/24eedb9a723c417ab48461aa64dd0156.png)

### 2.2、自定义加密处理器
`MybatisEncryptInterceptor#encryptHandler`<br>
![在这里插入图片描述](img01/a5293c5d268b40d2b4c615bde51a3ef1.png)

### 2.2.1、获取类加密字段缓存
`EncryptorManager#getFieldCache`<br>
![在这里插入图片描述](img01/a44290549ffe4f46acf283847bf487ec.png)

由于第一次调用缓存中没有，所以对所有标注 `@EncryptField` 注解的字段设置属性获取权限，并返回结果集。

### 2.2.2、属性值替换
![在这里插入图片描述](img01/721cc3230dac4c1d8cd1b3a4842682bd.png)

### 2.2.3、字段加密调用过程
`MybatisEncryptInterceptor#encryptField`<br>
![在这里插入图片描述](img01/d0533acc07c4438cac4504eb284bb643.png)

根据注解以及配置创建加解密上下文对象，调用加密方法。

`EncryptorManager#encrypt`![在这里插入图片描述](img01/0d3cdd4b63994c96a5897666b0ae9e25.png)

根据加密算法创建对应的加密器，并且存入缓存。

`EncryptorManager#registAndGetEncryptor`<br>
![在这里插入图片描述](img01/6ff75a1da77b4de489b442c4d4981f89.png)

调用加密器的加密方法进行加密。<br>

`Base64Encryptor#encrypt`<br>
![在这里插入图片描述](img01/2f66414480414e619099cbdf87c30f2d.png)

### 2.3、字段加密完成
加密完成后，返回步骤 `#2.2` 进行属性值替换。<br>

![在这里插入图片描述](img01/87539fe1ad8145a8a094f8878cc64713.png)

循环替换完成后的结果：<br>

![在这里插入图片描述](img01/c81813156acf46ddb4980b041db2bad0.png)

所有字段加密完成后，回到拦截方法并返回目标对象。<br>

![在这里插入图片描述](img01/f72088517bcb4c95966ebc7ed03d9cd5.png)
### 3、解密过程的实现分析
### 3.1、拦截解密实现方法
`MybatisDecryptInterceptor#intercept`<br>
![在这里插入图片描述](img01/5114ec82aa8c4019b607168ba63f840a.png)

### 3.2、自定义解密处理器
`MybatisDecryptInterceptor#decryptHandler`<br>
![在这里插入图片描述](img01/0ca7f12665ee47ae9a3890a757a085b1.png)

![在这里插入图片描述](img01/304fd323e53447f29351875c6d0fb3d6.png)
### 3.2.1、获取类加密字段缓存
这一步和加密类似，由于加密过程中已经将字段存入缓存中，因此可以直接获取到。验证方法：<br>

![在这里插入图片描述](img01/12946d9cfa0c48ff895a007b49e794b6.png)
### 3.2.2、属性值替换
上一步获取到类加密字段缓存后，循环进行字段解密。<br>

![在这里插入图片描述](img01/afc11736e1be4c33a24ddd5e9e4f0058.png)
### 3.2.3、字段解密调用过程
`MybatisDecryptInterceptor#decryptField`<br>

![在这里插入图片描述](img01/ec151f0cab3c4a49834e6323e526e8e1.png)

根据注解以及配置创建加解密上下文对象，调用解密方法。<br>

![在这里插入图片描述](img01/0ea21b13a8664aeda77418971cdecf58.png)

如果是第一次调用，同样会根据加密算法创建对应的加密器，并且存入缓存。<br>

![在这里插入图片描述](img01/53569e8b91f249b3ba8e4874685762f5.png)

调用加密器的解密方法进行解密。<br>

`Base64Encryptor#decrypt`<br>
![在这里插入图片描述](img01/0889cb12dc7046dba414a224696a4599.png)

### 3.3、字段解密完成
解密完成后，返回步骤 `#3.2` 进行属性值替换。<br>

![在这里插入图片描述](img01/e2854064da524dc3a0ee4de878723ceb.png)

循环替换完成后的结果：<br>

![在这里插入图片描述](img01/d77fe4948d924f848d46bd583c44451d.png)

所有字段解密完成后，回到拦截方法并返回执行结果。<br>

![在这里插入图片描述](img01/3f6eb8d929f74493befefe718cc6d311.png)

以上是加解密调用流程的分析。<br>

（完）