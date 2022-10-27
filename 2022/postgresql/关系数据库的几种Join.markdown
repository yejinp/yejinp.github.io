什么是Join操作： 依据一定的条件，查找2个表或者多个表

nested loop join
嵌套循环，简单的说就是暴力循环

算法描述如下：
    for each tuple tr in r do begin
        for each tuple ts in s do begin
            test pair (tr, ts) to see if they satisfy the join condition θ
            if they do, add tr ⋅ ts to the result;
        end
    end

merge join
归并Join
先将Outer Loop 和 Inner loop 分别排序，然后两边归并忽略不相同的项；由于两个Loop都是排序后的，因此排序后的性能为O（Outer+Inner），排序的性能为 O(Outer) + O(Inner)

hash join
通过计算hash值比较，忽略hash值不同的，这个地方不需要排序；


* 参考资料：
  - [1] Abraham Silberschatz, Henry F. Korth, and S. Sudarshan, "Database System Concepts", McGraw-Hill Education, ISBN-13: 978-0073523323