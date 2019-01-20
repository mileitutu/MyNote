1.union算子原理
    RDD-01 有两个partition
    RDD-02 有两个partition
    对两个RDD应用union算子，RDD-01会把RDD-02的partition原封不动的挪过来，新的RDD的partition的数量是两个RDD之和

2.groupbykey算子原理
    把一个RDD里面相同的key聚合到一起，把value放到一个集合里，这个集合实现iterator
   
    一般来说，在执行shuffle类的算子，比如groupbykey，reduceByKey，join等，在算子内部会隐式地创建集合RDD出来，那些被隐式创建的RDD,主要是作为这个操作的一些中间数据的表达，以及作为stage划分的边界
    因为有的隐式生成的RDD，可能是shuffledRDD,dependency就是shuffleDependency，根据DAGScheduler源码，出现了shuffledependency，就会将这个RDD作为新的stage的第一个RDD划分出来

    groupByKey算子在执行时会隐式地创建一个shuffledRDD，可以通过给groupbykey传入参数控制shuffledRDD的并行度：groupbykey(2)此时shuffledRDD的并行度就是2，也就是有两个partition，每个task计算一个partition，有两个task可以并行执行，不代表在这个任务中只有两个task
    这个shuffledRDD是stage1的第一个rdd
    shuffledRDD时还没有形成iterator集合，再下一步进行mappartitions才完成最后的步骤
    rdd.groupbykey(2)->shuffledRDD->mapartitions->(key,iterator<>(值的集合))
    
    groupbykey等shuffle算子，都会创建一些隐式地RDD，比如这里的shuffledRDD，作为一个shuffle过程中的中间数据的代表。
    一来这个shuffledRDD创建出来一个新的stage，shuffledRDD会触发shuffle read操作，从上游stage的task所在节点拉去过来相同的key，做进一步聚合
    接着对shuffledRDD中的数据执行一个map类的操作，主要是对每个partition中的数据进行映射和聚合。这里主要是将每个key中对应的数据都聚合到一个iterator集合中


3.reduceByKey算子
    隐式生成一个mapartitionsRDD，stage 0 （本地规约聚合的结果）
    mapartitionsRDD会做本地聚合，将key相同，value不同的记录，把value加起来
    reduceByKey会进行本地聚合
    stage 1 shuffledRDD 
    val wordCounts = pairs.reduceByKey(_+_,2)这个2代表shuffledRDD只给两个partition
    shuffledRDD会把上一步中，不同partition中相同的key拉到一个partition中，接着，再进行一步聚合

    reduceByKey和groupbykey有何异同？
    1.不同之处：reduceby多了一个rdd，mappartitionsRDD，存在于stage 0 ，这个rdd代表了进行本地数据规约后的rdd，所以要网络传输的数据量，及磁盘io等会减少，性能更高
    2.相同之处：后面进行shuffle read 和聚合的过程基本和groupbykey类似，都是shuffledRDD去做shuffle read。然后聚合，聚合后的数据就是最终的rdd


4.distinct算子
    内部实现类reduceByKey算子
    首先做个maptopair操作，每条记录做map，加上一个null作为value，形成一个新的rdd，接着对这个新rdd做reducebykey，因为之前给的值是null，所以相同的key都被聚合了，value因为是null，所以不影响
    最后，因为之前给每个key加了一个null，所以要map一下，把null去掉

    1.首先，先给自己每个值，加上一个value，变成一个tuple
    2.得到tuple组成的rdd后，进行reduceByKey
    3.将去重后的数据，从tuple还原为单值
    