# SQL优化的一般步骤是什么，怎么看执行计划（explain），如何理解其中各个字段的含义。

- show status 命令了解各种 sql 的执行频率
- 通过慢查询日志定位那些执行效率较低的 sql 语句
- explain 分析低效 sql 的执行计划（这点非常重要，日常开发中用它分析Sql，会大大降低Sql导致的线上事故）
- show profiles开启性能

[[explain]]

[[慢查询日志&开启优化]]