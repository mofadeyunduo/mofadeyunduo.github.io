## 作用

Sentinel，哨兵，维护 Redis 服务的`高可用性`。

Sentinel 通过`投票`计算出正确的节点，将**失效的 Master 下线，工作的 Slave 上线**，保证高可用性。

## 原理

- 假设 `Sentinel` 有 n 个，设置**失效临界值** `quorum` = m
- 每个 `Sentinel` s1...sn 会定时去**探测** `Master`、`Slave` 的状况
- 假设有 p 个 `Sentinel` 认为 `Master` 失效
  - 当 p < m 时，判定 `Master` **在线**，结束
- 此时，`Master` **下线**，并需要将一个 `Slave` **上线**，变成 `Master`
- `Sentinel` sx 寻找**可用** `Slave`
  - 如果没有 `Slave`，**服务不可用**，结束
  - sx 发现有 q 个 `Sentinel` 认为 `Slave` **在线**
    - 当 q > n / 2，成功将 `Slave` **上线**，结束
    - 否则，**寻找下一个** `Slave`，重复上面流程

## 部署

> 相对推荐部署方式，在每台 `Master`、`Slave` 上部署 `Sentinel`

### Sentinel 部署数量为 2 

- `quorum` = 1，`Master` 和 s1 **同时下线**，此时 s2 可以触发 `Master` **下线**，但由于可用 `Sentinel` 数量未满足 > 2 / 2 = 1，导致**无法成功**将 `Slave` 变为 `Master`，导致服务不可用，实际上 `Slave` **可以变为** `Master` 进行服务

### Sentinel 部署在 Client 上

- 个人觉得和业务服务器部署在一起，不是很好，耦合性很强
  - 如果业务下线，那么 `Sentinel` 也下线；如果想维护高可用性，**业务必须高可用**，这在很多情况是达不到的
  - `Client` **同时必须部署三个**，保证 `Sentinel` 正常工作

### 脑裂

当 `Sentinel` **连接不上** `Master`、`Slave`，但是 `Client` **可以连接** `Master` 时，`Client` 可以向 `Master` 写入数据，但是 `Sentinel` 会选举出**新的** `Master`
- 第一种，假设 `Client` 未能及时刷新 `Sentinel`，未获取到最新的 `Master` 节点，会持续向**旧的** `Master` 写入，
之后当 `Master` 可以重新和 `Sentinel`、`Slave` 连接上时，Master 发现自己是 `Slave` 节点，会清空自己的数据进行同步，**丢失所有数据**
- 第二种，后续连接的 `Client` 会写入**新的** `Master` 中

这种情况称为**脑裂**，解决办法是设置 `min-replicas-to-write` `min-replicas-max-lag`，通过设置**最小副本数**和**最大复制延迟数**，
从而保证值的**同步**，并且在无法正常连接 `Slave` 时，`Master` **及时下线**