## 1. 简介

### 1.1 数据库

关系型数据库：MySQL、SQL Server、Oracle

非关系型数据库：例如图数据库

### 1.2 在Win10上安装MySQL

安装社区版

下载Installer进行安装

安装开发人员默认设置

设置用户名和密码

### 1.3 MySQL WorkBench简介

顶部为工具栏

左侧为导航栏：包括Admininstration（管理）和Schemas（数据库结构）

中间为查询语句窗口

### 1.4 数据库对象

tabels：用来保存数据

views：把多个表的数据整合，放在view中，是一个虚拟表

stored procedures：保存查询语句

functions：保存查询语句


## 2. 查询（retrive）数据

### 2.1 从一张表中查询数据

```s
USE sql_store;
SELECT *
FROM customers
WHERE customer_id = 1
ORDER BY first_name
```

- SELECT 子句

```s
USE sql_store;
SELECT 
  first_name, 
  last_name, 
  points AS 'shopping points',  -- 在列名中使用空格
  points * 10 + 100 AS discount_factor  -- 可以是表达式
FROM customers
```

```s
SELECT DISTINCT state  -- 去重
FROM customers
```

```s
USE sql_store;  -- 练习
SELECT 
  name,
  unit_price,
  unit_price * 1.1 AS new_price
 FROM products
```

- WHERE 子句（filter data）

```s
-- > >= < <= = !=/<>
SELECT *
FROM  customers
WHERE points > 3000  -- 比较数字
```

```s
SELECT * 
FROM customers
WHERE state != 'VA'  -- 比较字符串
```

```s
SELECT *
FROM customers
WHERE birth_date > '1990-01-01'  -- 比较日期
```
  - and or not 操作符

```s
SELECT *
FROM customers
WHERE birth_date >= '1990-01-01' AND points > 1000  -- 多条件
```

```s
SELECT *
FROM customers
WHERE birth_date >= '1990-01-01' OR 
      (points > 1000 AND state = 'VA')  -- AND的优先级更高
```

```s
SELECT *
FROM customers
WHERE NOT (birth_date >= '1990-01-01' OR points > 1000)  # 类似于数学中的交集和并集运算
```

```s
SELECT *
FROM order_items
WHERE order_id = 6 AND quantity * unit_price > 30  -- 可以在WHERE子句中使用表达式
```
  - in 操作符

```s
SELECT *
FROM customers
WHERE state = 'VA' OR state = 'GA' OR state = 'FL'  -- or 运算符
```

```s
SELECT *
FROM customers
WHERE state IN ('VA', 'GA', 'FL')  -- in 运算符
```

```s
SELECT *
FROM customers
WHERE state NOT IN ('VA', 'GA', 'FL')  -- in 运算符
```
  - between 操作符

```s
SELECT *
FROM customers
WHERE points >= 1000 AND points <= 3000  -- and 运算符
```

```s
SELECT *
FROM customers
WHERE points BETWEEN 1000 AND 3000  -- between 运算符 数字
```

```s
SELECT *
FROM customers
WHERE birth_date BETWEEN '1990-01-01' AND '2000-01-01'  -- 字符串
```

  - like 操作符

```s
SELECT *
FROM customers
WHERE last_name LIKE 'b%'  -- like 运算符 字符串以b开头，不区分大小写，%表示任意个字符
```

```s
SELECT *
FROM customers
WHERE last_name LIKE '%b%'  -- 字符串包含b
```

```s
SELECT *
FROM customers
WHERE last_name LIKE '%y'  -- 字符串以y结尾
```

```s
SELECT *
FROM customers
WHERE last_name LIKE '_____y'  -- 字符串包含6个字符，以y结尾，_表示1个字符
```

```s
SELECT *
FROM customers
WHERE last_name LIKE '_____y'  -- 字符串包含6个字符，以y结尾
```

```s
SELECT *
FROM customers
WHERE (address LIKE '%trail%' OR 
      address LIKE '%avenue%') AND 
      phone LIKE '%9'
```

  - regexp 运算符

```s
SELECT *
FROM customers
WHERE last_name REGEXP 'field|mac|rose'  -- ^表示开始，$表示结束，|表示or
```

```s
SELECT *
FROM customers
WHERE last_name REGEXP '[gim]e'  -- []表示任意一个
```

```s
SELECT *
FROM customers
WHERE last_name REGEXP '[a-h]e'
```

```s
SELECT *
FROM customers
WHERE first_name REGEXP 'elka|amber' AND
      last_name REGEXP 'ey$|on$' AND  -- 都要加上$
      last_name REGEXP '^my|se' AND  -- 只有前一个加上^
      last_name REGEXP 'b[ru]'
```

  -   is null 操作符

```s
SELECT *
FROM customers
WHERE phone IS NULL
```

```s
SELECT *
FROM customers
WHERE phone IS NOT NULL
```

```s
SELECT *
FROM orders
WHERE shipped_date IS NULL
```

- order by 子句（默认按照主键递增排序）

```s
SELECT *
FROM customers
ORDER BY first_name
```

```s
SELECT *
FROM customers
ORDER BY first_name DECS
```

```s
SELECT last_name
FROM customers
ORDER BY state DESC, first_name  -- 这些字段不必是查询字段（for MySQL）
```

```s
SELECT last_name, 10 AS points
FROM customers
ORDER BY points, first_name  -- 可以对别名进行排序
```

```s
SELECT order_id, product_id, quantity, unit_price
FROM order_items
WHERE order_id = 2
ORDER BY quantity * unit_price DESC  -- 对表达式进行排序
```

- limit 子句（限制返回的记录数，用于分页）

```s
SELECT *
FROM customers
LIMIT 3
```

```s
SELECT *
FROM customers
LIMIT 6, 3  -- 跳过前6条记录，返回3条记录
```

```S
SELECT *
FROM customers
ORDER BY points DESC
LIMIT 3  -- limit 子句一定是位于最后
```

### 2.2从多张表中查询数据

- inner join

```s
SELECT order_id, o.customer_id, first_name
FROM orders o
INNER JOIN customers c 
  ON o.customer_id = c.customer_id  -- inner 可以省略；如果设置了别名，别名必须在各处使用
```

```s
SELECT oi.order_id, oi.product_id, oi.quantity, oi.unit_price, p.name  -- 这里显示的历史价格
FROM order_items oi
JOIN products p
  ON oi.product_id = p.product_id
```

  - 跨数据库合并表

```s
SELECT *
FROM order_items oi
JOIN sql_inventory.products p
  ON oi.product_id = p.product_id
```

  - 表与自己合并（需要使用不同的别名）

```s
SELECT 
  e.employee_id,
  e.first_name,
  m.first_name AS manager
FROM employees e
JOIN employees m
  ON e.reports_to = m.employee_id
```

  - 合并3张表

```s
SELECT 
  o.order_id,
  o.order_date,
  c.first_name,
  c.last_name,
  os.name AS status
FROM orders o
JOIN customers c  -- 可以使用多个join子句
  ON o.customer_id = c.customer_id
JOIN order_statuses os 
  ON o.status = os.order_status_id
```

  -- 复合键（prder_id, product_id）

```s
SELECT *
FROM order_items oi
JOIN order_item_notes oin  -- 多条件join表
  ON oi.order_id = oin.order_id
  AND oi.product_id = oin.product_id
```

  - implicit join syntax（上面是显示的）
```s
SELECT *
FROM  orders o, customers c
WHERE o.customer_id = c.customer_id  -- 如果忘记where子句，将进行交叉合并
```

- outer join

```s
SELECT
  c.customer_id,
  c.first_name,
  o.order_id
FROM customers c
LEFT OUTER JOIN orders o  -- 主表的所有记录都将返回，不论条件是否为真；outer可以省略
  ON c.customer_id = o.customer_id
ORDER BY c.customer_id
```

```s
SELECT
  p.product_id,
  p.name,
  oi.quantity
FROM products p
LEFT JOIN order_items oi
  on p.product_id = oi.product_id
```

  - 合并3张表
  
```s
SELECT
  c.customer_id,
  c.first_name,
  o.order_id,
  sh.name AS shipper
FROM customers c
LEFT OUTER JOIN orders o
  ON c.customer_id = o.customer_id
LEFT JOIN shippers sh
  ON o.shipper_id = sh.shipper_id
ORDER BY c.customer_id
```

```s
SELECT 
  o.order_date,
  o.order_id,
  c.first_name AS customer,
  sh.name AS shipper,
  os.name AS status
FROM orders o
JOIN customers c  -- 这里不用左连接
  ON  o.customer_id = c.customer_id
LEFT JOIN shippers sh
  ON o.shipper_id = sh.shipper_id
LEFT JOIN order_statuses os
  ON o.status = os.order_status_id
```

  - 表与自己合并
  
 ```s
 SELECT
   e.employee_id,
   e.first_name,
   m.first_name AS manager
 FROM employees e
 LEFT JOIN employees m
    ON e.reports_to = m.employee_id
 ```

- on 子句 --> using 子句 （要求键在不同的表中有相同的名字）

```s
SELECT
  o.order_id,
  c.first_name,
  sh.name AS shipper
FROM orders o
JOIN customers c
  USING (customer_id)
LEFT JOIN shippers sh
  USING (shipper_id)
```

````s
SELECT *
FROM order_items oi
JOIN order_item_notes oin
  USING (order_id, product_id)  -- 复合键的时候也适用
```

```s
SELECT
  p.date,
  c.name AS client,
  p.amount,
  pm.name
FROM payments p
JOIN clients c USING (client_id)
LEFT JOIN payment_methods pm
  ON p.payment_method = pm.payment_method_id
```

- cross join（交叉连接）

```s
SELECT
  c.first_name AS customer,
  p.name AS product
FROM customers c
CROSS JOIN products p
ORDER BY c.first_name
````

  - implicit join syntax（上面是显示的）
```s
SELECT *
FROM  orders o, customers c
```

### 2.3 给一张表中添加列

```s
FROM orders
WHERE order_date >= '2019-01-01'
UNION  -- 连接多个查询结果
SELECT
  order_id, order_date,
  'Archived' AS status
FROM orders
WHERE order_date < '2019-01-01'
```

```s
SELECT first_name
FROM customers
UNION  -- 可以连接不同表的查询结果，但是结果的列数必须相同，最终显示第一个结果的列名
SELECT name
FROM shippers
```

```s
SELECT
  customer_id,
  first_name,
  points,
  'Bronze' AS type
FROM customers
WHERE points < 2000
UNION
SELECT
  customer_id,
  first_name,
  points,
  'Silver' AS type
FROM customers
WHERE points  BETWEEN 2000 AND 3000
UNION
SELECT
  customer_id,
  first_name,
  points,
  'Gold' AS type
FROM customers
WHERE points > 3000
ORDERBY first_name
```

## 3. 插入、更新、删除数据

### 3.1 字段的属性

VARCHAR相较于CHAR会更节省空间

PK primary key; NN not null; AI auto increment; default value

### 3.2 插入

- 在一个表中，插入一行记录
```s
INSERT INTO customers
VALUES (
    DEFAULT,  -- 自增
    'John', 
    'Smith', 
    '1990-01-01', 
    NULL,  -- 使用默认值
    'address',
    'city',
    'CA',
    DEFAULT)  -- 使用默认值
```

```s
INSERT INTO customers (  -- 顺序是可以改变的
    first_name,
    last_name,
    birth_date,
    address,
    city,
    state)
 VALUES (
    'John', 
    'Smith', 
    '1990-01-01', 
    'address',
    'city',
    'CA')
```

- 在一个表中，插入多行记录

```s
INSERT INTO shippers (name)
VALUES ('shippers1'),
       ('shippers2'),
       ('shippers3')
```

- 插入分层数据（hierarchical data）

orders 中的一条记录对应 order_items中的多条记录

```s
INSERT INTO orders (customer_id, order_date, status)
VALUES (1,'2019-01-02', 1);
INSERT INTO order_items
VALUES 
    (LAST_INSERT_ID(), 1, 1, 2.95),
    (LAST_INSERT_ID(), 2, 1, 3.95)
```

-  复制

把一张表中的所有记录存为档案
```s
-- 可以快速创建一张表的副本，但是在副本中没有自增的主键了
CREATE TABLE orders_archived AS
SELECT * FROM orders  -- 子查询 subquery
```

把一张表中的部分记录存为档案
```s
INSERTINTO orders_archived
SELECT *
FROM orders
WHERE order_date < '2019-01-01'
```

```s
CREATE TABLE invoice_archived AS
SELECT
    i.invoice_id,
    i.number,
    c.name AS client,
    i.invoice_total,
    i.payment_total,
    i.invoice_date,
    i.payment_date,
    i.due_date
FROM invoices i
LEFT JOIN clients c USING (client_id)
WHERE i.payment_date IS NOT NULL
```

### 3.3 更新

- 更新一行记录

```s
UPDATE invoices
SET payment_total = 10, payment_date = '2019-03-01'
WHERE invoice_id = 1
```

```s
UPDATE invoices
SET payment_total = DEFAULT, payment_date = NULL
WHERE invoice_id = 1
```

```s
UPDATE invoices
SET 
    payment_total = invoice_total * 0.5, 
    payment_date = due_date
WHERE invoice_id = 3
```

- 更新多行记录

Mysql workbench 使用安全的更新模式，仅允许更新一条记录

可以勾掉 save update 模式

```s
UPDATE invoices
SET 
    payment_total = invoice_total * 0.5, 
    payment_date = due_date
WHERE client_id = 3
```

```s
UPDATE invoices
SET 
    payment_total = invoice_total * 0.5, 
    payment_date = due_date
WHERE client_id IN (3, 4)
```

```s
UPDATE customers
SET points = points + 50
WHERE birth_date < '1990-01-01'
```

- 在更新语句中使用子查询

更新一条记录
```s
UPDATE invoices
SET 
    payment_total = invoice_total * 0.5, 
    payment_date = due_date
    WHERE client_id = 
        (SELECT client_id
        FROM clients
        WHERE name = 'Myworks')  -- 子查询
```

更新多条记录
```s
UPDATE invoices
SET 
    payment_total = invoice_total * 0.5, 
    payment_date = due_date
WHERE client_id IN 
    (SELECT client_id
    FROM clients
    WHERE state IN ('CA', 'NY'))  -- 子查询
```

```s
UPDATE orders
SET comments = 'Gold customer'
WHERE customer_id in
    (SELECT customer_id
    FROM customers
    WHERE points > 3000)
```

### 3.4 删除

```s
DELECT FROM invoices
WHERE invoice_id = 1
```

- 使用子查询

```s
DELETE FROM invoices
WHERE invoice_id = 
    (SELECT *
    FROM clients
    WHERE name = 'Myworks')
```
