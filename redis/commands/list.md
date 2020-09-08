## 消息队列 Message Queue

消息队列是*将消息生产者消息放入队列中，消息消费者从队列取出的队列*。

> Redis 实现消息队列功能较为简单，如果有复杂使用请使用专业的消息队列，如 Kafka、RabbitMQ 等。

### 算法

- 推送队列用 `LPUSH`
- 接受消息用 `BRPOP`

```
// 向队列左端推送消息
LPUSH MSQ_QUEUE MSG1
// 阻塞，直到可以从队列右端读取到消息
BRPOP MSQ_QUEUE 0
```

## 可靠消息队列 Reliable Message Queue

在生产环境中，消息队列需要**可靠性**：
- 消息**不能丢失**
- 消息如果未被正确处理，需要返回队列**重新被消费**

这就需要结合 `BRPOPLPUSH` 命令，该命令会从 Key1 右端弹出元素，Key2 左端弹入元素。

> Redis 实现消息队列功能较为简单，如果有复杂使用请使用专业的消息队列，如 Kafka、RabbitMQ 等。

### 算法

- 启动**守护线程**（单一守护线程），该线程监视 DoingQueue，每分钟遍历 `LRANGE` 查询该消息队列所有元素 
  - 如果该消息超出最大处理时间，`LREM` 该元素，`LPUSH` MessageQueue
- 客户端 A 将消息 A `LPUSH` 推入消息队列，名为 MessageQueue
- 客户端 B 阻塞等待读取消息；直到读取消息 A，利用 `BRPOPLPUSH`，从 MessageQueue 右端弹出，弹入 DoingQueue 左端
- 客户端 B 处理完消息，从 DoingQueue **删除** `LREM` 消息 A

如果 DoingQueue 过长，由于 `LRANGE` 是 O(n) 的时间复杂度，性能会很低，**数据大**时建议用**专业的消息队列**。

```
package main

import (
	"encoding/json"
	"fmt"
	"github.com/go-redis/redis"
	"math/rand"
	"time"
)

// 任务
type Task struct {
	Id        int
	Timestamp int64
}

// 发布任务
func Publish(messageQueue string, i int, client *redis.Client) {
	task := Task{Id: i, Timestamp: time.Now().Unix()}
	taskBs, err := json.Marshal(task)
	if err != nil {
		fmt.Println(err)
		panic(err)
	}
	// messageQueue 左端入队
	err = client.LPush(messageQueue, string(taskBs)).Err()
	if err != nil {
		fmt.Println(err)
		panic(err)
	}
	fmt.Println("push", task.Id, "to queue")
}

// 消费任务
func Receiver(messageQueue, doingQueue string, client *redis.Client) {
	// messageQueue 右端推出一个元素，doingQueue 左端入队该元素，表示正在使用
	taskJson, err := client.BRPopLPush(messageQueue, doingQueue, 0).Result()
	task := Task{}
	err = json.Unmarshal([]byte(taskJson), &task)
	if err != nil {
		fmt.Println(err)
		panic(err)
	}
	// 模拟随机失败
	if task.Id%10 == 5 && rand.NewSource(time.Now().UnixNano()).Int63()%2 == 0 {
		fmt.Println("task", task.Id, "execution failed")
		return
	}
	// 移除正在使用
	err = client.LRem(doingQueue, 1, taskJson).Err()
	if err != nil {
		fmt.Println(err)
		panic(err)
	}
	fmt.Println("task", task.Id, "execution successfully")
}

// 监控任务
func Monitor(messageQueue, doingQueue string, client *redis.Client) {
	ticker := time.NewTicker(time.Second)
	for ; ; <-ticker.C {
		// 取出所有任务
		taskJsons, err := client.LRange(doingQueue, 0, -1).Result()
		if err != nil {
			if err == redis.Nil {
				continue
			}
			fmt.Println(err)
			continue
		}
		for _, taskJson := range taskJsons {
			// 反序列化任务
			task := Task{}
			err = json.Unmarshal([]byte(taskJson), &task)
			if err != nil {
				fmt.Println(err)
				continue
			}
			// 超过 10 s 的任务，超时，重新放入队列
			if time.Now().Unix()-task.Timestamp > 10 {
				count, err := client.LRem(doingQueue, 1, taskJson).Result()
				if err != nil {
					fmt.Println(err)
					continue
				}
				if count == 0 {
					fmt.Println("already remove", task.Id, " by others, don't push to", messageQueue)
					continue
				}
				fmt.Println("push task", task.Id, "to message queue again")
				err = client.LPush(messageQueue, taskJson).Err()
				if err != nil {
					fmt.Println(err)
					continue
				}
			}
		}
	}
}

func main() {
	client := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "foobared",
		DB:       3,
	})
	messageQueue, doingQueue := "message_queue", "doing_queue"
	go func() {
		for i := 0; i < 10000; i++ {
			Publish(messageQueue, i, client)
			time.Sleep(time.Second)
		}
	}()
	go func() {
		for {
			Receiver(messageQueue, doingQueue, client)
		}
	}()
	go Monitor(messageQueue, doingQueue, client)
	go Monitor(messageQueue, doingQueue, client)
	time.Sleep(time.Minute * 10)
}

```

## 循环队列 Circular Queue

循环队列是*一种头尾相接的队列*，遍历到尾时，下一个节点是头。这种队列适合做定时任务去检查一组资源，比如，一组网站需要每隔 K 分钟检查连通性。

核心是利用 `RPOPLPUSH` 命令，在从右端读取元素的时候，再次写入到左端。
实质相当于组合用 `RPOP` 和 `LPUSH` 命令，区别是操作是**原子性**的，可以避免 `RPOP` 和 `LPUSH` 执行期间客户端崩溃从而**丢失数据**的问题。

### 算法

- `LLEN` 获取**队列长度**，`SETNX` 设置队列长度到 lenKey
  - 设置成功，说明无客户端正在执行任务，继续执行
  - 设置失败，说明存在客户端，**已经发起过任务**，`GET` lenKey 的长度，作为队列长度
- `INCR` idxKey（已经遍历过的元素规模） <= lenKey（大于则**遍历完成**）
  - 值为 1，表明该客户端是**任务发起者**，设置过期时间，继续下一步
  - 执行 `RPOPLPUSH` 元素 val，对元素 val 执行任务

```
package main

import (
	"fmt"
	"github.com/go-redis/redis"
	"strconv"
	"time"
)

func CircularQueue(queue, lenKey, idxKey string, handleTask func(id int), client *redis.Client) {
	ticker := time.NewTicker(time.Minute / 6)
	for ; ; <-ticker.C {
		// 获取队列长度
		length, err := client.LLen(queue).Result()
		if err != nil {
			fmt.Println(err)
			continue
		}
		ok, err := client.SetNX(lenKey, length, time.Minute/6).Result()
		if err != nil {
			fmt.Println(err)
			continue
		}
		// 其他客户端设置了本次轮询队列长度，放弃
		if !ok {
			lengthStr, err := client.Get(lenKey).Result()
			if err != nil {
				fmt.Println(err)
				continue
			}
			length, err = strconv.ParseInt(lengthStr, 10, 64)
			if err != nil {
				fmt.Println(err)
				continue
			}
		}
		// 右端推出，左端推入，遍历所有元素
		for i, err := client.Incr(idxKey).Result(); i <= length; i, err = client.Incr(idxKey).Result() {
			// 第一个取出元素的客户端设置过期时间
			if i == 1 {
				client.Expire(idxKey, time.Minute/6)
			}
			if err != nil {
				fmt.Println(err)
				continue
			}
			// 取出元素
			val, err := client.RPopLPush(queue, queue).Result()
			if err != nil {
				fmt.Println(err)
				continue
			}
			i, err := strconv.Atoi(val)
			if err != nil {
				fmt.Println(err)
				continue
			}
			// 执行任务
			go handleTask(i)
		}
	}
}

func main() {
	client := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "foobared",
		DB:       5,
	})
	circularQueue, lenKey, idxKey := "circular_queue", "len", "idx"
	for i := 0; i < 10; i++ {
		err := client.LPush(circularQueue, i).Err()
		if err != nil {
			fmt.Println(err)
			return
		}
	}
	go CircularQueue(circularQueue, lenKey, idxKey, func(id int) { fmt.Println("check", id, "finish") }, client)
	go CircularQueue(circularQueue, lenKey, idxKey, func(id int) { fmt.Println("check", id, "finish") }, client)
	go CircularQueue(circularQueue, lenKey, idxKey, func(id int) { fmt.Println("check", id, "finish") }, client)
	go CircularQueue(circularQueue, lenKey, idxKey, func(id int) { fmt.Println("check", id, "finish") }, client)
	go CircularQueue(circularQueue, lenKey, idxKey, func(id int) { fmt.Println("check", id, "finish") }, client)
	time.Sleep(time.Minute)
}

```