# 1. Trigger 触发器

## 1.1 创建

定义：a block of SQL code, 在select, insert, update, delete语句之前或之后自动执行

作用：保证数据一致性

举例：在payments表中insert一条记录，invoices表中payment_total列的值应当相应改变

```s
DELIMITER $$

CREATE TRIGGER payments_after_insert  -- table_after/before_statement
  AFTER INSERT ON payments
  FOR EACH ROW -- 对于插入的每一行，都要fire一次trigger（for MySQL）
BEGIN
  UPDATE invoices
  SET payment_total = payment_total + NEW.amount
  WHERE invoice_id = NEW.invoice_id;
END$$

DELIMITER ;
```

NEW: 用于update语句，返回刚刚insert的行的值; 通过.column来取对应列的值

OLD: 用于update/delete语句，返回旧的行的值

注意：在trigger中, 可以modify除了trigger依赖的表之外的所有表

```s
INSERT INTO payments
VALUES (DEFAULT, 5, 3, '2019-01-01', 10, 1)
```

```s
DELIMITER $$

CREATE TRIGGER payments_after_delete
  AFTER DELETE ON payments  -- 不要忘记ON
  FOR EACH ROW
BEGIN
  UPDATE invoices
  SET payment_total = payment_total - OLD.amount
  WHERE invoice_id = OLD.invoice_id;
END$$

DELIMITER ;
```

```s
DELETE
FROM payments
WHERE payment_id = 9
```

## 1.2 查询当前数据库中创建的Trigger

```s
SHOW TRIGGERS
```

```s
SHOW TRIGGERS LIKE 'payments%'  -- 对trigger的name进行筛选
```

## 1.3 删除

```s
DROP TRIGGER IF EXISTS payments_after_insert;
```

## 1.4 另一作用：audit, 稽查

举例：当insert/delete记录时，参看who made what changes when

- 创建表
note: 这个表没有主键
```s
USE sql_invoicing;

CREATE TABLE payments_audit
(
	client_id     INT               NOT NULL,
    date          DATE              NOT NULL,
    amount        DECIMAL(9, 2)     NOT NULL,
    action_type   VARCHAR(50)       NOT NULL,
    action_date   DATETIME          NOT NULL
)
```s

- 记录insert操作
```s
DROP TRIGGER IF EXISTS payments_after_insert;

DELIMITER $$

CREATE TRIGGER payments_after_insert
  AFTER INSERT ON payments
  FOR EACH ROW
BEGIN
  UPDATE invoices
  SET payment_total = payment_total + NEW.amount
  WHERE invoice_id = NEW.invoice_id;
  
  INSERT INTO payments_audit
  VALUES (NEW.client_id, NEW.date, NEW.amount, 'Insert', NOW());
END$$

DELIMITER ;
```

- 记录delete操作
```s
DROP TRIGGER IF EXISTS payments_after_delete;

DELIMITER $$

CREATE TRIGGER payments_after_delete
  AFTER DELETE ON payments  -- 不要忘记ON
  FOR EACH ROW
BEGIN
  UPDATE invoices
  SET payment_total = payment_total - OLD.amount
  WHERE invoice_id = OLD.invoice_id;
  
  INSERT INTO payments_audit
  VALUES (OLD.client_id, OLD.date, OLD.amount, 'Delete', NOW());
END$$

DELIMITER ;
```

```s
INSERT INTO payments
VALUES (default, 5, 3, '2019-01-01', 10, 1);

DELETE FROM payments
WHERE payment_id = 9
```

在现实中，不希望创建很多分离的表，例如payments_audit，来log changes；
会使用general structure for logging changes (general audit tables)

# 2. Events 事件

定义：a block of SQL code, gets executed according to a schedule, 例如, 每天早上10点/once a month

用法：自动完成数据库维护任务（database maintenance tasks），例如，delete data、copy data、aggregate data

## 2.1 turn on MySQL even scheduler 调度程序

查看所有的系统变量
```s
SHOW VARIABLES;
```
查看 event scheduler variable
```s
SHOW VARIABLES LIKE 'event%';
```

- use event
run the background process: constantly looks for events to execute
```s
SET GLOBAL event_scheduler = ON
```

- save system resources
```s
SET GLOBAL event_scheduler = OFF
```

## 2.2 创建

```s
DELIMITER $$

CREATE EVENT yearly_delete_stale_audit_rows
ON SCHEDULE
    -- AT ‘2019-05-01’  -- once
    EVERY 1 YEAR STARTS '2019-01-01' ENDS '2029-01-01' -- on a regular basis
DO BEGIN
    DELETE FROM payments_audit
    WHERE action_date < NOW() - INTERVAL 1 YEAR;  -- 删除older than one year 的记录
    
    -- DATEADD(NOW(), INTERVAL -1 YEAR)
    -- DATESUB(NOW(), INTERVAL 1 YEAR)
END$$

DELIMITER ;
```

- 命名
  - interval_statement_otherthing- 
   - interval: hourly, daily, monthly, yearly, once(event 仅仅 trigger 一次)

# 2.3 查看events

```s
SHOW EVENTS
```

查看每年执行一次的events
```s
SHOW EVENTS LIKE 'yarly%';
```

# 2.4 删除

```s
DROP EVENT IF EXISTS yearly_delete_stale_audit_rows;
```

# 2.5 更改

<=> drop + recreate

- change the schedule
- change the SQL statement
- 暂时的enable or sisable an event

```s
ALTER EVENT yearly_delete_stale_audit_rows DISABLE;
ALTER EVENT yearly_delete_stale_audit_rows ENABLE;
```

# 3. Transaction 事务

## 3.1 简介

定义：a group of SQL statements that represent a single unit of work

用途：make multiple changes to the database, 希望所有的change都成功，否则“全部”失败as a unit，以保证数据的一致性

特点：
 - atomicity: 单元性，work as a unit，如果一个语句执行错误，roll back
 - consistency: 一致性，不会出现有一笔order没有item的情况
 - isolation: 隔离性，作用于相同数据的事务互不干扰（有先后执行顺序）
 - durablity: 事务形成的改变是永久的

## 3.2 创建

- commit 交付
```s
USE sql_store;

START TRANSACTION;

INSERT INTO orders (customer_id, order_date, status)
VALUES (1, '2019-01-01', 1);

INSERT INTO order_items
VALUES (LAST_INSERT_ID(), 1, 1, 1);  -- 返回insert order的id

COMMIT;  -- MySQL write all the changes to the database; if one of the changes fails, the transaction is rolled back，之前的改变都undone
```

- rollback

用途：do some error checking and manually rolling back a transaction

```s
USE sql_store;

START TRANSACTION;

INSERT INTO orders (customer_id, order_date, status)
VALUES (1, '2019-01-01', 1);

INSERT INTO order_items
VALUES (LAST_INSERT_ID(), 1, 1, 1);  -- 返回insert order的id

ROLLBACK;
```

- 补充

```s
SHOW VARIABLES LIKE 'autocommit';
```
当执行一个single statement时, MySQL把其放入一个transaction中, 如果statement没有报错，就commit

# 3. concurrency

concurrency: 并发（多人同时处理同意数据）

## 3.1 MySQL如何处理并发: default lock mechanism

scenario：两个用户同时更新某个消费者的points

simulation
 - 打开两个session
 - 第一个session
	```s
	USE sql_store;
	START TRANSACTION;
	UPDATE customers
	SET points = points + 10
	WHERE cuatomer_id = 1;
	COMMIT;
	```
 - 第二个session
	```s
	USE sql_store;
	START TRANSACTION;
	UPDATE customers
	SET points = points + 10
	WHERE cuatomer_id = 1;
	COMMIT;
	```
  - 分别执行两个session的前3行
	第一个session的update在run, 因为当执行第一个update时，MySQL对要update的行put a lock；所以，如果另一个transaction试图update相同的行，它必须等到第一个transaction完成，either commited or rolled back.

## 3.2 并发带来的问题

- lost updates

当两个transaction更新相同的数据，但是没有使用locks;

在这种情况下，后commit的transaction将会overwrie第一个transaction对数据做出的更改;

所以，MySQL使用lock mechanism来防止两个transaction同时对相同的数据进行更改

- dirty reads

一个transaction读取到了还没有被commit的数据;

transaction A 更新数据，transaction B 读取相同数据，transaction A ROLLBACK， transaction B 读取到了脏数据;

对于事务，提出了"a level of isolation"，transaction更改的数据不会即刻被其他transaction看到，除非它被committed，即**read committed: transaction只能读取committed data**

- non-repeating or inconsistent reads

有很多的事务，两次读取得到不同的结果;

transaction A 读取数据，transaction B 更新数据，transaction A 两次读取得到不同的结果;

在业务情形下，如果希望读取到更改前的值，那么应该 increase transaction A 的 isolation level，这样 transaction B 对数据的改变，transaction A 是不可见的，即**repeatable read：transaction获得第一次读取的数据**

- phantom reads 幽灵

transaction A 读取满足条件的数据，transaction B 更新数据，更新后满足条件的数据（ghost）未被读取到；

在业务情形下，如果希望得到所有满足条件的数据，那么应该保证其他所有影响条件的transaction都被committed，即**serializable：先执行所有影响读取结果的transaction，再执行读取的transaction**

## 3.3 transaction isolation levels

|  | lost updates | dirty reads | non-repeating reads | phantom reads |
| ------ | ------ | ------ | ----- | ----- |
| read uncommitted |  |  |  |  |
| read committed |  | √ |   |   |
| repeatable read | √ | √ | √ |  | 
| serializable | √ | √ | √ | √ |

随着isolation level的升级，将面临更多的performances和scalability问题，因为将会给transaction加更多的locks;

MySQL默认的transaction isolation level是repeatable reads
