---
title: "Postgresql索引失效(1)"
date: 2025-08-25T20:00:14+08:00
draft: false
categories: ["Postgresql"]
tags: ["Postgresql", "数据库"]
---

## 案例一：批量删除历史数据导致索引失效的问题

### 表结构

```sql
create table example0
(
   id                                       uuid not null primary key,
   other_column                             text not null,
   created_at                               timestamp with time zone
);

create index idx_example0_created_at on example0 (created_at);
```

### 业务背景

* 表example1大概存储了五年的数据，大概有5亿条数据。
* 表没有做时间分区，所以数据都在同一张表中。
* 由于磁盘成本的压力，业务要求仅保留最近两年的数据
* 方案为`批量删除历史数据 + pgrepack重整表结构`

### 问题复现和分析

清理流程如下：

```sql
-- step 1
SELECT
   id
FROM
   example0
WHERE
   created_at <= '2023-08-25'
ORDER BY
   created_at ASC
LIMIT
   500;

-- step 2
DELETE FROM
   example0
WHERE
   id IN (step_1_ids)

-- step 3
DELETE FROM
   example0_relation_table
WHERE
   example0_id IN (step_1_ids)
```

整体逻辑:

```
while true:
   每次从example0表捞出最老的500条数据。
   删除与这500条数据相关的数据。
```

预期的执行逻辑:

由于存在 idx_example0_created_at 索引，预期情况是每次根据 created_at 从头扫描，找到 500 条后立即返回。随着数据逐步删除，下一次继续从索引的头部扫描，仍然只需扫描少量数据。

实际的执行逻辑:

* PostgreSQL 删除记录时，仅仅标记对应的 表记录和索引项为已删除，并不会立即回收空间。

* 只有在执行 VACUUM 后，标记删除的数据才会被彻底清理。

* 如果未及时执行 VACUUM，这些“已删除”的索引项仍会参与扫描，导致 step_1 每次都要遍历越来越多的无效索引项。

* 由于表体量很大，且未调整 autovacuum 参数，长时间未触发自动 VACUUM。

* 当删除量达到 100 万条时，step_1 查询明显变慢，因为每次扫描都需要跳过大量无效索引项。

最终表现：**批量删除越多，查询性能越差**。

### 解决方案

核心思路: **避免重复扫描已经删除的旧数据**。

```sql
SELECT
   id
FROM
   example0
WHERE
   created_at >= `<previous_step_last_created_at>`
   created_at <= '2023-08-25'
ORDER BY
   created_at ASC
LIMIT
   500;
```


* 维护一个 previous_step_last_created_at 变量，表示上一次处理的最后一条记录的时间。

* 每次循环后更新该值，下一次从该时间点继续扫描，而不是从头扫描。


## 案例二: 索引未命中导致查询性能问题

### 表结构

```sql
create table collection
(
   collection_id        integer not null，
   entity_id            varchar(24) not null,
   status               text，
   created_at           timestamp with time zone,
   primary key (collection_id, entity_id)
);

create index idx_collection_collection_id_status on collection (list_id, status);
```

### 业务背景

collection 表用于存储某类道具的集合，每个道具有两种状态：CREATED（未使用）和 USED（已使用）。
当前业务场景需要实现一个逻辑：**从指定的集合中随机取一个未使用的道具**。

### 问题复现和分析

业务sql如下:

```sql
SELECT
   *
FROM
   collection
WHERE
   collection_id = 113 AND
   status = 'CREATED'
LIMIT
   1
```

测试结果:

* `WHERE collection_id = 110 AND status = 'CREATED'`时会走`idx_collection_collection_id_status`索引，查询时间小于50ms

* `WHERE collection_id = 113 AND status = 'CREATED'`时，却会执行顺序扫描，(query_ms > 2s, 执行时间 > 2 s，扫描行数超过 200 万。


先看一下collection_id分布情况:

```sql
SELECT
   collection_id,
   COUNT(1) count
FROM
   collection
GROUP BY
   collection_id
```

| collection_id | count   |
|---------------|---------|
| 51            | 10000   |
| 57            | 10000   |
| 61            | 200000  |
| 94            | 300000  |
| 99            | 500000  |
| 110           | 50000   |
| 113           | 3000000 |
| 120           | 100000  |

再查看 collection_id 统计信息:

```sql
SELECT
   most_common_vals, most_common_freqs
FROM
   pg_stats
WHERE
   tablename = 'collection' AND
   attname = 'collection_id';
```

| most_common_vals | most_common_freqs   |
|---------------|---------|
| [113, ......]            | [0.68234, ......]   |


可以看到，collection_id = 113 的数据占比超过 68%，即该值在整个表中非常倾斜（skewed）。
PostgreSQL 的查询优化器会基于统计信息进行成本估算，认为对于高频值，顺序扫描比索引扫描更划算，因此放弃了索引。

这类问题本质是**数据分布倾斜 + 成本估算策略**导致的。

### 解决方案

核心思路:

   * **利用 ORDER BY 对成本计算的影响，迫使优化器选择索引扫描**。
   * 引入一个辅助列 rank_number，为每条记录生成一个随机数。
   * 新建包含 rank_number 的索引，并通过 ORDER BY rank_number 优化查询。


新的表结构:

```sql
create table collection
(
   collection_id         integer not null，
   entity_id             varchar(24) not null,
   status                text，
   created_at            timestamp with time zone,
   rank_number           integer not null,
   primary key (collection_id, entity_id)
);

create index idx_collection_collection_id_status_rank_number on collection (list_id, status, rank_number);
```

优化后的查询sql:

```sql
SELECT
   *
FROM
   collection
WHERE
   collection_id = 113 AND
   status = 'CREATED'
ORDER BY
   rank_number
LIMIT
   1
```

由于引入了 ORDER BY rank_number，优化器倾向使用索引扫描（因为排序成本在顺序扫描下非常高），从而避免全表扫描。

对于特定状态（如 CREATED），可以使用 部分索引（Partial Index），进一步减少索引体积：

```sql
create index
   idx_collection_collection_id_status_rank_number_where_status_created
on collection
   (list_id, rank_number)
where
   status = 'CREATED';
```


## 案例三：统计信息不准确导致索引选择异常

### 表结构

```sql
create table meeting_books
(
    book_id              integer primary key,
    user_id              text,
    meeting_id           text,
    created_at           timestamp with time zone,
);

create unique index idx_meeting_books_user_id_meeting_id
    on meeting_books (user_id, meeting_id);

create index idx_meeting_books_meeting_id_created_at
    on meeting_books (meeting_id, created_at);
```

### 业务背景

meeting_books 表记录了某个会议中给用户发放的书籍信息。
当前业务场景：查询某个用户在特定会议中领取的书籍。

### 问题复现和分析

业务sql如下:

```sql
SELECT
   *
FROM
   meeting_books
WHERE
   user_id = 'abc' AND meeting_id = 'T'
```

运行结果:

* 当 `WHERE user_id = 'abc' AND meeting_id = 'T'`, postgresql选择走`idx_meeting_books_meeting_id_created_at`索引，查询耗时大于1s，共扫描67W行数据。

* 当 `WHERE user_id = 'abc' AND meeting_id = 'A'`, postgresql选择走`idx_meeting_books_user_id_meeting_id`索引，查询耗时小于50ms，共扫描20行数据。


直接来查看 meeting_id 统计信息:

```sql
SELECT
   most_common_vals, most_common_freqs
FROM
   pg_stats
WHERE
   tablename = 'meeting_books' AND
   attname = 'meeting_id';
```

| most_common_vals | most_common_freqs   |
|---------------|---------|
| [A, B, C, D, E, F, G]            | [0.21, 0.19, 0.18, 0.14, 0.12, 0.09, 0.07]   |

可以看到:

* most_common_vals 只记录了 前 7 个频率最高的值（默认最多 100 个）。
* 这些频率加起来接近 1（100%），而 meeting_id = 'T' 并未出现在统计数据中。
* PostgreSQL 在 代价估算 时，认为 meeting_id = 'T' 是极低频值（甚至近似 0），导致 错误选择 idx_meeting_books_meeting_id_created_at。

根因：
**统计信息未及时更新**，导致优化器基于过时的分布信息做出错误的索引选择。

### 解决方案

```sql
VACUUM ANALYZE meeting_books;
```

执行后，统计信息会包含 meeting_id = 'T'，优化器重新选择合适的索引`idx_meeting_books_user_id_meeting_id`，查询性能恢复正常。

其他优化思路:

* 根据业务，在新的meeting举办后，且进行了一段时间后，手动analyze
* 调整适合`autovacuum_analyze_threshold`和`autovacuum_analyze_scale_factor`。如果表特别大(>1000W行),默认为(10%+50)=100W行数据发生改变，才会自动触发analyze，很容易出现新meeting的慢查询。
* 定时在业务低峰期进行vacuum + analyze


