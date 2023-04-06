# Nacos（一）寻址机制之单机寻址分析

## 前言
国庆放假回了一趟老家，然后回来连续搬砖七天（整个人快魔怔了），嘛，调整一下心情重新回到学习的路上。

之前 [狮子大佬](https://blog.csdn.net/weixin_40461281?type=blog) 有提议要不要研究一下【RuoYi-Cloud-Plus】，因为之前没有用过 Cloud 框架，对于相关的一些知识只是看过一些视频浅浅了解过，所以决定还是要从简单的开始入手学习一下这个框架，当然【RuoYi-Vue-Plus】也会继续更新下去。

## 参考文档
- [框架 wiki](https://gitee.com/JavaLionLi/RuoYi-Cloud-Plus/wikis/%E9%A1%B9%E7%9B%AE%E7%AE%80%E4%BB%8B)
- [Nacos 官方文档](https://nacos.io/zh-cn/docs/what-is-nacos.html)
- [Nacos架构&原理](https://developer.aliyun.com/ebook/36?spm=a2c6h.20345107.ebook-index.18.152c2984fsi5ST)<br>
强烈推荐阅读一下这本官方教程。

本文是在阅读上述教程《Nacos架构&原理》章节 `Nacos 寻址机制` 时，对于其内容在源码实现说明上的补充。

本文内容也相对简单，主要是关于单机寻址，也即目前框架中使用的方式（后面会再写关于集群模式以及文件寻址的内容）。

> 关于单机寻址的描述：<br>
> ![](img01/f35303c29a7b4b46bec689ea7e8eb3af.png)
## 源码分析
### 1、配置单机模式
`com.alibaba.nacos.Nacos`<br>
![在这里插入图片描述](img01/3893d2ca7b1f40f3b57df5ed4daeff34.png)
### 2、`StandaloneProfileApplicationListener#onAppliationEvent`
![在这里插入图片描述](img01/efd529b124c4429abc87961a0a6ac847.png)<br>

启动 Nacos，会自动注册 `StandaloneProfileApplicationListener`。<br>

![在这里插入图片描述](img01/2ea98e7744204b75b2f959c312292555.png)<br>

在前面启动类 main 方法中配置了单机模式，所以此处会加入这个配置。

### 3、`ServerMemberManager`

> ServerMemberManager 存储着本节点所知道的所有成员节点列表信息，提供了针对成员节点的
增删改查操作，同时维护了⼀个 MemberLookup 列表，方便进行动态切换成员节点寻址方式。

![在这里插入图片描述](img01/f33fb8096a1f477d96467f5d26f06543.png)

`ServerMemberManager#init`<br>
![在这里插入图片描述](img01/6ff6ef5d88e34bc4a9def5a728d2209e.png)

![在这里插入图片描述](img01/d1f9829aa5f74b7193543d67577c8693.png)
### 4、初始化寻址模式  `ServerMemberManager#initAndStartLookup`
![在这里插入图片描述](img01/25ce75abf08e467c90a512c17fc803c9.png)

通过 `LookupFactory` 创建。

### 4.1、`LookupFactory#createLookUp`
![在这里插入图片描述](img01/40b3cfe33f464c2cba7d08a1d3da2dc8.png)

创建单机寻址对象 `new StandaloneMemberLookup()`。<br>

关于 `LOOK_UP.injectMemberManager(memberManager)` ：<br>

> 用于将 ServerMemberManager 注入到 MemberLookup 中，方便利用 ServerMemberManager 的存储、查询能力。

### 4.2、`AbstractMemberLookup#start`
![在这里插入图片描述](img01/c46ed21488de43ae85dbafd4ed5937b3.png)

`StandaloneMemberLookup#doStart`<br>
![在这里插入图片描述](img01/3a15bc1593264f369106fddf0ce89dfe.png)

`MemberUtil#readServerConf`<br>
![在这里插入图片描述](img01/cb94ffb19011477bba6e3bd4f341850a.png)

### 4.2.1、 `AbstractMemberLookup#afterLookup`
![在这里插入图片描述](img01/cd8637abd8e94d9ab97fac9820948d2d.png)

> afterLookup 则是个事件接口，当 MemberLookup 需要进行成员节点信息更新时，会将当前最新的成员节点列表信息通过该函数进行通知给 ServerMemberManager，具体的节点管理方式，则是隐藏到具体的 MemberLookup 实现中。

`ServerMemberManager#memberChange`<br>
![在这里插入图片描述](img01/11ff6d31eb2743acba7a52b92694e918.png)

![在这里插入图片描述](img01/3026a69a1b6a4ecfb1e144bfc5309219.png)

![在这里插入图片描述](img01/5c626ce3c23a42dfb0e7d8437187e033.png)

至此，`ServerMemberManager` 初始化完毕。

![在这里插入图片描述](img01/73f95b23f5514a19ada052d563e91358.png)

Nacos 启动完成。

![在这里插入图片描述](img01/33676ba1616849aab529fff75eff1976.png)
