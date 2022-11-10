<h1>inner join 和 left join、right join、full join的区别和联系</h1>

* **natural join**操作符对两个关系进行操作，并生成一个关系。 与两个关系上的笛卡尔积产生的结果不同，笛卡尔积对第一个关系中的每一个元组与第二个关系中的每一个元组。**natural join**只考虑那些两个关系中在一些属性上有相同值的元组。

* 外联操作符用一种与联操作类似的方式工作，但保留联中结果为null值时会丢失的元组。

* 有三种类型的外联：
 - 左外联，保留左外联左边关系的元组
 - 右外联，保留左外联右边关系的元组
 - 全外联，保留左外联两边关系的元组

* Join类型
 - 内联
 - 左外联
 - 右外联
 - 全联

* Join 条件
 - natural
 - on < predicate>
 - using (A1, A2, . . ., An)


* **参考资料：**

  - Abraham Silberschatz, Henry F. Korth, and S. Sudarshan, "[Database System Concepts](https://www.amazon.com/dp/0073523321)", McGraw-Hill Education, ISBN-13: 978-0073523323