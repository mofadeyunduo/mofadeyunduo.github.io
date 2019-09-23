## 自动补全 Auto Completion

自动补全，就是*输入部分关键字，补全剩下关键字*。

### 算法

下述算法的基石是，Redis ZSET 首先以 SCORE 排序，如果 SCORE 相同，MEMBER **比较字符串大小**进行排序。
这里算法很多，我只是选取一些实现，之后进行比较，更多可以参见参考。

#### 插入所有前缀 

该算法是 Redis 作者提出的，参考见下。该算法会匹配出**不属于前缀的字符串**（不是很准确），可以当推荐值，也可以过滤掉。

这里的算法不支持热度，搜索的结果**不太理想**，比如我搜 "百"，可能会自动补全 "百变"，而热度最高的是 "百度"，应该第一个显示 "百度"。

空间占用**不会太大**，理论上一个单词需要 N 个存储空间，实际上很多前缀都是重复的，无需重复存储。读取时候由于是 `ZRANGE`，可能读出大量数据产生**性能问题**。

插入

- 对于字符串 str，对于其中**字符（一个中文算一个字符，32 位）**，下标 0...N；初始 i = 0，当 i < N - 1 时执行：
  - `ZADD` 单词表 <str[0, i], 0>
  - i++，继续
- `ZADD` 字符串 str$ 到单词表 
  
补全

- `ZRANK` 需要自动填充的字符串 to_str，得到下表 i
- `ZRANGE` i -1，取出所有字符串，如果字符串**以 $ 结尾**，去掉 $，加入自动补全集合

代码如下：

```
package main

import (
	"fmt"
	"github.com/go-redis/redis"
)

func Save(word, wordsKey string, client *redis.Client) {
	runes := []rune(word)
	_, err := client.Pipelined(func(pp redis.Pipeliner) error {
		for i := 0; i < len(runes)-1; i++ {
			pp.ZAdd(wordsKey, redis.Z{Score: 0, Member: string(runes[:i+1])})
		}
		pp.ZAdd(wordsKey, redis.Z{Score: 0, Member: word + "$"})
		return nil
	})
	if err != nil {
		panic(err)
	}
}

func AutoCompletion(str, wordsKey string, client *redis.Client) []string {
	i, err := client.ZRank(wordsKey, str).Result()
	if err != nil {
		if err == redis.Nil {
			return []string{}
		}
		// fixme
		panic(err)
	}
	ts, err := client.ZRange(wordsKey, i + 1, -1).Result()
	if err != nil {
		panic(err)
	}
	results := make([]string, 0)
	for _, t := range ts {
		if t[len(t)-1] == '$' {
			results = append(results, t[:len(t)-1])
		}
	}
	return results
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379", Password: "foobared", DB: 8})
	wordsKey := "words"
	Save("中国", wordsKey, client)
	Save("中华人民共和国", wordsKey, client)
	Save("中华香烟", wordsKey, client)
	Save("黄山", wordsKey, client)
	results := AutoCompletion("黄", wordsKey, client)
	fmt.Println("黄 auto completion")
	for _, r := range results {
		fmt.Println(r)
	}
	results = AutoCompletion("中", wordsKey, client)
	fmt.Println("中 auto completion")
	for _, r := range results {
		fmt.Println(r)
	}
}

```

#### HASH 存储单词，ZSET 存储内容索引，支持热度

支持**热度**，HASH 存储了单词 id 和单词，ZSET KEY 的值前缀，MEMBER 是单词 id，SCORE 为热度值。

空间占用**稍大**。

其实还支持**组合前缀补全**，例如，利用 `ZINTERSTORE` 即可。

插入

- `HSET` 字符串 <id, str> 到单词表 
- 对于字符串 str，对于其中**字符（一个中文算一个字符，32 位）**，下标 0...N；初始 i = 0，当 i < N - 1 时执行：
  - `ZADD` str[0...i] <id, 热度>
  - i++，继续
  
补全

- `ZRANGE` 前缀 str，结果为 单词 id 的集合
- `HMGET` 单词 id 的集合

代码如下：

```
package main

import (
	"fmt"
	"github.com/go-redis/redis"
	"strconv"
)

func Save(id int, word string, hot int, wordsKey string, client *redis.Client) {
	runes := []rune(word)
	_, err := client.Pipelined(func(pp redis.Pipeliner) error {
		pp.HSet(wordsKey, strconv.Itoa(id), word)
		for i := 0; i < len(runes)-1; i++ {
			key := string(runes[:i+1])
			pp.ZAdd(key, redis.Z{Score: float64(hot), Member: id})
		}
		return nil
	})
	if err != nil {
		panic(err)
	}
}

func AutoCompletion(str string, wordsKey string, client *redis.Client) []string {
	ts, err := client.ZRevRange(str, 0, -1).Result()
	if err != nil {
		panic(err)
	}
	is, err := client.HMGet(wordsKey, ts...).Result()
	if err != nil {
		panic(err)
	}
	results := make([]string, 0)
	for _, i := range is {
		results = append(results, i.(string))
	}
	return results
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379", Password: "foobared", DB: 8})
	wordsKey := "words"
	Save(1, "中国", 20, wordsKey, client)
	Save(2, "中华人民共和国", 90, wordsKey, client)
	Save(3, "中华香烟", 0, wordsKey, client)
	Save(4, "黄山", 1, wordsKey, client)
	results := AutoCompletion("黄", wordsKey, client)
	fmt.Println("黄 auto completion")
	for _, r := range results {
		fmt.Println(r)
	}
	results = AutoCompletion("中华", wordsKey, client)
	fmt.Println("中 auto completion")
	for _, r := range results {
		fmt.Println(r)
	}
}

```

#### 插入稍小稍大

假定有字符串 xyz，通过插入比 xyz **稍小**的一个字符串，和 xyz **稍大**的一个字符串，再 `ZRANGEBYLEX` 取出之间内容。

- 假定字符集 max 值不会检索到（实际上这个字符的确是未定义），比 xyz 更小的字符串为 xy(z-1)字符集 max 值（共 4 个字符），`ZRANGEBYLEX` 取开 (，**不包含**
- 比 xyz 更大的字符串为 xy(z+1)（共 3 个字符），`ZRANGEBYLEX` 取开 (，**不包含**

占用空间是最少的，但是效率十分有问题，因为所有单词在一个集合中。

插入

- `ZADD` 单词表 <字符串, 0>
  
补全

- 插入比检索词**稍小**的数组（参见上面计算方式）start
- 插入比检索词**稍大**的数组（参见上面计算方式）end
- `ZRANGEBYLEX` 单词表 start end
- 用完之后 `ZREM` **删除** start end

```
package main

import (
	"fmt"
	"github.com/go-redis/redis"
)

func Save(word string, wordsKey string, client *redis.Client) {
	err := client.ZAdd(wordsKey, redis.Z{Member: word, Score: 0}).Err()
	if err != nil {
		panic(err)
	}
}

func AutoCompletion(str string, wordsKey string, client *redis.Client) []string {
	startRunes, endRunes := append([]rune{'('}, []rune(str)...), append([]rune{'('}, []rune(str)...)
	// latest = latest - 1
	startRunes[len(startRunes)-1] = startRunes[len(startRunes)-1] - 1
	// 添加一个最大值到结尾
	startRunes = append(startRunes, 0x7fffffff)
	// 结尾字符串 + 1
	endRunes[len(endRunes)-1] = endRunes[len(endRunes)-1] + 1
	endRunes[0] = '('
	results, err := client.ZRangeByLex(wordsKey, redis.ZRangeBy{Min: string(startRunes), Max: string(endRunes)}).Result()
	if err != nil {
		panic(err)
	}
	client.ZRem(wordsKey, startRunes)
	client.ZRem(wordsKey, endRunes)
	return results
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379", Password: "foobared", DB: 8})
	wordsKey := "words"
	Save("中国", wordsKey, client)
	Save("中华人民共和国", wordsKey, client)
	Save("中华香烟", wordsKey, client)
	Save("黄山", wordsKey, client)
	Save("黃山", wordsKey, client)
	Save("黅山", wordsKey, client)
	Save("aa", wordsKey, client)
	Save("abb", wordsKey, client)
	Save("abbb", wordsKey, client)
	Save("ac", wordsKey, client)
	results := AutoCompletion("黄", wordsKey, client)
	fmt.Println("黄 auto completion")
	for _, r := range results {
		fmt.Println(r)
	}
	results = AutoCompletion("中华", wordsKey, client)
	fmt.Println("中 auto completion")
	for _, r := range results {
		fmt.Println(r)
	}
	results = AutoCompletion("ab", wordsKey, client)
	fmt.Println("ab auto completion")
	for _, r := range results {
		fmt.Println(r)
	}
}

```


### 比较

| 算法 | 优点 | 缺点 |
| --- | --- | --- | 
| 插入前缀 | 简单 | 不支持热度，数据量大存在性能问题 |
| HASH 存储单词，ZSET 存储内容索引 | 支持热度，可扩展性强 | 稍微复杂，占用内存稍多 |
| 插入稍小稍大 | 占用空间小 | 稍复杂，数据量大存在性能问题 |

我推荐 HASH + ZSET 方法。

### 参考

- [Redis 作者实现的自动补全](http://oldblog.antirez.com/post/autocomplete-with-redis.html)
- [另一种自动补全](https://dzone.com/articles/two-ways-using-redis-build)
- [《Redis 实战》自动补全](https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-1-autocomplete/)

## 随机权重 Random Weight

随机权重是指*根据权重随机分配资源*，常见于负载均衡算法，根据机器负载确定机器权重，然后负载均衡器根据权重进行随机分配。

### 算法

- 保存权重
  - 计算出所有资源权重**总和** sum
  - 初始化**当前权重**值 weightStart 0
  - 循环，直到所有权重添加入权重 ZSET
    - `ZADD` 权重 KEY，MEMBER 为资源标识，SCORE 为 weightStart
    - weightStart += 当前权重，继续

- 随机权重
  - `ZREVRANGEBYSCORE` KEY -inf rand[0...1) limit 0 1，得到选中的资源标识

代码如下：

```
package main

import (
	"fmt"
	"github.com/go-redis/redis"
	"math/rand"
	"strconv"
)

func Save(id int, weight float64, key string, client *redis.Client) {
	if weight <= 0 {
		panic("weight <= 0")
	}
	err := client.ZAdd(key, redis.Z{Member: id, Score: weight}).Err()
	if err != nil {
		panic(err)
	}
}

func Random(key string, client *redis.Client) int {
	f := rand.Float64()
	results, err := client.ZRevRangeByScore(key, redis.ZRangeBy{Min: "-inf", Max: fmt.Sprintf("%.2f", f), Offset: 0, Count: 1,}).Result()
	if err != nil {
		panic(err)
	}
	if len(results) == 0 {
		panic("results' size == 0")
	}
	i, err := strconv.Atoi(results[0])
	if err != nil {
		panic(err)
	}
	return i
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379", Password: "foobared", DB: 1})
	weightMap := map[int]float64{1: 50, 2: 30, 3: 20}
	sum := float64(0)
	// 计算总权重
	for _, weight := range weightMap {
		sum += weight
	}
	// 保存
	key := "random_weight"
	weightStart := float64(0)
	for id, weight := range weightMap {
		Save(id, weightStart/sum, key, client)
		weightStart += weight
	}
	m := make(map[int]int)
	m[1], m[2], m[3] = 0, 0, 0
	// 加权随机
	for i := 0; i < 1000; i++ {
		val := Random(key, client)
		fmt.Println("choose", val)
		m[val]++
	}
	fmt.Printf("%#v", m)
}

```

## 归一汇总排名 Normalize Summary Rank

归一汇总排名的意思是，*对于多个权重集合，将每个权重集合中的每个资源的值，通过一定方式，使其处于 [0, 1] 之间；之后对于每个资源，按照一定权重进行汇总（每个资源总和不超过 1），计算出目标总权重*。
这种使用方式适用于于分配资源等场景。

### 算法

- 归一
  - 确定一个**归一化**的公式，比如，某个资源的上限为 N，当前值为 M，那么得分为 N / M
  - `ZADD` 资源集合 资源标识 得分
- 汇总
  - 得到所有资源集合，`ZINTERSTORE` 资源 1 资源 2 资源 3 ... 资源 n，权重 w1 w2 w3 ... wn，其中，w1 + w2 + w3 + ... + wn = 1，保存到 KEY
  - 读取 KEY 的值

代码见下：

```
package main

import (
	"fmt"
	"github.com/go-redis/redis"
	"github.com/google/uuid"
	"strconv"
	"time"
)

func Save(key string, id int, score float64, client *redis.Client) {
	err := client.ZAdd(key, redis.Z{Member: strconv.Itoa(id), Score: score}).Err()
	if err != nil {
		panic(err)
	}
}

func SummarizeWeight(keys []string, ratios []float64, client *redis.Client) map[int]float64 {
	random := uuid.New().String()
	err := client.ZInterStore(random, redis.ZStore{Weights: ratios, Aggregate: "SUM"}, keys...).Err()
	if err != nil {
		panic(err)
	}
	err = client.Expire(random, time.Minute).Err()
	if err != nil {
		panic(err)
	}
	results, err := client.ZRangeWithScores(random, 0, - 1).Result()
	if err != nil {
		panic(err)
	}
	m := make(map[int]float64)
	for _, z := range results {
		i, err := strconv.Atoi(z.Member.(string))
		if err != nil {
			panic(err)
		}
		m[i] = z.Score
	}
	return m
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379", Password: "foobared", DB: 8})
	// 设定 CPU 上限为 8
	Save("CPU", 1, 2, client)
	Save("CPU", 2, 4, client)
	Save("CPU", 3, 2, client)
	// 设定 Memory 上限为 8
	Save("Memory", 1, 4, client)
	Save("Memory", 2, 1, client)
	Save("Memory", 3, 8, client)
	// 设定 Disk 上限为 1024
	Save("Disk", 1, 512, client)
	Save("Disk", 2, 1024, client)
	Save("Disk", 3, 256, client)
	// CPU 比重为 0.3，Memory 比重为 0.4，Disk 比重为 0.3
	// 归一计算公式： CPU / 8 * 0.3 + Memory / 8 * 0.4 + Disk / 1024 * 0.3
	results := SummarizeWeight([]string{"CPU", "Memory", "Disk"}, []float64{1.0 / 8.0 * 0.3, 1.0 / 8 * 0.4, 1.0 / 1024 * 0.3}, client)
	fmt.Printf("%#v", results)
}

```