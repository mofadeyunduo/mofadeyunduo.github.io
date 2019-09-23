## 字段排序 Field Sort

字段排序的意识是*根据 Hash 某个字段进行排序*，主要利用 SORT 命令。

其实，把每个 Hash 类比为一行，Field 的内容为 Column 的内容，就和 MySQL 表的数据结构基本保持一致。对于数据的操作，可以通过 `SORT` 命令实现，类比如下：

| SORT 命令参数 | MySQL 命令参数 |
| --- | --- | 
| SORT ${table} | FROM ${table} |
| BY hash*->${id} |  ORDER BY ${id} |
| GET hash*->${name} | SELECT ${name} | 
| DESC ASC | DESC ASC |
| LIMIT ${offset} ${count} | LIMIT ${offset} ${count} |
| ALPHA | |
| STORE ${table} | INSERT INTO ${table} SELECT |

**单表下**，**除了 WHERE 功能**，MySQL 简单处理数据的功能，基本上 Redis 也能做到。

### 算法

模拟场景，比如 Student 表，里面有 id、name、cscore、mscore 等字段，现在存储在 Redis 中，是以 Hash 结构保存，如下所示：

```
HASH stu:${id}
- id
- name
- cscore
- mscore 
```

- 插入数据
  - 插入元素 `HMSET`
  - 插入索引字段 `SADD` 到某个集合 A
- 查询数据
 - 利用 `SORT` 集合 A，`BY` 字段，`GET` 字段等

代码如下：

```
# 插入学生成绩
HMSET stu1 id 1 name A cscore 90 mscore 80
# 学生集合 id 插入学生集合
SADD stu 1
HMSET stu2 id 2 name B cscore 82 mscore 85
SADD stu 2
HMSET stu3 id 3 name C cscore 95 mscore 88
SADD stu 3
# 根据 cscore 排序（降序），显示第一名学生姓名
SORT stu BY stu*->cscore GET stu*->name LIMIT 0 1 DESC
# 根据 mscore 排序，显示学生 id
SORT stu BY stu*->mscore GET stu*->id
```