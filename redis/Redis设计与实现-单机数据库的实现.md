# Redis设计与实现-单机数据库的实现

## 一、数据库

#### 服务器中的数据库

Redis服务器所有数据库保存在服务器状态redis.h/redisServer结构的db数组中，数组中的每个redisDb结构代表一个数据库：

```c
struct redisServer {
    // ...
    // 一个数组，保存服务器的所有数据库
    redisDb *db;
    // 服务器的数据库数量
    int dbnum;
    // ...
}
```

+ dbnum：初始化服务器时，程序会根据服务器状态的dbnum属性决定应该创建多少个数据库，默认为16

![image-20210530170658616](https://gitee.com/tobing/imagebed/raw/master/image-20210530170658616.png)

#### 切换数据库

默认情况下，Redis的目标数据库为0号数据库，客户端可以通过SELECT命令来切换目标数据库。 

在服务器内部，客户端状态redisClient结果的db属性记录了客户端当前的目标数据库，是一个执行redisDb结构的指针：

```c
typedef struct redisClient {
    // ...
    // 记录客户端当前正在使用的数据库
    redisDb *db;
    // ...
} redisClient;
```

如下图，一个Redis客户端通过`select 1`连接到了Redis的1号数据库

![image-20210530171257179](https://gitee.com/tobing/imagebed/raw/master/image-20210530171257179.png)

此时如果要将Redis客户端连接到2号数据库，仅需要修改db的指针，这就是SELECT的命令原理。

#### 数据库键空间

Redis是一个键值对数据库服务器，里面的每个数据库都由redis.h/redisDb结构表示：

```c
typedef struct redisDb {
    // ...
    // 数据库键空间，保存数据库中的所有键值对
    dict *dict;
    // ...
}
```

键空间和用所见的数据库直接对应：

+ 键空间的键就是数据库的键，每个键都是一个字符串对象
+ 键空间的值就是数据库的值，每个值可以是字符串、列表、哈希表、集合和有序集合的任意一种Redis对象

![image-20210530171954551](https://gitee.com/tobing/imagebed/raw/master/image-20210530171954551.png)

从上面可以发现，该数据库一共有三个键分别为：

+ alphabet，列表键，值是一个包含是哪个元素的列表对象
+ book，哈希表键，值是有个包含三个键值对的哈希表对象
+ message，字符串键，值是有个包含hello world的字符串对象

因为数据库的键空间地址是一个字典，因此所有针对数据库的操作，如添加、更新、删除一个键值对，实际上都是对键空间的字典进行操作。

+ **添加键**：添加一个新键值对到数据库，实际上是添加一个新键值对到数据库的键空间字典上，其中键为字符串对象，值则可以是任意一种类型Redis对象。

+ **删除键**：删除数据库的一个键，实际上是在键空间里面删除键所对应的键值对对象。

+ **更新键**：对数据库键进行更新，实际上是在对键空间里面键所对应的值进行更新。

+ **对键取值**：对一个数据库键进行取值，实际上是在键空间上取出键对应的值的对象，根据值对象类型不同，具体的取值方法也不同。

+ **其他键空间操作**：如FLUSHDB命令，就是通过删除键空间中所有的键值对实现；如RANDOMKEY就是通过通过在空间中随机返回一个键。

+ **读写键空间时的维护**：使用Redis命令对数据库读写时，服务器除了会对键空间执行指定的读写操作，还会执行一些额外的维护操作，如：
  + 读取一个键之后，会根据键是否存在来更新数据库键空间的命中次数或不命中次数。
  + 读取一个键之后，服务器会更新键的LRU时间，这个值可以用于计算键的闲置时间。
  + 读取一个键发现已经过期，那么数据库就会删除这个过期键，然后才执行余下操作。
  + 如果有客户端使用WATCH命令监视某个键，那么服务器在对被被监视的键进行修改之后，会将这个键标记为dirty，从而让事务程序知道这个键已经修改。
  + 服务器每次修改一个键之后，都会对dirty键计数器加1，这个数计数器会触发数据库的持久化和复制操作。
  + 如果服务器开启了数据库通知功能，那么对键的修改，服务器将按配置发送相应的数据库通知。

#### TTL

**设置键的生存时间或过期时间**

通过EXPIRE命令或PEXPIRE命令，客户端可以以秒或毫秒精度为数据库中的某个键设置TTL，在经过指定的秒数或者毫秒数之后，服务器就会自动删除TTL为0的键。

与EXPIRE和PEXPIRE命令类似，客户端可以通过EXPIREAT或PEXPIREAT命令，以秒或毫秒精度个数据库中的某个键设置过期时间，这个过期时间是一个UNIX的时间戳，的过期时间来临，服务器会自动从数据库中删除这个键。

TTL或PTTL命令可以查询一个带生存时间或过期时间的键，返回距离这个键被删除还有多长时间。

Redis有四个不同命令可以用于设置键的生存时间或过期时间：

+ `EXPIRE [key] [ttl]`命令用于将key的生存时间设置为ttl秒
+ `PEXPIRE [key] [ttl]`命令用于将key的生存时间设置为ttl毫秒
+ `EXPIREEAT [key] [timestamp]`命令用于将key的生存时间设置为timestamp所指定的秒数时间戳
+ `PEXPIREAT [key] [timestamp]`命令用于将可以的过期时间设置为timestamp所指定的毫秒数时间戳

是加上，EXPIRE/PEXPIRE/EXPIREAT三个命令都是经过PEXPIREAT命令实现：

![image-20210530175535973](https://gitee.com/tobing/imagebed/raw/master/image-20210530175535973.png)

redisDb结构的expires字典保存了数据库所有键的过期时间，称为这个字典为过期字典：

```c
typedef struct redisDb {
    // ...
    // 过期字典，保存键的过期时间
    dict *expires;
    // ...
} redisDb;
```

+ 过期字典的键是有个指针，指向键空间的某个键对象
+ 过期字典的值是一个long long类型的整数，保存了键指向的数据库键的过期时间-一个毫秒精度的UNIX时间戳

![image-20210530180414209](https://gitee.com/tobing/imagebed/raw/master/image-20210530180414209.png)

下面是PEXPIREAT的伪代码：

```python
def PEXPIREAT(key, expire_time_in_ms) :
	# 如果给定键不存在键空间，不能设置过期时间
	if key not in redisDb.dict:
        return 0
    # 在过期字典中关联键和过期时间
    redisDb.expires[key] = expire_time_in_ms
    # 过期时间设置成功
    return 1
```

**移除过期时间**

PERSIST命令可以移除一个键的过期时间，是PEXPIREAT命令的反操作，其伪代码如下：

```python
def PERSISST(key) :
    # 如果键不存在，或者没有设置过期时间，直接返回
    if key not in redisDb.expires:
        return 0
    # 移除过期时间字典中给定的键值对关联
    redisDb.expires.remove(key)
    # 键的过期时间移除成功
    return 1;
```

**计算并返回TTL**

 TTL和PTTL命令都是通过计算键的过期时间和当前时间之间的差实现，一下为两个命令的伪代码：

```python
def PTTL(key):
	# 判断键是否存在数据库中
    if key not in redisDb.dict:
        return -2;
    # 获取键的过期时间戳
    expire_time_in_ms = redisDb.expires.get(key)
    # 判断键是否合法
    if expires_time_in_ms is None:
        return -1;
    # 获取当前的时间
    now_ms = get_current_unix_timestamp_in_ms()
    # 让过期时间减去当前时间，即为ttl
    return (expire_time_in_ms - now_ms)

def TTL(key):
    # 获取以毫秒为单位的TTL
    ttl_in_ms = PTTL(key)
    if ttl_in_ms < 0:
        # 处理返回值为-1/-2的情况
        return ttl_in_ms;
    else:
        # 将毫秒转换为秒
        return ms_to_sec(ttl_in_ms)
```

**过期键的判定**

通过过期字典，程序可以通过以下步骤检查一个给定键是否过期：

1. 检查给定键是否存在于过期字典，如果存在则取出键的过期时间。
2. 检查当前UNIX时间戳是否大于过期时间，如果是，那么键已经过期；否则键未过期。

#### 过期键的删除策略

如果一个键过期，将会有以下三种删除策略：

+ **定时删除：**设置键的过期时间的同时，参加一个定时器，让定时器在键过期时间来临时，立即执行删除操作。【主动删除】
+ **惰性删除：**放任键过期不管，当每次从空间中获取键时，先检查键是否过期，过期则惰性删除，没有过期则返回。【被动删除】
+ **定期删除：**每个一段时间，程序对数据库就行一次检查，删除里面的过期键。【主动删除】

**定时删除**

定时删除对内存友好，但是对CPU不友好。定时删除可以确保过期键的尽快删除，释放占用的内存；如果存在键较大的情况下，同时删除键可能会代理性能问题，占用CPU时间。

创建定时器需要使用Redis服务器中的时间事件，而处理时间事件的实现是无序链表，查找的时间复杂度为O(N)。

**惰性删除**

惰性删除对CPU友好，但是对内存不友好。惰性删除只会在取键的时候对键的过期时间进行检查，可以保证删除键的操作只在非做不可才执行；如果一个过期键一直不被访问，会一直占用内存。（内存泄漏）

**定期删除**

定期删除是一种折中方案。每隔一定时间执行一次删除过期键操作，可以通过限制删除频率来限制对CPU的影响。

**惰性删除实现**

惰性删除策略有expireIfNeeded函数实现，所有读写数据库的Redis命令在执行前都会调用国内expireIfNeeded函数对输入键机型检查：

+ 键已过期，expireIfNeeded将输入键数据库中移除
+ 键未过期，expireIfNeeded不做操作

![image-20210531094234422](https://gitee.com/tobing/imagebed/raw/master/image-20210531094234422.png)

**定期删除实现**

定期删除策略有activeExpireCycle函数实现，每当Redis服务器周期性操作serverCron函数执行时，该函数会被调用，它在规定的时间，多次遍历服务器中各个数据库，移除过期键。工作模式如下：

1. 每次运行时，从一定数量的数据库中取出一定数量的随机键进行检查，并删除过期键；
2. current_db会记录当前activeExpireCycle函数的进度，并在下一次activeExpireCycle调用时，接着上一次进度进行处理；
3. 随着函数的不断执行，数据库中的所有服务器都会被检查一遍，这是函数将current_db设置为0，然后开启一轮新的检查。

#### AOF、RDB和复制功能对过期键的处理

**生成RDB文件**

在中save命令或gbsave命令创建一个新的RDB文件时，程序会对数据库中的键进行检查，已过期的键不会被保存到新创建的RDB文件中。因此数据库中包含过期键不会对生成新的RDB文件造成影响。

**载入RDB文件**

启动Reids服务器时，如果启用RDB功能，服务器对RDB文件进行载入：

+ 如果当前服务器为主服务器模式运行，载入RDB时程序对文件中保存的键检查，未过期的键会被载入，过期则会被忽略，一次过期键对载入RDB文件的主服务器不会造成影响。
+ 如果当前服务器以从服务器模式运行，载入RDB文件时，不论其中的键是否过期，都会被载入。但是一般来说过期键对载入RDB的从服务器也不会造成影响。

**AOF文件写入**

以AOF持久化模式运行时，如果数据库中某个键已经过期，但还没有被惰性删除或定期删除，那么AOF文件不会因这个过期键而产生任何影响。

当过期键被惰性删除或定期删除之后，程序会向AOF追加一条DEL命令，显式记录该键已经被删除。

**AOF重写**

和RDB生成类似，AOF重写的时候，程序对数据库中的键进行检查，已经过期的键不会保存在和重写的AOF文件中。因此过期键不会对AOF重写造成影响。

**复制**

在主从复制模式下，从服务器的过期键生成由主服务器控制：

+ 主服务器在删除一个过期键之后，会显式向所有从服务器发送一条DEL命令，告诉从服务器删除该键；
+ 从服务器再执行客户端发送的读命令时，即使碰到过期键也不会自定删除，而是继续像处理未过期的键一样进行处理过期键（还是会返回）；
+ 从服务器只有在接到主服务器发来的DEL命令才删除过期键。

通过由主服务器控制从服务器统一删除过期键，可以确保主从服务器数据一致性。

#### 数据库通知

Redis2.8引入了一个新功能，数据库通知，可以让客户端通过订阅给定的频道或模式，获取数据库中键的变化以及数据库中命令的执行情况。

下面展示了获取0号数据库针对msg键执行的所有操作：

```bash
subscribe __keyspace@0__:msg
```

这一类关注“某个键执行了什么命令”的通知称为键空间通知。除此之外，还有一类称为**键事件通知**，它们**关注的是某个命令被什么键执行了**。

下面展示了客户端如何获取0号数据库中所有执行DEL命令的键

```bash
subscribe __keyevent@0__:del
```

服务器通过配置`notify-keyspace-events`选项来决定发送通知的类型：

+ 发送所有类型的键空间通知和键事件通知，将其设置为AKE；
+ 发送所有类型的键空间通知，将其设置为AK；
+ 发送所有类型的键事件通知，将其设置为AE；
+ 只发送和字符串键有关的键空间通知，将其设置为K$；
+ 只发送和列表键有关的键事件通知，将其设置为El。

**发送通知**

发送数据库通知有notifyKeyspaceEvent函数实现：

```c
void notifyKeyspaceEvent(int type, char*event, robj *key, int dbid);
```

+ event、key和dbid分别是事件的名称、产生事件的键、以及产生事件的数据库号码；
+ 函数会根据type以及上述三个参数构建事件通知的内容以及接收通知的频道号。

每当一个Redis命令需要发送数据库通知时，该命令的实现函数就会调用notify-notifyKeyspaceEvent函数，并向函数传递命令引发的事件的相关信息。  

下面是notifyKeyspaceEvent伪代码实现：

```python
void notifyKeyspaceEvent(type, event, key, dbid):
    # 给定的通知不是服务器允许的，直接返回
    if not(server.notify_keyspace_event & type):
        return
    # 发送键空间通知
    if server.notify_kepsapce_events & REDIS_NOTIFY_KEYSAPCE:
        # 构建频道名字
        chan = "__keysapce@{dbid}__:{key}".format(dbid=dbid, key=key)
        # 发送通知
        pubsubPublishMessage(chan, event)
        
    # 发送键事件通知
    if server.notify_kepsapce_events & REDIS_NOTIFY_KEYEVENT:
        # 构建频道名字
        chan = "__keyevent@{dbid}__:{key}".format(dbid=dbid, key=key)
        # 发送通知
        pubsubPublishMessage(chan, key)
```

+ notify_kepsapce_events属性是服务器配置信息，如果给定的通知类型type不是服务器允许发送的类型，则直接返回。
+ 如果给定的通知是服务器允许发送的通知，下一步函数会检测服务器是否允许发送键空间通知，如果允许程序会构建事件并发送。
+ 最后，函数检测服务器是否允许发送键事件通知，允许则构建并发送。

#### 总结

Redis服务器的所有数据库都保存在redisServer.db数组中，而数据库的数量则有redisServer.dbnum属性保存；

客户端通过修改目标数据库指针，让它指向RedisServer.db数组中的不同元素来切换不同数据库；

数据库由dict和expires两个字典构成，其中dict字典负责保存键值对，expires字典保存键的过期时间；

因为数据库由字典构成，所以对数据库的所有操作构建在字典操作之上。

数据库中的键是有个字符串对象，而值可以是任意一种Redis对象类型，包括字符串、哈希表、集合对象、列表对象和有序集合对象，分别对应字符串键、哈希表键、集合键、列表键和有序集合键。

expires字典的键指向数据库中的某个键，而值记录了数据库键的过期时间，过期时间是以毫秒为单位的UNIX时间戳。

Redis使用惰性删除和定期删除两种策略来删除过期的键：惰性删除只在碰到过期才执行删除操作，定期删除会每个一段时间主动查找以及过期键进行删除。

执行save或bgsave产生的新RDB文件不会包含以及过期的键。

执行bgrewriteaof命令产生的重写AOF文件不糊包含以及过期的键。

当一个过期键被删除，服务器会追加一条DEL命令到AOF文件末尾，显式删除键。

让主服务器删除一个过期键，会先所有其他从服务器发送一条DEL命令，显式删除过期键。

主从模式下，从服务器即使在键过期之后，仍可能会被访问，知道主服务器发送的DEL命令到达，这种统一、中心化的过期键删除策略可以保证主从数据库的数据一致性。

当Redis命令对数据经修改之后，服务器会根据配置向客户端发送数据库通知。

## 二、RDB持久化

#### 概述

Redis是内存数据库，即将数据状态信息保存在内存。如果没有其他的保护措施，一旦Redis进程退出，Redis中的数据将会丢失。

为了解决这个问题，Redis提供了持久化功能，可以将内存的数据保存到磁盘中，主要有RDB和AOF两种方式。

RDB持久化既可以手动执行，也可以根据服务器配置选项定期执行，该功能可以将某个时间点的数据库状态保存到一个RDB文件中。

RDB是一个经过压缩的二进制文件，可以通过该文件还原某一个时刻数据库的状态。

![image-20210531111316765](https://gitee.com/tobing/imagebed/raw/master/image-20210531111316765.png)

#### RDB文件的创建与载入

SAVE和BGSAVE命令都可以生成RDB文件。

SAVE命令会阻塞Redis服务器进程，知道RDB文件创建完毕；BGSAVE命令会派生出一个子进程，然后有子进程负责创建RDB文件，服务器进程继续处理命令请求。

创建RDB文件实际由rdb.c/rdbSave函数完成，SAVE命令和BGSAVE命令以不同方式调用该函数：

```python
def SAVE():
    rdbSave()

def BGSAVE():
    # 创建子进程
    pid = fork()
    if pid == 0:
        # 子进程创建RDB文件
        rdbSave()
        # 完成之后向父进程发送信号
        signal_parent()
    elif pid > 0:
        # 父进程继续处理命令请求，并通过轮询等待子进程的信号
        handle_request_and_wait_signal()
    else:
        # 处理出错的情况
        handle_fork_error()
```

和RDB的创建不同，RDB的载入是由服务器启动时自动执行，Redis没有专门用于载入RDB文件的命令，只要Redis服务器启动时检测到RDB文件存在，就会自动载入RDB文件。

需要注意的是，AOF文件的更新频率通常比RDB文件更新频率高，因此：

+ 如果服务器开启了AOF，有优先使用AOF还原数据；
+ 只有在AOF关闭时，服务器才会使用RDB文件来还原数据；

![image-20210531112546293](https://gitee.com/tobing/imagebed/raw/master/image-20210531112546293.png)



**SAVE命令执行时服务器状态**

SAVE执行时服务器会被阻塞，这时，客户端发来的所有命令请求都会被拒绝。

只有在服务器执行完SAVE命令、重新开始接受命令请求之后，客户端发送的命令才会被处理。

**BGSAVE命令执行时服务器状态**

BGAVE命令的保存工作交给子进程执行，子进程在创建RDB文件的过程中，Redis仍可以处理客户端的命令请求，但在执行BGSAVE命令期间，服务器处理SAVE、BGSAVE和BGREWRITEAOF三个命令的方式和平时有所不同：

1. BGSAVE执行期间，客户端发送的SAVE命令会被服务器拒绝，服务器禁止SAVE命令和BGSAVE命令同时执行时为了避免父进程和子进程同时调用rdbSave，进而引发资源竞争。

2. BGSAVE执行期间，客户端发送的BGSAVE命令会被拒绝，同时执行两个BGSAVE也会引发资源竞争。

3. BGREWRITEAOF和BGSAVE两个命令不能同时执行：

   + 如果BGSAVE命令执行，那么客户端发送的BGREWRITEAOF命令会被延迟到BGSAVE命令执行完毕之后执行；
   + 如果BGREWRITEAOF，命令正在执行，那么客户端发送的BGSAVE命令会被服务器拒绝。

   BGREWRITEAOF和BGSAVE都是由子进程执行，两个命令没有冲突的地方，不能同时执行主要是从性能方面考虑，并发两个子进程来同时执行大量的磁盘写入会严重应影响性能。

**RDB文件载入时服务器状态**

服务器载入RDB时，会一直处于阻塞状态，知道载入工作完成为止。

#### 自动间隔性保存

再使用BGSAVE时，可以通过配置save来设定BGSAVE的执行间隔，配置如下：

```bash
save 900 1		# 900秒之内，对数据库至少修改1次 
save 300 10		# 300秒之内，对数据库至少需改10次
save 60 10000	# 60秒之内，对数据库只是10000次修改
```

满足上述任一条件都会触发BGSAVE。

上面配置的参数将会被读入到saveparams结构中：

```c
struct saveparam {
    // 秒数
    time_t seconds;
    // 修改数
    int changes;
}
```

![image-20210531121753807](https://gitee.com/tobing/imagebed/raw/master/image-20210531121753807.png)

除了saveparams结构，服务器还维持了一个dirty计数器和lastsave属性。

+ dirty计数器，记录距离上一次成功执行SAVE或BGSAVE命令之后，服务器对数据库状态进行了多少次修改
+ lastsave属性，一个UNIX时间戳，记录服务器上一次成功执行SAVE命令或BGSAVE命令的时间

```c
struct redisServer {
    // ...
    
    // 记录了保存条件的数组
    struct saveparam *saveparams;
    // 修改计数器
    long long dirty;
    // 上一次执行保存的时间
    time_t lastsave;
    
    // ...
}
```

Reids服务器周期性操作函数serverCron默认每隔100ms执行一次，该函数用于对正在运行的服务器进行维护，其中一项工作是检查save选项设置的保存条件是否满足，满足则执行BGSAVE命令。

#### RDB文件结构

一个完整的RDB文件主要包含五部分，如下图：

![image-20210531121902554](https://gitee.com/tobing/imagebed/raw/master/image-20210531121902554.png)

+ REDIS部分，5字节，保存“REDIS”五个字符，通过这五个字符，快速检查载入的文件是否为RDB文件。
+ db_version部分，4字节，是一个字符串表示的整数，整数记录了RDB文件的版本号，如“0006”表示为第六版。

+ database部分，包含另个或多个数据库，以及数据库中键值对的数据：
  + 如果服务器的数据状态为空，那么这个部分也为空，长度0字节；
  + 如果数据库的数据状态非空，那么这个部分非空，长度不固定；
+ EOF常量部分，1字节，标志RDB文件内容结束，遇到则表示所有键值对已经载入完成。
+ check_sum部分，8字节，无符号整数，保存了校验和，对前4部分进行计算得出，载入时通过判断此值来判断文件是否已经损坏。

一个RDB文件的database部分可以保存多个非空的数据库，如下图所示：

![image-20210531122555079](https://gitee.com/tobing/imagebed/raw/master/image-20210531122555079.png)

每个非空数据库在RDB文件中都可以保存为三部分，下图所示：

![image-20210531122627907](https://gitee.com/tobing/imagebed/raw/master/image-20210531122627907.png)

+ SELECTDB，1字节，读到这个值表示下面是一个数据库号码。
+ db_number，1/2/5字节，读到这部分会调用SELECT命令读入该数值就行数据库切换，保证key_value_paris载入正确的数据库中。
+ key_value_pairs，部分保存了数据库的所有键值对数据，如果键值对带有过期时间，那么过期时间会和键值对保存在一起。

key_value_pairs部分保存了了一个或以上数量的键值对，如果键值对有过期时间，也会保存在其中，完整结构如下：

![image-20210531123055840](https://gitee.com/tobing/imagebed/raw/master/image-20210531123055840.png)

+ EXPIRETIME_MS，1字节，告知读入程序，接下来读入的是有个以毫秒为单位的时间戳。【可选】
+ ms，8字节，以毫秒为单位的UNIX时间戳，对应键值对的过期时间【可选】
+ TYPE，1字节，记录了value的类型，程序会根据TYPE值来决定如何读入和解释value的数据：
  + REDIS_RDB_TYPE_STRING/LIST/SET/ZSET/AHSH/LIST_ZIPLIST/SET_INSET/ZSET_ZIPLIST/HASH_ZIPLIST。
+ key和value分别保存了键值对的键对象和值对象。

对于不同类型RDB中的value部分结构和长度有所不同：

**字符串对象（REDIS_RDB_TYPE_STRING）**

字符串对的编码可以是INT、RAW。INT表示为长度不超过32位的整数，value采用下图方式保存

![image-20210531124156854](https://gitee.com/tobing/imagebed/raw/master/image-20210531124156854.png)

当编码为RAW，根据字符串长度不停，会采用不同方式保存字符串：

+ 不压缩，字符串长度小于等于20字节，直接原样保存；
+ 压缩，字符串长度大于20字节，先压缩再保存。

无压缩（上）与压缩（下）

![image-20210531124343196](https://gitee.com/tobing/imagebed/raw/master/image-20210531124343196.png)

![image-20210531124401164](https://gitee.com/tobing/imagebed/raw/master/image-20210531124401164.png)

+ REDIS_RDB_ENC_LZF，表明字符串被LZF算法压缩
+ compressed_len，压缩之后的长度
+ origin_len，压缩前长度
+ compressed_string，压缩后的内容

下面分别展示了压缩和不压缩的储存方式

![image-20210531124535395](https://gitee.com/tobing/imagebed/raw/master/image-20210531124535395.png)

**列表对象（REDIS_RDB_TYPE_LIST）**

REDIS_RDB_TYPE_LIST表示保存一个LINKEDLIST编码的列表对象，结构如下：

![image-20210531124811436](https://gitee.com/tobing/imagebed/raw/master/image-20210531124811436.png)

+ list_lenght，记录列表长度
+ item，每一项的的具体内容

**集合对象（REDIS_RDB_TYPE_SET）**

REDIS_RDB_TYPE_SET表示保存一个REDIS_ENCODING_HT编码的集合对象，结构如下：

![image-20210531125133068](https://gitee.com/tobing/imagebed/raw/master/image-20210531125133068.png)

+ set_size，集合大小
+ elemN，元素内容

**哈希表对象（REDIS_RDB_TYPE_HASH）**

REDIS_RDB_TYPE_HASH表示保存一个REDIS_ENCODING_HT编码的集合对象，结构如下：

![image-20210531125453167](https://gitee.com/tobing/imagebed/raw/master/image-20210531125453167.png)

+ hash_size，哈希表大小
+ key_value_pairN：，键值对数据

**有序集合对象（REDIS_RDB_TYPE_ZSET）**

REDIS_RDB_TYPE_ZSET表示保存一个REDIS_ENCODING_SKIPLIST编码的有序集合对象，结构如下：

![image-20210531125710812](https://gitee.com/tobing/imagebed/raw/master/image-20210531125710812.png)

+ sorted_set_size，有序集合大小
+ elementN，每一项有序集合元素

**INSET编码集合（REDIS_RDB_TYPE_SET_INTSET）**

REDIS_RDB_TYPE_SET_INTSET表示保存一个整数集合对象。

**ZIPLSIT编码的集合、哈希表或有序集合（REDIS_RDB_TYPE_LIST_ZIPLIST、REDIS_RDB_TYPE_HASH_ZIPLIST、REDIS_RDB_TYPE_ZSET_ZIPLIST）**

REDIS_RDB_TYPE_LIST_ZIPLIST、REDIS_RDB_TYPE_HASH_ZIPLIST、REDIS_RDB_TYPE_ZSET_ZIPLIST表示value是压缩列表对象。

RDB保存这种文件的方法如下：

1. 将压缩列表转换成一个字符串对象；
2. 将转换得到的字符串对象保存到RDB文件。

RDB解析这种文件的方法如下：

1. 读入字符串对象，将其转换为压缩类表对象。
2. 根据TYPE，设置压缩类表的类型。

#### 分析RDB文件

往Redis中写入部分文件，执行SAVE命令。然后通过`od`命令，分析Redis产生RDB文件。

```bash
# 清空Redis数据库
FLUSHALL
# 手动生成RDB
SAVE
```

```bash
# 以ASCII编码打印dump.rdb
od -C dump.rdb
```

## 三、AOF持久化

#### 概述

除了RDB持久化，Redis还提供了AOF（Append Only File）持久化功能。

但与RDB持久化不同，AOF持久化通过保存Redis服务器执行的写命令来记录服务器的状态。

#### AOF持久化实现

AOF持久化功能可以分为命令追加（append）、文件写入、文件同步（sync）三步。

**命令追加**

AOF持久化打开时，服务器在执行完一个写命令后，会以协议格式将被执行的写命令追加到服务器状态的aof_buf缓冲区末尾。

```c
struct redisServer {
    // ...
    // AOF缓冲区
    sds aof_buf;
    // ...
}
```

下面展示了AOF文件的内容：

![image-20210601214321802](https://gitee.com/tobing/imagebed/raw/master/image-20210601214321802.png)

每执行一个写命令，都会将协议内容添加到aof_buf缓存区的末尾。

**文件写入与同步**

可以将Redis服务器进程看作一个时间循环，不断接收客户端的命令请求，响应客户端，执行定时函数，执行aof_buf缓冲区追加任务等，其伪代码如下：

```python
def eventLoop():
    while True:
        # 处理文件事件请求，接收命令请求以及发送命令回复
        # 处理命令请求是可能会有新内容被追加到aof_buf缓冲区中
        processFileEvents()
        
        # 处理时间事件
        processTimeEvents()
        
        # 考虑是要将aof_buf中的内容写入和保存到aof文件里面
        flushAppendOnlyFile()
```

flushAppendOnlyFile的行为有服务器参数appendfsync选项值来决定，如下表

| appendfsync | 行为                                             |
| ----------- | ------------------------------------------------ |
| always      | 总是将aof_buf缓冲区的内容刷新到AOF文件中         |
| everysec    | 每秒将aof_buf缓冲区的内容刷新到AOF文件中         |
| no          | aof_buf缓冲区的内容刷新到AOF文件中的时机由OS决定 |

显然，这几个选项可以在不同的场景灵活调整，分别是针对性能和可靠性来考虑，之前在Redis核心技术与实战提到过，不赘述。

#### AOF载入与数据还原

AOF文件记录了重建数据库的所有写命令，因此服务器只需要读入并执行AOF文件中的命令即可还原数据库的状态。

Redis读取AOF文件的详细步骤如下：

1. 重建一个不带网络连接的伪客户端，执行AOF文件保存的命令。
2. 从AOF文件中分析并读取出一条写命令。
3. 使用伪客户端执行读取出的写命令。
4. 重复2和3，指定AOF文件所有写命令被执行完。

![image-20210601220939491](https://gitee.com/tobing/imagebed/raw/master/image-20210601220939491.png)

#### AOF重写

AOF持久化时通过写命令的方式来保存数据库状态，随着数据库的执行，AOF文件的内容将会越来越多（即使数据量可能并没有增大，如对一个数据反复修改），AOF文件大小将会越来越大，最终可能会超出操作系统的文件的限制等。而且如果AOF文件越大，使用AOF文件还原需要的时间就越长。

为了解决AOF文件体积太大的问题Redis提供了AOF重写的功能。通过该功能，Redis会创建一个新的AOF文件，代替已有的AOF文件，新旧两个AOF文件保存的数据库状态相同，但新的AOF文件值保存数据的最终状态，因此新的AOF文件通常比旧的AOF文件小。

需要注意，虽然称为“AOF重写”，但并不是代表重写AOF文件是从原AOF文件中读取。Redis中AOF重写是读取数据库的键的最新状态，这样就可能采用一条命令来保存之前若干条命令执行的状态，如下：

```bash
# 旧AOF文件记录内容
SADD set "aaa"
SADD set "bbb"
SADD set "ccc"
SADD set "ddd"

# AOF重写优化
SADD set "aaa" "bbb" "ccc" "ddd"
```

下面是AOF重写的伪代码如下：

```python
def aof_rewrite(new_aof_file_name):
    # 创建新AOF文件
    # 遍历数据库
    	# 忽略空数据库
        # 写入SELECT命令，指定当前数据库号码
        # 遍历数据库的所有键，进行重写
        # 如果键有过期时间，过期时间也需要被重写记录
    # 写入完毕，关闭文件
```

需要注意，对于集合元素，通过批量写入命令来优化AOF空间

从上面可以返现，新的AOF文件只包含当前数据库最新状态的命令，因此与就的AOF文件相比更加节省空间。

为了避免AOF重写对数据库主程序造成阻塞，Redis将AOF重写写入到子进程执行，可以实现：

+ 子进程进行AOF重写期间，服务器进程可以进行处理命令请求；
+ 子进程带有服务器进行的副本，子进程可以避免使用保证数据库安全性。【关系到操作系统的细节，需要自己深入】

在子进程AOF重写期间，服务器还需要处理新的命令请求，这段时间的修改通过**AOF重写缓冲区**来记录。

AOF重写期间，服务器需要执行以下三个工作：

1. 执行客户端发来的命令
2. 将执行后的命令追加到AOF缓冲区（AOF缓冲区的内容定期被刷回，保证现有AOF文件正常处理）
3. 将执行后的写命令追加到AOF重写缓冲区（保证在AOF重写阶段的写命令都会被记录）

![image-20210601223548517](https://gitee.com/tobing/imagebed/raw/master/image-20210601223548517.png)

当子进程执行完AOF重写工作，会先父进程发送一个信号，父进程在接到信号会调用一个信号处理函数，并执行一下操作：

1. 将AOF重写缓冲区的所有内容写到新的AOF文件中，这是新的AOF文件保存的数据库状态和现有的一致；
2. 对新的AOF文件进行改名，原子性覆盖现有的AOF文件，完成新旧AOF文件的替换。

父进程处理完子进程的信号后就可以像往常一样处理命令请求。

## 四、事件

#### 概述

Redis服务器是一个事件驱动程序，服务器需要处理一下两种事件：

+ 文件事件（file event）：Redis服务器通过套接字与客户端进行连接，而键事件就是服务器对套接字操作的抽象。服务器与客户端的通信会产生文件事件，而服务器则通过监听并处理这些事件来完成一系列的网络通信操作。
+ 时间事件（time event）：Redis服务器中的一些操作需要在给定时间执行，时间事件就是服务器对这类定时操作的抽象。

#### 文件事件

Redis是基于Reactor模式开发自己的网络事件处理器，称为文件事件处理器（file event handler）

+ 文件事件处理器使用IO多路复用（multiplexing）程序来同时监听多个套接字，并根据套接字目前执行的任务为套接字关联不同的事件处理器。
+ 当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关闭（colse）等操作时，与操作相对应的文件事件就会产生，这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件。

虽然文件事件处理器以单线程运行，但是通过使用IO多路复用程序来监听多个套接字，即实现了高性能的网络通信模型，又可以很好地与Redis服务器中其他以单线程方式运行的模块进行对接，这保持了Redis内部单线程的简单性。

文件事件处理器有四部分组成：套接字、IO多路复用程序、文件事件分派器以及事件处理器。

![image-20210602112138289](https://gitee.com/tobing/imagebed/raw/master/image-20210602112138289.png)

文件事件是对套接字操作的抽象，每当一个套接字准备好执行连接应答、写入、读取、关闭等操作时，就会产生一个文件事件。因为一个服务器通常会连接多个套接字，因此多个文件事件可能会并发出现。

**IO多路复用程序**负责监听多个套接字，并向**文件事件分派器**传送传送产生了事件的套接字。

尽管多个文件事件可能会并发出现，但IO多路复用程序总是将所有产生事件的套接字都放到一个队列，通过这个队列，有序、同步、依次将一个套接字传送到文件事件分派器。当上一个套接字产生的事件被处理完毕之后，IO多路复用程序才会继续向文件事件分派器传送下一个套接字。

![image-20210602112728786](https://gitee.com/tobing/imagebed/raw/master/image-20210602112728786.png)

文件事件分派器接收IO多路复用传来的套接字，并根据套接字产生的事件类型，调用相应的事件处理器，这些处理器都是一个个函数，定义了某事件发生是，服务器应该执行的动作。

**IO多路复用程序的实现**

Redis的IO多路复用程序的所有功能都是通过包装常见的select、epoll、evport和kqueue这些IO多路复用函数库来实现，每个IO多路复用函数库在Redis源码中都对应一个单独的文件，如ae_select.c、ae_epoll.c、ae_kqueue.c等。

Redis为每个IO多路复用函数库都实现了相同的API，因此IO多路复用程序的底层实现是可以互换的：

![image-20210602113352347](https://gitee.com/tobing/imagebed/raw/master/image-20210602113352347.png)

IO多路复用程序可以监听多个套接字的AE_READABLE事件和AE_WRITABLE事件，两类事件的与套接字之间的对应关系如下：

+ 当套接字变得可读（客户端对套接字执行write或close），或者有新的可应答套接字出现时（客户端对服务器的监听套接字执行connect），套接字产生AE_READABLE事件。
+ 当套接字变得可写（客户端对套接字执行read），套接字产生AE_WRITEABLE事件。

如果一个套接字同时产生了上述的两种事件，先处理AE_READABLE，处理完才处理AE_WRITABLE。

**API**

+ ae.c/aeCreateFileEvent函数，接受一个套接字描述符、一个事件类型以及一个事件处理器，将给定的套接字的给定事件加入到IO多路复用程序的监听范围之内，并对事件和事件处理器关联。
+ ae.c/aeDeleteFileEvent函数，接受一个套接字描述符和一个监听时间类型，让IO多路复用程序取消对给定套接字的监听、取消事件和事件处理器之间的关联
+ ae.c/aeGetFileEvents函数，接受一个套接字描述符返回套接字正在被监听的事件类型。
+ ae.c/aeWait函数，接受一个套接字描述符、一个事件类型和一个毫秒数为参数，在给定阻塞时间内，阻塞并等待套接字的给定类型事件产生，当事件成功产生，或者等待超时之后，函数返回。
+ ae.c/aeApiPoll函数，接受struct timeval结构，并在指定时间内阻塞并等待所有被aeCreateFileEvent函数设置为监听状态的套接字产生文件事件，当有至少一个事件产生，或者等待超时后，函数返回。
+ ae.c/aeProcessEvents函数是文件事件分派器，先调用aeApiPoll函数来等待事件产生，然后遍历所有已经产生的事件，并调用相应的事件处理器来处理这些事件。
+ ae.c/aeGetApiName函数返回IO多路复用程序底层使用的函数库名称。

**文件事件处理器**

Redis问文件事件编写了多个处理器，这处理器分别用于实现不同网络通信需求：

+ 监听套接字关联连接应答处理器，对连接服务器的各个客户端进行应答
+ 为客户端套接字关联命令请求处理器，接收客户端传来的命令请求
+ 为客户端套接字关联命令回复处理器，先客户端返回命令的执行结果
+ 复制处理器，用于主从复制

其中最常用的为**连接应答处理器、命令请求处理器和命令回复处理器**。

1. 连接应答处理器

   用于对连接服务器监听套接字的客户端进行应答。Redis初始化时，将连接应答处理器与AE_READABLE事件关联，当有客户端用connect函数连接服务器监听套接字，就会产生AE_READABLE事件引发连接应答处理器执行

   ![image-20210602120043935](https://gitee.com/tobing/imagebed/raw/master/image-20210602120043935.png)

2. 命令请求处理器

   负责从套接字中读入客户端发送的命令请求内容。客户端通过连接应答处理器成功连接到服务器之后发送命令请求时，引发命令请求处理器执行。

   ![image-20210602120235756](https://gitee.com/tobing/imagebed/raw/master/image-20210602120235756.png)

3. 命令回复处理器

   负责将服务器执行的命令回复通过套接字返回给客户端。当服务器有命令回复需要传送给客户端时，服务器会将客户端套接字的AE_WRITEABLE时间和命令回复处理器关联，当客户端准备好接收服务器传回的命令回复是，就会产生AE_WRITEABLE事件，引发命令回复处理器执行，并执行相应套接字写入操作。

   ![image-20210602120521129](https://gitee.com/tobing/imagebed/raw/master/image-20210602120521129.png)

   当命令发送完毕，服务器会接触命令回复处理器与客户端套接字的AE_WRITEABLE事件关联。

综上所述，一次Redis客户端与服务器连接，发送命令的过程如下：

1. Redis服务器运行，服务器监听套接字的AE_READABLE时间处于监听状态，对应的处理器为连接应答处理器；
2. 此时一个Redis客户端发起连接，监听套接字产生AE_READABLE事件，触发连接应答处理器执行；
3. 连接应答处理器对客户端请求进行应答，之后创建客户端套接字、客户端状态，将AE_READABLE事件与命令请求处理器关联；
4. 接下来，客户端可以服务器发送命令请求，之后每发送一个命令，客户端套接字产生AE_READABLE事件，引发命令请求处理器执行；
5. 出去读取客户端的命令内容，传到相关程序执行，执行完毕将产生响应的命令回复；
6. 服务器将客户端套接字的AE_WRITEABLE时间与命令回复处理器关联；
7. 当客户端尝试读取命令回复时，会产生AE_WRITEABLE时间，触发命令回复处理器执行；
8. 当命令回复处理器将命令回复全部写入到套接字之后，服务器会解除客户端套接字与AE_WRITEABLE事件与命令回复处理器之间的关联。

![image-20210602121341968](https://gitee.com/tobing/imagebed/raw/master/image-20210602121341968.png)

#### 时间事件

Redis的时间事件分类为两类：

+ 定时事件：让一段程序在指定时间之后执行一次。
+ 周期事件：让一段程序每隔指定时间就执行一次。

一个时间事件由三个属性组成：

+ id：服务器为时间事件创建全局唯一ID，从小到大递增，新事件ID比旧时间ID大。
+ when：毫秒精度的UNIX时间戳，记录了时间事件到达时间。
+ timeProc：时间事件处理器，一个函数。时间事件到达，服务器调用相关的处理器处理事件。

一个时间事件是定时事件还是正确时间取决于时间事件处理器的返回值：

+ 返回AE_NOMORE，为定时事件，事件到达一次之后就被删除，之后不再到达；
+ 返回非AE_NOMORE，为周期事件，根据返回的值，对时间事件的when属性更新，让整个事件在一段时间之后再次到达，并一这种方式一致套下去。

服务器将所有的时间事件放到一个无序链表，每当时间事件执行器运行时，会遍历整个链表，查找所有已经到达的时间事件，调用相应的事件处理器。

![image-20210602122311819](https://gitee.com/tobing/imagebed/raw/master/image-20210602122311819.png)

注意，上述链表为无序，不是不按照Id排序，而是不按照when大小排序。因此时间事件执行器执行时需要遍历链表，处理到达（UNIX时间戳小于当前系统时间戳）的时间事件。

由于正常模式下，Reids服务器只是用serverCron一个时间事件，而benchmark模式下，也只使用两个时间事件，相当于是只有一个节点或两个节点，退化为一个指针的使用，不会影响事件执行效率。

**serverCron函数**

Redis会通过serverCron函数来定期对自身自检和状态进行检查和调整，确保自身能够持续运行，其中serverCron函数的主要工作包括：

+ 更新服务器各类统计信息，如时间、内存占用、数据库占用情况等
+ 清理数据库中的过期键值对
+ 关闭和清理连接失效的客户端
+ 尝试进行AOF或RDB持久化操作
+ 如果服务器是主服务器，对服务器进行定期同步
+ 如果处于集群模式，对集群进行定期同步和连接测试

Redis服务器以周期性事件的方式运行serverCron函数，在服务器运行期间，每隔一段时间，serverCron会执行一次，知道服务器关闭。

serverCron函数的运行周期可以通过`hz 10`调整，单位为秒。

#### 事件的调度与执行

服务器需要决定何时处理文件事件，何时处理时间事件，处理多久。事件的调度与执行由ae.c/aeProcessEvents函数实现， 伪代码如下：

```python
def aeProcessEvents() :
    # 获取到的时间里当前时间最近的时间事件
    # 计算最接近的时间事件距离达到还有多少毫秒
    # 如果事件已经到达，值可能为负数，此时将其设置为0
    # 根据值，创建timeval结构
    # 阻塞等待文件事件产生，最大阻塞时间传入的timeval结构决定
    # 如果值为0，aeApiPoll调用之后马上执行，不阻塞
    # 处理所有已产生的文件事件
    # 处理所有已产生的时间事件
```

下来观察一下Redis的主函数伪代码：

```python
def main() : 
    # 初始化服务器
    init_server()
    # 循环处理事件，直到服务器关闭
    while server_is_not_shutdown():
        aeProcessEvents()
    # 服务器关闭，执行清理操作
    clean_server()
```

从事件处理角度，服务器的运行流程如下：

![image-20210602124532357](https://gitee.com/tobing/imagebed/raw/master/image-20210602124532357.png)

下面展示了实践的调度和执行规则：

1. aeApiPoll函数的最大阻塞时间由到达时间最接近当前时间的时间事件决定，这个方法既可以避免服务器对时间事件进行频繁轮询，也可以确保aeApiPoll函数不会阻塞过长时间。
2. 因为文件事件具有随机性，如果等待并处理完一次文件事件之后，仍未有任何时间事件到达，那么服务器会再次等待并处理文件事件。随着文件事件不断执行，时间会组件向时间事件设置的到达时间逼近，并最终来到达到时间，这是服务器可以开始处理到达的时间事件
3. 对文件事件和时间事件的处理都是同步、有序、原子的，服务器不会中途中断事件处理，也不会对事件抢占。因此无论是时间事件还是文件事件都会尽可能减少程序阻塞时间，并在需要是主动让出执行权，从而降低造成饥饿的可能性。
4. 因为时间事件在文件事件之后执行，并且事件之间不会抢占，因此时间事件实际处理时间通常会比设定的到达时间晚一点。

## 五、客户端

#### 概述

Redis服务器是典型的一对多服务器程序：一个服务器可以与多个客户端建立网络连接，每个客户端可以向服务器发送命令请求，而服务器接收并处理客户端发送的命令请求，并向客户端返回命令回复。

通过使用IO多路复用技术，Redis服务器使用单线程单进程的方式来处理命令请求，并与多个客户端进行网络通信。

对于每个服务器进行连接的客户端，服务器都为这些客户端建立一个对应的redisClient结果，这个结构保存了客户端当前的状态信息，以及执行相关功能需要的数据结构，其中包括：

+ 客户端的套接字描述符
+ 客户端的名字
+ 客户端的标志值
+ 指向客户端正在使用的数据库指针，以及数据库的号码
+ 客户端当前要执行的命令、命令的参数、命令参数的个数，以及指向命令实现函数的指针
+ 客户端的输入缓冲区和输出缓冲区
+ 客户端的复制状态信息，以及进行复制需要的数据结构
+ 客户端执行的BRPOP、BLPOP等列表阻塞命令时使用的数据结构
+ 客户端的事务状态，以及执行WATCH命令时用到的数据结构
+ 客户端会执行发布于订阅功能时用到的数据结构
+ 客户端的身份验证标志
+ 客户端的创建时间，客户端和服务器最后一次通信时间，以及客户端输出缓冲区大小超出软性限制的时间

Redis服务器用clients链表来保存了所有与服务器连接的客户端的状态结构，对客户端执行批量操作或者查找某个指定的客户端，都可以通过遍历clients链表完成。

![image-20210602154510319](https://gitee.com/tobing/imagebed/raw/master/image-20210602154510319.png)

#### 客户端属性

客户端状态包含的属性可以分为两类：

+ 比较通用属性：这些属性与特定功能相关，无论客户端执行的是什么工作，都需要用到这些属性。
+ 特定功能属性：如操作数据库时用到的db属性和dictid属性，执行事务时需要用到的mstate属性，以及执行WATCH命令需要用到的watched_keys属性等等。

**套接字描述符**

客户端状态的fd属性记录了客户端正在使用的套接字描述符。

根据客户端类型不同，fd属性的值可以是-1或者大于-1的整数。

+ 伪客户端的fd属性的值为-1：伪客户端处理的命令请求来源于AOF文件或Lua脚本，而不是网络，所以这种客户端不需要套接字连接，因此不需记录套接字。
+ 普通客户端的fd属性的值为大于-1的整数：普通客户端使用套接字与服务器进行通信，所以服务器会用fd属性来记录客户端套接字的描述符。合法的的套接字描述符不是-1，普通客户端套接字描述符必须是大于-1的整数。

**名字**

默认情况下，连接到服务器的客户端没有名字。

可以使用CLIENT setname命令可以为客户端设置一个名字，让客户端的身份变得更加清晰。

客户端的名字记录在客户端状态的name属性里面，如果客户端没有为自己设置名字，则那么属性指向NULL；如果设置了名字，将执行一个保存了客户端名字的字符串对象。

![image-20210602155656779](https://gitee.com/tobing/imagebed/raw/master/image-20210602155656779.png)

**标志**

客户端的标志属性flags记录了客户端的角色，以及客户端目前所处的状态。

flags属性的值可以是单个标志，也可以是多个标志的二进制或。

```bash
# 单个标志
flags = <flag>
# 多个标志
flags = <flag1> | <flag2> | ...
```

每个标志使用一个常量表示，一部分标志记录了客户端的角色：

+ 主从复制时，主服务器会成为从服务器的客户端，而从服务器也会成为主服务器的客户端。REDIS_MASTER标志表示客户端代表的是一个主服务器，REDIS_SLAVE标志表示客户端代表的是一个从服务器。
+ REDIS_PRE_PSYNC标志表示客户端代表的是一个版本低于2.8的从服务器，主服务器不能使用PSYNC命令与这个从服务器进行同步。
+ REDIS_LUA_CLIENT标志表示客户端是专门用来处理Lua脚本里包含Redis命令的伪命令客户端。

另一部分标志记录了客户端目前所处的状态：

+ REDIS_MONITOR，表示客户正在执行的MONITOR命令
+ REDIS_UNIX_SOCKET，标志表示服务器使用UNIX套接字来连接客户端
+ REDIS_BLOCKED，表示客户端正在被BRPOP或BLPOP等命令阻塞
+ REDIS_UNBLOCKED，表示客户端已经能够REDIS_BLOCKED标志表示的阻塞状态，不再阻塞。
+ REDIS_MULTI，表示客户端正在执行事务
+ REDIS_DIRTY_CAS，表示事务使用WATCH命令监视的数据库键已经被修改
+ REDIS_DIRTY_EXEC，表示事务在命令入队时出现了错误，以上两个标志都表示事务的安全性已经被破坏，只要任意一个被打开，EXEC命令必然会执行失败。
+ REDIS_CLOSE_ASAP，表示客户端的输出缓冲区大小超出了服务器允许的范围，服务器会在下一次执行serverCron函数时关闭这个客户端，以免服务器在下一次执行serverCron函数时关闭这个客户端，以免服务器的稳定性受到这个客户端的影响。
+ REDIS_CLOSE_AFTER_REPLY，表示有用户对这个客户端执行了CLIENT KILL命令，或客户端发送了错误的请求，服务器会将客户端堆积在输出缓冲区中的所有内容发送个客户端，然后关闭客户端。
+ REDIS_ASKING，表示客户端集群节点方了ASKING命令。
+ REDIS_FORCE_AOF，标志强制服务器将当前执行的命令写入到AOF文件里面。
+ REDIS_FORCE_REPL，标志强制主服务器将当前执行的命令复制到所有从服务器。

**`PUBSUB` 命令和 `SCRIPT LOAD` 命令的特殊性**

通常情况下， Redis 只会将那些对数据库进行了修改的命令写入到 AOF 文件， 并复制到各个从服务器： 如果一个命令没有对数据库进行任何修改， 那么它就会被认为是只读命令， 这个命令不会被写入到 AOF 文件， 也不会被复制到从服务器。

以上规则适用于绝大部分 Redis 命令， 但 PUBSUB 命令和 SCRIPT_LOAD 命令是其中的例外。

PUBSUB 命令虽然没有修改数据库， 但 PUBSUB 命令向频道的所有订阅者发送消息这一行为带有副作用， 接收到消息的所有客户端的状态都会因为这个命令而改变。 因此， 服务器需要使用 `REDIS_FORCE_AOF` 标志， 强制将这个命令写入 AOF 文件， 这样在将来载入 AOF 文件时， 服务器就可以再次执行相同的 PUBSUB 命令， 并产生相同的副作用。

SCRIPT_LOAD 命令的情况与 PUBSUB 命令类似： 虽然 SCRIPT_LOAD 命令没有修改数据库， 但它修改了服务器状态， 所以它是一个带有副作用的命令， 服务器需要使用 `REDIS_FORCE_AOF` 标志， 强制将这个命令写入 AOF 文件， 使得将来在载入 AOF 文件时， 服务器可以产生相同的副作用。

另外， 为了让主服务器和从服务器都可以正确地载入 SCRIPT_LOAD 命令指定的脚本， 服务器需要使用 `REDIS_FORCE_REPL` 标志， 强制将 SCRIPT_LOAD 命令复制给所有从服务器。

客户端状态的输入缓存去用于保存客户端发送的命令请求，输入缓冲区的大小会根据输入的内容动态缩小或增大，但它的大小不会超过1GB，否则服务器将关闭这个客户端。

![image-20210602194840476](https://gitee.com/tobing/imagebed/raw/master/image-20210602194840476.png)

服务器将客户端发送的命令请求保存到客户端状态的querybuf属性之后，服务器将对命令请求的内容进行分析，并将得出的命令参数自己命令参数的个数分别保存客户端状态的argv属性和argc属性中。

![image-20210602195041124](https://gitee.com/tobing/imagebed/raw/master/image-20210602195041124.png)

+ argv属性是有个数组，数组中的每一项是有个字符串对象，其中argv[0]是要执行的命令，而之后的其他项则是传命令的参数。
+ argc属性是argv数组的长度。

当服务器从协议内容中分析并得出argv属性和argc属性的值之后，服务器将根据项argv[0]的值，在命令表中查找命令对应的命令实现函数。

命令表是一个字典，键是SDS结构，保存了命令的名字，字典的值对应一个redisCommand结构，保存了命令的实现函数、命令的标志、命令应该给的参数个数、命令总的执行次数和总消耗时长统计信息。

![image-20210602195513685](https://gitee.com/tobing/imagebed/raw/master/image-20210602195513685.png)

上图展示了服务器根据argv[0]项到命令表中查找指定命令的过程。

执行命令得到的恢复会被保存在客户端状态的输出缓冲区中，每个客户端都有两个输出缓存去可用，一个输出缓存区大小固定，另一个缓冲区的大小可变：

+ 固定大小的缓存区用来保存长度较小的回复，如OK、简短的字符串值、整数值、回复值等等
+ 可变缓冲区用来保存长度较大的恢复，如一个非常长的字符串，一个由很多项组成的列表，一个包含了很多元素的集合等等。

固定缓冲区有两部分组成：

+ buf数组，长度为REDIS_REPLY_CHUNK_BYTES，默认为16*1024
+ bufpos，buf数组长度

如果buf数组用完或者回复数据太长放不下，就会放到可变缓冲区中，可变缓冲区有链表和一个多字符串对象组成：

![image-20210602200124973](https://gitee.com/tobing/imagebed/raw/master/image-20210602200124973.png)

客户端的authenticated属性记录客户端是否通过了身份验证，0表示不通过，1表示通过。因此当authenticated=0时，处理AUTH命令，其他命令都会被服务器拒绝。

除此之外，客户端还有几个与时间相关的属性：

+ ctime，记录了创建客户端的时间，这个时间可以用来计算客户端与服务器以及连接多少秒。
+ lastinteraction，记录了客户端与服务器最后一次互动，可以用来计算客户端的空转时间。
+ obuf_soft_limit_reached_time，记录了输出缓冲区第一次到达软性限制的时间

#### 客户端的创建与关闭

通过网络连接到服务器的普通客户端在使用connect函数连服务器时，服务器会调用连接事件处理器，为客户的创建相应的客户端状态，并将新客户端的状态添加到服务器状态的clients链表中。

一个普通客户端可以分为多种原因被关闭：

+ 如果客户端进程退出或被杀死，客户端与服务器之间的网络连接会被关闭，从而造成客户端被关闭
+ 如果客户端向服务器发送了带有不符合协议格式的命令请求，这个客户端也会被服务器关闭
+ 如果客户端称为了CLIENT KILL的目标，那么它也会被关闭
+ 如果服务器设置了timeout，在客户端空转时间到达时，客户端会被关闭。（主从模式下有例外）
+ 如果客户端发送的命令请求大小超过了输入缓冲区的限制大小（默认1GB）那么整个客户端也会被服务器关闭
+ 如果发送给客户端的命令回复大小超过了输出缓冲区的限制大小，那么整个客户端会被服务器关闭
  + 硬性限制：如果输出缓冲区大小超过了硬性限制，客户端会被立即关闭
  + 软性限制：如果输出缓冲区大小超过了软性限制但没超过硬性要求，服务端会记录超过软性限制的时间，如果持续特定时间会关闭客户端。

服务器在初始化时创建负责执行Lua脚本中包含的Redis命令的伪客户端，并将这个伪客户端关联在服务器状态结构的lua_client属性中。lua_client伪客户端在服务器运行的整个生命周期一直存在。

服务器在载入AOF文件时，会创建用于执行AOF文件包含的Redis命令的伪客户端，并在载入完成之后，关闭这个伪客户端。

## 六、服务器

#### 概述

Redis服务器负责与多个客户端建立网络连接，处理客户端发送的命令请求，在数据库中保存客户端执行命令产生的数据，并通过资源管理来维持服务器自身的运转。

#### 命令请求的执行过程

一个命令请求从发送到获得回复的过程中，客户端和服务器需要完成一系列操作。以`SET KEY VALUE`命令为例，需要执行以下流程：

1. 客户端向服务器发送命令请求`SET KEY VALUE`
2. 服务器接收并处理客户端发送的命令请求，在数据库中进行设置操作，并产生回复OK
3. 服务器将命令回复OK发送个客户端
4. 客户端接收服务器返回的命令回复OK，并将这个回复打印给用户观看

**发送命令请求**

客户端会将用户输入命令的请求转换成协议格式，然后连接到服务器的套接字，将协议格式命令请求发送个服务器。

![image-20210602204630842](https://gitee.com/tobing/imagebed/raw/master/image-20210602204630842.png)

**读取命令请求**

客户端与服务器之间的连接套接字因为客户端的写入而变得可读时，服务器将会调用命令请求处理器执行以下操作：

1. 读取套接字中协议格式的命令请求，并将其保存到客户端状态的输入缓冲区中
2. 对输入缓冲区的命令就行解析，提前命令请求的命令参数，以及参数个数，粉笔将参数和参数格式保存到客户端状态的argv属性和argc属性中
3. 调用命令执行器，执行客户端指定的命令

**命令执行器：查找命令**

命令执行器执行客户端指定命令时，第一步需要根据客户端状态argv[0]参数，在命令表中国查找指定的命令，并将找到的命令保存到客户端状态的cmd属性中。

命令表是有个字典，子弹的键是命令，值是一个redisCommand结构。如对于SET命令，实现函数为setCommand；对于GET命令，实现函数为getCommand。

![image-20210602205342881](https://gitee.com/tobing/imagebed/raw/master/image-20210602205342881.png)

**命令执行器：执行预处理**

命令执行器在查找命令之后，可以获取命令的是实现函数、参数、参数个数等信息，但在真正执行前还需要执行一些预备操作来确保命令正确，顺利执行。包括：

+ 检查客户端状态的cmd指针是否执行NULL，为NULL说明用户输入的命令找不到对应的命令实现，返回一个错误
+ 检查客户端状态的cmd指针执行的redisComman结构的arity属性，检查给定的参数个数是否正确，不正确将返回一个错误
+ 检查客户端是否以及通过身份验证，未通过则需要通过AUTH命令，执行其他指令会返回一个错误
+ 如果服务器开启了maxmemory，在执行命令前会先检查服务器的内存占用，有需要是进行内存回收，确保命令顺利执行，回收失败则返回一个错误
+ 如果服务器上一次执行BGSAVE命令储存，并且服务器开启了stop-writes-on-bgsave-error功能，且是一个写命令，服务器将拒绝执行，返回错误
+ 如果客户端正处于用SUBSCRIBE命令订阅频道，或这是用PSUBSCRIBE命令订阅模式，那么服务器只会执行客户端发来的SUBSCRIBE、PSUBSRCIBE、UNSUBSCRIBE、PUNSUBSCRIBE四个命令，其他命令都会被其他服务器拒绝。
+ 如果服务器正在进行数据载入，那么客户端发送的命令必须带有1标识才会被服务器中，其他命令都会被服务器拒绝
+ 如果服务器以为执行Lua脚本而超时并进入阻塞状态，那么服务器之后执行SHUTDOWN nosave命令和SCRIPT KILL命令，其他命令都会被服务器拒绝。
+ 如果客户端正在执行事务，那么 服务器之后执行客户端发来的EXEC、DISCARD、MULTI、WATCH四个命令，其他命令放到事务队列中
+ 如果服务器开起来监视器功能，那么服务器将要执行的命令和参数发送给监视器。

命令执行器：调用命令的实现函数

服务器将解析的命令保存在客户端状态cmd属性，将命令的参数和参数个数分别保存在argv和argc。当服务器决定执行一个命令，只需执行下面语句：

```c
client->cmd->proc(client);
```

![image-20210602211301648](https://gitee.com/tobing/imagebed/raw/master/image-20210602211301648.png)

被调用的命令实现函数执行并产生相应的命令回复，这些回复会被保存到客户端状态的输出缓冲区汇总，之后实现函数会为客户端套接字关联回复处理器，将命令返回给客户端。

**命令执行：后继工作**

命令执行完之后，服务器还会执行一些工作：

+ 如果开启了慢查询日志，那么慢查询日志模块会检查是否需要为刚执行完的命令添加一行新的慢查询日志
+ 根据执行命令的耗时，更新被自行命令的redisCommand的统计信息
+ 如果服务器开启了AOF持久化功能，AOF持久化模块将会记录执行的命令
+ 如果存在从服务器，那么服务器会将执行的命令传递所有从服务器

**命令回复发送给客户端**

命令被服务器执行之后，需要将结果保存到客户端输出缓冲区，并为客户端套接字关联命令回复处理器，当客户端套接字可写，服务器会执行命令回复处理器，将保存在客户端输出缓冲区的内容回复发送给客户端。

当客户端申冬奥协议格式的命令，将其转换为可读格式，并输出打印。

![image-20210602212533435](https://gitee.com/tobing/imagebed/raw/master/image-20210602212533435.png)

#### serverCron函数

默认Redis服务器中的serverCron函数每100ms执行一次，这个函数负责管理服务器资源，保持服务器自身的良好运转。

**更新服务器时间缓存**

Redis服务器很多功能需要获取系统当前时间，而每次获取系统当前时间需要执行一次系统调用，为了减少系统调用，redisServer缓存了unixtime属性和mstime属性。serverCron函数默认以100ms一次的频率更新unixtime和mstime，因此这两个属性的精度不高：

+ 服务器只会在打印日志、更新服务器的LRU时钟、决定是否持久化、计算服务器上线时间这类对时间精确度要求不高的功能
+ 对于为减设置过期时间、添加慢日志查询这种需要高精度时间的功能，服务器会以系统调用获取精确的时间

**更新LRU时钟**

redisServer中的lruclock属性保存了服务器的LRU时钟，和unixtime、mstime类似都是服务器缓存时间的一种。

每个Redis对象都会有一个lru属性，这个属性保存了对象最后一次被命令访问的时间。当服务器需要计算数据库键的空转时间时，会用服务器lruclock记录的时间减去对象lru属性记录的时间。

**更新服务器每秒执行的命令次数**

serverCron函数中的会以100ms一次频率，抽样计算机服务器最近1s内处理的命令请求数。

**更新服务器内存峰值记录**

redisServer的stat_peak_memory属性记录了服务器的内存峰值大小，每次serverCron执行，会查看服务器当前使用的内存数量，如果数值比stat_peak_memory大，会更新stat_peak_memory。【注意是峰值】

**处理sigterm信号**

启动服务器时，Redis会为服务器进程的SIGTERM信号关联处理器sigtermHandler，处理器在收到SIGTERM信号，到打开服务器状态showdown_asap标识。每次serverCron函数执行会检查该标志来判断是否关闭服务器。

**管理客户端资源**

serverCron函数每次执行会调用clientsCron函数，clientCron函数会对一定数量的客户端进行以下检查：

+ 如果客户端与服务器之间已经超时，那么程序释放该连接
+ 如果客户端在上一次执行命令请求之后，输入缓冲区的大小超过了一定长度，那么程序会释放客户端当前的输入缓冲区，并重新创建一个默认大小的输入缓冲区来防止客户端输入缓冲区消耗太多内存

**管理数据库资源**

serverCron函数内存执行会调用databasesCron函数，这个函数会对服务器中的一部分数据库进行检查，删除过期键，有需要时是对字典执行收缩操作。

**执行被延迟的BGREWRITEAOF**

执行BGSAVE命令期间，如果客户端向服务器发送BGREWRITEAOF命令，BGREWRITEAOF命令会被延后到BGSAVE之后。服务器通过aof_rewrite_scheduled标识是否延迟。

serverCron函数每次执行检查BGREWRITEAOF和BGSAVE是否执行，如果不执行，判断aof_rewrite_scheduled=1就会执行BGREWRITEAOF。

**检查持久化操作的运行状态**

服务器通过rdb_child_pid和aof_child_pid记录执行BGSAVE和BGREWRITEAOF命令的子进程ID，这两个属性也可以用于检测BGSAVE和BGREWRITEAOF是否正在执行。

serverCron函数每次执行检查rdb_child_pid和aof_child_pid，只要其中一个部位-1，就会执行一次wait3函数，检测子进程是否有信号发来服务器进程：

+ 有，表名RDB文件或AOF文件已经生成，服务器会执行后继操作，如用心RDB文件替换现有RDB文件，或用重写的AOF替换现有AOF文件
+ 无，表示持久化未完成，不执行其他操作

如果都为-1，表示没有持久化，需要执行以下检查：

+ 查看是否有BGREWRITEAOF延迟，有开始一场新的BGREWRITEAOF
+ 检查服务器自定保存条件是否满足，满足会在服务器没执行持久化操作时开启一次新的SAVE操作
+ 检查服务器设置的AOF重写条件是否满足，满足开启新一轮BGREWRITEAOF

![image-20210602215518875](https://gitee.com/tobing/imagebed/raw/master/image-20210602215518875.png)

**将AOF缓冲区中的内容写入AOF文件**

如果开启AOF，serverCron将AOF缓冲区内容写到AOF文件

**关闭异步客户端**

serverCron会关闭输出缓冲区大小超出限制的客户端

**增加cronloops计数器的值**

cronloops记录了serverCron函数执行次数，因此函数每次执行都要更新

#### 初始化服务器

初始化服务器需要经历以下操作：初始化服务器状态，接受用户指定的配置，创建对应的数据结构和网络连接等。

**初始化服务器状态结构**

初始化服务器第一步是创建一个redisServer类型的实例变量server作为服务器的状态，并为结构中的各个属性赋默认值。

server变量初始化有initServerConfig函数实现，主要完成以下工作：

+ 设置服务器运行ID
+ 设置服务器默认运行频率
+ 设置服务器默认配置文件路径
+ 设置服务器运行架构
+ 设置服务器默认端口号
+ 设置服务器默认RDB持久化条件和AOF持久化条件
+ 初始化服务器的LRU时钟
+ 创建命令表

注意initServerConfig函数没有创建服务器状态的其他数据结构。函数执行完，服务器会载入配置选项。

**载入配置选项**

启动服务器，服务器会根据用户配置的参数修改服务器默认参数

**初始化服务器数据结构**

initServerConfig函数执行完毕，只创建了命令表，还需要其他数据结构，如：

+ server.clients链表，记录与服务器连接的客户端状态结构
+ server.db数组，包含服务器的所有数据库
+ server.pubsub_channels字典，保存频道订阅信息
+ server.pubsub_patterns字典，保存模式订阅信息
+ server.lua，用于执行Lua脚本的Lua环境
+ server.slowlog，保存慢查询日志

上述结构由调用initServer函数完成内存分配等操作。

之所以区分initServerConfig和initServer是因为initServer需要initServerConfig阶段获取的参数信息。

除此之外，initServer函数还会进行一些重要设置：

+ 为其设置进程信号处理器
+ 创建共享对象，包含服务器常用的对象，如“OK”、“ERR”、1-10000字符串对象等
+ 打开服务器监听端口，为监听套接字关联连接应答事件处理器，等待服务器正是运行时接收客户端的连接
+ 为serverCron函数创建时间事件
+ 打开现有AOF文件，如果无，则创建新的文件（开启AOF）
+ 初始化服务器的后台IO模块

initServer函数执行完，服务器以ASCII字符在日志中打印出Redis图标，以及Redis版本号信息。

```bash
[2708] 02 Jun 22:12:04.642 # Warning: no config file specified, using the default config. In order to specify a config file use D:\Develop\redis2.8win32\redis2.8win32\redis-server.exe /path/to/redis.conf
[2708] 02 Jun 22:12:04.648 # Warning: 32 bit instance detected but no memory limit set. Setting 3 GB maxmemory limit with 'noeviction' policy now.
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 2.8.19 (00000000/0) 32 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in stand alone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 2708
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[2708] 02 Jun 22:12:04.649 # Server started, Redis version 2.8.19
[2708] 02 Jun 22:12:04.649 * DB loaded from disk: 0.001 seconds
[2708] 02 Jun 22:12:04.649 * The server is now ready to accept connections on port 6379
```

**还原数据库状态**

完成对服务器状态初始化，服务器需要载入RDB文件或AOF文件，并根据文件记录还原服务器的数据库状态。

**执行事件循环**

初始化最后一步，服务器将会打印以下日志

```bash
[2708] 02 Jun 22:12:04.649 * The server is now ready to accept connections on port 6379
```


























