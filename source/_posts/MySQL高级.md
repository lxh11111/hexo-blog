---
title: MySQL高级
date: 2024-3-1
tags:
  - java
description: MySQL存储引擎、索引、SQL优化、视图/存储过程/触发器、锁、InnoDB引擎、MySQL管理等
abbrlink: 10006
categories: 
  - Learn
  - MySQL
---

### 存储引擎

#### MySQl体系结构

![alt text](https://pic.imgdb.cn/item/65e201839f345e8d03418034.png)

#### 存储引擎简介

* 存储数据、建立索引、更新查询数据的实现方式
* 基于**表**
* 默认**InnoDB引擎**
* 创建表的时候指定引擎``engine=InnoDB``

#### 存储引擎特点

* InnoDB
  支持事务、行级锁、外键
  逻辑存储结构：![alt text](https://pic.imgdb.cn/item/65e326f39f345e8d03e11cc0.jpg)
* MyISAM
  支持表锁，不支持行锁、事务、外键
* Memory
  存储在内存中，只能作为临时表或缓存
  hash索引

#### 存储引擎选择

![alt text](https://pic.imgdb.cn/item/65e2e2409f345e8d03ffb27f.png)

#### Linux下安装MySQL

* 根据redis，还是建议直接在linux里安装mysql
* 安装到windows需上传至linux并解压

### 索引

#### 索引概述

高效获取数据的数据结构

#### 索引结构

不同的存储引擎有不同的索引结构

* B+Tree索引 大部分引擎都支持
* Hash索引 InnoDB不支持
* R-tree 空间索引
* Full-text 全文索引--倒排索引

##### B+树索引

* 二叉树缺点：顺序插入容易**形成一个链表，且大数据量下层级很深**
* 红黑树缺点：本质还是二叉树，**大数据量下层级深**
* B树(多路平衡查找树)缺点：叶子节点和非叶子节点均储存数据，导致一页中存储的键值减少，增加树高度
* B+树 均在**叶子节点**上，且添加双向指针(双向链表)

**MySQL里的B+树**
增加了优化
![alt text](https://pic.imgdb.cn/item/65e326ab9f345e8d03dfc3b6.jpg)

##### Hash索引

通过hash算法计算值映射
InnoDB不支持hash索引，但有自适应功能，指定条件下根据B+索引自动创建
**只支持等值匹配，不支持范围匹配及排序**--所以选择B+树索引

#### 索引分类

* 主键索引
* 唯一索引
* 常规索引
* 全文索引

**InnoDB也可以分为两种**

* 聚集索引
  * 存在主键 **主键索引**为聚集索引
  * 不存在主键 第一个唯一索引为聚集索引
  * 均没有 自动生成rowid作索引
* 二级索引
* 回表查询
  ![alt text](https://pic.imgdb.cn/item/65e2e2499f345e8d03ffc98d.png)
  
#### 索引语法

* 创建索引
``create [unique|fulltext] index index_name on table_name (index_col_name,...);``
* 查看索引
``show index from table_name;``
* 删除索引
``drop index index_name on table_name;``

#### SQL性能分析

* 查看SQL执行频率
  ``show global status like 'Com_______'``
  七个下划线
  查看增删改查的频率--查询频率高要进行sql优化
* 慢查询日志
  * 定位sql语句，查看哪一句效率低
  * 首先在``/etc/my.cnf``中开启慢查询日志
  ``slow_query_log=1 #开启``
  * ``long_query_time=2 #设置超过时间``
  超时则会在慢查询日志中输出
  * 慢查询日志在``/var/lib/mysql/localhost-slow.log``里
  ``tail -f localhost-slow.log``命令可查看慢查询日志尾部信息
* profile详情
  * 打开开关
    ``select @@have_profiling;``
    ``set profiling=1;``
  * 查看
  
    ```sql
    show profiles;
    show profile for query query_id;
    show profile cpu for query query_id;
    ```

* explain执行计划
  * 查询性能
  ``explain select 字段 from 表...``
  * 查询结果字段含义
  ![alt text](https://pic.imgdb.cn/item/65e2e24c9f345e8d03ffd43c.png)

#### 索引使用

* 最左前缀法则
  * 联合索引遵守最左前缀法则，从最左列开始且不跳过索引中的列，**跳过的话后面的字段索引会失效**
  * 最左索引存在即可，与后面字段位置顺序无关
  * 联合索引如果出现范围查询(>,<)，范围查询右侧的索引失效--**规避：尽量采用大于等于或小于等于**
* 索引失效
  * 在索引列上计算，会导致索引失效
  * 字符串类型字段使用查询不加引号，会导致索引失效
  * 尾部模糊匹配索引不失效，**头部模糊匹配**索引失效
  * or连接：如果or前的列有索引但后面的列没有，所涉及的索引均失效。**两侧均有索引**才会有效
  * 数据分布情况：如果评估全表扫描比索引效率高，则不使用索引
* SQL提示
  * 加入人为的提示来优化，在from 表名后指定
  ``use index`` 使用某个索引
  ``ignore index``  不使用某个索引
  ``force index``  强制使用某个索引
  * use只是建议mysql使用该索引，最终使用哪个**取决于mysql的判断**
* 覆盖索引
  * 尽量不写``select *``，容易产生回表查询(二级索引没覆盖到需要查询的字段，需要根据返回的id回查聚集索引)，性能低
  * 查询性能extra字段：
    ![alt text](https://pic.imgdb.cn/item/65e312039f345e8d038bf803.png)
* 前缀索引
  * 字段类型为字符串或大文本时需要索引很长的字符串--只将一部分前缀建立索引
  * ``create index idx_xxx on table table_name(column(n));``n为截取前缀长度
  * 长度的选择：根据不重复的索引值与数据表总记录数的**比值**(选择性)决定
* 联合索引
  * 如果有多个查询条件，考虑根据查询字段尽力联合索引，避免回表查询
  * 创建联合索引要**考虑字段顺序**--参考最左前缀法则

#### 索引设计原则

![alt text](https://pic.imgdb.cn/item/65e321d79f345e8d03c9457e.jpg)
![alt text](https://pic.imgdb.cn/item/65e322199f345e8d03ca4c62.jpg)
![alt text](https://pic.imgdb.cn/item/65e3223c9f345e8d03cad74d.jpg)

### SQL优化

#### 插入数据

插入多条数据优化

* 批量插入
  大批量数据采用insert插入性能低，可以采用mysql提供的**load指令**插入

  ```sql
  -- 客户端连接服务端时，加上参数 -–local-infile
  mysql –-local-infile -u root -p
  -- 设置全局参数local_infile为1，开启从本地加载文件导入数据的开关
  set global local_infile = 1;
  -- 执行load指令将准备好的数据，加载到表结构中
  load data local infile '/root/sql1.log' into table tb_user fields
  terminated by ',' lines terminated by '\n' ;
   ```

* 手动事务提交
  
  ```sql
  start transaction;
  insert into... values...;
  insert...;
  commit;
  ```

* 主键顺序插入

#### 主键优化

* InnoDB中表数据是根据主键顺序存放的
* **页分裂，页合并**：主键乱序插入时可能会造成页分裂，删除可能会造成页合并
* 主键设计原则
  * 尽量降低主键长度
  * 尽量**顺序插入**
  * 尽量不使用UUID或自然主键如身份证号作主键

#### order by优化

* extra字段
  * using filesort
  通过表的索引或全表扫描，在排序缓冲区完成排序
  * using index
  通过有序索引扫描直接返回，不需要额外排序
  前提：使用覆盖索引
* 优化
  * 尽量使用**有序索引**扫描，且**使用覆盖索引**
  * 使用不了有序索引--大数据情况下，可以增大排序缓冲区大小

#### group by优化

* 优化
建立适当索引提高分组效率(联合索引)
满足最左前缀法则

#### limit优化

* 优化
  * 数据量大的情况下，使用limit分页查询很慢，使用orderby排序查询后再查询，降低查询时间
  * 创建**覆盖索引+子查询**
    ``select * from tb1 a,(select id from tb2 order by limit 10000000,10) b where a.id=b.id;``
    说明：直接进行子查询，包含了limit字段会报错，所以采用多表查询，将子查询得到的结果视为一张表，进行多表查询

#### count优化

* count()字段说明
  ![alt text](https://pic.imgdb.cn/item/65e349e99f345e8d03863358.jpg)
* 优化
  效率：字段<主键<1≈*(不需要取值则效率高)
  尽量使用``count(*)``
  
#### update优化

* 优化
  * 根据索引字段更新
  * InnoDB行锁针对索引加锁，如果索引失效或不使用该索引，就会从行锁升级为表锁，并发性能降低

### 视图/存储过程/触发器

#### 视图

##### 介绍

* 一种虚拟存在的表，不存储数据，保存sql语句，数据均在基表中
* 语法
  * 创建
  ``CREATE [OR REPLACE] VIEW 视图名称[(列名列表)] AS SELECT语句 [ WITH [CASCADED | LOCAL ] CHECK OPTION ]
  ``
  * 查询
  ``
  查看创建视图语句：SHOW CREATE VIEW 视图名称;
  查看视图数据：SELECT * FROM 视图名称 ...... ;
  ``
  * 修改
  ``
  方式一：CREATE [OR REPLACE] VIEW 视图名称[(列名列表)] AS SELECT语句 [ WITH
  [ CASCADED | LOCAL ] CHECK OPTION ]``
  ``方式二：ALTER VIEW 视图名称[(列名列表)] AS SELECT语句 [ WITH [ CASCADED |LOCAL ] CHECK OPTION ]
  ``
  * 删除
  ``DROP VIEW [IF EXISTS] 视图名称 [,视图名称] ..``

##### 检查选项

* 使用``with check option``创建视图，mysql通过视图检查每个更改行，使其符合视图定义
* 允许基于一个视图创建另一个，检查**依赖视图**中的规则保持一致性
* 检查范围
  * cascaded(默认)
    检查当前视图和其依赖的视图
    ![alt text](https://pic.imgdb.cn/item/65e352589f345e8d03a8cb53.jpg)
  * local
    若依赖视图无条件检查，则不检查
    ![alt text](https://pic.imgdb.cn/item/65e353469f345e8d03ac091a.jpg)

##### 更新

* 更新视图条件：
  视图行与基础表行必须**存在一对一的关系**
* 包含以下则视图不可更新：
  聚合/窗口函数、distinct、group by、having、union
* 作用
  * 简单 简化操作与理解
  * 安全 只查询修改所见到的数据
  * 数据独立 屏蔽真实表结构变化带来的影响

#### 存储过程

##### 基础

* 介绍
  事先经过编译并存储在数据库中的一段sql语句，简化开发，减少数据传输，即**sql语句的封装与重用**
* 语法
  * 创建

    ```sql
    create procedure name(参数)
    begin
      -sql语句
    end;
    ```

  * 注意：在命令行中执行上述语句会报错，因为执行到sql语句分号时视为语句结束，并没有识别到end后的分号
    **解决**：使用``delimiter $$``设置结束符号(这里设置为``$$``)，这样将end后的分号改为$$，就可以执行整个创建语句，注意结束后记得将结束符号改回``;``

  * 调用
    ``call name(参数);``
  * 查看
  
    ```sql
    select * from information_schema routines where routine_schema='xxx'; #查询指定数据库存储过程及状态信息
    show create procedure name; #查询某个存储过程定义
    ```

  * 删除
    ``drop procedure [if exists] name;``

##### 变量

* 系统变量
  * 全局变量**global**/会话变量**session**
  * 查看

    ```sql
    SHOW [ SESSION | GLOBAL ] VARIABLES ; -- 查看所有系统变量
    SHOW [ SESSION | GLOBAL ] VARIABLES LIKE '......'; -- 可以通过LIKE模糊匹配方式查找变量
    SELECT @@[SESSION | GLOBAL] 系统变量名; -- 查看指定变量的值
    ```

  * 设置

  ```sql
  SET [ SESSION | GLOBAL ] 系统变量名 = 值 ;
  SET @@[SESSION | GLOBAL]系统变量名 = 值 ;
  ```

* 用户自定义变量
  **不需要声明**，直接``@name``即可
  * 赋值
    方式一：

    ```sql
    SET @var_name = expr [, @var_name = expr] ... ;
    SET @var_name := expr [, @var_name := expr] ... ;
    ```

    方式二：

    ```sql
    SELECT @var_name := expr [, @var_name := expr] ... ;
    SELECT 字段名 INTO @var_name FROM 表名;
    ```
  
    没赋值直接使用也不会报错，返回null
  * 使用
    ``select @var_name;``
* 局部变量
  访问之前**需要声明**，在begin~end间生效
  * 声明
    ``DECLARE 变量名 变量类型 [DEFAULT ... ] ;``
  * 赋值

    ```sql
    SET 变量名 = 值 ;
    SET 变量名 := 值 ;
    SELECT 字段名 INTO 变量名 FROM 表名 ... ;
    ```

##### 条件判断

* if
  * 语法

  ```sql
  IF 条件1 THEN
  .....
  ELSEIF 条件2 THEN -- 可选
  .....
  ELSE -- 可选
  .....
  END IF;
  ```

* 参数
  * 类型
    * ``in`` 输入参数(默认)
    * ``out`` 输出，返回值
    * ``inout`` 既可以输入也可以输出
  * 用法

    ```sql
    CREATE PROCEDURE 存储过程名称 ([ IN/OUT/INOUT 参数名 参数类型 ])
    BEGIN
    -- SQL语句
    END ;
    ```

* case
  * 语法一
  
  ```sql
  -- 含义： 当case_value的值为 when_value1时，执行statement_list1，当值为 when_value2时，
  执行statement_list2， 否则就执行 statement_list
  CASE case_value
  WHEN when_value1 THEN statement_list1
  [ WHEN when_value2 THEN statement_list2] ...
  [ ELSE statement_list ]
  END CASE;
  ```

  * 语法二
  
  ```sql
  -- 含义： 当条件search_condition1成立时，执行statement_list1，当条件search_condition2成
  立时，执行statement_list2， 否则就执行 statement_list
  CASE
  WHEN search_condition1 THEN statement_list1
  [WHEN search_condition2 THEN statement_list2] ...
  [ELSE statement_list]
  END CASE;
  ```

##### 循环

* while
  * 语法
  
  ```sql
  -- 先判定条件，如果条件为true，则执行逻辑，否则，不执行逻辑
  WHILE 条件 DO
  SQL逻辑...
  END WHILE;
  ```

* repeat
  * 语法
  
  ```sql
  -- 先执行一次逻辑，然后判定UNTIL条件是否满足，如果满足，则退出。如果不满足，则继续下一次循环
  REPEAT
  SQL逻辑...
  UNTIL 条件
  END REPEAT;
  ```

* loop
  * ``leave`` 退出循环
  * ``iterate`` 跳过当前，进入下一次循环
  * 语法
  
  ```sql
  [begin_label:] LOOP
  SQL逻辑...
  END LOOP [end_label];
  LEAVE label; -- 退出指定标记的循环体
  ITERATE label; -- 直接进入下一次循环
  ```
  
##### 游标

用于**存储查询结果集**的数据类型，存储过程和函数中使用游标对结果集循环处理

* 声明
  需要先声明普通变量，再声明游标
  ``DECLARE 游标名称 CURSOR FOR 查询语句 ;``
* open
  ``OPEN 游标名称 ;``
* fetch
  ``FETCH 游标名称 INTO 变量 [, 变量 ] ;``
* close
  ``CLOSE 游标名称 ;``

* 条件处理函数
  解决使用游标获取数据时循环条件的处理
  * 语法
  
  ```sql
  DECLARE handler_action HANDLER FOR condition_value [, condition_value]
  ... statement ;
  handler_action 的取值：
  CONTINUE: 继续执行当前程序
  EXIT: 终止执行当前程序
  condition_value 的取值：
  SQLSTATE sqlstate_value: 状态码，如 02000
  SQLWARNING: 所有以01开头的SQLSTATE代码的简写
  NOT FOUND: 所有以02开头的SQLSTATE代码的简写
  SQLEXCEPTION: 所有没有被SQLWARNING 或 NOT FOUND捕获的SQLSTATE代码的简写
  ```
  
  * [状态码官方文档](https://dev.mysql.com/doc/mysql-errors/8.0/en/server-error-reference.html)
  
#### 存储函数

* 有返回值的存储过程，存储函数的参数只能是IN类型的
* 语法
  
  ```sql
  CREATE FUNCTION 存储函数名称 ([ 参数列表 ])
  RETURNS type [characteristic ...]
  BEGIN
  -- SQL语句
  RETURN ...;
  END ;
  ```

* characteristic说明
  * DETERMINISTIC：相同的输入参数总是产生相同的结果
  * NO SQL ：不包含 SQL 语句
  * READS SQL DATA：包含读取数据的语句，但不包含写入数据的语句
* 使用较少，而且要求有返回值，通常可以使用**存储过程**代替

#### 触发器

* 介绍：
  与表有关的数据库对象，指在insert/update/delete之前(BEFORE)或之后(AFTER)，触发并执行触发器中定义的SQL语句集合。这种特性可以协助应用在数据库端确保数据的完整性、日志记录、数据校验等操作
* 使用别名**OLD和NEW**来引用触发器中发生变化的记录内容，与其他的数据库相似。现在触发器还只支持**行级触发**，不支持语句级触发
* 类型
  ![alt text](https://pic.imgdb.cn/item/65e49b919f345e8d0317e8a1.jpg)
* 语法
  * 创建
  
  ```sql
  CREATE TRIGGER trigger_name
  BEFORE/AFTER INSERT/UPDATE/DELETE
  ON tbl_name FOR EACH ROW -- 行级触发器
  BEGIN
  trigger_stmt ;
  END;
  ```

  * 查看
  
  ```sql
  SHOW TRIGGERS ;
  ```
  
  * 删除
  
  ```sql
  DROP TRIGGER [schema_name.]trigger_name ; -- 如果没有指定 schema_name，默认为当前数据库 。
  ```
  
### 锁

#### 全局锁

典型使用场景：做**全库的逻辑备份**

* 一致性备份语法
  * 加全局锁
  ``flush tables with read lock ;``
  * 数据备份
  ``mysqldump -uroot –p(密码) 数据库名 > 复制后数据库路径.sql``
  * 释放锁
  ``unlock tables ;``
* **弊端**
  * 如果在主库上备份，那么在备份期间都不能执行更新
  * 如果在从库上备份，那么在备份期间从库不能执行主库同步过来的二进制日志（binlog），会导致主从延迟
  * 在InnoDB引擎中可以在备份时加上参数 --single-transaction 参数来完成不加锁的一致性数据备份
  ``mysqldump --single-transaction -uroot –p xxx > xxx.sql``

#### 表级锁

##### 表锁

* 分类
  * 表共享读锁 read lock
  自身客户端只读不写，不影响其他客户端读，但**阻塞写**
  * 表独占写锁 write lock
  自身客户端可读写，**阻塞其他客户端读写**
* 语法
  * 加锁
  ``lock tables 表名 read/write``
  * 释放锁
  ``unlock tables / 客户端断开连接``

##### 元数据锁(MDL)

* 加锁过程是系统自动控制，无需显式使用，在访问一张表的时候会自动加上
* MDL锁主要作用是维护表元数据的数据一致性，在表上有活动事务的时候，不可以对元数据进行写入操作。为了**避免DML与DDL冲突**，保证读写的正确性
* 当对一张表进行增删改查的时候，加**MDL读锁(共享)**，当对表结构进行变更操作的时候，加**MDL写锁(排他)**
* ![alt text](https://pic.imgdb.cn/item/65e5ca679f345e8d03c98403.jpg)

##### 意向锁

避免DML在执行时，加的行锁与表锁的冲突，在InnoDB中引入了意向锁，使得表锁不用检查每行数据是否加锁，使用意向锁来**减少表锁的检查**

* 分类
  * 意向共享锁(IS): 由语句select ... lock in share mode添加。与表锁共享锁(read)兼容，与表锁排他锁(write)**互斥**
  * 意向排他锁(IX): 由insert、update、delete、select...for update添加。与表锁共享锁(read)及排他锁(write)都互斥，**意向锁之间不会互斥**
  * ``select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from performance_schema.data_locks;``

#### 行级锁

##### 分类

* 行锁（Record Lock）：锁定单个行记录的锁，防止其他事务对此行进行update和delete。在RC、RR隔离级别下都支持
* 间隙锁（Gap Lock）：锁定索引记录间隙（不含该记录），确保索引记录间隙不变，防止其他事务在这个间隙进行insert，产生幻读。在RR隔离级别下都支持
* 临键锁（Next-Key Lock）：行锁和间隙锁组合，同时锁住数据，并锁住数据前面的间隙Gap。在RR隔离级别下支持

##### 行锁

* 分类
  * 共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排它锁
  * 排他锁（X）：允许获取排他锁的事务更新数据，阻止其他事务获得相同数据集的共享锁和排他锁
  * ![alt text](https://pic.imgdb.cn/item/65e5d2e49f345e8d03ed06fe.jpg)
* 常见sql语句加的行锁
  ![alt text](https://pic.imgdb.cn/item/65e5d3339f345e8d03ee27b8.jpg)
* 默认情况下，InnoDB在 REPEATABLE READ事务隔离级别运行，InnoDB使用 next-key 锁进行搜索和索引扫描，以防止幻读
  * 针对**唯一索引**进行检索时，对已存在的记录进行等值匹配时，将会**自动优化为行锁**
  * InnoDB的行锁是**针对于索引**加的锁，不通过索引条件检索数据，那InnoDB将对表中的所有记录加锁，此时就会升级为表锁
  * ``select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from performance_schema.data_locks;``
  
##### 间隙锁&临键锁

默认情况下，InnoDB在 REPEATABLE READ事务隔离级别运行，InnoDB使用 next-key 锁进行搜索和索引扫描，以防止幻读

* 索引上的**等值查询(唯一索引)**，给不存在的记录加锁时, 优化为**间隙锁**
* 索引上的**等值查询(非唯一普通索引)**，向右遍历时最后一个值不满足查询需求时，next-key lock 退化为**间隙锁**
* 索引上的范围查询(唯一索引)--会访问到不满足条件的第一个值为止

注意：间隙锁唯一目的是**防止其他事务插入间隙**。间隙锁可以共存，一个事务采用的间隙锁不会阻止另一个事务在同一间隙上采用间隙锁。

### InnoDB引擎

#### 逻辑存储结构

![alt text](https://pic.imgdb.cn/item/65e6e8389f345e8d0322c09a.png)

* 表
  是InnoDB存储引擎逻辑结构的最高层， 如果用户启用了参数 innodb_file_per_table(在8.0版本中默认开启)，则每张表都会有一个表空间（xxx.ibd，一个mysql实例可以对应多个表空间，用于**存储记录、索引**等数据
* 段
  分为数据段（Leaf node segment）、索引段（Non-leaf node segment）、回滚段（Rollback segment），InnoDB是索引组织表，数据段就是B+树的**叶子节点**， 索引段即为B+树的**非叶子节点**。段用来管理多个Extent（区）
* 区
  表空间的单元结构，每个区的大小为1M。 默认情况下，InnoDB存储引擎页大小为16K， 即一个区中一共有64个连续的页
* 页
  是InnoDB 存储引擎磁盘管理的最小单元，每个页的大小默认为 **16KB**。为了保证页的连续性，InnoDB 存储引擎每次从磁盘申请 4-5 个区
* 行
  InnoDB 存储引擎数据是按行进行存放的。
  默认有两个隐藏字段：
  * Trx_id：每次对某条记录进行改动时，都会把对应的事务id赋值给trx_id隐藏列
  * Roll_pointer：每次对某条引记录进行改动时，都会把旧的版本写入到undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息
  
#### 架构

![alt text](https://pic.imgdb.cn/item/65e6ae819f345e8d0397e0be.jpg)

##### 内存结构

![alt text](https://pic.imgdb.cn/item/65e6aea89f345e8d03983ecd.jpg)

* Buffer Pool
  可以**缓存磁盘**上经常操作的真实数据，在执行增删改查操作时，先操作缓冲池中的数据（若缓冲池没有数据，则从磁盘加载并缓存），然后再以一定频率刷新到磁盘，从而减少磁盘IO，加快处理速度
  缓冲池以Page页为单位，底层采用**链表**数据结构管理Page
  * free page：空闲page，未被使用
  * clean page：被使用page，数据没有被修改过
  * dirty page：脏页，被使用page，数据被修改过，也中数据与磁盘的数据产生了不一致
* Change Buffer
  更改缓冲区（针对于**非唯一二级索引页**），在执行DML语句时，如果这些数据Page没有在Buffer Pool中，不会直接操作磁盘，而会将数据变更存在更改缓冲区 Change Buffer中，在未来数据被读取时，再将数据合并恢复到Buffer Pool中，再将合并后的数据刷新到磁盘中
* Adaptive Hash Index
  InnoDB存储引擎会监控对表上各索引页的查询，如果观察到在特定的条件下hash索引可以提升速度，则建立hash索引

* Log Buffer
  * 用来保存要写入到磁盘中的log日志数据（redo log 、undo log），默认大小为 16MB，日志缓冲区的日志会定期刷新到磁盘中。如果需要更新、插入或删除许多行的事务，增加日志缓冲区的大小可以节省磁盘 I/O
  * 参数
    ``innodb_log_buffer_size``：缓冲区大小
    ``innodb_flush_log_at_trx_commit``

##### 磁盘结构

![alt text](https://pic.imgdb.cn/item/65e6d4869f345e8d03f472f4.jpg)

* System Tablespace
  **更改缓冲区**的存储区域。如果表是在系统表空间而不是每个表文件或通用表空间中创建的，它也可能包含表和索引数据。(在MySQL5.x版本中还包含InnoDB数据字典、undolog等)
* File-Per-Table Tablespaces
  如果开启了innodb_file_per_table开关，则每个表的文件表空间包含单个InnoDB表的**数据和索引**，并存储在文件系统上的单个数据文件中
* General Tablespaces
  * 创建表空间
  ``CREATE TABLESPACE ts_name ADD DATAFILE 'file_name' ENGINE = engine_name;``
  * 创建时指定表空间
  ``CREATE TABLE xxx ... TABLESPACE ts_name;``
* Undo Tablespaces
  **撤销**表空间，MySQL实例在初始化时会自动创建两个默认的undo表空间（初始大小16M），用于存储undo log日志
* Temporary Tablespaces
  存储用户创建的**临时表**等数据
* Doublewrite Buffer Files
  双写缓冲区，innoDB引擎将数据页从Buffer Pool刷新到磁盘前，先将数据页写入**双写缓冲区**文件中，便于系统异常时恢复数据
* Redo Log
  用来实现事务的**永久性**，该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log）,前者是在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都会存到该日志中, 用于在刷新脏页到磁盘时,发生错误时, 进行数据恢复使用

##### 后台线程

**作用**：将缓存区的数据在合适的时机刷新到磁盘文件中
![alt text](https://pic.imgdb.cn/item/65e6d7299f345e8d03fa665b.jpg)

分类：

* Master Thread
  核心后台线程，负责调度其他线程，还负责将缓冲池中的数据异步刷新到磁盘中, 保持数据的一致性，还包括脏页的刷新、合并插入缓存、undo页的回收
* IO Thread
  InnoDB存储引擎中大量使用了AIO来处理IO请求, 这样可以极大地提高数据库的性能，IO Thread主要负责这些IO请求的回调
  ![alt text](https://pic.imgdb.cn/item/65e6d7f49f345e8d03fc3a61.jpg)
* Purge Thread
  用于回收事务已经提交了的undo log，在事务提交之后，undo log可能不用了，就用它来回收
* Page Cleaner Thread
  协助 Master Thread 刷新脏页到磁盘的线程，减轻 Master Thread 的工作压力，减少阻塞

#### 事务原理

![alt text](https://pic.imgdb.cn/item/65e6d9ae9f345e8d03005377.jpg)

* redo log
  **重做日志**，记录的是事务提交时数据页的物理修改，是用来实现事务的持久性
  * 件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log file）,前者是在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都存到该日志文件中, 用于在刷新脏页到磁盘,发生错误时进行数据恢复使用
  * ![alt text](https://pic.imgdb.cn/item/65e737bb9f345e8d032a3f2a.jpg)

* undo log
  **回滚日志**，用于记录数据被修改前的信息,提供**回滚**(保证事务的原子性)和**MVCC**(多版本并发控制),是**逻辑日志**
  * 销毁：undo log在事务执行时产生，事务提交时，并不会立即删除undo log，因为这些日志可能还**用于MVCC**
  * 存储：undo log采用段的方式进行管理和记录，存放在前面介绍的 rollback segment回滚段中，内部包含1024个undo log segment

#### MVCC

##### 基本概念

* 当前读：
  读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。
  ``select ... lock in share mode(共享锁)`` ``select ...for update、update、insert、delete(排他锁)``都是一种当前读
* 快照读：
  简单的select（不加锁）就是快照读，读取的是记录数据的可见版本，有可能是历史数据(保证**可重复读**)，不加锁，是非阻塞读
  * ``Read Committed``：每次select，都生成一个快照读
  * ``Repeatable Read``：开启事务后第一个select语句才是快照读的地方
  * ``Serializable``：快照读会退化为当前读
* MVCC
  多版本并发控制
  指维护一个数据的多个版本，使得读写操作没有冲突，快照读为MySQL实现MVCC提供了一个非阻塞读功能

##### 隐藏字段

三个隐藏字段
![alt text](https://pic.imgdb.cn/item/65e7cfd49f345e8d0337f257.jpg)

##### undo log

* undo log
  回滚日志
* 版本链
  * 不同事务或相同事务对同一条记录进行修改，会导致该记录的undo log生成一条记录版本链表，链表的头部是最新的旧记录，链表尾部是最早的旧记录
  * ![alt text](https://pic.imgdb.cn/item/65e7f1459f345e8d039b0375.jpg)

##### readview

快照读 SQL执行时MVCC提取数据的依据，记录并维护系统当前活跃的事务（未提交的）id

* 核心字段
  ![alt text](https://pic.imgdb.cn/item/65e7f8e39f345e8d03b441e4.jpg)
* 版本链数据访问规则
  ![alt text](https://pic.imgdb.cn/item/65e7fb109f345e8d03bb5d3b.jpg)
* 不同隔离级别生成readview时机不同
  * ``READ COMMITTED``:在事务中每一次执行快照读时生成ReadView
  * ``REPEATABLE READ``:仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView

##### 原理

* RC
  ![alt text](https://pic.imgdb.cn/item/65e7fec99f345e8d03c64c41.jpg)
* RR
  ![alt text](https://pic.imgdb.cn/item/65e800a29f345e8d03cbd099.jpg)
  复用第一次快照读readview--实现**可重复读**
* 总体原理
  ![alt text](https://pic.imgdb.cn/item/65e801529f345e8d03cdc110.jpg)

### MySQL管理

#### 系统数据库

自带四大数据库
![alt text](https://pic.imgdb.cn/item/65e6dc7f9f345e8d0306a008.jpg)

#### 常用工具

##### mysql-客户端工具

```sql
语法 ：
mysql [options] [database]
选项 ：
-u, --user=name #指定用户名
-p, --password[=name] #指定密码
-h, --host=name #指定服务器IP或域名
-P, --port=port #指定连接端口
-e, --execute=name #执行SQL语句并退出
```

``-e``可以直接在mysql客户端执行sql语句，无需连接到数据库再执行

##### mysqladmin-管理工具

执行管理操作，检查服务器的配置和当前状态、创建并删除数据库等

```sql
语法:
mysqladmin [options] command ...
选项:
-u, --user=name #指定用户名
-p, --password[=name] #指定密码
-h, --host=name #指定服务器IP或域名
-P, --port=port #指定连接端口
```

##### mysqlbinlog-二进制日志查看工具

检查二进制日志文件的文本格式

```sql
语法 ：
mysqlbinlog [options] log-files1 log-files2 ...
选项 ：
-d, --database=name 指定数据库名称，只列出指定的数据库相关操作。
-o, --offset=# 忽略掉日志中的前n行命令。
-r,--result-file=name 将输出的文本格式日志输出到指定文件。
-s, --short-form 显示简单格式， 省略掉一些信息。
--start-datatime=date1 --stop-datetime=date2 指定日期间隔内的所有日志。
--start-position=pos1 --stop-position=pos2 指定位置间隔内的所有日志。
```

##### mysqlshow-查看数据库/表/字段统计信息

客户端对象查找工具，用来快速查找存在哪些数据库、数据库中的表、表中的列或者索引

```sql
语法 ：
mysqlshow [options] [db_name [table_name [col_name]]]
选项 ：
--count 显示数据库及表的统计信息（数据库，表 均可以不指定）
-i 显示指定数据库或者指定表的状态信息
示例：
#查询test库中每个表中的字段书，及行数
mysqlshow -uroot -p2143 test --count
#查询test库中book表的详细情况
mysqlshow -uroot -p2143 test book --count
```

##### mysqldump-数据备份工具

来备份数据库或在不同数据库之间进行数据迁移,备份内容包含创建表，及插入表的SQL语句

```sql
语法 ：
mysqldump [options] db_name [tables]
mysqldump [options] --database/-B db1 [db2 db3...]
mysqldump [options] --all-databases/-A
连接选项 ：
-u, --user=name 指定用户名
-p, --password[=name] 指定密码
-h, --host=name 指定服务器ip或域名
-P, --port=# 指定连接端口
输出选项：
--add-drop-database 在每个数据库创建语句前加上 drop database 语句
--add-drop-table 在每个表创建语句前加上 drop table 语句 , 默认开启 ; 不
开启 (--skip-add-drop-table)
-n, --no-create-db 不包含数据库的创建语句
-t, --no-create-info 不包含数据表的创建语句
-d --no-data 不包含数据
-T, --tab=name 自动生成两个文件：一个.sql文件，创建表结构的语句；一
个.txt文件，数据文件
```

##### mysqlimport/source-数据导入工具

客户端数据导入工具

* mysqlimport--导入mysqldump 加 -T 参数后导出的文本文件
  ``mysqlimport [options] db_name textfile1 [textfile2...]``
* source--导入sql文件
  ``source /root/xxxxx.sql``
