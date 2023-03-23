# 数据脱敏 Json 序列化工具 SensitiveJsonSerializer
- - -
## 前言
在框架 V4.0.0 版本更新了序列化脱敏的功能。
![在这里插入图片描述](img01/51db8ddb3936427697be3c281b13e169.png)

在 Demo 模块也增加了数据脱敏的测试类 `TestSensitiveController`，本文来简单分析一下该功能的调用流程。
## 参考目录
- [GitHub - Jackson](https://github.com/FasterXML/jackson)

## 功能代码实现及测试
### 1、数据脱敏注解 `Sensitive`
![在这里插入图片描述](img01/ae0d16e11f9a459ca023eb0805a28099.png)
### 2、脱敏策略枚举 `SensitiveStrategy`
![在这里插入图片描述](img01/23b6acca89f144a2a7fa1a9392fcb6a1.png)

这里所使用的的脱敏工具类底层是 Hutool 的 `DesensitizedUtil`。
### 3、脱敏 Json 序列化器 `SensitiveJsonSerializer`
![在这里插入图片描述](img01/689d7601f96649e7a0a77891f0fcbf07.png)
### 4、测试类 `TestSensitiveController`
![在这里插入图片描述](img01/097e1d303e87483280f8a539620bbb1c.png)

需要注意的是，由于在序列化工具中定义了 admin 用户不进行数据脱敏，因此使用 test 用户进行测试，测试结果如下：
![在这里插入图片描述](img01/140514073dd54885bc9e74523dc06e18.png)
## 功能调用流程分析
### ##、流程简图（重点）
首先按照惯例，请记住这张图（后面的流程截图都是基于此）：<br>
![在这里插入图片描述](img01/6542ed14fc49443895d8cef261b20584.png)

**温馨提醒，由于流程步骤较多（比起以往的多很多），结合这张图走不容易迷路。**
### #1、处理返回结果 `RequestResponseBodyMethodProcessor#handleReturnValue`
由于前置步骤较多，这里从 REST 请求的处理返回值方法开始进行分析，这里是 Spring Web MVC REST 处理流程的其中一个步骤（处理 HandleMethod 执行结果），如果不熟悉的话可以找相关资料复习一下，这里不做详细说明。
![在这里插入图片描述](img01/b1e75e89f58c4b95a83ca4463ad047ea.png)

![在这里插入图片描述](img01/acaafb6d2b1c4ccf8f91b6ce41f71e91.png)

执行过程：<br>
![在这里插入图片描述](img01/e1fae044519b4a279d51603284018700.png)
### #2、使用消息转换器转换返回结果 `AbstractMessageConverterMethodProcessor#writeWithMessageConverters`
在该方法中有以下执行步骤：<br>

1. 判断返回值类型是否是字符类型？
   否 —— 解析返回值类型 R <br>
   ![在这里插入图片描述](img01/24611c9702b84b6a90ed84afd548b5a3.png)
2. 判断是否是资源类型？
   否 <br>
   ![在这里插入图片描述](img01/06b5adb2f57c471ba2840c2c3e0918b8.png)
3. 判断是否设置了 contentType ？
   否 —— 获取 selectedMediaType 为 application/json <br>
   ![在这里插入图片描述](img01/fdce7aee714c4c83be18f5be36f979d6.png)
4. 遍历所有消息转换器，使用转换器写出消息。
   ![在这里插入图片描述](img01/c15e66823f374b63ad71211dd52b7361.png)
### #3、`AbstractGenericHttpMessageConverter#write`
![在这里插入图片描述](img01/91599fc73755480ea0980d6d8f739874.png)
### #4、`AbstractJackson2HttpMessageConverter#writeInternal`
![在这里插入图片描述](img01/65da55f2ad174b49ae22135a400dd25c.png)

![在这里插入图片描述](img01/e8b6bfaec8e94b9e968aec7d20fe336a.png)
### #5、返回值序列化为JSON对象 `com.fasterxml.jackson.databind.ObjectWriter#writeValue`
![在这里插入图片描述](img01/99d505f94f9744398d9877d6994bd21f.png)
### #6、`Prefetch#serialize`
![在这里插入图片描述](img01/0ec39e1423554c2da4e787e9ea24b182.png)
### #7、`DefaultSerializerProvider#serializeValue`
![在这里插入图片描述](img01/ae87a45aaa8741f8b6d354a68f403cd2.png)
### #8、`DefaultSerializerProvider#_serialize`
![在这里插入图片描述](img01/349f4fb4fbd44e8a8c6d225c2a25891a.png)
### #9、`BeanSerializer#serialize`
![在这里插入图片描述](img01/0ec52faff67d47d58f15a739fe19b680.png)
### #10、循环序列化返回值对象所有字段 `BeanSerializerBase#serializeFields`
![在这里插入图片描述](img01/0ef82900f80b4a9ea6b686433cae765e.png)
### #11、`BeanPropertyWriter#serializeAsField`
### #11.1、判断Json序列化器是否为空？
**注：第一次请求时为空，后续请求不为空。**<br>

是：获取序列化器 `BeanPropertyWriter#_findAndAddDynamic`；<br>
否：执行步骤 `#26` 序列化。<br>
### #11.2、获取序列化器 `BeanPropertyWriter#_findAndAddDynamic`
![在这里插入图片描述](img01/3a8d3541ddb04e0c8f65b0f62c5f1f18.png)
### #12、`BeanPropertyWriter#_findAndAddDynamic`
![在这里插入图片描述](img01/f0b29adf76454c499bae986c475e4c80.png)
### #13、`PropertySerializerMap#findAndAddPrimarySerializer`
![在这里插入图片描述](img01/ee5532e230d940318c58f7c243766384.png)
### #14、`SerializerProvider#findPrimaryPropertySerializer`
![在这里插入图片描述](img01/81cb5beede7c48c185c24583eaa7d5e7.png)
### #15、`SerializerProvider#_createAndCacheUntypedSerializer`
![在这里插入图片描述](img01/a4eddfe7e0de4959afd5a0440ae47db5.png)
### #16、`BeanSerializerFactory#createSerializer`
![在这里插入图片描述](img01/0c40781aef9a4773b7202f80c2356dba.png)

![在这里插入图片描述](img01/f30d17aefbf9416ebe05a1611d56ef1c.png)

### #17、`BeanSerializerFactory#_createSerializer2`
![在这里插入图片描述](img01/dda30089099b4013994d7f63a390faef.png)
### #18、`BeanSerializerFactory#findBeanOrAddOnSerializer`
![在这里插入图片描述](img01/9944bcc3cc734f8787946ad6b4ee7080.png)
### #19、构造序列化器 `BeanSerializerFactory#constructBeanOrAddOnSerializer`
![在这里插入图片描述](img01/92468a313d204389b00186a4e1a4d36e.png)<br>

![在这里插入图片描述](img01/6dd5a5ad417e4627bc1fd988dbda1c41.png)<br>

执行完后面的流程后获得的序列化器：<br>
![在这里插入图片描述](img01/150ebfa73f1049d6859a8a960c22982e.png)
### #20、获取所有Bean对象属性 `BeanSerializerFactory#findBeanProperties`
![在这里插入图片描述](img01/9b1caef6416c4fbda7c3ac8f199dd45b.png)

![在这里插入图片描述](img01/5d4d522fd8d5407387a94ef2afc533e5.png)
### #21、循环构造BeanPropertyWriter `BeanSerializerFactory#_constructWriter`
![在这里插入图片描述](img01/cf8fe6fb30ec43c3be51552d54f422de.png)
### #21.1、`PropertyBuilder#buildWriter`
![在这里插入图片描述](img01/a21ed29ebe514ba885d62d52d52909bd.png)

![在这里插入图片描述](img01/4afa9d6df1ba40228a81e45921f97eb9.png)

![在这里插入图片描述](img01/4ea29e94309a43f1ab787f23852a6ea9.png)
### #22、`PropertyBuilder#_constructPropertyWriter`
![在这里插入图片描述](img01/10ee37da82744fd4a6335732ff598293.png)
### #23、`BeanPropertyWriter#BeanPropertyWriter`
![在这里插入图片描述](img01/d4a24bc92c7c4effa1faec017bf93902.png)

**注：将 Json 序列化器保存到上下文中，后面调用不再需要新建。**
### #24、`SerializerProvider#handlePrimaryContextualization`
![在这里插入图片描述](img01/493149c25f3046bf9124e0e3bfabf6ba.png)
### #25、创建自定义上下文序列化器 `SensitiveJsonSerializer#createContextual`
![在这里插入图片描述](img01/d40829d2055b4eadabb83fd9f54cc9c6.png)
### #26、`SensitiveJsonSerializer#serialize`

继续执行步骤 `#11`。<br>
![在这里插入图片描述](img01/a3aed37fd85743c09e4814dd50d27559.png)
### #27、自定义数据脱敏 Json 序列化工具 `SensitiveJsonSerializer#serialize`
![在这里插入图片描述](img01/08ce452494bc4e91bb80ae8520ece964.png)
### #27.1、获取脱敏策略，根据脱敏策略脱敏
![在这里插入图片描述](img01/efe22b2a640e4355865314d0d2fa3121.png)

![在这里插入图片描述](img01/e876df7f95a94dc1a70cd656e51c47aa.png)
### #27.2、JsonGenerator 写出内容
在步骤 `#27` 方法中，不同字段对应了不同脱敏策略，由于逻辑比较简单可以自行理解，因此这里只展示其中一种，其他流程同理。

至此，完成了所有字段的序列化流程后，将向客户端返回 REST HTTP 消息。