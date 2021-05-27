---
title: 剖析 mysql order by
date: 2021-05-24 22:27:46 PM
description: 剖析 mysql order by 是如何工作的
tags:
  - 数据库
  - mysql
categories: mysql
---

## 剖析 mysql order by 工作流程

实际开发中总是有 mysql 的 order by 的排序需求，开始 剖析 order by 的工作流程

假设 user 表 schema:

```sql
-- user table schema
CREATE TABLE `user` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;
```

从个 sql 开始

```sql
select name, age, addr from user where city="杭州" order by name limit 1000;
```

## 全字段排序

用 explain 看一下语句执行情况
![图1](https://raw.githubusercontent.com/zhaogaolong/PicGo/master/20210524194904.png)

Extra 字段中的 "Using filesort" 表示需要排序， Mysql 为每个线程分配一块用于 sort 的空间，成为 sort_buffer。

通常的语句执行流程：

1. 初始化 sort buffer， 放入 name, age, addr 三个字段。
2. 从索引 city，找到第一个满足 city="杭州" 的主键 id。
3. 从主键 id 去出整行， 取出 name, age, addr 三个字段值，放入 sort buffer 里。
4. 从 city 取出下一个主键 id
5. 循环 3, 4 步骤，知道 city != "杭州"为止。
6. sort buffer 中的数据按照字段 name 做快速排序
7. 按照排序结果取出前 1000 行返回

过程图：
![](https://raw.githubusercontent.com/zhaogaolong/PicGo/master/20210524201741.png)

图中的 name 排序这个动作有可能在**内存** 或 **文件** 中完成的。这个取决于排序内存所需的参数配置：`sort_buffer_size`。
排序的数据如果小于 `sort_buffer_size` 就在内存完成，反之就不得不利用磁盘临时文件辅助排序了。

验证

```sql

/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on';

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000;

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```

通过查看 `optimizer_trace` 查看 `num_of_tmp_files` 查看结果
![](https://raw.githubusercontent.com/zhaogaolong/PicGo/master/20210524205116.png)

`num_of_tmp_files` 表示排序过程需要多少个临时文件。如果内存放不下，就需要使用外部排序，一般使用归并排序。

如果 `sort_buffer_size` 空间越小，则使用 `num_of_tmp_files` (临时文件)数量越多。

## rowid 排序

上面的例子是从原表里读取一遍，剩下就在 sort buffer 里操作了，如果查询的字段很多，但内存里存放数据很少，就面临着使用多个临时文件，从而造成排序性能下降。

**如果 mysql 认为单行长度过大会怎么做?**

修改一个参数

```sql
SET max_length_for_sort_data = 16; -- default 1024
```

`max_length_for_sort_data`是设置排序字段想加的总长度

回到查询语句，city, name, age 三个字段的定义总长度是 36， 超过了 `max_length_for_sort_data` 设置的 16， 算法过程出现了什么变化。

新的算法放入 sort_buffer 只有排序的列（name） 和主键 id，但排好序就缺少了 city 和 age 字段，不能直接返回，这个执行流程就需要回一次表完成。
步骤：

1. 初始化 sort_buffer， 放入两个字段，name 和 id
2. 从 索引 city 上找到所有 `city="杭州"` 的主键 id
3. 从主键 id 中找到 name、id 两个字段放入 sort_buffer 里
4. 从索引 city 取出下一个主键 id，
5. 反复 3，4 步骤，直到 `city != "杭州"` 为止
6. 对 sort_buffer 里的数据按照 name 进行排序
7. 遍历所有的结果，取出前 1000 行，并按照 id 回到原表中取出 city, name, age 字段返回给客户端.

首先 examined_rows 还是 4000， 但是 `select @b-@a` 语句值变成了 5000 了，为啥捏？这是因为最后（第七步）有一次 1000 个数据需要回表查询。

![](https://raw.githubusercontent.com/zhaogaolong/PicGo/master/20210527193416.png)

- `sort_mode` 变成了 `sort_key, rowid`, 表示参与排序的只有 name 和 id 两个字段
- number_of_tmp_files 变成了 10， 是因为 4000 行的数据每一行都变小了，临时文件也变小了

## 全字段排序 VS rowid 排序

- mysql 担心排序内存小，会影响到效率，退化为 rowid 排序，这个需要回到原多表查询一次
- 如果内存足够大，直接在内存里完成
- 多使用内存，尽量少访问磁盘
- innoDB 表，rowid 排序需要回表，就造成多访问磁盘几次（索引里都是 xx，有可能在内存，有可能在磁盘上）,因此不被优先选择

## order by 不一定真的执行排序

### 普通索引

如果索引上天然取出的 name 就是排好序的，**那么数据就不需要排序了**

添加二级索引

```sql
alter table t add index city_user(city, name);
```

就使用索引的最左原则，name 就是排好序的了。
大概过程：

1. 从索引（city，name）中取出满足 `city="杭州"`条件的主键 id。
2. 从主键 id 中取出整行数据，取出 name、city、age 三个字段，返回
3. 从索引（city，name）取出下一个记录的主键 id
4. 反复 2、3 操作，直到查到第 1000 条记录，或者不满足 city="杭州" 的条件

总结：整个过程不需要排序动作，只需要扫描 1000 行就够了。

### 覆盖索引

如果普通二级索引上就有 name, city, age 三个字段了，那么就不需要回表查询了，直接在这个覆盖索引上完成数据查找并返回，效率更高

```sql
alter table t add index city_user_age(city, name, age);
```

执行过程：

1. 从索引(city, name, age) 中找到第一个 满足 city="杭州" 的条件记录， 取出 city, name, age 三个字段的值，作为结果集的一部分返回
2. 从索引(city, name, age) 取下一条记录， 同样取出三个字段，直接返回。
3. 重复 2 步骤，直到查到 1000 条记录，或不满足 city="杭州" 的条件结束循环。

### 总结

分析了排序的不同条件使用不同的排序算法。
从效率排序：覆盖索引 > 二级索引 > 全字段排序 > rowid 排序

## reference

- https://time.geekbang.org/column/article/73479
