# Redisson（八）RedissonMapCache 缓存流程分析（上）
- - -
## 前言
最近和 [狮子大佬](https://blog.csdn.net/weixin_40461281?type=blog) 研究讨论了一下 Redisson 集成 Spring Cache 在底层实现的缓存相关操作。

在 [上一篇笔记](07_Spring_Cache.md) 中，着重点在于前面自动装配流程，对于缓存的相关操作分析的内容比较少（源码分析部分第 4 点有提及），因此再在此基础上深入来分析一下缓存的 Lua 脚本执行流程。
## 参考目录
- [SpringBoot 改造 Spring-Cache 注解实现 支持注解指定扩展参数](https://mp.weixin.qq.com/s/b_ZKJhLPep8wgtAETCyxIA)
  狮子大佬写的关于框架优化 Spring Cache 的功能实现。
- [Redisson 的 Spring cache 实现（下）](https://github.com/angryz/my-blog/issues/10)
  讨论中找到的基于 Redisson V2.2.17 版本的分析，虽然版本古老（本文基于 V3.17.5 版本），但是也给了答疑解惑的灵感。
## 提出问题
在缓存带有 TTL 值等相关参数的对象时，Redis 中的对象前面会有一段二进制的信息，例如下图：<br>
![在这里插入图片描述](img08/ad962afd3e1a40bfb97fb037f32541af.png)<br>

注意，此处如果是 Key-Value 对象，不会有前面的这段信息，例如 OSS 配置信息：<br>
![在这里插入图片描述](img08/eea88bcaae2149c889047545a342fa9b.png)<br>
## 问题分析
### 1、为什么只有带有 TTL 等信息的对象值前面会有二进制编码？
这个问题可以在源码中找到答案。<br>

`RedissonCache#put`<br>
![在这里插入图片描述](img08/485ba7a46f1442adaf9727af9c203f6d.png)<br>

缓存对象不同，底层操作是不同的方法。方法 `mapCache.fastPut()` 在底层 Lua 脚本中对数据进行了打包（下面会详细分析脚本）。<br>

`RedissonMapCache#fastPutOperationAsync`<br>
![在这里插入图片描述](img08/4b2f5e1b3f2b48c09033f6c3a2fb1632.png)<br>
### 2、二进制编码的内容是什么？
从上面的脚本可以看到，前面的参数 `maxIdleDelta` 实际上就是设置的 TTL。<br>
![在这里插入图片描述](img08/95aa5a0427cd43b08a97aecfcb843d26.png)<br>

一开始没有得到这个答案时，狮子大佬也去 Redisson 提了 [issue](https://github.com/redisson/redisson/issues/4485)，得到的答复如下：<br>
![在这里插入图片描述](img08/8ca1ca0adcad450bb2db3ceaff8c74c8.png)<br>

下面来探讨一下这个缓存流程。
## 测试执行流程说明
### 1、请求接口
![在这里插入图片描述](img08/5fb1667d12ef42aa8b054abd90183a2b.png)
调用次数：两次

### 2、请求参数
| 名称          | 值                    |
|-------------|----------------------|
| CacheName   | demo:cache#60s#10m#2 |
| 第一次请求 key   | test                 |
| 第一次请求 value | test                 |
| 第二次请求 key   | test                 |
| 第二次请求 value | test1                |

### 3、预想结果

- 第一次请求 Redis 缓存值为 test - test
- 第二次请求 Redis 缓存值更新为 test - test1

**注：缓存结果实际上是返回值对象 R ，这里简化为 R 中的 data 值。**

### 4、请求结果
第一次请求：<br>
![在这里插入图片描述](img08/16ae86422a854306858db5ce0eeee383.png)<br>

第二次请求：<br>
![在这里插入图片描述](img08/709748bf28704236a741b6569ee66db0.png)<br>

![在这里插入图片描述](img08/a5d18a5046344aab97da07c0d70a067c.png)
## 源码分析
源码主要分析 Lua 脚本部分，即前面提及的 put 方法`RedissonMapCache#fastPutOperationAsync`。

因为脚本内容比较多，所以会分成两篇进行分析，本篇是 **关于对象的保存以及更新操作** 。
### 1、流程简图（非常重要）
为了便于理解，我对完整的脚本执行流程画了流程图，这张图很长，主要是三个部分（已经用线区分开）：

- 第一部分：计算过期时间，判断是 **插入** 操作还是 **更新** 操作
- 第二部分：对于 **超出最大限度的缓存值** 的操作
- 第三部分：打包并缓存，发布订阅消息

![在这里插入图片描述](img08/45bf719cb7b847d2a0dfb49e8a41798a.png)

本篇内容主要是第一第三部分的分析，第二部分会在下一篇进行说明。

### 2、Lua 脚本
```lua
local insertable = false; 
local value = redis.call('hget', KEYS[1], ARGV[5]); 
local t, val;
if value == false then 
	insertable = true; 
else 
	t, val = struct.unpack('dLc0', value); 
	local expireDate = 92233720368547758; 
	local expireDateScore = redis.call('zscore', KEYS[2], ARGV[5]); 
	if expireDateScore ~= false then 
		expireDate = tonumber(expireDateScore) 
	end; 
	if t ~= 0 then 
		local expireIdle = redis.call('zscore', KEYS[3], ARGV[5]); 
		if expireIdle ~= false then 
			expireDate = math.min(expireDate, tonumber(expireIdle)) 
		end; 
	end; 
	if expireDate <= tonumber(ARGV[1]) then 
		insertable = true; 
	end; 
end; 

if tonumber(ARGV[2]) > 0 then 
	redis.call('zadd', KEYS[2], ARGV[2], ARGV[5]); 
else 
	redis.call('zrem', KEYS[2], ARGV[5]); 
end; 

if tonumber(ARGV[3]) > 0 then 
	redis.call('zadd', KEYS[3], ARGV[3], ARGV[5]); 
else 
	redis.call('zrem', KEYS[3], ARGV[5]); 
end; 

--// last access time
local maxSize = tonumber(redis.call('hget', KEYS[8], 'max-size')); 
local mode = redis.call('hget', KEYS[8], 'mode'); 
if maxSize ~= nil and maxSize ~= 0 then     
	local currentTime = tonumber(ARGV[1]);     
	local lastAccessTimeSetName = KEYS[6]; 
	
	if mode == false or mode == 'LRU' then 
		redis.call('zadd', lastAccessTimeSetName, currentTime, ARGV[5]); 
	end;     
	
	local cacheSize = tonumber(redis.call('hlen', KEYS[1]));     
	if cacheSize >= maxSize then         
		local lruItems = redis.call('zrange', lastAccessTimeSetName, 0, cacheSize - maxSize);         
		for index, lruItem in ipairs(lruItems) do             
			if lruItem and lruItem ~= ARGV[5] then                 
				local lruItemValue = redis.call('hget', KEYS[1], lruItem);                 
				redis.call('hdel', KEYS[1], lruItem);                 
				redis.call('zrem', KEYS[2], lruItem);                 
				redis.call('zrem', KEYS[3], lruItem);                 
				redis.call('zrem', lastAccessTimeSetName, lruItem);                 
				if lruItemValue ~= false then                 
					local removedChannelName = KEYS[7]; 
					local ttl, obj = struct.unpack('dLc0', lruItemValue);                
					local msg = struct.pack('Lc0Lc0', string.len(lruItem), lruItem, string.len(obj), obj);                
					redis.call('publish', removedChannelName, msg); 
				end;             
			end;         
		end;    
	end; 
	
	if mode == 'LFU' then 
		redis.call('zincrby', lastAccessTimeSetName, 1, ARGV[5]); 
	end; 
end; 

local value = struct.pack('dLc0', ARGV[4], string.len(ARGV[6]), ARGV[6]); 
redis.call('hset', KEYS[1], ARGV[5], value); 
if insertable == true then 
	local msg = struct.pack('Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6]); 
	redis.call('publish', KEYS[4], msg); 
	return 1;
else 
	local msg = struct.pack('Lc0Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6], string.len(val), val); 
	redis.call('publish', KEYS[5], msg); 
	return 0;
end;
```

脚本调用的方法参数对照表：

| 脚本参数名   | Java参数名                        | 参数值（第一次；第二次）                                                                                                                                                                                                                                                                                  | 含义                   |
|---------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------|
| KEYS[1] | name                           | "demo:cache"                                                                                                                                                                                                                                                                                  | CacheName            |
| KEYS[2] | getTimeoutSetName(name)        | "redisson__timeout__set:{demo:cache}"                                                                                                                                                                                                                                                         | TimeoutSet 名称        |
| KEYS[3] | getIdleSetName(name)           | "redisson__idle__set:{demo:cache}"                                                                                                                                                                                                                                                            | IdleSet 名称           |
| KEYS[4] | getCreatedChannelName(name)    | "redisson_map_cache_created:{demo:cache}"                                                                                                                                                                                                                                                     | CreatedChannel 名称    |
| KEYS[5] | getUpdatedChannelName(name)    | "redisson_map_cache_updated:{demo:cache}"                                                                                                                                                                                                                                                     | UpdatedChannel 名称    |
| KEYS[6] | getLastAccessTimeSetName(name) | "redisson__map_cache__last_access__set:{demo:cache}"                                                                                                                                                                                                                                          | LastAccessTimeSet 名称 |
| KEYS[7] | getRemovedChannelName(name)    | "redisson_map_cache_removed:{demo:cache}"                                                                                                                                                                                                                                                     | RemovedChannel 名称    |
| KEYS[8] | getOptionsName(name)           | "{demo:cache}:redisson_options"                                                                                                                                                                                                                                                               | OptionsMap 名称        |
| ARGV[1] | System.currentTimeMillis()     | "1660564385882"；"1660564415922"                                                                                                                                                                                                                                                               | 当前系统时间               |
| ARGV[2] | ttlTimeout                     | "1660564445882"；"1660564475922"                                                                                                                                                                                                                                                               | 过期时间                 |
| ARGV[3] | maxIdleTimeout                 | "1660564985882"；"1660565015922"                                                                                                                                                                                                                                                               | 最大空闲过期时间             |
| ARGV[4] | maxIdleDelta                   | "600000"                                                                                                                                                                                                                                                                                      | 最大空闲增量（用于计算过期时间）     |
| ARGV[5] | encodeMapKey(key)              | "\"test\""                                                                                                                                                                                                                                                                                    | 序列化后的Key             |
| ARGV[6] | encodeMapValue(value)          | "{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}" ；"{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}" | 序列化后的缓存值             |

### 3、Lua 脚本代码分析
说明：
- 分析注释部分只列出相关的部分的说明，没有说明的部分可以暂时忽略。
- `##` 代表注释说明或者是没有实际执行的请求。
- `##(数值)` 代表第几次请求，后面紧跟着的是底层执行的命令。

```lua
local insertable = false; 
local value = redis.call('hget', KEYS[1], ARGV[5]); 
-- "hget" "demo:cache" "\"test\""

local t, val;
-- t：ttl，val：缓存值

-- 判断是否有值
if value == false then 
	insertable = true; 
	-- 没有值：则可插入(insertable)
	
else 
	-- 有值
	t, val = struct.unpack('dLc0', value); 
	-- 获取 ttl 以及 value
	
	local expireDate = 92233720368547758; 
	local expireDateScore = redis.call('zscore', KEYS[2], ARGV[5]); 
	-- 过期时间分值
	-- #2 "zscore" "redisson__timeout__set:{demo:cache}" "\"test\""
	
	-- 判断过期时间分值是否存在
	if expireDateScore ~= false then 
		expireDate = tonumber(expireDateScore) 
		-- 获取过期时间
	end; 
	
	-- 判断 ttl 不为 0
	if t ~= 0 then 
		local expireIdle = redis.call('zscore', KEYS[3], ARGV[5]); 
		-- 过期空闲值
		-- #2 "zscore" "redisson__idle__set:{demo:cache}" "\"test\""
		
		if expireIdle ~= false then 
			expireDate = math.min(expireDate, tonumber(expireIdle)) 
			-- 获取过期时间以及空闲值更小值
		end; 
	end; 
	
	-- 判断过期时间是否小于等于系统时间
	if expireDate <= tonumber(ARGV[1]) then 
		insertable = true; 
		-- 可插入
	end; 
end; 

-- 判断  ttlTimeout 是否大于 0
if tonumber(ARGV[2]) > 0 then 
	redis.call('zadd', KEYS[2], ARGV[2], ARGV[5]); 
	-- 大于 0：加入 ttlTimeout set（redisson__timeout__set）
	-- #1 "zadd" "redisson__timeout__set:{demo:cache}" "1660564445882" "\"test\""
	-- #2 "zadd" "redisson__timeout__set:{demo:cache}" "1660564475922" "\"test\""
	
else 
	redis.call('zrem', KEYS[2], ARGV[5]); 
	-- 小于等于 0：从 set 移除该值
	-- ## "zrem" "redisson__timeout__set:{demo:cache}" "\"test\""
	
end; 

-- 判断 maxIdleTimeout 是否大于 0
if tonumber(ARGV[3]) > 0 then 
	redis.call('zadd', KEYS[3], ARGV[3], ARGV[5]); 
	-- 大于 0：加入 idle set（redisson__idle__set）
	-- #1 "zadd" "redisson__idle__set:{demo:cache}" "1660564985882" "\"test\""
	-- #2 "zadd" "redisson__idle__set:{demo:cache}" "1660565015922" "\"test\""
	
else 
	redis.call('zrem', KEYS[3], ARGV[5]); 
	-- 小于等于 0：从 set 移除该值
	-- ## "zrem" "redisson__idle__set:{demo:cache}" "\"test\""
	
end; 

local maxSize = tonumber(redis.call('hget', KEYS[8], 'max-size')); 
local mode = redis.call('hget', KEYS[8], 'mode'); 

if maxSize ~= nil and maxSize ~= 0 then     
	local currentTime = tonumber(ARGV[1]);     
	local lastAccessTimeSetName = KEYS[6]; 
	
	if mode == false or mode == 'LRU' then 
		redis.call('zadd', lastAccessTimeSetName, currentTime, ARGV[5]); 
	end;     
	
	local cacheSize = tonumber(redis.call('hlen', KEYS[1]));   
	
	if cacheSize >= maxSize then         
		local lruItems = redis.call('zrange', lastAccessTimeSetName, 0, cacheSize - maxSize);         
		
		for index, lruItem in ipairs(lruItems) do          
			if lruItem and lruItem ~= ARGV[5] then        
			
				local lruItemValue = redis.call('hget', KEYS[1], lruItem);                 
				redis.call('hdel', KEYS[1], lruItem);      
				redis.call('zrem', KEYS[2], lruItem);     {demo:cache}" "\"test\""
				redis.call('zrem', KEYS[3], lruItem);        
				redis.call('zrem', lastAccessTimeSetName, lruItem);           
				if lruItemValue ~= false then                 
					local removedChannelName = KEYS[7]; 
					local ttl, obj = struct.unpack('dLc0', lruItemValue);            
					local msg = struct.pack('Lc0Lc0', string.len(lruItem), lruItem, string.len(obj), obj);     
					redis.call('publish', removedChannelName, msg); 
				end;             
			end;         
		end;    
	end; 
	
	if mode == 'LFU' then 
		redis.call('zincrby', lastAccessTimeSetName, 1, ARGV[5]); 
	end; 
end; 

local value = struct.pack('dLc0', ARGV[4], string.len(ARGV[6]), ARGV[6]); 
-- 打包 缓存值（ttl，value）
-- "\x00\x00\x00\x00\x00L\xed@Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"

redis.call('hset', KEYS[1], ARGV[5], value); 
-- 插入缓存  
-- #1 "hset" "demo:cache" "\"test\"" "\x00\x00\x00\x00\x80O\"AY\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
-- #2 "hset" "demo:cache" "\"test\"" "\x00\x00\x00\x00\x80O\"AZ\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"

if insertable == true then 
	-- 插入缓存
	
	local msg = struct.pack('Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6]); 
	redis.call('publish', KEYS[4], msg); 
	-- #1 "publish" "redisson_map_cache_created:{demo:cache}" "\x06\x00\x00\x00\"test\"Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
	
	return 1;
else 
	-- 更新缓存
	
	local msg = struct.pack('Lc0Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6], string.len(val), val); 
	redis.call('publish', KEYS[5], msg); 
	-- #2 "publish" "redisson_map_cache_updated:{demo:cache}" "\x06\x00\x00\x00\"test\"Z\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
	
	return 0;
end;
```
### 4、Redis 实际执行结果
```bash
### 第一次请求：

"demo:cache"  KEYS[1] name
"redisson__timeout__set:{demo:cache}"  KEYS[2] getTimeoutSetName(name)
"redisson__idle__set:{demo:cache}"  KEYS[3] getIdleSetName(name)
"redisson_map_cache_created:{demo:cache}"  KEYS[4] getCreatedChannelName(name)
"redisson_map_cache_updated:{demo:cache}"  KEYS[5] getUpdatedChannelName(name)
"redisson__map_cache__last_access__set:{demo:cache}" KEYS[6] getLastAccessTimeSetName(name)
"redisson_map_cache_removed:{demo:cache}" KEYS[7] getRemovedChannelName(name)
"{demo:cache}:redisson_options" KEYS[8] getOptionsName(name)
"1660564385882" ARGV[1] System.currentTimeMillis()
"1660564445882" ARGV[2] ttlTimeout
"1660564985882" ARGV[3] maxIdleTimeout
"600000" ARGV[4] maxIdleDelta
"\"test\"" ARGV[5] encodeMapKey(key)
"{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}" ARGV[6] encodeMapValue(value)

1660564385.887959 [5 lua] "hget" "demo:cache" "\"test\""
1660564385.888021 [5 lua] "zadd" "redisson__timeout__set:{demo:cache}" "1660564445882" "\"test\""
1660564385.888065 [5 lua] "zadd" "redisson__idle__set:{demo:cache}" "1660564985882" "\"test\""
1660564385.888076 [5 lua] "hget" "{demo:cache}:redisson_options" "max-size"
1660564385.888082 [5 lua] "hget" "{demo:cache}:redisson_options" "mode"
1660564385.888097 [5 lua] "hset" "demo:cache" "\"test\"" "\x00\x00\x00\x00\x80O\"AY\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660564385.888146 [5 lua] "publish" "redisson_map_cache_created:{demo:cache}" "\x06\x00\x00\x00\"test\"Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"

-------------------------------------

### 第二次请求：

"demo:cache"  KEYS[1] name
"redisson__timeout__set:{demo:cache}"  KEYS[2] getTimeoutSetName(name)
"redisson__idle__set:{demo:cache}"  KEYS[3] getIdleSetName(name)
"redisson_map_cache_created:{demo:cache}"  KEYS[4] getCreatedChannelName(name)
"redisson_map_cache_updated:{demo:cache}"  KEYS[5] getUpdatedChannelName(name)
"redisson__map_cache__last_access__set:{demo:cache}" KEYS[6] getLastAccessTimeSetName(name)
"redisson_map_cache_removed:{demo:cache}" KEYS[7] getRemovedChannelName(name)
"{demo:cache}:redisson_options" KEYS[8] getOptionsName(name)
"1660564415922" ARGV[1] System.currentTimeMillis()
"1660564475922" ARGV[2] ttlTimeout
"1660565015922" ARGV[3] maxIdleTimeout
"600000" ARGV[4] maxIdleDelta
"\"test\"" ARGV[5] encodeMapKey(key)
"{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}" ARGV[6] encodeMapValue(value)

1660564415.923910 [5 lua] "hget" "demo:cache" "\"test\""
1660564415.923919 [5 lua] "zscore" "redisson__timeout__set:{demo:cache}" "\"test\""
1660564415.923932 [5 lua] "zscore" "redisson__idle__set:{demo:cache}" "\"test\""
1660564415.923943 [5 lua] "zadd" "redisson__timeout__set:{demo:cache}" "1660564475922" "\"test\""
1660564415.923956 [5 lua] "zadd" "redisson__idle__set:{demo:cache}" "1660565015922" "\"test\""
1660564415.923965 [5 lua] "hget" "{demo:cache}:redisson_options" "max-size"
1660564415.923971 [5 lua] "hget" "{demo:cache}:redisson_options" "mode"
1660564415.923980 [5 lua] "hset" "demo:cache" "\"test\"" "\x00\x00\x00\x00\x80O\"AZ\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660564415.924113 [5 lua] "publish" "redisson_map_cache_updated:{demo:cache}" "\x06\x00\x00\x00\"test\"Z\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
```

## 附文
### Redis monitor 模式下控制台输出完整内容
```bash
127.0.0.1:6379> monitor
OK
1660564385.875990 [5 127.0.0.1:1397] "GET" "Authorization:login:token:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY"
1660564385.877948 [5 127.0.0.1:1398] "GET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY"
1660564385.879625 [5 127.0.0.1:1399] "PTTL" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY"
1660564385.880673 [5 127.0.0.1:1392] "SET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY" "\"1660564385878\""
1660564385.880832 [5 127.0.0.1:1392] "EVAL" "local result = 0;for j = 1, #KEYS, 1 do local expireSet; if ARGV[2] ~= '' then expireSet = redis.call('pexpire', KEYS[j], ARGV[1], ARGV[2]); else expireSet = redis.call('pexpire', KEYS[j], ARGV[1]); end; if expireSet == 1 then result = expireSet;end; end; return result; " "1" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY" "84581000" ""
1660564385.880967 [5 lua] "pexpire" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY" "84581000"
1660564385.887505 [5 127.0.0.1:1391] "EVAL" "local insertable = false; local value = redis.call('hget', KEYS[1], ARGV[5]); local t, val;if value == false then insertable = true; else t, val = struct.unpack('dLc0', value); local expireDate = 92233720368547758; local expireDateScore = redis.call('zscore', KEYS[2], ARGV[5]); if expireDateScore ~= false then expireDate = tonumber(expireDateScore) end; if t ~= 0 then local expireIdle = redis.call('zscore', KEYS[3], ARGV[5]); if expireIdle ~= false then expireDate = math.min(expireDate, tonumber(expireIdle)) end; end; if expireDate <= tonumber(ARGV[1]) then insertable = true; end; end; if tonumber(ARGV[2]) > 0 then redis.call('zadd', KEYS[2], ARGV[2], ARGV[5]); else redis.call('zrem', KEYS[2], ARGV[5]); end; if tonumber(ARGV[3]) > 0 then redis.call('zadd', KEYS[3], ARGV[3], ARGV[5]); else redis.call('zrem', KEYS[3], ARGV[5]); end; local maxSize = tonumber(redis.call('hget', KEYS[8], 'max-size')); local mode = redis.call('hget', KEYS[8], 'mode'); if maxSize ~= nil and maxSize ~= 0 then     local currentTime = tonumber(ARGV[1]);     local lastAccessTimeSetName = KEYS[6]; if mode == false or mode == 'LRU' then redis.call('zadd', lastAccessTimeSetName, currentTime, ARGV[5]); end;     local cacheSize = tonumber(redis.call('hlen', KEYS[1]));     if cacheSize >= maxSize then         local lruItems = redis.call('zrange', lastAccessTimeSetName, 0, cacheSize - maxSize);         for index, lruItem in ipairs(lruItems) do             if lruItem and lruItem ~= ARGV[5] then                 local lruItemValue = redis.call('hget', KEYS[1], lruItem);                 redis.call('hdel', KEYS[1], lruItem);                 redis.call('zrem', KEYS[2], lruItem);                 redis.call('zrem', KEYS[3], lruItem);                 redis.call('zrem', lastAccessTimeSetName, lruItem);                 if lruItemValue ~= false then                 local removedChannelName = KEYS[7]; local ttl, obj = struct.unpack('dLc0', lruItemValue);                local msg = struct.pack('Lc0Lc0', string.len(lruItem), lruItem, string.len(obj), obj);                redis.call('publish', removedChannelName, msg); end;             end;         end;     end; if mode == 'LFU' then redis.call('zincrby', lastAccessTimeSetName, 1, ARGV[5]); end; end; local value = struct.pack('dLc0', ARGV[4], string.len(ARGV[6]), ARGV[6]); redis.call('hset', KEYS[1], ARGV[5], value); if insertable == true then local msg = struct.pack('Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6]); redis.call('publish', KEYS[4], msg); return 1;else local msg = struct.pack('Lc0Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6], string.len(val), val); redis.call('publish', KEYS[5], msg); return 0;end;" "8" "demo:cache" "redisson__timeout__set:{demo:cache}" "redisson__idle__set:{demo:cache}" "redisson_map_cache_created:{demo:cache}" "redisson_map_cache_updated:{demo:cache}" "redisson__map_cache__last_access__set:{demo:cache}" "redisson_map_cache_removed:{demo:cache}" "{demo:cache}:redisson_options" "1660564385882" "1660564445882" "1660564985882" "600000" "\"test\"" "{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660564385.887959 [5 lua] "hget" "demo:cache" "\"test\""
1660564385.888021 [5 lua] "zadd" "redisson__timeout__set:{demo:cache}" "1660564445882" "\"test\""
1660564385.888065 [5 lua] "zadd" "redisson__idle__set:{demo:cache}" "1660564985882" "\"test\""
1660564385.888076 [5 lua] "hget" "{demo:cache}:redisson_options" "max-size"
1660564385.888082 [5 lua] "hget" "{demo:cache}:redisson_options" "mode"
1660564385.888097 [5 lua] "hset" "demo:cache" "\"test\"" "\x00\x00\x00\x00\x80O\"AY\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660564385.888146 [5 lua] "publish" "redisson_map_cache_created:{demo:cache}" "\x06\x00\x00\x00\"test\"Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660564390.215476 [5 127.0.0.1:1060] "ping"
1660564393.919090 [5 127.0.0.1:1060] "scan" "0" "MATCH" "*" "COUNT" "500"
1660564393.920848 [5 127.0.0.1:1060] "info"
1660564395.507559 [5 127.0.0.1:1060] "type" "demo:cache"
1660564395.518355 [5 127.0.0.1:1060] "ttl" "demo:cache"
1660564395.520778 [5 127.0.0.1:1060] "hlen" "demo:cache"
1660564395.530076 [5 127.0.0.1:1060] "hscan" "demo:cache" "0" "MATCH" "*" "COUNT" "200"
1660564399.210430 [5 127.0.0.1:1060] "ping"
1660564407.741175 [5 127.0.0.1:1391] "PING"
1660564407.741249 [5 127.0.0.1:1393] "PING"
1660564407.741257 [5 127.0.0.1:1395] "PING"
1660564407.741405 [5 127.0.0.1:1394] "PING"
1660564407.741460 [5 127.0.0.1:1392] "PING"
1660564407.741495 [5 127.0.0.1:1396] "PING"
1660564407.741512 [5 127.0.0.1:1397] "PING"
1660564407.742088 [5 127.0.0.1:1398] "PING"
1660564407.742404 [5 127.0.0.1:1399] "PING"
1660564410.211844 [5 127.0.0.1:1060] "ping"
1660564415.916463 [5 127.0.0.1:1394] "GET" "Authorization:login:token:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY"
1660564415.917887 [5 127.0.0.1:1395] "GET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY"
1660564415.920153 [5 127.0.0.1:1396] "PTTL" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY"
1660564415.921689 [5 127.0.0.1:1397] "SET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY" "\"1660564415918\""
1660564415.921738 [5 127.0.0.1:1397] "EVAL" "local result = 0;for j = 1, #KEYS, 1 do local expireSet; if ARGV[2] ~= '' then expireSet = redis.call('pexpire', KEYS[j], ARGV[1], ARGV[2]); else expireSet = redis.call('pexpire', KEYS[j], ARGV[1]); end; if expireSet == 1 then result = expireSet;end; end; return result; " "1" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY" "84550000" ""
1660564415.921819 [5 lua] "pexpire" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJVRzNpaFBZc1hiUGx3NzdjM2hUZFd5UGNHYkJYcmQ1TyJ9.X5b9I9ZNIucfwyEMWSvB99oabna9nvMM7QK6xBBcPtY" "84550000"
1660564415.923539 [5 127.0.0.1:1398] "EVAL" "local insertable = false; local value = redis.call('hget', KEYS[1], ARGV[5]); local t, val;if value == false then insertable = true; else t, val = struct.unpack('dLc0', value); local expireDate = 92233720368547758; local expireDateScore = redis.call('zscore', KEYS[2], ARGV[5]); if expireDateScore ~= false then expireDate = tonumber(expireDateScore) end; if t ~= 0 then local expireIdle = redis.call('zscore', KEYS[3], ARGV[5]); if expireIdle ~= false then expireDate = math.min(expireDate, tonumber(expireIdle)) end; end; if expireDate <= tonumber(ARGV[1]) then insertable = true; end; end; if tonumber(ARGV[2]) > 0 then redis.call('zadd', KEYS[2], ARGV[2], ARGV[5]); else redis.call('zrem', KEYS[2], ARGV[5]); end; if tonumber(ARGV[3]) > 0 then redis.call('zadd', KEYS[3], ARGV[3], ARGV[5]); else redis.call('zrem', KEYS[3], ARGV[5]); end; local maxSize = tonumber(redis.call('hget', KEYS[8], 'max-size')); local mode = redis.call('hget', KEYS[8], 'mode'); if maxSize ~= nil and maxSize ~= 0 then     local currentTime = tonumber(ARGV[1]);     local lastAccessTimeSetName = KEYS[6]; if mode == false or mode == 'LRU' then redis.call('zadd', lastAccessTimeSetName, currentTime, ARGV[5]); end;     local cacheSize = tonumber(redis.call('hlen', KEYS[1]));     if cacheSize >= maxSize then         local lruItems = redis.call('zrange', lastAccessTimeSetName, 0, cacheSize - maxSize);         for index, lruItem in ipairs(lruItems) do             if lruItem and lruItem ~= ARGV[5] then                 local lruItemValue = redis.call('hget', KEYS[1], lruItem);                 redis.call('hdel', KEYS[1], lruItem);                 redis.call('zrem', KEYS[2], lruItem);                 redis.call('zrem', KEYS[3], lruItem);                 redis.call('zrem', lastAccessTimeSetName, lruItem);                 if lruItemValue ~= false then                 local removedChannelName = KEYS[7]; local ttl, obj = struct.unpack('dLc0', lruItemValue);                local msg = struct.pack('Lc0Lc0', string.len(lruItem), lruItem, string.len(obj), obj);                redis.call('publish', removedChannelName, msg); end;             end;         end;     end; if mode == 'LFU' then redis.call('zincrby', lastAccessTimeSetName, 1, ARGV[5]); end; end; local value = struct.pack('dLc0', ARGV[4], string.len(ARGV[6]), ARGV[6]); redis.call('hset', KEYS[1], ARGV[5], value); if insertable == true then local msg = struct.pack('Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6]); redis.call('publish', KEYS[4], msg); return 1;else local msg = struct.pack('Lc0Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6], string.len(val), val); redis.call('publish', KEYS[5], msg); return 0;end;" "8" "demo:cache" "redisson__timeout__set:{demo:cache}" "redisson__idle__set:{demo:cache}" "redisson_map_cache_created:{demo:cache}" "redisson_map_cache_updated:{demo:cache}" "redisson__map_cache__last_access__set:{demo:cache}" "redisson_map_cache_removed:{demo:cache}" "{demo:cache}:redisson_options" "1660564415922" "1660564475922" "1660565015922" "600000" "\"test\"" "{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660564415.923910 [5 lua] "hget" "demo:cache" "\"test\""
1660564415.923919 [5 lua] "zscore" "redisson__timeout__set:{demo:cache}" "\"test\""
1660564415.923932 [5 lua] "zscore" "redisson__idle__set:{demo:cache}" "\"test\""
1660564415.923943 [5 lua] "zadd" "redisson__timeout__set:{demo:cache}" "1660564475922" "\"test\""
1660564415.923956 [5 lua] "zadd" "redisson__idle__set:{demo:cache}" "1660565015922" "\"test\""
1660564415.923965 [5 lua] "hget" "{demo:cache}:redisson_options" "max-size"
1660564415.923971 [5 lua] "hget" "{demo:cache}:redisson_options" "mode"
1660564415.923980 [5 lua] "hset" "demo:cache" "\"test\"" "\x00\x00\x00\x00\x80O\"AZ\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660564415.924113 [5 lua] "publish" "redisson_map_cache_updated:{demo:cache}" "\x06\x00\x00\x00\"test\"Z\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
```