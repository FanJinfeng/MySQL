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

### 1.3.2 传参给calling program

