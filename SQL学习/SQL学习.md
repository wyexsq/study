# SQL学习

```postgresql
--1

create  table  Product
(product_id char(4) not null ,
 product_name varchar(100) not null ,
 product_type varchar(32) not null ,
 sale_price integer ,
 purchase_price integer,
 regist_data date,
 primary key (product_id));


alter  table Product add column product_name_pinyin varchar(100);


-- 删除列使用的语句
alter table Product drop  column product_name_pinyin ;


-- Oracle 上面两个操作都不用写column


-- 插入数据
begin transaction ;--开始插入的指令语句
insert  into  Product values ('001','T恤衫','衣服',
                              1000,500,'2009-09-20');
insert  into  Product values ('002','打孔器','办公用品',
                              500,320,'2009-09-11');
insert  into  Product values ('003','运动T恤','衣服',
                              4000,2800,'2009-09-20');
insert  into  Product values ('004','菜刀','厨房用品',
                              3000,2800,'2009-09-20');
insert  into  Product values ('005','高压锅','厨房用品',
                              6800,5000,'2009-01-15');
insert  into  Product values ('006','叉子','厨房用品',
                              500,null,'2009-09-20');
insert  into  Product values ('007','擦菜板','厨房用品',
                              880,790,'2009-09-28');
insert  into  Product values ('008','圆珠笔','办公用品',
                              100,null,'2009-09-11');
commit ;

--注意在MySQL中运行时，需要把①中的BEGIN TRANSACTION;改写成 START TRANSACTION;

--表名的变更ALTER TABLE Poduct RENAME TO Product; RENAME TABLE Poduct to Product;

--练习：1.1编写一条 CREATE TABLE 语句，用来创建一个包含表 1-A 中所列各项
--的表 Addressbook（地址簿），并为 regist_no（注册编号）列设置
--主键约束。
create table Addressbook
(
    regist_no integer not null ,
    name varchar(128) not null ,
    address varchar(256) not null,
    tal_no char(10) ,
    mail_address char(10),
    primary key (regist_no));

--全面出现问题，通过执行下面这个语句清理了缓存，然后重新执行练习的创建任务
ROLLBACK;
/*
 在一个事务中，只要有任何一条 SQL 出错（包括语法错误或违反约束），整个事务就会进入 aborted 状态，之后的所有命令（即使是完全正确的 CREATE TABLE）都会被拒绝，并要求先执行 ROLLBACK。
而且错误信息可能会被某些客户端吞掉或延迟显示，让你只看到“事务被终止”的报错，而看不到真正的出错原因。
   我刚刚应该是不小心执行过应该错误命令
 */


--练习1.2：假设在创建练习 1.1 中的 Addressbook 表时忘记添加如下一列 postal_code（邮政编码）了，请把此列添加到 Addressbook 表中。
alter table Addressbook add column postal_code char(8) not null default '00000000';
--怪不得我直接输入not null 不可以，原来，我订长了，本来就不是空，既然如此，我就要用占位符号，不然本来就是空格，所以刚刚理解逻辑有问题

--1.3：删表
DROP TABLE Addressbook;


--2
--列的查询
select Product.product_id ,Product.product_name,Product.purchase_price
from Product;


select  *
from Product;

--给列设置别名 as关键字
select  Product.product_id as id,
        Product.product_name as name,
        Product.purchase_price as price
    from Product;
--注意，如果使用中文名，记得使用双引号

--常数查询
select  '商品' as string ,38 as number ,'2009-02-24' as data,
        Product.product_id,Product.product_name
from Product;

--去除重复行
select  distinct  product_type
from Product;

--null算一类数据，数据去重时候会被保留下来

--多列组合去重
select  distinct Product.product_type,Product.regist_data
    from Product;
rollback ;--这个玩意居然是回滚，怪不得每次东西就不见了，服了我了

--经验。创建或者操作后记得commit提交一下,就这个数据库奇奇怪怪，真的是MySQL简单

select distinct Product.purchase_price
    from Product;
--这个例子证明null是一个数据

--而且而且，distinct，只对第一个去重

--使用where取特定的数字
select Product.product_name,Product.product_type
    from Product
    where product_type='衣服';

--执行逻辑：首先通过WHERE 子句查询出符合指定条件的记录，然后再选取出 SELECT 语句指定的列

--算数运算符：SQL语句里面可以使用算数表达式
select  Product.product_name,Product.sale_price,
        Product.sale_price*2 as "sale_price x 2"
    from Product;

--和平时一样，括号里面的有优先级
--null怎么算，都是null

--比较运算符
select Product.product_name,Product.product_type
    from Product
    where sale_price=500;

select Product.product_name,Product.product_type
    from Product
    where sale_price<>500;

select Product.product_name,Product.sale_price,Product.purchase_price
    from Product
    where sale_price-purchase_price>=500;

--不能对null使用比较

select  Product.product_name,Product.purchase_price
    from  Product
    where purchase_price<>2800;
--没有叉子和圆珠笔，因为他们是空
--OK了，不能用比较运算符，难道我还没办法取null了吗，万一要用呢，没事，有办法
select  Product.product_name,Product.purchase_price
    from  Product
    where purchase_price is null;
--有的是办法。如果想取非null，全面加一个not就好了

--使用not反转
select Product.product_name,Product.product_type,Product.sale_price
    from Product
    where  not  sale_price >=1000;

--如果有多个条件，我既要还要。当然有and辣，还有一个，我这两个，都可以要，or
select  Product.product_name,Product.purchase_price
    from Product
    where  product_type='厨房用品'
    and sale_price>=3000;

select  Product.product_name,Product.purchase_price
    from Product
    where  product_type='厨房用品'
    or sale_price>=3000;

select Product.product_name,Product.product_type,regist_data
    from Product
    where product_type='办公用品'
    and regist_data='2009-9-11'
    or  regist_data='2009-09-20';
--这个是一个错误，我按照自己想法没想到写了一个标准错误，书中说明了，and的优先级高于or
/*
 所以，处理逻辑就变成了
 「product_type = '办公用品' AND regist_date = '2009-09-11'」
OR
「regist_date = '2009-09-20'」
 */
--其实加一个括号就好了
select Product.product_name,Product.product_type,regist_data
    from Product
    where product_type='办公用品'
    and (regist_data='2009-9-11'
         or  regist_data='2009-09-20');


--真值：其实就是true和false。比较运算符会把比较结果已真值的结果返回
--SQL有三值逻辑，第三个是不确定
/*
 练习：
编写一条 SQL 语句，从 Product（商品）表中选取出“登记日期（regist_
date）在 2009 年 4 月 28 日之后”的商品。查询结果要包含 product_
name 和 regist_date 两列。
 */
select Product.product_name,Product.regist_data
    from Product
    where regist_data<='2009-4-28';

/*
 2.2	 请说出对 Product 表执行如下 3 条 SELECT 语句时的返回结果。
① SELECT *
 FROM Product
 WHERE purchase_price = NULL;
② SELECT *
 FROM Product
 WHERE purchase_price <> NULL;
③ SELECT *
 FROM Product
 WHERE product_name > NULL;
 */
--都有问题，没有结果，比较运算符不能对null使用

/*
代码清单 2-22（2-2 节）中的 SELECT 语句能够从 Product 表中取出“销
售单价（sale_price）比进货单价（purchase_price）高出 500
日元以上”的商品。请写出两条可以得到相同结果的 SELECT 语句。
 */
select Product.product_name
    from Product
    where sale_price-purchase_price>=500;

--聚合与排序
/*
 常用函数：
COUNT：计算表中的记录数（行数）
SUM： 计算表中数值列中数据的合计值
AVG： 计算表中数值列中数据的平均值
MAX： 求出表中任意列中数据的最大值
MIN： 求出表中任意列中数据的最小值
 */
select count(*)
    from Product;

select count(Product.purchase_price)--没有把null算进去
FROM Product;

select sum(Product.sale_price)
    from Product;
--有一个好玩的地方，你看，聚合如果有null，他不会想运算一样返回null，因为，传入的时候，就已经排除null，根本没有参加运算

select avg(Product.purchase_price)
    from Product;
--这里有一个好玩的，因为传入就排除了null，所以求平均值的分母是6，不是8

--使用聚合函数删除重复值：distinct
select  count(distinct Product.product_type)
    from Product;

--分组
--group by
select Product.product_type,count(*)
    from Product
    group by product_type;

--如果有null
select Product.purchase_price,count(*)
    from Product
    group by purchase_price;

/*
 使用WHERE子句和GROUP BY子句进行汇总处理
 像这样使用 WHERE 子句进行汇总处理时，会先根据 WHERE 子句指
定的条件进行过滤，然后再进行汇总处理。
 */
select Product.purchase_price,count(*)
    from Product
    where product_type='衣服'
    group by purchase_price;
/*
 常见错误① ——在SELECT子句中书写了多余的列
在使用 COUNT 这样的聚合函数时，SELECT 子句中的元素有严格
的限制。实际上，使用聚合函数时，SELECT 子句中只能存在以下三种
元素。
●	常数
●	聚合函数
● GROUP BY子句中指定的列名（也就是聚合键）

常见错误② ——在GROUP BY子句中写了列的别名
执行select的as起别名时候，人家group by就已经执行完，返回一个取好名的表

常见错误③——GROUP BY子句的结果能排序吗
GROUP BY 子句的结果通常都包含多行，有时可能还会是成百上千
行。那么，这些结果究竟是按照什么顺序排列的呢？
答案是：“随机的。”

常见错误④——在WHERE子句中使用聚合函数
 */


--为聚合结果指定条件HAVING字句
--执行逻辑：SELECT → FROM → WHERE → GROUP BY → HAVING
select Product.product_type,count(*)
    from Product
    group by product_type
    having count(*)=2;

select Product.product_type,avg(Product.sale_price)
    from Product
    group by product_type
    having avg(sale_price)>=2500;
/*
 HAVING 子句和包含 GROUP BY 子句时的 SELECT 子句一样，能
够使用的要素有一定的限制，限制内容也是完全相同的。HAVING 子句中
能够使用的 3 种要素如下所示。
●	常数
●	聚合函数
● GROUP BY子句中指定的列名（即聚合键）
 */
SELECT product_type, COUNT(*)
 FROM Product
 GROUP BY product_type
 HAVING product_type = '衣服';

SELECT product_type, COUNT(*)
 FROM Product
WHERE product_type = '衣服'
 GROUP BY product_type;
--这两个例子结果一样
--WHERE 子句 = 指定行所对应的条件
--HAVING 子句 = 指定组所对应的条件。所以用第二个
/*
因为使用where，那么执行之前就对数据进行处理，所以减少了count的排序对速度的消耗
 */

--对查询结果进行排序
SELECT product_id, product_name, sale_price, purchase_price
 FROM Product;

--利用order by进行排序
SELECT product_id, product_name, sale_price, purchase_price
    FROM Product
    order by sale_price;--升序

SELECT product_id, product_name, sale_price, purchase_price
    FROM Product
    order by sale_price desc ;--降序

--有的商品，目标排序字段一样，那他们几个的顺序是什么。答案是，随机
SELECT product_id, product_name, sale_price, purchase_price
 FROM Product
ORDER BY sale_price, product_id;--指定多个字段，这样子相同也可以继续排序

--ok,那如果，有一个是null，null不是不能比较吗，那怎么办，对于这个机制，很简单，执行时候会直接放开头或者结尾
SELECT product_id, product_name, sale_price, purchase_price
 FROM Product
ORDER BY purchase_price;

--GROUP BY不难永别名，但是ORDER BY可以
SELECT product_id AS id, product_name, sale_price AS sp, purchase_price
 FROM Product
ORDER BY sp, id;
--原因其实和执行顺序有关：FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY

-- ORDER BY 子句中也可以使用存在于表中、但并不包含在 SELECT子句之中的列
SELECT product_name, sale_price, purchase_price
 FROM Product
ORDER BY product_id;

--也可以使用聚合函数
select  Product.product_type,count(*)
    from Product
    group by product_type
    order by count(*);

--可以通过列编号来指定
-- 通过列名指定
SELECT product_id, product_name, sale_price, purchase_price
 FROM Product
ORDER BY sale_price DESC, product_id;
-- 通过列编号指定
SELECT product_id, product_name, sale_price, purchase_price
 FROM Product
ORDER BY 3 DESC, 1;

--练习
/*
 3.1 请指出下述 SELECT 语句中所有的语法错误。
SELECT product_id, SUM(product_name)
-- 本SELECT语句中存在错误。
 FROM Product
 GROUP BY product_type
 WHERE regist_date > '2009-09-01';

 3.2 请编写一条 SELECT 语句，求出销售单价（sale_price 列）合计值是
进货单价（purchase_price 列）合计值 1.5 倍的商品种类。执行结果
如下所示。

 3.3	 此前我们曾经使用 SELECT 语句选取出了 Product（商品）表中的全部记
录。当时我们使用了 ORDER BY 子句来指定排列顺序，但现在已经无法记
起当时如何指定的了。请根据下列执行结果，思考 ORDER BY 子句的内容。
 */
--3.1where和group by顺序不对 group by 出现了没有的列 产品名称求和
--3.2
select Product.product_type,sum(sale_price) as sum,sum(purchase_price) as sum
    from Product
    group by product_type
    having  sum(sale_price)>=1.5*sum(purchase_price);


--数据更新INSERT

INSERT INTO Product VALUES ('0002', '打孔器',
'办公用品', 500, 320, '2009-09-11');

-- 省略列清单
INSERT INTO Product VALUES ('0005', '高压锅', '厨房用具',
6800, 5000, '2009-01-15');

--如果要插入null，直接写null就好
/*
 可以用下面的句子，可以给一些列设置默认值

CREATE TABLE ProductIns
(product_id CHAR(4) NOT NULL,
 （略）
 sale_price INTEGER DEFAULT 0, -- 销售单价的默认值设定为0;
 （略）
 PRIMARY KEY (product_id));

 --使用
 INSERT INTO ProductIns (product_id, product_name, product_type,
sale_price, purchase_price, regist_date) VALUES ('0007',
'擦菜板', '厨房用具', DEFAULT, 790, '2009-04-28');
 */


```

