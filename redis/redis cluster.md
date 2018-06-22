# redis集群

## sentinel
sentinel(哨兵)是redis高可用的解决方案：由一个或者多个sentinel的实例组成的sentinel系统可以监视任意多个主服务器及其从服务器。当检测到主服务器进入下线状态时，从其所属的从服务器中选取一个作为主服务器。  
sentinel的最后一步是向监控的主服务器创建网络连接，sentinel将成为主服务器的客户端。它可以向主服务器发送命令，也可以从回复中获取相关信息。  

### 检测下线状态
- 主观下线  
	默认情况下，sentinel会每秒向与它创建了命令连接的实例发送PING消息。通过PING命令的回复状态判定是否在线。
	1. 有效回复
	2. 无效回复
	超过了down-after-milliseconds之后，sentinel判定实例进入了主观下线状态
- 客观下线
	主观下线之后，需要通过客观下线判定是否真正的下线了。
	
## 数据查询
redis cluster并不会代理查询，当查询一个不属于当前节点的key时，这个节点是怎么处理的？
会返回客户端以下信息  
```
GET msg  
-MOVED ip地址：端口
```
会导致客户端两次查找才能够找到正确的数据。
	
## 数据分片
redis cluster中并没有使用一致性hash，而是使用 **数据分片** 引入 **哈希槽(hash slot)** 来实现。  
一个redis cluster包含16384（0-16383）个slot。所有存在于cluster的数据都会被hash到这些slot中。使用 <mark>slot=CRC16(key) / 16384</mark> 来计算key属于哪个slot。  
redis一个集群中，用slots数组表示该集群对应哪些slot。譬如查找5，只要判断slots[5]中的值是否为1。

## 数据迁移
可以理解为slot和key的迁移。极大的方便了集群做线性扩展，实现平滑的扩容或缩容。迁移的过程如下：   
假设有集群A和集群B，slot从A迁移到B。  
1. 把A设置成MIGRATING状态，B设置成IMPORTING状态。  
2. DUMP  
3. RESTORE  
4. DEL  


**处于MIGRATING状态时，客户端命令怎么办？**  
1. 如果key存在，则处理成功。  
2. 如果key不存在，则返回ASK。但是不改变key和slot关系，即下一次查询还是打到这台机器上。  
3. 如果是个批量操作，如果都存在，返回成功。如果都不存在，返回ASK。如果只有部分的成功，则返回TRYAGAIN。当所有key迁移成功后，客户端再试，就可以返回ASK。

**处于IMPORTING状态时，客户端命令怎么办？**  
1. 正常的命令会被MOVED重定向，如果是ASKING操作则会执行。这样已经被迁移的key可以正常操作。  
2. 如果key不存在则新建。  


