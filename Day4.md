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


