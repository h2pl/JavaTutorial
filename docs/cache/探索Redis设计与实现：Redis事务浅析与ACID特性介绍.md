# Table of Contents

* [事务](#事务)
  * [用法](#用法)
  * [事务中的错误](#事务中的错误)
  * [为什么 Redis 不支持回滚（roll back）](#为什么-redis-不支持回滚（roll-back）)
  * [放弃事务](#放弃事务)
  * [使用 check-and-set 操作实现乐观锁](#使用-check-and-set-操作实现乐观锁)
  * [了解`WATCH`](#了解`watch`)
    * [使用 WATCH 实现 ZPOP](#使用-watch-实现-zpop)
  * [Redis 脚本和事务](#redis-脚本和事务)
* [redis事务的ACID特性](#redis事务的acid特性)


[toc]

本文转自互联网
本文将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，Redis基本的使用方法，Redis的基本数据结构，以及一些进阶的使用方法，同时也需要进一步了解Redis的底层数据结构，再接着，还会带来Redis主从复制、集群、分布式锁等方面的相关内容，以及作为缓存的一些使用方法和注意事项，以便让你更完整地了解整个Redis相关的技术体系，形成自己的知识框架。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->


# 事务

[MULTI](http://www.redis.cn/commands/multi.html)、[EXEC](http://www.redis.cn/commands/exec.html)、[DISCARD](http://www.redis.cn/commands/discard.html)和[WATCH](http://www.redis.cn/commands/watch.html)是 Redis 事务相关的命令。事务可以一次执行多个命令， 并且带有以下两个重要的保证：

*   事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。

*   事务是一个原子操作：事务中的命令要么全部被执行，要么全部都不执行。

[EXEC](http://www.redis.cn/commands/exec.html)命令负责触发并执行事务中的所有命令：

*   如果客户端在使用[MULTI](http://www.redis.cn/commands/multi.html)开启了一个事务之后，却因为断线而没有成功执行[EXEC](http://www.redis.cn/commands/exec.html)，那么事务中的所有命令都不会被执行。
*   另一方面，如果客户端成功在开启事务之后执行[EXEC](http://www.redis.cn/commands/exec.html)，那么事务中的所有命令都会被执行。

当使用 AOF 方式做持久化的时候， Redis 会使用单个 write(2) 命令将事务写入到磁盘中。

然而，如果 Redis 服务器因为某些原因被管理员杀死，或者遇上某种硬件故障，那么可能只有部分事务命令会被成功写入到磁盘中。

如果 Redis 在重新启动时发现 AOF 文件出了这样的问题，那么它会退出，并汇报一个错误。

使用`redis-check-aof`程序可以修复这一问题：它会移除 AOF 文件中不完整事务的信息，确保服务器可以顺利启动。

从 2.2 版本开始，Redis 还可以通过乐观锁（optimistic lock）实现 CAS （check-and-set）操作，具体信息请参考文档的后半部分。

## 用法

[MULTI](http://www.redis.cn/commands/multi.html)命令用于开启一个事务，它总是返回`OK`。[MULTI](http://www.redis.cn/commands/multi.html)执行之后， 客户端可以继续向服务器发送任意多条命令， 这些命令不会立即被执行， 而是被放到一个队列中， 当[EXEC](http://www.redis.cn/commands/exec.html)命令被调用时， 所有队列中的命令才会被执行。

另一方面， 通过调用[DISCARD](http://www.redis.cn/commands/discard.html)， 客户端可以清空事务队列， 并放弃执行事务。

以下是一个事务例子， 它原子地增加了`foo`和`bar`两个键的值：



```
> MULTIOK> INCR fooQUEUED> INCR barQUEUED> EXEC1) (integer) 12) (integer) 1
```



[EXEC](http://www.redis.cn/commands/exec.html)命令的回复是一个数组， 数组中的每个元素都是执行事务中的命令所产生的回复。 其中， 回复元素的先后顺序和命令发送的先后顺序一致。

当客户端处于事务状态时， 所有传入的命令都会返回一个内容为`QUEUED`的状态回复（status reply）， 这些被入队的命令将在 EXEC 命令被调用时执行。

## 事务中的错误

使用事务时可能会遇上以下两种错误：

*   事务在执行[EXEC](http://www.redis.cn/commands/exec.html)之前，入队的命令可能会出错。比如说，命令可能会产生语法错误（参数数量错误，参数名错误，等等），或者其他更严重的错误，比如内存不足（如果服务器使用`maxmemory`设置了最大内存限制的话）。
*   命令可能在[EXEC](http://www.redis.cn/commands/exec.html)调用之后失败。举个例子，事务中的命令可能处理了错误类型的键，比如将列表命令用在了字符串键上面，诸如此类。

对于发生在[EXEC](http://www.redis.cn/commands/exec.html)执行之前的错误，客户端以前的做法是检查命令入队所得的返回值：如果命令入队时返回`QUEUED`，那么入队成功；否则，就是入队失败。如果有命令在入队时失败，那么大部分客户端都会停止并取消这个事务。

不过，从 Redis 2.6.5 开始，服务器会对命令入队失败的情况进行记录，并在客户端调用[EXEC](http://www.redis.cn/commands/exec.html)命令时，拒绝执行并自动放弃这个事务。

在 Redis 2.6.5 以前， Redis 只执行事务中那些入队成功的命令，而忽略那些入队失败的命令。 而新的处理方式则使得在流水线（pipeline）中包含事务变得简单，因为发送事务和读取事务的回复都只需要和服务器进行一次通讯。

至于那些在[EXEC](http://www.redis.cn/commands/exec.html)命令执行之后所产生的错误， 并没有对它们进行特别处理： 即使事务中有某个/某些命令在执行时产生了错误， 事务中的其他命令仍然会继续执行。

从协议的角度来看这个问题，会更容易理解一些。 以下例子中，[LPOP](http://www.redis.cn/commands/lpop.html)命令的执行将出错， 尽管调用它的语法是正确的：



```
Trying 127.0.0.1...Connected to localhost.Escape character is '^]'.MULTI+OKSET a 3abc+QUEUEDLPOP a+QUEUEDEXEC*2+OK-ERR Operation against a key holding the wrong kind of value
```



[EXEC](http://www.redis.cn/commands/exec.html)返回两条[bulk-string-reply](http://www.redis.cn/topics/protocol.html#bulk-string-reply)： 第一条是`OK`，而第二条是`-ERR`。 至于怎样用合适的方法来表示事务中的错误， 则是由客户端自己决定的。

最重要的是记住这样一条， 即使事务中有某条/某些命令执行失败了， 事务队列中的其他命令仍然会继续执行 —— Redis 不会停止执行事务中的命令。

以下例子展示的是另一种情况， 当命令在入队时产生错误， 错误会立即被返回给客户端：



```
MULTI+OKINCR a b c-ERR wrong number of arguments for 'incr' command
```



因为调用[INCR](http://www.redis.cn/commands/incr.html)命令的参数格式不正确， 所以这个[INCR](http://www.redis.cn/commands/incr.html)命令入队失败。

## 为什么 Redis 不支持回滚（roll back）

如果你有使用关系式数据库的经验， 那么 “Redis 在事务失败时不进行回滚，而是继续执行余下的命令”这种做法可能会让你觉得有点奇怪。

以下是这种做法的优点：

*   Redis 命令只会因为错误的语法而失败（并且这些问题不能在入队时发现），或是命令用在了错误类型的键上面：这也就是说，从实用性的角度来说，失败的命令是由编程错误造成的，而这些错误应该在开发的过程中被发现，而不应该出现在生产环境中。
*   因为不需要对回滚进行支持，所以 Redis 的内部可以保持简单且快速。

有种观点认为 Redis 处理事务的做法会产生 bug ， 然而需要注意的是， 在通常情况下， 回滚并不能解决编程错误带来的问题。 举个例子， 如果你本来想通过[INCR](http://www.redis.cn/commands/incr.html)命令将键的值加上 1 ， 却不小心加上了 2 ， 又或者对错误类型的键执行了[INCR](http://www.redis.cn/commands/incr.html)， 回滚是没有办法处理这些情况的。

## 放弃事务

当执行[DISCARD](http://www.redis.cn/commands/discard.html)命令时， 事务会被放弃， 事务队列会被清空， 并且客户端会从事务状态中退出：



```
> SET foo 1OK> MULTIOK> INCR fooQUEUED> DISCARDOK> GET foo"1"
```



## 使用 check-and-set 操作实现乐观锁

[WATCH](http://www.redis.cn/commands/watch.html)命令可以为 Redis 事务提供 check-and-set （CAS）行为。

被[WATCH](http://www.redis.cn/commands/watch.html)的键会被监视，并会发觉这些键是否被改动过了。 如果有至少一个被监视的键在[EXEC](http://www.redis.cn/commands/exec.html)执行之前被修改了， 那么整个事务都会被取消，[EXEC](http://www.redis.cn/commands/exec.html)返回[nil-reply](http://www.redis.cn/topics/protocol.html#nil-reply)来表示事务已经失败。

举个例子， 假设我们需要原子性地为某个值进行增 1 操作（假设[INCR](http://www.redis.cn/commands/incr.html)不存在）。

首先我们可能会这样做：



```
val = GET mykeyval = val + 1SET mykey $val
```



上面的这个实现在只有一个客户端的时候可以执行得很好。 但是， 当多个客户端同时对同一个键进行这样的操作时， 就会产生竞争条件。举个例子， 如果客户端 A 和 B 都读取了键原来的值， 比如 10 ， 那么两个客户端都会将键的值设为 11 ， 但正确的结果应该是 12 才对。

有了[WATCH](http://www.redis.cn/commands/watch.html)， 我们就可以轻松地解决这类问题了：



```
WATCH mykeyval = GET mykeyval = val + 1MULTISET mykey $valEXEC
```



使用上面的代码， 如果在[WATCH](http://www.redis.cn/commands/watch.html)执行之后，[EXEC](http://www.redis.cn/commands/exec.html)执行之前， 有其他客户端修改了`mykey`的值， 那么当前客户端的事务就会失败。 程序需要做的， 就是不断重试这个操作， 直到没有发生碰撞为止。

这种形式的锁被称作乐观锁， 它是一种非常强大的锁机制。 并且因为大多数情况下， 不同的客户端会访问不同的键， 碰撞的情况一般都很少， 所以通常并不需要进行重试。

## 了解`WATCH`

[WATCH](http://www.redis.cn/commands/watch.html)使得[EXEC](http://www.redis.cn/commands/exec.html)命令需要有条件地执行： 事务只能在所有被监视键都没有被修改的前提下执行， 如果这个前提不能满足的话，事务就不会被执行。[了解更多->](http://code.google.com/p/redis/issues/detail?id=270)

[WATCH](http://www.redis.cn/commands/watch.html)命令可以被调用多次。 对键的监视从[WATCH](http://www.redis.cn/commands/watch.html)执行之后开始生效， 直到调用[EXEC](http://www.redis.cn/commands/exec.html)为止。

用户还可以在单个[WATCH](http://www.redis.cn/commands/watch.html)命令中监视任意多个键， 就像这样：



```
redis> WATCH key1 key2 key3OK
```



当[EXEC](http://www.redis.cn/commands/exec.html)被调用时， 不管事务是否成功执行， 对所有键的监视都会被取消。

另外， 当客户端断开连接时， 该客户端对键的监视也会被取消。

使用无参数的[UNWATCH](http://www.redis.cn/commands/unwatch.html)命令可以手动取消对所有键的监视。 对于一些需要改动多个键的事务， 有时候程序需要同时对多个键进行加锁， 然后检查这些键的当前值是否符合程序的要求。 当值达不到要求时， 就可以使用[UNWATCH](http://www.redis.cn/commands/unwatch.html)命令来取消目前对键的监视， 中途放弃这个事务， 并等待事务的下次尝试。

### 使用 WATCH 实现 ZPOP

[WATCH](http://www.redis.cn/commands/watch.html)可以用于创建 Redis 没有内置的原子操作。举个例子， 以下代码实现了原创的[ZPOP](http://www.redis.cn/commands/zpop.html)命令， 它可以原子地弹出有序集合中分值（score）最小的元素：



```
WATCH zsetelement = ZRANGE zset 0 0MULTIZREM zset elementEXEC
```



程序只要重复执行这段代码， 直到[EXEC](http://www.redis.cn/commands/exec.html)的返回值不是[nil-reply](http://www.redis.cn/topics/protocol.html#nil-reply)回复即可。

## Redis 脚本和事务

从定义上来说， Redis 中的脚本本身就是一种事务， 所以任何在事务里可以完成的事， 在脚本里面也能完成。 并且一般来说， 使用脚本要来得更简单，并且速度更快。

因为脚本功能是 Redis 2.6 才引入的， 而事务功能则更早之前就存在了， 所以 Redis 才会同时存在两种处理事务的方法。

不过我们并不打算在短时间内就移除事务功能， 因为事务提供了一种即使不使用脚本， 也可以避免竞争条件的方法， 而且事务本身的实现并不复杂。

不过在不远的将来， 可能所有用户都会只使用脚本来实现事务也说不定。 如果真的发生这种情况的话， 那么我们将废弃并最终移除事务功能。











# redis事务的ACID特性

```
在传统的关系型数据库中,尝尝用ACID特质来检测事务功能的可靠性和安全性。
```

在redis中事务总是具有原子性(Atomicity),一致性(Consistency)和隔离性(Isolation),并且当redis运行在某种特定的持久化
模式下,事务也具有耐久性(Durability).

> ①原子性
> 
> ```
> 事务具有原子性指的是,数据库将事务中的多个操作当作一个整体来执行,服务器要么就执行事务中的所有操作,要么就一个操作也不执行。
> ```
> 
> 但是对于redis的事务功能来说,事务队列中的命令要么就全部执行,要么就一个都不执行,因此redis的事务是具有原子性的。我们通常会知道

两种关于redis事务原子性的说法,一种是要么事务都执行,要么都不执行。另外一种说法是redis事务当事务中的命令执行失败后面的命令还
会执行,错误之前的命令不会回滚。其实这个两个说法都是正确的。但是缺一不可。我们接下来具体分析下

```
 我们先看一个可以正确执行的事务例子
```

```
redis > MULTIOK redis > SET username "bugall"QUEUED redis > EXEC1) OK2) "bugall"
```

```
与之相反,我们再来看一个事务执行失败的例子。这个事务因为命令在放入事务队列的时候被服务器拒绝,所以事务中的所有命令都不会执行,因为前面我们有介绍到,redis的事务命令是统一先放到事务队列里,在用户输入EXEC命令的时候再统一执行。但是我们错误的使用"GET"命令,在命令放入事务队列的时候被检测到事务,这时候还没有接收到EXEC命令,所以这个时候不牵扯到回滚的问题,在EXEC的时候发现事务队列里有命令存在错误,所以事务里的命令就全都不执行,这样就达到里事务的原子性,我们看下例子。
```

```
redis > MULTIOK redis > GET(error) ERR wrong number of arguments for 'get' command redis > GET usernameQUEUED redis > EXEC(error) EXECABORT Transaction discarded because of previous errors
```

```
redis的事务和传统的关系型数据库事务的最大区别在于,redis不支持事务的回滚机制,即使事务队列中的某个命令在执行期间出现错误,整个事务也会继续执行下去,直到将事务队列中的所有命令都执行完毕为止,我们看下面的例子
```

```
redis > SET username "bugall"OK redis > MULTIOK redis > SADD member "bugall" "litengfe" "yangyifang"QUEUED redis > RPUSH　username "b" "l" "y" //错误对键username使用列表键命令QUEUED redis > SADD password "123456" "123456" "123456"QUEUED redis > EXEC1) (integer) 32) (error) WRONGTYPE Operation against a key holding the wrong kind of value3) (integer) 3
```

```
redis的作者在十五功能的文档中解释说,不支持事务回滚是因为这种复杂的功能和redis追求的简单高效的设计主旨不符合,并且他认为,redis事务的执行时错误通常都是编程错误造成的,这种错误通常只会出现在开发环境中,而很少会在实际的生产环境中出现,所以他认为没有必要为redis开发事务回滚功能。所以我们在讨论redis事务回滚的时候,一定要区分命令发生错误的时候。
```

> ②一致性
> 
> ```
>     事务具有一致性指的是,如果数据库在执行事务之前是一致的,那么在事务执行之后,无论事务是否执行成功,数据库也应该仍然一致的。    ”一致“指的是数据符合数据库本身的定义和要求,没有包含非法或者无效的错误数据。redis通过谨慎的错误检测和简单的设计来保证事务一致性。
> ```
> 
> ③隔离性
> 
> ```
>     事务的隔离性指的是,即使数据库中有多个事务并发在执行,各个事务之间也不会互相影响,并且在并发状态下执行的事务和串行执行的事务产生的结果完全    相同。    因为redis使用单线程的方式来执行事务(以及事务队列中的命令),并且服务器保证,在执行事务期间不会对事物进行中断,因此,redis的事务总是以串行    的方式运行的,并且事务也总是具有隔离性的
> ```
> 
> ④持久性
> 
> ```
>     事务的耐久性指的是,当一个事务执行完毕时,执行这个事务所得的结果已经被保持到永久存储介质里面。    因为redis事务不过是简单的用队列包裹起来一组redis命令,redis并没有为事务提供任何额外的持久化功能,所以redis事务的耐久性由redis使用的模式    决定    - 当服务器在无持久化的内存模式下运行时,事务不具有耐久性,一旦服务器停机,包括事务数据在内的所有服务器数据都将丢失    - 当服务器在RDB持久化模式下运作的时候,服务器只会在特定的保存条件满足的时候才会执行BGSAVE命令,对数据库进行保存操作,并且异步执行的BGSAVE不    能保证事务数据被第一时间保存到硬盘里面,因此RDB持久化模式下的事务也不具有耐久性    - 当服务器运行在AOF持久化模式下,并且appedfsync的选项的值为always时,程序总会在执行命令之后调用同步函数,将命令数据真正的保存到硬盘里面,因此    这种配置下的事务是具有耐久性的。    - 当服务器运行在AOF持久化模式下,并且appedfsync的选项的值为everysec时,程序会每秒同步一次
> ```







