## 分布式锁 Distribution Lock

分布式锁是传统锁的分布式版本，很多情况下不仅仅是在一台服务器竞争资源，而是在*多台服务器竞争资源*，比如秒杀。

### 构建正确的分布式锁需要考虑什么

基础问题：
- 添加锁，**竞争产生**的时候，是否正确？如果多个客户端同一时刻都可以获得锁，这个锁是错误的
- 添加锁，是否是一个**原子性**操作？若不是一个原子性操作，那么在锁执行的过程中，如果客户端崩溃，锁无法释放会产生死锁
- 延长锁，是否可以正确延长锁（例如**锁被抢占**）？
- 释放锁，在崩溃的情况下，其他进程是否可以正确的处理锁（**不产生死锁**）？
- 释放锁，是否会释放别人的锁？比如超时释放锁的时，**不能释放别人的锁**。

进阶问题：
- 在**集群环境**下，上述问题是否都能解决？

### 算法

- Redis 所有命令都是**原子性**的，如果有很多命令，那可以通过 `EVAL` 编写 lua 执行，执行 lua 脚本是原子性的操作。
- 防止客户端崩溃无法释放锁，需要设置**超时时间**，到时间时锁自动过期。
- 为了保证添加锁的原子性，需要利用 `SET` 指令的 **expire 选项**。
- 延长锁和释放锁，为了确定该客户端还拥有锁，需要在锁上设置所属者的信息——简单的做法是客户端用 uuid 生成唯一 id。操作时校验**锁所属者**。
- 为了保证延长锁和释放锁的原子性，需要用 **EVAL** 执行 lua 脚本。

基础问题方案（详见参考）：

```
    // 加锁，setnx，key 为需要加锁的资源，value 为锁拥有者（随机字符串即可），ttl 为超时时间
    ok, err := c.client.SetNX(key, value, ttl).Result()
    // 刷新锁，KEYS[1] 为需要加锁的资源，ARGV[1] 为锁拥有者，只有锁的当前拥有者是操作该锁的人才能延长
    luaRefresh = redis.NewScript(`if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("pexpire", KEYS[1], ARGV[2]) else return 0 end`)
    // 释放锁，KEYS[1] 为需要加锁的资源，ARGV[1] 为锁拥有者，只有锁的当前拥有者是操作该锁的人才能释放
    luaRelease = redis.NewScript(`if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) else return 0 end`)
```

大多数情况下，并不需要**强一致性**，集群环境下可以按照单机环境简单处理。如果需要强一致性，必会导致**低可用性**。Redis 官方提出了一个 RedLock 算法（详见参考）：
- 需要 n 个 master Redis 节点
- 在 **n 个节点**上进行加锁操作，和单机一致
- 只要有 **n / 2 + 1 节点**加锁成功，并且加锁时间没有超出锁最大有效时间，即可认为成功；原因是，其他节点获取的节点数没有当前节点数多，肯定不能成功加锁
- 加锁失败**及时释放锁**，让其他客户端去竞争锁
- n 个 Redis 节点挂掉 k 个节点，当 k > n / 2 (k 为偶数) 或者 k >= n / 2（k 为奇数），因为强一致性，此时**无法工作**一段时间

进阶问题方案（详见参考）：

```
// Lock locks m. In case it returns an error on failure, you may retry to acquire the lock by calling this method again.
func (m *Mutex) Lock() error {
    // 生成一个随机字符串，表明用户身份
	value, err := m.genValueFunc()
	if err != nil {
		return err
	}

	for i := 0; i < m.tries; i++ {
		if i != 0 {
			time.Sleep(m.delayFunc(i))
		}

		start := time.Now()

        // 对所有客户端进行加锁，传入加锁操作参数，n 为返回成功的 Redis 数量
		n := m.actOnPoolsAsync(func(pool Pool) bool {
			return m.acquire(pool, value)
		})

		now := time.Now()
		until := now.Add(m.expiry - now.Sub(start) - time.Duration(int64(float64(m.expiry)*m.factor)))
        // 半数以上成功，并且时间没有超过最大锁有效期
		if n >= m.quorum && now.Before(until) {
			m.value = value
			m.until = until
			return nil
		}
        // 加锁失败，及时释放，让其他客户端竞争锁
		m.actOnPoolsAsync(func(pool Pool) bool {
			return m.release(pool, value)
		})
	}

	return ErrFailed
}
```

> 如需强一致性的场景，请使用相对专业，用 raft 算法实现的 etcd。

### 参考

- [go-redis 实现单机版分布式锁](https://github.com/bsm/redislock)
- [redis 实现集群版分布式锁](https://github.com/go-redsync/redsync)
- [分布式锁官方文档](https://redis.io/topics/distlock) 

## 频率限制器 Rate Limiter 

频率限制器主要是*限制了某一类资源的访问*，比如针对 TestAPI 接口某个 IP N 段时间可以访问 K 次。

### 算法

- Key 的名为 TestAPI，过期时间为 N，初始值为不存在（nil）
- 检测 Key 的当前值是否大于最大限制值 K
  -  大于等于，返回 0，表示**超出限制**，结束
- INCR Key，表明访问一次
  - 如果 INCR 结果为 1，表明**第一次触发限制**，设置过期时间为 N
- 返回 1，结束

需要保证该算法**原子性**，如果竞争存在，存在这样的情况，两个客户端同时发现未到最大限制值 K - 1，之后同时 `INCR` Key，会超出最大限制值 K（实际值为 K + 1）。

代码如下：

```
package main

import (
	"errors"
	"fmt"
	"github.com/go-redis/redis"
	"reflect"
	"strconv"
	"time"
)

const (
	LuaIPLimiter = `
-- 获取当前值
local cur = redis.call('get', KEYS[1])
-- 如果达到了上限，返回 0
if (cur ~= false and cur >= ARGV[1]) then
return 0 end 
-- 如果第一次触发计数器，需要设置过期时间
if redis.call('incr', KEYS[1]) == 1 then redis.call('pexpire', KEYS[1], ARGV[2]) end
-- 正常返回 1
return 1
`
)

// IP 限制，client 是 Redis 客户端，ip 是客户端 IP，max 是最大次数，ttl 是限制的间隔
func IPLimitation(client *redis.Client, ip string, max int64, ttl time.Duration) (bool, error) {
	// 调用 Eval 脚本，原子化操作
	obj, err := client.Eval(LuaIPLimiter, []string{ip}, []string{strconv.FormatInt(max, 10), strconv.FormatInt(ttl.Milliseconds(), 10)}).Result()
	if err != nil {
		fmt.Println(err)
		return false, err
	}
	if i, ok := obj.(int64); !ok {
		return false, errors.New("convert result to int64 failed, type is " + reflect.TypeOf(obj).String())
	} else {
		if i > 0 {
			return true, nil
		} else {
			return false, nil
		}
	}
}

func main() {
	redisClient := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "foobared",
		DB:       2,
	})
	for i := 1; i <= 10; i++ {
		ok, err := IPLimitation(redisClient, "127.0.0.1", 3, time.Minute)
		if err != nil {
			fmt.Println(err)
			return
		}
		if ok {
			fmt.Println("count", i, "127.0.0.1 can visit")
		} else {
			fmt.Println("count", i, "127.0.0.1 can't visit")
		}
	}
}

```

### 参考

- [go-redis 实现 limiter](https://github.com/ulule/limiter)，该类库的对于代码分层是很好的，不过对于以 Redis 实现的 Store 过于繁琐，不如作者实现

## 计数器 Counter

计数器就是*计算某一个资源的访问次数*，例如某个接口访问了 K 次。

### 算法

很简单，就是 INCR 命令，由于 Redis 命令的**原子性**，使用该命令无需担心竞争条件。

```
  INCR Resource
```

## 权限控制 Authority（节约内存）

权限控制，就是*根据不同的级别控制不同权限*，这里使用 Redis 的 `BITFIELD` 命令设置权限，会**节约很多内存**。

- 对于一个只有 yes 和 no 的权限，表示这个权限我们只需要 `1 bit`。0 表示 no，1 表示 yes。
- 对于有 N 个值的权限，表示该权限只需要 `2 ^ (log N + 1) bit`。例如，有 3 个权限值， 00 表示权限值 1，01 表示权限值 2，...

> 这里用权限控制展现了如何使用 bit 节约内存，其他用法还有很多，比如存用户的一些数据，是否第一次访问 A 功能、第一次访问 B 功能等等，每条数据占一个 bit。
> 在用户量很大的情况下，这样做会非常节约内存，读取速度也非常快。

### 算法

- 初始状态，数据为 0
- 寻找权限 k 的偏移量 idx
- 设置权限偏移量的值，或者读取


下图展现了数据存储的内容：

```
           repor2 = read write
           |
binary: 00 11 00 00
        |
        repo1 = no 
```

代码如下：

```
package main

import (
	"fmt"
	"github.com/go-redis/redis"
	"reflect"
	"strconv"
)

// Repository
const (
	Repo1 = iota
	Repo2
	Repo3
)

// Authority, 2 bits
const (
	No        = 0            // binary: 00
	Read      = 1            // binary: 01
	Write     = 2            // binary: 10
	ReadWrite = Read | Write // binary: 11
)

func SetAuthority(user string, idx, authority int, client *redis.Client) {
	// bitfield 命令请查看相关官方文档，简单的说，这个命令在 idx * 2（u2的2）的位置（#idx），设置了两个无符号字节（u2）的值（authority）
	err := client.Do("bitfield", user, "set", "u2", "#"+strconv.Itoa(idx), authority).Err()
	if err != nil {
		panic(err)
	}
}

func GetAuthority(user string, idx int, client *redis.Client) int {
	// 得到 authority 的值，bitfield 命令请查看相关官方文档，简单的说，这个命令在 idx * 2（u2的2）的位置（#idx），获取了两个无符号字节（u2）的值（authority）
	i, err := client.Do("bitfield", user, "get", "u2", "#"+strconv.Itoa(idx)).Result()
	if err != nil {
		panic(err)
	}
	// 很多强制类型转换，不多解释
	if array, ok := i.([]interface{}); !ok {
		panic("convert to array failed, type is " + reflect.TypeOf(i).String())
	} else {
		if authority, ok := array[0].(int64); !ok {
			panic("convert to int64 failed, type is " + reflect.TypeOf(array[0]).String())
		} else {
			return int(authority)
		}
	}
}

func AuthorityString(i int) string {
	switch i {
	case No:
		return "no"
	case Read:
		return "read"
	case Write:
		return "write"
	case ReadWrite:
		return "read-write"
	default:
		return "unknown"
	}
}

func main() {
	client := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "foobared",
		DB:       2,
	})
	SetAuthority("Piers", Repo1, ReadWrite, client)
	SetAuthority("Piers", Repo2, Read, client)
	SetAuthority("Fiers", Repo3, No, client)
	fmt.Println("Piers repo1 authority:", AuthorityString(GetAuthority("Piers", Repo1, client)))
	fmt.Println("Piers repo2 authority:", AuthorityString(GetAuthority("Piers", Repo2, client)))
	fmt.Println("Fiers repo3 authority:", AuthorityString(GetAuthority("Fiers", Repo3, client)))
}

```

