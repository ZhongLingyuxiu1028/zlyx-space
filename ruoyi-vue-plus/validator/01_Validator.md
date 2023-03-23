# Validator（一）校验器对 Model 属性校验调用流程分析
- - -
## 前言
前几天群里聊到了有关 EasyExcel 导入校验的问题，因为用到了 `Spring Validation` 校验（底层是 `Hibernate Validator`），加上涉及到监听器等方面的组件，所以本篇先对 `Hibernate Validator` 校验流程进行分析，后面会单独再出一篇关于 EasyExcel 导入流程校验的文章。

## 参考目录
- [Hibernate Validator 6.2.4.Final - Jakarta Bean Validation Reference Implementation: Reference Guide](https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/)
  Hibernate Validator V6.2.4.Final 官方文档（Spring Boot V2.7.3 集成）
- [Java Bean Validation](https://docs.spring.io/spring-framework/docs/5.3.22/reference/html/core.html#validation-beanvalidation)
  Spring Framework V5.3.22 官方文档（Spring Boot V2.7.3 集成）
- [springboot 配置 Validator 校验框架国际化 支持快速返回（@疯狂的狮子Li）](https://blog.csdn.net/weixin_40461281/article/details/121597834)
  框架集成 Validator 介绍
- [Validated、Valid 、Validator，他们的区别你知道几个（@琦彦）](https://blog.csdn.net/fly910905/article/details/119850168)
  关于 @Validated、@Valid 的介绍，很详细，建议读一读
- [Spring Boot 从入门到实战 - 4.4 数据验证（@章为忠）](https://weread.qq.com/web/bookDetail/0f832150727abda40f80656)
- [SpringBoot 如何进行参数校验，老鸟们都这么玩的！（@飘渺Jam）](https://blog.csdn.net/jianzhang11/article/details/119632467)

## 框架集成
### 1、Maven
`ruoyi-common` 模块 `pom.xml`<br>
![在这里插入图片描述](img01/e1ca407904b14b01b5f49f2da639d1ef.png)<br>

由 Spring Boot 引入 `spring-boot-dependencies-2.7.3.pom`<br>
![在这里插入图片描述](img01/c6034f288ca5426ea19422a3d9ecee5a.png)<br>
### 2、校验框架配置类 `ValidatorConfig`
![在这里插入图片描述](img01/398cea9cee914c7a8a13187fb45efb03.png)<br>

`fail_fast` 模式实际上就是报错了之后立即返回，不再校验后面的参数，如果需要返回所有的错误信息（例如 Excel 导入数据验证），则此处可以不设置或者设置为 `false`。
### 3、测试方法
简单起见，我用了原有的 `TestDemoBo` 对象，对属性 `id` 进行非空校验。<br>

`com.ruoyi.demo.controller.TestDemoController#testValidator`<br>
![在这里插入图片描述](img01/c5a0876a939b40539a1fc02eb8dc4e70.png)

**注：这里的测试方法只是为了演示方便，所以直接在 Controller 调用了 mapper 的方法，实际开发中请遵守代码规范进行编写。**

`com.ruoyi.demo.domain.bo.TestDemoBo`<br>
![在这里插入图片描述](img01/c9f31001f4aa4fd9876ee51637afda87.png)<br>

估计很多朋友看我的博客的时候都不会去看参考文档，所以我在此再强调一下，`@Validated` 是 Spring 官方提供的校验注解，里面增加了对分组 `groups` 的支持。

分组的好处是：同一个对象，可以根据不同的业务功能进行不同的分组校验（例如，新增——`AddGroup`，编辑——`EditGroup`），无需再重复编写类似的多个对象（例如，新增——`AddBo`，编辑——`EditBo`），避免过多的冗余代码。

### 4、接口测试
### 4.1、校验失败（参数为 null）
![在这里插入图片描述](img01/a94d17c0850a490facef141b7870168e.png)

控制台输出（部分）：<br>
![在这里插入图片描述](img01/21bdc649feae4ecd8dd48389c4bbb1c0.png)
### 4.2、校验成功（参数不为 null）
![在这里插入图片描述](img01/14caae52ce8e4d05babb2eff3f28f03d.png)

控制台输出：<br>
![在这里插入图片描述](img01/1024147f1ed1436ab079298e59c0f1b0.png)

下面对于校验失败的流程进行简单分析。

## 执行流程分析
在 Spring 处理请求时，需要先对请求参数进行解析，Validator 校验器就是在参数解析前根据注解来进行校验。
### `InvocableHandlerMethod#invokeForRequest`
![在这里插入图片描述](img01/1c7d9412de574a7c9971683a19dafdc4.png)
### `InvocableHandlerMethod#getMethodArgumentValues`
![在这里插入图片描述](img01/6db079a87e574c26b7e0e4399e65da43.png)
### Model 参数解析 `ModelAttributeMethodProcessor#resolveArgument`
![在这里插入图片描述](img01/cffab349010945ef990cdf83fb45e40e.png)<br>

![在这里插入图片描述](img01/a0ce1f327cf049858a07189efb881067.png)
### 验证适用判断 `ModelAttributeMethodProcessor#validateIfApplicable`
![在这里插入图片描述](img01/d172c5af01cb4250bcd3e1ad59dc6a0f.png)<br>

`ValidationAnnotationUtils#determineValidationHints`<br>
![在这里插入图片描述](img01/dc75eff503a24d54a2243bbffb9ec59f.png)<br>

请求参数标注了 `@Validated`，返回适配的验证提示（`QueryGroup`）。<br>

`ValidationAnnotationUtils#convertValidationHints`<br>
![在这里插入图片描述](img01/dc2e11ac6e314855b29c2767704bcc7d.png)<br>

`validationHints` 不为 null，需要进行校验。<br>
![在这里插入图片描述](img01/c7c522dedfb04690b1d89338a64e4d83.png)
### `DataBinder#validate`
![在这里插入图片描述](img01/82abc3bad2764e63baa8c8c5c6346004.png)<br>

![在这里插入图片描述](img01/b4e1f3db016f4a77861c9eadce5581a3.png)
### `ValidatorAdapter#validate`
![在这里插入图片描述](img01/d02adc83096f499a9a338eab45264713.png)
### `SpringValidatorAdapter#validate`
![在这里插入图片描述](img01/8a7d8bc754c84351bc1fbf7ac54b44b3.png)
### Hibernate 校验器 `ValidatorImpl#validate`
![在这里插入图片描述](img01/6b556b1c43de4ea595a345480576846c.png)
### 上下文分组校验 `ValidatorImpl#validateInContext`
![在这里插入图片描述](img01/412c6f6dce34485fb4ed5fa57d508d8e.png)

`ValidatorImpl#validateConstraintsForCurrentGroup`<br>
![在这里插入图片描述](img01/564d92e1c244452eaba07b82526d1b5a.png)

![在这里插入图片描述](img01/a2987f41e82d425facd6453fc29300eb.png)

![在这里插入图片描述](img01/d542c21b50d74976894bfe8c624e0026.png)

此处循环校验所有的约束。
### 校验约束 `ValidatorImpl#validateMetaConstraint`
![在这里插入图片描述](img01/62d78fdfb7d243629e2e97b2e4068b8e.png)

判断是否需要验证 `ValidatorImpl#isValidationRequired`，此处只有属性 `id` 需要校验。<br>
![在这里插入图片描述](img01/4cedb2a7cf8f4915bf27093e6269b2dd.png)

![在这里插入图片描述](img01/d7b83419c74746c5a373bd02a0c8080f.png)

![在这里插入图片描述](img01/d14e894a37cd4229b03bbfdce53d659e.png)

判断完成，回到 `ValidatorImpl#validateMetaConstraint` 继续进行校验。<br>
![在这里插入图片描述](img01/5f3c96ce8dbd4725a4abb74e24240aa9.png)

![在这里插入图片描述](img01/cdbf3d5f83884d288cb31b917d4e1148.png)

`MetaConstraint#doValidateConstraint`<br>
![在这里插入图片描述](img01/9cc6f7220545492b9c19ec4cdf9a2526.png)
### 校验约束 `ConstraintTree#validateConstraints`
![在这里插入图片描述](img01/4bf23254702f446c834c3412331ef6ae.png)
### `SimpleConstraintTree#validateConstraints`
![在这里插入图片描述](img01/35316eeb74d24fb297d4a1076155b260.png)
### 单一约束判断 `ConstraintTree#validateSingleConstraint`
![在这里插入图片描述](img01/6e066e7bb3e34e09a70066c6ef464fd0.png)
### Not Null 校验器 `NotNullValidator#isValid`
![在这里插入图片描述](img01/d47c5a65a14c43c5af6fd59306e8eeb5.png)

至此可知校验结果为 `false`。

### 返回校验结果
`ConstraintTree#validateSingleConstraint`
![在这里插入图片描述](img01/1156232fd5fd453c9fb633f13ec6a63b.png)

`SimpleConstraintTree#validateConstraints`<br>
![在这里插入图片描述](img01/ac990ef2b25b43aa87173c710014161e.png)

`ConstraintTree#validateConstraints`<br>
![在这里插入图片描述](img01/9e39786ec90c41bda40deb7621e89713.png)

`MetaConstraint#doValidateConstraint`<br>
![在这里插入图片描述](img01/d902c8f2ba2c4d629d6c509e16848dec.png)

`ValidatorImpl#validateMetaConstraint`<br>
![在这里插入图片描述](img01/5f737ed7aa1b43308dbde12f9d8b809f.png)

判断是否是 `fail_fast` 模式：<br>
![在这里插入图片描述](img01/fbc8b85580ef44f6ae843755bbc8b3e7.png)

![在这里插入图片描述](img01/a3117932ac1e4e3da6b00c0eed25395e.png)

`ValidatorImpl#validateInContext`<br>
![在这里插入图片描述](img01/290681aca0254a0bab4908f5292c9949.png)
### 抛出异常 `ModelAttributeMethodProcessor#resolveArgument`
![在这里插入图片描述](img01/bc4ca40e931d475287016c998c12c01d.png)
### 全局异常捕获 `GlobalExceptionHandler#handleBindException`
![在这里插入图片描述](img01/248de03f2d284713b3ac00484b4c79d5.png)
