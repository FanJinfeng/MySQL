把 SQL code 放在 application code 外面，存储在数据库中的 stored precodure 或者 function 中。

# 1. stored procedure

stored precodure：一个数据库对象，存储 SQL code，在application code 中可以 call 这些 precodures 去 get 和 save 数据
  1. store and organize SQL
  2. faster execution
  3. 数据安全
 
## 1.1 创建

; default, 分隔查询语句, 在procedure中必须使用;来分隔语句
  
  
在运行过程中，$$...$$之间不可以加注释

```s
DELIMITER  $$  -- 把下面的statement作为一个unit
CREATE PROCEDURE get_clients()
BEGIN  -- body
    SELECT * FROM clients
END$$

DELEMITER ;  -- change it back to ;
```

```s
CALL get_clients()  -- 在 sql 中 call procedure
```

```s
DELIMITER $$
CREATE PROCEDURE get_invoices_with_balance()
BEGIN 
    SELECT *
    FROM invoices
    WHERE (invoice_total - payment_total) > 0;
END$$

DELIMITER ;
```

```s
DELIMITER $$
CREATE PROCEDURE get_invoices_with_balance()
BEGIN
    SELECT *
    FROM invoices_with_balance
    WHERE balance > 0;
END$$

DELIMITER ;
```

- 在workbench中创建procedure的快捷方式

1. 在导航栏右击stored procedure创建
2. 专注于select
3. 点击apply自动补充代码

```s
CREATE PROCEDURE `get_payments` ()
BEGIN
	SELECT * FROM payments;
END
```

## 1.2 删除

```s
DROP PROCEDURE get_clients
```

如果 procedure 不存在，程序报错

```s
DROP PROCEDURE IF EXISTS get_clients
```

分享procedure给别人

```s
DROP PROCEDURE IF EXISTS get_clients;

DELIMITER $$
CREATE PROCEDURE get_clients()
BEGIN
    SELECT *
    FROM clients;
END$$

DELIMITER ;
```

## 1.3 在procedure中加入parameter

1. 传参给 procedure
2. 传参给 calling program

### 1.3.1 传参给procedure

- 不含默认值的参数

一种方式是更改参数的名称：加上前缀p_，后缀_param
另一种方式是给table一个别名

```s
DROP PROCEDURE IF EXISTS get_clients_by_state;

DELIMITER $$
CREATE PROCEDURE get_clients_by_state
(
    state CHAR(2)  -- char varhcar
)
BEGIN
    SELECT * FROM clients c
    WHERE c.state = state;
END$$

DELIMITER ;
```

```s
CALL get_clients_by_state('CA')  -- call procedure
```

```s
DROP PROCEDURE IF EXISTS get_invoices_by_client;

DELIMITER $$
CREATE PROCEDURE get_invoices_by_client(client_id INT)
BEGIN
    SELECT * FROM clients c
    WHERE c.client_id = client_id;
END$$

DELIMITER ;
```

- 含默认值的参数

```s
DROP PROCEDURE IF EXISTS get_invoices_by_client;

DELIMITER $$
CREATE PROCEDURE get_invoices_by_client(client_id INT)
BEGIN
    IF client_id IS NULL THEN
        SET client_id = 1;
    END IF;
    
    SELECT * FROM clients c
    WHERE c.client_id = client_id;
END$$

DELIMITER ;
```

```s
CALL get_invoices_by_client(NULL)
```

```s
DROP PROCEDURE IF EXISTS get_invoices_by_client;

DELIMITER $$
CREATE PROCEDURE get_invoices_by_client(client_id INT)
BEGIN
    IF client_id IS NULL THEN
        SELECT * FROM clients;
    ELSE
        SELECT * FROM clients c
        WHERE c.client_id = client_id;
    END IF;
END$$

DELIMITER ;
```

简化上述SQL代码

```s
DROP PROCEDURE IF EXISTS get_invoices_by_client;

DELIMITER $$
CREATE PROCEDURE get_invoices_by_client(client_id INT)
BEGIN
    SELECT * FROM clients c
    WHERE c.client_id = IFNULL(client_id, c.client_id);
END$$

DELIMITER ;
```

- 检查传入的参数是否符合要求

```s
DROP PROCEDURE IF EXISTS make_payments;

DELIMITER $$
CREATE PROCEDURE make_payments(
    invoice_id INT,
    payment_amount DECIMAL(9, 2),
    payment_date DATE
)
BEGIN
    IF payment_amount <= 0 THEN
        SIGNAL SQLSTATE '22003'  -- 错误编码
	    SET MESSAGE_TEXT = 'Invalid payment amount';
    END IF;
    UPDATE invoices i
    SET
    	i.payment_total = payment_amount,
	i.payment_date = payment_date
    WHERE i.invoice_id = invoice_id;
END$$

DELIMITER ;
```

parameters: placeholder 传给procedure或者function的参数

arguments: parameters的值

### 1.3.2 传参给calling program

input parameter; output parameter

```s
DROP PROCEDURE IF EXISTS get_unpaid_invoices_for_client;

DELIMITER $$
CREATE PROCEDURE get_unpaid_invoices_for_client(
    client_id INT,
    OUT invoices_count INT,
    OUT invoices_total DECIMAL(9, 2)
)
BEGIN
    SELECT COUNT(*), SUM(invoice_total)
    INTO invoices_count, invoices_total  -- 通过select语句赋值
    FROM invoices i
    WHERE i.client_id = client_id
        AND payment_total = 0;
END$$

DELIMITER ;
```

```s
set @invoices_count = 0;  -- 声明变量，并初始化
set @invoices_total = 0;
call sql_invoicing.get_unpaid_invoices_for_client
    (2, @invoices_count, @invoices_total);
select @invoices_count, @invoices_total;  -- 读取并输出
```

# 2. 变量

## 2.1 user or session variable

SET @name=0

用于获取output parameters的值

当断开与MySQL的连接时，释放缓存

## 2.2 local variable

DECLARE name INT

定义在procedure或者function内部，用于计算

当precodure或者function执行完成后，释放缓存

```s
DROP PROCEDURE IF EXISTS get_risk_factor;

DELIMITER $$
CREATE PROCEDURE get_risk_factor()
BEGIN
    -- risk_factor = invoices_total / invoices_count * 5
    DECLARE risk_factor DECIMAL(9, 2) DEFAULT 0;
    DECLARE invoices_total DECIMAL(9, 2);  -- 使用SELECT语句赋值
    DECLARE invoices_count INT;
    
    SELECT COUNT(*), SUM(invoice_total  -- select语句赋值
    INTO invoices_count, invoices_total
    FROM invoices;
    
    SET risk_factor = invoices_total / invoices_count * 5;  -- 计算
    
    SELECT risk_factor;  -- 读取
END$$

DELIMIETR ;
```

# 3. 函数

function: 只能返回single value, RETURN 子句

procedure: 可以返回result sets

```s
DROP FUNCTION IF EXISTS get_risk_factor_for_client;

DELIMITER $$
CREATE FUNCTION get_risk_factor_for_client(
    client_id INT
) 
RETURNS int  -- 返回值的类型
READS SQL DATA  -- attribute
BEGIN
    DECLARE risk_factor DECIMAL(9, 2) DEFAULT 0;  -- 设定默认值
    DECLARE invoices_total DECIMAL(9, 2);  -- 使用SELECT语句赋值
    DECLARE invoices_count INT;
    
    SELECT COUNT(*), SUM(invoice_total)
    INTO invoices_count, invoices_total
    FROM invoices i
    WHERE i.client_id = client_id;
    
    SET risk_factor = invoices_total / invoices_count * 5;  -- 计算
    
    RETURN IFNULL(risk_factor, 0);
END
```

```s
SELECT
    client_id,
    name,
    get_risk_factor_for_client(client_id) AS risk_factor
FROM clients
```

- attribute
 - DETERMINISTIC: 相同的输入，函数产生相同的输出
 - READS SQL DATA: 可以在函数中使用select语句读取数据
 - MODIFIES SQL DATA: 可以在函数中使用insert, update, delete语句
