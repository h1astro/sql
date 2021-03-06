## 05_window_function

> notes written by h1astro
>
> 学习了窗口函数的使用，partition by用来分组结合order by对窗口内分组的进行排序。了解了RANK、DENSE_RANK、ROW_NUMBER函数，以及计算平移平均，PRECEDING截止到之前n行以及FOLLOWING。最后学习了ROLLUP合计及小记函数的使用。

窗口函数也称OLAP函数(OnLine Analytical Processing)的简称，对数据库数据进行实时分析处理。常规的SELECT语句都是对整张表进行查询，而窗口函数可以让我们有选择的去某一部分数据进行汇总、计算和排序。

```sql
<窗口函数> OVER ([PARTITION BY <列名>]
                     ORDER BY <排序用列名>)  
```

**PARTITION BY** 是用来分组，即选择要看哪个窗口，类似于 GROUP BY 子句的分组功能，但是 PARTITION BY 子句并不具备 GROUP BY 子句的汇总功能，并不会改变原始表中记录的行数。

**ORDER BY** 是用来排序，即决定窗口内，是按那种规则(字段)来排序的。

```sql
SELECT product_name
       ,product_type
       ,sale_price
       ,RANK() OVER (PARTITION BY product_type
                         ORDER BY sale_price) AS ranking
  FROM product;  
```

![](imgs/05/partition.png)

PARTITION BY 能够设定窗口对象范围。本例中，为了按照商品种类进行排序，我们指定了product_type。即一个商品种类就是一个小的"窗口"。

ORDER BY 能够指定按照哪一列、何种顺序进行排序。为了按照销售单价的升序进行排列，我们指定了sale_price。此外，窗口函数中的ORDER BY与SELECT语句末尾的ORDER BY一样，可以通过关键字ASC/DESC来指定升序/降序。省略该关键字时会默认按照ASC，也就是

升序进行排序。本例中就省略了上述关键字

![](imgs/05/ch0502.png)

### 窗口函数种类

大致可以分为两类。

一是 将SUM、MAX、MIN等聚合函数用在窗口函数中

二是 RANK、DENSE_RANK等排序用的专用窗口函数

#### 专用窗口函数

* **RANK函数**

计算排序时，如果存在相同位次的记录，则会跳过之后的位次。

例）有 3 条记录排在第 1 位时：1 位、1 位、1 位、4 位……

* **DENSE_RANK函数**

同样是计算排序，即使存在相同位次的记录，也不会跳过之后的位次。

例）有 3 条记录排在第 1 位时：1 位、1 位、1 位、2 位……

* **ROW_NUMBER函数**

赋予唯一的连续位次。

例）有 3 条记录排在第 1 位时：1 位、2 位、3 位、4 位

```sql
SELECT  product_name
       ,product_type
       ,sale_price
       ,RANK() OVER (ORDER BY sale_price) AS ranking
       ,DENSE_RANK() OVER (ORDER BY sale_price) AS dense_ranking
       ,ROW_NUMBER() OVER (ORDER BY sale_price) AS row_num
FROM product;  
```

![](imgs/05/partition2.png)

####  聚合函数在窗口函数上的使用

聚合函数在开窗函数中的使用方法和之前的专用窗口函数一样，只是出来的结果是一个**累计**的聚合函数值。

```sql
SELECT  product_id
       ,product_name
       ,sale_price
       ,SUM(sale_price) OVER (ORDER BY product_id) AS current_sum
       ,AVG(sale_price) OVER (ORDER BY product_id) AS current_avg  
  FROM product;  
```

![](./imgs/05/over.png)

![图片](./imgs/05/ch0505.png)

聚合函数结果是，按我们指定的排序，这里是product_id，**当前所在行及之前所有的行**的合计或均值。即累计到当前行的聚合。

### 窗口函数的的应用 - 计算移动平均

聚合函数在窗口函数使用时，计算的是累积到当前行的所有的数据的聚合。 实际上，还可以指定更加详细的**汇总范围**。该汇总范围成为**框架(frame)。**

```sql
<窗口函数> OVER (ORDER BY <排序用列名>
                 ROWS n PRECEDING )  
                 
<窗口函数> OVER (ORDER BY <排序用列名>
                 ROWS BETWEEN n PRECEDING AND n FOLLOWING)
```

PRECEDING（“之前”）， 将框架指定为 “截止到之前 n 行”，加上自身行

FOLLOWING（“之后”）， 将框架指定为 “截止到之后 n 行”，加上自身行

BETWEEN 1 PRECEDING AND 1 FOLLOWING，将框架指定为 “之前1行” + “之后1行” + “自身”

```sql
SELECT  product_id
       ,product_name
       ,sale_price
       ,AVG(sale_price) OVER 
       		(ORDER BY product_id
           ROWS 2 PRECEDING) AS moving_avg
       ,AVG(sale_price) OVER 
       		(ORDER BY product_id
       		ROWS BETWEEN 1 PRECEDING 
        	AND 1 FOLLOWING) AS moving_avg  
FROM product;  
```

![](./imgs/05/rows.png)

注意观察框架的范围。

ROWS 2 PRECEDING：

![](./imgs/05/ch0506.png)

ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING：

![](./imgs/05/ch0507.png)

#### 窗口函数适用范围和注意事项

* 原则上，窗口函数只能在SELECT子句中使用。
* 窗口函数OVER 中的ORDER BY 子句并不会影响最终结果的排序。其只是用来决定窗口函数按何种顺序计算。

### GROUPING运算符

#### ROLLUP - 计算合计及小计

常规的GROUP BY 只能得到每个分类的小计，有时候还需要计算分类的合计，可以用 ROLLUP关键字。

```sql
SELECT  product_type
       ,regist_date
       ,SUM(sale_price) AS sum_price
 FROM  product
GROUP BY product_type, regist_date WITH ROLLUP;  
```

<img src="./imgs/05/rollup.png" style="zoom:50%;" />

<img src="./imgs/05/ch0508.png" style="zoom:80%;" />

### 练习题

#### 5.1

请说出针对本章中使用的 product（商品）表执行如下 SELECT 语句所能得到的结果。

```sql
SELECT  product_id
       ,product_name
       ,sale_price
       ,MAX(sale_price) OVER (ORDER BY product_id) AS Current_max_price
  FROM product;
```

展示四列数据，分别为product_id, product_name, sale_price, Current_max_price. 其中Current_max_price，按照product_id顺序排下来，显示当前及前几行最大的sale_price。

#### 5.2

继续使用product表，计算出按照登记日期（regist_date）升序进行排列的各日期的销售单价（sale_price）的总额。排序是需要将登记日期为NULL 的“运动 T 恤”记录排在第 1 位（也就是将其看作比其他日期都早）



```sql
SELECT regist_date,SUM(sale_price)
FROM   product
GROUP BY regist_date
ORDER BY regist_date;
```

<img src="./imgs/05/answer1.png" style="zoom:60%;" />

#### 5.3

思考题

① 窗口函数不指定PARTITION BY的效果是什么？

② 为什么说窗口函数只能在SELECT子句中使用？实际上，在ORDER BY 子句使用系统并不会报错。

①  不会根据product_type里面对每个类别的sale_price进行排序，而是对整个sale_price

② 窗口函数一般用于对where或者group by子句处理后的数据进行操作，（很明显按照sql执行顺序）所以窗口函数原则上只能写在select子句中。

