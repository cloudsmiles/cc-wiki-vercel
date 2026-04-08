# 修改列change和modify区别

1. CHANGE old_col_name column_definition子句对列进行重命名。重命名时，需给定旧的和新的列名称和列当前的类型
2. 改列的类型而不是名称， CHANGE语法仍然要求旧的和新的列名称，即使旧的和新的列名称是一样的
3. 使用MODIFY来改变列的类型，此时不需要重命名：

例如： ALTER TABLE t1 MODIFY b BIGINT NOT NULL

reference:

[https://blog.csdn.net/weixin_42591413/article/details/113271951](https://blog.csdn.net/weixin_42591413/article/details/113271951)