# Hive SQL(报表开发相关)

基本理解：

hadoop生态圈里面的一个组件。说白了就是解析器，解析Hive SQL并转换成相应的MR代码，达到与写代码相同的效果，降低Hadoop分析的使用门槛(倒是有操作系统的味道在里面了)。

一般会用HBase整合Hive使用，然而直接使用的话也很麻烦，所以公司里面进行报表开发的时候一般会选择使用图形化界面来进行，一来效果是一样的，二来也不会有人因为操作不熟悉乱敲指令导致出什么幺蛾子。



工作场景：Hive整合DBeaver使用。

DBeaver等数据库图形化工具可以直接连接上Hive，直接将Hive当做传统数据库使用进行一些开发非常方便，总而言之只要会写SQL就行了，然后注意一些比如HiveSQL过滤条件中不支持子查询这种细节就行了。



## 1.Hive执行计划 explain

在SQL开头声明 explain；可以查看Hive的执行计划

![1618197277454](HelloHive.assets/1618197277454.png)

因为测试环境比较简陋，而且估计因为账号权限的原因，只要SQL中一加配置就无法运行，只有执行计划可以正常看。

## 2.Oracle Sql优化为Hive Sql 

1--66--22.45S--原Sql

select a.id from table_1 a where not exists (select 'X'
          from table_2 b where b.contno = a.otherno and b.moneytype in ('YFLQ', 'EFLQ'))

--2--62--21.33S--用left join 优化

select a.id from t1 a left join table_2 b on  on b.contno = a.otherno and b.moneytype in ('YFLQ', 'EFLQ')
where b.contno is null

有个好玩的地方，如果原SQL是Exists的时候，使用这种方法会有一些差别：

使用Exists判断的结果在Hive中运行完了会有一个去重的效果，而使用Left join 优化以后得到的结果不会去重(如果需要的话，手动加一个去重感觉反而不划算，不如不优化)，这个差别让我感觉很有意思，不知道在Oracle中会不会有这种差异。(对于Exists，Hive有专门的特殊用法，所以这个问题很难发生)

## 3.Oracle和Hive的一些函数差异(Hive为主)

### 3.1.字符串操作函数

substr(a,1,10);-->Oracle中用法一样，起始位置注意一下，Hive中是从一开始的

regexp_replace(a,'^[b]');-->注意正则表达式用的是JAVA的正则

instr(a,'b');-->Oracle中带四个参数：instr(原字段，要匹配的字符串，起始位置，第几次匹配)，很方便

concat(a,b,c......);-->在Oracle中只能连接两个字符串(concat(a,b))，而用:`||`可以连接多个字符串达到与Hive相同的效果(a||b||c...)。

concat_ws(',',collect_set(t1.name));列转行函数，将所有的name通过逗号连接在一起

目前用到的就是这几个，在Hive中，substr需要三个参数，输出一个截取后的字符串

#### 截取函数substr

substr(字段，起始截取位置，截取长度)

substr(regexp_replace(l.standbyflag6,'^(.*?\\//.*?/.*?/.*?/)',''),1,10)

#### 替换函数regexp_replace

regexp_replace(原字符串,'正则表达式','替换后显示的字符串')带三参返回一个经过替换操作的字符串

#### 匹配位置函数instr

instr(原字段,'要匹配的字符串')

#### 列转行函数：

concat_ws(',',collect_set(t1.name))

需要两个参数，第一个是分隔符，默认应该是空格，没有试过

collect_set()，还有collect_list(),将一列转换为一个列表，但是set有去重的效果

例：篇幅原因，看前五条就行了

原本数据的样子：

![1619060368811](HelloHive.assets/1619060368811.png)

这是用List：结果成了一行，有重复

![1619060435689](HelloHive.assets/1619060435689.png)

这是用Set：结果也是一行，但是没有重复

![1619060498583](HelloHive.assets/1619060498583.png)

出于好奇，我又跑了一下collect_set(),不多逼逼，结果如下：
![1619060653147](HelloHive.assets/1619060653147.png)



### 3.2 时间函数

date();-->Oracle 中用法为to_date(日期字符串，格式)

但是Hive中用date转换的结果不能直接用于计算(日期的加减，取月份之类的)，我找了一个麻烦一点的代替，可以用于运算的函数方法：

from_unixtime(unix_timestamp(日期字符串，字段格式)，目标格式)

后面有碰到一种：

date_format(to_date(),'yyyy-MM-dd')

#### 3.2.1 测试1

--测试to_date()函数无法使用unix_timestamp()的问题
select to_date('${P1}') from ms_ods_p06.lpedoritem a limit 1 --P10：2020-3-1

select to_date(unix_timestamp('${P1}','yyyy-MM-dd')) from ms_ods_p06.lpedoritem a limit 1  -- 报错了

select unix_timestamp('${P1}','yyyy-MM-dd') from ms_ods_p06.lpedoritem a limit 1  -- 1582992000

select from_unixtime(unix_timestamp('${P1}','yyyy-MM-dd'),'yyyy/MM/dd') from ms_ods_p06.lpedoritem a limit 1
--结论: unix_timestamp() 拿到的是 Bigint 格式的时间, to_date 和 date_format 函数无法直接使用

### 3.3 NVL

看到网上有人四月份发帖，说Hive不支持NVL，遂写个demo验证下：
![1619159516699](HelloHive.assets/1619159516699.png)

结果为一列。

![1619159748297](HelloHive.assets/1619159748297.png)

说明生效了，没有NVL不能用的说法。

然后是另一个帖子说的，NVL函数前后结果类型必须一样，为此我找了一个类型为string的函数，结果如下：

![1619160630520](HelloHive.assets/1619160630520.png)

然而找了好久没在集群的库中找到原本为date类型的函数，无法验证，甚是遗憾。

## 4.Exist和not Exist

`Exist`可以用`left semi join` 代替（执行计划上看一模一样）

`not exist` 可以用` t1 left join t2 on t1.id=t2.id where t2.id is null `代替,记住一定要用`where`不然结果不准确(有待深究)

### 4.1 Left Semi Join

Left semi join 可以完全代替Exists，但是同时也有很多限制
1.Left Semi Join的所有过滤条件必须都加在on连接条件之后，否则无法运行

2.Left Semi Join只支持简单的过滤，如果涉及到非等值连接还有or连接的判断的话就无法使用，建议拆开用 Union all

## 5.函数嵌套的修改(遇到一个复杂的记一个)

```sql
nvl(nvl((select codename
               from ldcode
              where codetype = 'paymode'
                and code = (select c.paymode from ljaget c where c.otherno = d.edorconfno and rownum = 1 and c.enteraccdate is not null)
                and rownum = 1
         ),
                (select codename
               from ldcode
              where codetype = 'paymode'
                and code = (select c.paymode from ljaget c where c.otherno = d.edorconfno and rownum = 1 and c.enteraccdate is null)
                and rownum = 1))
     ,
             (select codename
                from ldcode
               where codetype = 'edorgetpayform'
                 and code = d.payform
                 and rownum = 1))
                 
explain
select               
nvl(nvl(ldcode1.code_name,ldcode2.code_name),ldcode3.code_name)
from ms_ods_p05.lpedorapp d
left join ms_ods_p05.ljaget c1 on c1.otherno = d.edorconfno and c1.enteraccdate is not null
left join ms_ods_p05.ljaget c2 on c2.otherno = d.edorconfno and c2.enteraccdate is null
left join ms_ods_p06.ldcode ldcode1 on ldcode1.code=c1.paymode and ldcode1.code_type = 'paymode' 
left join ms_ods_p06.ldcode ldcode2 on ldcode2.code=c2.paymode and ldcode2.code_type = 'paymode' 
left join ms_ods_p06.ldcode ldcode3 on ldcode3.code = d.payform and ldcode3.code_type = 'edorgetpayform'
```

## 6.比较头疼的一些暂时没解决的逻辑问题(持续更新)

### 6.1 过滤条件Exists中的逻辑问题

Hive 子查询不支持非等值连接，t1.id < t2.id 这种连接条件无法在子查询中使用

and过滤条件中用exists判断两个子查询or的结果，问题是第二个子查询中带了非等值连接导致问题解决不了：

```sql
and exists
    (select 1
     from lmriskapp
     where riskcode = a.riskcode
     and riskperiod = 'L')
     and ((appflag = '1' and exists
         (select 1
          from lccontstate
          where polno = a.polno
          and statetype = 'Available'
          and state = '0'
          and enddate is null)) or
          (exists
           (select 1
            from lccontstate
            where polno = a.polno
            and statetype = 'Available'
            and state = '0'
            and startdate >= to_date('${P1}', 'YYYY-mm-dd')
            and enddate <= add_months(cvalidate, 24))))
```

**难点：**

1.子查询判断中用or连接，这个可以用`union all`解决

2.上半部分分开以后没问题，下半部分单独运行都会报错

```sql
(exists
     (select 1
      from lccontstate
      where polno = a.polno
      and statetype = 'Available'
      and state = '0'
      and startdate >= to_date('${P1}', 'YYYY-mm-dd')
      and enddate <= add_months(cvalidate, 24)))
```

`and enddate <= add_months(cvalidate, 24)` 这个条件是非等值连接，hive 本身不支持非等值连接

最终解决办法：

经过分析得知，`enddate <= add_months(cvalidate, 24)` 过滤的是到期两年后，也就是过期的订单。

然后我直接

```sql
with temp as (select lc.polno from lccontstate lc,主表 a where enddate > add_months(a.cvalidate, 24))
先拿到所有没过期的订单的订单号
```

然后通过

```sql
and polno=temp.polno
```

留下的都是没过期的，实现了要实现的功能。

### 6.2 麻烦的函数嵌套问题

原：

```sql
nvl((select  decode(nvl(LTRIM(substr(e.impartparammodle, 1,INSTR(e.impartparammodle, '/', 1, 1) - 1),'0123456789'),'1'),'1',substr(e.impartparammodle, 1,
                               INSTR(e.impartparammodle, '/', 1, 1) - 1)||'元','0')
                   from LCCustomerImpart e
                  where e.CustomerNoType = '0'
                    and e.CustomerNo =a.appntno
                    and e.ContNo = a.contno
                    and e.impartver ='102'
                    AND E.IMPARTCODE ='20B'
                    and rownum = 1),(select  decode(nvl(LTRIM(substr(e.impartparammodle, 1,INSTR(e.impartparammodle, '/', 1, 1) - 1),'0123456789'),'1'),'1',substr(e.impartparammodle, 1,
                INSTR(e.impartparammodle, '/', 1, 1) - 1)||'万','0')
                   from LCCustomerImpart e
                  where e.CustomerNoType = '0'
                    and e.CustomerNo = a.appntno
                    and e.ContNo =a.contno
                    and e.impartver = 'A06'
                    AND E.IMPARTCODE = 'A0534'
                    and rownum = 1)) 年收入,
```

修改后：

```Sql
nvl( t1.num1 , t2.num2 ) `年收入`,

left join (select  case nvl(regexp_replace(substr(e.impartparammodle, 1, instr(e.impartparammodle,'/') ),'^\\d*',''),'1') when '1' then concat(substr(e.impartparammodle, 1,
           INSTR(e.impartparammodle, '/') - 1),'元') else '0' end  as num1,CustomerNo,ContNo
				   from ms_ods_p06.LCCustomerImpart e
				  where e.CustomerNoType = '0'
				 	and e.impartver ='102'
					AND E.IMPARTCODE ='20B' group by e.impartparammodle,CustomerNo,ContNo) t1 on t1.CustomerNo =a.appntno and t1.ContNo = a.contno
left join (select  case nvl(regexp_replace(substr(e.impartparammodle, 1,INSTR(e.impartparammodle, '/') ),'^\\d*',''),'1') when '1' then concat(substr(e.impartparammodle, 1,
                INSTR(e.impartparammodle, '/') ),'万') else '0' end as num2,CustomerNo,ContNo
                   from ms_ods_p06.LCCustomerImpart e
                  where e.CustomerNoType = '0'
                    and e.impartver = 'A06'
                    AND E.IMPARTCODE = 'A0534' group by e.impartparammodle，CustomerNo,ContNo) t2 on t2.CustomerNo =a.appntno and t2.ContNo = a.contno


```

## 7. 92写法和99写法带来的差异

```sql
--Hive 中 ,连接和 inner join 连接到底有什么区别
--92
explain
select a.contno,b.contno from ms_ods_p06.lpedoritem a,ms_ods_p08.lccont b  where b.contno = a.contno
--99
explain
select a.contno,b.contno from ms_ods_p06.lpedoritem a inner join ms_ods_p08.lccont b on b.contno = a.contno
```

分别看执行计划，前面都一模一样，在第二个stage才出现了差异：

92：

![1619159175816](HelloHive.assets/1619159175816.png)

而99：

![1619159110547](HelloHive.assets/1619159110547.png)

很清楚地看到92语法在select操作前多了一步过滤操作，导致后面进行的操作影响的行数比较少。但是由于过滤操作本身带来的性能差异就难说了，数据量不同的情况下性能优劣真不好说。

## 8.去重留一

Hive 版：

select edorstate from ( select edorstate,ROW_NUMBER() over(partition by edorstate order by edorno desc) as num from ms_ods_p06.lpedoritem ) temp where temp.num=1;

Row_number()函数可以跟开窗，好用

![1619511376529](HelloHive.assets/1619511376529.png)

漂亮的去重留一。

```sql
-- mysql语法，分组留最小值
select * from (select * from student group by Ssex,SId order by SId asc ) t group by t.Ssex;
-- mysql语法，分组留最大值
select * from (select * from student group by Ssex,SId order by SId desc ) t group by t.Ssex;
```

## 9.Left Join = Left Outer Join

这两个是等价的，left join 只是简称，实际使用起来效果没差别的。只是平时没见过用left outer join的，突然看到有点懵。

## 10.Hive 动态分区

开启Hive动态分区的指令：

```shell
--第一条指令开启动态分区
set hive.exec.dynamici.partition=true;
--第二条指令关闭严格模式，允许所有分区都为动态，否则无法正常使用(系统会提示你关闭严格模式)
set hive.exec.dynamic.partition.mode=nonstrict;
--第三条指令确定每个mapper或者reducer可以创建的最大动态分区的个数
set hive.exec.max.dynamic.partitions.pernode=100
--第四条指令指定一条动态分区创建语句可以创建的最大动态分区的个数
set hive.exec.max.dynamic.partitions=1000;
--第五条指令指定全局可以创建的最大文件的个数
set hive.exec.max.created.files=100000;
```

为什么要开启动态分区：静态分区就是用户手动指定分区字段的值，这样在SQL运行时就可以直接拿到改分区的字段，但是如果需要拿到很多分区的结果的时候，这样就很麻烦，所以需要开启动态分区，这样不需要用户指定分区的值，系统自动根据分区的值将结果查询出来。

## 11.union all 和 or

因为union自带去重效果所以先不考虑

关于这两者哪个执行效率高，网上说什么的都有，所以我回来就写几个DEMO试一下。先说在MySQL中测试的结果：实际执行的时候or的效率远高于union all，查看执行计划发现，union all查询了两次，所以几乎耗时是or的两倍。然后此时又顺手测了一下group by 和 over(partition by)，结果说明开窗是真的香。

Hive：

```SQL
--1
select l.branchmanager ,avg(l.agentgroup) from ms_ods_cms.labranchgroup l group by l.branchmanager having l.branchmanager = '8611001188' or l.branchmanager = '8611001411';
--2
select l.branchmanager ,avg(l.agentgroup) over(partition by l.branchmanager) from ms_ods_cms.labranchgroup l where l.branchmanager = '8611001188' or l.branchmanager = '8611001411';
--3
select l.branchmanager ,avg(l.agentgroup) from ms_ods_cms.labranchgroup l group by l.branchmanager having l.branchmanager = '8611001188' 
union all 
select l.branchmanager ,avg(l.agentgroup) from ms_ods_cms.labranchgroup l group by l.branchmanager having l.branchmanager = '8611001411';
--4
select l.branchmanager ,avg(l.agentgroup) over(partition by l.branchmanager) from ms_ods_cms.labranchgroup l where l.branchmanager = '8611001188' 
union all 
select l.branchmanager ,avg(l.agentgroup) over(partition by l.branchmanager) from ms_ods_cms.labranchgroup l where l.branchmanager = '8611001411';
```

因为DBeaver要开启设置才能同时运行多条用`;`分割的SQL，但是会看不到后面的查询的耗时，所以这里跑了四次，然后尽量多跑几次保证结果正确。

(耗时单位：s)

结果一：20.146,20.306,20.160

结果二：19.274,21.250,20.390

结果三：53.137,52.931,54.257

结果四：54.968,52.466,54.254

结果证明 对本次查询来说 or 的效率要优于 union all 。但开窗的优势就看不出来了？怎么放大差距？遂，又有了四次实验(不过滤了)：

```SQL
--1
select l.branchmanager ,avg(l.agentgroup) from ms_ods_cms.labranchgroup l group by l.branchmanager;
--2
select l.branchmanager ,avg(l.agentgroup) over(partition by l.branchmanager) from ms_ods_cms.labranchgroup l;
--3
select l.branchmanager ,avg(l.agentgroup) from ms_ods_cms.labranchgroup l group by l.branchmanager
union all 
select l.branchmanager ,avg(l.agentgroup) from ms_ods_cms.labranchgroup l group by l.branchmanager;
--4
select l.branchmanager ,avg(l.agentgroup) over(partition by l.branchmanager) from ms_ods_cms.labranchgroup l 
union all 
select l.branchmanager ,avg(l.agentgroup) over(partition by l.branchmanager) from ms_ods_cms.labranchgroup l;
```

(耗时单位：s)

结果一：19.524，20.981，20.360

结果二：20.560，22.206，20.255

结果三：57.798，54.720，53.858

结果四：55.136，56.645，55.230

结论：我测了个寂寞，数据量太小，压根体现不出差距。以后有机会找一张10G以上的表试一试吧。