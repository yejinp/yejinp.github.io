<h1>cross join、inner join、left join、right join和full join的区别和联系</h1>

<h1>自然连接，内连接，左连接，右连接和全连接等各种连接操作之间的区别和联系</h1>

* **自然连接(natural join)**操作符对两个关系进行操作，并生成一个新的关系。 与两个关系上的笛卡尔积产生的结果不同，笛卡尔积对第一个关系中的每一个元组与第二个关系中的每一个元组。**自然连接**只考虑那些两个关系中在一些属性上有相同值的元组。

图解四大连接

图解用到的表：
表student

| id |name |dept_name|
|-|-|-|
| 0| Jane|Sci. |
| 1| Lucy|History |
| 2| YiFei |Finance |

表course

|id | name| year  |
|-|-|-|
|0||A|
|1|2|B|
|2|1|C|
|3|1|D|

表takes
| id | |year | grade |
|-|-|-|
| | | |
| | | |


* 外联操作符用一种与联操作类似的方式工作，但保留连接中结果为null值时会丢失的元组。

内连接(inner join,或称等值连接)：返回两张表中匹配的记录；
除了内连接，还有外连接

* 有三种类型的外联：
 - 左外连接，返回两张表匹配的记录，以及左表中没有匹配的记录
 - 右外连接，返回两张表匹配的记录，以及右表中没有匹配的记录
 - 全外连接，返回两张表匹配的记录，以及左右两表中各自没有匹配的记录


* 示例：
 - 内连接 
 - 左外连接, 对于上面的表A和表B，进行左连接
 - 右外连接
 - 全外连接

* Join类型
 - 内连接
 - 左外连接
 - 右外连接
 - 全连接

* Join 条件
 - natural
 - on < predicate>
 - using (A1, A2, . . ., An)


* **参考资料：**

  - Abraham Silberschatz, Henry F. Korth, and S. Sudarshan, "[Database System Concepts](https://www.amazon.com/dp/0073523321)", McGraw-Hill Education, ISBN-13: 978-0073523323