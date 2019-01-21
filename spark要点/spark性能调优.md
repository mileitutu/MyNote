性能调优
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

    合理的并行度设置应该是设置的足够大，达到可以完全合理的利用集群资源，
