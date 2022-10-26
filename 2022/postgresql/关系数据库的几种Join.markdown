

nested loop join

    for each tuple tr in r do begin
        for each tuple ts in s do begin
            test pair (tr, ts) to see if they satisfy the join condition θ
            if they do, add tr ⋅ ts to the result;
        end
    end

merge join

hash join

* 参考资料：
 - [1] Abraham Silberschatz, Henry F. Korth, and S. Sudarshan, "Database System Concepts", McGraw-Hill Education, ISBN-13: 978-0073523323

- 