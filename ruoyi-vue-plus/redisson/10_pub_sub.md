# Redisson（十）发布 | 订阅功能分析
- - -

## 前言
本来承接上篇应该到 OSS 模块的初始化分析了，结果半路杀出个程咬金：在做分析的时候遇到了发布订阅的相关功能的一些问题，去请教了一下 [狮子大佬](https://lionli.blog.csdn.net/) ，所以决定这篇先简单分析一下发布订阅功能，下一篇再回归 OSS 功能。

## 参考目录
### 使用文档
- [Redisson 官方文档 - 6.7. 话题（订阅分发）](https://github.com/redisson/redisson/wiki/6.-%E5%88%86%E5%B8%83%E5%BC%8F%E5%AF%B9%E8%B1%A1#67-%E8%AF%9D%E9%A2%98%E8%AE%A2%E9%98%85%E5%88%86%E5%8F%91)
- [Redis 官方文档 - Redis Pub/Sub](https://redis.io/docs/manual/pubsub/)
### 发布订阅相关概念
- Redis设计与实现（黄健宏）第18章 发布与订阅
  后面的三篇文章应该都有参考此书。
- [观察者模式与订阅发布模式的区别](https://www.cnblogs.com/onepixel/p/10806891.html)
- [观察者模式与发布订阅模式真的不同](https://juejin.cn/post/6844903842501378055)
- [Redis进阶 - 消息传递：发布订阅模式详解](https://pdai.tech/md/db/nosql-redis/db-redis-x-pub-sub.html)

**对于相关概念，我就不再在此重复了，感兴趣的朋友可以自行查看或者查找。**

## 框架集成
### 版本信息
- RuoYi-Vue-Plus `V4.3.0`
- Redisson `V3.17.5`
### 发布消息 `RedisPubSubController#pub`
![在这里插入图片描述](img10/6310c0d489544f7bafd0e53ae09e43e3.png)

![在这里插入图片描述](img10/d83979719d8847fa9e317c55f327065c.png)

### 订阅消息 `RedisPubSubController#sub`
![在这里插入图片描述](img10/d6a599851d354206aaa7a4be91765ee4.png)

![在这里插入图片描述](img10/76b7f85bbe004c688bffea4002a162be.png)
### 接口请求控制台打印
![在这里插入图片描述](img10/8fa8e1eeed984a989715a304a64fd7dc.png)

## 功能调用流程分析之【订阅流程】
### 1、`RedisPubSubController#sub`
![在这里插入图片描述](img10/58c438b0406b4616a474a43a327144d9.png)
### 2、`RedisUtils#subscribe`
![在这里插入图片描述](img10/30ebdff350cc45d288650f8ef469dca6.png)

方法 `topic.addListener` 是为 RTopic 添加一个监听器，框架中对监听器的方法进行了重写，在后面发布的部分会详细说明。

### 3、`RedissonTopic#addListener`
![在这里插入图片描述](img10/3f3ef115d7e6423a8a26ac357328c6f9.png)
### 4、`RedissonTopic#addListenerAsync`
![在这里插入图片描述](img10/46ff4df9c6814c91bcf9961179fb2236.png)

![在这里插入图片描述](img10/6d08d2b0c99b47d8ba55633e9cb2fafa.png)
### 5、`PublishSubscribeService#subscribe`
![在这里插入图片描述](img10/a3b3ceb8480c41759d199c4e438f00fc.png)

![在这里插入图片描述](img10/2cc0c7cc4db84b4cb42b43b49c980595.png)

![在这里插入图片描述](img10/a708fb22ab484f5eb086767e4a99f8c5.png)

### 6、`PublishSubscribeService#subscribeNoTimeout`
![在这里插入图片描述](img10/0eb92549b7464fccb6b84af975f62783.png)

第一次请求时没有 `PubSubConnectionEntry` 对象，继续执行下面的逻辑（后面存入容器对象 name2PubSubConnection 中），后续请求如果存在对象则直接返回。

![在这里插入图片描述](img10/404cf9316b644d0e9058d6a52b7ecbc5.png)

### 7、`PubSubConnectionEntry#subscribe`
![在这里插入图片描述](img10/e3f551c0668249a5999b63ed5fa48f8e.png)

### 8、`RedisPubSubConnection#subscribe`
![在这里插入图片描述](img10/e9c7f4ed9df943f291b3a4ae9d0c5f12.png)

### 9、`RedisPubSubConnection#async`
![在这里插入图片描述](img10/bf68b089f8fd439480c81405a9d5fa96.png)

断点到这里后面就不再深入了，是由`io.netty.channel.AbstractChannelHandlerContext#safeExecute` 这个方法完成底层订阅操作。
### 10、Redis 控制台打印结果
![在这里插入图片描述](img10/b1f3f548ca094857b459db702ceafb9b.png)
## 功能调用流程分析之【发布流程】
### 1、`RedisPubSubController#pub`
![在这里插入图片描述](img10/57af9173d9cd43fcac0e6924abc9b262.png)

这里 `System.out.println("发布通道 => " + key + ", 发送值 => " + value);` 实际上就是上面第 `3` 步中重写的方法中的逻辑。
### 2、`RedisUtils#publish`
![在这里插入图片描述](img10/b5e73eb2c3554935930639a87a7f94ba.png)

### 3、`RedissonTopic#publish`
![在这里插入图片描述](img10/9aadc359fe86416b8bde6f6c48d663a5.png)

### 4、`RedissonTopic#publishAsync`
![在这里插入图片描述](img10/9e3216acca854731be719cfb3e4a2bb4.png)

### 5、`CommandAsyncService#writeAsync`
![在这里插入图片描述](img10/6085d3a1e7ee4aa7b0acbd3c8ca65b63.png)

### 6、`CommandAsyncService#async`
![在这里插入图片描述](img10/5dcff2429d7b4e2daff05e8b9061400c.png)

![在这里插入图片描述](img10/807d7363e80f49f8988dca90b8053f3b.png)

### 7、`RedisExecutor#execute`
![在这里插入图片描述](img10/794fad07ad484ba0b21558cf7a6a39c7.png)

### 8、`RedisExecutor#sendCommand`
![在这里插入图片描述](img10/9d928b79171c4495aeecbca7aa801262.png)

同样，断点到这里后面就不再深入了，最后发布操作同样是由`io.netty.channel.AbstractChannelHandlerContext#safeExecute` 这个方法完成。
### 9、Redis 控制台打印结果
![在这里插入图片描述](img10/750fb5cb230e4848ba1fdd819d6e9263.png)
## 功能调用流程分析之【订阅通道接收到发布消息】
特别说明：这里分析截图用的订阅通道是【test1】，发布消息内容是【test1】。

**如果先进行订阅操作，再进行发布操作，那么在发布完成后订阅通道就能够直接接收到消息，因此实际上这一部分内容在时间上是紧接着【发布流程】的。**

### 1、`CommandPubSubDecoder#decodeResult`
这里解码并根据消息类型进行对应的操作。<br>
![在这里插入图片描述](img10/0b90047853624d5eacd5b9fbb8db5155.png)<br>

这个方法前面还有一些操作步骤（参考以下序列图，只保留了关键方法）：<br>
![在这里插入图片描述](img10/71009b3a94f8416e8527c6a3b8c84e73.png)
### 2、`RedisPubSubConnection#onMessage`
![在这里插入图片描述](img10/0f8c33769c9a4f6382126585a7fdb255.png)

此处的逻辑是遍历所有的监听器，执行 `onMessage` 方法。

### 3、`PubSubMessageListener#onMessage`
![在这里插入图片描述](img10/3f1bf36318e948fbab9122b05cf6a9bd.png)

`MessageListener` 接口：<br>
![在这里插入图片描述](img10/d86d09c260a7409187417463a025e655.png)
### 4、`RedisUtils#subscribe`
重写 `onMessage` 方法：<br>
![在这里插入图片描述](img10/894a0fab5d7f4189a65d67cc1908b66e.png)

换种写法比较容易理解：<br>
![在这里插入图片描述](img10/4195c96f49e14d018d0b8f4ce51dfd6d.png)
### 5、`RedisPubSubController#sub`
![在这里插入图片描述](img10/250049a7825f4000881b5f6ea6ab3f8c.png)
### 6、控制台打印结果
![在这里插入图片描述](img10/94b3481456e742e385dfd31c9dd8dedc.png)
