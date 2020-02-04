## 1.创建/修改数据库表

创建数据库表的基本语法格式：

```sql
create table <table_name>(
		C1 D1,
		C2 D2,
		...,
		An Dn,
		<完整性约束1>，
		...
		<完整性约束k>,
);
```

Ci是列名，Di是该列的域，Di指定了Ci的类型和可选值的约束。

完整性约束包括：

1. primary key(Aj1, Aj2, ...):主码，唯一主键。关系集合中， 没有任何两行在所有的主键取值上完全相同。

2. foreign key(Aj1, Aj2,...)

3. not null

   

修改数据库表结构:

```sql
alter table <table_name> add A,D;
alter table <tn> drop A;
```

## 2.查询

### 2.1 单关系查询

```sql
select <colname> from <tblname> where ...;
```

### 2.2 多关系查询

```sql
select <tb1>.<col1>, <tb2>.<col2>, <tbn>.<colm>
from <tb1>,<tb2>,...<tbn>
where ...;
```



### 2.3 自然连接

```sql
select col1,col2,..,coln
from tb1 natural join tb2;
```

自然连接将两个表中col相同的列作为join条件，将col中相同值的记录合并。

更普遍的:

```sql
select col1, col2, ..., coln
from (tbl1 natural join tbl2) natural join tbl3; 
```

需要注意的是：natural join要求tbl1, tbl2,tal3表中都有相同的column name。

### 2.4 字符串运算

SQL提供了在字符串上的多种运算函数：串联(||)、提取子串、计算字符串长度、大小写转换(upper() & lower())、去除字符串后面的空格(trim(s))、模糊匹配(like).

### 2.5 where子句谓语

```sql
-- between 谓语
select * from staff where salary between 5000 and 10000; -- salary >= 5000 and salary <= 10000

-- not between 谓语
select * from staff where salary not between 5000 and 10000; -- salary < 5000 or salary > 10000;

-- 分量查询
select * from staff where (salary, age) >= (10000, 30); -- salary >= 10000 and age >= 30

```

### 2.6 集合运算

SQL的union, intersect和except对应数学集合论上的并集，交集，差集概念。

```sql
-- 找出班上学习math和physics的学生作为并集。
(select student_name from section where course='math') 
union 
(select student_name from section where course='physic');

-- 找出班上既学习math又学习physics的学生
(select student_name from section where course='math')
intersect
(select student_name from section where course='physics');

-- 找出班上学习math并且不学习physics的学生
(select student_name from section where course='math')
except
(select student_name from section where course='physics');
```

注意：以上运算符和数学集合论意义上的交、并、差集概念，意味着最后的输出结果也是去重的。如果不想去重，可以使用`union all`  ,`intersect all`, `except all`。

### 2.7 聚集函数

SQL的聚集函数是对一组值的集合进行处理产生输出的函数。SQL提供了常用的五种基本聚集函数：

- avg
- min
- max
- sum
- count

sum和avg必须作用于数字集上。其他函数可以作用于非数据集上。

### 2.8 分组聚集

有时候我们希望聚集函数不是作用在某一个单一的数据集上，而是希望作用到一组数据集上，给每一组各自数据聚集函数的结果。`group by`可以实现这种功能。

```
-- 按照部门对公司员工分组，计算每个部门的平均工资
select department, avg(salary) as avg_salary
from staff
group by department；
```

需要注意的是： 任何一个属性，如果没有出现在group by子句中，在select子句中就只能出现在聚集函数中。下面这个例子是错的：

```sql
-- 因为ID没有出现在group by中，所以不能用select语句中直接选中，但是salary是可以的，因为salary出现在聚集函数avg中。
select department, ID, avg(salary)
from staff
group by department;
```

### 2.9 having子句

where子句的谓语作用到的是每一个单行的记录，有时候我们也需要对一组记录使用谓语过滤，这时候可以使用`having`子句

```sql
-- 只查找平均工资高于10000元的部门
select department, avg(salary) as avg_salary
from staff
group by department
having avg(salary) > 10000;
```

注意到：`平均工资高于10000元`这个条件不是针对单一记录的过滤。它是在一组元组产生之后才起作用的。

`having`子句和`group by`一样。如果一个属性没有在group by中出现，就不能用select选中，但是可以出现在聚集函数中。

> 岔开话题，思考一下如何看懂别人写的一个复杂的SQL语句：
>
> 一个复杂SQL语句的含义可以通过下面的操作序列来分析：
>
> 1. 先抛开所有条件，从from语句计算出一个关系，这个关系就是两个表的笛卡尔集。
> 2. 如果有where子句，将where子句的谓语应用到第1步计算出的关系上
> 3. 如果出现group by子句。将第2步计算出的关系按照group by的条件分为多个元组。如果没有group by，第2步计算出的关系作为一个元组。
> 4. 如果有having子句，将having子句的谓语作用到每一个分组上。
> 5. 看select子句从第4步计算的关系上查询了那些属性，重新组装成一个新的关系集合
> 6. 如果有order by,  在第5步的关系集上应用order by子句
>
> 总体流程：from  -> where  -> group by -> having -> select -> order by -> offset -> limit

### 2.10 嵌套子查询

嵌套子查询包含两种形式： where子句的嵌套子查询和from子句的嵌套子查询。

where子句的嵌套子查询一般用于测试一个集合的成员资格、集合比较和集合基数检查。

from子句的嵌套子查询一般是基于一个查询的结果进行新一轮的查询（对查询结果的查询）。

#### 1. where子句的嵌套子查询

##### 集合成员资格查询

使用`in` 或`not in `可以用于查询一个/一组记录是否在一个集合中。

```sql
-- 从students表中找出学习了math课程的学生的姓名。
-- 注意： course表只记录了student_id，而没有studet_name。
select student_name
from students
where student_id in(
		select student_id 
		from course
		where course_name = 'math'
);

-- 常用的枚举类型也可以作为一个集合
select course_name
from course
where course_name not in ('math', 'computer science');
```

##### 集合比较

`some`  `all` 可以用于集合比较。

```sql
-- 找出满足这样条件的所有员工的姓名：他们的工资至少要大于某一个研发部门员工的工资

select T.staff_name
from staff as T, staff as S
where T.salary > S.Salary and S.department = 'RD';

-- 或者下面的方法：
select staff_name
from staff
where salary > some(
	select salary 
	from staff 
	where department='RD'
);
```

`>some`语法适合表达“至少比某一个要大”的含义。与之相反的：`<some`表示“至少比某个要小”

`>=some`表示"至少不小于某一个"，`<=some`表示"至少不大于某一个"

`<>some`表示"至少不等于某一个"，`=some`和`in`是等效的。

`>all`表示："比任何一个都大", `<all`表示“比任何一个都小”

```sql
-- 找出平均工资最高的部门
select department
from staff
group by department
having avg(salary) >= all(
	select avg(salary)
	from staff
	group by department
);
```

##### 空关系测试

`exists`和`not exists`可以用于测试一个元组是否在另一个元组中。

##### 重复元组存在性测试

`unique`

#### 2. from子句嵌套子查询

with语句用于定义临时关系，这个临时关系只对包含with语句的查询有效。

```sql
-- 查询具有最大工资的员工
with max_salary(value) as(
	select max(salary) from staff
) select staff_name, salary from teacher, max_salary
where teacher.salary = max_salary.value;
```

该怎样理解上面with语句呢?可以这样分析：

`with  temp_tbl(value)` 定义了一个临时的表（实际不是）。后面的`select max(salary) from staff`计算的结果给temp_tbl赋值。然后在下一个`select`语句中像使用一个普通的数据库表一样使用它。`select col1, col2 from tbl1, temp_tbl where tbl1.col1 = temp_tbl.value`. 如果你认为`temp_tbl`是一个临时表的话，value就是它的列，这一列的类型是as语句后面选出的那一列的类型。它的值是as后面select选中的值。

带着这个思路，看下面一个例子：

```sql
with dept_total(department, value) as(
	select department, sum(salary)
	from staff
	group by department
),
dept_total_avg(value) as(
	select avg(salary)
	from staff
)
select department
from dept_total, dept_total_avg
where dept_total.value >= dept_total_avg.value;
```

上面这个SQL语句的含义是什么样的？

`dept_total`是一个临时表，这个表有两列：`department`和`value`。因为`value`是经过`group by department`sum计算得来的，所以`value`是每个部门的工资总额. 

`dept_total_avg`是另一个临时表，它只有一列`value`，`value`的值是`avg(salary)`计算得来的。最后的`where`语句是找出“部门工资总额>=部门平均工资那些记录”，最后select出`department`.

**特别需要说明的是**:with语句并没有真的产生临时表（临时表有其他的定义方法）。with语句返回的仍然是一个记录或一个元组(一组记录). 基于with语句做后续的查询本质上也是对with过程产生的记录或者元组的查询。但即使从数据库理论层面上也很难分清楚一组记录和一个临时表有什么区别。这里姑且不做详细讨论。　