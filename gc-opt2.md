在引入sync包的Pool这个概念之前，先说一个场景。
写代码的时候总是避免不对对象的创建，比如new一个结构体，创建一个连接，甚至于定义一个int都属于对象。在某些特定场景下，会存在某些对象需要频繁创建销毁，这个时候会出现两种情况：

1. 创建对象需要申请内存，这个操作往往是最耗时的；
2. 因为频繁创建对象，会对GC造成较大压力，Golang中最主要想用以优化的地方；

那么Pool的出现可以干嘛呢

sync.Pool的思想就是对象池，因为创建连接、线程比较消耗资源，所有用池化的思想来解决这些问题，直接复用。

作为参照，对比下go1.12跟1.13版本的原理差异：

Go1.12
首先先看看结构体
<code>
type Pool struct {
    noCopy noCopy
    local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
    localSize uintptr // size of the local array
    New func() interface{}
}
</code>
noCopy 字段的类型也是noCopy，这个结构实际也是一个empty的结构体
type noCopy struct{}
这么做是因为，Go中没有原生禁止拷贝的方式，所以如果有的结构体，你希望使用者无法拷贝，只能指针传递保证全局唯一的话，可以这么处理，定义一个结构体叫noCopy，并实现下面两个方法
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
这么处理之后go vet就能检测出来，但是如果不检测的话代码也是能跑的。回过头，为什么我们的对象池要禁止拷贝呢，pool的使用是基于各个协程之间的，相互偷对象又加锁等，最重要的，我们GC要保证pool的字段情况，如果pool在清空之前被copy了，那pool在清空之后的拷贝对象就没有被清掉，这样一来，pool里面指针所指向的对象是不会被GC掉的。

New函数用于创建新对象。重点关注local和localsize，实际是指向数组的地址，该数组实际的类型[P]poolLocal，这里的P指的实际上是goroutine调度里面的概念，每个goroutine都必须要绑定一个P才能得以执行，而每个P都有一个待执行的goroutine队列，P的个数一般设置跟CPU核数一致，这里不做展开讲述，后续会有一个相对详尽的篇幅来说明。

我们再看看poolLocal结构长什么样。
<code>
type poolLocal struct {
    poolLocalInternal
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
type poolLocalInternal struct {
    private interface{}   // Can be used only by the respective P.
    shared  []interface{} // Can be used by any P.
    Mutex                 // Protects shared.
}
</code>
poolLocal里面包了 一层结构体poolLocalInternal，里面的private，shared，Mutex。为了更高效使用goroutine，sync.Pool为每个P都分配了一个本地池，当执行Get或者Put操作的时候，会先将goroutine和某个P的子池关联，再对该子池进行操作，每个P的子池分为私有对象和共享对象，私有对象只能被特定的P访问，共享对象可以被任何P访问。因为同一时刻一个P只能执行一个goroutine，所以无需加锁，但是对共享列表对象进行操作的时候，可能有多个goroutine同时操作，所以需要加锁。

需要注意的是poolLocal结构体里面有一个pad成员，目的就是为了防止false  sharing。该字段的类型为
[128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
false sharing，国内比较多的翻译是伪共享。简单说就是，一次请求，从处理访问到内容，中间有好几级缓存。缓存的存储单位是cacheline，也就是说每次从内存中加载数据都是以cache line 为单位的。这个时候，假定同个cache  line 里面有A，B两个变量，刚好不同处理器要分别处理这两个变量，当修改A的时候，缓存系统会强制使另外一个处理器处理失效，反之也同样，来回从低级的缓存读取数据，影响性能。会出现这种情况都是基于并发场景，很显然poolLocal这里便是，这里怎么避免跟其他变量公用一个cache  line 呢？最简单的方法就是，让它占满整个cache  line，进行补齐。当不同线程同时读写同一个cache line 上不同数据时就可能发生，会导致多核处理器上严重的系统性能下降，这块内容也有一块内容做详细说明。

从使用的角度分析，我们根据Get函数剖析下：
<code>
func (p *Pool) Get() interface{} {
    if race.Enabled {
        race.Disable()
    }
    l := p.pin()
    x := l.private
    l.private = nil
    runtime_procUnpin()
    if x == nil {
        l.Lock()
        last := len(l.shared) - 1
        if last >= 0 {
            x = l.shared[last]
            l.shared = l.shared[:last]
        }
        l.Unlock()
        if x == nil {
            x = p.getSlow()
        }
    }
    if race.Enabled {
        race.Enable()
        if x != nil {
            race.Acquire(poolRaceAddr(x))
        }
    } 
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}
</code>

一开始，代码是关闭了race竞态检测的，整个获取过程可以简化为下面几点：
1. 尝试从本地P对应的那个本地池中获取一个对象值，并从本地池中删除该值；
2. 如果获取失败，那么从共享池中获取，并从共享队列中删除该值；
3. 如果获取失败，那么从其他P的共享池中偷一个过来，并删除共享池中的该值(p.getSlow())；
4. 如果仍失败，那么直接通过New()分配一个返回值，注意这个分配的值不会被放入池中。New返回用户注册的New函数的值，如果用户未注册New，那么返回nil；

这个环节有一个函数值的关注：runtime_procPin()，该函数返回的是P的id。这里又牵扯出goroutine的调度原理，下面简单也阐述下，详细会在goroutine调度的内容里面说明。procPin函数实际上是获取当前的goroutine，然后对当前写成绑定的线程(即m)加锁，然后返回m目前绑定的P的id。这里的加锁，系统线程在对协程调度的时候，有时候会抢占当前正在执行的协程的所属P，原因是不能让某个协程一直占用计算资源，那么在进行抢占的时候会判断m是否适合抢占，其中有一个条件就是判断m的lock数是否为0，那么procPin的含义就是禁止当前P被抢占。相对的，procUnpin就是解锁了，取消禁止抢占。


再多看一次init函数
<code>
func init() {
    runtime_registerPoolCleanup(poolCleanup)
}
</code>

一开始初始化的时候会注册一个poolCleanup的函数，这个函数会清除sync.Pool中的所有缓存对象，很粗暴，直接所有指针清空，这个注册函数会在每次GC的时候运行，所以sync.Pool中的值只在两次GC中间的时间段有效。

Go1.13
看完go1.12版本的内容，尤其是poolCleanup函数，会发现一个问题，Pool包会经常清空我们的pool，使得我们经常new新对象，同时，在复用比的P的共享对象的时候会经常加锁，针对这些问题，Pool的底层处理做了相对比较大的调整。
首先是结构体：
<code>
type Pool struct {
    noCopy noCopy

    local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
    localSize uintptr        // size of the local array

    victim     unsafe.Pointer // local from previous cycle
    victimSize uintptr        // size of victims array
    New func() interface{}
}
</code>
直观的可以看出来，相对比1.12版本，1.13多了两个字段，victim 跟victimsize。但是再看Pool的Get方法其实是没有什么调整的，那么这两个字段又有什么作用呢？

熟悉CPU的可能早已经接触过这个词汇了，也就是victim cache机制。
简单说就是，一个与直接匹配或者低相联缓存并用的、容量很小的全联缓存，当一个数据块被逐出缓存时，并不直接丢弃，而是暂时进入victim cache，如果该缓存已满，就替换掉其中一项。当进行缓存标签匹配时，在与索引指向标签匹配的多同时，并行查看victim cache，如果发现匹配，就将此数据块与不匹配数据块做交换，同时返回给处理器。

这已经是成功的解决算法了，1.13版本只是引用进来，这么做的好处是什么呢？

之前我们总是会提到，pool会被全部清除，这就导致了每次都要大量重新创建对象，会造成短暂的GC压力，而此次我们只会清空victim cache，清空的量就会变少，清空victim cache之后，我们再把之前的cache内容作为新的victim cache。如果从别的P的shared中还拿不到对象，就会直接去victim cache拿对象，用完之后肯定再put到我们的原先的cache中，这么做，如果我们GET速度跟put速度比较相似，即使每次GC的时候，都会把老的cache置为victim cache，但是很快GET就会从victim cache那出来了，即使GET的速度降下来，此时对象会有一部分分在victim cache，一部分在localCache，分成两次GC，第一次直接清空victim cache，然后localCache置为victim cache，第二次GC再把刚才victim cache清空，这种思想有点像分代垃圾回收的思想，分代垃圾回收其实就是把生命周期短的对象回收，尽量保留生命周期长的对象，关于垃圾回收会有另外的篇幅来说明。

其实，1.13的shard由以前的数组改为双向链表了，类型为poolChai。在此之前，Pool都是使用一个受Mutex保护的slice来存储每个shard的overflow对象。（sync.Pool使用shard方式存储池化对象，减少竞争。每个P对应一个shard。如果需要创建多于一个池化对象，这些对象就叫做overflow）。但是频繁的锁操作仍存在一定的优化空间，我们知道实现lock-free的数据结构一般采用atomic的方式实现，通过CAS避免操作block。sync.Pool也是如此，定义了一个lock-free的双向链表。会从headTail开始拿起，headTail指的是ring的首尾位置，放在一个uint64字段里面，主要是为了方便原子操作，并可以通过位操作从headTail中拆分出head和tail。下面是链表操作的代码：
<code>
func (d *poolDequeue) popHead() (interface{}, bool) {
    var slot *eface
    for {
        ptrs := atomic.LoadUint64(&d.headTail)
        head, tail := d.unpack(ptrs)
        if tail == head {
            return nil, false
        }
        head--
        ptrs2 := d.pack(head, tail)
        if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
            slot = &d.vals[head&uint32(len(d.vals)-1)]
            break
        }
    }

    val := *(*interface{})(unsafe.Pointer(slot))
    if val == dequeueNil(nil) {
        val = nil
    }
    *slot = eface{}
    return val, true
}
</code>
1. 如果head和tail指向同一个位置，则代表当前ring是空的，直接返回false
2. 对head减一，然后pack出新的headTail
3. 接下来用原子操作CompareAndSwapUint64来更新headTail，就是说我们从读取headTail到此时间段内，没有其他协程对headtail操作的话，我们才会真正更新它，否则继续循环，尝试操作，如果成功则赋值头部对象为slot，break循环。
4. 把slot转成interface{}类型的value
5. 注意到最后一步，slot=eface{}，是把ring数组的head位置置空，但是置空的方式是为空eface，注释写到，这里清空的方式与popTail不同，这里与pushHead没有竞争关系

再看看popTail函数：
<code>
func (c *poolChain) popTail() (interface{}, bool) {
    d := loadPoolChainElt(&c.tail)
    if d == nil {
        return nil, false
    }

    for {
        d2 := loadPoolChainElt(&d.next)

        if val, ok := d.popTail(); ok {
            return val, ok
        }

        if d2 == nil {
            return nil, false
        }

        if atomic.CompareAndSwapPointer((*unsafe.Pointer)(unsafe.Pointer(&c.tail)), unsafe.Pointer(d), unsafe.Pointer(d2)) {

            storePoolChainElt(&d2.prev, nil)
        }
        d = d2
    }
}
</code>
1. 获取到链表的尾部指针，如果指针为空则直接返回
2. for循环，首先遍历d的next对象，调用d.popTail获取对象。如果获取成功则返回，否则判断next是否为空，如果为空则返回false，不为空就直接把刚才的d从链表中删除，继续从刚才的next对象获取对象，以此类推

为什么无法从d中获取对象，并且对d删除还需要原子操作呢？为什么从尾部的节点无法获取到对象，因为可能尾部节点是空的。上一次get操作成功之后是不会对节点进行删除的，等着下一次删除，因为此时可能有多个协程从这个链表的尾部节点获取对象，因为steal别的P。尾部节点为空的情况下，大家都去删除，要保证只有一个删除成功。这可以理解为是收缩链表，对于popHead操作，很简单就是不断遍历链表节点，不断获取对象，直到获取成功。最主要的是我们还要从头部push节点，避免出现多协程操作链表，比如你把头部节点删除了，我此时正好要往里面插入数据，那就很麻烦了。

上面说的其实还不是核心，lock-free的核心在ring的popTail函数
<code>
func (d *poolDequeue) popTail() (interface{}, bool) {
    var slot *eface
    for {
        ptrs := atomic.LoadUint64(&d.headTail)
        head, tail := d.unpack(ptrs)
        if tail == head {
            return nil, false
        }

        ptrs2 := d.pack(head, tail+1)
        if atomic.CompareAndSwapUint64(&d.headTail, ptrs, ptrs2) {
            slot = &d.vals[tail&uint32(len(d.vals)-1)]
            break
        }
    }
    val := *(*interface{})(unsafe.Pointer(slot))
    if val == dequeueNil(nil) {
        val = nil
    }
    slot.val = nil
    atomic.StorePointer(&slot.typ, nil)
    return val, true
}
</code>
1. 先解析下代码，在for内部，会先获取headTail指针，解析成head和tail，两者相等，ring就是空的，直接返回
2. 给tail+1 然后pack成新的headTail，采用原子操作来更新
3. 跳出循环之后，代表我们拿到数据了，则转化成interface{}，然后清空slot，清空方式是给eface的value跟type都置为nil，这里是分为两步的，先清空value，然后用原子操作把type置为nil


看到这里我们回头再看下最初的Get函数，如果从自己的shard中还是无法获取到对象，调用的是getSlow，在1.12版本，是从其他的P的shard数组中获取对象，但是1.13处理则不一样，因为底层由数组变为了链表。获取的时候，会从head节点开始，如果head节点为空，则直接创建一个新节点，但是这里用了原子操作，因为popTail的时候会收缩链表，清除节点的操作，会有竞争关系，所以需要原子操作，但是head指针赋值因为收缩的时候不会修改head指针，所以不需要原子操作。接下来往head节点push对象，如果成功，则直接返回，没成功就说明ring满了，需要扩容ring为之前的2倍，并且有最大长度限制
const dequeueBits = 32
const dequeueLimit = (1 << dequeueBits) / 4

说了这么，其实总结链表的原子操作，会发现，pushHead和popTail有竞争问题，而pushHead和popHead没有竞争问题。
对于前者，pushHead是对本地P的localCache操作，而popTail则是抢其他P的localCache操作，所以会存在竞争的可能；而popHead只会在本地的localCache中才会popHead，只有本地的拿不出来，才会去popTail拿别的P的，同一个P同时跑的goroutine只能是一个，所以肯定是不会跟pushHead冲突的。

最后再惯例看看clean
<code>
func poolCleanup() {
    for _, p := range oldPools {
        p.victim = nil
        p.victimSize = 0
    }
    for _, p := range allPools {
        p.victim = p.local
        p.victimSize = p.localSize
        p.local = nil
        p.localSize = 0
    }
    oldPools, allPools = allPools, nil
}
</code>
跟之前说的一样，清空victim cache，然后把现有的local cache置为victim cache。

总结来说，sync.Pool的分析可以归纳为以下几点：
1. cacheline  伪共享问题
2. noCopy
3. lock-free
4. Victim cache 问题
5. GC细节

