1.控制shuffle reduce端缓冲大小以避免OOM
    map端的task是不断的输出数据的，数据量可能很大，但是reduce端的task，并不是等到map端将属于自己的那份数据全都写入磁盘之后，再去拉取得。

    map端协议点数据，reduce端的task，就回去拉取一小部分数据，立即进行后面的聚合算子函数的应用。

    每次reduce能够拉取多少数据由buffer决定，因为拉去过来的数据都要先放入buffer中，然后，采用后面的executor分配的堆内存占比0.2，hashmap去进行后续的聚合、函数的执行

    reduce端缓冲buffer，可能出现什么问题？
    可能会出现，默认48M，也许大多数的时候，reduce端task一边拉取一边计算，可能大多数的时候，拉取10M就计算掉了。

    如果map端数量大，速度快，一下子把48M撑满了，就会OOM

    如何解决？
    这时，应该减少reduce端task缓冲的大小，一下拉取48M容易堵住，拉的没那么快，所以减少一下拉12M，多拉几次，以性能换执行

    当资源充足时，也可以吧这个buffer调大，减少拉去次数，减少网络开销，以及reduce端聚合操作执行的次数，会提升性能

    spark.reducer.maxSizeInFlight，48


2.JVM GC导致shuffle文件拉取失败
    GC时，所有工作线程全部停止工作，blockmanager也停止工作，基于netty网络通信
    这样就可能会shuffle file note found
    这种错误常见，可多试几次，gc成功了就行了

    spark.shuffle.io.maxRetries 3
    spark.shuffle.io.retryWait 5s
    如果full gc 时间较长，可能超过这两个参数，可以调大


3.yarn队列资源不足导致application直接失败
    1、YARN，发现资源不足时，你的spark作业，并没有hang在那里，等待资源的分配，而是直接打印一行fail的log，直接就fail掉了。
    2、YARN，发现资源不足，你的spark作业，就hang在那里。一直等待之前的spark作业执行完，等待有资源分配给自己来执行。

    如何解决？
    1.yarn限制只能同时提交一个spark作业
    2.长短application分开调度队列
    3.队列内给最大资源
    4.在J2EE中，通过线程池的方式（一个线程池对应一个资源队列），来实现上述我们说的方案。
    ExecutorService threadPool = Executors.newFixedThreadPool(1);
    threadPool.submit(new Runnable() {         
    @Override
    public void run() {
    }             
    });


4.序列化导致的报错
    log。如果出现了类似于Serializable、Serialize等等字眼，报错的log就碰到了序列化问题导致的报错。

    注意3个点
    1.算子函数里面如果用到了外部自定义变量，那么就要求你的自定义的类型必须可序列化
    2.如果将自定义的类型，作为RDD的元素类型，那么自定义的类型也必须可序列化
    3.不能再上述两种情况下，去使用一些第三方，不支持序列化的类型
    
    Connection是不支持序列化的

5.解决算子函数返回null导致的问题
    在有些算子函数里面，是需要我们有一个返回值的。但是，有时候，我们可能对某些值，就是不想有什么返回值。我们如果直接返回NULL的话，会报错的。
    如果碰到你的确是对于某些值，不想要有返回值的话，有一个解决的办法：
    在返回的时候，返回一些特殊的值，不要返回null，比如“-999”
    在通过算子获取到了一个RDD之后，可以对这个RDD执行filter操作，进行数据过滤。filter内，可以对数据进行判定，如果是-999，那么就返回false，给过滤掉就可以了。
    然后再coalesce一下，让各个partition数据比较紧凑一些，也能提升一些性能

6.yarn-client模式导致网卡流量激增
yarn-client测试环境下使用

7.yarn-cluster模式的JVM内存溢出无法执行的问题
有时候会允许一些包含了spark sql的作业，会碰到yarn-client模式下，可以正常提交，但是yarn-cluster模式不能正常提交，会报JVM的PermGen OOM
yarn-client模式下，driver是运行在本地机器上的，spark使用的JVM的PermGen的配置，是本地的spark-class文件（spark客户端是默认有配置的），JVM的永久代的大小是128M，这个是没有问题的；但是呢，在yarn-cluster模式下，driver是运行在yarn集群的某个节点上的，使用的是没有经过配置的默认设置（PermGen永久代大小），82M。

所以，此时，如果对永久代的占用需求，超过了82M的话，但是呢又在128M以内；就会出现如上所述的问题，yarn-client模式下，默认是128M，这个还能运行；如果在yarn-cluster模式下，默认是82M，就有问题了。会报出PermGen Out of Memory error log。

spark-submit脚本中，加入以下配置即可：
--conf spark.driver.extraJavaOptions="-XX:PermSize=128M -XX:MaxPermSize=256M"


sql，有大量的or语句,当达到or语句，有成百上千的时候，此时可能就会出现一个driver端的jvm stack overflow，JVM栈内存溢出的问题or特别多的话，就会发生大量的递归。
这种时候，建议不要搞那么复杂的spark sql语句。
根据生产环境经验的测试，一条sql语句，100个or子句以内，是还可以的。通常情况下，不会报那个栈内存溢出。


8.错误持久化方式，及checkpoint的使用

正确：usersRDD = usersRDD.cache()
    val cachedUsersRDD = usersRDD.cache()

错误：usersRDD.cache()
    usersRDD.count()
    usersRDD.take()

之后再去使用usersRDD，或者cachedUsersRDD，就可以了。就不会报错了。所以说，这个是咱们的持久化的正确的使用方式。

在对RDD进行计算的时候，如果发现它的缓存数据不见了。优先就是先找一下有没有checkpoint数据（到hdfs上面去找）。如果有的话，就使用checkpoint数据了。不至于说是去重新计算。

checkpoint，用性能换可靠性。

说一下checkpoint的使用

1、SparkContext，设置checkpoint目录
2、对RDD执行checkpoint操作
