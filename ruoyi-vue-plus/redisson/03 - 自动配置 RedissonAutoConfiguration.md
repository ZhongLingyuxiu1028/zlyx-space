# Redisson（三）自动配置 RedissonAutoConfiguration

## 前言
本来想着写一下关于 `@RepeatSubmit` 防止重复提交的内容，因为有点久没有更新代码，Redis 一开始连不上（Redis服务和密码也没有问题），后面发现是代码有变动，重新打包就可以了。关于变动的地方去群里问了一下 [狮子大佬](https://blog.csdn.net/weixin_40461281)，他让我看一下 `RedissonAutoConfiguration`，所以就写一下这篇博客。
## 参考目录
[Redisson 官方文档 - 2. 配置方法](https://github.com/redisson/redisson/wiki/2.-%E9%85%8D%E7%BD%AE%E6%96%B9%E6%B3%95)

## 代码变动
变动的类：`com.ruoyi.framework.config.RedisConfig`<br>

> ![这里是引用](img03/258a30a9386242fb9ed43ff77574f52d.png)

在 V3.5.0 的版本，类里面的方法和 `org.redisson.spring.starter.RedissonAutoConfiguration` 的写法比较相似，相当于直接覆盖了原有的 Bean `redisson`。

但在最新的版本 V4.2.0 中，使用了默认的 `redisson`，只是在配置类里面使用 Redisson 定制器定制了其他的属性。

因为我使用的是单机模式，所以简单截取这一部分来进行对比：<br>
![在这里插入图片描述](img03/ebd1d4af2e9c456c9d175e5439eaa5ac.png)

由上图，旧版本中需要自行去创建 `RedissonClient`。

## 源码分析
自动配置类 `org.redisson.spring.starter.RedissonAutoConfiguration`：<br>
![在这里插入图片描述](img03/67190e300d204a199f8c5da5ba75316c.png)

自动配置方法 `RedissonAutoConfiguration#redisson`：<br>
![在这里插入图片描述](img03/4e86eb2d7a6d41a6a7282cd3b13dbf46.png)

从官方文档可以知道有多种配置方式：<br>

> ![这里是引用](img03/20006911a2844e2db5a73ff43c79fa0f.png)<br>

因此在方法中也根据不同的配置方法进行了判断。

### 0、配置读取
![在这里插入图片描述](img03/7f0b09f19a794d8aab43797b464e6612.png)<br>

![在这里插入图片描述](img03/92ed07a0554245089e598c198645c8f8.png)<br>

框架中没有直接配置 RedissonProperties，只有 RedisProperties。

![在这里插入图片描述](img03/d95368ad4be044c0a14ed210f32b7f68.png)
### 1、通过文件或者JSON配置
![在这里插入图片描述](img03/4532f2d15fcd41748f5f112d9f535408.png)

从前面可以知道这里没有配置，运行结果也是 `null`。<br>
![在这里插入图片描述](img03/045eae0797f743d182ded465fe6526e8.png)
### 2、哨兵模式
![在这里插入图片描述](img03/867da1411d6841b7a5816ae64d4de448.png)
### 3、集群模式
![在这里插入图片描述](img03/2b419cca766341e792a018ed22673448.png)
### 4、单Redis节点模式
![在这里插入图片描述](img03/9ebc1a9a22764fab875964c47c6bceb9.png)

在这里就把原本的路径（Address）、超时时间（ConnectTimeout）、库（Database）、密码（Password）进行了设置。<br>
![在这里插入图片描述](img03/8911b6100b684c6991c7e4401eecdd51.png)
### 5、Redisson 定制器自定义配置
![在这里插入图片描述](img03/08a3ad37f3b04ec1b176c6548e75b922.png)<br>

遍历所有的定制器，进行自定义配置。<br>
![在这里插入图片描述](img03/27c3f7aab1d740bd849d16ed9f46e1b4.png)

![在这里插入图片描述](img03/6c30679cffac4c92ac2eacdc4f476731.png)

![在这里插入图片描述](img03/c885773859034701ada0249166ff1d1d.png)

至此，Redisson 自动配置完成。

最后创建 `RedissonClient` 并注入到容器中。

## 变动原因说明

> ![这里是引用](img03/7132ac00dcac4b1a8a4a749132ef3dbd.png)
