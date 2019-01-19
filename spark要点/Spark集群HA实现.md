1.基于zookeeper实现HA高可用及自动主备切换
    standalone 和 cluster 两种模式对worker节点失败具有容错性（丢失的计算工作会迁移到其他worker节点上执行），但是调度器是依托于master进程来做出调度决策的，如果只有一个master会造成单点故障，如果master挂了就没法提交新的应用程序了
        spark拥有两种HA方案：
                    1.基于zookeeper的HA方案
                    2.基于文件系统的HA方案

    基于zookeeper的HA方案：
        使用zookeeper来提供leader选举以及一些状态存储，可以在集群中启动多个master进程，让他们连接到zookeeper实例。
        其中一个master进程会被选举为leader，其他的master进程会被指定为standby模式，如果当前的leader挂了，其他的standby模式的master会被选举，从而恢复master的状态，并且恢复作业调度，整个恢复过程大概1-2分钟（从leader master挂掉开始算），这个过程只会推迟调度新的application，master挂掉之前就运行的application不受影响

    配置：
        如果要启用基于zookeeper 的HA 方案，要修改spark_env.sh的SPARK_DAEMON_JAVA_OPTS选项：
            1.spark.deploy.recoveryMode     设置为ZOOKEEPER
            2.spark.deploy.zookeeper.url    zookeeper集群的url
            3.spark.deploy.zookeeper.dir    zookeeper用来存储恢复状态的目录，默认为/spark
    如果在集群中启用了多个master却没有正确配置zookeeper，当恢复master时会失败，因为各个master都无法发现其他master，都认为自己是leader，所有的master会自顾自进行资源调度
    启动zookeeper集群之后，可以单点的启动master，master会被动加入master集群，可以在任何时间被移除

    为了调度新的application或者想集群中添加worker节点，他们需要知道leader master的ip地址，可以通过传递一个master列表给sparkContext来实现。
    将sparkContext连接的地址指向spark://host1:port1,host2:port2
    sparkContext会去尝试注册所有的master，直到找到一个可用的

    一旦application注册成功，会把信息保存在zookeeper中，如果故障发生，new leader会去通知之前注册过得所有应用程序和worker，通知他们master改变了。

    用于new master可以在任何时候创建，唯一需要担心的就是新的application和worker可以找到并注册到master，一旦注册就不用担心他们了



    zookeeper HA 是实现生产环境HA的最佳方式，但是如果不想用zookeeper自动恢复，想要手动重启，就要用filesystem模式————当application和worker都注册到master之后，master会将信息写入指定的文件系统目录中，以便master重启时，可以从文件系统中恢复注册的应用程序和worker状态

    配置：要启用这种filesystem HA模式，需要在spark-env.sh中设置SPARK_DAEMON_JAVA_OPTS:
        1.spark.deploy.recoveryMode    filesystem
        2.spark.deploy.recoveryDirectory    必须是master可访问的目录
    通过stop-master.sh脚本杀掉一个master进程是不会清理它的恢复状态的，所以当你重启一个新的master进程时，它会进入恢复模式。

    
    
    
    
spark web UI
    每提交一个spark作业并且启动sparkContext之后，都会启动一个对应的spark web UI，默认情况下spark web UI的访问地址是driver进程所在节点的4040端口
    如果一个节点上有多个driver，端口默认自增1

    这些信息默认情况下仅仅在作业运行期间有效并且可以看到。一旦作业完毕，那么driver进程以及对应的web ui服务也会停止，我们就无法看到已经完成的作业的信息了。
    如果要在作业完成之后，也可以看到其Spark Web UI以及详细信息，那么就需要启用Spark的History Server。
