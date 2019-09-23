## 关键词搜索 Keyword Search

关键词搜索，是指*搜索包含关键词的内容*。如果系统只需要简单的搜索功能，Redis 可以简单实现关键词搜索的功能。

> 若需要更进阶的功能，请参考 Elastic Search、Redis Search 等。

### 算法

总共有三个流程：

- 生成索引，添加内容的时候，分词，根据分词的结果生成生成索引。
- 检索，将关键词分词，取出每个单词的集合，根据需要取交集、并集、差集等。
- 清理，删除内容清理索引。

具体而言：

- 生成索引
  - `SET` Key 为文章 id，Value 为文章内容
  - （假定单词为英文，分词为空格，中文有分词器）文章分词，`SADD` Key 为词，Value 为文章 id

- 检索
  - 设定 + 取并集 ，- 取差集，比如，hello world 为包含 hello **和** world 的文章，hello +world 为包含 hello **或者** world 的文章，hello -world 表示包含 hello **不包含** world 的文章
  - 无符号：`SINTER` 所有单词
  - 符号 +：上一步结果叠加 `SUNION` 所有单词
  - 符号 -：上一步结果叠加 `SDIFF` 所有单词

- 清理
  - 取出需要清理的文章，分词
  - 先 `DEL` 删除索引，避免被再次检索到
  - 后 `DEL` 删除文章

代码如下:

```
package main

import (
	"fmt"
	"github.com/go-redis/redis"
	"github.com/google/uuid"
	"strconv"
	"strings"
	"time"
)

func split(content string) []string {
	return strings.Split(content, " ")
}

func Create(id int, content string, client *redis.Client) {
	if len(content) == 0 {
		return
	}
	// 使用 pipeline 提高写入性能
	_, err := client.Pipelined(func(pp redis.Pipeliner) error {
		// 文章
		_ = pp.Set(strconv.Itoa(id), content, 0).Err()
		// 索引
		words := split(content)
		for _, word := range words {
			pp.SAdd(word, id)
		}
		return nil
	})
	if err != nil {
		panic(err)
	}
}

func Search(origins, pluses, minuses []string, client *redis.Client) []string {
	if len(origins) == 0 {
		return []string{}
	}
	// 结果存储在一个键中
	randomStore := uuid.New().String()
	// 无符号
	err := client.SInterStore(randomStore, origins...).Err()
	if err != nil {
		panic(err)
	}
	if len(pluses) > 0 {
		// 处理符号 +
		pluses = append(pluses, randomStore)
		pluses[0], pluses[len(pluses)-1] = pluses[len(pluses)-1], pluses[0]
		err = client.SUnionStore(randomStore, pluses...).Err()
		if err != nil {
			panic(err)
		}
	}
	if len(minuses) > 0 {
		// 处理符号 -
		minuses = append(minuses, randomStore)
		minuses[0], minuses[len(minuses)-1] = minuses[len(minuses)-1], minuses[0]
		err = client.SDiffStore(randomStore, minuses...).Err()
		if err != nil {
			panic(err)
		}
	}
	ids, err := client.SMembers(randomStore).Result()
	if err != nil {
		panic(err)
	}
	if len(ids) == 0 {
		return []string{}
	}
	err = client.Expire(randomStore, time.Minute).Err()
	if err != nil {
		panic(err)
	}
	// 处理结果
	contentInterfaces, err := client.MGet(ids...).Result()
	if err != nil {
		panic(err)
	}
	contents := make([]string, len(contentInterfaces))
	for i := 0; i < len(contentInterfaces); i++ {
		contents[i] = contentInterfaces[i].(string)
	}
	return contents
}

func Clean(id int, client *redis.Client) {
	// 获取文章
	content, err := client.Get(strconv.Itoa(id)).Result()
	if err != nil {
		panic(err)
	}
	// 使用 pipeline 提高写入性能，MULTI EXEC 事务删除
	_, err = client.TxPipelined(func(pp redis.Pipeliner) error {
		// 删除索引
		words := split(content)
		for _, word := range words {
			pp.SRem(word, id)
		}
		// 删除文章
		pp.Del(strconv.Itoa(id))
		return nil
	})
	if err != nil {
		panic(err)
	}
}

func main() {
	client := redis.NewClient(&redis.Options{Addr: "localhost:6379", Password: "foobared", DB: 7})
	contents := []string{"Hello", "Hello World", "World", "Ignore"}
	for i, content := range contents {
		Create(i, content, client)
	}
	results := Search([]string{"Hello", "World"}, []string{}, []string{}, client)
	fmt.Println("Search Hello World:")
	for _, result := range results {
		fmt.Println(result)
	}
	results = Search([]string{"Hello"}, []string{"World"}, []string{}, client)
	fmt.Println("Search Hello +World")
	for _, result := range results {
		fmt.Println(result)
	}
	results = Search([]string{"Hello"}, []string{}, []string{"World"}, client)
	fmt.Println("Search Hello -World:")
	for _, result := range results {
		fmt.Println(result)
	}
	Clean(1, client)
}

```

### 参考

- [Redis Search](https://github.com/RediSearch/RediSearch)