数据类型:
- string
- numeric
- date and time
- blob (storing binary data)
- spatial (storing geographical values)

# 1. string types

## 1.1 类型
(1) CHAR(X)  
  
fixed-length strings

(2) VARCHAR(X)  
  
variable-length strings;
  
max charactres: 65,535 (64KB);
  
可以加索引

(3) MEDIUMTEXT
  
max characters: 16 million (16MB); 
  
storing json object, cs view strings, ...

(4) LONGTEXT  
  
max: 4GB; 
  
storing text book, years of log files, ...

(5) TINYTEXT 

max: 255 characters

(6) TEXT  

max: 65000 characters;

范围与varchar类似，但是不可以加索引

## 1.2 惯例
- VARCHAR(50)  for short strings
- VARCHAR(255)  for medium-length strings

## 1.3 bytes
- English letters use 1 byte;
- European and Middle-eastern languages use 2 bytes;
- Asian languages use 3 bytes (CHAR(10)  30 bytes)

## 1.4 注
如果strings超出范围，会阶段

# 2. integer types

## 2.1 类型

|类型|bytes|范围|
|---|---|---|
|TINYINT|1|[-128, 127]
|UNSIGNED TINYINT||[0, 255]|
|SMALLINT|2|[-32K, 32K]|
|MEDIUMINT|3|[-8M, 8M]|
|INT|4|[-2B, 2B]|
|BIGINT|8|[-9Z, 9Z]|

## 2.2 zero fill

用途：用0填充，使得数字有相同的位数

INT(4) => 0001

决定MySQL的展示方式，而不是存储方式

## 2.3 注

如果数值超出范围，会报错

尽可能使用满足需求的smallest data type

# 3. fixed-point and floating-point types













