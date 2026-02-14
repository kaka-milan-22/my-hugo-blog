---
title: "数据库锁与MVCC：从入门到精通的完整逻辑链"
date: 2026-02-14T12:30:00+08:00
draft: false
tags: ["数据库", "锁", "MVCC", "并发控制", "MySQL"]
categories: ["数据库技术", "并发编程"]
author: "Kaka"
description: "为什么数据库需要锁？表锁、行锁、MVCC到底是什么？一篇文章理清数据库并发控制的完整逻辑"
toc: true
---

## 开场：一个让人抓狂的BUG

2016年，我刚做运维的时候，遇到了一个诡异的问题。

某个下单接口，压测的时候发现：
- 单用户测试：响应时间50ms，一切正常 ✅
- 100并发测试：响应时间5000ms，慢了100倍 ❌

后端开发说："我的代码没问题啊，为什么并发就这么慢？"

我看了看监控，CPU不高，内存不高，磁盘IO也不高。那问题在哪？

直到我打开MySQL的慢查询日志，看到了一堆`Waiting for table lock`...

原来，100个请求在**排队等锁**。就像100个人在一个单人厕所门口排队，当然慢。

那天，我第一次意识到：**数据库的锁，是性能杀手，也是数据安全的保护神。**

今天，让我用最简单的逻辑，给你讲清楚数据库的锁和MVCC。

<!--more-->

---

## 第一章：为什么需要锁？——问题的起源

### 场景：两个人同时改同一条数据

想象一个最简单的场景：

```
初始状态：商品库存 = 10

时间轴：
10:00:00.000  用户A查询库存：SELECT stock FROM products WHERE id=1
10:00:00.001  用户B查询库存：SELECT stock FROM products WHERE id=1
              两人都看到：stock = 10

10:00:00.100  用户A购买1个：UPDATE products SET stock = 9 WHERE id=1
10:00:00.101  用户B购买1个：UPDATE products SET stock = 9 WHERE id=1

结果：库存变成9（应该是8！）
```

这就是**并发问题**——多个操作同时进行，导致数据不一致。

### 核心矛盾：性能 vs 正确性

数据库面临一个两难选择：

```
方案1：串行执行（一个接一个）
  ✅ 数据绝对正确
  ❌ 性能极差（并发能力为1）

方案2：完全并行执行（随便执行）
  ✅ 性能极好
  ❌ 数据可能错乱

理想方案：在保证正确性的前提下，尽可能并行
  → 这就是为什么需要"锁"
```

**锁的本质：控制并发访问，保证数据正确性。**

---

## 第二章：表锁——最简单粗暴的方案

### 什么是表锁？

**表锁：锁住整张表，只允许一个人操作。**

```
┌─────────────────────────┐
│   products 表           │
│  ┌─────────────────┐    │
│  │  🔒 表锁        │    │ ← 整张表被锁住
│  ├─────────────────┤    │
│  │ id | stock      │    │
│  │ 1  | 10         │    │
│  │ 2  | 20         │    │
│  │ 3  | 30         │    │
│  └─────────────────┘    │
└─────────────────────────┘

用户A正在操作 → 用户B等待 → 用户C等待 → ...
```

### 表锁的使用

**MySQL示例：**

```sql
-- 加表锁（读锁）
LOCK TABLES products READ;
SELECT * FROM products;  -- 可以读
-- 其他会话也可以读，但不能写
UNLOCK TABLES;

-- 加表锁（写锁）
LOCK TABLES products WRITE;
UPDATE products SET stock = stock - 1 WHERE id = 1;
-- 其他会话既不能读也不能写，只能等待
UNLOCK TABLES;
```

### 表锁的问题

**问题1：并发能力极差**

```
100个用户同时访问：
用户1：占用表锁，操作中...（100ms）
用户2：等待...
用户3：等待...
...
用户100：等待...

总耗时 = 100个用户 × 100ms = 10000ms = 10秒
```

即使100个用户操作的是不同的商品，也要排队。这太蠢了。

**问题2：容易死锁**

```sql
-- 会话1
LOCK TABLES products WRITE, orders WRITE;

-- 会话2（同时执行）
LOCK TABLES orders WRITE, products WRITE;

-- 💀 死锁：会话1等orders，会话2等products
```

**问题3：锁粒度太大**

100个用户访问100个不同的商品，理论上应该可以并行，但表锁让它们串行了。

### 什么时候用表锁？

**适用场景：**
- 批量导入/导出数据
- 表结构变更（ALTER TABLE）
- 全表统计（COUNT、SUM）

**不适用场景：**
- 高并发的OLTP系统（在线交易处理）
- 只操作少数几行的场景

**结论：表锁太粗糙，我们需要更细粒度的锁。**

---

## 第三章：行锁——精确打击

### 什么是行锁？

**行锁：只锁住需要操作的那一行，其他行不受影响。**

```
┌─────────────────────────┐
│   products 表           │
│  ┌─────────────────┐    │
│  │ id | stock      │    │
│  │ 1  | 10   🔒   │    │ ← 只锁这一行
│  │ 2  | 20         │    │ ← 其他行可以并发访问
│  │ 3  | 30         │    │
│  └─────────────────┘    │
└─────────────────────────┘

用户A操作id=1 → 用户B操作id=2（同时进行，互不影响）
```

### 行锁的使用

**InnoDB默认使用行锁：**

```sql
START TRANSACTION;

-- 查询并锁行（悲观锁）
SELECT * FROM products WHERE id = 1 FOR UPDATE;
-- 此时，id=1这行被锁住，其他事务无法修改

-- 更新
UPDATE products SET stock = stock - 1 WHERE id = 1;

COMMIT;  -- 释放锁
```

### 行锁的优势

**并发能力大幅提升：**

```
场景：100个用户访问100个不同商品
表锁：串行执行，总耗时 = 100 × 100ms = 10秒
行锁：并行执行，总耗时 ≈ 100ms

性能提升100倍！
```

### 行锁的类型

**1. 共享锁（S锁，Shared Lock）**

也叫**读锁**。

```sql
SELECT * FROM products WHERE id = 1 LOCK IN SHARE MODE;
```

- 多个事务可以同时持有S锁（可以同时读）
- 持有S锁时，其他事务不能加X锁（不能写）

**2. 排他锁（X锁，Exclusive Lock）**

也叫**写锁**。

```sql
SELECT * FROM products WHERE id = 1 FOR UPDATE;
-- 或直接UPDATE/DELETE
```

- 只有一个事务可以持有X锁
- 持有X锁时，其他事务既不能读也不能写

**锁的兼容性：**

```
┌──────────┬────────┬────────┐
│   已持有  │  S锁   │  X锁   │
├──────────┼────────┼────────┤
│ 请求S锁  │  ✅    │  ❌    │
│ 请求X锁  │  ❌    │  ❌    │
└──────────┴────────┴────────┘
```

简单记忆：
- 读-读可以并行 ✅
- 读-写互斥 ❌
- 写-写互斥 ❌

---

## 第四章：行锁的陷阱——锁升级与索引

### 陷阱1：没有索引 → 行锁退化成表锁

这是最坑的一个陷阱。

```sql
-- 假设stock列没有索引
UPDATE products SET stock = stock - 1 WHERE stock = 10;
```

**你以为：** 只锁stock=10的那几行  
**实际上：** 锁住了整张表！

**原因：**
- 没有索引，InnoDB无法精确定位行
- 只能全表扫描
- 为了保证正确性，锁住了扫描过的所有行
- 结果就是整张表都被锁了

**解决方案：**

```sql
-- 添加索引
CREATE INDEX idx_stock ON products(stock);

-- 现在只会锁stock=10的行
UPDATE products SET stock = stock - 1 WHERE stock = 10;
```

### 陷阱2：范围查询锁住了不该锁的行

```sql
-- 假设id是主键（有索引）
UPDATE products SET stock = stock - 1 WHERE id > 5 AND id < 10;
```

**你以为：** 只锁id=6,7,8,9这4行  
**实际上：** 可能锁了id=5到id=10之间的所有行，包括不存在的行

这就是**间隙锁（Gap Lock）**。

**为什么需要间隙锁？**

防止**幻读**：

```sql
-- 事务1
START TRANSACTION;
SELECT * FROM products WHERE id > 5 AND id < 10;
-- 结果：3行（id=6,7,8）

-- 事务2（同时）
INSERT INTO products (id, stock) VALUES (7, 100);
COMMIT;

-- 事务1再次查询
SELECT * FROM products WHERE id > 5 AND id < 10;
-- 结果：4行（多了一个幽灵行！）
```

间隙锁锁住了(5, 10)这个区间，防止其他事务插入新行。

### 陷阱3：死锁

```sql
-- 事务1
START TRANSACTION;
UPDATE products SET stock = stock - 1 WHERE id = 1;
-- 等待1秒
UPDATE products SET stock = stock - 1 WHERE id = 2;

-- 事务2（同时）
START TRANSACTION;
UPDATE products SET stock = stock - 1 WHERE id = 2;
-- 等待1秒
UPDATE products SET stock = stock - 1 WHERE id = 1;

-- 💀 死锁！
-- 事务1持有id=1的锁，等待id=2
-- 事务2持有id=2的锁，等待id=1
```

**MySQL的处理：**
- 自动检测死锁
- 回滚其中一个事务
- 返回错误：`Deadlock found when trying to get lock`

**如何避免死锁？**
1. 按相同顺序访问资源
2. 缩短事务时间
3. 使用乐观锁代替悲观锁

---

## 第五章：MVCC——不用锁也能并发

### 问题：读写冲突

即使有了行锁，还有一个问题：

```
时间线：
用户A：START TRANSACTION（准备生成报表，需要读很多数据）
用户A：SELECT * FROM products;  -- 读操作，加S锁

用户B：UPDATE products SET stock = stock - 1 WHERE id = 1;
       -- 写操作，需要X锁，但被A的S锁阻塞
       -- 等待...等待...等待...

用户A：（10分钟后）COMMIT;  -- 终于释放S锁

用户B：终于可以写了（等了10分钟）
```

读操作阻塞了写操作，性能很差。

**理想情况：**
- 读操作不应该阻塞写操作
- 写操作不应该阻塞读操作
- 读和写应该可以并行

**这就是MVCC要解决的问题。**

### MVCC是什么？

**MVCC（Multi-Version Concurrency Control，多版本并发控制）**

核心思想：**一份数据，保存多个版本。**

```
同一行数据的多个版本：

版本1（旧版本）: stock = 10  ← 事务A读这个版本
版本2（新版本）: stock = 9   ← 事务B写这个版本

读和写互不干扰！
```

### MVCC的实现原理

InnoDB在每行记录后面添加了三个隐藏列：

**1. DB_TRX_ID（6字节）**
- 最后修改这行的事务ID

**2. DB_ROLL_PTR（7字节）**
- 指向Undo Log的指针（旧版本的数据）

**3. DB_ROW_ID（6字节）**
- 行ID（如果表没有主键）

**示例：**

```
当前数据：
┌────┬────────┬───────────┬──────────────┐
│ id │ stock  │ DB_TRX_ID │ DB_ROLL_PTR  │
├────┼────────┼───────────┼──────────────┤
│ 1  │ 8      │ 103       │ → Undo Log   │
└────┴────────┴───────────┴──────────────┘

Undo Log（旧版本链）：
版本103: stock = 8  (当前版本)
         ↓
版本102: stock = 9  (上一个版本)
         ↓
版本101: stock = 10 (更早的版本)
```

### Read View：快照

当一个事务开始时，InnoDB会创建一个**Read View（读视图）**，记录：

- 当前活跃的事务ID列表
- 当前最大的事务ID

**可见性判断：**

```python
def is_visible(row_trx_id, read_view):
    # 1. 如果数据是我自己修改的，可见
    if row_trx_id == read_view.current_trx_id:
        return True
    
    # 2. 如果数据在我开始前已提交，可见
    if row_trx_id < read_view.min_trx_id:
        return True
    
    # 3. 如果数据是在我开始后才创建的，不可见
    if row_trx_id >= read_view.max_trx_id:
        return False
    
    # 4. 如果数据的事务在我开始时还未提交，不可见
    if row_trx_id in read_view.active_trx_ids:
        return False
    
    # 5. 其他情况，可见
    return True
```

**如果当前版本不可见，就沿着Undo Log链往前找，直到找到可见的版本。**

### MVCC的实际效果

**场景：生成报表 vs 下单**

```
时间线：

10:00:00  事务A（TRX_ID=100）开始，生成报表
          Read View = {min: 100, max: 101, active: [100]}
          
10:00:01  事务A：SELECT * FROM products WHERE id = 1;
          读到：stock = 10（版本99的数据）

10:00:02  事务B（TRX_ID=101）开始，下单
          UPDATE products SET stock = 9 WHERE id = 1;
          COMMIT;
          写入新版本（版本101）

10:00:03  事务A：SELECT * FROM products WHERE id = 1;
          读到：stock = 10（还是版本99！）
          因为版本101在事务A的Read View之后，不可见

10:05:00  事务A：COMMIT;
```

**关键点：**
- 事务A始终读到一致的数据（stock=10）
- 事务B的写操作没有被阻塞
- **读和写完全并行，互不影响**

---

## 第六章：快照读 vs 当前读

### 快照读（Snapshot Read）

使用MVCC机制，读历史版本：

```sql
-- 普通的SELECT就是快照读
SELECT * FROM products WHERE id = 1;
```

**特点：**
- 不加锁
- 读的是快照（历史版本）
- 高性能，高并发

### 当前读（Current Read）

读最新版本，并加锁：

```sql
-- 以下都是当前读：
SELECT * FROM products WHERE id = 1 FOR UPDATE;  -- 加X锁
SELECT * FROM products WHERE id = 1 LOCK IN SHARE MODE;  -- 加S锁
UPDATE products SET stock = stock - 1 WHERE id = 1;  -- 加X锁
DELETE FROM products WHERE id = 1;  -- 加X锁
INSERT INTO products VALUES (1, 10);  -- 加X锁
```

**特点：**
- 加锁
- 读的是最新版本
- 会阻塞其他写操作

### 什么时候用哪个？

**快照读：**
- 报表生成
- 数据分析
- 统计查询
- 对实时性要求不高的场景

**当前读：**
- 库存扣减（必须读最新值）
- 余额扣款（必须读最新值）
- 任何需要"检查后更新"的场景

---

## 第七章：完整的逻辑链——从问题到方案

让我们把整个逻辑链串起来：

### 第一层：为什么需要并发控制？

```
问题：多个用户同时修改数据 → 数据不一致
方案：需要某种机制控制并发访问
```

### 第二层：为什么需要锁？

```
问题：如何控制并发访问？
方案1：完全串行 → 性能太差
方案2：使用"锁"来协调访问
```

### 第三层：为什么从表锁到行锁？

```
问题：表锁并发能力太差
原因：锁粒度太大，100个用户访问100行，也要排队
方案：使用行锁，只锁需要的行
```

### 第四层：为什么需要MVCC？

```
问题：即使有行锁，读写还是互斥的
场景：生成报表（读）阻塞了下单（写）
方案：MVCC让读写并行
原理：保存多个版本，读旧版本，写新版本
```

### 第五层：完整的并发控制策略

```
┌─────────────────────────────────────────┐
│          并发控制策略                    │
├─────────────────────────────────────────┤
│                                         │
│  普通查询（SELECT）                      │
│    → 快照读 (MVCC)                      │
│    → 不加锁，读历史版本                  │
│    → 高性能，高并发                      │
│                                         │
│  需要最新数据（SELECT FOR UPDATE）       │
│    → 当前读                              │
│    → 加行锁（X锁或S锁）                  │
│    → 读最新版本                          │
│                                         │
│  写操作（UPDATE/DELETE/INSERT）          │
│    → 当前读 + 加X锁                     │
│    → 阻塞其他写操作                      │
│                                         │
└─────────────────────────────────────────┘
```

---

## 第八章：实战指南

### 1. 如何查看锁信息

**MySQL 5.7+：**

```sql
-- 查看当前锁
SELECT * FROM performance_schema.data_locks;

-- 查看锁等待
SELECT * FROM performance_schema.data_lock_waits;

-- 查看事务
SELECT * FROM information_schema.innodb_trx;
```

**实用查询：找出被阻塞的SQL**

```sql
SELECT 
    r.trx_id AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query
FROM information_schema.innodb_lock_waits w
INNER JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
INNER JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
```

### 2. 如何避免锁等待

**策略1：缩短事务时间**

```python
# ❌ 不好：事务太长
with db.transaction():
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    send_email(user.email)  # 慢！可能需要5秒
    update_logs(user.id)    # 慢！
    db.execute("UPDATE users SET last_login = NOW() WHERE id = ?", user_id)

# ✅ 好：缩短事务
user = db.query("SELECT * FROM users WHERE id = ?", user_id)
send_email(user.email)  # 在事务外
update_logs(user.id)    # 在事务外

with db.transaction():
    db.execute("UPDATE users SET last_login = NOW() WHERE id = ?", user_id)
```

**策略2：避免在事务中进行用户交互**

```python
# ❌ 不好：等待用户输入
with db.transaction():
    order = create_order(user_id, items)
    payment_url = request_payment_url(order)
    print("请访问：", payment_url)
    input("支付完成后按回车...")  # 锁一直持有！
    confirm_order(order)

# ✅ 好：分成多个事务
order = create_order(user_id, items)
payment_url = request_payment_url(order)
# 用户支付...
confirm_order(order)  # 新的事务
```

**策略3：使用乐观锁代替悲观锁**

```sql
-- 悲观锁：SELECT FOR UPDATE
START TRANSACTION;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;  -- 立即加锁
-- 其他事务必须等待
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;

-- 乐观锁：版本号
-- 不加锁，在UPDATE时检查版本号
UPDATE products 
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;  -- 如果version不是5，更新失败

-- 如果失败，重试
```

### 3. 如何监控锁问题

**关键指标：**

```sql
-- 1. 锁等待次数
SHOW STATUS LIKE 'Innodb_row_lock_waits';

-- 2. 平均锁等待时间
SHOW STATUS LIKE 'Innodb_row_lock_time_avg';

-- 3. 最长锁等待时间
SHOW STATUS LIKE 'Innodb_row_lock_time_max';

-- 4. 死锁次数
SHOW ENGINE INNODB STATUS;  -- 查看LATEST DETECTED DEADLOCK部分
```

**告警阈值建议：**
- 锁等待时间 > 1秒 → 警告
- 锁等待时间 > 5秒 → 严重
- 死锁发生 → 立即通知

---

## 第九章：常见问题FAQ

### Q1: 为什么我的UPDATE这么慢？

**可能原因：**
1. 没有索引，行锁退化成表锁
2. 更新的行太多，锁冲突严重
3. 长事务持有锁不释放
4. 死锁导致事务被回滚

**排查步骤：**
```sql
-- 1. 查看慢查询
SHOW PROCESSLIST;

-- 2. 查看锁等待
SELECT * FROM information_schema.innodb_trx WHERE trx_state = 'LOCK WAIT';

-- 3. 查看是否有表锁
SHOW OPEN TABLES WHERE In_use > 0;

-- 4. 查看是否有索引
EXPLAIN UPDATE products SET stock = stock - 1 WHERE name = 'iPhone';
```

### Q2: 快照读会读到脏数据吗？

**不会。**

快照读读的是**已提交的历史版本**，不会读到未提交的数据（脏读）。

但是，在同一个事务内，快照读读到的数据可能不是最新的（这是设计如此）。

### Q3: REPEATABLE READ能防止幻读吗？

**SQL标准：** 不能防止  
**MySQL InnoDB：** 能防止（通过间隙锁）

```sql
-- MySQL InnoDB在REPEATABLE READ下
START TRANSACTION;
SELECT * FROM products WHERE id > 5 AND id < 10;
-- 加了间隙锁，其他事务无法插入id在(5,10)之间的行
```

但如果是当前读，可能还是会有幻读。

### Q4: 乐观锁 vs 悲观锁，选哪个？

**悲观锁（SELECT FOR UPDATE）：**
- 适合：写多读少、冲突概率高
- 优点：不需要重试
- 缺点：会阻塞其他事务

**乐观锁（版本号）：**
- 适合：读多写少、冲突概率低
- 优点：不阻塞，性能好
- 缺点：需要处理重试逻辑

**经验法则：**
- 库存扣减（冲突高）→ 悲观锁
- 文章编辑（冲突低）→ 乐观锁

---

## 第十章：总结——完整的知识地图

```
并发问题的演进：

问题层1：并发修改导致数据不一致
    ↓
解决方案1：表锁
    优点：简单，保证正确性
    缺点：并发能力差
    ↓
问题层2：表锁太粗糙，影响性能
    ↓
解决方案2：行锁
    优点：并发能力大幅提升
    缺点：读写还是互斥
    ↓
问题层3：读操作阻塞写操作
    ↓
解决方案3：MVCC
    优点：读写完全并行
    原理：多版本，读旧版本
    ↓
最终方案：行锁 + MVCC
    普通SELECT → MVCC（快照读）
    SELECT FOR UPDATE → 行锁（当前读）
    UPDATE/DELETE → 行锁（当前读）
```

### 核心要点

**1. 锁的本质**
- 协调并发访问
- 在性能和正确性之间权衡

**2. 锁的粒度**
- 表锁 → 行锁 → 更细粒度
- 粒度越细，并发能力越强

**3. MVCC的价值**
- 让读不阻塞写，写不阻塞读
- 代价是需要额外的存储空间（多版本）

**4. 实战建议**
- 优先使用行锁（确保有索引）
- 大部分查询用快照读（不加锁）
- 只在必要时用当前读（FOR UPDATE）
- 缩短事务时间
- 监控锁等待和死锁

---

## 尾声：从抓狂到从容

回到开头的故事。那个慢了100倍的接口，最终是怎么解决的？

我们发现问题出在一个没有索引的WHERE条件，导致行锁退化成了表锁。100个并发请求，变成了100个串行请求。

**解决方案：**
```sql
-- 添加索引
ALTER TABLE orders ADD INDEX idx_user_status (user_id, status);

-- 性能从5000ms降到50ms
-- 问题解决 ✅
```

从那以后，我学会了：
- 锁不是敌人，它是并发控制的工具
- 理解锁的原理，才能写出高性能的代码
- 监控锁的状态，才能及时发现问题

**数据库的锁和MVCC，看起来复杂，其实就是在解决一个问题：**

> 如何让多个人安全地同时操作同一份数据？

理解了这个逻辑链，一切就都清楚了。

---

## 附录：快速参考

### 锁的类型

```
表锁
  ├─ 读锁（LOCK TABLES ... READ）
  └─ 写锁（LOCK TABLES ... WRITE）

行锁（InnoDB）
  ├─ 共享锁/S锁/读锁（LOCK IN SHARE MODE）
  ├─ 排他锁/X锁/写锁（FOR UPDATE）
  ├─ 间隙锁（Gap Lock）
  └─ 临键锁（Next-Key Lock）
```

### SQL速查

```sql
-- 查看锁
SELECT * FROM performance_schema.data_locks;

-- 查看锁等待
SELECT * FROM performance_schema.data_lock_waits;

-- 查看事务
SELECT * FROM information_schema.innodb_trx;

-- 查看死锁日志
SHOW ENGINE INNODB STATUS;

-- 手动加锁
SELECT * FROM products WHERE id = 1 FOR UPDATE;  -- X锁
SELECT * FROM products WHERE id = 1 LOCK IN SHARE MODE;  -- S锁

-- 设置锁等待超时
SET innodb_lock_wait_timeout = 50;  -- 默认50秒
```

### 最佳实践

1. ✅ 始终在WHERE条件字段上建立索引
2. ✅ 使用主键或唯一索引访问数据
3. ✅ 缩短事务时间
4. ✅ 避免大事务
5. ✅ 按相同顺序访问资源（避免死锁）
6. ✅ 大部分SELECT不加锁（使用MVCC）
7. ❌ 不要在事务中调用外部服务
8. ❌ 不要在事务中等待用户输入
9. ❌ 不要在范围查询上使用FOR UPDATE（除非必要）

---

## 推荐资源

**书籍：**
- 《高性能MySQL》第三版，第1章和第7章
- 《MySQL技术内幕：InnoDB存储引擎》第6章

**文档：**
- MySQL官方文档：InnoDB Locking
- MySQL官方文档：InnoDB Multi-Versioning

**工具：**
- pt-deadlock-logger：监控死锁
- innotop：实时监控InnoDB
- mysqldumpslow：分析慢查询

---

*下一篇，我们聊聊分布式系统中的锁：Redis分布式锁、ZooKeeper、etcd的实现原理。*

*如果你对某个主题感兴趣（如两阶段锁协议、间隙锁细节、MVCC的Undo Log实现），欢迎留言。*
