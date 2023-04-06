# Nacos（三）使用 Nginx 实现地址服务器寻址及其原理分析

## 前言
Nacos 的寻址机制一共有三种，前两篇分别介绍了 [单机寻址](/ruoyi-cloud-plus/nacos/01_StandaloneMemberLookup.md) 和 [文件寻址](/ruoyi-cloud-plus/nacos/02_FileConfigMemberLookup.md)，本篇来介绍最后一种地址服务器寻址。

## 参考文档
- [框架 wiki](https://gitee.com/dromara/RuoYi-Cloud-Plus/wikis/%E9%A1%B9%E7%9B%AE%E7%AE%80%E4%BB%8B)
- [Nacos 官方文档](https://nacos.io/zh-cn/docs/what-is-nacos.html)
- [Nacos架构&原理](https://developer.aliyun.com/ebook/36?spm=a2c6h.20345107.ebook-index.18.152c2984fsi5ST)

## 关于地址服务器寻址
在上一篇文件寻址操作中，如果遇到需要扩缩容的情况，就需要去修改每一个节点的配置文件 `cluster.conf`，并且还可能存在修改失败导致的节点数据不一致的问题。

> ![在这里插入图片描述](img03/3f7575d0e942479d8640276fafee4eff.png)<br>
> ![在这里插入图片描述](img03/e9bf86593ddb4c6088dbe1d23cb1d5c1.png)<br>
> ![在这里插入图片描述](img03/57d278fe087249d0aaefa53b2dc62e70.png)<br>

为了解决上述问题，因而有了更为方便的地址服务器寻址方式。

> ![在这里插入图片描述](img03/ae11a3f9e24744598cb0cea0da973eec.png)<br>
> ![在这里插入图片描述](img03/18a06da9a7134f62ac33ec86841cc3ec.png)<br>

## 框架集成
本文使用的【RuoYi-Cloud-Plus】框架版本是 `V1.3.0`。<br>
![在这里插入图片描述](img03/905fac4a3eda428ca8260075b5fab9a3.png)<br>

Nacos采用的是源码集成方式，版本为 `V2.1.1`。<br>
![在这里插入图片描述](img03/555c996f23f8442ebccd6a8505a0311c.png)
## 集群启动演示
关于集群启动的部分，和上一篇一样采用的是 IDEA 启动多个服务的方式，不再需要单独创建 `cluster.conf` 配置文件。
### 步骤一：使用 Nginx 作为地址服务器
首先需要说明的是，地址服务器形式多样，并不一定是 Nginx，也可以是 Web 服务器等。

修改 Nginx 配置文件：

```bash
server {
	listen 8989;
	server_name nacoslist;
	
	location /nacos/serverlist {
	    add_header Content_type text/plain;
	    return 200 '192.168.10.1:8848\r\n192.168.10.1:8858\r\n192.168.10.1:8868\r\n';
	}
}
```

监听的端口号是 `8989`，一共三个节点。Nacos 底层解析节点时是按行解析，因此需要用 `\r\n`。

修改后查看返回结果是否正确：<br>
![在这里插入图片描述](img03/bb4797841cc24af593474b0a0b923585.png)<br>

**注：此处使用的 Nginx 是 Linux 云服务器上的。一开始在 Win10 环境下使用会出现节点同步异常的问题，改用了 Linux 虚拟机，也出现了节点同步异常的情况，因此最后采用的是 Linux 云服务器。**

如果没有云服务器的情况下，可以使用接口进行模拟。

```java
@GetMapping("/nacos/serverlist")
public String serverlist() {
    return "192.168.10.1:8848\r\n192.168.10.1:8858\r\n192.168.10.1:8868\n";
}
```

### 步骤二：修改 Nacos 配置文件
`ruoyi-visual/ruoyi-nacos/src/main/resources/application.properties`<br>
![在这里插入图片描述](img03/870903ca09464961b7ff42197db86ed7.png)<br>

请需要注意寻址类型的写法！<br>

路径前缀直接使用 IP 访问，如果有域名也可以设置域名。<br>

### 步骤三：修改 Nacos 启动类（同上篇）
修改 Nacos 启动类，将单机模式修改为 `false`：<br>
![在这里插入图片描述](img03/c80fca9d54c44e24945e63e8019ec0b4.png)
### 步骤四：设置 IDEA 启动项（同上篇）
![在这里插入图片描述](img03/6ac41539ce8e49ef988322e012ca183c.png)<br>

设置允许多个服务启动：<br>
![在这里插入图片描述](img03/ded3615e4f31441fb3ac3f1911b55cae.png)<br>

![在这里插入图片描述](img03/0fd13c1de1054843b733a81ed9e2b8fa.png)<br>

设置完成会出现：<br>
![在这里插入图片描述](img03/b4531fdd5068426bb39f931726ad7baa.png)

为了和文件寻址配置地址区分开，这里设置的home地址分别是 nacos4、nacos5、nacos6：
```bash
# Nacos-8848
-Dserver.port=8848 -Dnacos.home=F:/study/Nacos/test/nacos4

# Nacos-8858
-Dserver.port=8858 -Dnacos.home=F:/study/Nacos/test/nacos5

# Nacos-8868
-Dserver.port=8868 -Dnacos.home=F:/study/Nacos/test/nacos6
```
这里的 home 文件夹会在 Nacos 启动时自动创建。

### 步骤五：启动服务
![在这里插入图片描述](img03/72e008d6f6fb489bb4846450a18f91f4.png)

新生成的文件：<br>
![在这里插入图片描述](img03/b59f9df1149745ff8f5be5b4c03c38c3.png)<br>

生成的配置文件内容：<br>
![在这里插入图片描述](img03/0db28b7a0e5d4743b4130a159c1d47a4.png)<br>

Nacos 控制台：<br>
![在这里插入图片描述](img03/575ba3321d9d41e780902b67e671189c.png)<br>
### 拓展步骤：缩容
去掉 `8868` 节点。

修改地址服务器配置（即修改 Nginx 配置文件）<br>
![在这里插入图片描述](img03/a68083790c07433ba2ccdef6a98556a7.png)<br>

查看Nacos控制台：<br>
![在这里插入图片描述](img03/7a39693aab5b475dab5515c9294b3c34.png)<br>

操作成功。

### 拓展步骤：扩容
在上一步缩容的基础上增加 `8878` 节点。

修改地址服务器配置（即修改 Nginx 配置文件）<br>
![在这里插入图片描述](img03/09251ced007b4c838a81a5394c6f6e7c.png)<br>

启动节点并查看 Nacos 控制台：<br>
![在这里插入图片描述](img03/1a80399873284514941a6f15a597b21a.png)<br>

操作成功。

## 源码分析
### 寻址模式初始化流程图（重要）
在上一篇流程图的基础上进行了细化和整理，重新画了一张流程图，可以根据流程图进行分析。

![在这里插入图片描述](img03/e29dad4cf88641449c01bdad2dcd5f76.jpeg)
### 1、`ServerMemberManager` 节点管理器初始化
`ServerMemberManager#init`<br>
![在这里插入图片描述](img03/af1edec863d8427f96c610c9025e49b0.png)

![在这里插入图片描述](img03/845724d327c3494fb98ba960803d9b96.png)
### 2、初始化寻址模式  `ServerMemberManager#initAndStartLookup`
![在这里插入图片描述](img03/4a4adbf82ea04855a3a7ca72b9ea4280.png)

该方法主要有两个步骤：

1. 创建 `MemberLookup` 寻址对象
2. 开始寻址

### 2.1、创建寻址对象实例 `LookupFactory#createLookUp`
![在这里插入图片描述](img03/d6d575c1275d40fcbe11a6ba0667d49d.png)

此处先进行判断，如果是单机模式，则直接创建对象。而非单机模式，会进入 if 方法体：

1. 从配置环境获取寻址模式类型
2. 根据类型选择寻址模式枚举 `LookupFactory#chooseLookup`
3. 根据枚举获取寻址模式对象 `LookupFactory#find`

此处在配置文件中设置了 `LOOKUP_MODE_TYPE` 为 `address-server`。<br>
![在这里插入图片描述](img03/3a66d4c398414a35b7eba0eca773c609.png)<br>

根据 `lookupType` 获取枚举信息：<br>

![在这里插入图片描述](img03/d4034616bd82481ca247fb1bb93bf6e9.png)<br>

`LookupType#sourceOf`<br>
![在这里插入图片描述](img03/3c8cc56794314e349fbe0f0a62c5bbc9.png)<br>

`LookupFactory#chooseLookup`<br>
![在这里插入图片描述](img03/e343fd57d00b43c58faa57432e525319.png)<br>

根据返回的枚举信息获取寻址对象：<br>
![在这里插入图片描述](img03/e9386bce7fe543fb95e51e24622205ad.png)<br>

`LookupFactory#find`<br>
![在这里插入图片描述](img03/320d9b0c850e4378888aa82ba668e524.png)<br>

注入 `ServerMemberManager` 属性，返回寻址对象到上一层。<br>

![在这里插入图片描述](img03/59367203e2ff4b618ac10e59ab223b11.png)<br>
### 2.2、开始寻址 `AbstractMemberLookup#start`
`ServerMemberManager#initAndStartLookup`<br>
![在这里插入图片描述](img03/f9be4fdf2336493bb9bb824d25506299.png)

`AbstractMemberLookup#start`<br>
![在这里插入图片描述](img03/58ae747c9604451faedc289059bd9c9e.png)
### 3、地址服务器寻址 `AddressServerMemberLookup#doStart`
![在这里插入图片描述](img03/1b18c3bf4796467c884afb6db87dfa7d.png)
### 3.1、初始化地址服务 `AddressServerMemberLookup#initAddressSys`
![在这里插入图片描述](img03/2f777f4457b046a1a1c3febc033e6aee.png)
### 3.2、运行 `AddressServerMemberLookup#run`
![在这里插入图片描述](img03/2566b807acd14cb0bc35f7cdf6592951.png)

![在这里插入图片描述](img03/3ded56b059fa49789a8eccc96ee5f4ff.png)

同步节点 url：（定时任务也是调用此方法）<br>
`AddressServerMemberLookup#syncFromAddressUrl`<br>
![在这里插入图片描述](img03/b25033894d624a10b9df1ebd4ce4f6fb.png)

至此，地址服务器寻址初始化完成。