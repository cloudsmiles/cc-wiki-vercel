# join

## **SQL 语句中 left join 后用 on 还是 where**

前天写SQL时本想通过 A left B join on and 后面的条件来使查出的两条记录变成一条，奈何发现还是有两条。

**后来发现 join on and 不会过滤结果记录条数，只会根据and后的条件是否显示 B表的记录，A表的记录一定会显示。**

**不管and 后面的是A.id=1还是B.id=1,都显示出A表中所有的记录，并关联显示B中对应A表中id为1的记录或者B表中id为1的记录。**

运行sql :

```
select *from student sleftjoinclass con s.classId=c.idorderby s.id

```

[https://mmbiz.qpic.cn/mmbiz_png/x0kXIOa6owUmeEHjvYMC0PnEia20SJXMzeSlZ31SjByliaes5DtaHKDUNtcKqSy0NPh7KiaeXZLkDIOUibTTZpbiaZA/640?wx_fmt=png&wxfrom=13&tp=wxpic](https://mmbiz.qpic.cn/mmbiz_png/x0kXIOa6owUmeEHjvYMC0PnEia20SJXMzeSlZ31SjByliaes5DtaHKDUNtcKqSy0NPh7KiaeXZLkDIOUibTTZpbiaZA/640?wx_fmt=png&wxfrom=13&tp=wxpic)

运行sql :

```
select *from student sleftjoinclass con s.classId=c.idand s.name="张三"orderby s.id

```

[https://mmbiz.qpic.cn/mmbiz_png/x0kXIOa6owUmeEHjvYMC0PnEia20SJXMzTH63xwNxqNPrFCZjtD0oPsMGEOftCqItT8G8vnl0AafiaYkBjzwunPw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1](https://mmbiz.qpic.cn/mmbiz_png/x0kXIOa6owUmeEHjvYMC0PnEia20SJXMzTH63xwNxqNPrFCZjtD0oPsMGEOftCqItT8G8vnl0AafiaYkBjzwunPw/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

### 原因

**在使用left jion时，on和where条件的区别如下：**

1、 on条件是在生成临时表时使用的条件，它不管on中的条件是否为真，都会返回左边表中的记录。

2、where条件是在临时表生成好后，再对临时表进行过滤的条件。这时已经没有left join的含义（必须返回左边表的记录）了，条件不为真的就全部过滤掉。