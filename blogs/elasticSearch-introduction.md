#摘要#

对于Elastic Search的初始印象就是全文本搜索，与SOLR是竞品。它与其他的存储型数据库有何区别，为什么其他数据库已经能够提供文本搜索功能了，还需要ES等一系列问题都是心中的困惑。这篇文章主要就是总结这些问题以及Elastic Search的概括介绍


#ES资源#

ES 下载：https://www.elastic.co/downloads/elasticsearch
ES 文档: https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
ES 源码：https://github.com/elastic/elasticsearch
ES 中国论坛: https://elasticsearch.cn/

#ES介绍#

Elastic 是一个基于ApacheV2的开源系统，能够提供分布式，实时的文档索引并提供在线分析。Schema free.

##专有名词##
**映射(Mappings)**

就是关系型数据库中的schemas，不同的地方在于ES不用提前明确的指定数据的类型。

**索引（indexes）**

这边的索引与通常意义上的索引有些不同，这边的索引是ES的专业名词，表示的意义就是关系型数据库中database,是一个逻辑命名。

**类型（Types）**

就是关系型数据中的表的意思

## 集群架构 ##

**主节点**

集群中只有1个，是整个集群的协调节点。down掉以后，会自动从选举出一个。负责集群level的操作，比如创建删除一个索引。跟踪哪些节点时集群的一部分，决定哪个shard分配给哪个节点。

**数据节点**

保存Lucene的索引分片，构成ES的分布式索引

**接待节点（ingest node）**

这个是可以执行预处理的pipeline工作，

**只协调节点**

只负责路由，处理ｓｅａｒｃｈ　ｒｅｄｕｃｅ　阶段

默认情况下，一个节点这些角色都有，当集群扩大了，想划分职责了，可以ｄｉｓａｂｌｅ到某些角色，让某个节点只做一件事。

> 吐槽：
> 觉得这方面是ＥＳ做的不好，没有一个清晰明确的集群扩展拓扑图。比如说在数据规模比较小的时候，所有的节点都是各个角色都有，到数据量大的时候再划分特定的角色，这个实际操作不切实际，什么时候就可以确定数据量大，什么时候特定的角色就比所有角色都有效率高。而且在切换过程中需要怎么切换，ES不支持resharding,和分片分裂。所以索引也没办法迁移了，如何能够进行角色转换。而且一旦划分出coordinate node,客户端连接节点还得更新。


#ES初识#

**分片**

没有基于节点层面的分片，这点与mongo不同，与cassandra的partition类似。通过id hash来分片的。shard number 需要在创建index一开始去指定，后期如果要更改需要reindex


	{
	  "settings": {
	    "number_of_shards":   5, 
	    "number_of_replicas": 2
	  }
	}

**分布式查询**

分布式查询就是查询各个shard,然后对结果进行组合，在各个shard上的查询可以并行进行，但是如果shard number 比节点数多的，在同一个节点上就必须得串行执行了，效率就会降低。一般单个节点上的shard不要超过2个，也就是说index 的shard number 设置为10，节点水平扩展到20个就差不多了。

官方对于不支持shard分裂理由

1. shard分裂就是一个reindex data,这个过程相比较从一个将shard从一个节点copy到另外一个节点会更重
2. 分裂是指数性，1分2，2分4，4分8，不会允许只扩容50%
3. shard 分裂要求你有足够的能力去保存第二份索引，通常当你意识到你需要扩展时，你可能没有足够的空间去支持分片了。


ES的reindex过程也是一个比较重的过程，同样需要有足够的空间去完成，但是至少你能够控制新的索引中的索引数量。

> 吐槽:这个分片机制，ES设计的实在是太烂了，对于使用者来说，需要的是自动分片，而不是像MySQL那样的自己实现，自动分片的含义不仅仅是初始的分片，当然也包括对扩展的要求。使用者怎么知道自己的数据规模增长的上限，怎么知道在开始的时候使用多少分片，reindexing在数据量大的情况下，完全是灾难，当然应该把这部分工作放在日常解决。

**索引**

ES的索引是倒排索引，这个与一般的B-Tree索引不同，这也是为什么ES是全文本搜索引擎，B-Tree索引的话是针对字段的一个排序，对于模糊查询比如说like,没法进行查索引，而倒排索引的话，是一个反向索引，首先是进行分词，然后记录词会哪些字段中出现，这是为什么一般的数据库提供不了全文本搜索引擎的原因。

**NoSQL数据库**

ES也有人拿来作为NoSQL数据库，因为默认保存了原始数据，而且有transaction log,保证不丢数据，另外的query比较丰富，一些group by都可以通过aggregation 来实现。所以有些用户拿来做BI分析


#总结#
ES使用起来比较简单，容易上手。提供RESTFUL API接口，JSON文档存储，调用起来方便。作为全文本搜索是一个选择，与solr的区别暂时不清楚。作为NoSQL数据库存储，查询丰富在数据量不多，集群规模不大的情况下可以选用，但是集群扩展能力不行，分片策略有缺陷。不易在大规模数据存储时使用。

#参考#

https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html

https://www.elastic.co/guide/en/elasticsearch/guide/current/overallocation.html