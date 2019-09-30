---
title: Hadoop 3.0 纠删码技术（Erasure Coding）
tags: [HDFS,EC]
---

## 背景
随着大数据技术的发展，HDFS作为Hadoop的核心模块之一得到了广泛的应用。为了数据的可靠性，HDFS通过多副本机制来保证。在HDFS中的每一份数据都有两个副本，1TB的原始数据需要占用3TB的磁盘空间，存储利用率只有1/3。而且系统中大部分是使用频率非常低的冷数据，却和热数据一样存储3个副本，给存储空间和网络带宽带来了很大的压力。因此，在保证可靠性的前提下如何提高存储利用率已成为当前HDFS面对的主要问题之一。

Hadoop 3.0 引入了纠删码技术（Erasure Coding），它可以提高50%以上的存储利用率，并且保证数据的可靠性。

纠删码技术（Erasure coding）简称EC，是一种编码容错技术。最早用于通信行业，数据传输中的数据恢复。它通过对数据进行分块，然后计算出校验数据，使得各个部分的数据产生关联性。当一部分数据块丢失时，可以通过剩余的数据块和校验数据计算出丢失的数据块。

## 原理
Reed-Solomon（RS）码是存储系统较为常用的一种纠删码，它有两个参数k和m，记为RS(k,m)。如下图所示，k个数据块组成一个向量被乘上一个生成矩阵（Generator Matrix）GT从而得到一个码字（codeword）向量，该向量由k个数据块和m个校验块构成。如果一个数据块丢失，可以用(GT)-1乘以码字向量来恢复出丢失的数据块。RS(k,m)最多可容忍m个块（包括数据块和校验块）丢失。

![ec1](/img/ec1.png)

比如：我们有 7、8、9 三个原始数据，通过矩阵乘法，计算出来两个校验数据 50、122。这时原始数据加上校验数据，一共五个数据：7、8、9、50、122，可以任意丢两个，然后可以通过算法进行恢复。

![ec2](/img/ec2.png)

## HDFS EC 方案
传统模式下HDFS中文件的基本构成单位是block，而EC模式下文件的基本构成单位是block group。以RS(3,2)为例，每个block group包含3个数据块，2个校验块。
### 连续布局（Contiguous Layout）
文件数据被依次写入块中，一个块写满之后再写入下一个块，这种分布方式称为连续布局。

优点：
- 容易实现
- 方便和多副本存储策略进行转换

缺点：
- 需要客户端缓存足够多的数据块   
- 不适合存储小文件

![ec3](/img/ec3.png)

### 条形布局（Striping Layout）
条（stripe）是由若干个相同大小的单元（cell）构成的序列。文件数据被依次写入条的各个单元中，当一个条写满之后再写入下一个条，一个条的不同单元位于不同的数据块中。这种分布方式称为条形布局。

优点：
- 客户端缓存数据较少
- 无论文件大小都适用

缺点：
- 会影响一些位置敏感任务的性能，因为原先在一个节点上的块被分散到了多个不同的节点上
- 和多副本存储策略转换比较麻烦

![ec4](/img/ec4.png)

## HDFS EC 开发计划
整个HDFS EC项目主要分为两个阶段：

1. 用户可以读、写一个条形布局（Striping Layout）的文件；如果该文件的一个块丢失，后台能够检查出并恢复；如果在读的过程中发现数据丢失，能够立即解码出丢失的数据从而不影响读操作。
2. 支持将一个多副本模式（HDFS原有模式）的文件转换成连续布局（Contiguous Layout），以及从连续布局转换成多副本模式。

由于条形布局无论文件大小，都可以节省存储空间，所以HDFS优先考虑支持条形布局。目前第一阶段 [HDFS-7285](https://issues.apache.org/jira/browse/HDFS-7285) 已经实现，第二阶段 [HDFS-8030](https://issues.apache.org/jira/browse/HDFS-8030) 正在进行中。

![ec5](/img/ec5.png)

![ec6](/img/ec6.png)

## HDFS EC 设置方式
```
Usage: bin/hdfs ec [COMMAND]
          [-setPolicy -path <path> [-policy <policy>] [-replicate]]
                为指定目录设置存储策略
          [-unsetPolicy -path <path>]
                取消为指定目录设置的存储策略
          [-getPolicy -path <path>]
                获取指定目录的存储策略
          [-enablePolicy -policy <policy>]
                开启一个存储策略
          [-disablePolicy -policy <policy>]
                关闭一个存储策略
          [-addPolicies -policyFile <file>]
                增加一个存储策略
          [-removePolicy -policy <policy>]
                移除一个存储策略
          [-listPolicies]
                显示所有的存储策略（如：RS-6-3-1024k、RS-3-2-1024k、XOR-2-1-1024k等），只有enabled状态的策略，可以被setPolicy命令使用
          [-listCodecs]
                显示当前系统支持的全部编解码器（如：RS、XOR等）
```
**注意**
1. EC存储策略下的文件，不支持append()、hflush()、hsync()
2. 不同存储策略的目录或文件，目前没有提供转换的方法。比如想把一个以RS(3,2)存储的文件，转换为RS(6,3)存储策略，或者三副本存储策略，目前并没有转换方法，但可以通过把文件复制到相应存储策略的目录来达到这个目的（比如cp、distcp）

## 总结
Hadoop纠删码技术相比于传统多副本方案，可以在保证可靠性的前提性，节省50%以上的存储空间；但当有块丢失的时候，修复过程需要连接多个DataNode，对网络带宽、磁盘IO、CPU都消耗过大，所以一般建议用于存储冷数据。

## 参考
[1] http://hadoop.apache.org/docs/r3.0.0-beta1/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html  
[2] http://blog.cloudera.com/blog/2015/09/introduction-to-hdfs-erasure-coding-in-apache-hadoop  
[3] https://issues.apache.org/jira/secure/attachment/12697210/HDFSErasureCodingDesign-20150206.pdf  
[4] https://www.iteblog.com/archives/1684.html  
[5] http://geek.csdn.net/news/detail/77338  
