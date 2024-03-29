## 第一章节(数据结构与基础)
书中提到一个概念：语义抽象
> 举一个例子：我们只需要用SQL语句,就能直接拿数据,屏蔽了实现细节。

列举了两种数据查询类型

|引擎类型|	请求数量|	数据量|	瓶颈|	存储格式|	用户|	场景举例|	产品举例
|---|---|---|---|---|---|---|---|
|OLTP|	相对频繁，侧重在线交易|	总体和单次查询都相对较小|	Disk Seek|	多用行存|	比较普遍，一般应用用的比较多|	银行交易|	MySQL
|OLAP|	相对较少，侧重离线分析|	总体和单次查询都相对巨大|	Disk Bandwidth|	列存逐渐流行|	多为商业用户|	商业分析|	ClickHouse

OLTP主要分两种存储引擎

|流派|	主要特点|	基本思想|	代表
|---|---|---|---|
|log-structured| 流|	只允许追加，所有修改都表现为文件的追加和文件整体增删	变随机写为顺序写|	Bitcask、LevelDB、RocksDB、Cassandra、Lucene
|update-in-place| 流|	以页（page）为粒度对磁盘数据进行修改	面向页、查找树|	B 族树，所有主流关系型数据库和一些非关系型数据库



介绍了bitcask、lsm-tree、B+ tree等多种结构实现的数据库, 本人对于这部分知识有谢谢了解,但总觉得不算透彻。于是产生了重复造轮子的想法

bitcask实现 

[https://github.com/cccvip/ZDB](https://github.com/cccvip/ZDB)

lsm-tree实现



