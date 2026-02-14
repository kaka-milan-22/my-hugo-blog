---
title: "数据库ACID原理深度解析：从理论到实践的完整指南"
date: 2026-02-14T02:00:00+08:00
draft: false
tags: ["数据库", "ACID", "事务", "隔离级别", "MySQL", "PostgreSQL"]
categories: ["数据库技术", "后端开发"]
author: "Your Name"
description: "深入理解关系数据库的ACID特性：原子性、一致性、隔离性、持久性，以及开发中的最佳实践和常见陷阱"
toc: true
---

## 开场：那笔消失的转账

2015年，某互联网公司的数据库出了个灵异事件。

用户A转账1000元给用户B。但是：
- 用户A的账户扣了1000元 ✅
- 用户B的账户没有收到 ❌

1000元凭空消失了。

技术团队紧急排查，发现问题出在一个看似简单的转账操作：

```sql
-- 转账操作（错误示范）
UPDATE accounts SET balance = balance - 1000 WHERE user_id = 'A';
-- 💥 这里服务器突然断电了
UPDATE accounts SET balance = balance + 1000 WHERE user_id = 'B';
```

第一条SQL执行了，第二条没执行。钱就这么丢了。

如果他们用了**事务（Transaction）**和**ACID原则**，这个问题根本不会发生。

今天，我们就来聊聊ACID——关系数据库最核心的特性，它是如何保证你的数据安全的。

<!--more-->

---

## 第一章：什么是ACID？

ACID是四个英文单词的首字母缩写：

- **A**tomicity（原子性）
- **C**onsistency（一致性）
- **I**solation（隔离性）
- **D**urability（持久性）

这四个特性共同保证了数据库事务的可靠性。

### 什么是事务？

**事务（Transaction）** 是数据库的一组操作，要么全部成功，要么全部失败。

```sql
START TRANSACTION;  -- 开始事务

UPDATE accounts SET balance = balance - 1000 WHERE user_id = 'A';
UPDATE accounts SET balance = balance + 1000 WHERE user_id = 'B';

COMMIT;  -- 提交事务（全部生效）
-- 或
ROLLBACK;  -- 回滚事务（全部撤销）
```

事务就像一个**原子操作的容器**，保证一组操作的完整性。

---

## 第二章：A - 原子性（Atomicity）

### 定义：要么全做，要么全不做

原子性意味着事务中的所有写操作要么全部执行，要么全部不执行，不能只执行一部分。如果执行过程中出现故障，事务中的所有写操作都会被回滚。

```
┌─────────────────────────────────────┐
│  Transaction（事务）                 │
│                                     │
│  ① UPDATE accounts SET balance...   │
│  ② UPDATE accounts SET balance...   │
│  ③ INSERT INTO logs ...             │
│                                     │
│  要么全部成功 ✅                     │
│  要么全部失败 ❌（自动回滚）         │
└─────────────────────────────────────┘
```

### 实际例子：银行转账

**正确的转账操作：**

```sql
START TRANSACTION;

-- 扣款
UPDATE accounts SET balance = balance - 1000 
WHERE user_id = 'Alice' AND balance >= 1000;

-- 检查扣款是否成功
IF (affected_rows != 1) THEN
    ROLLBACK;  -- 余额不足，回滚
    RETURN 'Insufficient funds';
END IF;

-- 入账
UPDATE accounts SET balance = balance + 1000 
WHERE user_id = 'Bob';

-- 记录日志
INSERT INTO transfer_logs (from_user, to_user, amount, timestamp)
VALUES ('Alice', 'Bob', 1000, NOW());

COMMIT;  -- 全部成功，提交
```

**如果中途失败会怎样？**

```
场景1: 服务器断电（在扣款后、入账前）
→ 数据库重启后自动回滚未提交的事务
→ Alice的余额恢复原状
→ 钱没有丢失 ✅

场景2: 程序异常（在入账后、记录日志前）
→ ROLLBACK被调用
→ 扣款和入账都被撤销
→ 系统回到初始状态 ✅
```

### 原子性是如何实现的？

数据库用**预写日志（Write-Ahead Logging, WAL）**实现原子性：

```
1. 事务开始前，数据库在日志中记录"事务开始"
2. 每次修改前，先写入日志："准备修改X"
3. 修改数据
4. 如果成功，写入日志："事务提交"
5. 如果失败，根据日志回滚所有修改
```

**Undo Log（撤销日志）：**
```
Transaction 101 开始
  - 修改前：accounts(Alice).balance = 5000
  - 修改后：accounts(Alice).balance = 4000
  - 修改前：accounts(Bob).balance = 3000
  - 修改后：accounts(Bob).balance = 4000
Transaction 101 提交
```

如果事务失败，数据库读取Undo Log，反向执行所有操作。

---

## 第三章：C - 一致性（Consistency）

### 定义：保持数据库的完整性约束

一致性意味着保持数据库的不变性。事务写入的任何数据都必须符合所有定义的规则，并保持数据库处于良好状态。

**一致性确保：**
- 所有约束都被满足
- 数据符合业务规则
- 数据库从一个有效状态转换到另一个有效状态

### 例子1：账户总额不变

**业务规则：** 转账前后，所有账户总额应该相同。

```sql
-- 转账前
SELECT SUM(balance) FROM accounts;  -- 结果：10000

START TRANSACTION;
UPDATE accounts SET balance = balance - 1000 WHERE user_id = 'Alice';
UPDATE accounts SET balance = balance + 1000 WHERE user_id = 'Bob';
COMMIT;

-- 转账后
SELECT SUM(balance) FROM accounts;  -- 结果：10000（不变）✅
```

### 例子2：数据库约束

```sql
CREATE TABLE accounts (
    user_id VARCHAR(50) PRIMARY KEY,
    balance DECIMAL(10,2) NOT NULL,
    CONSTRAINT check_balance CHECK (balance >= 0)  -- 余额不能为负
);

-- 尝试非法操作
START TRANSACTION;
UPDATE accounts SET balance = balance - 5000 WHERE user_id = 'Alice';
-- ❌ 错误：CHECK constraint violated: balance >= 0
-- 事务自动回滚
```

### 例子3：外键约束

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id VARCHAR(50),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- 尝试创建订单给不存在的用户
INSERT INTO orders (order_id, user_id) VALUES (1, 'NonExistentUser');
-- ❌ 错误：Foreign key constraint violated
```

### 一致性的层次

**1. 数据库层面：**
- 主键约束
- 外键约束
- 唯一性约束
- CHECK约束

**2. 应用层面：**
- 业务逻辑规则
- 数据验证
- 状态机约束

**重要提醒：** 数据库只能保证数据库层面的一致性。应用层面的业务规则需要开发者自己保证。

---

## 第四章：I - 隔离性（Isolation）

### 定义：并发事务互不干扰

当有来自两个不同事务的并发写入时，这两个事务是相互隔离的。最严格的隔离是"可串行化"，即每个事务的行为就像它是数据库中唯一运行的事务一样。

隔离性是ACID中**最复杂**的部分，因为它涉及到并发控制。

### 为什么需要隔离？

想象这个场景：

```
时间线：
T1: Alice查余额：5000元
T2: Bob也查余额：5000元
T1: Alice转出1000元 → 余额变成4000元
T2: Bob也转出1000元 → 余额变成4000元？

结果：Alice和Bob都以为自己转账成功了，但实际上余额只扣了一次！
```

这就是**并发问题**。隔离性就是为了解决这类问题。

### 四种隔离级别

SQL标准定义了四种隔离级别，从弱到强：

```
┌─────────────────────┬──────────┬──────────┬──────────┐
│   隔离级别          │脏读      │不可重复读│幻读      │
├─────────────────────┼──────────┼──────────┼──────────┤
│ READ UNCOMMITTED    │ 可能 ❌  │ 可能 ❌  │ 可能 ❌  │
│ READ COMMITTED      │ 不可能✅ │ 可能 ❌  │ 可能 ❌  │
│ REPEATABLE READ     │ 不可能✅ │ 不可能✅ │ 可能 ❌  │
│ SERIALIZABLE        │ 不可能✅ │ 不可能✅ │ 不可能✅ │
└─────────────────────┴──────────┴──────────┴──────────┘
```

让我们逐个理解这些问题。

### 问题1：脏读（Dirty Read）

**定义：** 读到了其他事务未提交的数据。

```sql
-- 时间线
-- T1: 事务1                    -- T2: 事务2
START TRANSACTION;
                                START TRANSACTION;
UPDATE accounts 
SET balance = 10000 
WHERE user_id = 'Alice';
                                SELECT balance 
                                FROM accounts 
                                WHERE user_id = 'Alice';
                                -- 读到：10000（脏数据）
ROLLBACK;  -- 回滚了！
                                -- 但事务2已经用了10000这个数据做决策
                                COMMIT;
```

**危害：** 事务2基于一个"从未存在过"的数据做了决策。

**防止方法：** 使用READ COMMITTED或更高级别。

### 问题2：不可重复读（Non-Repeatable Read）

**定义：** 同一个事务内，两次读取同一数据，结果不一样。

```sql
-- T1: 事务1                    -- T2: 事务2
START TRANSACTION;
                                START TRANSACTION;
SELECT balance 
FROM accounts 
WHERE user_id = 'Alice';
-- 读到：5000

                                UPDATE accounts 
                                SET balance = 4000 
                                WHERE user_id = 'Alice';
                                COMMIT;

SELECT balance 
FROM accounts 
WHERE user_id = 'Alice';
-- 读到：4000（和第一次不一样！）

COMMIT;
```

**危害：** 同一个事务内，数据不一致，可能导致逻辑错误。

**防止方法：** 使用REPEATABLE READ或更高级别。

### 问题3：幻读（Phantom Read）

**定义：** 同一个事务内，两次查询，结果集的行数不一样。

```sql
-- T1: 事务1                    -- T2: 事务2
START TRANSACTION;
                                START TRANSACTION;
SELECT COUNT(*) 
FROM accounts 
WHERE balance > 1000;
-- 结果：5行

                                INSERT INTO accounts 
                                VALUES ('Charlie', 5000);
                                COMMIT;

SELECT COUNT(*) 
FROM accounts 
WHERE balance > 1000;
-- 结果：6行（多了一行"幽灵"）

COMMIT;
```

**危害：** 统计、报表可能不准确。

**防止方法：** 使用SERIALIZABLE级别。

### 隔离级别的选择

**READ UNCOMMITTED（读未提交）**
- **特点：** 最低隔离级别，性能最好
- **问题：** 脏读、不可重复读、幻读都可能发生
- **使用场景：** 几乎不用，数据一致性无法保证

**READ COMMITTED（读已提交）**
- **特点：** 只能读到已提交的数据
- **问题：** 不可重复读、幻读可能发生
- **使用场景：** 
  - 大部分业务的默认选择
  - 电商显示库存（可以容忍临时不一致）
  - 实时监控面板
- **数据库默认：** Oracle, PostgreSQL

**REPEATABLE READ（可重复读）**
- **特点：** 事务内多次读取结果一致
- **问题：** 幻读可能发生（MySQL InnoDB通过间隙锁避免了幻读）
- **使用场景：**
  - 需要数据一致性的业务
  - 报表生成
  - 数据分析
- **数据库默认：** MySQL InnoDB

**SERIALIZABLE（可串行化）**
- **特点：** 最高隔离级别，完全隔离
- **问题：** 性能最差，并发度最低
- **使用场景：**
  - 金融交易（绝对不能出错）
  - 关键业务操作
  - 数据一致性要求极高的场景

### 实际案例：库存扣减的并发问题

**问题代码：**

```python
def purchase_item(user_id, item_id, quantity):
    # 查询库存
    stock = db.query("SELECT stock FROM items WHERE item_id = ?", item_id)
    
    if stock >= quantity:
        # 扣减库存
        db.execute("UPDATE items SET stock = stock - ? WHERE item_id = ?", 
                   quantity, item_id)
        # 创建订单
        db.execute("INSERT INTO orders (user_id, item_id, quantity) VALUES (?, ?, ?)",
                   user_id, item_id, quantity)
        return "Success"
    else:
        return "Out of stock"
```

**并发场景：**
```
初始库存：10件
T1: 用户A购买9件 → 查到库存10 → 通过
T2: 用户B购买9件 → 查到库存10 → 通过
T1: 扣减库存 → 库存变成1
T2: 扣减库存 → 库存变成-8（超卖了！）
```

**正确做法1：使用事务 + 行锁**

```python
def purchase_item(user_id, item_id, quantity):
    with db.transaction():
        # FOR UPDATE 加行锁
        stock = db.query(
            "SELECT stock FROM items WHERE item_id = ? FOR UPDATE", 
            item_id
        )
        
        if stock >= quantity:
            db.execute("UPDATE items SET stock = stock - ? WHERE item_id = ?", 
                       quantity, item_id)
            db.execute("INSERT INTO orders (user_id, item_id, quantity) VALUES (?, ?, ?)",
                       user_id, item_id, quantity)
            return "Success"
        else:
            return "Out of stock"
```

**正确做法2：乐观锁（版本号）**

```python
def purchase_item(user_id, item_id, quantity):
    max_retries = 3
    for attempt in range(max_retries):
        # 查询库存和版本号
        item = db.query(
            "SELECT stock, version FROM items WHERE item_id = ?", 
            item_id
        )
        
        if item.stock >= quantity:
            # 使用版本号做乐观锁
            affected = db.execute(
                "UPDATE items SET stock = stock - ?, version = version + 1 "
                "WHERE item_id = ? AND version = ?", 
                quantity, item_id, item.version
            )
            
            if affected == 1:
                # 成功更新，创建订单
                db.execute("INSERT INTO orders ...")
                return "Success"
            else:
                # 版本号不匹配，重试
                continue
        else:
            return "Out of stock"
    
    return "Too many retries"
```

---

## 第五章：D - 持久性（Durability）

### 定义：提交后永久保存

持久性保证事务提交后，即使系统发生故障，数据也会被持久化。在分布式系统中，这意味着数据被复制到其他节点。

```
用户转账1000元
    ↓
事务COMMIT成功
    ↓
💥 服务器突然断电
    ↓
服务器重启
    ↓
数据依然存在 ✅（持久性保证）
```

### 持久性是如何实现的？

**1. Write-Ahead Logging (WAL)**

```
写数据前，先写日志：

1. 用户提交事务
2. 数据库先写Redo Log（重做日志）到磁盘
3. 日志写入成功后，返回"提交成功"
4. 后台慢慢地将数据写入数据文件
```

**为什么这样做？**
- 日志是**顺序写**，速度快
- 数据文件是**随机写**，速度慢
- 即使断电，可以用日志恢复数据

**2. 数据复制**

在分布式系统中：

```
主数据库（Primary）
    ↓ 复制
从数据库1（Replica 1）
    ↓ 复制
从数据库2（Replica 2）
```

数据被复制到多个节点，一个节点挂了，其他节点还有数据。

### MySQL的持久性保证

```sql
-- 查看持久性配置
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
```

**配置选项：**
- **0**：每秒刷新一次日志（性能最好，但可能丢失1秒数据）
- **1**：每次提交都刷新（最安全，但性能较差）⭐ 推荐
- **2**：每次提交写到OS缓存，每秒刷新到磁盘（折中方案）

**生产环境建议：**
```sql
SET GLOBAL innodb_flush_log_at_trx_commit = 1;  -- 保证持久性
SET GLOBAL sync_binlog = 1;  -- 保证binlog持久性
```

---

## 第六章：实战指南

### 1. 如何设置隔离级别

**MySQL：**

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置会话级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 设置全局级别（需要重启会话）
SET GLOBAL transaction_isolation = 'READ-COMMITTED';

-- 在配置文件中设置
[mysqld]
transaction-isolation = READ-COMMITTED
```

**PostgreSQL：**

```sql
-- 查看当前隔离级别
SHOW transaction_isolation;

-- 设置当前事务
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 设置会话级别
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### 2. 如何选择隔离级别？

**决策树：**

```
开始
  ↓
你的业务能容忍临时数据不一致吗？
  ├─ Yes → READ COMMITTED（大部分场景）
  │         例如：显示商品列表、用户评论
  │
  └─ No → 需要在事务内多次读取相同数据吗？
            ├─ Yes → REPEATABLE READ
            │         例如：生成报表、数据分析
            │
            └─ No → 是否是关键金融操作？
                      ├─ Yes → SERIALIZABLE
                      │         例如：转账、支付
                      │
                      └─ No → READ COMMITTED
```

### 3. 常见陷阱

**陷阱1：忘记开启事务**

```python
# ❌ 错误：没有事务保护
def transfer(from_user, to_user, amount):
    db.execute("UPDATE accounts SET balance = balance - ? WHERE user = ?", 
               amount, from_user)
    # 如果这里出错，钱就丢了
    db.execute("UPDATE accounts SET balance = balance + ? WHERE user = ?", 
               amount, to_user)

# ✅ 正确：使用事务
def transfer(from_user, to_user, amount):
    with db.transaction():
        db.execute("UPDATE accounts SET balance = balance - ? WHERE user = ?", 
                   amount, from_user)
        db.execute("UPDATE accounts SET balance = balance + ? WHERE user = ?", 
                   amount, to_user)
```

**陷阱2：长事务**

```python
# ❌ 错误：事务太长
with db.transaction():
    orders = db.query("SELECT * FROM orders WHERE status = 'pending'")
    for order in orders:
        process_order(order)  # 可能很慢
        send_email(order)     # 可能很慢
        update_inventory(order)
        # 事务一直没提交，锁一直持有
    # 提交

# ✅ 正确：缩短事务
orders = db.query("SELECT * FROM orders WHERE status = 'pending'")
for order in orders:
    process_order(order)
    send_email(order)
    # 每个订单一个事务
    with db.transaction():
        update_inventory(order)
```

**陷阱3：在事务中调用外部服务**

```python
# ❌ 错误：事务中调用外部API
with db.transaction():
    db.execute("INSERT INTO orders ...")
    result = call_payment_api(order)  # 外部API很慢
    if result.success:
        db.execute("UPDATE orders SET status = 'paid' ...")

# ✅ 正确：先调用外部API
result = call_payment_api(order)
with db.transaction():
    db.execute("INSERT INTO orders ...")
    if result.success:
        db.execute("UPDATE orders SET status = 'paid' ...")
```

**陷阱4：死锁**

```sql
-- T1                          -- T2
START TRANSACTION;              START TRANSACTION;
UPDATE A SET ...;               UPDATE B SET ...;
                                UPDATE A SET ...;  -- 等待T1释放A的锁
UPDATE B SET ...;               -- 等待T2释放B的锁
-- 💀 死锁！
```

**解决方案：**
1. 按相同顺序访问资源
2. 使用`NOWAIT`或`SKIP LOCKED`
3. 设置死锁超时时间
4. 监控死锁，优化代码

### 4. 性能优化建议

**优化1：使用合适的隔离级别**

```
不要盲目使用SERIALIZABLE
↓
大部分场景用READ COMMITTED
↓
性能提升20-50%
```

**优化2：减少事务范围**

```python
# Before: 慢
with db.transaction():
    data = fetch_external_data()  # 10秒
    db.execute("INSERT ...")       # 0.01秒

# After: 快
data = fetch_external_data()  # 在事务外
with db.transaction():
    db.execute("INSERT ...")  # 事务只持有0.01秒
```

**优化3：使用批量操作**

```python
# Before: 慢（1000个事务）
for item in items:
    with db.transaction():
        db.execute("INSERT INTO ...")

# After: 快（1个事务）
with db.transaction():
    for item in items:
        db.execute("INSERT INTO ...")
# 或使用批量插入
with db.transaction():
    db.execute_many("INSERT INTO ...", items)
```

---

## 第七章：监控与调试

### 1. 查看当前事务

**MySQL：**

```sql
-- 查看正在运行的事务
SELECT * FROM information_schema.innodb_trx;

-- 查看锁等待
SELECT * FROM performance_schema.data_locks;

-- 查看死锁历史
SHOW ENGINE INNODB STATUS;
```

**PostgreSQL：**

```sql
-- 查看当前活动
SELECT * FROM pg_stat_activity WHERE state = 'active';

-- 查看锁
SELECT * FROM pg_locks;

-- 查看等待
SELECT * FROM pg_stat_activity WHERE wait_event IS NOT NULL;
```

### 2. 日志分析

**慢查询日志：**

```sql
-- MySQL
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;  -- 超过1秒记录

-- 查看慢查询
SELECT * FROM mysql.slow_log;
```

### 3. 监控指标

**关键指标：**
- 事务吞吐量（TPS）
- 事务响应时间（P95, P99）
- 锁等待数量
- 死锁发生次数
- 事务回滚率

---

## 第八章：ACID vs BASE

### 为什么NoSQL用BASE？

NoSQL数据库（如MongoDB, Cassandra）通常不遵循ACID，而是遵循BASE：

- **B**asically **A**vailable（基本可用）
- **S**oft state（软状态）
- **E**ventually consistent（最终一致性）

**为什么？**

```
CAP定理：
分布式系统只能同时满足以下三个中的两个：
- Consistency（一致性）
- Availability（可用性）
- Partition tolerance（分区容错性）

ACID数据库选择：CP（一致性 + 分区容错）
BASE数据库选择：AP（可用性 + 分区容错）
```

**什么时候用NoSQL/BASE？**
- 社交媒体（点赞数可以延迟更新）
- 日志系统（可以容忍丢失少量日志）
- 缓存系统

**什么时候必须用ACID？**
- 金融交易
- 订单系统
- 库存管理
- 任何涉及钱的系统

---

## 尾声：理解权衡

回到开头的故事。那笔消失的1000元，最终是怎么找回来的？

技术团队加班三天三夜，从备份、日志、监控中拼凑出了真相，手动修复了数据。然后：

1. 所有转账操作改用事务
2. 隔离级别设置为REPEATABLE READ
3. 增加了事务监控和告警
4. 写了一套自动化测试，覆盖各种并发场景

**教训：**
- ACID不是理论，是生产环境的救命稻草
- 不要自作聪明地"优化"掉事务
- 了解你的数据库，了解你的隔离级别
- 测试并发场景，不要等到生产环境出问题

**记住：**

> 数据库的ACID特性，是用50年的工程实践换来的。  
> 不要轻易放弃它。

---

## 附录：快速参考

### ACID速查表

```
┌──────────┬─────────────────────────────────────┐
│ 原子性   │ 事务要么全做，要么全不做            │
│(Atomicity)│ 实现：Undo Log                     │
├──────────┼─────────────────────────────────────┤
│ 一致性   │ 数据符合所有约束和业务规则          │
│(Consistency)│ 实现：约束检查 + 应用逻辑        │
├──────────┼─────────────────────────────────────┤
│ 隔离性   │ 并发事务互不干扰                    │
│(Isolation)│ 实现：锁 + MVCC                    │
├──────────┼─────────────────────────────────────┤
│ 持久性   │ 提交后永久保存                      │
│(Durability)│ 实现：WAL + 复制                  │
└──────────┴─────────────────────────────────────┘
```

### 隔离级别速查

```
┌──────────────────┬──────────────────────┐
│ READ UNCOMMITTED │ 几乎不用（不安全）   │
├──────────────────┼──────────────────────┤
│ READ COMMITTED   │ 推荐（大部分场景）   │
├──────────────────┼──────────────────────┤
│ REPEATABLE READ  │ 报表、分析           │
├──────────────────┼──────────────────────┤
│ SERIALIZABLE     │ 金融、关键操作       │
└──────────────────┴──────────────────────┘
```

### 最佳实践

1. ✅ **始终使用事务**保护关键操作
2. ✅ **选择合适的隔离级别**（默认READ COMMITTED）
3. ✅ **缩短事务时间**（不要在事务中调用外部服务）
4. ✅ **监控事务性能**（TPS、锁等待、死锁）
5. ✅ **测试并发场景**（压力测试、混沌测试）
6. ❌ **不要使用自动提交**模式处理复杂操作
7. ❌ **不要在循环中开启独立事务**（使用批量操作）
8. ❌ **不要忽略死锁**（添加重试逻辑）

---

## 推荐资源

**书籍：**
- 《设计数据密集型应用》（Designing Data-Intensive Applications）
- 《数据库系统概念》（Database System Concepts）
- 《高性能MySQL》

**在线资源：**
- MySQL官方文档：Transaction Isolation Levels
- PostgreSQL官方文档：Transaction Isolation
- ByteByteGo：Database and Storage Guides

**工具：**
- pt-deadlock-logger：死锁分析
- mysqltuner：性能分析
- pgAdmin：PostgreSQL管理

---

*下一篇，我们聊聊分布式事务：当数据分散在多个数据库时，如何保证ACID？*

*如果你对某个主题感兴趣（如MVCC原理、分布式锁、Saga模式），欢迎留言。*
