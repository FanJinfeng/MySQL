把 SQL code 放在 application code 外面，存储在数据库中的 stored precodure 或者 function 中。

# 1. stored procedure

stored precodure：一个数据库对象，存储 SQL code，在application code 中可以 call 这些 precodures 去 get 和 save 数据
  1. store and organize SQL
  2. faster execution
  3. 数据安全
 
## 1.1 创建

; default, 分隔查询语句

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

```








