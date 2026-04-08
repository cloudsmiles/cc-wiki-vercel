# mysql中in 和exists的区别

这个，跟一下demo来看更刺激吧，啊哈哈

假设表A表示某企业的员工表，表B表示部门表，查询所有部门的所有员工，很容易有以下SQL:

```
select * from A where deptId in (select deptId from B);
```

这样写等价于：

> 先查询部门表B
> 
> 
> select deptId from B
> 
> 再由部门deptId，查询A的员工
> 
> select * from A where A.deptId = B.deptId
> 

可以抽象成这样的一个循环：

```
   List<> resultSet ;    
		for(int i=0;i<B.length;i++) {          
			for(int j=0;j<A.length;j++) {          
				if(A[i].id==B[j].id) {             
					resultSet.add(A[i]);             
					break;          
				}       
			}    
		}
```

显然，除了使用in，我们也可以用exists实现一样的查询功能，如下：

```
select * from A where exists (select 1 from B where A.deptId = B.deptId);
```

因为exists查询的理解就是，先执行主查询，获得数据后，再放到子查询中做条件验证，根据验证结果（true或者false），来决定主查询的数据结果是否得意保留。

那么，这样写就等价于：

> select * from A,先从A表做循环
> 
> 
> select * from B where A.deptId = B.deptId,再从B表做循环.
> 

同理，可以抽象成这样一个循环：

```
   List<> resultSet ;    
		for(int i=0;i<A.length;i++) {          
			for(int j=0;j<B.length;j++) {          
				if(A[i].deptId==B[j].deptId) {             
					resultSet.add(A[i]);             
					break;          
				}       
			}    
		}
```

数据库最费劲的就是跟程序链接释放。假设链接了两次，每次做上百万次的数据集查询，查完就走，这样就只做了两次；相反建立了上百万次链接，申请链接释放反复重复，这样系统就受不了了。即mysql优化原则，就是小表驱动大表，小的数据集驱动大的数据集，从而让性能更优。

因此，我们要选择最外层循环小的，也就是，如果**B的数据量小于A，适合使用in，如果B的数据量大于A，即适合选择exists**，这就是in和exists的区别。