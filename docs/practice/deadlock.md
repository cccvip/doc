# 死锁问题
> 记一次数据库产生的死锁问题
## 时间节点
> 在周五早上10点左右,测试同学反馈怎么页面加载如此缓慢,有些功能甚至都用不了。我瞬间就登陆到日志平台查看,发现是出现大量的SQL异常,获取事务等待超时情况。
> 心想,应该是数据库出现了问题,赶紧找DBA同学,拿完整的数据库日志。然后重启应用,先保证功能可用
> 由于一些原因,完整的日志就不便截图。
## 数据库错误日志分析

### 数据库隔壁级别: RR
默认使用RR的隔离级别,通过增加间歇锁(gap-lock)。

```sql
crate table student {
  user_id int,
  class_id int,
  rank int
  unique key (user_id,class_id)
}
```
### 数据库错误日志:
- insert WAITING FOR lock_mode X insert intention waiting

> insert student(user_id,class_id,rank) values(1,1,1)

- insert WAITING FOR lock_mode X insert intention waiting, HOLDS lock_mode S

> insert student(user_id,class_id,rank) values(1,1,2)

> 通过日志,我们可以得到几件事
- Insert语句,锁升级。从隐藏锁升级成了显示锁(LOCK_REC_NOT_GAP | LOCK_X)
因为user_id,class_id组成了一个唯一索引,此时执行SQL,会出现一个锁升级。
LOCK_REC_NOT_GAP: 锁带上这个 FLAG 时，表示这个锁对象只是单纯的锁在记录上，不会锁记录之前的 GAP
LOCK_X: 排它锁,意思就是这一行被占了。

- 根据文档,唯一索引重复,必须进行一次当前读，加的锁是Next-Key,此时锁(LOCK_GAP|LOCK_S|LOCK_WAIT)
LOCK_GAP:
> 表示只锁住一段范围，不锁记录本身，通常表示两个索引记录之间，或者索引上的第一条记录之前，或者最后一条记录之后的锁。可以理解为一种区间锁，一般在RR隔离级别下会使用到GAP锁。
你可以通过切换到RC隔离级别，或者开启选项innodb_locks_unsafe_for_binlog来避免GAP锁。这时候只有在检查外键约束或者duplicate key检查时才会使用到GAP LOCK。

LOCK_S:
>  其他事务进行读取会请求共享锁

LOCK_Wait:
> 由于第一个事务未进行提交,出现锁等待

- 在插入意向锁的时候与Gap锁,产生了冲突导致死锁情况发生


## 业务日志分析
数据库数据

| userId   | classId |
|  ----  | ----  |
| 101  | 102 |
| 101  | 107 |

> 错误的SQL找到,还需要找到关键错误原因点。当前有一个场景,针对表的数据进行insert或者Update操作,
伪代码如下
```shell
//数据库操作
long count="select * from student where user_id=101,class_id=109"
if(count>0){
 dao.update("student","rank=?",X)  
}else {
 dao.insert("insert student(user_id,class_id,rank) values(101,109,?)",X)  
}
//执行Redis操作
//执行MQ操作
//RPC操作
```
由于上游服务崩了,导致接口被高频调用,结合前面的日志,进行一波分析

- 多个事务同时在进行insert操作,第一个事务获取的锁,为Lock_X,其他事务只能申请获取Lock_S。
- RR的隔离级别情况,无论是select操作,或者insert ... ON duplicate都申请间歇锁, 此时的gap lock范围为
- 第一个事务由于异常原因执行Rollback操作
- 其他事务,此时进行insert操作,插入意向锁的时候会检查是否在gap范围内
  
  | session1   | session2 | session3 |
  |  ----  | ----  | ----|
  | insert student(user_id,class_id,rank) values(101,109,1)  | ... | ...|
  | ...  | insert student(user_id,class_id,rank) values(101,109,2) | ... |
  | ...  | ... | insert student(user_id,class_id,rank) values(101,109,3) |
  | rollback  | ... |  |
  | ...  | Gap锁(101-107,101-109) |  |
  | ...  | ... | Gap锁(101-107,101-109) |
  | ...  | 等带事务3释放 | 等待事务2释放 |

- 意向X锁(LOCK_INSERT_INTENTION标记的间隙锁LOCK_GAP写锁LOCK_X)与S锁不兼容

## 总结
[mysqlInsert文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html)
```shell
如果发生重复键错误，则会在重复索引记录上设置共享锁。如果另一个会话已经拥有排它锁，
则如果多个会话尝试插入同一行，则使用共享锁可能会导致死锁。
如果另一个会话删除该行，则可能会发生这种情况。假设一个InnoDB表 t1具有以下结构：
CREATE TABLE t1 (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;
现在假设三个会话按顺序执行以下操作：

第一节：

START TRANSACTION;
INSERT INTO t1 VALUES(1);
第二节：

START TRANSACTION;
INSERT INTO t1 VALUES(1);
第三节：

START TRANSACTION;
INSERT INTO t1 VALUES(1);
第一节：

ROLLBACK;

会话 1 的第一个操作获取该行的排他锁。会话 2 和 3 的操作都会导致重复键错误，并且它们都请求该行的共享锁。当会话 1 回滚时，它会释放其对该行的独占锁，并且会话 2 和 3 的排队共享锁请求将被授予。此时，会话 2 和会话 3 发生死锁：由于对方持有共享锁，双方都无法获取该行的排他锁。

如果表已包含键值为 1 的行并且三个会话按顺序执行以下操作，则会出现类似的情况：

第一节：

START TRANSACTION;
DELETE FROM t1 WHERE i = 1;
第二节：

START TRANSACTION;
INSERT INTO t1 VALUES(1);
第三节：

START TRANSACTION;
INSERT INTO t1 VALUES(1);
第一节：

COMMIT;
会话 1 的第一个操作获取该行的排他锁。会话 2 和 3 的操作都会导致重复键错误，并且它们都请求该行的共享锁。当会话 1 提交时，它会释放其对该行的独占锁，并且会话 2 和 3 的排队共享锁请求将被授予。此时，会话 2 和会话 3 发生死锁：由于对方持有共享锁，双方都无法获取该行的排他锁。
```
[InnoDB隐式锁](http://mysql.taobao.org/monthly/2020/09/06/)
[死锁日志错误分析参考](https://www.aneasystone.com/archives/2018/04/solving-dead-locks-four.html)
