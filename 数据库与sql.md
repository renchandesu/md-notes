```mysql
-- drop table test if EXISTS;
-- create table test(
-- 	id int primary key auto_increment,
-- 	name VARCHAR(20) DEFAULT '1234'
-- )charset = utf8;

-- alter table test rename as test1
-- ALTER TABLE test1 MODIFY COLUMN name VARCHAR(32) DEFAULT NULL;
-- ALTER TABLE test1 add age int;
-- ALTER TABLE test1 drop COLUMN age;
-- alter table test1 add index(name);
-- alter table test1 add UNIQUE(age);

create index idx_duration on examination_info(duration);
create unique index uniq_idx_exam_id on examination_info(exam_id);
create fulltext index full_idx_tag on examination_info(tag);
```

```
url: jdbc:mysql://localhost:3306/blog?serverTimezone=Asia/Shanghai
username: root
password: ying1234
druid:
  driver-class-name: com.mysql.cj.jdbc.Driver
```

```
select count(id) as total_pv, count(submit_time) as complete_pv, count(distinct exam_id and score is not null) //count函数内的表达式
as complete_exam_cnt 
from exam_record
```

```
select date_format(submit_time,"%Y%m"),
format(count(distinct uid,date_format(submit_time,"%Y%m%d"))/count(distinct uid),2),
count(distinct uid)  
from exam_record 
where submit_time is not null 
and year(submit_time) = 2021
group by date_format(submit_time,"%Y%m");
```

```
select   //any_Value last_day函数
DATE_FORMAT(submit_time,'%Y%m') as submit_month,
count(submit_time) as month_q_cnt,
round(count(submit_time) / any_value(day(last_day(submit_time))) ,3) as avg_day_q_cnt
from practice_record where score is not null 
and year(submit_time) = '2021'
group by DATE_FORMAT(submit_time,'%Y%m')
```

```
select uid
        , sum(if(submit_time is null, 1, 0)) as incomplete_cnt  // if函数的用法
        , sum(if(submit_time is null, 0, 1)) as complete_cnt    
        , group_concat(distinct CONCAT(DATE_FORMAT(start_time, '%Y-%m-%d'),':',tag) separator ';') as detail  // group_concat的用法 连接这个分组里面的内容 分割符  concat函数，连接内容
from exam_record er join examination_info ei on er.exam_id = ei.exam_id
where YEAR(start_time) = 2021
group by uid
having incomplete_cnt>1
and incomplete_cnt<5
and complete_cnt >= 1
order by incomplete_cnt desc
```

```
TIMESTAMPDIFF
语法：TIMESTAMPDIFF(unit,datetime_expr1,datetime_expr2)

结果：返回（时间2-时间1）的时间差，结果单位由interval参数给出。
```

## 事务

![image-20230712223453135](/Users/renchan/Library/Application Support/typora-user-images/image-20230712223453135.png)



## MySQL性能问题

### 删除数据的方法

如果一次要删除大量数据的话，最好使用truncate或者drop,使用delete会一条条删除数据，很慢

### explain字段解释

explain 可以用来帮助我们分析查询语句的性能问题，有没有走索引，各个字段的含义如下

1. id: 查询块的唯一标识符，用于标识查询块的执行顺序。
2. select_type: 查询类型，表示查询的类型，如简单查询、联接查询、子查询等。
   1. SIMPLE: 简单查询，不包含子查询或联接操作。
   2. PRIMARY: 主查询，嵌套在子查询中的查询。
   3. SUBQUERY: 子查询，嵌套在其他查询中的查询。
   4. DERIVED: 派生表，派生查询的结果作为临时表使用。
   5. UNION: UNION 查询中的第二个或后续查询。
   6. UNION RESULT: UNION 查询的结果。
   7. DEPENDENT SUBQUERY: 依赖子查询，子查询的结果依赖于外部查询。
   8. DEPENDENT UNION: 依赖 UNION，UNION 查询的结果依赖于外部查询。
   9. DEPENDENT UNION RESULT: 依赖 UNION RESULT，UNION 查询结果的依赖结果依赖于外部查询。
   10. UNION SUBQUERY: UNION 子查询，嵌套在 UNION 查询中的子查询。
   11. MATERIALIZE: 用于存储结果集的临时表。
3. table: 表名，表示查询涉及的表。
4. partitions: 分区信息，如果查询涉及到分区表，则显示分区信息。（不常用）
5. type: 访问类型，表示查询时使用的访问方法，常见的访问类型包括：
   - ALL: 全表扫描，需要扫描整个表。
   - index_merge 对同一表多次进行不同的索引扫描，并合并结果
   - index: 使用索引扫描。
   - range: 使用索引范围扫描。
   - ref: 使用非唯一索引或唯一索引的前缀进行查找。
   - eq_ref: 使用唯一索引进行查找。
   - const: 使用常量进行扫描，说明条件是给定的常量，mysql直接去指定位置读取，这个是性能最好的。通常发生在使用主键或唯一索引进行等值匹配。
6. possible_keys: 可能使用的索引，表示查询可能使用的索引。
7. key: 实际使用的索引，表示查询实际使用的索引。
8. key_len: 使用的索引长度，表示索引字段的长度。
9. ref: 表示与索引比较的列或常数。
10. rows: 估计的扫描行数，表示在执行查询时估计需要扫描的行数。
11. filtered: 过滤的行百分比，表示查询条件过滤后保留的行的百分比。
12. Extra: 额外的信息，提供了关于查询执行的其他详细信息，如使用了临时表、使用了文件排序等。



### 索引有用的地方

- 快速查找符合where条件的记录

- 快速确定候选集。若where条件使用了多个索引字段，则MySQL会优先使用能使候选记录集规模最小的那个索引，以便尽快淘汰不符合条件的记录。

- 如果表中存在几个字段构成的联合索引，则查找记录时，这个联合索引的最左前缀匹配字段也会被自动作为索引来加速查找。
   例如，若为某表创建了3个字段(c1, c2, c3)构成的联合索引，则(c1), (c1, c2), (c1, c2, c3)均会作为索引，(c2, c3)就不会被作为索引，而(c1, c3)其实只利用到c1索引。

- 多表做join操作时会使用索引（如果参与join的字段在这些表中均建立了索引的话）

- 若某字段已建立索引，求该字段的min()或max()时，MySQL会使用索引

- 对建立了索引的字段做sort或group操作时，MySQL会使用索引

•B+Tree可被用于sql中对列做比较的表达式，如=, >, >=, <, <=及between操作

- 若like语句的条件是不以通配符开头的常量串，MySQL也会使用索引
   比如，SELECT * FROM tbl_name WHERE key_col LIKE 'Patrick%'或SELECT * FROM tbl_name WHERE key_col LIKE 'Pat%_ck%'可以利用索引，而SELECT * FROM tbl_name WHERE key_col LIKE '%Patrick%'（以通配符开头）和SELECT * FROM tbl_name WHERE key_col LIKE other_col（like条件不是常量串）无法利用索引。
   若已对名为col_name的列建了索引，则形如"col_name is null"的SQL会用到索引

- 对于联合索引，sql条件中的最左前缀匹配字段会用到索引

- 若sql语句中的where条件不只1个条件，则MySQL会进行Index Merge优化来缩小候选集范围

### 什么时候会索引失效

- 联合索引的最左前缀匹配未匹配上的情况
- 模糊搜索最左前缀匹配失效的情况
- 条件中的计算函数，或者参与计算
- 使用不等于 或者 !=NULL 注意is NULL走索引
- 使用or条件时,某个字段没有索引则不会走索引
- 索引列使用 in 语句(小部分情况下会失效)



### 慢日志

MySQL提供的一种日志记录，它用来记录在MySQL中响应时间超过阀值的语句，具体指运行时间超过long_query_time值或没用到索引的SQL，则会被记录到慢查询日志中。

`mysqldumpslow`

##### or 的性能问题

or 的索引失效问题：对于单列来说，用or是没有任何问题的，但是or涉及到多个列的时候，每次select只能选取一个index，如果选择了area，population就需要进行table-scan，即全部扫描一遍

union与or的比较：使用union就可以解决这个问题，分别使用area和population上面的index进行查询。 但是这里还会有一个问题就是，UNION会对结果进行排序去重，可能会降低一些performance(这有可能是方法一比方法二快的原因），所以最佳的选择应该是两种方法都进行尝试比较。
