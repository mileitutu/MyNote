JVM调优：JVM相关的参数，通常情况下，如果硬件配置，基础的JVM的配置都ok的话，JVM通常不会造成严重的性能问题，更多是在troubleshooting中，JVM占了很重要的地位，JVM造成线上的spark作业运行报错，甚至失败比如(OOM)


1.降低cache操作的内存占比
    我们的RDD的缓存，task运行定义的算子函数，可能会创建很多对象，都可能会占用大量内存，没搞好的话，可能导致JVM出问题

每一次放对象的时候，都是放入Eden区域，和其中一个survivor区，另外的一个survivor区是空闲的

当eden区和survivor区放慢以后（spark运行过程中，产生对象多了），就会出发minor gc ，把不再使用的对象，从内存中清空，给后面创建的对象腾出来点地方
清理掉了不再使用的对象之后，那么会将存活下来还要继续使用的对象，放入之前空闲的哪一个survivor区域中。
默认eden，survivor1，survivor2的比例是8:1:1，如果存活下来的对象是1.5怎么办？超过了survivor2的1，一个survivor2区域放不下，此时就会通过JVM的担保机制，将多余的对象，直接放入老年代
如果JVM内存不够大，可能导致频繁的年轻带内存满溢，频繁的进行minor gc，频繁的minor gc，会导致有些存活的对象多次垃圾回收还没有回收掉，会导致一些短生命周期的对象，年龄过大，跑到老年代
老年代可能因为内存不足，囤积一大堆，短生命周期的，本来应该在年轻代中的，可能马上就要被回收掉的对象，此时会导致老年代频繁满溢，频繁进行full gc
full gc的算法设计，针对的是老年代，一般来说老年代中的对象较少，满溢进行full gc的频率应该很少，因此采取了不太复杂消耗性能和时间的垃圾回收算法，换种说法就是full gc很慢
full gc,minor gc，无论是快是慢都会导致jvm工作线程停止工作，stop the world，gc的时候，spark停止工作，等着gc结束

内存不充足导致的问题：
1.频繁的minor gc，所有的gc都会导致spark工作停止
2.理想情况下，老年代都是放一些生命周期很长的对象，数量应该很少，比如数据库连接池，老年代囤积大量活跃对象（短生命周期对象），将导致频繁full gc，full gc时间长，有可能长达数小时，会导致spark长时间停止工作

    JVM调优第一个技术点：降低cache操作的内存占比
    spark中，堆内存又被划分为2块
        1.专门用来给RDD进行cache，persist操作。进行数据缓存用
        2.用来给spark算子函数运行使用的，存放函数中自己创建的对象
    
    默认情况下，给RDD cache到做的内存占比是0.6,如果某些情况下，cache不是那么紧张，而task算子函数中创建的对象过多，然后内存又不大，导致频繁的额minor gc，甚至full gc，导致spark频繁停止工作。
    针对这种情况，查看spark UI，查看gc时间，如果gc太频繁，时间太长，此时就可以适当调整这个比例。

    降低cache操作的内存占比，大不了用persist操作，选择将一部分缓存的RDD 写入磁盘，或者序列化的方式，配合kryo序列化类，减少rdd缓存的内存占用，降低cache操作的内存占比，对应的，算子函数的内存占比就提升了，这时候，就可以减少minor gc的频率，同时减少full gc的频率。
    
    总结：让task执行算子函数的时候，有更多的内存可以使用
    spark.storage.memoryFraction,0.6 -> 0.5 -> 0.4 -> 0.2




2.executor堆外内存与连接等待时长
    executor堆外内存溢出
    如果spark作业处理的数据量特别大，几亿的数据量，然后，spark作业运行的时候会时不时的报错，shuffle file cannot find，executor、task lost，out of memory（内存溢出）；等
    说明executor的堆外内存不够用，导致executor运行过程中内存溢出，导致，后续的stage的task在运行时，当需要从executor去拉去shuffle map output文件时，executor已经挂掉了，关联的block manager也没有了，

    
    如果stage0（这里是举例）的executor挂了，blockmanager没有了，stage1的task，通过driver的mapoutputtracker获取到了自己数据的地址，但是实际上去找对应的blockmanager去获取数据的时候，是获取不到的

    所以就会报shuffle output file not found；resubmitting task；executor lost；spark作业一直挂，反复几次作业就崩溃了

    这种情况下，要考虑调节executor堆外内存，有可能会避免报错，而且堆外内存调节比较大时对于性能也会带来一定的提升


    --conf spark.yarn.executor.memoryOverhead=2048
    在spark-submit脚本里，用--conf的方式，去添加设置：一定要注意不是在代码中以sparkconf().set()的方式添加，这样没有用，一定是要在spark-submit脚本中以--conf方式添加

    --conf spark.yarn.executor.memeoryOverhead=
    针对的是基于yarn的提交模式

    默认情况下，这个堆外内存上限大概是300多M，但是通常在项目中，真正出力大数据的时候，这里都会出现问题，导致spark作业反复崩溃，此时要去调节这个参数，至少设置为1024M，甚至到2G,4G

    通常这个参数调整好以后，就会避免到某些OOM异常，同时会让spark作业整体性能得以得到较大的提升


    中华石杉工作中用的spark-submit脚本
    /usr/local/spark/bin/spark-submit \
    --class com.ibeifeng.sparkstudy.WordCount \
    --num-executors 80 \
    --driver-memory 6g \
    --executor-memory 6g \
    --executor-cores 3 \
    --master yarn-cluster \
    --queue root.default \
    --conf spark.yarn.executor.memoryOverhead=2048 \
    --conf spark.core.connection.ack.wait.timeout=300 \
    /usr/local/spark/spark.jar \
    ${1}



    调节连接等待时长
    当一个executor需要拉取数据时，它先从自己本地的blockmanager去尝试拉取，当他发现没有，就回去通过transferservice去远程连接其他节点上executor的block manager去获取，如果此时被拉取得executor正在GC，就会没有响应，无法建立网络连接，会卡住，超过spark默认的网络连接超时时长，60s，就宣告失败

    偶尔的，如果出现一串file id， uuid（dsfsfd-2342vs--sdf--sdfsd）。not found。file lost。
    这种情况下，很有可能，有那份数据的executor正在gc，所以拉取数据的时候，建立不了连接，默认超过60s以后，直接宣告失败
    报错几次都拉取不到的话，就会导致spark作业崩溃，也可能会导致DAGScheduler，反复提交几次stage，taskscheduler反复提交几次task，大大延长spark作业时长

    这时应考虑，调节连接超时时长
    --conf spark.core.connection.ack.wait.timeout=300
    注意是在spark-submit脚本中设置
    spark.core.connection.ack.wait.timeout（建立不上连接的时候的等待时长）
    把这个值调大，可以避免部分的偶尔出现的某某文件拉取失败，某某文件lost

以上两个参数在实际工作中用的较多
