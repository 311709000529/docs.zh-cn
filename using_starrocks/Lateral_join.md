# Lateral Join

## 功能

「行列转化」是ETL处理过程中常见的操作，Lateral 一个特殊的Join关键字，能够按照每行和内部的子查询或者 table function 关联，通过 Lateral 与 unnest 配合，我们可以实现一行转多行的功能。

## 使用说明

使用 Lateral join 需要打开新版优化器：

~~~SQL
set global enable_cbo = true;
~~~

Lateral 关键字语法说明：

~~~SQL
from table_reference join [lateral] table_reference
~~~

Unnest 关键字，是一种 table function，可以把数组类型转化成 table 的多行，配合 Lateral Join 就能实现我们常见的各种行展开逻辑。

~~~SQL
SELECT student, score , t.unnest
FROM tests
CROSS JOIN LATERAL UNNEST(scores) AS t ;

SELECT student, score, t.unnest
FROM tests, UNNEST(scores) AS t ;
~~~

这里第二种写法是第一种的简写，可以使用 Unnest 关键字省略 Lateral Join。

## 注意事项

* 当前版本 Lateral join 仅用于和 Unnest 函数配合使用，实现行转列的功能，后续会支持其他 table function / UDTF。
* 当前 Lateral join 还不支持子查询。
* 多列 unnest 操作需要指定别名。示例如下：

~~~sql
select v1,t1.unnest as v2,t2.unnest as v3 
from lateral_test , unnest(v2) t1 ,unnest(v3) t2;
~~~

## 示例

当前版本 StarRocks 支持 Bitmap、String、Array、Column 之间的转化关系如下：
![Lateral Join 中一些类型间的转化](../assets/lateral_join_type_convertion.png)

配合 Unnest，我们可以实现以下功能：

* String 展开成多行

~~~SQL
CREATE TABLE lateral_test2 (
    `v1` bigint(20) NULL COMMENT "",
    `v2` string NULL COMMENT "",
    `v3` string NULL COMMENT ""
)
duplicate key(v1)
DISTRIBUTED BY HASH(`v1`) BUCKETS 1
PROPERTIES (
    "replication_num" = "1",
    "in_memory" = "false",
    "storage_format" = "DEFAULT"
);

insert into lateral_test2 values (1, "1,2,3","1,2"), (2, "1,3","1,3");
~~~

~~~Plain Text
select * from lateral_test2;
+------+-------+------+
| v1   | v2    | v3   |
+------+-------+------+
|    1 | 1,2,3 | 1,2  |
|    2 | 1,3   | 1,3  |
+------+-------+------+

select v1,unnest from lateral_test2 , unnest(split(v2, ",")) ;
+------+--------+
| v1   | unnest |
+------+--------+
|    1 | 1      |
|    1 | 2      |
|    1 | 3      |
|    2 | 1      |
|    2 | 3      |
+------+--------+

---多列 unnest 时需要指定别名
select v1,t1.unnest as v2,t2.unnest as v3 from lateral_test2 , unnest(split(v2, ",")) t1,unnest(split(v3, ",")) t2 ;
+------+------+------+
| v1   | v2   | v3   |
+------+------+------+
|    1 | 1    | 1    |
|    1 | 1    | 2    |
|    1 | 2    | 1    |
|    1 | 2    | 2    |
|    1 | 3    | 1    |
|    1 | 3    | 2    |
|    2 | 1    | 1    |
|    2 | 1    | 3    |
|    2 | 3    | 1    |
|    2 | 3    | 3    |
+------+------+------+
~~~

* Array 类型展开成多行

~~~SQL
CREATE TABLE lateral_test (
    `v1` bigint(20) NULL COMMENT "",
    `v2` ARRAY<int> NULL COMMENT ""
) 
duplicate key(v1)
DISTRIBUTED BY HASH(`v1`) BUCKETS 1
PROPERTIES (
    "replication_num" = "1",
    "in_memory" = "false",
    "storage_format" = "DEFAULT"
);

insert into lateral_test values (1, [1,2]), (2, [1, null, 3]), (3, null);
~~~

~~~Plain Text
select * from lateral_test;

+------+------------+
| v1   | v2         |
+------+------------+
|    1 | [1,2]      |
|    2 | [1,null,3] |
|    3 | NULL       |
+------+------------+


select v1,v2,unnest from lateral_test , unnest(v2) ;

+------+------------+--------+
| v1   | v2         | unnest |
+------+------------+--------+
|    1 | [1,2]      |      1 |
|    1 | [1,2]      |      2 |
|    2 | [1,null,3] |      1 |
|    2 | [1,null,3] |   NULL |
|    2 | [1,null,3] |      3 |
+------+------------+--------+
~~~

* Bitmap 类型输出

~~~SQL
CREATE TABLE lateral_test3 (
`v1` bigint(20) NULL COMMENT "",
`v2` Bitmap BITMAP_UNION COMMENT ""
)
Aggregate key(v1)
DISTRIBUTED BY HASH(`v1`) BUCKETS 1;

insert into lateral_test3 values (1, bitmap_from_string('1, 2')), (2, to_bitmap(3));
~~~

~~~Plain Text
select v1, bitmap_to_string(v2) from lateral_test3;

+------+------------------------+
| v1   | bitmap_to_string(`v2`) |
+------+------------------------+
|    1 | 1,2                    |
|    2 | 3                      |
+------+------------------------+


insert into lateral_test3 values (1, to_bitmap(3));

select v1, bitmap_to_string(v2) from lateral_test3;

+------+------------------------+
| v1   | bitmap_to_string(`v2`) |
+------+------------------------+
|    1 | 1,2,3                  |
|    2 | 3                      |
+------+------------------------+


select v1,unnest from lateral_test3 , unnest(bitmap_to_array(v2)) ;

+------+--------+
| v1   | unnest |
+------+--------+
|    1 |      1 |
|    1 |      2 |
|    1 |      3 |
|    2 |      3 |
+------+--------+
~~~
