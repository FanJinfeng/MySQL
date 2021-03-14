# 1. Trigger 触发器

定义：a block of SQL code, 在select, insert, update, delete语句之前或之后自动执行

作用：保证数据一致性

举例：在payments表中insert一条记录，invoices表中payment_total列的值应当相应改变

```s
DELIMITER $$

CREATE TRIGGER payments_after_insert  --table_after/before_statement
  AFTER INSERT ON payments
  FOR EACH ROW -- 对于插入的每一行，都要fire一次trigger（for MySQL）
BEGIN
END$$

DELIMITER
```

