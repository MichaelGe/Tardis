.. highlight:: 

MongoDB Tips
===========

Pre
____
    工作中遇到的一些问题，以及个人理解。


MongoDB适合的场景
________________
    
    1. time-series data
       适合存日志，统计数据

    #. real-time analytics
        实时的数据分析，展现

    #. document-based 
        文档型的数据都可以用mongodb存，可以动态的加字段（percona5.6也支持在线增加字段）

    #. file
        用GridFS存文件, 据我了解，有一些公司在用mongodb在存视频, 用的还不错，比起FDFS来说，它更简单

MongoDB Limits
_____________
    比较重要的是要知道 Indexs、Capped Collections 和 Operations的限制
    `更多 <http://docs.mongodb.org/manual/reference/limits/>`_

服务器初始化
___________
    
    1. 链接句柄数
        服务器一般都会修改默认参数，防止链接数不过，造成服务不可用
        echo ulimit -n 65535 >>/etc/profile
        ulimit -n 65535
    #. numactl 
        见下文NUMA
         sudo yum install numactl

NUMA
____
  
     现代得服务器一般都支持NUMA(非一致性内存访问), 在部署数据库的时候都会遇到NUMA，的问题
         echo 0 > /proc/sys/vm/zone_reclaim_mode
         sudo  numactl --interleave=all /usr/local/bin/mongod --dbpath /data/rs001-4 --port 10014 --journal --fork --logpath /data/rs001-4.log --replSet rs001 --oplogSize 10240

Oplog
_____
    Oplog是Capped Collection,它的大小很重要，尽量设的大些。
    当瞬间插入速度过快的时候，replica sets中PRIMARY的oplog变化太快，SECONDARY来不及从PRIMARY的oplog中同步数据, 会导致SECONDARY坏掉。

Index
_____
    还来不及写，困了。 


Replication、Sharding和Cluster
____________________
    
    1. Replication
        一个replica sets至少有3个mongod的instance是比较合理的，如果硬件资源比较好可以考虑4个instance一个replica sets
    #. Sharidng
        1. 一个sharding 一个replica sets 比较好，这样分片不容易挂掉
        #.  选一个好的sharding key, 最好有多维的keys

    #. Cluster
        一个Cluster 下要有多少的sharding 没有看到过相关的优化说明，但是如果你在加sharding的时候指定sharding的大小size了，
        当数据超过 n * size，你就只能再加机器和sharding。

    #. Scaling
        没有深入研究过，只说一些现象。不要一味的增加单个机器的mongod instance, 根据NUMA node的个数和资源的情况规划比较好。
        Scale horizontally（横向扩展）是比较合适的。

     
Auto sharding
____________
    在对collection 开启sharding的时候，MongoDB Balancer 会auto sharding, 这样可能会有数据的重复移动的问题
    1. Pre-split
        db.runCommand( { split : "mydb.mycollection" , middle : { shardkey : value } } )
    #. Increase chunk size
        增加chunk的大小可以提升插入的速度，但是开启balancer的时候不建议设的过大 
    #. Turn off the balancer
        use config
        db.settings.update( { _id: "balancer" }, { $set : { stopped: true } } , true );

迁移、备份与恢复
_______________
    1. 迁移与备份
       mongoexport --port 30001 -d  stats -c collection -q '{"ts": {$gt: "1386172800"}}' --out dump.json


故障排查
_______
    1. netstat -an|wc -l  
       查看句柄数
    #. strace 
        用strace可以排查高CPU, MEM使用问题


监控
___
    主要要查看mem, cpu, operater, connection, lock %, page faults, db storage
    mongostat -h 127.0.0.1  --port 27017 --discover
    可视化推荐用MongoDB官网提供的监控工具https://mms.mongodb.com



参考资料
_______
    http://docs.mongodb.org/manual/
    http://docs.mongodb.org/manual/faq/
    http://docs.mongodb.org/ecosystem/use-cases/
    http://docs.mongodb.org/manual/reference/limits/
