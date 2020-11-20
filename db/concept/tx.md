## 为什么需要数据库事务

为了数据的正确性，拿提现举例：

- 提现时，钱包余额必须要减少，产生一条提现记录
- 钱包余额减少的数量必须和提现记录的数量保持一致
- 同时提现时，钱包扣减的总数是正确的，提现记录产生的数量也是正确的
- 数据库崩溃恢复之后，钱包余额和提现记录必须是正确的数据

## 事务定义

数据库事务是构成单一逻辑工作单元的操作集合。

### 案例

```sql
create table wallet -- 钱包
(
	account varchar(32) not null, -- 账户
	amount bigint not null -- 账户总计金额
);

create table withdrawal -- 提现
(
    account varchar(32) not null, -- 账户
    amount bigint not null -- 提现金额
);
```

举例：钱包提现 10 元
```sql
-- 减少钱包余额
update wallet set total_amount = total_amount - 10 where account = "亚古兽";
-- 增加提现记录
insert into withdrawal values ("亚古兽", 10)
```

#### 原子性

事务中的所有操作作为一个整体像原子一样不可分割，要么*全部成功*，要么*全部失败*。

举例：给 yagushou 扣除钱包余额和增加提现记录，必须同时成功或者同时失败，不存在其他状态。

数据库保证这个特性。

#### 一致性

事务的执行结果必须使数据库*从一个一致性状态到另一个一致性状态*。一致性状态是指:

1. 系统的状态满足数据的完整性约束(主码，参照完整性，check约束等) 
1. 系统的状态反应数据库本应描述的现实世界的真实状态。

举例：给亚古兽创建 10 元入账记录的时候，最终的结果必定是余额扣减 10 元，开发者必须判断余额充足。

这一个特性比较特殊，是唯一一个特性，需要依赖于开发者保证。

#### 隔离性

*并发执行*的事务*不会相互影响*，其对数据库的影响和它们串行执行时一样。

举例：同时给亚古兽增加 10 个 10 元的钱包余额，最终钱包余额增加 100 元。

#### 持久性

事务一旦提交，其对数据库的更新就是持久的，任何系统故障都*不会导致数据丢失*。

举例：增加亚古兽入账记录 10 元，并增加亚古兽钱包余额 10 元，最后提交事务，若此时数据库崩溃，恢复正常时，提现记录应该增加记录，钱包余额应该减少。

### 事务隔离性：级别

这里按照标准定义的[事务](https://zh.wikipedia.org/wiki/%E4%BA%8B%E5%8B%99%E9%9A%94%E9%9B%A2#%E9%BB%98%E8%AE%A4%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB)级别进行说明。

#### READ UNCOMMITTED

可以读到未提交的事务。

假设给用户余额 < 10 元的用户增加 10 元作为新手礼包。

| 事务 1 | 事务 2 |
| ---- | ---- |
| begin | begin |
| | update wallet set amount=20 where account='yagushou'  |
| select account from wallet where amount<10 |  |
| 查出了 yagushou，20 元 | | 
| rollback | |
| | rollback | 

事务 1 读到了事务 2 没提交的脏数据，发现 yagushou 的余额超过 20 元，但事务 2 回滚了，导致 yagushou 无法享受新手礼包。

这种情况叫做`脏读`，A 事务读取 B 事务尚未提交的数据,此时如果 B 事务发生错误并执行回滚操作,那么 A 事务读取到的数据就是脏数据，一般情况下不建议使用该级别。

#### READ COMMITTED

只能读取已提交的数据。

假设给用户余额 < 10 元的用户增加 10 元作为新手礼包，给余额在 > 10 且 < 15 元的用户增加 5 元作为普照礼包。

| 事务 1 | 事务 2 |
| ---- | ---- |
| begin | begin |
| select account from wallet where amount < 10 |  |
| 查出了 yagushou，5 元 | |
| update wallet set amount=5+10 where account='yagushou' |  |
| | 亚古兽提现 3 元 |
|  | update wallet set amount=15-3 where account='yagushou'  | 
| | commit | 
| select account from wallet where amount >= 10  and amount < 15 | | 
| 查出了 yagushou，12 元 | |
| update wallet set amount=12+5 where account='yagushou' |  |
| commit | |

最终，yagushou 既领了新手礼包，又领了普照礼包。

这种情况叫做`不可重复读`，一个事务范围内两个相同的查询却返回了不同数据。80% 的情况使用该级别是足够的，大多数事务不会重复读取数据。

#### REPEATABLE READ

可重复读，在一个事务内读取的数据都是一样的。

假设救济贫穷，给**现有**的穷困用户，发放 10 元。

| 事务 1 | 事务 2 |
| ---- | ---- |
| begin | begin |
| select account from wallet where amount < 10 |  |
| 只查出了 yagushou，5 元 | |
| | insert into wallet values('wukong', 0)
| | commit | 
| update wallet set amount=5+10 where amount < 10 |  |
| 更新记录 2 条 | |
| commit  | | 

本来只查到 yagushou，但是这时候 wukong 创建了账号，wukong 也获得了 10 元。

这种情况叫做`幻读`，是事务非独立执行时发生的一种现象。在范围查询的时候，重复读取发下多出了数据。这个级别基本可以应对 99% 的情况。

#### SERIALIZABLE

事务只能按照顺序执行修改。

| 事务 1 | 事务 2 |
| ---- | ---- |
| begin | begin |
| select account from wallet where account < 10 |  |
| | select account from wallet where account < 20 |
| | 正在被事务占用 |
| | rollback | 
| commit  | | 

这种情况不会出现什么问题，但是很多查询不能并发进行了，性能急剧下降。不建议使用这个级别，并发性基本趋近于 0。

## PostgreSQL 实现

[参考链接](https://www.postgresql.org/docs/9.4/transaction-iso.html)。

相比于标准事务隔离级别：
- 没有实现 READ UNCOMMITTED，READ UNCOMMITTED 效果和 READ COMMITTED 一样
- REPEATABLE READ 不会有幻读的问题

具体内部细节：
- READ UNCOMMITTED，每次进行查询时，都会读取最新快照，所以当其他事务提交了修改时，快照会刷新，所以总是能发现。
- REPEATABLE READ，进入事务的那一刻，就做了快照，直到事务结束。
- SERIALIZABLE，多个事务有修改操作，第一个提交的事务成功，其余在提交的时候会失败。

```sql
-- Init table & data
drop table wallet;

create table wallet -- 钱包
(
    account varchar(32) not null, -- 账户
    amount bigint not null -- 账户总计金额
);
insert into wallet values('yagushou', 5);
insert into wallet values('wukong', 10);

-- READ UNCOMMITTED
begin transaction isolation level read uncommitted;
select * from wallet where account='yagushou';
-- 调用 READ UNCOMMITTED 1 修改记录
select * from wallet where account='yagushou';
-- 未发生脏读，没有实现 read uncommitted
commit;

-- READ COMMITTED
begin transaction isolation level read committed;
select * from wallet where amount < 10;
-- 调用 READ COMMITTED 1 修改记录
select * from wallet where amount < 10;
-- 调用 READ COMMITTED 2 增加记录
select * from wallet where amount < 10;
commit;

-- REPEATABLE READ
begin transaction isolation level repeatable read;
select * from wallet where amount < 10;
-- 调用 REPEATABLE READ 1 增加记录
select * from wallet where amount < 10;
-- 未发生幻读
commit;
select * from wallet where amount < 10;

-- REPEATABLE READ
begin transaction isolation level repeatable read ;
select * from wallet where amount < 10;
-- 调用 REPEATABLE READ 2 增加记录
insert into wallet values ('bumblebee', 9);
commit;

-- SERIALIZABLE
begin transaction isolation level serializable;
select * from wallet where amount < 10;
insert into wallet values ('bumblebee', 9);
-- 调用 SERIALIZABLE 1 写入数据
commit;
```

```sql
-- READ UNCOMMITTED 1
begin transaction isolation level read uncommitted;
update wallet set amount= 6 where account='yagushou';
commit;

-- READ COMMIT 1
begin transaction isolation level read committed;
update wallet set amount= 6 where account='yagushou';
commit;

-- READ COMMIT 2
begin transaction isolation level read committed;
insert into wallet values ('bumblebee', 8);
commit;

-- REPEATABLE READ 1
begin transaction isolation level read committed;
insert into wallet values ('bumblebee', 8);
commit;

-- REPEATABLE READ 2
begin transaction isolation level repeatable read;
select * from wallet where amount < 10;
insert into wallet values ('bumblebee', 8);
commit;

-- SERIALIZABLE 1
begin transaction isolation level serializable;
select * from wallet where amount < 10;
insert into wallet values ('bumblebee', 8);
commit;
```

## MySQL 实现

[参考链接](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)。

具体细节：
- READ COMMITTED，lock 匹配上的行，例如 id > 5 查出三条记录，这三条记录会被锁定。
- REPEATABLE READ，lock 匹配的范围，例如 id > 5，会把 id > 5 的范围（gap）锁定。
- SERIALIZABLE，如果有事务进行了修改，表会变为只读状态，其他的事务只能访问，如果修改就返回失败。

```sql
-- Init table & data
drop table wallet;

create table wallet -- 钱包
(
    account varchar(32) not null, -- 账户
    amount bigint not null -- 账户总计金额
);
insert into wallet values('yagushou', 5);
insert into wallet values('wukong', 10);

-- READ UNCOMMITTED:
set session transaction isolation level read uncommitted;
start transaction;
select * from wallet where account='yagushou';
-- 调用 READ UNCOMMITTED 1 修改记录
select * from wallet where account='yagushou';
-- 脏读
commit;

-- READ COMMITTED：
set session transaction isolation level read committed;
start transaction;
select * from wallet where amount < 10;
-- 调用 READ commit 1 修改记录
select * from wallet where amount < 10;
-- 调用 READ commit 2 增加记录
select * from wallet where amount < 10;
commit;

-- REPEATABLE READ
set session transaction isolation level repeatable read;
start transaction;
select * from wallet where amount < 10;
-- 调用 REPEATABLE READ 1 增加记录
select * from wallet where amount < 10;
-- 未发生幻读，why？Postgresql 做了一次 Table 快照，在同一个事务中只读快照
commit;
select * from wallet where amount < 10;

-- REPEATABLE READ
set session transaction isolation level repeatable read;
start transaction;
select * from wallet where amount < 10;
-- 调用 REPEATABLE READ 2 增加记录
insert into wallet values ('bumblebee', 9);
commit;

-- SERIALIZABLE
set session transaction isolation level serializable;
start transaction;
select * from wallet where amount < 10;
insert into wallet values ('bumblebee', 9);
commit;
```

```sql
-- READ UNCOMMITTED 1
set session transaction isolation level read uncommitted;
start transaction;
update wallet set amount= 6 where account='yagushou';
-- 返回查看另一个事务的值
commit;

-- READ COMMIT 1
set session transaction isolation level read committed;
start transaction;
update wallet set amount= 6 where account='yagushou';
commit;

-- READ COMMIT 2
set session transaction isolation level read committed;
start transaction;
insert into wallet values ('bumblebee', 8);
commit;

-- REPEATABLE READ 1
set session transaction isolation level read committed;
start transaction;
insert into wallet values ('bumblebee', 8);
commit;

-- REPEATABLE READ 2
set session transaction isolation level repeatable read;
start transaction;
select * from wallet where amount < 10;
insert into wallet values ('bumblebee', 8);
commit;

-- SERIALIZABLE 1
set session transaction isolation level serializable;
start transaction;
select * from wallet where amount < 10;
insert into wallet values ('bumblebee', 8);
commit;
```

## 扩展

- 锁的类型：基于版本控制的锁、共享锁与排他锁
- 事务恢复机制：redo 和 undo

## 参考

https://www.cnblogs.com/takumicx/p/9998844.html

https://www.cnblogs.com/dwxt/p/8807899.html
