性能调优

分为四大部分
基础调优
JVM调优
shuffle调优
spark算子调优


（1）分配更多资源
    性能调优的王道，就是增加和分配更多的资源。
    在写完一个spark程序之后，调优第一步要做的就是尽可能拿到更多资源，达到天花板之后，再去考虑接下来的调优工作

    1.分配那些资源？
        executor，cpu per executor，memory per executor，drver memory

    2.在哪里分配这些资源？
        在生产环境中，提交spark作业时，用spark-submit脚本来调整对应的参数

        /usr/local/spark/bin/spark-submit \ 
        --class XXX \
        --num-executors \ 配置executor的数量
        --driver-memory 100m \ 配置driver的内存（影响不大）
        --executor-memory 100m \ 配置每个executor的内存大小
        --executor-cores 3 \ 配置每个executor的cpu core的数量
        /usr/local/###

    3.调节到多大，算最大？
        第一种，spark standalone ， 在公司集群上，搭建一套spark集群，心里应该清楚，每台机器还有多少资源给你使用，内存，cpu core，根据这个实际情况，去调节每个spark作业的资源分配。

        第二种：yarn自愿队列，资源调度。去查看spark作业要提交到的资源队列，大概有多少资源
        一个原则，能使用的资源有多大，就尽量去调节到最大的大小


    4.为什么分配这些资源后，性能会提升？
    sparkcontext会将算子切割成多个task提交到executor，增加每个executor的cpu core的数量就是增加并行执行的task的数量，这样可以提升执行速度。
    
    增加每个executor的内存量
        1.如果需要对rdd进行cache，那么更多的内存，就可以缓存更多的数据，将更少的数据写入磁盘，甚至不写入磁盘，减少了磁盘IO
        2.对于shuffle操作，reduce端，需要内存来存放拉取的数据并进行聚合，如果给executor分配更多的内存，就会有更少的数据写入磁盘，甚至不需要写入磁盘，减少了磁盘IO，提升了性能
        3.对于task的执行，可能会创建很多对象，如果内存比较小，可能会导致JVM堆内存满了，然后频繁GC，minor GC 和 full GC ，内存加大以后，带来了更少的GC，提升了性能。

    增加executor：
    如果executor数量较少，那么能够并行执行的task数量就较少，意味着application并行执行能力弱

（2）调节并行度
    并行度是什么？
    并行度是每个spark作业中，每个executor中并行执行的task的数量

    如果不调节并行度，导致并行度过低，会导致资源浪费。
    比如，有50个executor，每个executor3个cpu core，可只设置了100个task，平均每个executor 2个task，这样每个executor就有一个cpu core空闲

    合理的并行度设置应该是设置的足够大，达到可以完全合理的利用集群资源
    只要合理设置并行度，就可以完全充分利用集群计算资源，并且减少每个task处理的数据量，最终提升整个spark作业性能
    task数量应设置为application 总的cpu core数量的2-3倍，因为有的task可能先完成，有的task可能慢一些，如果设置为与cpu core一样，会造成资源浪费。应尽量让cpu不要空闲

    如何设置application并行度？
    应设置spark.default.parallelism 参数
    SparkConf conf = new SparkConf().set("spark.default.parallelism",500)

    “重剑无锋”：真正起到最大作用的都看起来平凡无奇，但是每次写完代码，进入调优阶段最应该做的事情就是这些（增加资源，提高并行度）
    “炫酷”：数据倾斜，可能只有十分之一的application会出现，JVM调优



    以reduceByKey为例，在执行到shuffle算子后，会为每个stage1（第二个stage，第一个stage是stage 0），的task都创建一份文件（也可能是合并在少量文件里面），每个stage1的task，回去各个节点上的各个task创建的属于自己的那一份文件里，拉取数据，每个stage1的task，拉取到的数据，一定是相同key对应的数据。
    对相同的key，对应的values，才能去执行我们自定义的fuction操作

(3) 重构RDD
    1.默认情况下，多次对一个RDD执行算子，去获取不同的RDD，都会对这个RDD以及之前的父RDD，全部重新计算一次。
    这种情况是绝对绝对一定要避免的，一旦出现一个RDD重复计算的情况，就会导致性能急剧降低

    2.另一种情况，从一个RDD，到几个不同的RDD，算子和逻辑其实是完全一样的，结果因为认为的疏忽，计算了多次，获取到了多个RDD。

    如何优化？
        1.RDD架构重构与优化
        尽量复用RDD，差不多的RDD，可以抽取为一个共同的RDD，供后面的RDD计算时，反复使用
        2.公共RDD一定要实现持久化
        对于多次计算和使用的公共RDD一定要进行持久化
        持久化，就是将RDD的数据缓存到内存中/磁盘中，（blockmanager），以后无论对这个RDD做多少次计算，都是直接取这个RDD的持久化的数据。
        3.持久化，是可以进行序列化的
        如果正常将数据持久化在内存中，可能会导致内存占用过大，也许会导致OOM
        当纯内存无法支撑公共RDD数据完全存放的时候，就要优先考虑，使用序列化的方式，在纯内存中存储，将RDD的每个partition的数据，序列化成为一个打的字节数组，就是一个对象，序列化后，将大大减少内存的占用空间。
        序列化的缺点就是在获取数据时，需要反序列化
        如果序列化纯内存方式，还是导致OOM了，就只能考虑，磁盘的方式，内存+磁盘的普通方式（无序列化）
        还不行，内存+磁盘（序列化）

       4. 为了保证数据高可靠性，可以考虑双副本机制，进行持久化，这种方式，仅仅针对内存资源极度充足的情况。
        持久化的双副本机制，因为机器宕机了，副本丢了，就还是要重新计算一次。持久化的每个数据单元，存储一份副本，放在其他节点上面。


（4）使用广播变量
    比如随机抽取map，每个task会拷贝一份map副本，如果这个map副本很大，是个大变量，比如（1m-100m）,此时就该使用广播，broadcast，将大变量广播到各个节点上。而不是直接使用
    默认情况下，task执行的算子中，使用了外部变量，每个task都会获取一份变量的副本
    
    比如1m的外部变量，有100个task都用到了这个外部变量，就要传输100次的1m的外部变量到各个task中去，光网络传输就会耗费一定资源。

    不必要的内存消耗和占用，就导致了，在RDD持久化到内存也许就没法完全在内存中放下，就只能写入磁盘，最后导致后续的操作在磁盘IO上消耗性能。

    task在创建对象的时候，也许就会发现堆内存放不下所有对象，会导致频繁GC，GC的时候会导致工作线程停止，也就是导致spark暂停工作一点时间，但是如果频繁GC的话，也会有可观的影响。

    广播变量的好处在于，不是每个task一份变量副本，而是每个节点的executor才一份副本，这样的话就可以让变量产生的副本大大减少。

    广播变量，在初始的时候，就在driver上有一份副本，task想要使用广播变量的数据的时候，就会首先在本地的executor对应的blockmanager中尝试获取变量副本，如果本地没有，就从driver远程拉去变量副本，并保存在本地的blockmanager中，伺候，这个executor上的task，都会直接去使用本地的blockmanager中的副本
    executor的blockmanager除了从driver上拉取，也可能从其他节点的blockmanager上拉取变量副本，越近越好。
    blockmanager是负责管理某个executor对应的内存和磁盘上的数据的。

    虽然不一定会产生决定性的作用，比如30分钟的spark作业，做了广播变量后，快了2分钟或者5分钟，但是积少成多。

（5）使用kryo序列化
    当算子函数使用到了外部变量，外部变量会被序列化，此时可以把使用kryo来序列化外部变量
    默认情况下，spark内部使用的是java的序列化机制，objectoutputstream / objectinputStream 对象输入输出流机制来进行序列化
    默认序列化机制好处：方便，不需要手动设置，只是算子里面使用的变量必须实现serializable接口，可序列化即可
    缺点：序列化效率不高，速度慢，序列化后的数据占用的内存空间相对较大

    所以可以手动进行序列化格式的优化
    spark支持kryo序列化机制，kryo序列化机制，比默认的java序列化机制速度要快，序列化后的数据更小，大概是java序列化机制的1/10
    所以kryo序列化优化后，可以让网络传输数据变少，在集群中耗费的内存资源大大减少。

    当在application中启用kryo序列化机制后，会在如下几个地方生效
    1.算子函数中使用到的外部变量
        算子函数中使用到的外部变量，使用kryo以后：优化网络传输的性能，可以优化集群中内存占用和消耗
    2.持久化RDD时进行序列化，序列化级别加ser的
        持久化RDD占用的内存越少，task执行的时候，创建的对象，就不至于频繁的占满内存，频繁的发生GC
        当时用了序列化的持久化级别时，在将每个RDD partition序列化为一个打的字节数组时，就会用kryo进一步优化序列化的效率和性能
    3.shuffle
        可以优化网络传输性能

    如何使用kryo序列化机制？
    1.在SparkConf中设置一个属性，spark.serializer.KryoSerializer类
    SparkConf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")

    kryo之所以没有被作为默认的序列化类库的原因：因为kryo要求，如果要达到他的最佳性能的话，那么就要注册自己定义的类（比如，算子函数中使用到了外部自定义类型的对象变量，这是就必须要注册你自己定义的类，否则kryo达不到最佳性能）
    
    2.注册使用到的，需要通过kryo序列化的，自定义类
    项目中的使用：
    .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
    .registerKryoClasses(new Class[]{CategorySortKey.class})


（6）使用fastutil优化数据格式

（7）调节数据本地化等待时长

    数据本地化级别：
        1.PROCESS_LOCAL:进程本地化，代码和数据在同一个进程中，也就是在同一个executor中，计算数据的task由executor执行，数据在executor的blockmanager中，性能最好
        2.NODE_LOCAL:节点本地化，代码和数据在同一个节点中，比如数据作为一个HDFS block块，就在节点上，而task在节点上某个executor中运行；或者数据和task在同一节点的不同executor中；数据需要在进程间进行传输
        3.NO_PREF:对task来说，从哪里获取都一样，没有好坏之分
        4.RACK_LOCAL:机架本地化，数据和task在一个机架的两个节点上，数据需要通过网络在节点之间进行传输
        5.ANY:数据和task可能在集群中的任何地方，而且不在一个机架中，性能最差

    1.为什么要调节数据本地化？
    spark在driver上，对application的每一个stage的task，进行分配之前，都要计算每个task是哪个分片数据，分片的意思是RDD的某个partition。spark的task分配算法，优先会希望每个task正好分配到他要计算的数据所在的节点，这样的话，就不用在网络间传输数据

    但是，有时，task没机会分配到它的数据所在的节点，因为哪个节点的计算资源可能满了，这是spark会等待一段时间，默认情况下是3s（不是绝对的，还有很多情况，对于不同的本地化级别都会去等待），等到最后实在等不了了，就选择一个比较差的本地化级别，比如，将task分配到，靠它要计算的数据所在节点的比较近的一个节点上去，然后进行计算。
    对于这种情况，肯定要发生数据传输，task会通过其所在的节点的blockmanager来获取数据
    blockmanager发现自己本地没有数据，会通过一个getRemote方法，通过transferservice（网络传输组件）从数据所在节点blockmanager中，获取数据，通过网络传输回task所在节点。
    对于我们来说，不希望出现第二种情况，最好的就是task和数据在一个节点上，直接从本地executor的blockmanager中获取数据，纯内存，或者带一点磁盘IO，如果要通过网络传输数据的话那么性能会有所下降，大量的网络传输，及磁盘IO，都是性能杀手

    什么时候调节？
    观察日志，日志里面会显示starting task，PROCESS LOCAL,NODE LOCAL等观察大部分task的数据本地化级别
    如果大多数都是PROCESS_LOCAL，那就不用调节，如果发现好多的级别都是NODE_LOCAL,ANY,等那么最好调节一下数据本地化等待时长
    应该是会反复调节，每次调节完，再来运行，观察日志，看看大部分task的本地化级别有没有提升，看看整个spark作业运行时间有没有缩短
    注意不要本末倒置，不要本地化级别提升了，但是因为大量等待时长，spark作业的运行时间反而增加了，那就还是不要调节了

    如何调节？
    spark.locality.wait 默认是3s
    下面这几个参数可以细微调整，默认和上面一样都是3s
    spark.localit.wait.process
    spark.locality.wait.node
    spark.locality.wait.rack

    new SparkConf().set("spark.locality.wait", "10")

    在一个节点，不在一个executor要走进程间传输
    不在一个节点，要走网络间传输
    最差就是跨机架拉取方式，速度非常慢，对性能影响相当大