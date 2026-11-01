## mysql

### 增删改查基本语句

> | 操作            | 关键字 | 基本语法模板                                   |
> | --------------- | ------ | ---------------------------------------------- |
> | **查** (Read)   | SELECT | SELECT 列名 FROM 表名 WHERE 条件;              |
> | **插** (Create) | INSERT | INSERT INTO 表名 (列1, 列2) VALUES (值1, 值2); |
> | **改** (Update) | UPDATE | UPDATE 表名 SET 列1 = 值1 WHERE 条件;          |
> | **删** (Delete) | DELETE | DELETE FROM 表名 WHERE 条件;                   |
>
> ```sql
> UPDATE users SET age = 21 WHERE id = 1;
> ```
>
> ```sql
> INSERT INTO users (name, age, email) VALUES 
> ('王五', 25, 'wangwu@example.com'),
> ('赵六', 30, 'zhaoliu@example.com');
> ```
>
> ```sql
> -- 查询姓“张”的用户（%代表任意多个字符）
> SELECT * FROM users WHERE name LIKE '张%';
> ```

### 建表

```sql
CREATE TABLE accounts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(20),
    money DECIMAL(10, 2)
);
```

### 单引号和反引号

> -- 正确写法：表名和字段名用【反引号】，具体的值用【单引号】 SELECT `desc` FROM `order` WHERE `user_name` = '张三';

- **单引号 '值'**：用来包裹**具体的数据**（如 '张三'）。
- **反引号 `字段名/表名`**：用来包裹**表名**或**列名**，防止和系统关键字冲突。

### 事务特性

| 特性  | 英文                     | 核心大白话解释                                               | 例子（张三给李四转 100 元）                                  |
| ----- | ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **A** | **原子性** (Atomicity)   | **要么全做，要么全不做。** 事务是最小单位，不能只执行一半。  | 张三扣 100 和李四加 100 必须都成功；只要李四加钱失败，张三扣的 100 必须退回。 |
| **C** | **一致性** (Consistency) | **数据必须符合业务规则，总量守恒。** 无论成功或失败，状态合法。 | 转账前两人总资金是 1000 元，转账后不管成没成，两人总资金依然必须是 1000 元。 |
| **I** | **隔离性** (Isolation)   | **并发操作互不干扰。** 多个事务同时执行时，互相看不见中间过程。 | 张三转账的事务还没 COMMIT 时，李四去查余额，看不见这笔正在转的钱。 |
| **D** | **持久性** (Durability)  | **一旦提交，不可逆转。** 即使数据库马上断电，数据也已永久写入硬盘。 | 只要收到 COMMIT 成功的提示，就算机房着火拔电源，重启后钱也是转成功的。 |

#### 提交

```sql
-- 1. 开启事务（也可以用 BEGIN;）
START TRANSACTION;

-- 2. 执行一系列 SQL 操作
-- 张三扣除 100 元
UPDATE accounts SET money = money - 100 WHERE name = '张三';

-- 李四增加 100 元
UPDATE accounts SET money = money + 100 WHERE name = '李四';

-- 3. 检查无误，提交事务（持久化到硬盘）
COMMIT;

-- 查询验证：张三 900，李四 1100，转账成功！
SELECT * FROM accounts;
```

#### 回滚

```sql
-- 1. 开启事务
START TRANSACTION;

-- 2. 张三扣除 100 元（此时张三在内存里变成了 900）
UPDATE accounts SET money = money - 100 WHERE name = '张三';

-- 假设这里发生了意外！
-- 比如：程序员把李四的名字手滑打错了，或者网络断了，导致第二步执行失败：
UPDATE accounts SET money = money + 100 WHERE name = '李四_错误的名字'; 

-- 3. 此时发现出现错误/受影响行数为0，放弃所有操作，执行回滚！
ROLLBACK;

-- 查询验证：张三依然是 1000，李四依然是 1000，之前扣掉的 100 元被“撤销”了！
SELECT * FROM accounts;
```

