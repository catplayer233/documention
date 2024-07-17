# Redis Lua 使用指南

我们都知道redis是一款非常易用且功能强大的nosql数据库，由于其数据结构的丰富，以及非常快速的执行速度，常常我们会使用redis做一些非缓存相关的需求，如排行榜，秒杀等。但是随着业务的复杂度的提升，我们常常需要同时执行多个redis命令，但是这些命令的执行如果都是放在客户端去发起，常常存在着一些问题。
- 原子性：一般多条命令同时执行是处理某个单一请求，但如果多条命令执行过程中有失败的情况，则会导致已执行的命令的结果无法回滚，因而出现一致性的问题。当然这种问题也可以通过redis的事务来避免，但是redis的事务存在着其自身的问题。
- 性能：多条命令的执行就意味着客户端要发起多次网络调用，如果这类场景比较多，数据量比较大，也会对我们的服务造成负担。
- 逻辑复杂：每条命令我们都需要对结果进行处理，然后再发起新的命令，这个过程，会使得我们的执行逻辑变复杂，同时也加大了我们维护成本和调试的难度。

那么有没有那么一种东西，可以让我们像操作内存一样去操作redis中的数据呢，可以客户端发起的单次请求中处理n个数据呢，甚至可以对n个数据进行处理，产生一个新的数据呢？如果redis没有这么一样东西，那么它也不会变成nosql中的头部产品了。redis对此提供的方案就是：支持lua编程。

## why lua?
我们都知道redis是纯c编写的软件。而lua就是这么一款寄生在c语言中的脚本语言（起码lua官方的解释器是c的）。因此使用lua，对于redis的团队来说就是一件非常简单的事情。同时lua作为一门脚本语言，其语法简洁，动态执行，用来做一些简单的逻辑是完全符合需求的。

## how lua?
那么我们应该怎么在redis中使用lua呢？

### 命令

使用lua编程，只要发起命令，将lua代码以参数的形式发送到redis，redis就会执行，并将结果返回。

`EVAL lua_script numkeys [key [key ...]] [arg [arg ...]]`

但是如果脚本代码量比较大，每次都传入会影响带宽，redis同样也贴心的给我们提供了一些更加方便的命令

`SCRIPT LOAD script //该命令会将脚本保存在redis中，并返回一个sha1的摘要值`

`EVALSHA sha1 numkeys [key [key ...]] [arg [arg ...]] //将上述命令中返回的摘要保存到客户端，后续可以直接使用该命令执行目标脚本`

### 参数

上述命令我们可以看到除去脚本参数，其他的参数分为了两组，一种叫做key，一种叫做arg，那么这两者有什么区别呢？

#### key
key 就是指代的lua脚本中需要使用到的redis中关联的数据的key，如保存用户信息的key，保存排行榜的key等等。

#### arg
arg 则指代的lua脚本中一些变量的值，如获取排行榜的前n名，这个n则需要通过arg来进行传递。

这里大家可能会疑惑，为啥要分key和arg呢，都放在key或者arg传递不就行了，这里因为redis的集群下，key可能分配在不同的slot里，而lua脚本中使用到的key都必须在相同的slot中才可以成功执行，因此，key单独出来，就可以进行校验以及定位本次脚本的执行节点。

### 编程

那么lua脚本应该怎么操作数据呢？如果需要对数据进行json序列化/反序列化又该怎么办呢？对此redis为我们内置了一些api，以及集成了一些常用的库。

#### API

##### redis 对象

`redis.call(command [,arg...]) //执行redis命令，执行失败会抛异常`

`redis.pcall(command [,arg...]) //执行redis命令，失败不抛异常`

##### KEYS, ARGV

KEYS(数组) 全局变量，保存的是命令中传递的key参数

ARGV(数组) 全局变量，保存的是命令中传递的arg参数

#### 库

redis集成了众多常用的第三方库：
- struct
- cjson
- bit
- cmsgpack

需要注意的是，lua官方库redis并不是都支持的，以下列出的是其支持的
- string
- table
- math

#### 数据类型映射

##### to lua
- RESP2 integer reply -> Lua number
- RESP2 bulk string reply -> Lua string
- RESP2 array reply -> Lua table (may have other Redis data types nested)
- RESP2 status reply -> Lua table with a single ok field containing the status string
- RESP2 error reply -> Lua table with a single err field containing the error string
- RESP2 null bulk reply and null multi bulk reply -> Lua false boolean type

##### from lua
- number -> RESP2 integer reply (the number is converted into an integer)
- string -> RESP bulk string reply
- table (indexed, non-associative array) -> RESP2 array reply (truncated at the first Lua nil value encountered in the table, if any)
- table with a single ok field -> RESP2 status reply
- table with a single err field -> RESP2 error reply
- boolean false -> RESP2 null bulk reply
- Boolean true -> RESP2 integer reply with value of 1.

## 示例
``` lua
-- KEYS[1]: a hash key
-- ARG[1]:  the hash entry key

local item_id = ARGV[1]

local item_json = redis.call("HGET", KEYS[1], item_id)

if not item_json then
    return -2
end

-- convert json string to table
local item = cjson.decode(item_json)

if not item then
    return -3
end

local current_left_num = item["left_num"]

if current_left_num <= 0 then
    return -1
end

item["left_num"] = current_left_num - 1

local target = cjson.encode(item)

redis.call("HSET", KEYS[1], item_id, target)

-- return current remained num
return current_left_num - 1
```

## 注意

- 集群情况下确保使用的key都在相同的slot，可以通过使用 `hash tag` 保证（不建议，会破坏数据的平均分布）
- 使用阿里云版本的redis时，脚本中不要将key不要使用变量，应该直接使用KEYS[1,2,3...]
- 尽可能避免使用执行慢的命令，如 `keys`，redis对于执行过久的脚本，会主动kill掉，因此导致执行失败。同时过慢的速度也会严重影响整体服务的稳定性。

## 参考

- [Redis Lua API reference](https://redis.io/docs/latest/develop/interact/programmability/lua-api/#redis_object)
- [EVAL](https://redis.io/docs/latest/commands/eval/)
- [RedisLua脚本的基本语法与使用规范_云数据库 Redis 版-阿里云帮助中心](https://help.aliyun.com/zh/redis/support/usage-of-lua-scripts?spm=a2c4g.11186623.0.0.6c4bff26zzrM2e#section-8f7-qgv-dlv)