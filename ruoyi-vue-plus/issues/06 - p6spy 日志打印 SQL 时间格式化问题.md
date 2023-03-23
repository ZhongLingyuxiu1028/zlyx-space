# 问题笔记 06 - p6spy 日志打印 SQL 时间格式化问题

## 前言
今天遇到一个之前没有注意的问题，群里有人提及控制台打印的日志格式有问题，无法直接执行。经过一番排查之后找到了解决的方法，在此记录一下。
## 参考目录
- [p6spy sql打印时，日期格式显示问题](https://segmentfault.com/q/1010000020263100)
  谷歌大法找到的解决方案。
- [p6spy 官方文档 - 配置使用说明](https://p6spy.readthedocs.io/en/latest/configandusage.html)
  官方文档也写了相关的配置，只是比较难找。

## 问题说明
如下图：控制台打印的日志时间格式是 `yyyy-MM-dd'T'HH:mm:ss.SSSZ`，而非我们常用的 `yyyy-MM-dd HH:mm:ss`。
![在这里插入图片描述](img06/c13deff38bb24e5ea2f73f0f4c7cf0dc.jpeg)
## 问题解决方法
修改 p6spy 配置文件 `spy.properties`
![在这里插入图片描述](img06/2a539fccf2504f56b5892d8c8ec92b89.png)

增加配置：

```bash
databaseDialectTimestampFormat=yyyy-MM-dd HH:mm:ss
```
执行效果：<br>
![在这里插入图片描述](img06/2ae2d58a8f17402289c7793ce5696607.png)
## 问题分析
### 1、官方文档说明

> 官方文档说明：<br>
> ![在这里插入图片描述](img06/a97e55b981974352858d1964f0af888b.png)
### 2、默认配置
p6spy 默认配置 `P6SpyOptions`<br>
![在这里插入图片描述](img06/1c17bbe74c71470b8dee752e51f5459a.png)
### 3、配置加载流程简单说明
### 3.1、配置文件加载 `SpyDotProperties#SpyDotProperties`
![在这里插入图片描述](img06/98d64c4a78664d3f911ef2a5043866e9.png)
### 3.2、注册模块 `P6ModuleManager#registerModule`
![在这里插入图片描述](img06/6433422245004465b99a50c35616f4d7.png)
### 3.3、加载配置 `P6ModuleManager#loadOptions`
![在这里插入图片描述](img06/21a5ba0fd481476eb2c868bfc6e682a6.png)

![在这里插入图片描述](img06/746d1e06fd124205bbae97ef783f5016.png)

### 4、格式化方法 `Value#convertToString`
![在这里插入图片描述](img06/a758e9e1ae304cd9b02aa68cab58470f.png)

![在这里插入图片描述](img06/0e9d51d4e8134121846c428de39d80bb.png)
