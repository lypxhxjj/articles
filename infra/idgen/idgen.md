# 号段模式
### 1. 表设计
号段模式需要一个表：
```

type IdgenSegment struct {
	ID         int       `json:"id" bdb:"id"`
	BizTag     string    `json:"biz_tag" bdb:"biz_tag"`
	MaxID      int64     `json:"max_id" bdb:"max_id"`
	Step       int       `json:"step" bdb:"step"`       
	UpdateTime time.Time `json:"update_time" bdb:"update_time"`
	CreateTime time.Time `json:"create_time" bdb:"create_time"`
}
```
解释下里面的几个字段：
1. BizTag，自定义tag，代表一个场景下的id生成。目前定义为 group + service + namespace。服务下多个ns代表一个服务下可以使用多个idgen。
2. MaxID：目前可以分配的最小ID，初始化为1；
3. Step：MaxID增大的步长，即一次取走多少个ID。初始化默认是1，但是会随着使用情况调整，这个后续详细说明。

### 2. 配置设计
```
type SegmentModeConfig struct {
	GroupName string `properties:"groupName"`
	ServName  string `properties:"servName"`
	Namespace string `properties:"namespace"`
	// 默认50ms。
	Timeout    int64 `properties:"timeout"`
	DefaultQps int64 `properties:"defaultQps"`
}
```
解释下每个字段：
1. 前三个字段用于获取唯一标识，即group + service + namespace;
2. timeout：超时，默认是50ms，平台上默认没有给出配置。这个字段的含义是，超过50ms还没有拿到id，会直接返回错误。这个timeout与实现息息相关，因为需要考虑为啥拿个id还会有延时。直接设置context.Deadline不好吗？
3. qps：用户配置的。qps在代码中用于计算step，即一次取出多少号段出来。虽然是不精确的计算。

### 3. 如何进行的配置初始化。
现状：每次启动从apollo中拉取配置解析，之后挨个配置使用insert ignore插入数据库。

可优化：idgen需要单独提供配置注册的功能，只在配置注册的时候入库就好了，没必要每次启动都尝试入库。

一个学习的点：insert ignore适用于存在主键、唯一键的场景，如果没有唯一键，想要实现条件插入，可以参考代码中使用dual表 + insert + select语法：
```
INSERT INTO idgen_segment (biz_tag, step) 
SELECT ?, ?
FROM dual 
WHERE NOT EXISTS (
	SELECT *
	FROM idgen_segment
	WHERE biz_tag = ?
);
```

### 4. 单机内如何使用生产者消费者模型消费id
由于获取id操作是在gprc单独的协程下，把id源想象成一个对象的话，即很多协程并发访问这个对象。

所以第一种方法：加锁。理解上，如果qps不大，加锁完全可以解决这个问题。

第二种方法：使用channel。channel的优点是，将id生产者和id消费者的逻辑解耦了。这样生产者和消费者都可以有很多自己的扩展逻辑可以做，从逻辑上是彻底分开的！

所以，每个idgen都会创建一个协程用于id生产。反过来消费逻辑就比较简单了，grpc请求过来，直接读取channel就好了。难点在于如何生产，以保证消费尽量不阻塞。

类比锁和channel：
1. channel实现了生产者和消费者的解耦，使得各自可以扩展，互不影响；
2. 锁会导致多写问题。同一时刻，多个读发现没有数据，会触发多个写，多写不好维护。

### 5. 多物理实例下id生产者如何生产id？
号段模式基于的是db来生成id。而多物理实例共享同一个db，就会存在并发读写的问题，这个问题如何解决？

生产id的本质在于修改idgen_segment表，所以无论怎么样，一定要修改成功之后才能拿到id。

并发写基本两种方式：
1. 加锁；
2. 乐观锁的思路；

由于我们使用的tidb版本较低，tidb本身使用的是乐观锁处理事务，所以我们也只能使用乐观锁的思路更新db。

即：
```
select * from idgen_segment where biz_tag = xxx;
update idgen_segment set max_id = xxx, step = xxx where biz_tag = xxx and max_id = xxx;
```
每次执行完成上面两个语句，检查update的返回值，如果不为1，则重试执行上面两条语句，直到返回值为1，代表更新成功。即乐观锁的思想进行并发更新数据库表。

### 6. 生产者如何尽量保证不阻塞消费者？
正常情况下的逻辑：生产者从db那获取来一批id，之后使用channel提供给消费者。一旦id用完，再去db获取新的一批，如此往复。

这样的成本：id用完之后到来的请求，都需要等到请求db完成才能获取。如果好长时间才有这么一次成本的话还好，如果流量特别高，那么很短时间内，就需要一次等待，这个成本是不可接受的。（每次等待的不是一个请求，而是一批请求）

ps下：**相比锁，使用channel的方式，可以不用考虑读引起的多写问题**。比如同一时刻，很多读都发现没有数据，都会触发写，这种多写就很难维护。

解决这个问题的方案：预取。

预取体现在两个方面：
1. 每次获取的id尽量多一点，使得消费者需要花更多的时间消费完，减少均摊等待时间；
2. 不等消费完，即进行下一次的预取。直接干掉等待db的时间成本。

所以算法是这样的：
1. 基于预估qps，申请至少15min的id数量，即qps * 15 * 60个id，即idgen_segment.step字段；
2. 一旦这一批id消费完成一半，另起一个协程，再次从第一步开始执行。
3. 此时存在两个生产者，平均下来，第二个协程消费完成一半，第一个协程的id已经消费完，第二个协程会启动第三个协程。如此往复，基本只存在2个生产者。
4. 自适应qps。在执行第二步的时候，每次都重新计算下qps，以此来更好地执行第一步。（当然qps很小的话，一般会取一个范围[100, 2000]），如何重新计算，我不想理解代码中的内容，其实简单在生产或者消费的时候加个原子计数就可以了。可以比较精确地计算qps，而不是像代码中的很长时间的粗略计算。

伪代码如下：
```
func BeginProduce() {
	step := QPSManager.GetCurrentQPS() * 15 * 60
	idBegin := applyIdsFromDB(step)
	produce(idBegin, idBegin + step / 2)
	go BeginProduce() 							// 递归调用
	produce(idBegin + step / 2, idBegin + step)
}
```

### 7. 总结下
号段模式，总体来说使用的是生产者和消费者模型来实现的。

消费者非常简单，只是简单地读取 + 超时处理。麻烦点的是生产者。

生产者主要使用两种预取方式避免消费者等待；同时使用乐观更新的方式从db申请id。

其中表设计和配置设计参考第一点和第二点。