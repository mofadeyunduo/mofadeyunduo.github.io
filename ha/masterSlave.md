## 主从复制流程

- 从节点（slave，缩写 m）主节点（master，缩写 s）发起 `slaveof` 命令
- m 返回主节点 id = mid
- s 保存 mid
- s 开始数据同步，请求和 m 建立 Socket 链接
- m 建立连接
- s 发起 `ping` 命令，验证主从连通性
- m 返回 `pong`
- s 进行权限验证
- m 返回权限验证结果
- s 的当前主节点 id = cmid，s 接受 m 返回的 mid，假设是全量复制（从未同步、或者 cmid != mid）
  - s 发起 `psync mid -1`
  - m 返回 `FULLRESYNC mid mOffset`，主节点命令字节长度 = mOffset 
    - 同时，s 保存 mid mOffset
    - 同时，m 执行 `bgsave`，保存 rdb，并把 `bgsave` 之后执行的命令加入 `buffer` 中
  - m 发送 `rdb`
  - s 接受 `rdb`
  - s 发送 `buffer`
  - m 接受 `buffer`
  - s 执行 `flushall`
  - s 加载接收的 `rdb`
  - s 加载接收的 `buffer`
  - 如果 s 开启了 `aof`，打开 `aof`
- s 的当前主节点 id = cmid，s 接受 m 返回的 mid，假设是增量复制（cmid == mid，mOffset != sOffset，中途断线）
  - s 发起 `psync mid sOffset`
  - m 比较 mOffset 和 从节点命令字节长度 sOffset，不相等，发送 `CONTINUE`
  - m 从 buffer 中读取并发送数据给 s，更新 mOffset
  - s 接受数据，更新 sOffset

## 主从心跳

- 间隔 N 端时间，m 发送 `ping`，s 回复 `pong`
- 间隔 M 段时间，s 发送 `replconf ack sOffset`，表示收到字节大小，m 接受保存

## 主从复制的异步性

m 发送给 s 的数据是异步的，不阻塞 m 接受请求。

## 参考

- [主从复制详解](https://zhuanlan.zhihu.com/p/60239657)