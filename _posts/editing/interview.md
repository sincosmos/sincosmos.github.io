# Java
## Java 基础
### Integer Cache
为了节省内存和提高性能，部分整型对象（-128 ～ +127）在内部实现中通过使用相同的对象引用实现了缓存和重用。这种整型对象仅在自动装箱的时候才会重用，使用构造器创建的 Integer 对象会在 heap 区域新创建一个对象。
### 范型的原理
### switch(String) 的原理
switch 只支持整型，类如 int, short, byte, char 将被转化为相应的 ascii 码，而 String 则用的是 hashCode() 方法返回 int 型的 hash 值，然后再使用 equals 方法进行进一步判断。
### private 修饰的方法可以通过反射访问，那么 private 的意义是什么？
private 关键字并非是为了保证变量或方法的安全性，更多是为了实现面向对象编程中的 封装 特性，使得与对外功能无关的细节隐藏起来，让使用者只关注对外公布的接口，从而降低使用成本。
### Java 类初始化的顺序
基类静态代码块 -> 基类静态成员变量 -> 派生类静态代码块 -> 派生类静态成员变量，基类的构造代码块 -> 基类普通成员变量 -> 基类构造函数 -> 派生类构造代码块 -> 派生类普通成员变量 -> 派生类构造函数
静态代码块和静态变量只在类第一次加载时执行或初始化，多个静态代码块按照代码出现的顺序
## Java 数据结构/容器
### Java 常用的数据结构
1) 数组  
2) Collection -> List (ArrayList, LinkedList), Set(HashSet 基于 HashMap 实现, TreeSet 基于 TreeMap),
   Queue (LinkedBlockingQueue, Executors 中使用 LinkedBlockingQueue 实现 FixedThreadPool, ArrayDeque 常常比 LinkedList 效率高).  
3) Map -> HashMap, TreeMap, LinkedHashMap (LRU cache), WeakHashMap, ConcurrentHashMap
### HashMap put 的返回结果
如果 key 已经存在，则返回原来的 value；如果 key 不存在，返回 null

## JVM 
1) JVM 内存模型：方法区（存放类信息、常量、静态变量，运行时的常量池也是位于方法区，Hotspot虚拟机在java 8之前，以
   永久代的形式实现方法区，在 full GC 时进行垃圾回收；java 8 开始，使用元数据取代了永久代）、堆区（存放类实例，是
   GC 的主要对象）、栈区(线程私有，存放一个个的栈帧，当线程调用一个 java 方法时，jvm 将存放相应方法信息的栈帧 push 
   到线程的方法栈)、程序计数器（线程私有，当前线程所执行的字节码指针）、本地方法栈
2) 类加载过程：一个类从被加载到虚拟机内存中开始，到卸载出内存为止，它的生命周期包括加载（找到类，并将类二进制码加载到方法区）、
   验证（验证类的正确性）、准备（为类变量分配内存并初始化）、解析（将符号引用转变为直接引用）、初始化（执行类提供的初始化代码，
   初始化类变量）、使用和卸载。加载、验证、准备、解析和初始化过程由类加载器负责。JVM 有四种类加载器，即启动类加载器、扩展类
   加载器、应用程序类加载器和用户自定义类加载器。类加载器之间应用双亲委派机制加载类。
3) GC：通过引用计数和可达性分析确定哪些是待回收的垃圾对象，采用标记-清除算法（实现简单，但会产生内存碎片）、复制算法、标记-整理算法（解
   决内存碎片问题，但是垃圾回收代价大）或分代收集算法（目前主流的 GC 策略）中的一种。
   示例：java -Xmx3550m -Xms3550m -Xmn2g -Xss128k
   解析：-Xmx 设置 jvm 最大堆内存；-Xms 设置初始堆内存，一般和最大可用内存一样；-Xmn 设置年轻代大小；-Xss 设置每个线程的栈大小
   
## Java 并发与多线程  
### ThreadPoolExecutor
构造函数 ThreadPoolExecutor(int corePoolSize, int maxPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler)
### java.util.concurrent 包下面的集合类
ConcurrentHashMap, ArrayBlockingQueue (A bounded blocking queue backed by an array), LinkedBlockingQueue (An optionally-bounded blocking queue based on linked nodes), SynchronousQueue (A blocking queue in which each insert operation must wait for a corresponding remove operation by another thread, and vice versa)
### java.util.concurrent 包 CountDownLatch
A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.


# 软件设计
## 面向对象
1. 面向对象的三个基本特征：封装（private, public, protected，只暴露对外的 API）、继承（泛化，组合/聚合）、多态（覆盖）
2. 设计原则。单一职责原则（每个模块职责单一。高内聚：每个模块独立完成自己的功能；低耦合：减少模块间交互的复杂度）；里氏替换原则（程序中，子类必须能替换他们的基类而不会引起程序错误。不要破坏继承体系，子类可以扩展父类功能，但不要覆盖父类的非抽象方法。如果非要重写父类的方法，比较通用的做法应该是改变类的的继承结构，使原来的父类和子类都继承一个更抽象的父类，去掉原来的继承关系，采用依赖、聚合、组合等关系实现）；依赖倒置原则（面向接口编程，抽象不应该依赖细节）；接口隔离原则（多个专用接口优于一个单一的通用接口，例如函数式接口）；开闭原则（一个类、模块或函数应该对扩展开放，对修改关闭）
3. 依赖：一个类 A 的方法使用到了另一个类 B，B 的变化会影响到 A，但这种影响是较弱的；关联：两个类或者类与接口之间一种强依赖关系，例如类 A 中有一个属性是类 A 的实例；聚合：比关联更强的，是整体与部分的关系，多个整体可以共享组件；组合：比聚合更强，整体和组件拥有相同的生命周期，多个整体不能共享组件（少继承，多组合，组合可以被说成”我请了个老头在我家里干活” ，继承则是“我父亲在家里帮我干活”）。
## 领域驱动模型
简单地说，软件开发不是一蹴而就的事情，我们不可能在不了解产品（或行业领域）的前提下进行软件开发，在开发前，通常需要进行大量的业务知识梳理，而后到达软件设计的层面，最后才是开发。而在业务知识梳理的过程中，我们必然会形成某个领域知识，根据领域知识来一步步驱动软件设计，就是领域驱动设计的基本概念。而领域驱动设计的核心就在于建立正确的领域驱动模型。

# 常见框架
## Spring
### Spring AOP 与事务
1. 对于没有实现接口的类，Spring 利用 cglib 提供的代理策略，其底层原理是继承被代理类生成代理类，但代理类并不需要开发人员显式定义。开发人员借助 MthodInterceptor 接口定义被代理类中需要植入被代理的方法的操作（例如打印日志，增加事务等），借助 Enhancer 类来创建代理类实例。

# 网络基础
## HTTP和TCP协议
1) HTTP/S 超级文本传输协议，定义OSI七层网络模型中应用层的数据传输标准。理论上POST请求常用于向服务器发送信息，GET 请求常用于从服务器获取信息。GET请求把参数包含在 URL 中，POST通过 request body 传递参数；GET请求在 URL 中传递的参数是有长度限制的（URL 的最大长度是 2048 个字符），而POST请求的参数长度则不被限制；URL中不能包含任何非 ASCII 字符，因此 GET 请求的参数常常需要进行编码后传输。 
2) TCP/IP 协议定义在OSI网络模型中的传输层，是一种提供可靠通信的网络传输协议。TCP 协议为了保证服务端能收接受到客户端的信息并能做出正确的应答而进行前两次(第一次和第二次)握手，为了保证客户端能够接收到服务端的信息并能做出正确的应答而进行后两次(第二次和第三次)握手。TCP三次握手是"在不可靠的信道上可靠地传输信息"这一需求导致的。套接字（socket）是通信的基石，是支持TCP/IP协议的网络通信的基本操作单元。它是网络通信过程中端点的抽象表示，包含进行网络通信必须的五种信息：连接使用的协议，本地主机的IP地址，本地进程的协议端口，远地主机的IP地址，远地进程的协议端口。TCP四次挥手：由于TCP连接是全双工的，因此每个方向都必须单独进行关闭。客户端A发送一个FIN，用来关闭客户A到服务器B的数据传送。服务器B收到这个FIN，它发回一个ACK，确认序号为收到的序号加1。服务器B关闭与客户端A的连接，发送一个FIN给客户端A。客户端A发回ACK报文确认，并将确认序号设置为收到序号加1。

# 数据存储
## MySQL 数据库
[MySQL]()
## ElasticSearch
### 架构原理
ES 是近实时分布式搜索引擎，底层基于 Lucene 实现，核心思想是在多台机器上启动多个 ES 进程实例，构成 ES 集群。
### 数据写入
1. 客户端选择一个 node 发送写请求，这个 node 将作为 coordinating node
2. Coordinating node 根据 document ID （可以指定，也可以由服务器分配）对 document 进行路由计算，将内容转发给相应的 node 作为保存数据 primary shard 的节点
3. Primary shard 保存到相应的 node 上后，相应的 node 负责将数据同步到 replica node
4. Coordinating node 发现 Primary node 和所有的 replica node 都完成后，返回请求到客户端
写入时，数据先写入到 buffer 里，同时将数据写入到 translog 中（异步将数据同步到磁盘），在 buffer 里的数据是不会出现在检索结果中；
### 数据检索
根据 document ID 检索时，接受请求的 node (coordinating node) 将对 document 进行路由，将请求转发到对应的 node，获取数据。根据关键字进行检索时，coordinating node 将搜索请求转发给所有的分片机器（Primary or replica）；每一个分片都将自己的搜索结果的唯一标识返回给 coordinating node，由其对数据合并、排序、分页等操作，产出最后的结果；最后 coordinating node 再根据唯一标识去各个节点拉去数据，返回给客户端。
### 倒排索引
ES 分别为每个 field 都建立了一个倒排索引，根据针对 field 的分词策略，每一个值可能被拆分成许多个 term 对应相应的 posting list（即 term 出现了的 document ID 列表）。这样，一个 ES 索引可能会出现非常多的 term，为了快速定位某个 term，ES 将所有的 term 进行排序，使用二分法查找 term。但由于 term 太多，全部读入内存也会有较大消耗，于是有了 term index（缓存在内存），即取 term 的前缀建立一棵搜索树，通过 term index 快速定位到以某个前缀开头的 term 的 offset，然后从 offset 开始往后顺序查找，就会很快找到相应的 term。
## Redis
### 基本数据结构
参考资料 [Redis 数据结构基础教程](https://juejin.im/post/5b53ee7e5188251aaa2d2e16)
1) string 是一个可变的字节数组，常见的操作有 set, get, strlen, getrange, setrange, append, ttl, expire, del. 如果存入的是整数，还可以作为计数器使用，incrby, decrby, incr, decr. 使用 setbit, getbit, bitcount, bitop (bitop and/or/xor destkey key [key...]) 命令可以将 string 类型作为 bit map 使用。
2) redis 使用双向链表结构保存列表，是以列表结构为 list。因为是双链表，list 即可作为队列(lpush/rpop, rpush/lpop)使用，也可以作为堆栈(rpush/rpop, lpush/lpop)使用。用 list 可以实现 redis 消息队列（ redis 另外提供了发布/订阅机制实现消息队列)。list 的 push 支持一次操作多个元素（实际 push 是依次一个一个进行的），pop 操作一次只能弹出一个元素。list 常见的其它操作有 llen (list length), lindex (get element by index), lrange (get elements by range of indices), lset (set element by index), linsert before/after (insert element before/after sepcified element), lrem count (if count > 0, search from head, remove count of elements; if count < 0, search from tail, remove count of elements; if count = 0, remove all elements), ltrim (维护定长列表，除了范围内的元素外，其它元素将被删除)
3) hash 等价于 java 中的 HashMap，存储的是键值对。使用 hset (hset v_name key1 value 一次加入一个键值对)或 hmset（hmset v_name key2 value2 key3 value3... 一次加入多个键值对）。hdel 支持一次删除一个或多个键。hget 获得键对应的值，hexists 判断键是否存在于 hash 中。如果键对应的值是整数，我们也可以将整数作为计数器使用，相关的命令有 hincrby, hdecrby。hgetall 得到 hash 中所有的键值对。
4) 类似 java, redis set 的内部实现也是基于上面的 hash，所有的键指向同一个内部值。常见的命令有 sadd (增加一个或多个元素)，members (获得所有元素)，scard (获得元素个数)，srandmember v_name [count]（随机获取 count 个元素，如果省略 count，随机取一个元素），srem v_name key1 key2 (删除一个或多个元素)，spop（随机删除一个元素），sismember （判断元素是否在 set 中）
5) zset 是 redis 提供的有序集合，可以将其理解为 map<key, score>，除了 hash 结构，为了提高增删查改的效率，底层还使用了 skiplist 结构，skiplist 按照 score 进行排序，score 可以是整数或浮点数。常用的命令有 zadd v_name score key (增加一个或多个元素)，zcard (获取元素个数)，zrem (删除一个或者多个元素)，zincrby v_name 1.2 key1 (作为计数器使用)，zscore (获取对应 key 的权重)，zrank (获得元素的正向排名)，zrevrank（获得元素的倒数排名），zrange v_name start end [withscores]（获得正向排名从 start 到 end 的所有元素，可选是否同时获得元素对应的 scores），zrevrange 和 zrange 类似，只不过它按负向排名输出元素。zrangebyscore v_name score_start score_ed [withscores] 和 zrevrangebyscore 获取指定 score 范围内的元素。zremrangebyrank 和 zremrangebyscore 分别删除指定排名范围内和权重范围内的元素。
### 发布/订阅模式
1) subscribe channel [channel ...]，订阅给定的频道列表
2) publish channel message ，将消息发送到指定的频道
3) unsubscribe channel [channel ...]，退订给定的频道列表
4) psubscribe pattern [pattern ...]，订阅符合给定模式的频道列表
5) punsubscribe pattern [pattern ...]，退订符合给定模式的频道列表
6) pubsub <subcommand> [argument [argument ...]] 查看订阅与发布系统状态，它由数个不同格式的子命令组成。
### 分布式锁
1) setex key timeout(s) value，设置 key 并并设置过期时间，整个过程是原子操作
2) setnx key value, 当指定的 key 不存在时，为 key 设置指定的值
3）getset key value, 将 value 赋值给 key，并返回旧值，如果旧值不存在返回 nil，如果旧值不是字符串类型，返回错误

### 配置文件

## Spark SQL 内核剖析
1. 相比传统的 MapReduce，spark 提供了更加灵活的数据操作方式，有些需要分解成几轮的 MapReduce 操作，可以在 Spark 里一轮实现；另一方面，每轮的计算结果都可以分布式地存放在内存中，下一轮直接从内存读取，节省大量磁盘 IO 开销。
2. SQL-on-Hadoop 解决方案从架构上来看从上到下分为应用层（为用户提供数据管理查询接口），分布式执行层（例如 MapReduce, Spark 计算引擎等），存储层（HDFS, 分布式 NoSQL 数据库等，分布式执行层通过底层接口访问存储层中的数据）。


