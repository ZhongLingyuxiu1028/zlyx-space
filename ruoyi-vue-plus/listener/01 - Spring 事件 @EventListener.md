# Spring 事件监听器 @EventListener 注解简单分析

## 前言
因为之前比较忙所以匿了一段时间，顺便当了神雕大侠（“阳过”）。前段时间框架已经发布了新版本 V4.4.0，而在最新的 dev 分支中使用了事件监听器对代码进行解耦，所以这篇文章来简单分析一下它的源码。

## 参考目录
- [Spring 官方文档 - Standard and Custom Events](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-functionality-events)
- [狮子大佬的博客 - SpringBoot 事件机制使用方式与特性讲解](https://lionli.blog.csdn.net/article/details/128241806?spm=1001.2014.3001.5502)
- [it_lihongmin - Spring源码-事件监听机制（@EventListener实现方式）](https://blog.csdn.net/it_lihongmin/article/details/102940643)

事件的使用其实非常简单，特别是使用 Spring 提供的注解 `@EventListener`，只需要传入指定的参数即可，下面将从初始化以及方法调用两个方面进行简单的分析。

为了简单起见，我使用的是自定义方法，并且是在4.X分支（主分支）下进行演示，dev分支已经使用事件重构了OSS加载配置以及日志记录配置等方法，有兴趣的话可以自行查看。

## 测试方法
### 配置说明
- 框架版本：`V4.4.0`
- Spring Boot 版本：`V2.7.6`
  ![在这里插入图片描述](img01/f735910c7fa8444abc1077adc21d518a.png)
### 测试方法
`com.ruoyi.demo.controller.TestDemoController`
![在这里插入图片描述](img01/e1f188dc38a34667a1686e4dcc05289d.png)

事件对象：<br>
`com.ruoyi.demo.domain.event.DemoEvent`<br>
![在这里插入图片描述](img01/ab30e05389ba4691a7933783dc50b7a6.png)

`com.ruoyi.demo.domain.event.DemoEvent2`<br>
![在这里插入图片描述](img01/cda1359d452048c890860575717e936f.png)
## 功能调用流程分析
### 事件监听器初始化
由注解 `@EventListener` 标注的方法即为事件监听器。

`org.springframework.context.event.EventListener`<br>
![在这里插入图片描述](img01/6572f2cb92c04a6d8f047a19eca5163f.png)

可以看到红框中的注释，指出了主要的处理类是 `EventListenerMethodProcessor`。

`org.springframework.context.event.EventListenerMethodProcessor`<br>
![在这里插入图片描述](img01/a40be561b0c44193aef1e009f0138381.png)

这实际上是一个后置处理器，有一个重要的方法：<br>
`EventListenerMethodProcessor#afterSingletonsInstantiated`<br>
![在这里插入图片描述](img01/164455c4ae0e44beb971def23321ce27.png)

最主要的逻辑是处理bean的方法：<br>
`EventListenerMethodProcessor#processBean`<br>
![在这里插入图片描述](img01/61d9badd009643bda18d1ec1b14021eb.png)

最开始需要做判断：
1. `nonAnnotatedClasses` 是没有注解的类的集合，if内部的方法在每次处理 bean 时都会把新的放进去；
2. `isCandidateClass` 判断是否是候选类；
3. `isSpringContainerClass` 判断是否是 Spring 容器类。

判断通过进入方法体：<br>
![在这里插入图片描述](img01/516a41d6736549caa0bc8de80b708ab8.png)

当有符合的方法，则包装成 `ApplicationListenerMethodAdapter`。<br>
![在这里插入图片描述](img01/28a2b57643b44b4298dc3d0832db68d8.png)

`ApplicationListenerMethodAdapter#ApplicationListenerMethodAdapter`<br>
![在这里插入图片描述](img01/23bac57b4da149479215789ed5dbb365.png)

最终将监听器存入Spring context。<br>
![在这里插入图片描述](img01/b11b4cd5d3ef44d49c7690526002087b.png)
### 事件发布流程
发布事件操作很简单，只需要调用 `ApplicationContext` 的 `publishEvent` 方法即可。<br>

![在这里插入图片描述](img01/346c96546c9847e1845632bea89068ca.png)

`org.springframework.context.ApplicationEventPublisher`<br>
![在这里插入图片描述](img01/09dfcee411c74830a79849e1546944da.png)

`AbstractApplicationContext#publishEvent`<br>
![在这里插入图片描述](img01/472c95e737984f6fa0352233a918fe12.png)

同一个方法可以有多个监听器。<br>

![在这里插入图片描述](img01/2791bb702b2943b7a9d5c6d1577d2196.png)

`SimpleApplicationEventMulticaster#multicastEvent`<br>
![在这里插入图片描述](img01/55325fcb7f0b41bab7dd12ebfe20ce37.png)

![在这里插入图片描述](img01/8ed111b92ed049ea8e6e6ac9889ad1a2.png)

`SimpleApplicationEventMulticaster#invokeListener`<br>
![在这里插入图片描述](img01/f305b5ce850f451288a6e950a6f2554a.png)

`SimpleApplicationEventMulticaster#doInvokeListener`<br>
![在这里插入图片描述](img01/6b5fc901fa2e4e1aa03db23ff71ed90d.png)

注意这里是调用 `ApplicationListenerMethodAdapter` 的方法：<br>
![在这里插入图片描述](img01/4a14808413e244f5a41d2a51d032b9f5.png)

`ApplicationListenerMethodAdapter#onApplicationEvent`<br>
![在这里插入图片描述](img01/dd7c05cae437477abd50d88a7e6b2c1e.png)

`ApplicationListenerMethodAdapter#processEvent`<br>
![在这里插入图片描述](img01/6a6b303bc74d45579d6d4b34be9b5b64.png)

`ApplicationListenerMethodAdapter#doInvoke`<br>
![在这里插入图片描述](img01/0d14965b4e354e7ab6201f94577ddb30.png)

这里实际上就是调用具体事件监听器的方法：<br>
![在这里插入图片描述](img01/99a76b5156b14f8b8b118acf2c312166.png)

控制台的打印：<br>
![在这里插入图片描述](img01/42b2397087674d24bf6dc95f83e0498b.png)

（完）