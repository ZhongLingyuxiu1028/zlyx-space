# Redisson（九）RedissonMapCache 缓存流程分析（下）
- - -
## 前言
承接上篇，本篇来探讨一下 **当超过了缓存 maxSize 时，底层脚本是如何操作的**，事不宜迟，马上开始。

## 测试执行流程说明
### 1、请求接口
![在这里插入图片描述](img09/5fb1667d12ef42aa8b054abd90183a2b.png)

调用次数：三次

### 2、请求参数
| 名称          | 值                    |
|-------------|----------------------|
| CacheName   | demo:cache#60s#10m#2 |
| 第一次请求 key   | test                 |
| 第一次请求 value | test                 |
| 第二次请求 key   | test1                |
| 第二次请求 value | test1                |
| 第三次请求 key   | test2                |
| 第三次请求 value | test2                |

### 3、预想结果

- 第一次请求 Redis 缓存值为 [ test - test ]
- 第二次请求 Redis 缓存值更新为 [ test - test，test1 - test1 ]
- 第三次请求 Redis 缓存值更新为 [ test1 - test1，test2 - test2 ]，淘汰了元素 [ test - test ]

**注：缓存结果实际上是返回值对象 R ，这里简化为 R 中的 data 值。**
### 4、请求结果
![在这里插入图片描述](img09/7213ccdc71614495b6f96dbc4faf22fb.png)
## 源码分析
### 1、流程简图（非常重要）
为了便于理解，我对完整的脚本执行流程画了流程图，这张图很长，主要是三个部分（已经用线区分开）：

- 第一部分：计算过期时间，判断是 **插入** 操作还是 **更新** 操作
- 第二部分：对于 **超出最大限度的缓存值** 的操作
- 第三部分：打包并缓存，发布订阅消息

![在这里插入图片描述](img09/45bf719cb7b847d2a0dfb49e8a41798a.png)
  
第一、第三部分在上一篇已经分析过，本篇重点说明第二部分。
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

| 脚本参数名   | Java参数名                        | 参数值（第一次；第二次；第三次）                                                                                                                                                                                                                                                                                                                                                                                                                                           | 含义                   |
|---------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------|
| KEYS[1] | name                           | "demo:cache"                                                                                                                                                                                                                                                                                                                                                                                                                                               | CacheName            |
| KEYS[2] | getTimeoutSetName(name)        | "redisson__timeout__set:{demo:cache}"                                                                                                                                                                                                                                                                                                                                                                                                                      | TimeoutSet 名称        |
| KEYS[3] | getIdleSetName(name)           | "redisson__idle__set:{demo:cache}"                                                                                                                                                                                                                                                                                                                                                                                                                         | IdleSet 名称           |
| KEYS[4] | getCreatedChannelName(name)    | "redisson_map_cache_created:{demo:cache}"                                                                                                                                                                                                                                                                                                                                                                                                                  | CreatedChannel 名称    |
| KEYS[5] | getUpdatedChannelName(name)    | "redisson_map_cache_updated:{demo:cache}"                                                                                                                                                                                                                                                                                                                                                                                                                  | UpdatedChannel 名称    |
| KEYS[6] | getLastAccessTimeSetName(name) | "redisson__map_cache__last_access__set:{demo:cache}"                                                                                                                                                                                                                                                                                                                                                                                                       | LastAccessTimeSet 名称 |
| KEYS[7] | getRemovedChannelName(name)    | "redisson_map_cache_removed:{demo:cache}"                                                                                                                                                                                                                                                                                                                                                                                                                  | RemovedChannel 名称    |
| KEYS[8] | getOptionsName(name)           | "{demo:cache}:redisson_options"                                                                                                                                                                                                                                                                                                                                                                                                                            | OptionsMap 名称        |
| ARGV[1] | System.currentTimeMillis()     | "1660568866705"；"1660568875018"；"1660568889036"                                                                                                                                                                                                                                                                                                                                                                                                            | 当前系统时间               |
| ARGV[2] | ttlTimeout                     | "1660568926705"；"1660568935018"；"1660568949036"                                                                                                                                                                                                                                                                                                                                                                                                            | 过期时间                 |
| ARGV[3] | maxIdleTimeout                 | "1660569466705"；"1660569475018"；"1660569489036"                                                                                                                                                                                                                                                                                                                                                                                                            | 最大空闲过期时间             |
| ARGV[4] | maxIdleDelta                   | "600000"                                                                                                                                                                                                                                                                                                                                                                                                                                                   | 最大空闲增量（用于计算过期时间）     |
| ARGV[5] | encodeMapKey(key)              | "\"test\""；"\"test1\""；"\"test2\""                                                                                                                                                                                                                                                                                                                                                                                                                         | 序列化后的Key             |
| ARGV[6] | encodeMapValue(value)          | "{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":==\"test\"==,\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}" ；"{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":==\"test1\"==,\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}" ； "{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":==\"test2\"==,\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}" | 序列化后的缓存值             |

### 3、Lua 脚本代码分析
说明：
- `##` 代表注释说明或者是没有实际执行的请求。
- `##(数值)` 代表第几次请求，后面紧跟着的是底层执行的命令。

```lua
local insertable = false; 
local value = redis.call('hget', KEYS[1], ARGV[5]); 
-- #1 "hget" "demo:cache" "\"test\""
-- #2 "hget" "demo:cache" "\"test1\""
-- #3 "hget" "demo:cache" "\"test2\""

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
	
	-- 判断过期时间分值是否存在
	if expireDateScore ~= false then 
		expireDate = tonumber(expireDateScore) 
		-- 获取过期时间
	end; 
	
	-- 判断 ttl 不为 0
	if t ~= 0 then 
		local expireIdle = redis.call('zscore', KEYS[3], ARGV[5]); 
		-- 过期空闲值
		
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
	-- #1 "zadd" "redisson__timeout__set:{demo:cache}" "1660568926705" "\"test\""
	-- #2 "zadd" "redisson__timeout__set:{demo:cache}" "1660568935018" "\"test1\""
	-- #3 "zadd" "redisson__timeout__set:{demo:cache}" "1660568949036" "\"test2\""
	
else 
	redis.call('zrem', KEYS[2], ARGV[5]); 
	-- 小于等于 0：从 set 移除该值
	-- ## "zrem" "redisson__timeout__set:{demo:cache}" "\"test\""
	
end; 

-- 判断 maxIdleTimeout 是否大于 0
if tonumber(ARGV[3]) > 0 then 
	redis.call('zadd', KEYS[3], ARGV[3], ARGV[5]); 
	-- 大于 0：加入 idle set（redisson__idle__set）
	-- #1 "zadd" "redisson__idle__set:{demo:cache}" "1660569466705" "\"test\""
	-- #2 "zadd" "redisson__idle__set:{demo:cache}" "1660569475018" "\"test1\""
	-- #3 "zadd" "redisson__idle__set:{demo:cache}" "1660569489036" "\"test2\""
	
else 
	redis.call('zrem', KEYS[3], ARGV[5]); 
	-- 小于等于 0：从 set 移除该值
	-- ## "zrem" "redisson__idle__set:{demo:cache}" "\"test\""
	
end; 

--// last access time
local maxSize = tonumber(redis.call('hget', KEYS[8], 'max-size')); 
-- 缓存最大长度
-- "hget" "{demo:cache}:redisson_options" "max-size"

local mode = redis.call('hget', KEYS[8], 'mode'); 
-- "hget" "{demo:cache}:redisson_options" "mode"

if maxSize ~= nil and maxSize ~= 0 then     
	local currentTime = tonumber(ARGV[1]);     
	-- 当前时间 
	local lastAccessTimeSetName = KEYS[6]; 
	-- 上次访问时间 set（redisson__map_cache__last_access__set）
	
	if mode == false or mode == 'LRU' then 
		-- LRU 最近最少使用
		
		redis.call('zadd', lastAccessTimeSetName, currentTime, ARGV[5]); 
		-- #3 "zadd" "redisson__map_cache__last_access__set:{demo:cache}" "1660568889036" "\"test2\""
		
	end;     
	
	local cacheSize = tonumber(redis.call('hlen', KEYS[1]));     
	-- 缓存值 size 
	-- #3 "hlen" "demo:cache"
	
	-- 判断是否大于等于最大长度
	if cacheSize >= maxSize then         
		local lruItems = redis.call('zrange', lastAccessTimeSetName, 0, cacheSize - maxSize);         
		-- 对 lastAccessTimeSetName 集合进行排序，获取指定元素
		-- #3 "zrange" "redisson__map_cache__last_access__set:{demo:cache}" "0" "0"
		-- 此处获得 "redisson__map_cache__last_access__set:{demo:cache}" 第一个元素 "\"test\""
		
		for index, lruItem in ipairs(lruItems) do             
		-- 遍历  lruItems 集合，index：下标，lruItem：元素
		
			if lruItem and lruItem ~= ARGV[5] then                 
			-- 如果  lruItem 存在且不等于  "\"test2\"" 
			
				local lruItemValue = redis.call('hget', KEYS[1], lruItem);                 
				-- 获取  "demo:cache" 当前值 lruItem
				-- #3 "hget" "demo:cache" "\"test\""
				
				redis.call('hdel', KEYS[1], lruItem);                 
				-- 删除  "demo:cache" 当前值 lruItem
				-- #3 "hdel" "demo:cache" "\"test\""
				redis.call('zrem', KEYS[2], lruItem);                 
				-- 删除  "redisson__timeout__set:{demo:cache}" 当前值 lruItem
				-- #3 "zrem" "redisson__timeout__set:{demo:cache}" "\"test\""
				redis.call('zrem', KEYS[3], lruItem);           
				-- 删除  "redisson__idle__set:{demo:cache}" 当前值 lruItem
				-- #3 "zrem" "redisson__idle__set:{demo:cache}" "\"test\""
				redis.call('zrem', lastAccessTimeSetName, lruItem);           
				-- 删除  "redisson__map_cache__last_access__set:{demo:cache}" 当前值 lruItem
				-- #3 "zrem" "redisson__map_cache__last_access__set:{demo:cache}" "\"test\""
				
				if lruItemValue ~= false then                 
					local removedChannelName = KEYS[7]; 
					-- "redisson_map_cache_removed:{demo:cache}"
					local ttl, obj = struct.unpack('dLc0', lruItemValue);            
					-- 获取 lruItemValue 过期时间以及缓存对象值
					local msg = struct.pack('Lc0Lc0', string.len(lruItem), lruItem, string.len(obj), obj);                
					-- 打包过期消息
					redis.call('publish', removedChannelName, msg); 
					-- 发布 remove 消息
					-- #3 "publish" "redisson_map_cache_removed:{demo:cache}" "\x06\x00\x00\x00\"test\"Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
				end;             
			end;         
		end;    
	end; 
	
	if mode == 'LFU' then 
		-- LFU 最不经常使用
	
		redis.call('zincrby', lastAccessTimeSetName, 1, ARGV[5]); 
		-- 分数值 +1
		-- ## "zincrby"  "redisson__map_cache__last_access__set:{demo:cache}" 1 "\"test\""
		
	end; 
end; 

local value = struct.pack('dLc0', ARGV[4], string.len(ARGV[6]), ARGV[6]); 
-- 打包 缓存值（ttl，value）

redis.call('hset', KEYS[1], ARGV[5], value); 
-- 插入缓存  
-- #1 "hset" "demo:cache" "\"test\"" "\x00\x00\x00\x00\x80O\"AY\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
-- #2 "hset" "demo:cache" "\"test1\"" "\x00\x00\x00\x00\x80O\"AZ\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
-- #3 "hset" "demo:cache" "\"test2\"" "\x00\x00\x00\x00\x80O\"AZ\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test2\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"

if insertable == true then 
	-- 插入缓存
	
	local msg = struct.pack('Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6]); 
	redis.call('publish', KEYS[4], msg); 
	-- #1 "publish" "redisson_map_cache_created:{demo:cache}" "\a\x00\x00\x00\"test1\"Z\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
	-- #2 "publish" "redisson_map_cache_created:{demo:cache}" "\a\x00\x00\x00\"test2\"Z\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test2\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
	
	return 1;
else 
	-- 更新缓存
	
	local msg = struct.pack('Lc0Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6], string.len(val), val); 
	redis.call('publish', KEYS[5], msg); 
	
	return 0;
end;
```

### 4、Redis 实际执行结果
```bash
### 第一次请求：

1660568866.709243 [5 lua] "hget" "demo:cache" "\"test\""
1660568866.709255 [5 lua] "zadd" "redisson__timeout__set:{demo:cache}" "1660568926705" "\"test\""
1660568866.709272 [5 lua] "zadd" "redisson__idle__set:{demo:cache}" "1660569466705" "\"test\""
1660568866.709281 [5 lua] "hget" "{demo:cache}:redisson_options" "max-size"
1660568866.709290 [5 lua] "hget" "{demo:cache}:redisson_options" "mode"
1660568866.709300 [5 lua] "zadd" "redisson__map_cache__last_access__set:{demo:cache}" "1660568866705" "\"test\""
1660568866.709311 [5 lua] "hlen" "demo:cache"
1660568866.709318 [5 lua] "hset" "demo:cache" "\"test\"" "\x00\x00\x00\x00\x80O\"AY\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568866.709339 [5 lua] "publish" "redisson_map_cache_created:{demo:cache}" "\x06\x00\x00\x00\"test\"Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"

-------------------------------------

### 第二次请求：

1660568875.019854 [5 lua] "hget" "demo:cache" "\"test1\""
1660568875.019864 [5 lua] "zadd" "redisson__timeout__set:{demo:cache}" "1660568935018" "\"test1\""
1660568875.019906 [5 lua] "zadd" "redisson__idle__set:{demo:cache}" "1660569475018" "\"test1\""
1660568875.019917 [5 lua] "hget" "{demo:cache}:redisson_options" "max-size"
1660568875.019924 [5 lua] "hget" "{demo:cache}:redisson_options" "mode"
1660568875.019933 [5 lua] "zadd" "redisson__map_cache__last_access__set:{demo:cache}" "1660568875018" "\"test1\""
1660568875.019944 [5 lua] "hlen" "demo:cache"
1660568875.019950 [5 lua] "hset" "demo:cache" "\"test1\"" "\x00\x00\x00\x00\x80O\"AZ\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568875.019971 [5 lua] "publish" "redisson_map_cache_created:{demo:cache}" "\a\x00\x00\x00\"test1\"Z\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"

-------------------------------------

### 第三次请求：

"demo:cache" 
"redisson__timeout__set:{demo:cache}" 
"redisson__idle__set:{demo:cache}" 
"redisson_map_cache_created:{demo:cache}" 
"redisson_map_cache_updated:{demo:cache}" 
"redisson__map_cache__last_access__set:{demo:cache}" 
"redisson_map_cache_removed:{demo:cache}" 
"{demo:cache}:redisson_options" 
"1660568889036" 
"1660568949036" 
"1660569489036" 
"600000" 
"\"test2\"" 
"{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test2\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}" 

1660568889.038959 [5 lua] "hget" "demo:cache" "\"test2\""
1660568889.038993 [5 lua] "zadd" "redisson__timeout__set:{demo:cache}" "1660568949036" "\"test2\""
1660568889.039044 [5 lua] "zadd" "redisson__idle__set:{demo:cache}" "1660569489036" "\"test2\""
1660568889.039085 [5 lua] "hget" "{demo:cache}:redisson_options" "max-size"
1660568889.039093 [5 lua] "hget" "{demo:cache}:redisson_options" "mode"
1660568889.039104 [5 lua] "zadd" "redisson__map_cache__last_access__set:{demo:cache}" "1660568889036" "\"test2\""
1660568889.039116 [5 lua] "hlen" "demo:cache"
1660568889.039120 [5 lua] "zrange" "redisson__map_cache__last_access__set:{demo:cache}" "0" "0"
1660568889.039134 [5 lua] "hget" "demo:cache" "\"test\""
1660568889.039139 [5 lua] "hdel" "demo:cache" "\"test\""
1660568889.039144 [5 lua] "zrem" "redisson__timeout__set:{demo:cache}" "\"test\""
1660568889.039151 [5 lua] "zrem" "redisson__idle__set:{demo:cache}" "\"test\""
1660568889.039158 [5 lua] "zrem" "redisson__map_cache__last_access__set:{demo:cache}" "\"test\""
1660568889.039185 [5 lua] "publish" "redisson_map_cache_removed:{demo:cache}" "\x06\x00\x00\x00\"test\"Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568889.039229 [5 lua] "hset" "demo:cache" "\"test2\"" "\x00\x00\x00\x00\x80O\"AZ\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test2\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568889.039311 [5 lua] "publish" "redisson_map_cache_created:{demo:cache}" "\a\x00\x00\x00\"test2\"Z\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test2\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
```

## 附文
### Redis monitor 模式下控制台输出完整内容
```bash
127.0.0.1:6379> monitor
OK
1660568866.655633 [5 127.0.0.1:3687] "GET" "Authorization:login:token:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw"
1660568866.656867 [5 127.0.0.1:3688] "GET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw"
1660568866.657658 [5 127.0.0.1:3689] "PTTL" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw"
1660568866.663266 [5 127.0.0.1:3690] "SET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw" "\"1660568866657\""
1660568866.663322 [5 127.0.0.1:3690] "EVAL" "local result = 0;for j = 1, #KEYS, 1 do local expireSet; if ARGV[2] ~= '' then expireSet = redis.call('pexpire', KEYS[j], ARGV[1], ARGV[2]); else expireSet = redis.call('pexpire', KEYS[j], ARGV[1]); end; if expireSet == 1 then result = expireSet;end; end; return result; " "1" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw" "85880000" ""
1660568866.663396 [5 lua] "pexpire" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw" "85880000"
1660568866.694562 [5 127.0.0.1:3682] "HMSET" "{demo:cache}:redisson_options" "max-size" "2" "mode" "LRU"
1660568866.708868 [5 127.0.0.1:3681] "EVAL" "local insertable = false; local value = redis.call('hget', KEYS[1], ARGV[5]); local t, val;if value == false then insertable = true; else t, val = struct.unpack('dLc0', value); local expireDate = 92233720368547758; local expireDateScore = redis.call('zscore', KEYS[2], ARGV[5]); if expireDateScore ~= false then expireDate = tonumber(expireDateScore) end; if t ~= 0 then local expireIdle = redis.call('zscore', KEYS[3], ARGV[5]); if expireIdle ~= false then expireDate = math.min(expireDate, tonumber(expireIdle)) end; end; if expireDate <= tonumber(ARGV[1]) then insertable = true; end; end; if tonumber(ARGV[2]) > 0 then redis.call('zadd', KEYS[2], ARGV[2], ARGV[5]); else redis.call('zrem', KEYS[2], ARGV[5]); end; if tonumber(ARGV[3]) > 0 then redis.call('zadd', KEYS[3], ARGV[3], ARGV[5]); else redis.call('zrem', KEYS[3], ARGV[5]); end; local maxSize = tonumber(redis.call('hget', KEYS[8], 'max-size')); local mode = redis.call('hget', KEYS[8], 'mode'); if maxSize ~= nil and maxSize ~= 0 then     local currentTime = tonumber(ARGV[1]);     local lastAccessTimeSetName = KEYS[6]; if mode == false or mode == 'LRU' then redis.call('zadd', lastAccessTimeSetName, currentTime, ARGV[5]); end;     local cacheSize = tonumber(redis.call('hlen', KEYS[1]));     if cacheSize >= maxSize then         local lruItems = redis.call('zrange', lastAccessTimeSetName, 0, cacheSize - maxSize);         for index, lruItem in ipairs(lruItems) do             if lruItem and lruItem ~= ARGV[5] then                 local lruItemValue = redis.call('hget', KEYS[1], lruItem);                 redis.call('hdel', KEYS[1], lruItem);                 redis.call('zrem', KEYS[2], lruItem);                 redis.call('zrem', KEYS[3], lruItem);                 redis.call('zrem', lastAccessTimeSetName, lruItem);                 if lruItemValue ~= false then                 local removedChannelName = KEYS[7]; local ttl, obj = struct.unpack('dLc0', lruItemValue);                local msg = struct.pack('Lc0Lc0', string.len(lruItem), lruItem, string.len(obj), obj);                redis.call('publish', removedChannelName, msg); end;             end;         end;     end; if mode == 'LFU' then redis.call('zincrby', lastAccessTimeSetName, 1, ARGV[5]); end; end; local value = struct.pack('dLc0', ARGV[4], string.len(ARGV[6]), ARGV[6]); redis.call('hset', KEYS[1], ARGV[5], value); if insertable == true then local msg = struct.pack('Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6]); redis.call('publish', KEYS[4], msg); return 1;else local msg = struct.pack('Lc0Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6], string.len(val), val); redis.call('publish', KEYS[5], msg); return 0;end;" "8" "demo:cache" "redisson__timeout__set:{demo:cache}" "redisson__idle__set:{demo:cache}" "redisson_map_cache_created:{demo:cache}" "redisson_map_cache_updated:{demo:cache}" "redisson__map_cache__last_access__set:{demo:cache}" "redisson_map_cache_removed:{demo:cache}" "{demo:cache}:redisson_options" "1660568866705" "1660568926705" "1660569466705" "600000" "\"test\"" "{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568866.709243 [5 lua] "hget" "demo:cache" "\"test\""
1660568866.709255 [5 lua] "zadd" "redisson__timeout__set:{demo:cache}" "1660568926705" "\"test\""
1660568866.709272 [5 lua] "zadd" "redisson__idle__set:{demo:cache}" "1660569466705" "\"test\""
1660568866.709281 [5 lua] "hget" "{demo:cache}:redisson_options" "max-size"
1660568866.709290 [5 lua] "hget" "{demo:cache}:redisson_options" "mode"
1660568866.709300 [5 lua] "zadd" "redisson__map_cache__last_access__set:{demo:cache}" "1660568866705" "\"test\""
1660568866.709311 [5 lua] "hlen" "demo:cache"
1660568866.709318 [5 lua] "hset" "demo:cache" "\"test\"" "\x00\x00\x00\x00\x80O\"AY\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568866.709339 [5 lua] "publish" "redisson_map_cache_created:{demo:cache}" "\x06\x00\x00\x00\"test\"Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568871.706402 [5 127.0.0.1:3685] "EVAL" "if redis.call('setnx', KEYS[6], ARGV[4]) == 0 then return -1;end;redis.call('expire', KEYS[6], ARGV[3]); local expiredKeys1 = redis.call('zrangebyscore', KEYS[2], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys1) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;for i=1, #expiredKeys1, 5000 do redis.call('zrem', KEYS[5], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('zrem', KEYS[3], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('zrem', KEYS[2], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('hdel', KEYS[1], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); end; local expiredKeys2 = redis.call('zrangebyscore', KEYS[3], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys2) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;for i=1, #expiredKeys2, 5000 do redis.call('zrem', KEYS[5], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('zrem', KEYS[3], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('zrem', KEYS[2], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('hdel', KEYS[1], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); end; return #expiredKeys1 + #expiredKeys2;" "6" "demo:cache" "redisson__timeout__set:{demo:cache}" "redisson__idle__set:{demo:cache}" "redisson_map_cache_expired:{demo:cache}" "redisson__map_cache__last_access__set:{demo:cache}" "redisson__execute_task_once_latch:{demo:cache}" "1660568871705" "100" "5" "1"
1660568871.706626 [5 lua] "setnx" "redisson__execute_task_once_latch:{demo:cache}" "1"
1660568871.706638 [5 lua] "expire" "redisson__execute_task_once_latch:{demo:cache}" "5"
1660568871.706652 [5 lua] "zrangebyscore" "redisson__timeout__set:{demo:cache}" "0" "1660568871705" "limit" "0" "100"
1660568871.706667 [5 lua] "zrangebyscore" "redisson__idle__set:{demo:cache}" "0" "1660568871705" "limit" "0" "100"
1660568872.233405 [5 127.0.0.1:3647] "ping"
1660568875.015321 [5 127.0.0.1:3686] "GET" "Authorization:login:token:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw"
1660568875.016461 [5 127.0.0.1:3687] "GET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw"
1660568875.017166 [5 127.0.0.1:3688] "PTTL" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw"
1660568875.018006 [5 127.0.0.1:3689] "SET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw" "\"1660568875016\""
1660568875.018042 [5 127.0.0.1:3689] "EVAL" "local result = 0;for j = 1, #KEYS, 1 do local expireSet; if ARGV[2] ~= '' then expireSet = redis.call('pexpire', KEYS[j], ARGV[1], ARGV[2]); else expireSet = redis.call('pexpire', KEYS[j], ARGV[1]); end; if expireSet == 1 then result = expireSet;end; end; return result; " "1" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw" "85871000" ""
1660568875.018111 [5 lua] "pexpire" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw" "85871000"
1660568875.019530 [5 127.0.0.1:3690] "EVAL" "local insertable = false; local value = redis.call('hget', KEYS[1], ARGV[5]); local t, val;if value == false then insertable = true; else t, val = struct.unpack('dLc0', value); local expireDate = 92233720368547758; local expireDateScore = redis.call('zscore', KEYS[2], ARGV[5]); if expireDateScore ~= false then expireDate = tonumber(expireDateScore) end; if t ~= 0 then local expireIdle = redis.call('zscore', KEYS[3], ARGV[5]); if expireIdle ~= false then expireDate = math.min(expireDate, tonumber(expireIdle)) end; end; if expireDate <= tonumber(ARGV[1]) then insertable = true; end; end; if tonumber(ARGV[2]) > 0 then redis.call('zadd', KEYS[2], ARGV[2], ARGV[5]); else redis.call('zrem', KEYS[2], ARGV[5]); end; if tonumber(ARGV[3]) > 0 then redis.call('zadd', KEYS[3], ARGV[3], ARGV[5]); else redis.call('zrem', KEYS[3], ARGV[5]); end; local maxSize = tonumber(redis.call('hget', KEYS[8], 'max-size')); local mode = redis.call('hget', KEYS[8], 'mode'); if maxSize ~= nil and maxSize ~= 0 then     local currentTime = tonumber(ARGV[1]);     local lastAccessTimeSetName = KEYS[6]; if mode == false or mode == 'LRU' then redis.call('zadd', lastAccessTimeSetName, currentTime, ARGV[5]); end;     local cacheSize = tonumber(redis.call('hlen', KEYS[1]));     if cacheSize >= maxSize then         local lruItems = redis.call('zrange', lastAccessTimeSetName, 0, cacheSize - maxSize);         for index, lruItem in ipairs(lruItems) do             if lruItem and lruItem ~= ARGV[5] then                 local lruItemValue = redis.call('hget', KEYS[1], lruItem);                 redis.call('hdel', KEYS[1], lruItem);                 redis.call('zrem', KEYS[2], lruItem);                 redis.call('zrem', KEYS[3], lruItem);                 redis.call('zrem', lastAccessTimeSetName, lruItem);                 if lruItemValue ~= false then                 local removedChannelName = KEYS[7]; local ttl, obj = struct.unpack('dLc0', lruItemValue);                local msg = struct.pack('Lc0Lc0', string.len(lruItem), lruItem, string.len(obj), obj);                redis.call('publish', removedChannelName, msg); end;             end;         end;     end; if mode == 'LFU' then redis.call('zincrby', lastAccessTimeSetName, 1, ARGV[5]); end; end; local value = struct.pack('dLc0', ARGV[4], string.len(ARGV[6]), ARGV[6]); redis.call('hset', KEYS[1], ARGV[5], value); if insertable == true then local msg = struct.pack('Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6]); redis.call('publish', KEYS[4], msg); return 1;else local msg = struct.pack('Lc0Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6], string.len(val), val); redis.call('publish', KEYS[5], msg); return 0;end;" "8" "demo:cache" "redisson__timeout__set:{demo:cache}" "redisson__idle__set:{demo:cache}" "redisson_map_cache_created:{demo:cache}" "redisson_map_cache_updated:{demo:cache}" "redisson__map_cache__last_access__set:{demo:cache}" "redisson_map_cache_removed:{demo:cache}" "{demo:cache}:redisson_options" "1660568875018" "1660568935018" "1660569475018" "600000" "\"test1\"" "{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568875.019854 [5 lua] "hget" "demo:cache" "\"test1\""
1660568875.019864 [5 lua] "zadd" "redisson__timeout__set:{demo:cache}" "1660568935018" "\"test1\""
1660568875.019906 [5 lua] "zadd" "redisson__idle__set:{demo:cache}" "1660569475018" "\"test1\""
1660568875.019917 [5 lua] "hget" "{demo:cache}:redisson_options" "max-size"
1660568875.019924 [5 lua] "hget" "{demo:cache}:redisson_options" "mode"
1660568875.019933 [5 lua] "zadd" "redisson__map_cache__last_access__set:{demo:cache}" "1660568875018" "\"test1\""
1660568875.019944 [5 lua] "hlen" "demo:cache"
1660568875.019950 [5 lua] "hset" "demo:cache" "\"test1\"" "\x00\x00\x00\x00\x80O\"AZ\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568875.019971 [5 lua] "publish" "redisson_map_cache_created:{demo:cache}" "\a\x00\x00\x00\"test1\"Z\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test1\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568876.711315 [5 127.0.0.1:3682] "EVAL" "if redis.call('setnx', KEYS[6], ARGV[4]) == 0 then return -1;end;redis.call('expire', KEYS[6], ARGV[3]); local expiredKeys1 = redis.call('zrangebyscore', KEYS[2], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys1) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;for i=1, #expiredKeys1, 5000 do redis.call('zrem', KEYS[5], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('zrem', KEYS[3], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('zrem', KEYS[2], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('hdel', KEYS[1], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); end; local expiredKeys2 = redis.call('zrangebyscore', KEYS[3], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys2) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;for i=1, #expiredKeys2, 5000 do redis.call('zrem', KEYS[5], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('zrem', KEYS[3], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('zrem', KEYS[2], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('hdel', KEYS[1], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); end; return #expiredKeys1 + #expiredKeys2;" "6" "demo:cache" "redisson__timeout__set:{demo:cache}" "redisson__idle__set:{demo:cache}" "redisson_map_cache_expired:{demo:cache}" "redisson__map_cache__last_access__set:{demo:cache}" "redisson__execute_task_once_latch:{demo:cache}" "1660568876710" "100" "5" "1"
1660568876.711558 [5 lua] "setnx" "redisson__execute_task_once_latch:{demo:cache}" "1"
1660568876.711571 [5 lua] "expire" "redisson__execute_task_once_latch:{demo:cache}" "5"
1660568876.711585 [5 lua] "zrangebyscore" "redisson__timeout__set:{demo:cache}" "0" "1660568876710" "limit" "0" "100"
1660568876.711615 [5 lua] "zrangebyscore" "redisson__idle__set:{demo:cache}" "0" "1660568876710" "limit" "0" "100"
1660568881.719984 [5 127.0.0.1:3681] "EVAL" "if redis.call('setnx', KEYS[6], ARGV[4]) == 0 then return -1;end;redis.call('expire', KEYS[6], ARGV[3]); local expiredKeys1 = redis.call('zrangebyscore', KEYS[2], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys1) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;for i=1, #expiredKeys1, 5000 do redis.call('zrem', KEYS[5], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('zrem', KEYS[3], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('zrem', KEYS[2], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('hdel', KEYS[1], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); end; local expiredKeys2 = redis.call('zrangebyscore', KEYS[3], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys2) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;for i=1, #expiredKeys2, 5000 do redis.call('zrem', KEYS[5], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('zrem', KEYS[3], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('zrem', KEYS[2], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('hdel', KEYS[1], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); end; return #expiredKeys1 + #expiredKeys2;" "6" "demo:cache" "redisson__timeout__set:{demo:cache}" "redisson__idle__set:{demo:cache}" "redisson_map_cache_expired:{demo:cache}" "redisson__map_cache__last_access__set:{demo:cache}" "redisson__execute_task_once_latch:{demo:cache}" "1660568881719" "100" "5" "1"
1660568881.720210 [5 lua] "setnx" "redisson__execute_task_once_latch:{demo:cache}" "1"
1660568881.720223 [5 lua] "expire" "redisson__execute_task_once_latch:{demo:cache}" "5"
1660568881.720236 [5 lua] "zrangebyscore" "redisson__timeout__set:{demo:cache}" "0" "1660568881719" "limit" "0" "100"
1660568881.720250 [5 lua] "zrangebyscore" "redisson__idle__set:{demo:cache}" "0" "1660568881719" "limit" "0" "100"
1660568882.232483 [5 127.0.0.1:3647] "ping"
1660568884.694637 [5 127.0.0.1:3682] "PING"
1660568884.694652 [5 127.0.0.1:3681] "PING"
1660568884.694657 [5 127.0.0.1:3684] "PING"
1660568884.694725 [5 127.0.0.1:3685] "PING"
1660568884.695285 [5 127.0.0.1:3686] "PING"
1660568884.695292 [5 127.0.0.1:3687] "PING"
1660568884.695575 [5 127.0.0.1:3688] "PING"
1660568884.695941 [5 127.0.0.1:3689] "PING"
1660568884.696019 [5 127.0.0.1:3690] "PING"
1660568886.187073 [5 127.0.0.1:3647] "scan" "0" "MATCH" "*" "COUNT" "500"
1660568886.192898 [5 127.0.0.1:3647] "info"
1660568887.369769 [5 127.0.0.1:3647] "type" "demo:cache"
1660568887.394142 [5 127.0.0.1:3647] "ttl" "demo:cache"
1660568887.396718 [5 127.0.0.1:3647] "hlen" "demo:cache"
1660568887.405131 [5 127.0.0.1:3647] "hscan" "demo:cache" "0" "MATCH" "*" "COUNT" "200"
1660568888.735152 [5 127.0.0.1:3685] "EVAL" "if redis.call('setnx', KEYS[6], ARGV[4]) == 0 then return -1;end;redis.call('expire', KEYS[6], ARGV[3]); local expiredKeys1 = redis.call('zrangebyscore', KEYS[2], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys1) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;for i=1, #expiredKeys1, 5000 do redis.call('zrem', KEYS[5], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('zrem', KEYS[3], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('zrem', KEYS[2], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); redis.call('hdel', KEYS[1], unpack(expiredKeys1, i, math.min(i+4999, table.getn(expiredKeys1)))); end; local expiredKeys2 = redis.call('zrangebyscore', KEYS[3], 0, ARGV[1], 'limit', 0, ARGV[2]); for i, key in ipairs(expiredKeys2) do local v = redis.call('hget', KEYS[1], key); if v ~= false then local t, val = struct.unpack('dLc0', v); local msg = struct.pack('Lc0Lc0', string.len(key), key, string.len(val), val); local listeners = redis.call('publish', KEYS[4], msg); if (listeners == 0) then break;end; end;end;for i=1, #expiredKeys2, 5000 do redis.call('zrem', KEYS[5], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('zrem', KEYS[3], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('zrem', KEYS[2], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); redis.call('hdel', KEYS[1], unpack(expiredKeys2, i, math.min(i+4999, table.getn(expiredKeys2)))); end; return #expiredKeys1 + #expiredKeys2;" "6" "demo:cache" "redisson__timeout__set:{demo:cache}" "redisson__idle__set:{demo:cache}" "redisson_map_cache_expired:{demo:cache}" "redisson__map_cache__last_access__set:{demo:cache}" "redisson__execute_task_once_latch:{demo:cache}" "1660568888733" "100" "7" "1"
1660568888.735531 [5 lua] "setnx" "redisson__execute_task_once_latch:{demo:cache}" "1"
1660568888.735547 [5 lua] "expire" "redisson__execute_task_once_latch:{demo:cache}" "7"
1660568888.735566 [5 lua] "zrangebyscore" "redisson__timeout__set:{demo:cache}" "0" "1660568888733" "limit" "0" "100"
1660568888.735590 [5 lua] "zrangebyscore" "redisson__idle__set:{demo:cache}" "0" "1660568888733" "limit" "0" "100"
1660568889.035013 [5 127.0.0.1:3686] "GET" "Authorization:login:token:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw"
1660568889.035492 [5 127.0.0.1:3687] "GET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw"
1660568889.035957 [5 127.0.0.1:3688] "PTTL" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw"
1660568889.036382 [5 127.0.0.1:3689] "SET" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw" "\"1660568889034\""
1660568889.036416 [5 127.0.0.1:3689] "EVAL" "local result = 0;for j = 1, #KEYS, 1 do local expireSet; if ARGV[2] ~= '' then expireSet = redis.call('pexpire', KEYS[j], ARGV[1], ARGV[2]); else expireSet = redis.call('pexpire', KEYS[j], ARGV[1]); end; if expireSet == 1 then result = expireSet;end; end; return result; " "1" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw" "85856000" ""
1660568889.036480 [5 lua] "pexpire" "Authorization:login:last-activity:eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpblR5cGUiOiJsb2dpbiIsImxvZ2luSWQiOiJzeXNfdXNlcjoxIiwicm5TdHIiOiJOQVNiTDl4SmYxSVZFZnpTMk9HeFB5UG5LeW5HTXVPMCJ9.PsBOEkagDvsT3ZCtiFRgBNs5Mte7eDZsWgjbL_446pw" "85856000"
1660568889.038546 [5 127.0.0.1:3690] "EVAL" "local insertable = false; local value = redis.call('hget', KEYS[1], ARGV[5]); local t, val;if value == false then insertable = true; else t, val = struct.unpack('dLc0', value); local expireDate = 92233720368547758; local expireDateScore = redis.call('zscore', KEYS[2], ARGV[5]); if expireDateScore ~= false then expireDate = tonumber(expireDateScore) end; if t ~= 0 then local expireIdle = redis.call('zscore', KEYS[3], ARGV[5]); if expireIdle ~= false then expireDate = math.min(expireDate, tonumber(expireIdle)) end; end; if expireDate <= tonumber(ARGV[1]) then insertable = true; end; end; if tonumber(ARGV[2]) > 0 then redis.call('zadd', KEYS[2], ARGV[2], ARGV[5]); else redis.call('zrem', KEYS[2], ARGV[5]); end; if tonumber(ARGV[3]) > 0 then redis.call('zadd', KEYS[3], ARGV[3], ARGV[5]); else redis.call('zrem', KEYS[3], ARGV[5]); end; local maxSize = tonumber(redis.call('hget', KEYS[8], 'max-size')); local mode = redis.call('hget', KEYS[8], 'mode'); if maxSize ~= nil and maxSize ~= 0 then     local currentTime = tonumber(ARGV[1]);     local lastAccessTimeSetName = KEYS[6]; if mode == false or mode == 'LRU' then redis.call('zadd', lastAccessTimeSetName, currentTime, ARGV[5]); end;     local cacheSize = tonumber(redis.call('hlen', KEYS[1]));     if cacheSize >= maxSize then         local lruItems = redis.call('zrange', lastAccessTimeSetName, 0, cacheSize - maxSize);         for index, lruItem in ipairs(lruItems) do             if lruItem and lruItem ~= ARGV[5] then                 local lruItemValue = redis.call('hget', KEYS[1], lruItem);                 redis.call('hdel', KEYS[1], lruItem);                 redis.call('zrem', KEYS[2], lruItem);                 redis.call('zrem', KEYS[3], lruItem);                 redis.call('zrem', lastAccessTimeSetName, lruItem);                 if lruItemValue ~= false then                 local removedChannelName = KEYS[7]; local ttl, obj = struct.unpack('dLc0', lruItemValue);                local msg = struct.pack('Lc0Lc0', string.len(lruItem), lruItem, string.len(obj), obj);                redis.call('publish', removedChannelName, msg); end;             end;         end;     end; if mode == 'LFU' then redis.call('zincrby', lastAccessTimeSetName, 1, ARGV[5]); end; end; local value = struct.pack('dLc0', ARGV[4], string.len(ARGV[6]), ARGV[6]); redis.call('hset', KEYS[1], ARGV[5], value); if insertable == true then local msg = struct.pack('Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6]); redis.call('publish', KEYS[4], msg); return 1;else local msg = struct.pack('Lc0Lc0Lc0', string.len(ARGV[5]), ARGV[5], string.len(ARGV[6]), ARGV[6], string.len(val), val); redis.call('publish', KEYS[5], msg); return 0;end;" "8" "demo:cache" "redisson__timeout__set:{demo:cache}" "redisson__idle__set:{demo:cache}" "redisson_map_cache_created:{demo:cache}" "redisson_map_cache_updated:{demo:cache}" "redisson__map_cache__last_access__set:{demo:cache}" "redisson_map_cache_removed:{demo:cache}" "{demo:cache}:redisson_options" "1660568889036" "1660568949036" "1660569489036" "600000" "\"test2\"" "{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test2\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568889.038959 [5 lua] "hget" "demo:cache" "\"test2\""
1660568889.038993 [5 lua] "zadd" "redisson__timeout__set:{demo:cache}" "1660568949036" "\"test2\""
1660568889.039044 [5 lua] "zadd" "redisson__idle__set:{demo:cache}" "1660569489036" "\"test2\""
1660568889.039085 [5 lua] "hget" "{demo:cache}:redisson_options" "max-size"
1660568889.039093 [5 lua] "hget" "{demo:cache}:redisson_options" "mode"
1660568889.039104 [5 lua] "zadd" "redisson__map_cache__last_access__set:{demo:cache}" "1660568889036" "\"test2\""
1660568889.039116 [5 lua] "hlen" "demo:cache"
1660568889.039120 [5 lua] "zrange" "redisson__map_cache__last_access__set:{demo:cache}" "0" "0"
1660568889.039134 [5 lua] "hget" "demo:cache" "\"test\""
1660568889.039139 [5 lua] "hdel" "demo:cache" "\"test\""
1660568889.039144 [5 lua] "zrem" "redisson__timeout__set:{demo:cache}" "\"test\""
1660568889.039151 [5 lua] "zrem" "redisson__idle__set:{demo:cache}" "\"test\""
1660568889.039158 [5 lua] "zrem" "redisson__map_cache__last_access__set:{demo:cache}" "\"test\""
1660568889.039185 [5 lua] "publish" "redisson_map_cache_removed:{demo:cache}" "\x06\x00\x00\x00\"test\"Y\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568889.039229 [5 lua] "hset" "demo:cache" "\"test2\"" "\x00\x00\x00\x00\x80O\"AZ\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test2\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
1660568889.039311 [5 lua] "publish" "redisson_map_cache_created:{demo:cache}" "\a\x00\x00\x00\"test2\"Z\x00\x00\x00{\"@class\":\"com.ruoyi.common.core.domain.R\",\"code\":200,\"data\":\"test2\",\"msg\":\"\xe6\x93\x8d\xe4\xbd\x9c\xe6\x88\x90\xe5\x8a\x9f\"}"
```