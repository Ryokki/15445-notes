这节两课，我们会学习`数据库内部的存储`，讨论`如何使用tuple来表示数据`

Database Storage 在 CMU 分成了两部分，在两节课中讲。这是第二部分。

这部分设涉及`基础数据类型`的表示，OLAP, OLTP, HTAP, row-store (NSM), column store (DSM) 等等。

![image-20210509164020229](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509164020229.png)

从⾼级层⾯来讲，⼀个tuple就是⼀个字节序列，就是⼀个字节数组，这就取决于DBMS如何去解释它的意思，以及弄清楚它的类型，就⽐如说，Oh这是⼀个 Integer，这是⼀个float，这⾥⼜是⼀个字符串属性之类的东⻄。    ———— 这就是这节课要做的事

我们要将这些字节数组组织成我们的tuples，然后，当数据库系统执⾏查询时，我们需要去解释下这些字节数组中的实际内容，以此来⽣成我们所寻找的答案



# 1.Data Representation(数据表示方式)

Page中就是tuple的集合，tuple是a sequence of bytes，但如果我们要使用这些数据，首先要把这些bytes转化为相应类型的数据

主要的类型如下:

![image-20210509164252290](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509164252290.png)

> `整数：`和C++的表示一致
>
> `小数:`有浮点数也有定点数
>
> `不定长数据:`会在Header处存一个length，后面跟着实际的数据（C++是用'\0'来表示结束，而这里是给个Header来表明长度）
>
> `时间类:`不同的数据库中实现不一样。看似分成了Time，Date等，其实存储时，保存的都是完整的时间戳，只是查询时去掉了不需要的部分。在大多数系统中：它们会去保存从1970年1⽉1⽇起的秒数或毫秒数或者微秒数来 处理时间



`Variable Length Data `（可变长度数据）

* An array of bytes of arbitrary length.

  一个由任意长度的字节组成的数组。

* Has a header that keeps track of the length of the string to make it easy to jump to the next value.

  有一个header，记录着字符串的长度，以便于跳到下一个值。

* Most DBMSs do not allow a tuple to exceed the size of a single page, so they solve this issue by writing the value on an overflow page and have the tuple contain a reference to that page.

  大多数DBMS不允许一个元组超过一个Page的大小，所以他们通过在一个溢出(overflow )的Page上写值来解决这个问题，并让元组包含对该页面的引用。

* Some systems will let you store these large values in an external file, and then the tuple will containa pointer to that file.  For example,  if our database is storing photo information,  we can store thephotos in the external files rather than having them take up large amounts of space in the DBMS. Onedownside of this is that the DBMS cannot manipulate the contents of this file.

  有些系统会让你把这些大的数值存储在一个外部文件中，然后元组将包含一个指向该文件的指针。 例如，如果我们的数据库正在存储照片信息，我们可以将照片存储在外部文件中，而不是让它们在数据库管理系统中占据大量空间。这样做的坏处是，DBMS不能操纵这个文件的内容。

* Example:VARCHAR,VARBINARY,TEXT,BLOB.

## 1.1 Variable Precision Numbers 浮点数

•不精确的可变精度数值类型，它使用IEEE-754标准指定的“本机”C / C ++类型。
•比任意精度数字更快，因为CPU可以直接对它们执行指令。
•示例：FLOAT，REAL

![image-20210509165550461](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509165550461.png)

- https://en.wikipedia.org/wiki/IEEE_754
- 对 `FLOAT`, `REAL/DOUBLE` 的操作会比较**快**<`优点`>，因为 CPU 有直接对应的指令 (instruction) 可以使用，但是因为**精度有限**<`缺点`>的原因，使用这几个依然会失去精度。(和编程语言中一样)。因为计算机对数字存储是离散的，有限的，必然会失去一定精度，结果是近似的。

### 1.1.1 Demo - 失去精度

![image-20210509165934018](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509165934018.png)

上面这个没有问题，正确输出0.3

![image-20210509170115551](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509170115551.png)

上面这个就有问题了，可以看到，我们规定了保留20位小数时，这个精度就出现了问题，因为硬件没法精确表示浮点数

你们可能会想，在我这个⼩例⼦中，我们知道答案是0.3，这就够了，谁会去在意舍⼊误 差呢。 但是如果是在科学计算上遇上这种问题，那么当你试着将某物送⼊太空时，如果你发出的指令上有这 种舍⼊误差，那么这就会引发现实问题了，为了避免这种事情发生，我们会去使⽤⼀种称为固定精度数字的数字

如果希望允许数据**精确到任意精度**（arbitrary precision），则可以使用 `numeric/decimal` 类型类存储，它们就像 VARCHAR 一般，长度不定，见下1.2的介绍.



## 1.2 Fixed Precision Numbers 定点数

基本思路就是你将这个值作为varchar类型来存储，即⽤⼈类实际可读的值的形式 来表示，接着，通过⼀些元数据来表示，这⾥是⼩数点，那⾥是精度范围，接着另外⼀边是四舍五⼊信息。这些东⻄都要放在tuple⾥⾯，并且它是该字节数组的⼀部分

![image-20210509170603601](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509170603601.png)



* 具有任意精度和比例的数字数据类型。 通常存储在精确的可变长度二进制文件中具有附加元数据的表示。

* 只能表示固定精度

* 当舍入错误不可接受时使用。
* 示例：NUMERIC，DECIMAL



让我们来看一下定点数`NumericDigit`是怎么表示的（不需要理解细节噢，知道架构就行）：

![image-20210509171649614](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509171649614.png)



```c++
typedef unsigned char NumericDigit;
typedef struct 
{ 
    /*这个定点数的一些信息*/
    int ndigits; 
    int weight; 
    int scale; 
    int sign; 
    /*digit:是一个字符串,这是真正用来存储的实际数据,我们会根据上面的这些信息对这个数据进行解密*/
    NumericDigit *digits;	
} numeric;
```



PostgreSQL Source Code: https://doxygen.postgresql.org/interfaces_2ecpg_2pgtypeslib_2numeric_8c_source.html#l00722

我们通过源码中来看下，它实际上是如何做加法的

![image-20210509172054021](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509172054021.png)



源代码可以看到，这里做加法的代码如上，可以看到，这里有若干个if判断以及switch（⽐如说，要判断它的正负，是不是0，或者两个数字是 否相等），所以会比浮点数的运算要慢约2倍（浮点数时，cpu只需要一条指令即可完成加法）



Ok，如果我们不想因为精度问题⽽丢失数据，那我们就使⽤Fixed point decimal，但我们需要在我们的数据库系统中实现这个。





## 1.3 Large Values 超大值

一般我们不会允许一个tuple的值超过一个页的大小。但是万一数据库帧的需要存一个很大的值该怎么办呢？

1. 用指针指到一个OverFlow Page，这个依然是数据库Page的组织形式，且对用户是透明的

2. 直接引用外部文件，这个就脱离了数据库的文件管理了

   

由于tuple大小不能超过page，比如对于`varchar`，如果大于page，需要把多的存放到`overflow page`里面

![image-20210509172744171](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509172744171.png)



- 如果 `c` 很大，甚至超过了一个 page 的大小。比如 `c` 是一个 tuple 中一个很长的 `VARCHAR` 字段。对于这个超过一个 page 大小的 `c`，我们可以把它额外存储在一个 *overflow page* 上，这时上图中的 `c` 实际上是一个指向 overflow page 的一个指针。
- 当然 overflow page 可以是多个。假如一个 overflow page 依然不够大，我们可以使用几个 overflow page，它们之间继续**用指针相连**。
- 整体上 overflow page 只是一种实现存储 large value 的方式。它对使用数据库的应用是**透明的 (transparent)**, 使用数据库的应用只获得那个很长的 `c` 字符串，而不必知道它是如何被存储，存储在哪儿。





## 1.4 External Value Storage 外部存储

而对于blob<Binary Large Object data>这样的大的数据，干脆就需要存放到外部文件中，直接放在磁盘中，然后在Tuple中记下这个硬盘的位置(文件路径)

![image-20210509205033887](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509205033887.png)

- External Value Storage 是指那些特别大的文件，超过我们前面提到的 large value。它比如说是 GiB 大小的视频。
- BLOB: Binary Large Object data (二进制大型文件)
- 这种文件，我们没有必要把它存储在数据库内部，这样会浪费数据库空间。
- 我们直接在 `c` 的地方存储该文件在操作系统文件系统中硬盘的位置。这样做不会浪费数据库内部空间。
- 如果我们使用的 High End 的设备去运行高性能数据库，那么该服务器中磁盘容量是很宝贵的，不应该被浪费的。在 `c` **存储一个硬盘位置，能降低数据库的成本**，特别是这个硬盘可以是 HTFS 或者 network storage。
- 这里注意对于存放到外部文件的数据，是不保证 **transaction**<`事务`>和**durability protection**<`持久性保障`> 等语义的，所以往往只允许读，不能写.（毕竟那已经不是属于数据库管辖的文件了，那是整个操作系统管理的）



> 应用举例：
>
> 如果你想要在数据库中上传GB大小的视频，直接用数据库文件的一个page一个page去存，非常耗费资源，所以往往就是采用这种外部存储.
>



`超大值vs外部存储:`

——一般来说，不是很大的数据，用超大值的`overflow`的方法会更有效率，因为他在数据库文件系统里，所有的文件都已经打开好了，就可以直接去取数据了，而去文件系统的话就要根据指针去文件系统找到这个文件，然后用f.open打开，才能去获取数据，这样效率不高。	而对于很大的数据，用外部存储会更好。（这里感觉应该更多是经济的考量？）

# 2. System Catalogs 数据库文件目录

那么现在有个关键的问题，数据库的元数据是存储在什么地方的？

​	——`Catalogs`：目录

一个 DBMS 会将与数据有关的一些元数据存储到它内部的目录里：包括表格，列，索引，用户和权限还有内部统计数据等等。同时这个目录会和数据库的内容一起存在 DBMS 里。

![image-20210509211154706](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509211154706.png)

* DBMS的元数据存在目录中，这些元数据包括：
  * 表名、列、索引、视图
  * 用户权限
  * 内部数据统计（帮助知道数据的特征，以查询优化）

* 几乎所有的DBMS都会有一个`catalog`来存储所有的元数据，很多数据库系统都会将它们的Catalog⽤另⼀张表来保存

![image-20210509211953676](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509211953676.png)

* 几乎所有的数据库都会把目录<catalog>暴露出来

![image-20210509212230892](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509212230892.png)

比如mysql中：

```mysql
SHOW TABLES;
DESCRIBE student;
```

各种数据库都有各自的API，但是最终在内部，都会重写成标准所规定的那样：

```sql
SELECT * FROM INFORMATION_SCHEMA.TABLES
```

![image-20211012001736193](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20211012001736193.png)

举个例子：

```mysql
select TABLE_CATALOG,TABLE_SCHEMA,TABLE_NAME  from INFORMATION_SCHEMA.TABLES;
```

执行结果：

![image-20211012002548881](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20211012002548881.png)





![image-20210509213202461](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509213202461.png)



> 当我们查询或者构建索引时等等，我们都会用到目录.
>
> 用目录去查一个元组是很没有效率的一件事，因为你查一个元组，首先要根据目录去查这个表，查这个表的结构，查这个表的各个偏移量代表什么意思（数据类型），但是用这些元数据查到这个元组后，之后这些元数据如果要用，又得重新做一遍上面的步骤，而不能复用，因此很多数据库都采取了一个优化：
>
> ![image-20210510113211673](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510113211673.png)
>
> ![image-20210510113225078](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510113225078.png)
>
> ![image-20210510113239643](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510113239643.png)
>
> * 也就是把这些反复会用到的元数据`预先编译`好。





# 3.Storage Levels

![image-20210510113629557](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510113629557.png)

这是维基百科的数据库的设计。在这里，我们有三张表：

* `useracct`——<储存用户信息>
  * UserID  (Key)
  * UserName
* `pages`——<储存网页的基本信息>
  * PageID (key)
  * title
  * latest(最近一次的更新记录)（总是指向最新的版本<所以无需扫描，直接到这里拿数据>）
* `revision`——<保存每篇文章的新更新记录>
  * revID (key)
  * userID
  * pageID
  * content（网页的内容）
  * updated (更新的时间)



数据库的应用场景大体可以用两个维度来描述：**操作复杂度和读写分布**，如下图所示：

![image-20211012003510843](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20211012003510843.png)

1. 坐标轴左下角是 OLTP（On-line Transaction Processing），OLTP 场景包含简单的读写语句，且每个语句都只操作数据库中的一小部分数据
2. 在坐标轴右上角是 OLAP（On-line Analytical Processing），OLAP 主要处理复杂的，需要检索大量数据并聚合的操作

## 3.1 OLTP （联机事务处理）

![image-20210510133411101](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510133411101.png)

* `OLTP`——<On-line Transaction Processing>——`联机事务处理`
* OLTP是传统的关系型数据库的主要应用，主要是基本的、日常的事务处理，例如银行交易。
* 通常是对很小一部分 tuple 的写操作
* 适用于快速、短暂的事务
* 占用资源少
* 举例：亚马逊上买东西时，就可以把它看作是对应用程序的OLTP（联机事务处理），每个人买东西时只会更新很小一部分数据，比如这个人的购买记录，所以可以用OLTP。



OLTP：每次查询从外部读取一小部分数据然后存入（更新）本地数据库中，对应的维基百科例子操作如下所示：

```mysql
# 更新页面
SELECT P.*, R.*
    FROM pages AS P
  INNER JOIN revisions AS R
    ON P.latest = R.revID
  WHERE P.pageID = ?
```

```mysql
# 登陆账户
UPDATE useracct
    SET lastLogin = NOW(),
        hostname = ?
    WHERE userID = ?
```

```mysql
# 插入数据
INSERT INTO revisions
    VALUES (?,?…,?)
```



## 3.2 OLAP (联机分析处理)

![image-20210510134514285](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510134514285.png)

`OLTP`:<On-line Analytical Processing>(`在线分析处理`)，通常是对很大一部分是 tuple 做读操作，做复杂的分析

* 长期运行，更复杂的查询
* 读取数据库的大部分内容
* 探索性查询
* 从OLTP方面收集的数据中得出新的数据
* 例子：计算这些地理位置的人在一个月内购买最多的五件物品。



当你们已经从OLTP应⽤程序中收集到⼀⼤堆数据时，现在，你们想去分析它，并从中推断出新的信息，这种东⻄有时被称为数据科学，即从你拥有的数据中试着派⽣出新的信息。我们以Wikipedia中的查询为例：	假设，我们想去统计每个⽉的主机名以.gov结尾的⽤户登录数量

```mysql
SELECT COUNT(U.lastLogin)
       EXTRACT (month FROM U.lastLogin) AS month
  FROM useracct AS U
 WHERE U.hostname LIKE '%.gov'
 GROUP BY 
 	   EXTRACT(month FROM U.lastLogin);
```



>`OLTP` vs `OLAP:`
>
>数据库系统一般可以按照负载类型分成操作型数据库（Operational Support System）和决策型数据库（Decision Support System）。操作型数据库主要用于应对日常流水类业务，主要是面向消费者类的业务；决策型数据库主要应对的是企业报表类，可视化等统计类业务，主要面向企业类的业务。
>
>联机事务处理OLTP（on-line transaction processing）
>
>OLTP是传统的关系型数据库的`主要应用`，主要是基本的、日常的事务处理，例如银行交易。
>
>联机分析处理OLAP（On-Line Analytical Processing）
>
>OLAP是数据仓库系统的主要应用，支持复杂的分析操作，侧重`决策支持`，并且提供直观易懂的查询结果。
>
>OLTP的数据定期会通过etl（提取，转换，加载）工具把数据同步导入OLAP系统中。这就涉及到数据源滞后的问题。 OLAP的数据滞后，导致分析出来的结果时效性不够，对决策支持类系统的要求不够。比如说，双11期间，用户购物的行为和推荐系统的推荐结果之间的时间差越短，越有可能提高销量。
>
>
>下表列出了OLTP与OLAP之间的比较：
>
>|          | OLTP                                 | OLAP                                    |
>| -------- | ------------------------------------ | --------------------------------------- |
>| 用户     | 操作人员，低层管理人员               | 决策人员，高级管理人员                  |
>| 功能     | 日常操作处理                         | 分析决策                                |
>| DB 设计  | 面向应用                             | 面向主题                                |
>| 数据     | 当前的， 最新的细节的， 二维的分立的 | 历史的， 聚集的， 多维的集成的， 统一的 |
>| 存取     | 读/写数十条记录                      | 读上百万条记录                          |
>| 工作单位 | 简单的事务                           | 复杂的查询                              |
>| DB 大小  | 100MB-GB                             | 100GB-TB                                |
>
>总的来说，OLTP就是面向我们的应用系统数据库的，OLAP是面向数据仓库的.



## 3.3 HTAP (混合事务分析处理)

数据库的应用场景大体可以用两个维度来描述：操作复杂度和读写分布，如下图所示：

![image-20210510135130442](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510135130442.png)

* `HTAP:`<Hyper Transaction analytical processing>——`混合事务分析处理`

* 其实就是处理用户的事务时，也要用到我们OLAP分析出来的数据

> HTAP是混合 OLTP 和 OLAP 业务同时处理的系统，2014年Garnter公司给出了严格的定义：混合事务/分析处理(HTAP)是一种新兴的应用体系结构，它打破了事务处理和分析之间的“墙”。它支持更多的信息和“实时业务”的决策。
>
> 当前流行的方案是，采用快照的方式，分开处理OLTP和OLAP请求。让OLAP的请求在OLTP的最新的一致性快照上执行。同时对外暴露一套接口，从而从逻辑来看是一套系统。虽然内部是分开处理OLTP和OLAP的。重要的一点，就是保证快照是尽可能的保持“新”，快照不能太过滞后OLTP的数据。这就需要系统频繁的做快照操作。
>
> 目前两种流行的方案，一个是采用linux的系统快照能力，提供HTAP服务的方案，比如Hyper数据库系统。另一种是类似hana的方案，定期生成增量数据，然后合并到AP系统。



# 4.Storage Models 存储模型

![image-20210510135951299](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510135951299.png)

在了解两种工作方式，我们可以开始考虑存储模型。DBMS 针对 OLTP 或者 OLAP 工作方式可以有不同的方式去存储 tuples。目前我们假定统一用 n-ary 存储方式（行存储）。

## 4. 1 N-ary Storage Model (NSM): 行存储

![image-20210510140524160](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510140524160.png)

用这种模型 DBMS 连续存储 tuple 的所有数据在一个Page里，tuple 之间也是连续存放。这种方式对 OLTP 比较有利，因为大部分时间只需要读取/更新/插入单条信息（tuple）。

### 4.1.1 OLTP (good)

OLTP 往往有一个 index (索引)，如例子中的 userID，一个查询语句通过 Index 找到相应的 tuples，所以可以快速找到 OLTP 需要的那一个 tuple。

![image-20210510141132433](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510141132433.png)

图中可以看到，我们的一行行tuple存在一个Page中，这就是我们前面说的Page-slotted的存储方式

![image-20210510141246865](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510141246865.png)

* select：我们可以根据索引，`索引`会告诉我们我们想要的tuple所在的page和slot number，然后我们从这个位置取出来即可

  假设我要进⾏⼀次查询，我想根据给定的⽤户名和密码拿到所有的账号信息，我可以根据索引来进⾏查找，但基本上来讲，数据库会告诉我们，我们想要tuple所在的page id和slot number。我通过⼀次查找和读取，将该page放⼊内存中，然后我可以直接跳到我想要的那个数据所在的位置，我们的workload看起来就像图上这样，拿到一个实体的数据。So，让⼀个tuple的所有数据连续地放在⼀起是读取数据时最有效的⽅式。

* insert：我们找到空闲slot，然后插入这行tuple数据

  我只需找到⼀个空闲slot，并将所有数据⼀次性写⼊，接着将它刷⼊磁盘，并记录⽇志。



可以看到，数据量小的时候，select或insert都只需要查询或插入 1次Page即可，所以行存储效率很高。



优点：

* 快速插入，更新和删除
* 对于需要整个 tuple 的查询指令比较友好

缺点：

* 对于扫描某列的查询指令不友好（效率低）



### 4.1.2 OLAP (bad )

![image-20210510141657901](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510141657901.png)

OLAP 往往需要扫描很大一部分的表格，这种情况 index 的帮助不会很大。**扫描**具体就是从每一个 tuple 头到尾，我们之前提过 tuple 就是一个 byte array。即使我们下图中，只需要检查深蓝色和浅蓝色两个字段，但是整个 tuple 还是需要从内存加载到 CPU cache。而且一个 page 上的大部分数据也都和当前 query 无关，这实际上很不高效。所以下图红圈中的都是 useless data。

![image-20210510141719603](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510141719603.png)

> 所以，这里的问题就在于，我们查询时往往只需要一列或者某几列，但是用行存储的情况下，由于读取磁盘只能一次性把整个page都读过来，所以实际上，别的列对我们来说是无用的数据，但为了得到我们实际想要的那些列，我们不得不从磁盘把他们取出来放到内存
>
> （其实就是无用的列总是会浪费Page的空间，然后需要读更多的Page）



![image-20210510142545180](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510142545180.png)

`优点:`

* 可以快速的插入，更新和删除操作
* 对于需要整行数据的查询来说，效率已经最高了

`缺点:`

* 对于查询等操作时只用到某几列时，这不是一个good choice.



> 行存储适合OLTP，因为查询量少，往往查询的数据就是在一个Page中，直接查询一次就可以把所有要用的数据都拿到磁盘，不适合OLAP，因为查询量大，用行存储往往有很多用不到的列，这样会使得查询的次数变多了，性能差.





## 4.2 Decomposition Storage Model (DSM):列存储

![image-20210510143457569](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510143457569.png)

* `DBMS在一个页面中连续存储所有元组的单一属性值`。（也被称为 "列存储"）
* 列存储是`OLAP`工作负载的理想选择，只读查询在表的属性子集上进行大量扫描。



`row storage:`

![image-20210510143649176](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510143649176.png)

`col storage:`

![image-20210510143751276](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510143751276.png)

一个Page存放单个属性




![image-20210510143827443](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510143827443.png)

`Step:`

1. 拿到`hostname`的所有Page都放在内存中，然后对每个hostname进行扫描以及判断
2. 得到一系列匹配的tuple，我们将`lastLogin`的所有Page都放在内存中，遍历lastLogin，根据前面查到的匹配的tuple，找到这些匹配的lastLogin，并最终生成COUNT

> 既然列式存储，是按属性为单位存储的，那么怎么能区分出那个属性属于哪个tuple呢？解决方案有两种：
>
> 1. Fixed-length Offsets：每个 attribute 都是定长的，直接靠 offset 来跟踪（常用）。
>
>    使用固定长度的偏移量，这意味着对于一列中的每个值来说，它们的长度始终是固定的，当然如果属性是INT类型自然是固定长度的，但是如果是可变类型，就必须压缩所有tuple中的这一属性，让它们统一成一样的长度。
>
> 2. Embedded Tuple Ids：在每个 attribute 前面都加上 tupleID
>
>    对于列中的每一个值，都保存一个主键或是标识符。这种做法的问题就在于每个属性都要浪费额外的空间来存储标识符，所以基本没有哪个数据库采用这种方案。
>
> <img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20211012011744697.png" alt="image-20211012011744697" style="zoom:50%;" />
>
> 列式存储的优势在于避免读取不必要的数据节约了IO，劣势在于操作单一数据时会很慢，因为它要结合多个page的内容才能构建出一个完整的tuple。



而且，列存储由于是将同种属性的数据放在了一起，这样往往可以有很好的压缩效果，比如整个属性是温度，我们记录下平均温度和相对平均温度的差值，这样来做到压缩（比如原本每个page只能放1000个tuple，这种压缩后可以放10000个，就带来很好的性能提升）



![image-20210510150920548](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510150920548.png)



DBMS在一个数据块中连续存储所有元组的单个属性。也称为“列存储”。此模型非常适用于OLAP工作负载，其中只读查询对表的属性子集执行大型扫描。

优点：

* 减少需要的I/O数（只读取需要的数据）
* 因为同一个page 保存的是相同的类型，所以可以有数据压缩的方法

缺点：

* 对于查询（插入，更新，删除）单个 tuple 效率较低。



![image-20210510151430035](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510151430035.png)



# 5. Conclusion

![image-20210510151727867](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510151727867.png)



> Row stores are usually better for OLTP, while column stores ar better for OLAP.

大部分现代数据库:

- Frontend: OLTP in client, 处理用户的事务
- Backend: OLAP in Server, data warehouse　大数据分析所有用户的资料 (比如订单)

![image-20210510152617283](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510152617283.png)

![image-20210510152659718](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510152659718.png)

- 上图的客户端为传统的 OLTP Data Silos，将数据发到 Server - Data Warehouse (数据仓库)。Server 端运行 OLAP, 分析这些用户的信息。
- ETL (Extract-Transform-Load)，将数据从来源端经过抽取 extract、转换 transform、加载 load 至目的端的过程。是一个常用在数据仓库的技术。Server 短得到这些 ETL 后的数据，就可以运行 OLAP Query 或者数据挖掘等机器学习算法。

![image-20210510152844713](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510152844713.png)

- 上图的客户端有 HTAP， 即客户端中运行 TP 和 AP，再将在客户端中处理过的数据 ETL 至 Server - Data Warehouse。

> 比如，我们我们可以在其前端数据库用Mysql，Sql等，后端数据库(OLAP)可以用Hadoop，Spark，Greenplum或Vertica等



![image-20210510152106562](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210510152106562.png)

