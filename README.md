# Java学习笔记

缓存

1、![v2-97181b7aaa681a5f816ec10921853393_1440w](https://user-images.githubusercontent.com/23097388/211450254-74b34358-d1b8-423e-abc4-dea91d03ea8a.jpg)
Redis 分布式集群的几种方案
1.1、主从复制
从服务器连接主服务器，发送SYNC命令； 主服务器接收到SYNC命名后，开始执行BGSAVE命令生成RDB文件并使用缓冲区记录此后执行的所有写命令； 主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令； 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照； 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令； 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；（从服务器初始化完成）主服务器每执行一个写命令就会向从服务器发送相同的写命令，从服务器接收并执行收到的写命令（从服务器初始化完成后的操作）

主从复制优缺点：

优点：

支持主从复制，主机会自动将数据同步到从机，可以进行读写分离为了分载Master的读操作压力，Slave服务器可以为客户端提供只读操作的服务，写服务仍然必须由Master来完成Slave同样可以接受其它Slaves的连接和同步请求，这样可以有效的分载Master的同步压力。Master Server是以非阻塞的方式为Slaves提供服务。所以在Master-Slave同步期间，客户端仍然可以提交查询或修改请求。Slave Server同样是以非阻塞的方式完成数据同步。在同步期间，如果有客户端提交查询请求，Redis则返回同步之前的数据

缺点：

Redis不具备自动容错和恢复功能，主机从机的宕机都会导致前端部分读写请求失败，需要等待机器重启或者手动切换前端的IP才能恢复。主机宕机，宕机前有部分数据未能及时同步到从机，切换IP后还会引入数据不一致的问题，降低了系统的可用性。Redis较难支持在线扩容，在集群容量达到上限时在线扩容会变得很复杂。

1.2、 哨兵模式
当主服务器中断服务后，可以将一个从服务器升级为主服务器，以便继续提供服务，但是这个过程需要人工手动来操作。 为此，Redis 2.8中提供了哨兵工具来实现自动化的系统监控和故障恢复功能。

哨兵的作用就是监控Redis系统的运行状况。它的功能包括以下两个。
（1）监控主服务器和从服务器是否正常运行。
（2）主服务器出现故障时自动将从服务器转换为主服务器。

哨兵的工作方式：

每个Sentinel（哨兵）进程以每秒钟一次的频率向整个集群中的Master主服务器，Slave从服务器以及其他Sentinel（哨兵）进程发送一个 PING 命令。如果一个实例（instance）距离最后一次有效回复 PING 命令的时间超过 down-after-milliseconds 选项所指定的值， 则这个实例会被 Sentinel（哨兵）进程标记为主观下线（SDOWN）如果一个Master主服务器被标记为主观下线（SDOWN），则正在监视这个Master主服务器的所有 Sentinel（哨兵）进程要以每秒一次的频率确认Master主服务器的确进入了主观下线状态当有足够数量的 Sentinel（哨兵）进程（大于等于配置文件指定的值）在指定的时间范围内确认Master主服务器进入了主观下线状态（SDOWN）， 则Master主服务器会被标记为客观下线（ODOWN）在一般情况下， 每个 Sentinel（哨兵）进程会以每 10 秒一次的频率向集群中的所有Master主服务器、Slave从服务器发送 INFO 命令。当Master主服务器被 Sentinel（哨兵）进程标记为客观下线（ODOWN）时，Sentinel（哨兵）进程向下线的 Master主服务器的所有 Slave从服务器发送 INFO 命令的频率会从 10 秒一次改为每秒一次。若没有足够数量的 Sentinel（哨兵）进程同意 Master主服务器下线， Master主服务器的客观下线状态就会被移除。若 Master主服务器重新向 Sentinel（哨兵）进程发送 PING 命令返回有效回复，Master主服务器的主观下线状态就会被移除。

哨兵模式的优缺点

优点：

哨兵模式是基于主从模式的，所有主从的优点，哨兵模式都具有。
主从可以自动切换，系统更健壮，可用性更高。

缺点：

Redis较难支持在线扩容，在集群容量达到上限时在线扩容会变得很复杂。

1.3、Redis-Cluster集群
redis的哨兵模式基本已经可以实现高可用，读写分离 ，但是在这种模式下每台redis服务器都存储相同的数据，很浪费内存，所以在redis3.0上加入了cluster模式，实现的redis的分布式存储，也就是说每台redis节点上存储不同的内容。

Redis-Cluster采用无中心结构,它的特点如下：

所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽。
节点的fail是通过集群中超过半数的节点检测失效时才生效。
客户端与redis节点直连,不需要中间代理层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可。

工作方式：

在redis的每一个节点上，都有这么两个东西，一个是插槽（slot），它的的取值范围是：0-16383。还有一个就是cluster，可以理解为是一个集群管理的插件。当我们的存取的key到达的时候，redis会根据crc16的算法得出一个结果，然后把结果对 16384 求余数，这样每个 key 都会对应一个编号在 0-16383 之间的哈希槽，通过这个值，去找到对应的插槽所对应的节点，然后直接自动跳转到这个对应的节点上进行存取操作。

为了保证高可用，redis-cluster集群引入了主从模式，一个主节点对应一个或者多个从节点，当主节点宕机的时候，就会启用从节点。当其它主节点ping一个主节点A时，如果半数以上的主节点与A通信超时，那么认为主节点A宕机了。如果主节点A和它的从节点A1都宕机了，那么该集群就无法再提供服务了。

2、Redis 和数据库同步、缓冲穿透、雪崩问题、hyperloglog slowqery 实现原理？
2.1、Redis数据库同步
捕捉所有mysql的修改，写入和删除事件，对redis进行操作

2.2、缓冲穿透
用户想要查询一个数据，发现redis内存数据库没有，也就是缓存没有命中，于是向持久层数据库查询。发现也没有，于是本次查询失败。当用户很多的时候，缓存都没有命中，于是都去请求了持久层数据库。这会给持久层数据库造成很大的压力，这时候就相当于出现了缓存穿透。

2.3、雪崩
缓存雪崩是指，缓存层出现了错误，不能正常工作了。于是所有的请求都会达到存储层，存储层的调用量会暴增，造成存储层也会挂掉的情况。

2.4、hyperloglog
优点：在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。（在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。）
缺点： HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

3、100W并发4G数据，10W并发400G数据，如何设计Redis存储方式？
淘汰策略
存储吃紧的一个重要原因在于每天会有很多新数据入库，所以及时清理数据尤为重要。主要方法就是发现和保留热数据淘汰冷数据。

网民的量级远远达不到几十亿的规模，id有一定的生命周期，会不断的变化。所以很大程度上我们存储的id实际上是无效的。而查询其实前端的逻辑就是广告曝光，跟人的行为有关，所以一个id在某个时间窗口的（可能是一个campaign，半个月、几个月）访问行为上会有一定的重复性。

数据初始化之前，我们先利用hbase将日志的id聚合去重，划定TTL的范围，一般是35天，这样可以砍掉近35天未出现的id。另外在Redis中设置过期时间是35天，当有访问并命中时，对key进行续命，延长过期时间，未在35天出现的自然淘汰。这样可以针对稳定cookie或id有效，实际证明，续命的方法对idfa和imei比较实用，长期积累可达到非常理想的命中。

减少膨胀
Hash表空间大小和Key的个数决定了冲突率（或者用负载因子衡量），再合理的范围内，key越多自然hash表空间越大，消耗的内存自然也会很大。再加上大量指针本身是长整型，所以内存存储的膨胀十分可观。先来谈谈如何把key的个数减少。

大家先来了解一种存储结构。我们期望将key1=>value1存储在redis中，那么可以按照如下过程去存储。先用固定长度的随机散列md5(key)值作为redis的key，我们称之为BucketId，而将key1=>value1存储在hashmap结构中，这样在查询的时候就可以让client按照上面的过程计算出散列，从而查询到value1。

过程变化简单描述为：get(key1) -> hget(md5(key1), key1) 从而得到value1。

如果我们通过预先计算，让很多key可以在BucketId空间里碰撞，那么可以认为一个BucketId下面挂了多个key。比如平均每个BucketId下面挂10个key，那么理论上我们将会减少超过90%的redis key的个数。

具体实现起来有一些麻烦，而且用这个方法之前你要想好容量规模。我们通常使用的md5是32位的hexString（16进制字符），它的空间是128bit，这个量级太大了，我们需要存储的是百亿级，大约是33bit，所以我们需要有一种机制计算出合适位数的散列，而且为了节约内存，我们需要利用全部字符类型（ASCII码在0~127之间）来填充，而不用HexString，这样Key的长度可以缩短到一半。

下面是具体的实现方式

复制代码
public static byte [] getBucketId(byte [] key, Integer bit) {

    MessageDigest mdInst = MessageDigest.getInstance("MD5");
    mdInst.update(key);
    byte [] md = mdInst.digest();
    byte [] r = new byte[(bit-1)/7 + 1];// 因为一个字节中只有7位能够表示成单字符
    int a = (int) Math.pow(2, bit%7)-2;
    md[r.length-1] = (byte) (md[r.length-1] & a);
    System.arraycopy(md, 0, r, 0, r.length);

    for(int i=0;i<r.length;i++) {
        if(r[i]<0) r[i] &= 127;
    }
        return r;
}
复制代码
 
参数bit决定了最终BucketId空间的大小，空间大小集合是2的整数幂次的离散值。这里解释一下为何一个字节中只有7位可用，是因为redis存储key时需要是ASCII（0~127），而不是byte array。如果规划百亿级存储，计划每个桶分担10个kv，那么我们只需2^30=1073741824的桶个数即可，也就是最终key的个数。

减少碎片
碎片主要原因在于内存无法对齐、过期删除后，内存无法重新分配。通过上文描述的方式，我们可以将人口标签和mapping数据按照上面的方式去存储，这样的好处就是redis key是等长的。另外对于hashmap中的key我们也做了相关优化，截取cookie或者deviceid的后六位作为key，这样也可以保证内存对齐，理论上会有冲突的可能性，但在同一个桶内后缀相同的概率极低(试想id几乎是随机的字符串，随意10个由较长字符组成的id后缀相同的概率*桶样本数=发生冲突的期望值<<0.05,也就是说出现一个冲突样本则是极小概率事件，而且这个概率可以通过调整后缀保留长度控制期望值)。而value只存储age、gender、geo的编码，用三个字节去存储。

另外提一下，减少碎片还有个很low但是有效的方法，将slave重启，然后强制的failover切换主从，这样相当于给master整理的内存的碎片。

推荐Google-tcmalloc， facebook-jemalloc内存分配，可以在value不大时减少内存碎片和内存消耗。有人测过大value情况下反而libc更节约。

md5散列桶的方法需要注意的问题

1）kv存储的量级必须事先规划好，浮动的范围大概在桶个数的十到十五倍，比如我就想存储百亿左右的kv，那么最好选择30bit31bit作为桶的个数。也就是说业务增长在一个合理的范围（1015倍的增长）是没问题的，如果业务太多倍数的增长，会导致hashset增长过快导致查询时间增加，甚至触发zip-list阈值，导致内存急剧上升。

2）适合短小value，如果value太大或字段太多并不适合，因为这种方式必须要求把value一次性取出，比如人口标签是非常小的编码，甚至只需要3、4个bit（位）就能装下。

3）典型的时间换空间的做法，由于我们的业务场景并不是要求在极高的qps之下，一般每天亿到十亿级别的量，所以合理利用CPU租值，也是十分经济的。

4）由于使用了信息摘要降低了key的大小以及约定长度，所以无法从redis里面random出key。如果需要导出，必须在冷数据中导出。

5）expire需要自己实现，目前的算法很简单，由于只有在写操作时才会增加消耗，所以在写操作时按照一定的比例抽样，用HLEN命中判断是否超过15个entry，超过才将过期的key删除，TTL的时间戳存储在value的前32bit中。

6）桶的消耗统计是需要做的。需要定期清理过期的key，保证redis的查询不会变慢。

4、Redis消息队列消息实现原理是什么?
redis设计用来做缓存的，但是由于它自身的某种特性使得它可以用来做消息队列，它有几个阻塞式的API可以使用，正是这些阻塞式的API让其有能力做消息队列；另外，做消息队列的其他特性例如FIFO（先入先出）也很容易实现，只需要一个list对象从头取数据，从尾部塞数据即可；redis能做消息队列还得益于其list对象blpop brpop接口以及Pub/Sub（发布/订阅）的某些接口，它们都是阻塞版的，所以可以用来做消息队列。

5、两个线程交替打印0~100的奇偶
复制代码
using System;
using System.Threading;
namespace LeeCarry
{
    public class Test
    {
        public static EventWaitHandle oddFlag=new EventWaitHandle(false,EventResetMode.AutoReset);
        public static EventWaitHandle evenFlag=new EventWaitHandle(false,EventResetMode.AutoReset);

        public static void Main(string[] args)
        {
            Thread oddThread=new Thread(OddThread);
            Thread evenThread=new Thread(EvenThread);
            evenThread.Start(); 
            oddThread.Start();
                      
        }
        
        private static void EvenThread()
        {
            for(int i=0;i<=100;i+=2)
            {
                Console.WriteLine("线程1:{0}",i);
                oddFlag.Set();
                evenFlag.WaitOne();
            }
            oddFlag.Set();//最后开一次绿灯防止线程一直被阻塞，也可以在waitone加时间参数
        }
        
        private static void OddThread()
        {
            oddFlag.WaitOne();//确保偶数线程先运行
            for(int i=1;i<=100;i+=2)
            {
                Console.WriteLine("线程2:{0}",i);
                evenFlag.Set();
                oddFlag.WaitOne();
            }
            evenFlag.Set();//最后开一次绿灯防止线程一直被阻塞，也可以在waitone加时间参数
        }
    }
}
复制代码
 
6、高并发系统，一般需要怎么做
分布式
提升并发的最好的办法，便是提升硬件。举个大家都熟悉的例子，十年前的诺基亚手机，一般我们只能简单的挂一个QQ后台，多干几个事情，就不行了。五年前，我们用的安卓手机能开十来个任务，切换也比较流畅了，而今天，刚刚发布的苹果iPhone11，性能就更加强劲。但是我们也发现，这两年，好像手机的性能没有飞速发展了。无论是苹果、高通还是华为，或者是PC芯片的厂商因特尔或者AMD，都开始慢慢在挤牙膏了。这其实是受到物理定理的制约，晶体管不可能无限小，无限集成，硬件不可能一直保持突飞猛进。并且，越是高端的机器，成本越贵，并且这个价格很可能是指数级增长的。谷歌公司在很早之前就发现，于是开始组建分布式系统，使用一个集群而不是一台机器来完成相关的工作，凭借这一点，谷歌在互联网早期迅速发展。
缓存
缓存，是解决高并发问题的另一个有效手段。因为磁盘的读写速度较慢，所以我们常常用读写速度的更高的内存来防止流量到达磁盘。一般我们会把一些静态资源都放在缓存上，或者将一些动态的又不怎么重要的更新频率可以接受延迟的放在缓存里。举个例子，音乐服务器，我们可以把专辑的图片、音乐文件这些放在CDN等缓存服务上，对于一些热门的评论列表，我们也可以进行缓存，一定时间才刷新一次，可以大大减少磁盘的压力。当然，有时候有缓存还远远不够，例如前几天周杰伦的新专辑照样打垮了QQ音乐的服务器。
异步
即便是有缓存，有些请求仍然没有办法快速的响应。有些请求是写请求，举个例子，沙茶敏写了一份电子邮件，群发了1万个人，群发的人数非常多，服务器要往很多人的信箱投递消息，假设一个人需要0.1秒，1万个人也要1000秒。虽然可以并发到多台机器解决，但是非常浪费资源，如果很多人这么做，系统压力非常大。另外的情况，是有可能某个系统处理非常慢，这个系统既有可能是业务非常复杂，也有可能是第三方系统，举个例子，沙茶敏从支付宝提取一笔资金到某小银行，因为技术原因，某个小银行每次接口访问都要10秒钟，不可能在转账页面卡10秒，所以支付宝先告诉用户转账成功了，然后异步进行。异步，我们通常采用了异步队列，异步的好处除了削峰，限流，提升用户体验，还能很好的保护系统。

7、分布式系统怎么做服务治理
针对互联网业务的特点，eg 突发的流量高峰、网络延时、机房故障等，重点针对大规模跨机房的海量服务进行运行态治理，保障线上服务的高SLA，满足用户的体验，常用的策略包括限流降级、服务嵌入迁出、服务动态路由和灰度发布等

8、对分布式事务的理解
本质上来说，分布式事务就是为了保证不同数据库的数据一致性。
事务的ACID特性 原子性 一致性 隔离性 持久性
消息事务+最终一致性

CC提供了一个编程框架，将整个业务逻辑分为三块：Try、Confirm和Cancel三个操作。以在线下单为例，Try阶段会去扣库存，Confirm阶段则是去更新订单状态，如果更新订单失败，则进入Cancel阶段，会去恢复库存。总之，TCC就是通过代码人为实现了两阶段提交，不同的业务场景所写的代码都不一样，复杂度也不一样，因此，这种模式并不能很好地被复用。

9、如何实现负载均衡，有哪些算法可以实现？
经常会用到以下四种算法：随机（random）、轮训（round-robin）、一致哈希（consistent-hash）和主备（master-slave）。

10、分布式集群下如何做到唯一序列号
Redis生成ID 这主要依赖于Redis是单线程的，所以也可以用生成全局唯一的ID。可以用Redis的原子操作 INCR和INCRBY来实现。

11、.net 多线程的四种实现方式
11.1、Thread类创建多线程
复制代码
/// <summary>
/// Thread类启动
/// </summary>
public static void Thread_Start()
{
    Thread thread = new Thread(new ParameterizedThreadStart(AddA));
    thread.Start("Thread");
}
复制代码
11.2、委托方式创建多线程
复制代码
delegate void Delegate_Add(Object stateInfo);
/// <summary>
/// 委托方式启动
/// </summary>
public static void Delegate_Start()
{
    Delegate_Add dele_add = AddA;
    dele_add.BeginInvoke("Delegate", null,null);
}
复制代码
 
11.3、ThreadPool类创建多线程
复制代码
/// <summary>
/// 线程池方式启动
/// </summary>
public static void ThreadPool_Start()
{
    WaitCallback w = new WaitCallback(AddA);
    ThreadPool.QueueUserWorkItem(w,"ThreadPool");
}
复制代码
11.4、Task类创建多线程
复制代码
/// <summary>
/// Task方式启动
/// </summary>
public static void Task_Start()
{
    Action<object> add_Action = AddA;
    Task task = new Task(add_Action, "Task");
    task.Start();
}
复制代码
12、数组、链表、哈希、队列、栈数据结构特点，各自优点和缺点
数组(Array)：
优点：查询快，通过索引直接查找；有序添加，添加速度快，允许重复；
缺点：在中间部位添加、删除比较复杂，大小固定，只能存储一种类型的数据；
如果应用需要快速访问数据，很少插入和删除元素，就应该用数组。

链表(LinkedList)：
优点：有序添加、增删改速度快，对于链表数据结构，增加和删除只要修改元素中的指针就可以了；
缺点：查询慢，如果要访问链表中一个元素，就需要从第一个元素开始查找；
如果应用需要经常插入和删除元素，就应该用链表。

栈(Stack)：
优点：提供后进先出的存储方式，添加速度快，允许重复；
缺点：只能在一头操作数据，存取其他项很慢；

队列(Queue)：
优点：提供先进先出的存储方式，添加速度快，允许重复；
缺点：只能在一头添加，另一头获取，存取其他项很慢；

哈希(Hash)：
特点：散列表，不允许重复；
优点：如果关键字已知则存取速度极快；
缺点：如果不知道关键字则存取很慢，对存储空间使用不充分；

13、应用程序池集成模式和经典模式的区别
如果托管应用程序在采用集成模式的应用程序池中知运行，服务器将使用 IIS 和 ASP.NET 的集成请求处理管道来处理请求。
如果托管应用程序在采用经典模式的应用程序池中运行，服务器会继续通过 Aspnet_isapi.dll
路由托管代码请求，其处理请求的方式就像应用程序在 IIS 6.0 中运行一样。
14、垃圾回收的基本原理
回收分为两个阶段： 标记 –> 压缩

标记的过程，其实就是判断对象是否可达的过程。当所有的根都检查完毕后，堆中将包含可达(已标记)与不可达(未标记)对象。
标记完成后，进入压缩阶段。在这个阶段中，垃圾回收器线性的遍历堆，以寻找不可达对象的连续内存块。并把可达对象移动到这里以压缩堆。这个过程有点类似于磁盘空间的碎片整理。
在这里插入图片描述

如上图所示，绿色框表示可达对象，黄色框为不可达对象。不可达对象清除后，移动可达对象实现内存压缩(变得更紧凑)。
压缩之后，“指向这些对象的指针”的变量和CPU寄存器现在都会失效，垃圾回收器必须重新访问所有根，并修改它们来指向对象的新内存位置。这会造成显著的性能损失。这个损失也是托管堆的主要缺点。

基于以上特点，垃圾回收引发的回收算法也是一项研究课题。因为如果真等到托管堆满才开始执行垃圾回收，那就真的太“慢”了。

垃圾回收算法 – 分代(Generation)算法

代是CLR垃圾回收器采用的一种机制，它唯一的目的就是提升应用程序的性能。分代回收，速度显然快于回收整个堆。

CLR托管堆支持3代：第0代，第1代，第2代。第0代的空间约为256KB，第1代约为2M，第2代约为10M。新构造的对象会被分配到第0代。
在这里插入图片描述

如上图所示，当第0代的空间满时，垃圾回收器启动回收，不可达对象(上图C、E)会被回收，存活的对象被归为第1代。
在这里插入图片描述

当第0代空间已满，第1代也开始有很多不可达对象以至空间将满时，这时两代垃圾都将被回收。存活下来的对象(可达对象)，第0代升为第1代，第1代升为第2代。
实际CLR的代回收机制更加“智能”，如果新创建的对象生存周期很短，第0代垃圾也会立刻被垃圾回收器回收(不用等空间分配满)。另外，如果回收了第0代，发现还有很多对象“可达”，
并没有释放多少内存，就会增大第0代的预算至512KB，回收效果就会转变为：垃圾回收的次数将减少，但每次都会回收大量的内存。如果还没有释放多少内存，垃圾回收器将执行
完全回收(3代)，如果还是不够，则会抛出“内存溢出”异常。
也就是说，垃圾回收器会根据回收内存的大小，动态的调整每一代的分配空间预算！达到自动优化！

总结

垃圾回收背后有这样一个基本的观念：编程语言(大多数的)似乎总能访问无限的内存。而开发者可以一直分配、分配再分配——像魔法一样，取之不尽用之不竭。
.NET垃圾回收器的基本工作原理是：通过最基本的标记清除原理，清除不可达对象；再像磁盘碎片整理一样压缩、整理可用内存；最后通过分代算法实现性能最优化。

15、Redis集群搭建遇到了哪些问题？
1.IP回环地址的问题：
本地建立主从服务器的时候，bind 127.0.0.1是不会有问题；但是如果要建立集群，需要使用机子本身的IP地址。

2.卡在waiting for the cluster to join的问题：
首先检查IP地址绑定是否正确，如果没有问题，检查集群配置文件是否重名：cluster-config-file 这个参数配置后面的文件名称不能重复。另外appendfilename 这个参数名称最好每个节点也不一样

3.连接请求问题：
网上很多文档说连接请求的命令是 redis-cli -c -p **** ，但是如果不是使用的本地回环，这种链接是无法链接到集群的，必须在后面加IP地址：redis-cli -c -p **** -h .***.***.

4.配置文件选项前面一定不能有空格

Not all 16384 slots are covered by nodes错误
在执行redis-trib.rb create命令创建redis集群的时候,遇到了这个错误:Not all 16384 slots are covered by nodes.
