# Redis的使用技巧

Redis有着非常丰富的数据结构可供我们使用，不仅仅是传统的key-value（说的就是你，Memcached），如`hash`、`list`、`set`等等。同时Redis是用纯c写的因此在延迟、内存占用少方面都是比较优秀的。

## 积分排行榜设计

活动期间，玩家可以实时（客户端会进行缓存）查询排行榜信息，因此如果通过查询数据库，则要求每次查询都要对数据库中的积分信息进行排行比较，这样效率不高且数据库io压力较大。因此我们通过redis的zset来实现这一功能。

zset：Sorted sets是一个支持排序的集合，该集合的元素包含了一个数字叫做scores（ double 64-bit floating point number ）和一个客户端自己定义的value，该集合每次添加新的元素是都会根据scores进行排序，算法参见：[Skiplist算法以及在redis中的应用](./Skiplist算法以及在redis中的应用.md) 

因此使用时我们可以基于zset来实现我们的积分排行榜。

### Scores选择
不同玩家的累计积分会出现相同的情况，因此在使用时，不能直接以玩家的积分值作为scores，所以我们需要另外一个比较元素：玩家达到积分的时间。相同积分的玩家，谁最先达到该积分则谁应该排在前面。所以比较的逻辑应该是：
```
if(积分1>积分2)
    return 1;

if(积分1==积分2){
    if(时间1<时间2)
        return 1
    }
```

为了达到这一比较目的，我们设计了Scores的计算公式为：
`SCORES = 10000000 * score + (endTime - archiveTime);`

即高位保存积分，低位记录时间

### Value设计

value的值需要包含客户端进行展示的必要信息：玩家头像、玩家昵称等。如果我们每次都将玩家的基本信息都存入Value中，会导致数据不一致的问题，当玩家修改了头像、昵称等，我们无法将这部分数据及时更新到Value中，则会导致数据存在误差。

为了解决这个问题，我们实际上在设计Value的时候，只将userId作用Value，随后建立了一个用户基本信息的hash，key为userId，这样我们在最终返回数据时，则会拼接hash中保存的用户基本信息。

### 交互

上述的方案基本上满足了积分排行榜的设计，但是，我们在返回最终数据时，还需要对zset中的每个数据进一步查询用户基本信息的hash，这样一次请求则会产生（1+N）的io，为了解决这一问题，我们通过编写lua脚本，通过脚本来完成查询hash并进行封装，并最后返回结果集。这样我们就通过redis的执行lua脚本这一条命令，就可以得到最终结果，io只发生了1次。
```lua
--1->zset, 2->userId, 3->hash table, 4->rankNum
local zsetName = KEYS[1]
local userId = KEYS[2]
local hashTableName = KEYS[3]
local rankMinIndex = 0
local rankMaxIndex = KEYS[4] - 1

local userRankInfos = redis.call('ZREVRANGE', zsetName, rankMinIndex, rankMaxIndex, 'WITHSCORES')
local userRankMessage
local rankUserMessages = { 'null' }

local rankIndex = 1
for i, value in ipairs(userRankInfos) do
    if (i % 2 == 0) then

    else
        local score = userRankInfos[i + 1]
        local userdata = redis.call('hget', hashTableName, value)
        --userdata(json):score:rank
        local rankMessage = userdata .. '->' .. score .. '->' .. rankIndex
        rankIndex = rankIndex+1
        rankUserMessages[#rankUserMessages + 1] = rankMessage
        if userId == value then
            userRankMessage = rankMessage
            rankUserMessages[1] = userRankMessage
        end
    end
end
if userRankMessage == nil then
    local rank = redis.call('zrevrank', zsetName, userId)
    if rank then
        local score = redis.call('zscore', zsetName, userId)
        local userdata = redis.call('hget', hashTableName, userId)
        userRankMessage = userdata .. "->" .. score .. "->" .. (rank + 1)
        rankUserMessages[1] = userRankMessage
    end
end
return rankUserMessages
```

那么怎么写lua脚本让Redis进行执行呢？

## 使用lua脚本

因为Redis是c写的，那么Redis执行lua文件也是轻而易举的事情（lua虚拟机可以轻松内嵌在c程序进程里），因此Redis在2.6.0开始就内置了lua的解释器，因此我们只需要按照Redis提供的一些api（在lua中调用redis的命令），就可以轻松的实现跨多个数据集获取数据（zset+hash）。

### 命令
`eval <lua_file_bytes> <key_count> key1 key2 ... value1 value2 ...`

### 使用lua交互的优点
- 正如上述介绍的，使用lua，客户端只需要执行单个命令，就可以进行复杂的计算，减少了io的次数。
- 高并发下，使用lua，也可以保证原子性，因为Redis的工作线程是单线程的，执行单个lua脚本可以保证原子性。
- 维护性，lua写的溜也是一个考虑因素，Redis同时还支持对执行的lua脚本进行debug。

## 参考
- [EVAL – Redis](https://redis.io/docs/latest/commands/eval/)
- [Redis Lua scripts debugger – Redis](https://redis.io/docs/latest/develop/interact/programmability/lua-debugging/)
- [Data types – Redis](https://redis.io/docs/latest/develop/data-types/)
