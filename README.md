
## 一、使用场景


**试想一个场景，有一个配置服务系统，里面存储着各种各样的配置，比如直播间的直播信息、点赞、签到、红包、带货等等。这些配置信息有两个特点：**


**1、并发量可能会特别特别大**，试想一下，一个几十万人的直播间，可能在直播开始前几秒钟，用户就瞬间涌入进来了，那么这时候我们的系统就得加载这些配置信息。此时请求量就如同洪峰一般，一下子就冲击进入我们的系统。


**2、这些配置通常都是只需要读取**，在B端（管理后台）设置好的，一般直播开始后，修改的频率很低。


那么面对上述的业务场景，假设我们的目标是**扛住3wQPS，你们会选用什么技术架构和方案呢？**


**1、直接查数据库**，例如MySQL、Doris之类的关系型数据库。很明显这肯定扛不住，一般关系型数据库能让扛个几千就基本上到头了。


**2、使用单机版Redis**。理论上是可以的，腾讯云（下图）和一些Redis官方的数据，都说理论上高配置版本的单机Redis能抗住10W\+的QPS。可是理论毕竟是理论，实际上工作中，我使用Redis做过许多压测，都表明单机Redis上了两万多之后就性能会出现瓶颈，压测就压不上去。（当然，或许是我司的Redis还没升到顶配？）
![](https://img2024.cnblogs.com/blog/2133945/202410/2133945-20241027221456065-195116413.png)


**3、使用集群版Redis**，当然是可以解决这个问题，就是成本有点点高咯，公司不差钱完全可以使用这个方案。


**4、本地缓存，就是本文的重点，完美地解决这个问题。所谓本地缓存就是将这些所需要获取的数据存储在服务器的内存中**。服务器读取本地缓存的速度理论上来说没有上限，看服务器物理机的配置。但其下限就远比MySQL和单机Redis之类的高好几倍了，一台2核4G的Linux服务器，估计也至少10W\+QPS起步。我曾经在本地的Windows系统做过压测（四核八线程，16G），就达到过100W\+的QPS。换到同等配置的Linux系统上，那就更不用说了。


## 二、技术方案


既然选用了本地缓存这个策略，那么我们怎么设计这个本地缓存的技术方案呢？


![本地缓存数据流转图](https://img2024.cnblogs.com/blog/2133945/202410/2133945-20241027221605994-1187300936.png)


1、如上图所示，我们客户端获取数据首先会读取本地缓存，**如果本地缓存没有数据就会读取Redis数据，如果Redis没有就会读取DB数据。**


**2、需要注意的是，本地缓存和DB之间一般还会加入Redis这一层缓存**。这是因为本地缓存设置好后就无法再更新了（除非重启服务器），而Redis缓存我们是可以在DB有更改后，随时更新。这个也很好理解，因为Redis是有单独的Redis服务器，而本地缓存就只能在那台机器上更新和设置，**但实际项目中，设置本地缓存的DB数据源的机器和使用本地缓存的机器大概率都不在同一个系统中**。**所以我们本地缓存的时间都设置得很短，大部分都是秒级的，一般不会超过1分钟，比如1秒、2秒**... 。而Redis这个缓存时长明显可以设置长一些，比如半小时、1小时...。


## 三、如何更新本地缓存


上面讲了，**本地缓存最不好的地方就是更新问题，因为很可能设置本地缓存的DB数据源的系统和使用本地缓存的系统不是同一个，无法在DB数据更新的时候就同步更新本地缓存。但是实际使用的时候很可能就需要这种场景**，就是在更新数据源的时候去更新本地缓存。举个例子：


我们设置配置A的DB数据源的系统是一个API系统，但现在有一个脚本系统，需要根据某个配置A，去处理C端的一些行为数据，判断是否满足该配置A，然后进行对应的业务处理。好了，现在C端的行为数据量是非常庞大的，可以说是海量数据，**平均每秒钟有五十万的数据通过kafka推送过来**。此时我们就必须得用**本地缓存**存储配置A的信息了，**才能抗的住这个流量洪峰**。但是这是一个脚本系统啊，**我们更新配置A的DB信息是在对应的API系统中的**。那怎么办呢？


有几个方法：
**1、在脚本系统中维护一个脚本，每隔一段时间就去读取MySQL的数据**，然后更新到本地缓存。但这个得综合评估下时间和MySQL的性能，因为要一直扫表。


**2、拉取MySQL的binlog日志**，每当数据有变更时，kafka推送数据到下游。脚本监听kafka数据，当收到kafka数据是就更新配置A的本地缓存。但这个也得注意，因为脚本系统一般会同时起很多个服务，所以得注意有多少个服务就得设置多少个消费者组，因为要保证脚本系统的每个服务都消费到kafka对应的DB更新数据，进而更新各自机器上的本地缓存。


**3、使用Redis的发布订阅功能**，上游api有更新配置信息时就去发布信息，每个脚本服务都去订阅该信息，一有消息就去更新自己机器上的本地缓存。但这也有个弊端，Redis的发布订阅功能是没有确认机制的，所以可能某个脚本服务没收到信息导致没更新本地缓存，然后就出现bug了。demo如下：
**（1）发布者：**



```
package main

import (
	"context"
	"fmt"
	"github.com/go-redis/redis/v8"
	"time"
)

var ctx = context.Background()

// 发布订阅功能
// 发布者发布，所有订阅者都能接收到发布的消息。注意区分消息队列，消息队列是发布者发布，只有一个订阅者能抢到。
func main() {
	rdb := redis.NewClient(&redis.Options{
		Addr:     "localhost:16379",
		Password: "123456",
	})

	i := 0
	for {
		// 模拟数据更新时发布消息
		rdb.Publish(ctx, "money_updates", "New money value updated "+fmt.Sprintf("%d", i))
		fmt.Println("Message published " + fmt.Sprintf("%d", i))
		time.Sleep(5 * time.Second)
		i++
	}

}


```

**（2）订阅者：**



```
package main

import (
	"context"
	"fmt"
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

func main() {
	rdb := redis.NewClient(&redis.Options{
		Addr:     "localhost:16379",
		Password: "123456",
	})

	pubsub := rdb.Subscribe(ctx, "money_updates")
	defer pubsub.Close()

	// 等待消息
	for {
		msg, err := pubsub.ReceiveMessage(ctx)
		if err != nil {
			fmt.Println("Error receiving message:", err)
			return
		}
		fmt.Println("Received message:", msg.Payload)
	}
}


```

**4、跟方法1类似，只是可以把修改的配置A信息推送到Redis中**，然后脚本去扫描Redis信息，有则更新本地缓存。其实就是延迟队列。但这个就得上游的配置A增删改都要写入这个Redis，有时候增删改的口子太多，其实实施起来也比较困难。


如上所述，基本上更新本地缓存没有一个很合适、很高效的方法，只能选取其中一个比较符合自己业务场景的方法。


## 四、本地缓存常用类库


go如何使用本地缓存呢？


**1、可以自己实现一个本地缓存，一般可以使用LRU（最近最少使用）算法**。下面是自己实现的一个本地缓存的demo。



```
package main

import (
	"container/list"
	"fmt"
	"sync"
	"time"
)

type Cache struct {
	capacity int
	cache    map[int]*list.Element
	lruList  *list.List
	mu       sync.Mutex // 确保线程安全
}

type entry struct {
	key        int
	value      int
	expiration time.Time // 过期时间
}

// NewCache 创建新的缓存
func NewCache(capacity int) *Cache {
	return &Cache{
		capacity: capacity,
		cache:    make(map[int]*list.Element),
		lruList:  list.New(),
	}
}

// Get 从缓存中获取值
func (c *Cache) Get(key int) (int, bool) {
	c.mu.Lock()
	defer c.mu.Unlock()

	if elem, found := c.cache[key]; found {
		// 检查是否过期
		if elem.Value.(entry).expiration.After(time.Now()) {
			// 移动到链表头部 (最近使用)
			c.lruList.MoveToFront(elem)
			return elem.Value.(entry).value, true
		}
		// 如果过期，删除缓存项
		c.removeElement(elem)
	}
	return 0, false
}

// Put 将值放入缓存
func (c *Cache) Put(key int, value int, ttl time.Duration) {
	c.mu.Lock()
	defer c.mu.Unlock()

	if elem, found := c.cache[key]; found {
		// 更新现有值
		elem.Value = entry{key, value, time.Now().Add(ttl)}
		c.lruList.MoveToFront(elem)
	} else {
		// 添加新条目
		if c.lruList.Len() == c.capacity {
			// 删除最旧的条目
			oldest := c.lruList.Back()
			if oldest != nil {
				c.removeElement(oldest)
			}
		}
		newElem := c.lruList.PushFront(entry{key, value, time.Now().Add(ttl)})
		c.cache[key] = newElem
	}
}

// removeElement 从缓存中删除元素
func (c *Cache) removeElement(elem *list.Element) {
	c.lruList.Remove(elem)
	delete(c.cache, elem.Value.(entry).key)
}

// 清理过期项
func (c *Cache) CleanUp() {
	c.mu.Lock()
	defer c.mu.Unlock()

	for e := c.lruList.Back(); e != nil; {
		next := e.Prev()
		if e.Value.(entry).expiration.Before(time.Now()) {
			c.removeElement(e)
		}
		e = next
	}
}

func main() {
	cache := NewCache(2)
	cache.Put(1, 1, 5*time.Second) // 设置过期时间为5秒
	cache.Put(2, 2, 5*time.Second)

	fmt.Println(cache.Get(1)) // 输出: 1 true
	time.Sleep(6 * time.Second)

	// 过期后的访问
	fmt.Println(cache.Get(1)) // 输出: 0 false
	cache.CleanUp()           // 进行清理
}


```

该代码中使用LRU算法，通过将最新的缓存移动到链表头部（最近使用）来实现这个算法。但也有一些问题，CleanUp 方法需要手动调用去清理过期缓存，并没有定期自动清理的机制。这就意味着使用者可能需要频繁调用 CleanUp，否则过期项可能会在缓存中停留较长时间。代码中也加了锁，可能还会存在并发访问时的数据一致性和性能问题等等。


所以，这种自己实现的demo不建议放在生产环境中使用，可能会存在一些小问题。比如，**之前我部门写的一个本地缓存类库，就存在一个大bug：本地缓存的内存空间不能释放，导致内存一直蹭蹭地往上涨。隔好几天内存就飙到90%，然后我们临时处理方法是：隔好几天就去重启一次脚本**...


**2、所以呢，我们还是建议去使用开源的类库**，至少有许多前辈帮我们踩过坑了，这里推荐几个star数比较高的：
（1）[**go\-cache**](https://github.com)：一个简单的内存缓存库，支持过期和自动清理，适合简单的缓存key\-value需求。（本人项目中使用比较多，方便简单，推荐）
（2）[**bigcache**](https://github.com):[milou加速器](https://xinminxuehui.org)：高性能的内存缓存库，适用于大量数据的缓存，其设计旨在减少垃圾回收的压力。
（3）[**groupcache**](https://github.com)：Google 开发的一个缓存库，支持分布式缓存和单机缓存。适用于需要高并发和高可用性的场景。


以上，就是个人使用本地缓存的一些经验了。不得不说，这玩意用着是真香，物美价廉，能扛能打。唯一美中不足的就是本地缓存不太好实时去更新，当然这个上面也给出了几个解决方案。


