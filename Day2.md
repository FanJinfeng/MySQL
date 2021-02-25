# 1. 汇总数据

## 1.1 aggregate function

数值型、日期型、字符型

仅计算非NULL值

```s
SELECT 
    MAX(invoice_total) AS highest,
    MIN(invoice_total) AS lowest,
    AVG(invoice_total) AS average,
    SUM(invoice_total * 1.1) AS total,  -- 先计算表达式的值
    COUNT(invoice_total) AS number_of_invoices,
    COUNT(payment_date) AS count_of_payments, -- 统计有payment_date的记录数（不包含NULL）
    COUNT(*) AS total_records,  -- 统计所有记录数
    COUNT(DISTINCT client_id) AS  number_of_clients  -- 去重
FROM invoices
WHERE invoice_date > '2019-07-01'  -- 增加过滤条件
```

```s
SELECT
    'First half of 2019' AS date_range,
    SUM(invoice_total) AS total_sales,
    SUM(payment_total) AS total_payment,
    SUM(invoice_total - payment_total) AS what_we_expect
FROM invoices
WHERE invoice_date 
	BETWEEN '2019-01-01' AND '2019-06-30'
UNION
SELECT
    'Second half of 2019' AS date_range,
    SUM(invoice_total) AS total_sales,
    SUM(payment_total) AS total_payment,
    SUM(invoice_total - payment_total) AS what_we_expect
FROM invoices
WHERE invoice_date 
	BETWEEN '2019-07-01' AND '2019-12-31'
UNION
SELECT
    'Total' AS date_range,
    SUM(invoice_total) AS total_sales,
    SUM(payment_total) AS total_payment,
    SUM(invoice_total - payment_total) AS what_we_expect
FROM invoices
WHERE invoice_date
		BETWEEN '2019-01-01' AND '2019-12-31'
```

## 1.2 group by 子句

根据一个字段分组

```s
SELECT
    client_id,
    SUM(invoice_total) AS total_sales
FROM invoices
WHERE invoice_date >= '2019-07-01'
GROUP BY client_id
ORDER BY total_sales DESC
```

根据多个字段分组

```s
SELECT
    state,
    city,
    SUM(invoice_total) AS total_sales
FROM invoices i
JOIN clients c USING (client_id)
GROUP BY state, city 
```

```s
SELECT
    p.date,
    pm.name AS payment_method,
    SUM(p.amount) AS total_payments
FROM payments p
JOIN payment_methods pm
    ON p.payment_method = pm.payment_method_id
GROUP BY p.date
ORDER BY p.date
```

- 对分组后的数据进行筛选

WHERE 子句中不必是SELECT的字段；HAVING 子句中必须是SELECT的字段

单条件过滤

```s
SELECT
    client_id,
    SUM(invoice_total) AS total_sales,
    COUNT(*) AS number_of_invoices  -- 统计everything
FROM invoices
GROUP BY client_id
HAVING total_sales > 500
```

多条件过滤

```s
SELECT
    client_id,
    SUM(invoice_total) AS total_sales,
    COUNT(*) AS number_of_invoices  -- 统计everything
FROM invoices
GROUP BY client_id
HAVING total_sales > 500 AND number_of_invoices > 5
```

```s
SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    SUM(oi.quantity * oi.unit_price) AS total_spend
FROM customers c
JOIN orders o USING (customer_id)
JOIN order_items oi USING (order_id)
WHERE c.state = VA'
GROUP BY c.customer_id, c.first_name, c.last_name  -- 作为惯例，会使用所有的SELECT字段
HAVING total_spend > 100
```

- 对分组后的结果进行汇总

with rollup 是MySQL特有的

根据一个字段分组

```s
SELECT 
    client_id,
    SUM(invoice_total) AS total_sales
FROM invoices
GROUP BY client_id WITH ROLLUP
```

根据多个字段分组

```s
SELECT 
    c.state,
    c.city,
    SUM(i.invoice_total) AS total_sales
FROM invoices i
JOIN clients c USING (client_id) 
GROUP BY c.state, c.city WITH ROLLUP
```

```s
SELECT
    pm.name AS payment_method,
    SUM(p.amount) AS total
FROM payments p
JOIN payment_methods pm
    ON p.payment_method = pm.payment_method_id
GROUP BY pm.name WITH ROLLUP  -- 当在 group by 子句中使用 with rollup 时，不可以使用别名
```


# 2. 复杂的查询语句（SELECT 语句在另一个语句中）

## 2.1 where 子句

- 子查询返回1个值

```s
SELECT * 
FROM products
WHERE unit_price > (  -- 先执行括号中的查询
        SELECT unit_price
        FROM products
        WHERE product_id = 3
)
```

```s
SELECT *
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
)
```

- 子查询返回一系列值

```s
SELECT *
FROM products
WHERE product_id NOT IN (  -- 找到被下过单的product_id
    SELECT DISTINCT product_id
    FROM orders_items
)
```

```s
SELECT *
FROM clients
WHERE client_id NOT IN(
    SELECT DISTINCT client_id
    FROM invoices
)
```

使用join语句实现上述任务，两种方式的选择主要看那个可读性更强

```s
SELECT *
FROM clients
LEFT JOIN invoices USING (client_id)
WHERE invoice_id IS NULL 
```

```s
SELECT
    customer_id,
    first_name,
    last_name
FROM customers
WHERE customer_id IN (
    SELECT o.customer_id
    FROM order_items oi
    JOIN orders o USING (order_id)
    WHERE product_id = 3
)
```

```s
SELECT DISTINCT  -- 会有重复的记录
    customer_id,
    first_name,
    last_name
FROM customers
JOIN orders o USING (customer_id)
JOIN order_items oi USING (order_id)
WHERE oi.product_id = 3
```

  - 使用 all 使得满足一系列值

```s
SELECT *
FROM invoices
WHERE invoice_total > (
    SELECT MAX(invoice_total)
    FROM invoices
    WHERE client_id = 3
)
```

```s
SELECT *
FROM invoices
WHERE invoice_total > ALL (
    SELECT MAX(invoice_total)
    FROM invoices
    WHERE client_id = 3
)
```

  - 使用 any 使得满足任意值

```s
SELECT *
FROM clients
WHERE client_id IN (
    SELECT client_id
    FROM invoices
    GROUP BY client_id
    HAVING COUNT(*) >= 2
)
```

```s
SELECT *
FROM clients
WHERE client_id = ANY (
    SELECT client_id
    FROM invoices
    GROUP BY client_id
    HAVING COUNT(*) >= 2
)
```

- correlated 子查询（子查询和外面的查询产生关联，子查询执行多次，每一条记录都执行查询）

返回salary大于office均值的

```s
SELECT *
FROM employee e
WHERE salary > (
      SELECT AVG(salary)
      FROM employees
      WHERE office_id = e.office_id
)
```

```s
SELECT *
FROM invoices i
WHERE invoice_total > (
    SELECT AVG(invoice_total)
    FROM invoices
    WHERE client_id = i.client_id
)
```

  - exist 操作符，使得查询更高效

```s
SELECT DISTINCT *
FROM clients
JOIN invoices USING (client_id)
WHERE invoice_id IS NOT NULL
```

```s
SELECT *
FROM clients
WHERE client_id IN (  -- 可能会产生一个很长的list
   SELECT DISTINCT client_id
   FROM invoices
)
```

```s
SELECT *
FROM clients c
WHERE EXISTS (  -- 返回 true / false
   SELECT *
   FROM invoices
   WHERE client_id = c.client_id
)
```

```s
SELECT *
FROM products p
WHERE NOT EXISTS (
  SELECT *
  FROM order_items
  WHERE product_id = p.product_id
)
```

## 2.2 select 子句

```s
SELECT 
    invoice_id,
    invoice_total,
    (SELECT AVG(invoice_total)
        FROM invoices) AS invoice_average,
    invoice_total - (SELECT invoice_average) AS difference  -- 这里不能使用别名
FROM invoices
```

```s
SELECT
    client_id,
    name,
    (SELECT SUM(invoice_total)
        FROM invoices
        WHERE client_id = c.client_id) AS total_sales,
    (SELECT AVG(invoice_total)
        FROM invoices) AS average,
    (SELECT total_sales - average) AS difference
FROM clients c
```

## 2.3 from 子句

在 from 子句中加入子查询，会让代码变得非常复杂，事实上我们会用view来存储临时的表

```s
SELECT *
FROM (
    SELECT
        client_id,
        name,
        (SELECT SUM(invoice_total)
            FROM invoices
            WHERE client_id = c.client_id) AS total_sales,
        (SELECT AVG(invoice_total)
            FROM invoices) AS average,
        (SELECT total_sales - average) AS difference
    FROM clients c
) sales_summary  -- 必须给到一个别称
WHERE total_sales IS NOT NULL
```

# 3. 函数

## 3.1 数值型

```s
SELECT ROUND(5.7334, 2);  -- 舍入
SELECT TRUNCATE(5.7334, 2);  -- 截断
SELECT CEILING(5.7);  -- >=
SELECT FLOOR(5.7);  -- <=
SELECT ABS(-5.2);
SELECT RAND();  -- 0-1
```

## 3.2 字符型

```s
SELECT LENGTH('sky');
SELECT UPPER('sky');
SELECT LOWER('SKY');
SELECT LSTRIM('   SKY');
SELECT RTRIM('SKY    ');
SELECT TRIN('   SKY   ');
SELECT LEFT('kindergarten', 4);  -- 从左边截取
SELECT RIGHT('kindergarten', 6);  -- 从右边截取
SELECT SUBSTRING('kindergarten', 3, 5);  -- start=3, sub_len=5, 标号从1开始
SELECT LOCATE('garden', 'kindergarten');  -- 返回字符索引，如果不存在返回0
SELECT REOLACE('kindergarten', 'garten', 'garden');  -- 替换
SELECT CONCAT('forst', 'last');  -- 结合
```

```s
SELECT CONCAT(first_name, ' ', last_name) AS full_name
FROM customers
```

## 3.3 日期型

```s
SELECT NOW(), CURDATE(), CURTOME();
SELECT YEAR(NOW()), MONTH(NOW()), DAY(NOW()), HOUR(NOW());  -- 返回整数
SELECT DAYNAME(NOW()), MONTHNAME(NOW());  -- 返回字符串
SELECT EXTRACT(YEAR FROM NOW());  -- 标准化的SQL语言
```

```s
SELECT *
FROM orders
WHERE YEAR(order_date) = 2019
```

```s
SELECT *
FROM orders
WHERE EXTRACT(YEAR FROM order_date) = 2019
```

- fromat date and times

```s
SELECT DATE_FORMAT(NOW(), '%y');
SELECT DATE_FORMAT(NOW(), '%Y');  -- 4 digits
SELECT DATE_FORMAT(NOW(), '%m %y');
SELECT DATE_FORMAT(NOW(), '%M %y');
SELECT DATE_FORMAT(NOW(), '%m %d %y');
SELECT DATE_FORMAT(NOW(), '%H:%i %p');  -- i表示分, p表示PM
```

- calculate date and times

```s
SELECT DATE_ADD(NOW(), INTERVAL 1 DAY);
SELECT DATE_ADD(NOW(), INTERVAL 1 YEAR);
SELECT DATE_ADD(NOW(), INTERVAL -1 DAY);
SELECT DATE_SUB(NOW(), INTERVAL 1 DAY);
```

```s
SELECT DATEDIFF('2019-01-05 09:00', '2019-01-01 17:00');  -- 返回days, date1 - date2
SELECT TIME_TO_SEC('09:00');  -- 从00:00开始的秒数
SELECT TIME_TO_SEC('09:02') - TIME_TO_SEC('09:00')
```

## 3.4 条件

- 替换空值

单条件判断

```s
SELECT 
    order_id,
    IFNULL(shipper_id, 'Not assigned') AS shipper
FROM orders
```

```s
SELECT 
    order_id,
    COALESCE(shipper_id, 'Not assigned') AS shipper
FROM orders
```

多条件判断

```s
SELECT 
    order_id,
    COALESCE(shipper_id, comments, 'Not assigned') AS shipper  -- shipper_id, comments是否为空
FROM orders
```

- IF 语句

```s
-- IF(exp, first, second)
SELECT
    order_id,
    order_date,
    IF(
        YEAR(order_date) = 2019, 
        'Active', 
        'Archived') AS category
FROM orders
```

```s
SELECT
    product_id,
    name,
    COUNT(*) AS orders,
    IF(COUNT(*) > 1, 'Many times', 'Once') AS ferquency  -- 这里不可以使用别称
FROM products
JOIN order_items oi USING (product_id)
GROUP BY product_id, name
```

- case 语句

接受多个判断

```s
SELECT
    order_id,
    order_date,
    CASE
        WHEN YEAR(order_date) = 2019 THEN 'Ative'
        WHEN YEAR(order_date) = 2018 THEN 'Last year'
        WHEN YEAR(order_date) < 2018 THEN 'Archived'
        ELSE 'Feature'  -- 可选
    END AS category
FROM orders
```

```s
SELECT
    CONCAT(first_name, ' ', last_name) AS customer,
    points,
    CASE
        WHEN points > 3000 THEN 'Gold'
        -- WHEN points BETWEEN 2000 AND 3000 THEN 'Silver'
        WHEN points >= 2000 THEN 'Silver'
        -- WHEN points < 2000 THEN 'Bronze'
        ELSE 'Bronze'
    END AS category
FROM customers
```

# 4. Views（临时保存queries和subqueries）

view 是一个虚拟表，不存储数据，输出存储表原来的表中。

1. 简化语句
2. 减少changes的影响
3. 保护原数据

## 4.1 创建

```s
CREATE VIEW sales_by_client AS  -- 创建view对象，没有返回
SELECT 
    c.client_id,
    c.name,
    SUM(invoice_total) AS total_sales
FROM clients c
JOIN invoices i USING (client_id)
GROUP BY client_id, name
```

```s
SELECT *
FROM sales_by_client
JOIN clients USING (client_id)
WHERE total_sales > 500
ORDER BY total_sales DESC
```

```s
CREATE VIEW balance_by_client AS
SELECT
    c.client_id,
    c.name,
    SUM(i.invoice_total - i.payment_total) AS balance
FROM clients c
JOIN invoices i USING (client_id)
GROUP BY client_id, name
```

## 4.2 删除/更改

```s
DROP VIEW balance_by_clients
```

```s
CREATE OR REPLACE VIEW balance_by_client AS  -- 重复执行不会报错
SELECT
    c.client_id,
    c.name,
    SUM(i.invoice_total - i.payment_total) AS balance
FROM clients c
JOIN invoices i USING (client_id)
GROUP BY client_id, name
```

## 4.3 updatable view

可以在select中使用view

当view中不含
  DISTICT
  Aggregate func
  GROUP BY / HAVING
  UNION
可以使用insert/update/delete对view做出更改

```s
CREATE OR REPLACE VIEW invoices_with_balance AS
SELECT 
    invoice_id,
    number,
    client_id,
    invoice_total,
    payment_total,
    invoice_total - payment_total AS balance,
    invoice_date,
    due_date,
    payment_date
 FROM invoices
 WHERE (invoice_total - payment_total) > 0  -- 不能使用别名
```

删除

```s
DELETE FROM invoices_with_balance
WHERE invoice_id = 1
```

更改

```s
UPDATE invoices_with_balance
SET due_date = DATE_ADD(DUE_DATE, INTERVAL 2 DAY)
WHERE invoice_id = 2
```

必须包含underlying table的所有字段，才可以对view进行插入

```s
INSERT INTO invoices_with_balance  -- 这里插不进去
VALUES (
    DEFAULT(),
    '08-088-8888',
    1,
    200,
    100,
    '2019-10-01',
    '2020-10-01',
    '2020-01-01'
)
```

## 4.4 Tips

```s
UPDATE invoices_with_balance  -- 更改之后，这条记录从view中被过滤掉了
SET payment_total = invoice_total
WHERE invoice_id = 2
```

不希望删除/更改使得view中的记录被删除，加上WITH CHECK OPTITION，导致记录删除的语句会报错

```s
CREATE OR REPLACE VIEW invoices_with_balance AS
SELECT 
    invoice_id,
    number,
    client_id,
    invoice_total,
    payment_total,
    invoice_total - payment_total AS balance,
    invoice_date,
    due_date,
    payment_date
 FROM invoices
 WHERE (invoice_total - payment_total) > 0  -- 不能使用别名
 WITH CHECK OPTION
```
