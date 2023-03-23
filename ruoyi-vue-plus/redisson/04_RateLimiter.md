# Redisson（四） 限流器 RateLimiter
- - -
## 前言
在前几天查漏补缺的时候，看到了 Redisson 限流器 RateLimiter 的内容。Redisson 值得学习的底层原理其实挺多的，所以这次也花了一些时间去看源码和分析，这次关于 Lua 脚本的部分也进行了分析，重要的部分也画了流程图，希望可以帮助理解学习。

## 参考目录
- [Redisson分布式限流器RRateLimiter原理解析](https://github.com/oneone1995/blog/issues/13)
  不错的分析文章，但是是基于旧版本 Redisson，有些源码不太一样，本文是基于 Redisson 最新版 V3.17.2 来进行说明。

## 代码实现
首先需要说明一下，框架【RuoYi-Vue-Plus】中实现限流器也是基于 AOP 方式进行实现的。

### 1、自定义注解 `@RateLimiter`
`com.ruoyi.common.annotation.RateLimiter`<br>
![在这里插入图片描述](img04/ea4bc6158f0545ddb8773297a6b6c545.png)
### 2、切面逻辑类：`RateLimiterAspect`
`com.ruoyi.framework.aspectj.RateLimiterAspect`<br>
![在这里插入图片描述](img04/c9f05545ed4c4858aca96176878af707.png)
### 3、使用方式
使用方式很简单，也是在直接使用注解即可。也可以不设置参数，注解中都有设置默认值。<br>

`com.ruoyi.demo.controller.RedisRateLimiterController#test`<br>
![在这里插入图片描述](img04/7e28496168c44d8bae30892453053b8f.png)<br>

在 `com.ruoyi.demo.controller.RedisRateLimiterController` 中，有三种不同的限流方式示例，本文以第一种全局限流为例进行说明。<br>

## 源码分析
切面逻辑方法 `RateLimiterAspect#doBefore` 比较简单，最重要的部分是获取令牌方法 `RedisUtils#rateLimiter`，所以重点分析这个方法的逻辑。<br>

`RedisUtils#rateLimiter`<br>
![在这里插入图片描述](img04/d3cf56f2f274402a8a95a824c12b5831.png)

方法步骤：
1. 获取限流器；
2. 根据自定义参数初始化限流器；
3. 尝试从限流器中获取令牌，如果存在，返回可用令牌数，如果不存在，返回 -1。

## 源码分析1：初始化限流器
`RedissonRateLimiter#trySetRate`<br>
![在这里插入图片描述](img04/a22af9a353e94da1a6d636444e0d41ef.png)
### Lua 脚本
```lua
redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);
redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);
return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);
```
脚本调用的方法参数对照表：

| 脚本参数名   | Java参数名                     | 参数值       |
|---------|-----------------------------|-----------|
| KEYS[1] | getRawName()                | 限流器 key   |
| ARGV[1] | rate                        | 限流器令牌数    |
| ARGV[2] | unit.toMillis(rateInterval) | 时间间隔      |
| ARGV[3] | type.ordinal()              | 限流器类型（全局） |

### Redis 实际执行结果
```bash
1654399761.149145 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate" "2"
1654399761.149157 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval" "100000"
1654399761.149168 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type" "0"
```

## 源码分析2：尝试获取令牌
`RedissonRateLimiter#tryAcquire`<br>
![在这里插入图片描述](img04/4e20b428bdc9462c8d9c4354e34b2be7.png)<br>

`RedissonRateLimiter#tryAcquireAsync`<br>
![在这里插入图片描述](img04/8898fc624ae54bb4ae2b1479d8741c85.png)<br>
### Lua 脚本
```lua
local rate = redis.call('hget', KEYS[1], 'rate');
local interval = redis.call('hget', KEYS[1], 'interval');
local type = redis.call('hget', KEYS[1], 'type');
assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')

local valueName = KEYS[2];
local permitsName = KEYS[4];
if type == '1' then 
	valueName = KEYS[3];
	permitsName = KEYS[5];
end;

assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate'); 

local currentValue = redis.call('get', valueName); 
if currentValue ~= false then 
	local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); 
	local released = 0; 
	for i, v in ipairs(expiredValues) do 
		local random, permits = struct.unpack('Bc0I', v);
		released = released + permits;
	end; 
	
	if released > 0 then 
		redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); 
		if tonumber(currentValue) + released > tonumber(rate) then 
			currentValue = tonumber(rate) - redis.call('zcard', permitsName); 
		else 
			currentValue = tonumber(currentValue) + released; 
		end; 
		redis.call('set', valueName, currentValue);
	end;
	
	if tonumber(currentValue) < tonumber(ARGV[1]) then 
		local firstValue = redis.call('zrange', permitsName, 0, 0, 'withscores'); 
		return 3 + interval - (tonumber(ARGV[2]) - tonumber(firstValue[2]));
	else 
		redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); 
		redis.call('decrby', valueName, ARGV[1]); 
		return nil; 
	end; 
else 
	redis.call('set', valueName, rate); 
	redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); 
	redis.call('decrby', valueName, ARGV[1]); 
	return nil; 
end;
```
脚本调用的方法参数对照表：

| 脚本参数名   | Java参数名                    | 参数值            |
|---------|----------------------------|----------------|
| KEYS[1] | getRawName()               | 限流器 key        |
| KEYS[2] | getValueName()             | 限流器令牌数量 key    |
| KEYS[3] | getClientValueName()       | 限流器客户端令牌数量 key |
| KEYS[4] | getPermitsName()           | 限流器令牌 key      |
| KEYS[5] | getClientPermitsName()     | 限流器客户端令牌 key   |
| ARGV[1] | value                      | 请求限流器令牌数       |
| ARGV[2] | System.currentTimeMillis() | 当前时间戳          |
| ARGV[3] | random                     | 随机数            |

### Lua 脚本逻辑流程简图（重要）
![在这里插入图片描述](img04/2a1fe2b2147b45a485c81196efcf75eb.jpeg)

由于上面的脚本流程比较多，所以结合流程图来进行分析。<br>

### Lua 脚本代码分析
说明：
- `##` 代表注释说明或者是请求。
- `##(数值)` 代表第几次请求，后面紧跟着的是底层执行的命令。

```lua
### RateType：OVERALL
### rate：2
### rateInterval：100s

local rate = redis.call('hget', KEYS[1], 'rate');
## "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"

local interval = redis.call('hget', KEYS[1], 'interval');
## "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"

local type = redis.call('hget', KEYS[1], 'type');
## "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"

assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')

local valueName = KEYS[2];
local permitsName = KEYS[4];

## 单机模式
if type == '1' then 
	valueName = KEYS[3];
	## "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4"
	
	permitsName = KEYS[5];
	## "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4"
end;

assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate'); 
## 限流器令牌数 >= 默认令牌数 

## tonumber(2) >= tonumber("1")

local currentValue = redis.call('get', valueName); 
##1 "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" --- null
##2 "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" --- currentValue = 1
##3 "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" --- currentValue = 0
##4 "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" --- currentValue = 0

## 第一次请求 currentValue 为空，进入 else 分支
## 第二次请求 currentValue 不为空，进入 if 分支
if currentValue ~= false then 
	local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); 
	## 计算过期时间：系统当前时间 - 时间间隔
		
	##2 "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399668395"
	##3 "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399695739"
	##4 "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399900581"
	
	## 遍历保存需要释放的令牌数量 released
	local released = 0; 
	for i, v in ipairs(expiredValues) do 
		local random, permits = struct.unpack('Bc0I', v);
		released = released + permits;
	end; 
	
	## 有需要释放的令牌
	if released > 0 then 
		redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); 
		## "zremrangebyscore" 移除有序集中，指定分数区间内的所有成员。(ZREMRANGEBYSCORE key min max)
		
		##4 "zremrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399900581"
		
		if tonumber(currentValue) + released > tonumber(rate) then 
		## 当前令牌数 + 释放令牌数 > 指定令牌数	
			currentValue = tonumber(rate) - redis.call('zcard', permitsName); 
			## 当前令牌数 = 指定令牌数 - 指定key中元素数量
		else 
			currentValue = tonumber(currentValue) + released; 
			## 当前令牌数 = 当前令牌数 + 释放令牌数				
		end; 
		redis.call('set', valueName, currentValue);
		##4 "set" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "2"
	end;
	
	## 当前令牌数小于所需令牌数
	if tonumber(currentValue) < tonumber(ARGV[1]) then 
		local firstValue = redis.call('zrange', permitsName, 0, 0, 'withscores'); 
		## 获取 key:"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" 第一个值
		
		##3 "zrange" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "0" "withscores"
				
		return 3 + interval - (tonumber(ARGV[2]) - tonumber(firstValue[2]));
		## 返回剩余时间
		
		##3 3 + 100 - (tonumber(1654399695739) - tonumber(1654399761148)) --- firstValue[2] = 1654399761148
		##3 65,512
		
	else 
		redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); 
		## 存入新令牌数据
		
		##2 "zadd" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "1654399768395" "\bO^\xac\x8a\x84D\xe3\\\x01\x00\x00\x00"
		##4 "zadd" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "1654400000581" "\b\x8e\x12\\V\xce\x16h\xff\x01\x00\x00\x00"
		
		redis.call('decrby', valueName, ARGV[1]); 
		## 将 key:"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" 存储的令牌数减 1
		
		##2 "decrby" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "1"
		##4 "decrby" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "1"
		
		return nil; 
	end; 
else 
	redis.call('set', valueName, rate); 
	## 在 key:"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" 存储限流器令牌数
	
	##1 "set" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "2"
	
	redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); 
	## struct.pack 数据打包 struct.pack(格式化字符串，需要打包的数据1，需要打包的数据2 …)
	## 'Bc0I' 
	## "B" an unsigned char.
	## "cn" a sequence of exactly n chars corresponding to a single Lua string 
		(if n <= 0 then 
			for packing - the string length is taken, 
				unpacking - the number value of the previous unpacked value which is not returned).
	## "I" an unsigned int (4 bytes).
	
	##1 "zadd" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "1654399768395" "\bO^\xac\x8a\x84D\xe3\\\x01\x00\x00\x00"
	
	
	redis.call('decrby', valueName, ARGV[1]); 
	## 将 key:"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" 存储的令牌数减 1
	
	##1 "decrby" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "1"
	
	return nil; 
end;
```

### Redis 实际执行结果
在脚本中有很多判断的分支结构，为了进入不同的分支因此进行了四次请求测试：（限流器的参数：可用令牌数 2，时间间隔 100 秒）
1. 第一次请求：发放令牌，正常访问；
2. 第二次请求：发放令牌，正常访问；
3. 第三次请求：限制访问；
4. 前三次请求在时间间隔 100s 内完成；
5. 第四次请求：间隔超过100s，发放令牌，正常访问；

```bash
### 第一次请求：
"5" 
"rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" KEYS[1] getRawName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" KEYS[2] getValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[3] getClientValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" KEYS[4] getPermitsName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[5] getClientPermitsName()
"1" ARGV[1] value
"1654399761148" ARGV[2] System.currentTimeMillis()
"=y\xa3\"\x83\x00\xf1z" ARGV[3] random

### 执行结果：
1654399761.149821 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654399761.149838 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654399761.149850 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654399761.149864 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654399761.149875 [5 lua] "set" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "2"
1654399761.149893 [5 lua] "zadd" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "1654399761148" "\b=y\xa3\"\x83\x00\xf1z\x01\x00\x00\x00"
1654399761.149914 [5 lua] "decrby" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "1"

-------------------------------------

### 第二次请求：
"5" 
"rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" KEYS[1] getRawName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" KEYS[2] getValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[3] getClientValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" KEYS[4] getPermitsName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[5] getClientPermitsName()
"1" ARGV[1]
"1654399768395" ARGV[2]
"O^\xac\x8a\x84D\xe3\\" ARGV[3] random

### 执行结果：
1654399768.397061 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654399768.397078 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654399768.397089 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654399768.397106 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654399768.397125 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399668395"
1654399768.397150 [5 lua] "zadd" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "1654399768395" "\bO^\xac\x8a\x84D\xe3\\\x01\x00\x00\x00"
1654399768.397174 [5 lua] "decrby" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "1"

-------------------------------------

### 第三次请求：
"5" 
"rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" KEYS[1] getRawName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" KEYS[2] getValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[3] getClientValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" KEYS[4] getPermitsName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[5] getClientPermitsName()
"1" ARGV[1]
"1654399795739" ARGV[2]
"yJ\xb1\xea\x99\x16\x9f\xcf" ARGV[3] random

### 执行结果：
1654399795.740534 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654399795.740548 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654399795.740558 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654399795.740571 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654399795.740586 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399695739"
1654399795.740603 [5 lua] "zrange" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "0" "withscores"

-------------------------------------

### 第四次请求：
"5" 
"rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" KEYS[1] getRawName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" KEYS[2] getValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[3] getClientValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" KEYS[4] getPermitsName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[5] getClientPermitsName()
"1" ARGV[1]
"1654400000581" ARGV[2]
"\x8e\x12\\V\xce\x16h\xff" ARGV[3] random

### 执行结果：
1654400000.583083 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654400000.583098 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654400000.583109 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654400000.583123 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654400000.583140 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399900581"
1654400000.583167 [5 lua] "zremrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399900581"
1654400000.583186 [5 lua] "set" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "2"
1654400000.583202 [5 lua] "zadd" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "1654400000581" "\b\x8e\x12\\V\xce\x16h\xff\x01\x00\x00\x00"
1654400000.583243 [5 lua] "decrby" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "1"
```

## 源码分析3：返回可用令牌的数量
`RedissonRateLimiter#availablePermits`
![在这里插入图片描述](img04/dac18b1c522f44b2a63b64956b5aa3cc.png)
### Lua 脚本
```lua
local rate = redis.call('hget', KEYS[1], 'rate');
local interval = redis.call('hget', KEYS[1], 'interval');
local type = redis.call('hget', KEYS[1], 'type');
assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')

local valueName = KEYS[2];
local permitsName = KEYS[4];
if type == '1' then 
	valueName = KEYS[3];
	permitsName = KEYS[5];
end;

local currentValue = redis.call('get', valueName); 
if currentValue == false then 
	redis.call('set', valueName, rate); 
	return rate; 
else 
	local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[1]) - interval); 
	local released = 0; 
	for i, v in ipairs(expiredValues) do 
		local random, permits = struct.unpack('Bc0I', v);
		released = released + permits;
	end; 
	
	if released > 0 then 
		redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[1]) - interval); 
		currentValue = tonumber(currentValue) + released; 
		redis.call('set', valueName, currentValue);
	end;
	
	return currentValue; 
end;
```
脚本调用的方法参数对照表：

| 脚本参数名   | Java参数名                    | 参数值            |
|---------|----------------------------|----------------|
| KEYS[1] | getRawName()               | 限流器 key        |
| KEYS[2] | getValueName()             | 限流器令牌数量 key    |
| KEYS[3] | getClientValueName()       | 限流器客户端令牌数量 key |
| KEYS[4] | getPermitsName()           | 限流器令牌 key      |
| KEYS[5] | getClientPermitsName()     | 限流器客户端令牌 key   |
| ARGV[1] | System.currentTimeMillis() | 当前时间戳          |

### Lua 脚本代码分析
该方法是在尝试获取令牌成功后执行的方法，源码中的逻辑实际上和尝试获取令牌中的逻辑是一样的，所以理解了上面的逻辑之后这里也就能够很容易梳理清楚。

说明：<br>
- `##` 代表注释说明或者是请求。
- `##(数值)` 代表第几次请求，后面紧跟着的是底层执行的命令。

```lua
### RateType：OVERALL
### rate：2
### rateInterval：100s

local rate = redis.call('hget', KEYS[1], 'rate');
## "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"

local interval = redis.call('hget', KEYS[1], 'interval');
## "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"

local type = redis.call('hget', KEYS[1], 'type');
## "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"

assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')

local valueName = KEYS[2];
local permitsName = KEYS[4];

## 单机模式
if type == '1' then 
	valueName = KEYS[3];
	## "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4"
	
	permitsName = KEYS[5];
	## "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4"
end;

local currentValue = redis.call('get', valueName); 
##1 "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" --- currentValue = 1
##2 "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" --- currentValue = 0
##4 "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" --- currentValue = 1

## 第一次请求 currentValue 不为空，进入 else 分支
if currentValue == false then 
	redis.call('set', valueName, rate); 
	return rate; 
else 
	local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[1]) - interval); 
	## 计算过期时间：系统当前时间 - 时间间隔
	
	##1 "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399661149"
	##2 "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399668397"
	##4 "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399900582"
	
	## 遍历保存需要释放的令牌数量 released
	local released = 0; 
	for i, v in ipairs(expiredValues) do 
		local random, permits = struct.unpack('Bc0I', v);
		released = released + permits;
	end; 
	
	## 有需要释放的令牌
	if released > 0 then 
		redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[1]) - interval); 
		currentValue = tonumber(currentValue) + released; 
		redis.call('set', valueName, currentValue);
	end;
	
	return currentValue; 
	## 返回当前可用令牌数
end;
```

### Redis 实际执行结果
四次请求测试：（限流器的参数：可用令牌数 2，时间间隔 100 秒）
1. 第一次请求：发放令牌，正常访问；
2. 第二次请求：发放令牌，正常访问；
3. 第三次请求：没有进入该方法逻辑，限制访问；
4. 前三次请求在时间间隔 100s 内完成；
5. 第四次请求：间隔超过100s，发放令牌，正常访问；

```bash
### 第一次请求：
"5" 
"rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" KEYS[1] getRawName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" KEYS[2] getValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[3] getClientValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" KEYS[4] getPermitsName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[5] getClientPermitsName()
"1654399761149" ARGV[1] System.currentTimeMillis()

### 执行结果：
1654399761.150591 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654399761.150604 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654399761.150615 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654399761.150638 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654399761.150669 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399661149"

-------------------------------------

### 第二次请求：
"5" 
"rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" KEYS[1] getRawName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" KEYS[2] getValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[3] getClientValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" KEYS[4] getPermitsName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[5] getClientPermitsName()
"1654399768397" ARGV[1] System.currentTimeMillis()

### 执行结果：
1654399768.398240 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654399768.398256 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654399768.398269 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654399768.398281 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654399768.398304 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399668397"

-------------------------------------

### 第四次请求：
"5" 
"rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" KEYS[1] getRawName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" KEYS[2] getValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[3] getClientValueName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" KEYS[4] getPermitsName()
"{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" KEYS[5] getClientPermitsName()
"1654400000582" ARGV[1] System.currentTimeMillis()

### 执行结果：
1654400000.584141 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654400000.584168 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654400000.584199 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654400000.584210 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654400000.584239 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399900582"
```
## 附文
### Redis monitor 模式下控制台输出完整内容

```bash
127.0.0.1:6379> monitor
OK
1654399759.976596 [5 127.0.0.1:2245] "ping"
1654399761.146589 [5 127.0.0.1:3600] "GET" "Authorization:login:token:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654399761.147189 [5 127.0.0.1:3601] "GET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654399761.147655 [5 127.0.0.1:3602] "PTTL" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654399761.148108 [5 127.0.0.1:3597] "SET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "\"1654399761146\""
1654399761.148451 [5 127.0.0.1:3596] "EVAL" "local result = 0;for j = 1, #KEYS, 1 do local expireSet; if ARGV[2] ~= '' then expireSet = redis.call('pexpire', KEYS[j], ARGV[1], ARGV[2]); else expireSet = redis.call('pexpire', KEYS[j], ARGV[1]); end; if expireSet == 1 then result = expireSet;end; end; return result; " "1" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "86099000" ""
1654399761.148521 [5 lua] "pexpire" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "86099000"
1654399761.149107 [5 127.0.0.1:3594] "EVAL" "redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);" "1" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "2" "100000" "0"
1654399761.149145 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate" "2"
1654399761.149157 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval" "100000"
1654399761.149168 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type" "0"
1654399761.149518 [5 127.0.0.1:3598] "EVAL" "local rate = redis.call('hget', KEYS[1], 'rate');local interval = redis.call('hget', KEYS[1], 'interval');local type = redis.call('hget', KEYS[1], 'type');assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')local valueName = KEYS[2];local permitsName = KEYS[4];if type == '1' then valueName = KEYS[3];permitsName = KEYS[5];end;assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate'); local currentValue = redis.call('get', valueName); if currentValue ~= false then local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); local released = 0; for i, v in ipairs(expiredValues) do local random, permits = struct.unpack('Bc0I', v);released = released + permits;end; if released > 0 then redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); if tonumber(currentValue) + released > tonumber(rate) then currentValue = tonumber(rate) - redis.call('zcard', permitsName); else currentValue = tonumber(currentValue) + released; end; redis.call('set', valueName, currentValue);end;if tonumber(currentValue) < tonumber(ARGV[1]) then local firstValue = redis.call('zrange', permitsName, 0, 0, 'withscores'); return 3 + interval - (tonumber(ARGV[2]) - tonumber(firstValue[2]));else redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); redis.call('decrby', valueName, ARGV[1]); return nil; end; else redis.call('set', valueName, rate); redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); redis.call('decrby', valueName, ARGV[1]); return nil; end;" "5" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" "1" "1654399761148" "=y\xa3\"\x83\x00\xf1z"
1654399761.149821 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654399761.149838 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654399761.149850 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654399761.149864 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654399761.149875 [5 lua] "set" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "2"
1654399761.149893 [5 lua] "zadd" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "1654399761148" "\b=y\xa3\"\x83\x00\xf1z\x01\x00\x00\x00"
1654399761.149914 [5 lua] "decrby" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "1"
1654399761.150419 [5 127.0.0.1:3595] "EVAL" "local rate = redis.call('hget', KEYS[1], 'rate');local interval = redis.call('hget', KEYS[1], 'interval');local type = redis.call('hget', KEYS[1], 'type');assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')local valueName = KEYS[2];local permitsName = KEYS[4];if type == '1' then valueName = KEYS[3];permitsName = KEYS[5];end;local currentValue = redis.call('get', valueName); if currentValue == false then redis.call('set', valueName, rate); return rate; else local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[1]) - interval); local released = 0; for i, v in ipairs(expiredValues) do local random, permits = struct.unpack('Bc0I', v);released = released + permits;end; if released > 0 then redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[1]) - interval); currentValue = tonumber(currentValue) + released; redis.call('set', valueName, currentValue);end;return currentValue; end;" "5" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" "1654399761149"
1654399761.150591 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654399761.150604 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654399761.150615 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654399761.150638 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654399761.150669 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399661149"
1654399765.607634 [5 127.0.0.1:3597] "PING"
1654399765.607696 [5 127.0.0.1:3598] "PING"
1654399765.607702 [5 127.0.0.1:3594] "PING"
1654399765.607737 [5 127.0.0.1:3599] "PING"
1654399765.607878 [5 127.0.0.1:3595] "PING"
1654399765.607949 [5 127.0.0.1:3596] "PING"
1654399765.608275 [5 127.0.0.1:3601] "PING"
1654399765.608283 [5 127.0.0.1:3600] "PING"
1654399765.608288 [5 127.0.0.1:3602] "PING"
1654399768.392736 [5 127.0.0.1:3600] "GET" "Authorization:login:token:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654399768.393194 [5 127.0.0.1:3601] "GET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654399768.393559 [5 127.0.0.1:3602] "PTTL" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654399768.394062 [5 127.0.0.1:3597] "SET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "\"1654399768393\""
1654399768.394814 [5 127.0.0.1:3596] "EVAL" "local result = 0;for j = 1, #KEYS, 1 do local expireSet; if ARGV[2] ~= '' then expireSet = redis.call('pexpire', KEYS[j], ARGV[1], ARGV[2]); else expireSet = redis.call('pexpire', KEYS[j], ARGV[1]); end; if expireSet == 1 then result = expireSet;end; end; return result; " "1" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "86091000" ""
1654399768.394897 [5 lua] "pexpire" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "86091000"
1654399768.395935 [5 127.0.0.1:3594] "EVAL" "redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);" "1" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "2" "100000" "0"
1654399768.396008 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate" "2"
1654399768.396035 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval" "100000"
1654399768.396073 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type" "0"
1654399768.396622 [5 127.0.0.1:3598] "EVAL" "local rate = redis.call('hget', KEYS[1], 'rate');local interval = redis.call('hget', KEYS[1], 'interval');local type = redis.call('hget', KEYS[1], 'type');assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')local valueName = KEYS[2];local permitsName = KEYS[4];if type == '1' then valueName = KEYS[3];permitsName = KEYS[5];end;assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate'); local currentValue = redis.call('get', valueName); if currentValue ~= false then local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); local released = 0; for i, v in ipairs(expiredValues) do local random, permits = struct.unpack('Bc0I', v);released = released + permits;end; if released > 0 then redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); if tonumber(currentValue) + released > tonumber(rate) then currentValue = tonumber(rate) - redis.call('zcard', permitsName); else currentValue = tonumber(currentValue) + released; end; redis.call('set', valueName, currentValue);end;if tonumber(currentValue) < tonumber(ARGV[1]) then local firstValue = redis.call('zrange', permitsName, 0, 0, 'withscores'); return 3 + interval - (tonumber(ARGV[2]) - tonumber(firstValue[2]));else redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); redis.call('decrby', valueName, ARGV[1]); return nil; end; else redis.call('set', valueName, rate); redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); redis.call('decrby', valueName, ARGV[1]); return nil; end;" "5" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" "1" "1654399768395" "O^\xac\x8a\x84D\xe3\\"
1654399768.397061 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654399768.397078 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654399768.397089 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654399768.397106 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654399768.397125 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399668395"
1654399768.397150 [5 lua] "zadd" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "1654399768395" "\bO^\xac\x8a\x84D\xe3\\\x01\x00\x00\x00"
1654399768.397174 [5 lua] "decrby" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "1"
1654399768.398019 [5 127.0.0.1:3595] "EVAL" "local rate = redis.call('hget', KEYS[1], 'rate');local interval = redis.call('hget', KEYS[1], 'interval');local type = redis.call('hget', KEYS[1], 'type');assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')local valueName = KEYS[2];local permitsName = KEYS[4];if type == '1' then valueName = KEYS[3];permitsName = KEYS[5];end;local currentValue = redis.call('get', valueName); if currentValue == false then redis.call('set', valueName, rate); return rate; else local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[1]) - interval); local released = 0; for i, v in ipairs(expiredValues) do local random, permits = struct.unpack('Bc0I', v);released = released + permits;end; if released > 0 then redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[1]) - interval); currentValue = tonumber(currentValue) + released; redis.call('set', valueName, currentValue);end;return currentValue; end;" "5" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" "1654399768397"
1654399768.398240 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654399768.398256 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654399768.398269 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654399768.398281 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654399768.398304 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399668397"
1654399769.989952 [5 127.0.0.1:2245] "ping"
1654399779.988185 [5 127.0.0.1:2245] "ping"
1654399785.623602 [5 127.0.0.1:3600] "EVAL" "if redis.call('setnx', KEYS[6], ARGV[4]) == 0 then return -1;end;redis.call('expire', KEYS[6], ARGV[3]); local expiredKeys1 = redis.call('zrangebyscore', KEYS[2], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys1) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;if #expiredKeys1 > 0 then redis.call('zrem', KEYS[5], unpack(expiredKeys1)); redis.call('zrem', KEYS[3], unpack(expiredKeys1)); redis.call('zrem', KEYS[2], unpack(expiredKeys1)); redis.call('hdel', KEYS[1], unpack(expiredKeys1)); end; local expiredKeys2 = redis.call('zrangebyscore', KEYS[3], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys2) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;if #expiredKeys2 > 0 then redis.call('zrem', KEYS[5], unpack(expiredKeys2)); redis.call('zrem', KEYS[3], unpack(expiredKeys2)); redis.call('zrem', KEYS[2], unpack(expiredKeys2)); redis.call('hdel', KEYS[1], unpack(expiredKeys2)); end; return #expiredKeys1 + #expiredKeys2;" "6" "redissonCacheMap" "redisson__timeout__set:{redissonCacheMap}" "redisson__idle__set:{redissonCacheMap}" "redisson_map_cache_expired:{redissonCacheMap}" "redisson__map_cache__last_access__set:{redissonCacheMap}" "redisson__execute_task_once_latch:{redissonCacheMap}" "1654399785623" "100" "30" "1"
1654399785.623869 [5 lua] "setnx" "redisson__execute_task_once_latch:{redissonCacheMap}" "1"
1654399785.623883 [5 lua] "expire" "redisson__execute_task_once_latch:{redissonCacheMap}" "30"
1654399785.623895 [5 lua] "zrangebyscore" "redisson__timeout__set:{redissonCacheMap}" "0" "1654399785623" "limit" "0" "100"
1654399785.623910 [5 lua] "zrangebyscore" "redisson__idle__set:{redissonCacheMap}" "0" "1654399785623" "limit" "0" "100"
1654399785.639575 [5 127.0.0.1:3601] "EVAL" "if redis.call('setnx', KEYS[6], ARGV[4]) == 0 then return -1;end;redis.call('expire', KEYS[6], ARGV[3]); local expiredKeys1 = redis.call('zrangebyscore', KEYS[2], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys1) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;if #expiredKeys1 > 0 then redis.call('zrem', KEYS[5], unpack(expiredKeys1)); redis.call('zrem', KEYS[3], unpack(expiredKeys1)); redis.call('zrem', KEYS[2], unpack(expiredKeys1)); redis.call('hdel', KEYS[1], unpack(expiredKeys1)); end; local expiredKeys2 = redis.call('zrangebyscore', KEYS[3], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys2) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;if #expiredKeys2 > 0 then redis.call('zrem', KEYS[5], unpack(expiredKeys2)); redis.call('zrem', KEYS[3], unpack(expiredKeys2)); redis.call('zrem', KEYS[2], unpack(expiredKeys2)); redis.call('hdel', KEYS[1], unpack(expiredKeys2)); end; return #expiredKeys1 + #expiredKeys2;" "6" "testCache" "redisson__timeout__set:{testCache}" "redisson__idle__set:{testCache}" "redisson_map_cache_expired:{testCache}" "redisson__map_cache__last_access__set:{testCache}" "redisson__execute_task_once_latch:{testCache}" "1654399785638" "100" "30" "1"
1654399785.639777 [5 lua] "setnx" "redisson__execute_task_once_latch:{testCache}" "1"
1654399785.639789 [5 lua] "expire" "redisson__execute_task_once_latch:{testCache}" "30"
1654399785.639800 [5 lua] "zrangebyscore" "redisson__timeout__set:{testCache}" "0" "1654399785638" "limit" "0" "100"
1654399785.639813 [5 lua] "zrangebyscore" "redisson__idle__set:{testCache}" "0" "1654399785638" "limit" "0" "100"
1654399789.987030 [5 127.0.0.1:2245] "ping"
1654399795.702504 [5 127.0.0.1:3598] "PING"
1654399795.702529 [5 127.0.0.1:3597] "PING"
1654399795.702605 [5 127.0.0.1:3594] "PING"
1654399795.702610 [5 127.0.0.1:3599] "PING"
1654399795.702640 [5 127.0.0.1:3595] "PING"
1654399795.702646 [5 127.0.0.1:3596] "PING"
1654399795.702677 [5 127.0.0.1:3600] "PING"
1654399795.702684 [5 127.0.0.1:3601] "PING"
1654399795.702703 [5 127.0.0.1:3602] "PING"
1654399795.737068 [5 127.0.0.1:3602] "GET" "Authorization:login:token:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654399795.737552 [5 127.0.0.1:3597] "GET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654399795.737991 [5 127.0.0.1:3596] "PTTL" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654399795.738451 [5 127.0.0.1:3594] "SET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "\"1654399795737\""
1654399795.738831 [5 127.0.0.1:3598] "EVAL" "local result = 0;for j = 1, #KEYS, 1 do local expireSet; if ARGV[2] ~= '' then expireSet = redis.call('pexpire', KEYS[j], ARGV[1], ARGV[2]); else expireSet = redis.call('pexpire', KEYS[j], ARGV[1]); end; if expireSet == 1 then result = expireSet;end; end; return result; " "1" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "86063000" ""
1654399795.738903 [5 lua] "pexpire" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "86063000"
1654399795.739696 [5 127.0.0.1:3595] "EVAL" "redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);" "1" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "2" "100000" "0"
1654399795.739782 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate" "2"
1654399795.739802 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval" "100000"
1654399795.739814 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type" "0"
1654399795.740261 [5 127.0.0.1:3600] "EVAL" "local rate = redis.call('hget', KEYS[1], 'rate');local interval = redis.call('hget', KEYS[1], 'interval');local type = redis.call('hget', KEYS[1], 'type');assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')local valueName = KEYS[2];local permitsName = KEYS[4];if type == '1' then valueName = KEYS[3];permitsName = KEYS[5];end;assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate'); local currentValue = redis.call('get', valueName); if currentValue ~= false then local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); local released = 0; for i, v in ipairs(expiredValues) do local random, permits = struct.unpack('Bc0I', v);released = released + permits;end; if released > 0 then redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); if tonumber(currentValue) + released > tonumber(rate) then currentValue = tonumber(rate) - redis.call('zcard', permitsName); else currentValue = tonumber(currentValue) + released; end; redis.call('set', valueName, currentValue);end;if tonumber(currentValue) < tonumber(ARGV[1]) then local firstValue = redis.call('zrange', permitsName, 0, 0, 'withscores'); return 3 + interval - (tonumber(ARGV[2]) - tonumber(firstValue[2]));else redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); redis.call('decrby', valueName, ARGV[1]); return nil; end; else redis.call('set', valueName, rate); redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); redis.call('decrby', valueName, ARGV[1]); return nil; end;" "5" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" "1" "1654399795739" "yJ\xb1\xea\x99\x16\x9f\xcf"
1654399795.740534 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654399795.740548 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654399795.740558 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654399795.740571 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654399795.740586 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399695739"
1654399795.740603 [5 lua] "zrange" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "0" "withscores"
1654399799.979165 [5 127.0.0.1:2245] "ping"
1654399809.984335 [5 127.0.0.1:2245] "ping"
1654399819.988513 [5 127.0.0.1:2245] "ping"
1654399825.808971 [5 127.0.0.1:3598] "PING"
1654399825.809043 [5 127.0.0.1:3597] "PING"
1654399825.809051 [5 127.0.0.1:3594] "PING"
1654399825.809073 [5 127.0.0.1:3599] "PING"
1654399825.809079 [5 127.0.0.1:3595] "PING"
1654399825.809085 [5 127.0.0.1:3596] "PING"
1654399825.809109 [5 127.0.0.1:3600] "PING"
1654399825.809138 [5 127.0.0.1:3601] "PING"
1654399825.809187 [5 127.0.0.1:3602] "PING"
1654399829.981582 [5 127.0.0.1:2245] "ping"
1654399839.989779 [5 127.0.0.1:2245] "ping"
1654399849.986597 [5 127.0.0.1:2245] "ping"
1654399855.903082 [5 127.0.0.1:3597] "PING"
1654399855.903091 [5 127.0.0.1:3598] "PING"
1654399855.903180 [5 127.0.0.1:3599] "PING"
1654399855.903187 [5 127.0.0.1:3594] "PING"
1654399855.903192 [5 127.0.0.1:3595] "PING"
1654399855.903237 [5 127.0.0.1:3602] "PING"
1654399855.903245 [5 127.0.0.1:3596] "PING"
1654399855.903251 [5 127.0.0.1:3601] "PING"
1654399855.903305 [5 127.0.0.1:3600] "PING"
1654399859.980284 [5 127.0.0.1:2245] "ping"
1654399869.977741 [5 127.0.0.1:2245] "ping"
1654399879.987324 [5 127.0.0.1:2245] "ping"
1654399886.005897 [5 127.0.0.1:3597] "PING"
1654399886.005966 [5 127.0.0.1:3598] "PING"
1654399886.006169 [5 127.0.0.1:3599] "PING"
1654399886.006248 [5 127.0.0.1:3594] "PING"
1654399886.006326 [5 127.0.0.1:3595] "PING"
1654399886.006334 [5 127.0.0.1:3596] "PING"
1654399886.006399 [5 127.0.0.1:3600] "PING"
1654399886.006437 [5 127.0.0.1:3601] "PING"
1654399886.006544 [5 127.0.0.1:3602] "PING"
1654399889.979663 [5 127.0.0.1:2245] "ping"
1654399899.982645 [5 127.0.0.1:2245] "ping"
1654399909.989858 [5 127.0.0.1:2245] "ping"
1654399916.106729 [5 127.0.0.1:3597] "PING"
1654399916.106820 [5 127.0.0.1:3598] "PING"
1654399916.106831 [5 127.0.0.1:3599] "PING"
1654399916.106838 [5 127.0.0.1:3594] "PING"
1654399916.106843 [5 127.0.0.1:3595] "PING"
1654399916.106913 [5 127.0.0.1:3596] "PING"
1654399916.107132 [5 127.0.0.1:3600] "PING"
1654399916.107167 [5 127.0.0.1:3601] "PING"
1654399916.107192 [5 127.0.0.1:3602] "PING"
1654399919.986682 [5 127.0.0.1:2245] "ping"
1654399929.986132 [5 127.0.0.1:2245] "ping"
1654399939.976354 [5 127.0.0.1:2245] "ping"
1654399946.202455 [5 127.0.0.1:3594] "PING"
1654399946.202500 [5 127.0.0.1:3598] "PING"
1654399946.202508 [5 127.0.0.1:3597] "PING"
1654399946.202514 [5 127.0.0.1:3599] "PING"
1654399946.202519 [5 127.0.0.1:3595] "PING"
1654399946.202564 [5 127.0.0.1:3596] "PING"
1654399946.202591 [5 127.0.0.1:3601] "PING"
1654399946.202699 [5 127.0.0.1:3600] "PING"
1654399946.202753 [5 127.0.0.1:3602] "PING"
1654399949.979994 [5 127.0.0.1:2245] "ping"
1654399959.976689 [5 127.0.0.1:2245] "ping"
1654399969.979517 [5 127.0.0.1:2245] "ping"
1654399976.309418 [5 127.0.0.1:3597] "PING"
1654399976.309433 [5 127.0.0.1:3598] "PING"
1654399976.309438 [5 127.0.0.1:3594] "PING"
1654399976.309505 [5 127.0.0.1:3595] "PING"
1654399976.309510 [5 127.0.0.1:3599] "PING"
1654399976.309516 [5 127.0.0.1:3596] "PING"
1654399976.309554 [5 127.0.0.1:3600] "PING"
1654399976.309801 [5 127.0.0.1:3601] "PING"
1654399976.309808 [5 127.0.0.1:3602] "PING"
1654399979.978042 [5 127.0.0.1:2245] "ping"
1654399989.976115 [5 127.0.0.1:2245] "ping"
1654399999.981335 [5 127.0.0.1:2245] "ping"
1654400000.579232 [5 127.0.0.1:3601] "GET" "Authorization:login:token:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654400000.579896 [5 127.0.0.1:3602] "GET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654400000.580533 [5 127.0.0.1:3597] "PTTL" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY"
1654400000.581023 [5 127.0.0.1:3596] "SET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "\"1654400000579\""
1654400000.581518 [5 127.0.0.1:3594] "EVAL" "local result = 0;for j = 1, #KEYS, 1 do local expireSet; if ARGV[2] ~= '' then expireSet = redis.call('pexpire', KEYS[j], ARGV[1], ARGV[2]); else expireSet = redis.call('pexpire', KEYS[j], ARGV[1]); end; if expireSet == 1 then result = expireSet;end; end; return result; " "1" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "85858000" ""
1654400000.581607 [5 lua] "pexpire" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJZZ2REUlE1V0hPSGpKTUlDV3pvc05MQzFHd3VZM0pydiJ9.Quap0ssO6YgymJ0MyJ2UhcXQfrWUcdzu9LRc3SDdNNY" "85858000"
1654400000.582299 [5 127.0.0.1:3598] "EVAL" "redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);" "1" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "2" "100000" "0"
1654400000.582340 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate" "2"
1654400000.582354 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval" "100000"
1654400000.582365 [5 lua] "hsetnx" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type" "0"
1654400000.582825 [5 127.0.0.1:3595] "EVAL" "local rate = redis.call('hget', KEYS[1], 'rate');local interval = redis.call('hget', KEYS[1], 'interval');local type = redis.call('hget', KEYS[1], 'type');assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')local valueName = KEYS[2];local permitsName = KEYS[4];if type == '1' then valueName = KEYS[3];permitsName = KEYS[5];end;assert(tonumber(rate) >= tonumber(ARGV[1]), 'Requested permits amount could not exceed defined rate'); local currentValue = redis.call('get', valueName); if currentValue ~= false then local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); local released = 0; for i, v in ipairs(expiredValues) do local random, permits = struct.unpack('Bc0I', v);released = released + permits;end; if released > 0 then redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[2]) - interval); if tonumber(currentValue) + released > tonumber(rate) then currentValue = tonumber(rate) - redis.call('zcard', permitsName); else currentValue = tonumber(currentValue) + released; end; redis.call('set', valueName, currentValue);end;if tonumber(currentValue) < tonumber(ARGV[1]) then local firstValue = redis.call('zrange', permitsName, 0, 0, 'withscores'); return 3 + interval - (tonumber(ARGV[2]) - tonumber(firstValue[2]));else redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); redis.call('decrby', valueName, ARGV[1]); return nil; end; else redis.call('set', valueName, rate); redis.call('zadd', permitsName, ARGV[2], struct.pack('Bc0I', string.len(ARGV[3]), ARGV[3], ARGV[1])); redis.call('decrby', valueName, ARGV[1]); return nil; end;" "5" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" "1" "1654400000581" "\x8e\x12\\V\xce\x16h\xff"
1654400000.583083 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654400000.583098 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654400000.583109 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654400000.583123 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654400000.583140 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399900581"
1654400000.583167 [5 lua] "zremrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399900581"
1654400000.583186 [5 lua] "set" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "2"
1654400000.583202 [5 lua] "zadd" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "1654400000581" "\b\x8e\x12\\V\xce\x16h\xff\x01\x00\x00\x00"
1654400000.583243 [5 lua] "decrby" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "1"
1654400000.583828 [5 127.0.0.1:3600] "EVAL" "local rate = redis.call('hget', KEYS[1], 'rate');local interval = redis.call('hget', KEYS[1], 'interval');local type = redis.call('hget', KEYS[1], 'type');assert(rate ~= false and interval ~= false and type ~= false, 'RateLimiter is not initialized')local valueName = KEYS[2];local permitsName = KEYS[4];if type == '1' then valueName = KEYS[3];permitsName = KEYS[5];end;local currentValue = redis.call('get', valueName); if currentValue == false then redis.call('set', valueName, rate); return rate; else local expiredValues = redis.call('zrangebyscore', permitsName, 0, tonumber(ARGV[1]) - interval); local released = 0; for i, v in ipairs(expiredValues) do local random, permits = struct.unpack('Bc0I', v);released = released + permits;end; if released > 0 then redis.call('zremrangebyscore', permitsName, 0, tonumber(ARGV[1]) - interval); currentValue = tonumber(currentValue) + released; redis.call('set', valueName, currentValue);end;return currentValue; end;" "5" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value:89a74653-01ad-4f25-9698-0d891ef076f4" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits:89a74653-01ad-4f25-9698-0d891ef076f4" "1654400000582"
1654400000.584141 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "rate"
1654400000.584168 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "interval"
1654400000.584199 [5 lua] "hget" "rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test" "type"
1654400000.584210 [5 lua] "get" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:value"
1654400000.584239 [5 lua] "zrangebyscore" "{rate_limit:com.ruoyi.demo.controller.RedisRateLimiterController-test}:permits" "0" "1654399900582"
```
