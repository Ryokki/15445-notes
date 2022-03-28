今天我们想去讲的内容就是，我们该如何将磁盘中的数据库⽂件或page放到内存中，以便我们 可以对它们进⾏操作，要记住数据库系统⽆法直接在磁盘上进⾏操作，我们没办法在不将它们先放⼊内存的情况下对这些数据进⾏读写，这是冯诺伊曼架构

这节两课，我们会学习数据库的缓存区管理，这部分基于前面两节课 (Storage I 和 Storage II)。

缓存区 buffer pool 就是在内存中对 page 的缓存 (cache)。



![image-20210511082454454](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511082454454.png)

**目标：**

这节课主要要解决的一个问题是：DBMS(database mangement system) 如何管理它的内存 (memory) 使用以及数据在磁盘( disk ) 之间的来回转移，主要从以下两个方面来体现：

* **空间控制(Spatial Control)**：空间控制策略通过决定将 pages 写到磁盘的哪个位置，使得常常一起使用的 pages 能离得更近，从而提高 I/O 效率。
* **时间控制(Temporal Control)**：时间控制策略通过决定何时将 pages 读入内存，将page写回磁盘，使得读写的次数最小，从而提高 I/O 效率。

big-pictrue如下：

![image-20210511082633144](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511082633144.png)

在这里，BufferPool的作用就像是用于缓存page的一个cache。之前的课程已经介绍了关于 disk 的部分，也就是数据库如何在 disk 里面进行存储的，接下来要讲的是 memory 部分，主要是缓存池的管理已经数据的写入和写出。



# 1 Buffer Pool Manager

## 1.1 Buffer Pool Organization

首先，看一下缓存池的*组织结构*。在内存的区域空间被分成一系列固定尺寸的 page 的数组，每个数组参数被称为一帧 (frame)。每当 DBMS 请求一个 page 的时候，缓存池会从 disk 里面复制（不做任何修改）一个 page 进入一个 frame 当中，如下图所示：

![image-20210511083112282](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511083112282.png)

BufferPool其实就是在内存中开辟的一个空间，用于缓存经常要用的Disk-page，重要的一点是，这个内存是由数据库系统来管理的，而不是统统交给os来分配，我们是使用malloc自己来分配的.

BufferPool的一个`frame`就对应着磁盘的一个`page`

当DBMS需要一个page时，我们会将这些page从磁盘复制到frame，并从frame取出来用.

BufferPool是一个**全相联**的——即一个frame可以存放任何一个page

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511083518219.png" alt="image-20210511083518219" style="zoom:40%;" />

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511083532031.png" alt="image-20210511083532031" style="zoom:40%;" />







![image-20210511083708405](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511083708405.png)

缓存池除了 frame 之外，还需要维护关于内存里存在的 page 的元数据（`meta-data`），主要通过 `page-table` 来管理维护。 page-table 原理上是一个**`哈希表`**，它⽤来跟踪我们在内存中有哪些page。其实就是**维护一个PAGE-ID到FRAME-LOCATION的映射**，如果我们想找⼀个特定的page，通过page表和page id，我们就可以知道这个page在哪个 frame中。

除此之外，对于每个 page 来说，还需要有一些额外的变量（元数据）来维护其他信息，如

1. Dirty Flag (当前page从读入之后是否被修改过)

   如果被修改，从Frame被撤出时，需要写回到磁盘

2. Pin/Reference Counter （当前page 是不是正在被引用/引用计数）

   它⽤来跟踪想要使⽤(读或写都算)该page的当前线程数量，在访问这个Page之前就先增加计数。如果当前有线程正在使用这个page(引用计数>0)，那么这意味着我们并不想将该page从内存驱逐(`evict`)出去。  我们只允许将此数字为 0 的 page 写回硬盘，即当前没有任何 thread 在使用此 page。（Pin，这个page正在被读，不能被swap out）

   如下图，当我们正在使用Page3的时候，我们可以将Page 固定(PIN)住，表示我不想让这个page从buffer池中移除掉

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511084729173.png" alt="image-20210511084729173" style="zoom:50%;" />



如下图，当我们想要用一个不在Buffer Pool中的Page，比如Page2，我们需要在Page Table的这个位置上加一个锁(`Latch`)，然后就可以去读取这个Page到Buffer Pool，并更新Page Table去指向Buffer Pool的这个位置

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511084745537.png" alt="image-20210511084745537" style="zoom:50%;" />

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511092033392.png" alt="image-20210511092033392" style="zoom:50%;" />

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511092047257.png" alt="image-20210511092047257" style="zoom:50%;" />

当被请求的 page 不在 page table 中时，DBMS 会先申请一个 latch（lock 的别名），表示该 entry 被占用，然后从 disk 中读取相关 page 到 buffer pool，释放 latch



## 1.2 Locks vs. Latches

![image-20210511090758243](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511090758243.png)



这里我来讲一下 lock 和 latch 的区别，这2个术语是在dbms里的特指。广义上他们是一个东西。但为了区分os的锁，和db的锁。所以在dbms里分为2个词汇。

* Locks: Locks 是一个更高层次的逻辑元语，它保护数据库的内容（如元组、表、数据库）不受其他事务的影响。事务将在整个持续时间内持有一个 Locks。数据库系统可以在运行查询时向用户披露哪些 Locks 正在被持有。Locks 需要能够回滚。概念上接近于操作系统中的 Latches。
* Latches: Latches 是一种底层的保护元语，DBMS 将其用于内部数据结构的关键部分（例如，哈希表，内存区域）。Latches 只在所进行的操作的时间内保持。Latches 不需要支持回滚。概念上接近与操作系统的 Mutex。

### 1.2.1 Lock

——是一个High Level的锁

•保护数据库逻辑内容（比如tuple，表）免受其他事务的影响
•事务会在运行时持有这个lock，直到事务结束
•需要支持回滚
•保护元组（行），表，索引

### 1.2.2 Latch

——是一个底层实现的锁

•保护DBMS内部数据结构的关键部分不受其他线程的影响
•持有直到一个操作结束
•不需要支持回滚



> - locks 是一个 high level 很抽象的概念，它直接和 transaction 事务相关，应用于事务的 lock protocol 中。是应用层面的。是逻辑内容的互斥，比如行，表，事务。
> - latch 基本上和 low level 实现相关，指 `std::mutex` 这类和代码相关的类 (class)，来保护代码中的 critical section。是应用不可见的，内部数据的互斥。



## 1.3 Page Table vs. Page Directory

看一下页目录 (page directory) 和页表 (page table) 的区别：

![image-20210511090923364](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511090923364.png)

* Page Table：page id map to `the page in buffer pool frames` (pageid 映射到 buffer pool frame的位置)

* Page Directory：page id map to `location in database files` （pageid映射到 磁盘文件的位置）

  

  > 1. 对Page Directory的所有改变都需要写回到磁盘，因为如果系统崩溃了，我们需要依靠我们的Page Dicrectory来找回我们的数据库
  > 2. 对Page Table的改变不需要写回到磁盘，因为系统崩溃了是不需要PageTable来做任何事，所以就直接一直驻留在内存中即可（所以我们可以用hashtable或者hashmap来实现）（我们只需要保证线程安全，不用保证持久化）

  



## 1.4 Allocation Policies (分配策略)

现在我们要开始讨论该如何为我们数据库中的Buffer池分配内存了,关于内存分配策略，有两种思路：

*Global Policies* （全局策略）：我们所试着做出的决策能够使整个我们要试着执⾏的worklaod都受益。我们会去查看所有运⾏在该系统上的查询和事务，为了全局的性能来试图选择该内容是否应该存储在内存中

*Local Policies* （局部策略）：针对每个单个查询或者单个事务来进⾏，尝试做出可以让我的⼀个查询，⼀个事务进⾏得更快的最佳⽅法。即使对于整个系统⽽⾔，这实际上可能是个糟糕的想法

![image-20210511092635527](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511092635527.png)

- global policies: 指所有 query, transaction 使用同一个 buffer pool 的 replacement policy
- local policies: 指每一个 query, transaction 可以有一个自己的 buffer pool， 和自己的 replacement policy。同时可以在一个共有的 buffer pool 去 share pages
- 实际上会混合这两种主意， 比如数据 table 拥有一个 buffer pool, index (索引数据结构) 拥有另一个 buffer pool。这个我们会在 index 的课上提到。

> Project1使用global策略更好，因为它只需要找到最近最少使用的page，将其移除，即便对于特定的query它会很糟糕





# 2. Buffer Pool Optimizations 优化

![image-20210511093023738](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511093023738.png)

下面是一些优化策略：

* 多Buffer池

* 预取

* 扫描共享

* 缓冲池Bypass

  

## 2.1 Multiple Buffer Pools 多缓冲池

见前面提到的 Local Policies。如果一个数据库只有一个bufferPool，因为所有和磁盘间的数据交换都要通过他，很容易争抢.

解决的方式：我们可以用多个bufferPool，按不同的用途，维度区分开.

![image-20210511093249374](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511093249374.png)

比如，可以获得另⼀个很⼤的优点，减少那些试图访问Buffer池的不同线程间争抢 Latch的情况发⽣。减少Latch的竞争从而提高性能



![image-20210511094548678](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511094548678.png)

多个BufferPool时，如何知道查这个page需要去哪个BufferPool去找？

* 策略1-Object Id：可以在每个Page里面额外加一个 object id 来以此将其映射到某个缓存池中
* 策略2-Hashing：将Page ID 哈希到一个BufferPool，比如可以用PageID对缓冲池数取模，如下图

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511095852279.png" alt="image-20210511095852279" style="zoom:50%;" />

这样就可以知道去哪个Buffer Pool去找数据.



## 2.2 Pre-Fetching 预取

我们可以预加载一些 page, 一个 thread 在对 page A 进行扫描计算的同时，另外一个 thread 提前可以将下一个 page B 加载到 buffer pool 中。这个方法，是用猜测或者预先的只是，去加载未来可能用得到的 page, 最终目的减少从硬盘加载的时间。具体有两种方法：

- Sequential Scans: 顺序扫描即物理上连续存储的下一个 page
- Index Scans: 索引扫描

### 2.2.1 Sequential Scans 顺序扫描

一开始，BufferPool是空的，比如我们这是要读取page zero，我们会去磁盘取数据，如下图。下图中，这个箭头是`游标`<cursor>——记录上次去disk查的page时离开的位置，根据这个游标，我们可以来预取啦！

![image-20210511100145496](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511100145496.png)



此时，我们还想要page1，如下图：

![image-20210511100827073](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511100827073.png)



此时，数据库系统看到你连续扫描了两个page，这是一个sequenial scan，数据库可能会意识到，你可能想去扫描整个表，所以他会继续扫描下去，把page2，page3也拿过来。这样的话，Q1 结束对 page1 的扫描，page2,3 就已经在 buffer pool，之后就可以直接用，不用再去磁盘读：

![image-20210511100836764](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511100836764.png)

![image-20210511100844759](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511100844759.png)



...我们会一直这样下去，每次读IO时，都预读取下面几个page.

![image-20210511101353925](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511101353925.png)



* 预取的高效的原因：还是局部性！我们磁盘空间上紧邻的物理地址，往往是存着同个表这样的，所以经常会在这个物理地址局部读取若干的page.

* 预取在实现起来很简单，用mmap即可。由于我们预取的是一段连续的磁盘空间，所以我们直接用mmap映射到这个磁盘区域



### 2.2.2 Index Scans 索引扫描

有一些查询，os不知道怎么去做，但在数据库系统中可以做到，因为我们知道这个查询想要的是什么,比如这个例子，我们要去查询A中的val从100~250的tuple.

```mysql
SELECT * FROM A
    WHERE val BETWEEN 100 AND 250
```

![image-20210511101822363](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511101822363.png)



这里的 index 是一个 B+ Tree， 具体作法是从根节点查找至叶子节点，再水平遍历有关的叶子节点。这时候我们需要的 page 不一定是物理上连续存储的。

我们给value做了一个索引，如下图：

![image-20210511102529757](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511102529757.png)



首先缓存池会顺序扫描 disk 上的页并读入到缓冲池中（page0, page1），当读取到 page1 的时候，通过 page1 我们知道 page3 的 val 是 100-199（二叉树存储）；我们也知道我们需要查询的的范围是 100 - 250，所以我们可以根据我们是通过二叉树存储从而通过 page3 定位到 page5，并进而把那两个页面读入进来，而不需要逐一访问 page2, page4。我们之所以能够这么做是因为 DBMS 知道其内部是怎么存储数据的也能通过对 query 操作的分析来获取需要哪些 page，这是通用操作系统做不到的。

示意如下：

![image-20210511103538249](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511103538249.png)

如上图，此时搜到了根节点，它决定要去获取⼤于或等于100的value，所以接下来会去搜左侧

![image-20210511103546272](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511103546272.png)

此时把page1搜到了，根据这个就知道100是在page1的右边，所以去转向右侧

![image-20210511103607598](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511103607598.png)

此时就在叶子节点搜到了page3和page5.



可以看到，我们从根index-page开始，然后查到1，1的位置是100，由于我们要大于100，所以我们去查page3，以及它的右边page5.

这个例子就是os做不了的，因为需要的是 page3 和 page5, 它们不是连续的，也不紧跟在 page1 (当前 page) 之后。 但是 index 这个数据结构告诉我们，下一步是 page3 和 page5 被需要，因此去 prefetch 它们俩。这里也是一个很明显的例子，操作系统在这一点帮不了我们，操作系统最多能帮我们 pre fetch 连续的 page，但是它不知道 index 的存在，没有办法帮助我们准确去得到 page3 和 page5：



## 2.3 Scan Sharing 扫描共享

简单的说，就是Query之间可以共享已经cache的page.即共享别人查询的结果，复用从磁盘中读到的数据，将该数据用于其他查询。

![image-20210511104025964](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104025964.png)

![image-20210511104355148](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104355148.png)

具体流程：

- 第一个 Query 已经在运行，有一些中间过程值
- 第二个 Query 开始运行，DBMS 发现这第二个 Query 可以公用第一个 Query 的值，而且它们俩需要扫描的范围一样。
- DBMS 将第二个 Query attch 进第一个 Query, 记录下第二个 Query 被 attach 的地方，让它们俩一起扫描。
- 等它们俩共同扫描结束后，再扫描一开始第二个 Query 跳过的地方。



例子：

Q1，查询A表中val的总和，所以cursor开始移动，去扫描每个page

![image-20210511104554160](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104554160.png)



![image-20210511104601227](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104601227.png)



读完page2后，BufferPool满了，我们采取某种策略去替换一个BufferPool（这里，page0是最早访问的，所以我们将其替换为3）：

![image-20210511104615732](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104615732.png)



在0替换为3之后，假设来了新的查询，去查询A中的所有val的平均值，在没有扫描共享的情况下，我们就从头开始扫描。由于这里page0都已经被扔掉了，那又要从磁盘中去读。在扫描共享的情况下，如下图，我们的page3~page5会扫描共享，Q1和Q2会一起扫描

![image-20210511104622965](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104622965.png)

![image-20210511104631058](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104631058.png)

![image-20210511104641661](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104641661.png)



此时Q1结束了，Q2就继续把没查过的page0~page2再查一下：

![image-20210511104649973](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104649973.png)

![image-20210511104657291](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104657291.png)

此时，Q2也查完了，大功告成！





看下面的例子，在扫描共享的情况下，我们的Q2是随便查询100个数据的平均值，所以扫描共享和非扫描共享的结果就不一样了，但根据关系模型，这是没问题的，因为数据库是无序的：

![image-20210511104704714](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511104704714.png)





## 2.4 Buffer Pool Bypass 旁路

sequantial scan 会将大部分 page 加载进 buffer pool, 而这些 page 又并不一定在未来会被重复利用。对这种 sequantial scan 的 Query，我们可以单独给 allocate 一块内存区域，而独立于且不影响 buffer pool。这一块专属的内存区域依然能保证对当前 Query 的性能，而且因为不会*打乱污染* buffer pool 而影响其他 Query 的性能。这块内存区域会在该 sequantial scan Query 结束后被释放。

![image-20210511110100791](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511110100791.png)

顺序扫描操作不将获取的页面存储在缓冲池中来避免开销。在后面不需要用到这些原始数据时，可以在内存中计算完成后就可以丢掉。缓冲池旁路也可用于临时数据（排序、连接）



## 2.5 OS Page Cache

OS在文件系统操作的时候，本身会有page cache，维护了文件系统的缓存

既然dbms在buffer pool已经自己管理了page cache，那么os的这封cache显的有些多余，所以大部分数据库都会用direct IO（直接内存io），来绕过OS的 page cache

不关掉os的page cache有个好处，比如db进程重启了，但这个时候os的page cache还在，可以避免冷启动 (主流的db只有PostgresSQL使用了os的page cache)

![image-20210511110931522](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511110931522.png)

- O_DIRECT: https://linux.die.net/man/2/open
- 大部分 DBMS **使用 direct I/O**，省去文件被加载入操作系统文件缓存区。direct I/O 可以直接讲文件读取到数据库缓存区的 address space。省去了从操作系统缓存区复制进数据库缓存区的 address space，这样性能也更好。更多见 [[CMU-15445\]Lec03 - Why not use the OS? - mmap](https://cakebytheoceanluo.github.io/2020/03/11/CMU-15445-Lec03/#mmap)
- 也有其他的 DBMS **不使用 direct I/O**， 比如 PostgreSQL。它们从工程的角度觉得不使用 direct I/O 更好：数据库 buffer pool 出现 page fault 的时候，如果对应的 page 在操作系统的文件缓存区中，那这时候只需要一次内存中的复制 (从操作系统的文件缓存区复制到数据库 buffer pool)，而不是去硬盘做漫长的 I/O。比如 PostgreSQL 重启后 buffer pool 是空的，但是操作系统的 page cache 以及还是有对应文件，这时候可以从 page cache 中复制，避免冷启动。





## 2.6 Buffer Replacement Policies

——**缓存替换策略**

这部分中的理论和操作系统中的页替换没有太大区别。

![image-20211012145428206](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20211012145428206.png)

当 DBMS 需要释放一个frame来为一个新的页面腾出空间时，它必须决定从缓冲池中驱逐(evict)哪个页面。 替换策略是 DBMS 实现的一种当它需要空间时决定从缓冲池中驱逐哪些页面的算法。

它的主要目标是：

* Correctness：操作过程中要保证脏数据同步到 disk
* Accuracy：尽量选择不常用的 pages 移除
* Speed：决策要迅速，每次移除 pages 都需要申请 latch，使用太久将使得并发度下降
* Meta-data overhead：决策所使用的元信息占用的量不能太大

* 

### 2.6.1 Lest Recently Used (LRU)

![image-20210511112337839](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511112337839.png)

最近最少使用的替换策略维护了**每个页面最后被访问的时间戳**。这个时间戳可以存储在一个单独的数据结构中，比如维护一个队列，这个队列按照时间戳排序，刚刚使用了一个page，就把它拿出来放到队尾，当要驱逐一个page的时候，就驱逐队头的那个即可。该 DBMS 会选择**驱逐具有最古老时间戳的页面**。此外，页面被保存在排序的顺序中，以减少排序驱逐的时间。





### 2.6.2 Clock

——时钟算法.

LRU 需要获取真的 timestamp，而无论获取 software timer 或者 hardware timer 都是非常昂贵的，需要很多 CPU 周期。

> emmm....不是可以用队列吗....，而不是真的去记录时间戳，为啥....，算了不管了...

所以我们选用 Clock, 它是 LRU 的一个近似, 它不需要对每个 page 额外维护一个时间戳，而是为每个page保存一个reference bit，具体实现方法如下：

* 对每个 page 有一个 reference bit ，记录是否被访问过（访问过之后设为 1）

* 将所有 pages 以一个环形来组织，如下图所示，在需要释放 frame 的时候，顺时针访问每个 frame：
  * 如果当前 page reference bit 为 1，将其设为 0
  * 如果当前为 0，就把他释放掉

![image-20210511112347931](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511112347931.png)

这里我们有4个page，他们标志位一开始都是0.



![image-20210511112355432](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511112355432.png)

这里访问了1，所以ref1=1



假设现在需要移除一个page，因为我们要的缓冲区满了，所以我们的时钟会从page1开始走起来：

![image-20210511112402733](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210511112402733.png)



看到page1是1，因此他最近被访问了，不应该移除，我们将它的ref设为0



![image-20210512100440169](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512100440169.png)

然后转向下一个page

![image-20210512100510750](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512100510750.png)

我们发现page2的ref是0，所以我们将其移除，并把page5拿进来.新来的page5并不算访问过，所以是0.



假设之后访问了page3和4，那么它们都标记为1.

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20211012151459790.png" alt="image-20211012151459790" style="zoom:50%;" />

然后我们继续向后遍历，把page3的ref和page4的ref都置为0.(这里由于本来就是0，所以没有效果)

> 此时，所有的ref都=0



然后我们下一次驱逐时，看到ref[3] = 1,ref[4]=1，ref[1] = 0，所以将1驱逐，如下图

![image-20210512101306015](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512101306015.png)

我们找到第一个0，即page1，将其移除.



总结来说，这个策略是：

有一个不断旋转的钟摆，钟摆不断移动，遇到1就改成0，然后继续移动，直到遇到0了，换页。



**然而，LRU 和 CLOCK 很容易受到顺序泛滥的影响，即缓冲池的内容由于顺序扫描而被破坏。由于顺序扫描会读取每一页，所以读取的页面的时间戳可能并不反映我们真正想要的页面。换句话说，最近使用的页面实际上可能是最不需要的页面。  所以，比较好的办法是LRU-K**

### 2.6.3 Problem - Sequential Flooding 顺序泛洪

![image-20210512101442151](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512101442151.png)

对于一个序列操作来说（扫描整个table），LRU 和 clock 反而可能会使效率下降，因为**大部分数据只会在扫描的时候被读入进来一次然后再也不会使用**，这样的话，**最经常使用的 page （最后一个被扫描进来的） 实际上可能是最不需要的 page**。

这可能会去污染我们的page buffer，可能会将我们接下来真的要使⽤的page从buffer pool中移除掉，

具体可以参考以下例子（在这个例⼦中，实际上我想移除的page是那些最近被使⽤的，⽽不是那些最近最少被使⽤的）：

![image-20211012155840078](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20211012155840078.png)

比如我们需要反复的全表扫描这7个Page，但是Frame只有6个。那么在第一轮1~6放满后，会把Page1换成Page7.然后第二轮的时候，Page1不在BUFFER中，把Page2换出去，Page2又不在BUFFER中....之后每次都要从磁盘去读。



> 有三种方法可以解决这个问题：
>
> * 一种解决方案是 LRU-K，它以时间戳的形式跟踪最后 K 个引用的历史，并计算出后续访问的间隔时间。这个历史记录被用来预测一个页面下次被访问的时间。
> * 另一个优化是局部化每个查询。通过持续跟踪被查询读取过的页面，DBMS 在每个事务/查询的基础上选择哪些页面要被驱逐。这使得每次查询对缓冲池的污染最小化。
> * 最后，优先级提示允许事务在查询执行过程中根据每个页面的上下文告诉缓冲池页面是否重要。

### 2.6.4 LRU-K

![image-20210512102351877](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512102351877.png)

LRU-K 中的 `K` 代表**最近使用的次数**，因此 LRU 可以认为是 LRU-1。LRU-K 的主要目的是为了解决 LRU 算法” 缓存污染” 的问题，其核心思想是将 “最近使用过 1 次” 的判断标准**扩展为 “最近使用过 K 次”**。

也就是说没有到达K次访问的数据并不会被缓存， 访问记录不能⽆限记录，当访问次数达到K次后，将数据索引从历史队列移到缓存队列中。

相比 LRU，LRU-K 需要多维护**一个队列：缓存队列。它用于记录所有缓存数据被访问的历史。只有当数据的访问次数达到 K 次的时候，才将数据放入缓存队列**。当需要淘汰数据时，LRU-K 会淘汰第 K 次访问时间距当前时间最大的数据。详细实现如下：

1. 数据第一次被访问，加入到访问**历史队列**
2. 如果数据在访问**历史队列**里后没有达到 K 次访问，则按照一定规则 (FIFO，LRU) 淘汰
3. 当访问**历史队列**中的数据访问次数达到 K 次后，将数据索引从**历史队列**删除，将数据移到**缓存队列**中，并缓存此数据，**缓存队列**重新按照时间排序
4. **缓存队列**中被再次访问后，重新排序
5. 需要淘汰数据时，淘汰**缓存队列**中排在末尾的数据，即淘汰” 倒数第 K 次访问离现在最久” 的数据

LRU-K 具有 LRU 的优点，同时能够避免 LRU 的缺点，实际应用中 `LRU-2` 是综合各种因素后最优的选择，LRU-3 或者更大的 K 值命中率会高，但适应性差，需要大量的数据访问才能将历史访问记录清除掉。

另外 LRU-K 对 sequential scan **不敏感**，因为 sequential scan 只使用每个 page 一次。而使用一次的 page 在 LRU-K 非常容易被 evict。

> LRU-K在工业界使用挺多的，性能很高





### 2.6.5 Localization

——本地化

![image-20210512102950346](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512102950346.png)

这部分已经在这课中的实验中体现了：PostgreSQL 给每一个 Query 一个独立的 buffer pool， 同时所有 Query 公用一个 shared buffer pool。这最大限度地减少了每个 query 对 buffer pool 的污染。





### 2.6.6 Priority Hints

——优先级提示：DBMS 知道哪些 page 比较重要，给他们打上标签。

![image-20210512103349232](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512103349232.png)

这个例子中，查询id++，这意味着我们趋向于沿着树的右侧往下走，去拿到这些page，因此，我们应该提示buffer管理器，我们表示这些page应该留在内存



![image-20210512103437576](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512103437576.png)



## 2.7 Dirty Pages

——如何处理Dirty Page

![image-20210512104047428](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512104047428.png)



FAST：丢弃缓冲池中任何不脏的页面
SLOW：将脏页写回磁盘

在快速驱逐与脏写页面之间进行权衡，我们应该驱逐将来不会再被读取的页面。





## 2.8 Background Writing



![image-20210512104325143](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512104325143.png)


DBMS 有一个执行定时任务的线程，可以定期遍历 page table 并将脏页写回硬盘。因为如果每次等 eviction 的时候再去 flush 脏页，会让 eviction 的过程非常的慢，所以一般会有个后台进程定期批量的去写回脏页。当安全地写会脏页之后，DBMS 可以 evict 驱逐该页面或者只是取消设置 dirty bit, 因为这个页面已经**不再脏了**。需要注意的是，在写日志记录之前，我们不会去写脏页





## 2.9 Other Memory Pool

![image-20210512105403398](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512105403398.png)



DBMS 需要内存来处理元组和索引以外的事情。这些其他的内存池可能并不总是由磁盘支持，这取决于实现。

* Sorting + Join Buffers
* Query Caches
* Maintenance Buffers
* Log Buffers
* Dictionary Caches

# 3. Conclusion

![image-20210512105655196](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512105655196.png)

数据库系统需要实现自己缓存层，而不是复用os的，因为他比操作系统知道的更多。
可以在分配和驱逐上做出更好的决定，同时可以实现pre fetching。



Project1：

![image-20210512105800860](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512105800860.png)

构建自己的bufferpool管理器以及替换策略



![image-20210512105910129](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512105910129.png)

任务1中，我们有类ClockReplacer，我们要实现今天的Clock策略，So，当读一个page或写一个page时，都得更新ClockReplacer的reference bit

* 需要注意的是，如果所有的page都被修改，那么选择frame_id最小的那个page移除





![image-20210512110145211](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512110145211.png)



![image-20210512110252913](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512110252913.png)



![image-20210512110643996](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512110643996.png)



![image-20210512110657917](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20210512110657917.png)