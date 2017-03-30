---
layout: post
title: mgo的session与连接池
date: 2017-3-30 22:22:06 +0800
comments: false
---

#### 数据结构

```go
type mongoServer struct {
	sync.RWMutex
	Addr          string
	ResolvedAddr  string
	tcpaddr       *net.TCPAddr
	unusedSockets []*mongoSocket
	liveSockets   []*mongoSocket
	closed        bool
	abended       bool
	sync          chan bool
	dial          dialer
	pingValue     time.Duration
	pingIndex     int
	pingCount     uint32
	pingWindow    [6]time.Duration
	info          *mongoServerInfo
}
```
&#160; &#160; &#160; &#160;对应一个mongodb实例，其中liveSockets代表被session占用的连接，unusedSockets代表当前未使用的连接。加起来就是mgo中到对应服务器的连接池

```go
type Session struct {
	m                sync.RWMutex
	cluster_         *mongoCluster
	slaveSocket      *mongoSocket
	masterSocket     *mongoSocket
	slaveOk          bool
	consistency      Mode
	queryConfig      query
	safeOp           *queryOp
	syncTimeout      time.Duration
	sockTimeout      time.Duration
	defaultdb        string
	sourcedb         string
	dialCred         *Credential
	creds            []Credential
	poolLimit        int
	bypassValidation bool
}
```
&#160; &#160; &#160; &#160;Session代表用户使用的连接实例。核心成员是masterSocket和slaveSocket，存储该session到mongodb replica集群中primary或secondary服务器的连接。这两个指针有没有值，取决于Session的mode。

#### 每种session的行为

&#160; &#160; &#160; &#160;mgo共有八个对session mode的常量定义

```go
const (
	// Relevant documentation on read preference modes:
	//
	//     http://docs.mongodb.org/manual/reference/read-preference/
	//
	Primary            Mode = 2 // Default mode. All operations read from the current replica set primary.
	PrimaryPreferred   Mode = 3 // Read from the primary if available. Read from the secondary otherwise.
	Secondary          Mode = 4 // Read from one of the nearest secondary members of the replica set.
	SecondaryPreferred Mode = 5 // Read from one of the nearest secondaries if available. Read from primary otherwise.
	Nearest            Mode = 6 // Read from one of the nearest members, irrespective of it being primary or secondary.

	// Read preference modes are specific to mgo:
	Eventual  Mode = 0 // Same as Nearest, but may change servers between reads.
	Monotonic Mode = 1 // Same as SecondaryPreferred before first write. Same as Primary after first write.
	Strong    Mode = 2 // Same as Primary.
)
```
1. Primary、Strong
缓存一个master socket，若没有缓存则从master server取一条连接
1. Secondary
缓存一个slave socket，若没有缓存则从slave server取一条连接
1. Eventual
不缓存连接，每次操作根据操作类型从对应服务器实例取一条连接

&#160; &#160; &#160; &#160;操作单个Session实例，有缓存连接则用缓存连接，无缓存连接则从连接池取连接，若连接池unusedSockets中也无可用连接，则申请新的连接。因此，在生产上使用全局Session设置成Eventual时，高并发情况下，会快速产生很多的到数据库的连接，（而mgo自身又没有实现释放空闲连接）。若使用全局的strong session，writeOp又通过同一条链路串行写到mongodb，这在mmapv1引擎的mondgo3.0中表现很差，比十条链路的并发写性能相差一倍以上。
&#160; &#160; &#160; &#160;网上多数介绍mgo用法的文章都使用了SessionCopy的做法。
```go
func foo() {
  session := globalSession.Copy()
  defer session.Close
}
```
&#160; &#160; &#160; &#160;Copy操作首先会放弃缓存的链路，转而在连接池里寻找一条连接并且自己缓存。也就是说，这种做法在foo遭遇大并发的情况下，依然会频繁地操作缓冲池，也很容易对mongodb产生连接数上的压力。

#### 结论：

&#160; &#160; &#160; &#160;单纯使用mgo的session不能很好地应对并发。会缓存连接的单个Session，很多时候只能走一条链路，不缓存的Eventual mode虽然是并发操作，但又对数据库资源缺乏保护。因此，最精确的控制方式，我认为是实现带缓存Session实例的pool，既能使操作一定程度上并发，又不会对数据库资源有过分冲击。

&#160; &#160; &#160; &#160;这里([github.com/JodeZer/mgop](http://github.com/JodeZer/mgop/))实现了一个简单的session pool，以最基本的轮询方式，每次acquire都取出一个session（实际就是一条到primary的连接)。实现了负载。

&#160; &#160; &#160; &#160;有空再写如何通过实现[mongodb url标准](https://docs.mongodb.com/v3.2/reference/connection-string/)中`minPoolSize`和`maxIdleTimeMS`来释放空闲连接
