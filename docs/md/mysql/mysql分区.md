## 分区的优势

```text
分区的本质就是将数据分成多个文件进行存储所以具有如下有点
a.存储更多的数据
    多文件存储，肯定比单文件的上限高
b.优化查询
    将数据进行分区可以更好的将热点数据分散，从而达到优化查询的目的
c.并行处理
    多分区多文件可以更好的并行处理
d.快速删除数据
    删除老数据直接删除分区即可
e.更大的数据吞吐量
    并行处理当然可以带来更大的并发量
```

## 分区基础知识

```mysql
    #查看分区
show variables like '%partition%';
#查看插件
show plugins;
#查询mysql版本
select version();

```

```text
注意：
在MySQL所有的分区类型中，进行分区的数据表可以不存在主键或者唯一键
如果存在主键或者是唯一键，则不能使用主键或者是唯一键之外的其他字段进行分区操作
```

### range分区

range分区只对整型字段生效，如果不是整型需要进行转化

```mysql
#建表时创建分区
create table tab_01
(
    id       int not null,
    name     varchar(500),
    group_id int not null
)
    PARTITION BY RANGE (id)(
        PARTITION part0 VALUES LESS THAN (100),
        PARTITION part1 VALUES LESS THAN (200),
        PARTITION part2 VALUES LESS THAN (300),
        PARTITION part3 VALUES LESS THAN (400),
        PARTITION part4 VALUES LESS THAN (500)
        );

#查看分区表的数据分布
select partition_name        part,
       partition_expression  expr,
       partition_description part_desc,
       table_rows
from information_schema.partitions
where table_schema = schema()
  AND table_name = 'tab_01';

#我们定义了table_o1的最大分区是500所以当我们的id大于500时数据就会报错
#添加分区
alter table tab_01
    add PARTITION (
        PARTITION part5 values less than maxvalue
        );

#删除分区,删除分区后数据也会全部删除
alter table tab_01
    drop PARTITION part2;

#重新定义分区
alter table tab_01
    REORGANIZE PARTITION part4 into (
        PARTITION part4 VALUES LESS THAN (450),
        PARTITION part5 VALUES LESS THAN (500)
        );

#合并分区
alter table tab_01
    REORGANIZE PARTITION part4,part5 into (
        PARTITION part4 VALUES LESS THAN (500)
        );


```

### list分区

list是一个逗号分隔的整数列表，不必按照某种顺序进行排列。在MySQL5.5版本之后，支持对非整数类型进行LIST分区

```mysql
#建表时分区
#5_5版本之前
create table tab_01
(
    id       int not null,
    name     varchar(500),
    group_id int not null
)
    PARTITION BY LIST (group_id)(
        PARTITION part0 VALUES in (1,2,5),
        PARTITION part0 VALUES in (10,20,50));
#5_5版本之后
create table tab_01
(
    id       int not null,
    name     varchar(500),
    group_id int not null
)
    PARTITION BY LIST (group_id)(
        PARTITION part0 VALUES in ('a','b','c'),
        PARTITION part1 VALUES in ('d','e','f'));

#添加分区
alter table tab_01
    add PARTITION (
        PARTITION part5 values in ('a','b')
        );

#删除分区,删除分区后数据也会全部删除
alter table tab_01
    drop PARTITION part0;

#重新定义分区
alter table tab_01
    REORGANIZE PARTITION part1 into (
        PARTITION part1 VALUES in ('d','e'),
        PARTITION part2 VALUES in ('f')
        );

#合并分区
alter table tab_01
    REORGANIZE PARTITION part0,part1 into (
        PARTITION part1 VALUES in ('a','b','c','d','e')
        );
```

### hash分区

hash分区的作用是可以分散热点数据，但当表中的数据进行变更时，都需要使用hash算法计算一次，所以不推荐使用复杂的hash
算法,也不推荐对数据表中的多个字段进行hash分区。主要分为两种

常规HASH分区 主要的原理就是对字段进行hash之后再对分区总数进行取模

```mysql
create table tab_01
(
    id       int         not null,
    t_name   varchar(40) not null default '',
    group_id int         not null
) PARTITION BY HASH (group_id)
        PARTITIONS 4;
```
 线性HASH分区 根据一个线性对分区号计算方式进行分区计算，好处就是分区的建立、添加、删除、合并拆分更快，在对TB级别对数据量处理时
好处明显，但是坏处就是它没有常规对hash分区数据分散对更均匀

```text
step1：V = power(2, ceil(log(2, num)))
step2：N = value & (V-1)
step3：if N>=num: N=N & (ceil(V/2) - 1)
```

```mysql
#线性分区新增
create table tab_01
(
    id       int         not null,
    t_name   varchar(40) not null default '',
    group_id int         not null
) PARTITION BY LINEAR HASH (group_id)
    PARTITIONS 4;

#线性分区添加
#指的是新增分区，而不是改变总数
alter table tab_01
    add PARTITION PARTITIONS 11;

#合并分区
alter table tab_01
    coalesce partition 9;

```

### key分区

key分区在某种程度上与hash分区类似，只不过hash分区可以使用用户自定义的函数和表达式，而key分区不能。另外，hash分区只能对整数类型的列进行分区，
而key分区能够支持对除了blob和text数据类型以外的其他数据类型的列进行分区。

```mysql
#如果没有指定就会用primary key 做分区
#如果没有则会自动选择非空并且唯一对一列进行key分区
#如果都没有则需要指定
create table tab_01
(
    id       int not null primary key,
    t_name   varchar(40),
    group_id int not null
) PARTITION BY KEY (group_id) PARTITIONS 8;

#其他操作和HASH分区相同
```

### columns分区

COLUMNS分区MySQL 5.5 版本引入的新的分区类型，能够解决MySQL之前的版本中RANGE分区和LIST分区只支持整数分区的问题。

COLUMNS分区可以分为RANGE COLUMNS分区和LIST COLUMNS分区。 RANGE COLUMNS分区和LIST
COLUMNS分区都支持整数类型，日期时间类型和字符串类型

```mysql
#建表时分区
# RANGE COLUMNS
create table tab_01
(
    id         int not null,
    name       varchar(500),
    group_id   int not null,
    group_code int not null
)
    PARTITION BY RANGE COLUMNS (group_id,group_code)(
        PARTITION part0 VALUES LESS THAN (1,10),
        PARTITION part1 VALUES LESS THAN (10,20),
        PARTITION part2 VALUES LESS THAN (20, MAXVALUE),
        PARTITION part3 VALUES LESS THAN (MAXVALUE, MAXVALUE)
        );

# LIST COLUMNS
create table tab_01
(
    id         int not null,
    name       varchar(500),
    group_id   int not null,
    group_code int not null
)
    PARTITION BY RANGE COLUMNS (group_id,group_code)(
        PARTITION part0 VALUES LESS THAN ((1, 10), (1, 20), (1, 30)),
        PARTITION part0 VALUES LESS THAN ((2, 10), (2, 20), (2, 30))
        );

```

### 子分区

可以对数据表中的RANGE分区和LIST分区再次进行子分区，形成复合分区。其中，子分区可以使用HASH分区，也可以使用KEY分区

```mysql
create table tab_01
(
    id       int not null,
    t_name   varchar(30),
    group_id int
) PARTITION BY RANGE (group_id)
        SUBPARTITION BY HASH (group_id)
        SUBPARTITIONS 4
        (
        PARTITION part0 VALUES LESS THAN (10),
        PARTITION part1 VALUES LESS THAN (MAXVALUE)
        );
```
    

