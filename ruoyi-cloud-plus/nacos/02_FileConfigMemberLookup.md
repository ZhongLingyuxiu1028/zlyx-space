# Nacos（二）寻址机制之文件寻址分析

## 前言
承接上一篇博客，讨论完单机寻址模式后这篇来讨论一下文件寻址模式。

同样地，本文还是在[《Nacos架构&原理》]((https://developer.aliyun.com/ebook/36?spm=a2c6h.20345107.ebook-index.18.152c2984fsi5ST))章节 `Nacos 寻址机制` 的基础上对源码进行分析和学习。

## 参考文档
- [框架 wiki](https://gitee.com/dromara/RuoYi-Cloud-Plus/wikis/%E9%A1%B9%E7%9B%AE%E7%AE%80%E4%BB%8B)
- [Nacos 官方文档](https://nacos.io/zh-cn/docs/what-is-nacos.html)
- [Nacos架构&原理](https://developer.aliyun.com/ebook/36?spm=a2c6h.20345107.ebook-index.18.152c2984fsi5ST)

## 关于文件寻址

> ![在这里插入图片描述](img02/26b3c6d45d404a788c6294cb6210c4ea.png)
> ![在这里插入图片描述](img02/043fe44427154489b2e25a621fa7d3a8.png)

第一段话其实就已经涵盖了最重要的内容：

1. 集群模式下的默认寻址模式
2. 需要维护一个 cluster.conf 文件

## 框架集成
本文使用的【RuoYi-Cloud-Plus】框架版本是 `V1.3.0`。<br>
![在这里插入图片描述](img02/905fac4a3eda428ca8260075b5fab9a3.png)<br>

Nacos采用的是源码集成方式，版本为 `V2.1.1`。<br>
![在这里插入图片描述](img02/555c996f23f8442ebccd6a8505a0311c.png)

## 集群启动演示
关于集群启动的测试，很多教程是复制三份源码进行操作，本文稍微偷懒一下，在 idea（版本 2022.1）里面启动三次服务来模拟集群，具体步骤请看下文。

### 步骤一：创建 conf 文件
上文有说到，文件寻址模式需要每个节点都维护一份 conf 文件，因此三个节点需要三份 `cluster.conf`。

在本地创建三个节点（地址自定义）：<br>

`F:/study/Nacos/test`<br>
![在这里插入图片描述](img02/bf9cae092f30411b89a0d4cc13f475a3.png)<br>

在每个节点下都需要创建 `conf` 文件夹，然后在此文件夹下创建 `cluster.conf` 文件：<br>
![在这里插入图片描述](img02/beb785d50ca34000aff82334ae21ace3.png)<br>

注：路径 `/conf/cluster.conf` 是默认路径：<br>
![在这里插入图片描述](img02/e016202a3abf4023bddf91fea66e169e.png)<br>

变量值：<br>
![在这里插入图片描述](img02/691043ae6e694b328fac06ce72af33d9.png)

编辑 `cluster.conf` 文件，配置节点 `IP:PORT` 信息：<br>

```conf
192.168.10.1:8848
192.168.10.1:8858
192.168.10.1:8868
```

### 步骤二：修改 Nacos 启动类
修改 Nacos 启动类，将单机模式修改为 `false`：<br>
![在这里插入图片描述](img02/c80fca9d54c44e24945e63e8019ec0b4.png)
### 步骤三：设置 IDEA 启动项
![在这里插入图片描述](img02/6ac41539ce8e49ef988322e012ca183c.png)

设置允许多个服务启动：<br>
![在这里插入图片描述](img02/ded3615e4f31441fb3ac3f1911b55cae.png)

![在这里插入图片描述](img02/0fd13c1de1054843b733a81ed9e2b8fa.png)

设置完成会出现：<br>
![在这里插入图片描述](img02/b4531fdd5068426bb39f931726ad7baa.png)

复制三份配置，分别设置启动项参数：<br>
![在这里插入图片描述](img02/a00c582ef4414b9a950930821de0386e.png)

```bash
# Nacos-8848
-Dserver.port=8848 -Dnacos.home=F:/study/Nacos/test/nacos1

# Nacos-8858
-Dserver.port=8858 -Dnacos.home=F:/study/Nacos/test/nacos2

# Nacos-8868
-Dserver.port=8868 -Dnacos.home=F:/study/Nacos/test/nacos3
```

### 步骤四：启动服务
![在这里插入图片描述](img02/f40d996ba21a4f7c8fc9f1951994e45b.png)

## 源码分析
### 寻址模式初始化流程图（重要）
![在这里插入图片描述](img02/4f3ba570202c48ab8090a37222d4013d.png)
对于寻址模式初始化流程做了简单的流程图，下面的分析基于此图。

### 1、`ServerMemberManager` 节点管理器初始化
`ServerMemberManager#init`<br>
![在这里插入图片描述](img02/d321e296ff0740a98ab885d01c37dd33.png)

![在这里插入图片描述](img02/0a1033205b664ccfbe3e43072178d893.png)
### 2、初始化寻址模式  `ServerMemberManager#initAndStartLookup`
![在这里插入图片描述](img02/5e0cc2b891c5428db7682b3bbff7c095.png)

该方法主要有两个步骤：

1. 创建 `MemberLookup` 寻址对象
2. 开始寻址

### 2.1、创建寻址对象实例 `LookupFactory#createLookUp`
![在这里插入图片描述](img02/81d188db374f49d5bae1f1d0623a19e9.png)

此处先进行判断，如果是单机模式，则直接创建对象。而非单机模式，会进入 if 方法体：

1. 从配置环境获取寻址模式类型
2. 根据类型选择寻址模式枚举 `LookupFactory#chooseLookup`
3. 根据枚举获取寻址模式对象 `LookupFactory#find`

此处没有设置 `LOOKUP_MODE_TYPE`。<br>
![在这里插入图片描述](img02/62f9d239d3e44460a63ad4fbfa4a8809.png)

根据 `lookupType` 获取枚举信息：<br>

`LookupFactory#chooseLookup`<br>
![在这里插入图片描述](img02/197636a4f63240518dc6504ff8004b49.png)

根据返回的枚举信息获取寻址对象：<br>

`LookupFactory#find`<br>
![在这里插入图片描述](img02/0616d63661b942e49875c0a1436f566a.png)

注入 `ServerMemberManager` 属性，返回寻址对象到上一层。

![在这里插入图片描述](img02/c228a3cebd954f9c8e52f090e04c1e49.png)
### 2.2、开始寻址 `AbstractMemberLookup#start`
`ServerMemberManager#initAndStartLookup`<br>
![在这里插入图片描述](img02/def5e8d93cc044b68175fdfd84f2e776.png)

`AbstractMemberLookup#start`<br>
![在这里插入图片描述](img02/2bcf20e6f9dd4e06a33d1a41630e34a0.png)

### 3、文件寻址模式 `FileConfigMemberLookup#doStart`
![在这里插入图片描述](img02/c0e57e0a6e5f493d9ff2da7c315f011d.png)
### 3.1、从磁盘读取集群配置 `FileConfigMemberLookup#readClusterConfFromDisk`
![在这里插入图片描述](img02/52cc1927eab244d2b5393cef6756997d.png)

同上篇单机模式，`AbstractMemberLookup#afterLookup` 通知服务变更信息。

### 3.2、注册文件监控中心 `WatchFileCenter#registerWatcher`
![在这里插入图片描述](img02/5eeeddf7dcd1481f9b777cb588d3b8cc.png)

至此，文件寻址初始化完成。