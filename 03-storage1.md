[CMU 15-445 Ⅰ 数据库存储 | XonLab](https://xonlab.com/2020/11/08/15-445 Ⅰ 数据库存储/)

## Disk Manage简介

`课程大纲:`

![image-20210508142916469](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210508142916469.png)



`我们这门课是面向磁盘型的数据库系统设计`

![image-20210508143227082](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210508143227082.png)

DBMS假定数据库的主要存储位置是在非易失性磁盘上。
DBMS的组件管理了数据从非易失性存储到易失性存储之间的移动

由于是面向磁盘型的数据库，数据库的主要存储位置是在非易失性磁盘上，所以我们执行查询时，所访问的数据不在内存，而在磁盘



传统的 DBMS 架构都属于 **disk-oriented architecture**，即假设数据主要存储在非易失的磁盘（non-volatile disk）上。于是 DBMS 中一般都有磁盘管理模块（disk manager），它主要负责数据在非易失与易失（volatile）的存储器之间的移动。

这里需要理解两点：

* 为什么需要将数据在不同的存储器之间移动？
* 为什么要自己来做数据移动的管理，而非利用 OS 自带的磁盘管理模块？



`计算机的存储层次结构:`

![image-20210508143522374](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210508143522374.png)

如果数据是保存在易失性存储中的话，那它就⽀持快速随机访问。这就意味着，我们可以快速跳转到该存储设备中的任意位置处，⽆论我们访问数据的顺序是什么样的，访问的速度都⼤体⼀致。

* 易失性意味着，如果你从机器上拔掉电源，那么数据就会丢失。

* 易失性存储支持快速随机访问的字节寻址位置。  这意味着程序可以跳转到任何字节地址并获得其中的数据。

* 为了我们的目的，我们总是将这种存储类别称为 "内存"。



对于⾮易失性存储，它们具备的是**块可寻址能⼒**，⽽不是字节可寻址。在⾮易失性存储中，比如只想读取64bit⼤⼩的数据，不得不去获取该数据所存放的那个⼤⼩为4kb的⻚⾯，然后再从中获取我想要的那部分数据。		&&            另一方面是这些系统通常也具有更快的循序访问，这意味着，在⾮易失性存储设备中，⽐起随机读取不同位置上的内容，我可以更有效率地去读取 ⼀段连续的块中的内容，思考这点的最简单的⽅式就是去想象⼀下硬盘旋转的情况

* 非易失性是指存储设备不需要提供持续的电源，以使设备保留它所存储的比特。

* 它也是块/页可寻址的。 这意味着，为了读取一个特定偏移量的数值，程序首先必须将4KB的页面加载到内存中，以保存程序想要读取的数值。

* 非易失性存储在传统上更擅长顺序访问（在同一时间读取多块数据）。

* 我们将把它称为 "磁盘"。 我们不会对固态存储（SSD）或旋转硬盘（HDD）进行（重大）区分。

![image-20210508144324008](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210508144324008.png)

磁盘管理模块的存在就是为了**同时获得易失性存储器的性能和非易失性存储器的容量，让 DBMS 的数据看起来像在内存中一样。**

![image-20210508144545151](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210508144545151.png)

我们系统设计的目标是：允许DBMS管理超过可用内存量的数据库。向磁盘读/写是很昂贵的，所以必须小心管理，以避免性能下降。

所以，我们需要试着在我们的数据库系统中达成的⽬标是给应⽤程序提供⼀种错觉，即我们能提供⾜够 的内存将整个数据库存⼊内存中



![image-20210508144838357](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210508144838357.png)

> 在这里，我们看一下数据库系统的高级层面，并在后续的内容中主键填充这个图
>
> * 在最底层是`磁盘`,磁盘存储着`数据文件`，并把它们封装成不同的块<block>或者页面<page>来表示它们
> * 在`内存`中，我们有`buffer pool`<`buffer缓冲池`>，执行引擎向buffer pool进行请求读取某一页的文件，然后buffer pool将其装载进来，然后执行引擎`Execution Engine`对buffer pool进行操作。 
>
> 执行引擎或查询引擎会去向我们的buffer 缓存池进⾏请求，并表示，想要读取page 2的内容。但page 2并不在内存中，因此，我们要去磁盘上的page⽬录(`page directory`)中进⾏查找，在page目录维护了一个page列表，根据这个列表就可以找到page2的地址，就将其放到内存，然后将其起提交给我们的执行引擎。
>
> 在这个架构与其他软件架构的不同之处就在于内存由DBMS自己来管理而不是交给操作系统来管理。

**可以看到，上面是思想其实就是`虚拟化`，`虚拟内存`，但是，`操作系统`已经给我们虚拟化了，为什么数据库中还需要一个虚拟化？**

![image-20211010091045264](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20211010091045264.png)

为什么不使用 OS 自带的磁盘管理模块？

OS 为开发者提供了如 mmap 这样的系统调用，使开发者能够依赖 OS 自动管理数据在内外存之间的移动，那么 DBMS 为什么要重复造这样的轮子？主要原因在于，OS 的磁盘管理模块并没有、也不可能会有 DBMS 中的领域知识，因此 DBMS 比 OS 拥有更多、更充分的知识来**决定数据移动的时机和数量**，具体包括：

* 将 dirty pages 按正确地顺序写到磁盘
* 根据具体情况预获取数据
* 定制化缓存置换（buffer replacement）策略



> 由于操作系统只能看到内存在不断的刷新，它不知道数据库在做什么，所以操作系统可能会频繁的读盘（因为它不知道哪些数据是热点，是内存满时哪些数据不应被优先回收的）这也就导致了性能瓶颈。所以为了提升性能我们明显需要更加细粒度的管理内存，这也就是为什么我们需要buffer pool。





![image-20210509000913115](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509000913115.png)

我们今天的重点就是数据库存储这块内容，此处我们必须关⼼的问题主要有两个：

* 第一个问题是是我们如何表示磁盘上文件中的数据
* 第⼆个问题则是，我们实际该如何管理内存以及在硬盘间来回移动数据

## 问题一:如何用磁盘上的文件来表示数据库

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20211010092015993.png" alt="image-20211010092015993" style="zoom:50%;" />

### 一、File Storage(文件存储) 

我们首先讨论如何在一系列页（pages）上组织数据库

![image-20210509083358241](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509083358241.png)

数据库系统说白了就是一堆文件，这些数据⽂件的格式通常情况下都是专属于某个数据库管理系统的





![image-20210509083447539](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509083447539.png)

`Storage Manager:`——`存储管理器`(`存储引擎`)

我们所要构建的东⻄，本质上来讲，它被称为存储管理器，有时也被称为存储引擎，它是我们的数据库系统中的⼀个组件，它负责维护我们在磁盘上的数据库⽂件



`存储管理器`会将所有的文件组织成一个个`page`（一个page有`固定的空间大小`）的集合，So，我们的存储管理器将跟踪我们要在这些page上所执行的所有读取和写入操作(这就像是我们在跟踪我们的⻚（pages）中还有多少空间可允许我们往⾥⾯存储新的数据)

`page的目的在于，将经常会一起用到的数据放在同一个page（比如一个表的所有元组放在同一个page），这样就可以做到以最少的读取磁盘的次数来实现我们的操作` 



![image-20210509083837161](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509083837161.png)

1. 一个Page是一个固定大小的数据块。
   	→ 它可以包含元组、元数据、索引、日志记录...。
   	→ 大多数系统不混合页面类型(元组的就只写元组，索引的只写索引)。
   	→ 有些系统要求一个Page是自解释(self-contained)<即解释这个page的信息就存在这个page中>的。
2. 每个页面都有一个唯一的标识符（PAGE ID）。
   	→ DBMS使用一个中间层来将PAGE ID映射到磁盘位置 (是记录一个文件的某个位置，其实 就是记录⼀个相对位置，⽅便⽂件整体移动后，只要知道整体⽂件的初始位置，我依然可以通过 该相对位置即page ID找到某个⽂件某个位置的数据所对应的page，要知道⼀个⻚得到⼤⼩是固定的，id数*⻚⼤⼩就找到了offset值)



![image-20210509084425588](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509084425588.png)

`硬件(磁盘)中的页：`

一般都是`4KB`，这4KB的读写是有`原子性`的，即读的时候也是读全部的4KB，写的时候也是写全部的4KB，不会存在部分写了而部分没写的情况

`os中的页:`

一般也是`4KB`

`数据库中的页：`

由于数据库中的页是依赖磁盘的页，所以其实是做了一层抽象，一般都选择`512B~16KB`

> 最底层的硬件页(hardware page)的大小是4KB，但是像MySQL它的页是16KB，这也就导致了这种情况的发生：如果要写16KB的MySQL页(也就是对磁盘进行write和flush操作)，但是硬件页只能保证4KB的写入是原子性的，假如写入8KB以后被意外中断了，恢复后后再写入8KB此时得到的就是两份不连续数据。我们会在之后讨论⽇志和并发的时候，再去讨论这个问题





![image-20210509084752520](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509084752520.png)

不同的DBMS以不同的方式管理磁盘上的文件页，Page Storage的设计大致有下面三种，依次介绍.



#### Heap File Organization

最常⻅的⽅式是使⽤Heap File Organization，但有件事情要理解的是，在存储管理器最底层的级别中，我们不⽤关⼼我们的page中到底有什么，我们不介意⾥⾯放的是索引数据还是tuple数据或是其他，我们就可以对这个page进⾏read或delete操作。

So，数据库中的heap⽂件是⼀个**⽆序的page集合**，即可以以随机的顺序把tuple数据保存在⾥⾯。要说⼀下，关系模型并没有任何排序，如果我⼀个接⼀个地插⼊tuple，我并不能保证它们是按照我插⼊的顺序保存在磁盘上的，这并没有什么关系，因为在我写的SQL查询中并没有排序的概念。

我们会使⽤⼀些额外的元数据来跟踪我们有哪些page，哪些page中是空余空间，这样如果我们需要插⼊新数据时，我们就知道可以在哪个page上插⼊数据了。

![image-20210509085144533](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509085144533.png)

heap file 指的是一个无序的 pages 集合，pages 管理模块需要记录哪些 pages 已经被使用，而哪些 pages 尚未被使用。那么具体如何来记录和管理呢？主要有以下两种方法 Linked List 和 Page Directory。

##### Linked List（双向链表）

⾸先我们来谈论下链表这种⽅式，这是⼀种愚蠢的⽅式，实际上也没⼈使⽤这种⽅式，之后，我们会去了解page⽬录这种⽅式，它是⼀种更好的⽅案。

![image-20210509085348132](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509085348132.png)

在文件的Header保持一个header page，存储两个指针：

* 空闲页列表的header page (还有空闲)
* 数据页列表的header page (满了)

每个Page都记录了本身的空闲slot的数量。



当需要空间时，空Page被分配并append到Free Page List部分，当空闲的数据页变满时，它们被从Free Page List移到Data Page List

##### Page Directory(字典)

⼈们通常的做法是使⽤⼀个page⽬录，用一个bitmap来追踪当前文件中的每个page是free/occupied，此方案最大的问题是需要保证内存中该page和磁盘上该page的同步。

![image-20210509085627087](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509085627087.png)

我们的`Directory`目录中，**维护了`pageID`和他们所在的`物理位置`的映射关系**，另外，我们也可以在`Directory`维护某些元数据，比如这个页的空闲空间大小，这样，我们就不必真的去扫描，在目录中就可以判断该page是否有空闲空间



我们需要保证`directory`和`page data`数据同步，需要保证数据库崩溃了也 能重建数据库 (因为directory中有一些元数据，比如这个page的空闲空间，所以在写directory和Data Page的时候是没有原子性的)



原理和`hash table`很像，假设想要 page x,就可以从Page Directory中找到.







### 二、Page Layout(页内布局)

——`Page布局`

每个页面都包含一个header，用于记录有关页面内容的元数据：

![image-20210509091530517](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509091530517.png)

每个Page都包含一个描述元数据信息的HEADER。

header 中通常包含以下信息：

* Page Size
* Checksum
* DBMS Version
* Transaction Visibility
* Compression Information

![image-20210509105222298](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509105222298.png)

在页面中布置数据有两种方法：（1）元组导向（2）日志导向。

#### 1. Tuple oriented(元组导向数据库)

——`元组存储`

前面已经讲过Page Directory了，比如想要Page2，Page目录会告诉我们物理地址，现在，我们来谈论下当我们在看page内部的时候，它⾥⾯是什么样的

![image-20211011110817777](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20211011110817777.png)

在这个例⼦中，假设我们保存的是tuple。最上面那个Num Tuples的数字记录了当前Page有多少个Tuple。每次插入的时候，由于每个Tuple的长度相同(做这个假设)，且知道了已有的Tuple数，所以就可以知道新插入的位置，所以只需要这里加一个Tuple，并让Num++，

这其实是⼀个糟糕的注意，有一个显著的缺点：如果你删除⼀个tuple，那你就得移动所有的tuple。或者不移动也行，释放掉的空间不再去用，但这样又会有外部碎片。或者我们不移动，且释放掉的空间，待下次插入的时候直接插入到那，但这样又有问题，如果Tuple的长度不同，想插入的位置可能没有足够的空间来存储新tuple，只能放到下面 （而且需要循序去扫描，找到一个空闲位置...显然不是好方案）。 

> 主要问题就是两个：
>
> 1. 一旦出现删除操作，每次插入就需要遍历一遍，寻找空位，否则就会出现空间浪费
> 2. 无法处理变长的数据记录（tuple）



下面介绍实际中使用的方案——slotted pages。各个数据库的具体细节可能有不同，但high level上都是采用这个Slotted Pages的策略

![image-20210509094731391](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509094731391.png)

`header`⾥⾯能够保存基本的元数据，例如checksum或者访问时间之类的东⻄

接着，我们必须有能够保存数据的区域。在这里，在顶部我们有⼀个称为slot数组的东⻄,底部的空间则是我们⽤来保存我们想保存的数据。

**这⾥我们所保存的tuple可以是固定⻓度或者是可变⻓度的**

我们用一个`Slot Array`来记录每个 slot 的信息，如`起始位置，大小`等，这是一个类似hashtable的map关系，如下图，将⼀个特定的slot映射到page上的某个偏移量上，根据这个偏移量，你能找到你想要的那个tuple。

⼀条记录的位置是由page id和slot number来⼀起确定的

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509094849503.png" alt="image-20210509094849503" style="zoom:50%;" />

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509094927024.png" alt="image-20210509094927024" style="zoom:50%;" />

**Slotted Pages**:Page maps slots to offsets.

* 

> `Slot Array`向右增长，Tuple向左增长，当再次插入时会重叠时，那么就表示`满了`.
>
> `Slot Array`记录的Tuple位置是相对位置，是相对于这个Page起始位置的一个相对位置

![image-20210509104635217](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509104635217.png)

<u>`我们根据page_id和slot_id来查找一个tuple`</u>，也就是说，每个tuple对应了一个(page_id,slot_id)的2元组.



> 每个page被分成三个部分：Header、Slot Array、Tuple。
>
> 1. Header用于保存基本元数据，比如checksum或者访问时间等
>
> 2. Slot Array的作用是将一个特定的slot映射到page上的某个偏移量上，根据这个偏移量来找到所需的tuple。
>
> 
>
> * 填充Page的方式是从前往后对Slot Array进行填充，数据则是从后向前进行填充。
>
> * 所以对于上层应用而言，如果想找到一个tuple只需要page id和slot id，然后通过每个page里的Slot Array就可以定位到唯一的tuple了。这样的好处在于无论我是移动了整个Page还是移动了Page内的tuple，我们都无需更新索引或者其他一些东西。



##### 示例

https://cakebytheoceanluo.github.io/2020/03/11/DBMS-PostgreSQL-PageLayout/

`1.准备`

```sql
testdb=# drop table if exists r;
DROP TABLE
testdb=# create table r (id int primary key , val varchar(6));
CREATE TABLE
testdb=# insert into r values (101, 'aaa'), (102, 'bbb'), (103, 'ccc');
INSERT 0 3
testdb=# select * from r;
 id  | val 
-----+-----
 101 | aaa
 102 | bbb
 103 | ccc
(3 rows)
```

我们创建了一个表r(id,var)，并插入了三个数据





`2.ctid` — PostgreSQL 中的 RID

- `ctid` 就是 PostgreSQL 中的 Record ID, 是一个来自` page id 和 offset 的 pair`

```sql
testdb=# select r.ctid, r.* from r;
 ctid  | id  | val 
-------+-----+-----
 (0,1) | 101 | aaa
 (0,2) | 102 | bbb
 (0,3) | 103 | ccc
(3 rows)
testdb=# select r.ctid, r.* from r where ctid = '(0, 1)';
 ctid  | id  | val 
-------+-----+-----
 (0,1) | 101 | aaa
(1 row)
```

我们可以看到:

- page id 均为 0, 说明所有的 tuple 都在同一个 page 上

- `insert` 顺序插入 tuple 到 page 中，即第一个 insert 的 tuple 在 page 的第一个 offset

  

`3.delete` 后的行为

```sql
testdb=# delete from r where id = 102;
DELETE 1
testdb=# select r.ctid, r.* from r;
 ctid  | id  | val 
-------+-----+-----
 (0,1) | 101 | aaa
 (0,3) | 103 | ccc
(2 rows)
```

我们将中间的 `102`tuple 删除:

- 可以看到对应的 page layout 除了了 `102`tuple 以外，没有其他区别

- 这说明 PostgreSQL 没有在 delete 后马上 compact page, 删除的位置空着，而不是去移动其他的 tuple 去占有空出来的位置。——效率高

  

`4.delete` 后的 `insert` 行为

```mysql
testdb=# insert into r values (104, 'xxx');
INSERT 0 1
testdb=# select r.ctid, r.* from r;
 ctid  | id  | val 
-------+-----+-----
 (0,1) | 101 | aaa
 (0,3) | 103 | ccc
 (0,4) | 104 | xxx
(3 rows)
```

我们在删除 `102` 后，插入 `104`:

- 我们看到 `104`tuple 的位置的 offset4, 即之前最后一个 tuple 之后
- 说明 PostgreSQL 在删除后的插入，`不会重新利用空出来的空间`。



- 我们看到了上面 `delete后的行为`和 `delete后的insert行为` , 可以猜测，PostgreSQL 为了性能，并没有让 `delete` 和 `insert` 去干预 page layout





`5.vacuum full;` — 整合空间(可以看作是数据库的垃圾回收器,vaccum会去遍历page去整理page)

```sql
testdb=# vacuum full;
VACUUM
testdb=# select r.ctid, r.* from r;
 ctid  | id  | val 
-------+-----+-----
 (0,1) | 101 | aaa
 (0,2) | 103 | ccc
 (0,3) | 104 | xxx
(3 rows)
```

- 可以看到，`vacuum`将空间按顺序重新排放了，消除原本的碎片
- `vacuum full;` 可以显式 (explicit) 去 compact page，也就是占有空着的空间
- 这个指令之后，后面的 tuple 向前移动，占据了原 `102` 有的空间





`6.SQL Script`

```mysql
drop table if exists r;

create table r (id int primary key , val varchar(6));

insert into r values (101, 'aaa'), (102, 'bbb'), (103, 'ccc');

select * from r;

-- ctid := a pair of page id and offset
select r.ctid, r.* from r;

-- tuple 102 delete, but the page is not compacted after deletion
delete from r where id = 102;

select r.ctid, r.* from r;

insert into r values (104, 'xxx');

-- in PostgreSQL:
-- insert after after the last inserted offset
-- ignore the free space after the previous deletion
select r.ctid, r.* from r;

-- like GC (garbegac collection)
-- compact(reorganize) the pages
-- takes time
vacuum full;

select r.ctid, r.* from r;
```





上面是在`PostgreSQL`中的示例，下面看看`SQL Server`



`插入数据:`

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509102427863.png" alt="image-20210509102427863" style="zoom:50%;" />



`查询:`

![image-20210509102508934](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509102508934.png)

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509102519403.png" alt="image-20210509102519403" style="zoom:50%;" />



`删除一个tuple102`

![image-20210509102831632](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509102831632.png)

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509102843374.png" alt="image-20210509102843374" style="zoom:50%;" />

可以看到，删除时不会移动别的Slot



`插入一个tuple104`

![image-20210509103015033](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509103015033.png)

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509103024648.png" alt="image-20210509103024648" style="zoom:50%;" />

可以看到，在这里将Slot插入到之前删除掉的位置1

> `Posg..`，插入时不会在之前删除的地方插入，只会在末尾插入
>
> `SQL SERVER:`,插入时在之前删除的地方插入，所以空间利用率高，更紧凑







再来看看`Oracle SQL`

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509103426456.png" alt="image-20210509103426456" style="zoom:50%;" />

![image-20210509103513863](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509103513863.png)



`删除tuple102:`

![image-20210509103609153](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509103609153.png)



`插入tuple104:`

![image-20210509103703364](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509103703364.png)

新插入的是PLOT3，这个Postgres的结果一样，只会将空的slot留在那不管他了







#### 2. Log-structured(日志导向数据库)

——`日志存储`

Log-structured：相对于存放tuple，log-sturctured 的组织方式存放的用户用户操作记录，如下图所示：每次操作往 page 插入一条新的 entry，插入新 tuple 的时候就在该 entry 存放 tuple，删除 tuple 或者更新 tuple 就查找该 tuple 的位置标记为已删除/更新参数，这种方法优点是便于回滚以及写入速度快，缺点是查找某个 tuple 的时候需要从后往前扫描来重新生成需要的 tuple，优化方法是可以生成 indexes 来定位tuple在 log 的位置，以及周期性的压缩记录（把更新和删除反映到 该 tuple 创建的 log 上，减少存放的 log 数量）

![image-20210509104812821](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509104812821.png)

Log-Structured：DBMS不存储元组，而是存储日志记录。
 •将记录存储到文件中，了解数据库的修改方式（插入，更新，删除）。
 •为了读取一条记录，DBMS向后扫描日志并 "重新创建 "元组，以找到它所需要的东西
 •快速写入，读取速度慢。

> log-structured file organization 不是将所有的tuple都放在Page中，而是去存储tuple的创建和修改信息。这样做的好处很多比如写入很快，方便回滚等，HDFS和S3就是以这种方式来组织Page。







![image-20210515200541436](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210515200541436.png)

* 但是如果要读取那就只能从最新的记录开始上推来找到目标tuple



![image-20210509105556698](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509105556698.png)

* 为了提升读取速度可以给每个tuple建立索引（这里的索引是，传入一个ID，就可以跳到日志的特定偏移处，找到想要的数据）



![image-20210509105649550](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509105649550.png)

另⼀件事你可以做下，即重新把这个log⾛⼀遍，定期对它⾥⾯内容进⾏筛选压缩(`compact`)，对于每个tuple， 你只需知道⼀条记录即可。所以我们就可以给它转换成这种tuple形式



>**为了解决读取慢的问题**:
>
>1. 构建索引以允许它跳转到日志中的位置。
>2. 定期压缩日志。
>
>下面是使用日志导向的数据库:
>
>![image-20210509154452781](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509154452781.png)

`分布式数据库`往往采用这种~



2种日志压缩方式:

![image-20210509154522156](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509154522156.png)



`Log-Structured`:Instead of storing tuples, the DBMS only stores log records.

* Stores records to file of how the database was modified (insert, update, deletes).

  将数据库如何被修改（插入、更新、删除）的记录存入文件。

* To read a record, the DBMS scans the log file backwards and “recreates” the tuple.

  要读取一条记录，DBMS会逆向扫描日志文件并 "重新创建 "元组。

* Fast writes, potentially slow reads.

  写的快，读的可能慢。

* Works well on append-only storage because the DBMS cannot go back and update the data.

* To avoid long reads the DBMS can have indexes to allow it to jump to specific locations in the log. Itcan also periodically compact the log (if it had a tuple and then made an update to it, it could compactit down to just inserting the updated tuple).  The issue with compaction is the DBMS ends up withwrite amplification (it re-writs the same data over and over again).

  为了避免长时间的读取，DBMS可以有索引，允许它跳到日志中的特定位置。它还可以定期压缩日志（如果它有一个元组，然后对它进行了更新，它可以压缩到只插入更新的元组）。 压实的问题是DBMS最终会出现写入放大的情况（它反复地重新写入相同的数据）。



### 三、Tuple Layout(元组布局)

![image-20210509110041808](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509110041808.png)



![image-20210509110324357](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509110324357.png)

![image-20210509110401939](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509110401939.png)

![image-20210509110425049](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509110425049.png)

![image-20210509110440754](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509110440754.png)





![image-20210509110638013](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509110638013.png)

![image-20210509110646885](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509110646885.png)

![image-20210509110702513](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509110702513.png)





tuple其实就是一个字节序列，由更顶层的信息来知道这个字节序列的含义.

`1.Tuple Header`:维护 meta-data about the tuple.

* Visibility information for the DBMS’s concurrency control protocol (i.e., information about whichtransaction created/modified that tuple).

  DBMS的并发控制协议的可见性信息（即，关于哪个事务创建/修改了该元组的信息）。

* Bit Map for NULL values.

  有一个NULL数据的bit map

* Note that the DBMS does not need to store meta-data about the schema of the database here

  注意，DBMS不需要在这里存储关于数据库模式的元数据。

`2.Tuple Data`:Actual data for attributes.

* Attributes are typically stored in the order that you specify them when you create the table.

  属性通常按照你创建表时指定的顺序存储。

* Most DBMSs do not allow a tuple to exceed the size of a page

  大多数DBMS不允许一个元组超过一个页面的大小

`3.Unique Identifier:`

* Each tuple in the database is assigned a unique identifier.

  数据库中的每个元组都被分配一个唯一的标识符。

* Most common:pageid+ (offsetorslot).

  最常见的是：pageid+（offset or slot)

* An applicationcannotrely on these ids to mean anything

  一个应用程序不能依赖这些标识来表示任何东西 (只是数据库内部的信息而已)

### 四、总结

![image-20210509161451298](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210509161451298.png)

> 这节课中，我们学到了数据库是以page为基本的单元来组织的。
>
> 这节课中，我们分三个层次，从high level 逐渐讲到 low level:
>
> 1. 我们知道以Page为一个重要的单元来管理数据库文件。我们上面讲了Heap File Organization的组织形式，这里我们常用Page Directory，来维护PageID到Page物理地址的映射
> 2. 知道各个Page是怎么组织在一起之后，我们讲了Page内的布局，这里我们常用Slotted Pages。将一个Page分成Header + Slot Array + Data，Slot Array从前往后长，Data从后往前长，碰撞时表示存满了.

