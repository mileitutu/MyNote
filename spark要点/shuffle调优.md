什么情况会发生shuffle？
    在spark中主要是以下几个算子，reducebykey，groupbykey，countbykey，join等

什么是shuffle？
    reduceByKey：把分布在集群中各个节点的数据的同一个key，对应的values都集中到一起，具体来说，就是，集中到一个节点的，一个executor的一个task中。
    集中了一个key对应的values之后，形成<key,iterable<value>>；然后reduceByKey算子才能去对values集合进行reduce操作，最后变成一个value

    countbykey：需要在一个task中获取到一个key对应的所有的value，然后进行计数，统计有多少个value

    join：RDD<key, value>，RDD<key, value>，只要是两个rdd的key相同，对应的value都会拉取到一个节点的executor的task中，进行处理。

    
    wordcount程序为例，应用reduceByKey算子，同一个key可能落到不同节点上，为每个单词进行累加计数，必须让所有单词都跑到同一个节点的一个task中，给一个task来进行处理。

    每一个shuffle的前半部分stage的task，每个task都会创建下一个stage的task数量相同的文件，比如下一个stage会有100个task，那么当前stage每个task都会创建100份文件，会将同一个key对应的values，一定是写入同一个文件中，不同节点上的task，也一定会将，同一个key对应的value写入下一个stage，同一个task对应的文件中。

    shuffle的后半部分stage的task，每个task都会从各个节点上的task写的属于自己的那一份文件中，拉取key，value对，task会有一个内存缓冲区，会用hashmap进行key，value汇聚

    同样，shuffle前半部分的task在写入数据到磁盘文件之前，都会先写入一个内存缓冲，内存缓冲溢满后，再spill到磁盘文件中

    前半部分的stage，会写file，后半部分直接在内存里办了

    shuffle一定分为两个stage来完成



shuffle如何调优？
1.合并map端输出文件（性能提升明显）

    SparkConf().set("spark.shuffle.consolidateFiles","true")
    开启shuffle map端输出文件合并的机制；默认情况下，是不开启的，就是会发生大量map端输出文件，会严重影响性能

    开启了consolidate机制后，每个stage只创建下一个stage的task个map端文件，其他task复用该文件，将数据写入上一批task的输出文件中
    第二个stage，task在拉取数据的时候，不会去拉去上一个stage每一个task为自己创建的那份输出文件了，而是拉取少量的输出文件，每个输出文件中，包含了多个task给自己的map端的输出，总之就是找有自己需要的数据的文件进行拉取


     第一个stage，每个task，都会给第二个stage的每个task创建一份map端的输出文件
     第二个stage，每个task，会到各个节点上面会到各个节点上面去，拉去第一个stage每个task输出的，属于自己的那一份文件。

     shuffle中的写磁盘操作时shuffle中性能消耗最严重的部分，磁盘IO对性能和spark作业执行速度的影响极其惊人，基本上spark作业的性能，都消耗在shuffle中了

    注意（map端输出文件合并）
    只有并行执行的task会去创建新的输出文件，下一批并行执行的task会去复用之前已有的输出文件，但是：
    如果两个task在并行执行，此时又要同时启动执行2个task，那么这个时候，就无法复用刚才的两个task创建的输出文件，只能还是去创建新的输出文件

    也就是：要实现输出文件的合并的效果，必须是一批task先执行，然后再下一批task再执行，才能复用之前的输出文件。必须是一个task执行完，在下一个task执行才行

    合并map输出文件对application的性能的影响
    1.map task 写入磁盘文件的IO减少
    2.stage2 要拉取得文件减少，网络传输性能消耗，也大大减少



2.调节map端内存缓冲与reduce端内存占比
    spark.shuffle.file.buffer，默认32k(map端内存缓冲)
    spark.shuffle.memeoryFraction, 0.2（reduce端内存占比）
    以实际经验来看，这两个参数没有很重要，但是有点效果，最重要的还是与其他点合起来用

    默认情况下，shuffle的map task，输出到磁盘文件的时候，统一都会先写入每个task自己关联的一个内存缓冲区这个缓冲区大小是，默认32kb，每当一次内存缓冲区溢满后，才会进行spill操作，溢写操作，溢写到磁盘文件中去

    reduce端的task，拉取到数据后，会用hashmap的数据格式，来对各个key对应的values进行汇聚，针对每个key对应的values，执行我们自定义的函数
    reduce task 在进行汇聚，聚合等操作时，实际上就是使用的子集对应的executor的内存，executor默认给reduce task的内存比例是0.2

    如果拉去过来的数据过多，那么在这0.2的内存放不下都会spill到磁盘文件中去

    如果不对map的buffer和reduce端的fraction进行调优
    map task默认的buffer比较小，会频繁发生spill，发生大量的磁盘IO，降低性能
    reduce端默认0.2，如果不够，会频繁spill，而且后面进行聚合的时候还会从磁盘读数据进行聚合
    如果默认不调优reduce fraction，那么在数据量大的情况下，可能频繁的产生reduce端的磁盘文件读写
    这两个参数出问题的点差不多，数据量大，频繁读写，影响性能

    如何调油这两个参数？
        yarn模式，进入yarn页面，点击对应的application，进入spark UI，查看详情
        如果发现shuffle磁盘的write和read，很大，这就意味着最好调节一些shuffle参数，首先要考虑开启consolidate机制

        调节spark.shuffle.file.buffer,每次扩大一倍，看看效果，32,64,128
        调节spark.shuffle.memoryFraction，每次提高0.1看看效果
        不能调节太大，过犹不及，会影响其他方面

    调节后的效果？
        map task 缓冲变大了，spill减少
        reduce内存变大了，减少spill，减少了后面聚合时读取磁盘文件的数量



3.选择HashShuffleManger与SortShuffleManager
    1.2以前的版本默认HashShuffleManager，比较老，1.2版本后就不是默认的了
    默认的是sortShuffleManger
    spark 1.5以后出来一种新的manager，tungsten-sort，效果和sortShuffleManger是差不多的，但是实现了自己的一套内存管理机制，性能上有很大提升，可以避免shuffle过程中产生的大量OOM,GC等内存相关的异常

    sortShuffleManger与HashshuffleManager的不同点在于
    1.sortShuffleManger会对每个reducetask要处理的数据，进行排序（默认的）
    2.sortShuffleManger会避免像HashShuffleManger，默认就去创建多份磁盘文件，1个task，只会写入一个磁盘文件，不同的reduce task，通过offse来界定自己要拉取的的数据
    consolidate机制对于sortShuffleManger依然有用
    sortShuffleManger默认排序，如何解除？
    spark.shuffle.sort.bypassMergeThreshold：200
    使用这个参数，当reduce数量少于或等于200，map task创建的输出文件小于等于200，最后会将所有的输出文件合并为一个文件，这样做就避免了sort，节省了性能，并且还能将多个reduce task文件合并为1份，节省了reduce task拉取数据时候的磁盘IO
