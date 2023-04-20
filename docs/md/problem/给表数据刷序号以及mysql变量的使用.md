## 问题
```text
为了解决日渐庞大的订单表查询慢的问题，所以引入es。
于此同时就带了深分页的问题，解决深分页需要给表加一行序号
```

## sql
```sql
alter table order_item add serialNo bigint null comment '序列号';
update order_item t1
    join (
    select id, @rowNum := @rowNum + 1 'rank'
    from rder_item t3,(select @rowNum := 0)r order by createTime asc
    ) t2
on t1.id = t2.id
set serialNo = t2.`rank`;
```

## 变量简介
```sql
变量分为 系统全局变量 系统会话变量 会话用户变量 用户局部变量
系统全局变量
    SHOW GLOBAL VARIABLES;
    SHOW GLOBAL VARIABLES LIKE '%标识符%';
    SELECT @@global.变量名;
    SET GLOBAL 变量名=变量值;
    SELECT @@global.autocommit;
系统会话变量
    SHOW SESSION VARIABLES;
    SHOW SESSION VARIABLES LIKE '%标识符%';
    SELECT @@session.变量名;
    SET SESSION 变量名=变量值;
    SET @@session.变量名=变量值;
会话用户变量
    SET @用户变量 = 值;
    SET @用户变量 := 值;
    SELECT @用户变量 := 表达式 [FROM 等子句];
    SELECT 表达式 INTO @用户变量  [FROM 等子句];
    SELECT @用户变量
用户会话变量
BEGIN
	#声明局部变量
	DECLARE 变量名1 变量数据类型 [DEFAULT 变量默认值]; # 如果没有DEFAULT子句，初始值为NULL
	DECLARE 变量名2,变量名3,... 变量数据类型 [DEFAULT 变量默认值];

	#为局部变量赋值
    SET 变量名1 = 值;
    SELECT 值 INTO 变量名2 [FROM 子句];

    #查看局部变量的值
    SELECT 变量1,变量2,变量3;
    
    #定义变量
    DECLARE 变量名 类型 [default 值];  # 如果没有DEFAULT子句，初始值为NULL
    
    # 赋值
    SET 变量名=值;
    SET 变量名:=值;
    
    # 使用
    SELECT 局部变量名;
END
```
[变量的基本使用](https://blog.csdn.net/qq_41684621/article/details/125281323)