数据类型:
- string
- numeric
- date and time
- blob (storing binary data)
- spatial (storing geographical values)

# 1. string types

- 类型
 - CHAR(X)  
    fixed-length strings
 - VARCHAR(X)  
    variable-length strings;
    max charactres: 65,535 (64KB);
    可以加索引
 - MEDIUMTEXT
    max characters: 16 million (16MB); 
    storing json object, cs view strings, ...
 - LONGTEXT  
    max: 4GB; 
    storing text book, years of log files, ...

 - TINYTEXT 
    max: 255 characters
 - TEXT  
    max: 65000 characters;
    范围与varchar类似，但是不可以加索引

- 惯例
VARCHAR(50)  for short strings
VARCHAR(255)  for medium-length strings

- bytes
English letters use 1 byte;
European and Middle-eastern languages use 2 bytes;
Asian languages use 3 bytes (CHAR(10)  30 bytes)

- 注
如果strings超出范围，会阶段

# 2. integer types

- 类型



- 注
如果数值超出范围，会报错















