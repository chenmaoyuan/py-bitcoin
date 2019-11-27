公链实战系列13-分布式系统-分布式Hash表(DHT)-上篇

理论介绍
实现一个简单的DHT
Hash表
节点
Kademlia算法
代码实现
节点实现
节点测试
路由表实现
测试
总结
参考
理论介绍
DHT(Distributed Hash Table, 分布式hash表)类似Tracker的根据种子特征码返回种子信息的网络。DHT全称分布式哈希表(Distributed Hash Table)，是一种分布式存储方法。在不需要服务器的情况下，每个客户端负责一个小范围的路由，并负责存储一小部分数据，从而实现整个DHT网络的寻址和存储。

DHT是分布式计算系统中的一类，用来将一个关键值(key)的集合分散到所有分布式系统中的节点，并且可以有效地将信息转送到唯一一个拥有查询提供的关键值的节点(peer)。
分布式散列表具有以下性质

分散性： 构成系统的节点并没有任何中央式的协调机制。
规模性： 即使拥有众多节点，系统仍应保持效率
容错性： 即使节点不断地加入、离开或是停止工作，系统仍需达到一定可靠度。
在P2P系统或者分布式系统中，要实现一个分布式网络，需要实现两个核心的技术：

拓扑结构与内容路由
内容传送
目前来看，在分布式系统中，实现内容路由的方式以DHT(分布式Hash表)与一致性Hash为主。 如Bittorrent采用的DHT来进行路由寻址， openstack swift采用一致性Hash ring来进行路由寻址。

此外，由于在去中心化的P2P网络中，存在成千上万的节点。以比特币为例，由于比特币采用了POW的共识算法，所以比特币系统的节点数量也是非常巨大。要在这些节点直接进行数据同步，也是去中心技术网络的一个巨大挑战。当然，内容传送方式根据应用不同，也可以分为以下两类：

非实时类内容传送
实时类内容传送
关于内容传送相关的知识，可以重点关注IPFS项目，IPFS目前进展也相当乐观。除了有go-ipfs，js-ipfs的实现之外，很多其他语言的实现也已经开始开发了。

实现一个简单的DHT
在一个真实的DHT当中，节点分布在不同的机器上，可能跨地区、跨城市甚至跨国家, 他们之间基于底层的socket协议进行通信。在这里由于我们是一个简单的DHT，所以我们仅仅只是模仿一个DHT的实现，来解释DHT的工作原理。 当然，关于完整的DHT的实现，看后续有木有时间。当然，目前已经有很多关于DHT实现的开源项目，感兴趣的同学可以直接到github找源码读代码。

Hash表
首先熟悉下hash表，在《Go语言算法实战专栏》里面，我们有关于对hash表实现的讲解，建议首先了解下关于hash表的知识，然后再阅读下面的内容。当然，如果你已经具备了hash表相关的知识，那么就忽略吧。

每一个节点都会保存一个hash表。在分布式系统中，尤其是P2P网络中，并没法保证每个节点都是有效的可工作节点。所以为了保证系统的可用性，在hash表中保存的节点信息，需要剔除不可用节点。不同的分布式系统有不同的剔除策略，一般是通过检测心跳的方式进行验证，如果一个节点心跳异常，则可认为是有问题的节点，进行剔除。关于hash表的大小，可以根据具体的节点数量进行调整。

节点
顾名思义，既然是分布式，那么肯定包含很多节点。是的，在分布式hash表实现中，有很多个节点，每一个节点都有一个唯一的ID标识。同时还会用距离表示两个节点之前的亲密度。每一节点上都会有一个hash表，此hash表里面会根据节点亲密度来保存其他节点的信息，亲密度越高，记录信息越详细。我们需要做的事情，就是按照以上规则，在网络中找到匹配规则的节点，然后将信息存储到相对于的hash表中，以建立节点之间的联系。

Kademlia算法
前面我们提到用距离来表示节点之间的亲密度，在Kademlia算法中，距离用XOR来计算

distance(A, B) = |A xor B|
代码实现
在本节中我们实现节点跟路由表相关的代码

节点实现
package kademlia

import (
    "encoding/hex"
    "math/rand"
)

const IdLength = 20

type NodeID [IdLength]byte

func NewNodeID(data string) (ret NodeID) {

    for k, v := range []byte(data){
        ret[k] = byte(v)
    }

    return
}

func NewRandomNodeID() (ret NodeID) {
    for i := 0; i < IdLength; i++ {
        ret[i] = uint8(rand.Intn(256))
    }

    return
}

func (node NodeID) String() string {
    return hex.EncodeToString(node[:])
}

func (node NodeID) Equals(other NodeID) bool {
    for i := 0; i < IdLength; i++ {
        if node[i] != other[i] {
            return false
        }
    }
    return true
}

func (node NodeID) Less(other interface{}) bool {
    for i := 0; i < IdLength; i++ {
        if node[i] != other.(NodeID)[i] {
            return node[i] < other.(NodeID)[i]
        }
    }
    return false
}

func (node NodeID) Xor(other NodeID) (ret NodeID) {
    for i := 0; i < IdLength; i++ {
        ret[i] = node[i] ^ other[i]
    }
    return
}

func (node NodeID) PrefixLen() (ret int) {
    for i := 0; i < IdLength; i++ {
        for j := 0; j < 8; j++ {
            if (node[i]>>uint8(7-j))&0x1 != 0 {
                return i*8 + j
            }
        }
    }
    return IdLength*8 - 1
}
在节点实现中，我们实现了两个函数，根据字符串生成节点ID以及随机生成节点ID的函数。另外对于节点ID我们实现了下面几个方法

Equals
Less
Xor
PrefixLen
其中Equals用来判断两个节点是否相同，Less判断节点的ID的大小，Xor是前面介绍的kademlia算法的核心，用来计算两个节点之间的距离。

PrefixLen的解释详细解释下。首先我们分布式hash表中会包含NodeId的范围为0～2**160个。根据kademlia算法，所有的路由表被分割到"buckets(桶)"中，每个节点上都保存有部分的节点信息，这些节点会根据亲密度也就是距离来存储，距离越近的存储越详细。决定每个节点上的桶里面需要存储的bucket则由我们的PrefixLen来计算。在此算法中，我们用到了位运算。

node[i]>>uint8(7-j)    // 此运算不为0则返回
节点测试
package kademlia

import (
    "testing"
    "fmt"
)

func TestNodeID(t *testing.T)  {
    a := NewNodeID("Node1")
    b := NewNodeID("Node2")
    c := NewNodeID("Node3")

    fmt.Printf("%s %s %s\n", a, b, c)

    d := NewRandomNodeID()
    fmt.Println(d)

    if !a.Equals(a){
        t.Fatal("Equals func is wrong!")
    }

    if a.Equals(b){
        t.Fatal("Equals func is wrong!")
    }

    e := NodeID{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19}
    f := NodeID{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 19, 18}
    g := NodeID{0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1}

    if !e.Xor(f).Equals(g){
        t.Error(a.Xor(b))
    }

    if g.PrefixLen() != 151{
        t.Error("prefixlen is error")
    }

    if b.Less(a){
        t.Error("b should more a!")
    }

    fmt.Println(e, e.PrefixLen())

}
路由表实现
package kademlia

import (
    "container/list"
    "fmt"
)

const BucketSize = 20

type RoutingTable struct {
    node    Contract
    buckets [IdLength * 8]*list.List
}

type ContractRecord struct {
    node    *Contract
    sortKey NodeID
}

func (rec *ContractRecord) Less(other interface{}) bool {
    return rec.sortKey.Less(other.(*ContractRecord).sortKey)
}

func NewRoutingTable(node *Contract) (ret *RoutingTable) {
    ret = new(RoutingTable)
    for i := 0; i < IdLength*8; i++ {
        ret.buckets[i] = list.New()
    }
    ret.node = *node
    return
}

func (table *RoutingTable) Update(contract *Contract) {
    prefixLength := contract.id.Xor(table.node.id).PrefixLen()
    bucket := table.buckets[prefixLength]

    var element interface{}

    iter := bucket.Front()

    for iter != nil {
        if Equal(iter.Value, table) == true {
            element = bucket.Front()
        } else {
            iter = bucket.Front().Prev()
        }
    }

    iterBack := bucket.Back()
    for iterBack != nil {
        if Equal(iterBack.Value, table) == true {
            element = bucket.Back()
        } else {
            iterBack = bucket.Back().Next()
        }
    }

    if element == nil {
        fmt.Println("-------")
        if bucket.Len() < BucketSize {
            bucket.PushFront(contract)
        }
        fmt.Println(bucket.Front().Value)
        // 剔除节点
        // todo list

    } else {
        bucket.MoveToFront(element.(*list.Element))
    }

}

func Equal(x interface{}, table *RoutingTable) bool {
    return x.(*Contract).id.Equals(table.node.id)
}

func (table *RoutingTable) FindClosest(target NodeID, count int) (ret []interface{}) {

    bucketNum := target.Xor(table.node.id).PrefixLen()
    bucket := table.buckets[bucketNum]

    ret = append(ret, bucket.Front())

    for i := 1; (bucketNum-i >= 0 || bucketNum+i < IdLength*8) && len(ret) < count; i++ {
        if bucketNum-i >= 0 {
            bucket = table.buckets[bucketNum-i]
            ret = append(ret, bucket.Front())
        }
        if bucket_num+i < IdLength*8 {
            bucket = table.buckets[bucketNum+i]
            ret = append(ret, bucket.Front())
        }
    }

    if len(ret) > count {
        ret = ret[:count]
    }
    return
}
在实现路由表的代码中，我们实现了一个函数NewRoutingTable，用来新建一个路由表。两个方法

Update
FindClosest
Update方法用于更新我们的理由表，随着节点的变化，我们会添加或者剔除节点信息。
FindClosest方法用于找出节点的临近节点，我们可以通过指定临近节点的数量来查看，当前节点有多少个临近节点。代码没什么可以多讲的地方，这里我们用到了*list.List 这是一个双向的链表，使用的时候需要注意下，另外对于链表不熟悉的同学可以查看《Go语言算法实战》了解相关内容。

测试
package kademlia

import (
    "testing"
    "fmt"
)

func TestNewRoutingTable(t *testing.T) {
    n1 := NewNodeID("FFFFFFFF")
    n2 := NewNodeID("FFFFFFF0")
    n3 := NewNodeID("11111111")

    rt := NewRoutingTable(&Contract{n1, "localhost:5000"})
    rt.Update(&Contract{n2, "localhost:5001"})
    rt.Update(&Contract{n3, "localhost:5002"})

    //fmt.Println(rt)
    n4 := NewNodeID("22222222")
    res := rt.FindClosest(n4, 2)

    for _, r := range res{
        fmt.Println(r)
    }

}
总结
通过上述代码，我们已经实现了一个分布式Hash表最核心的部分。但是截止目前，我们的代码还不完整，因为既然是分布式系统，那么就应有有多个节点通信。 但是现在我们节点之间通信的部分都还没有，仅仅只是实现了节点路由的存储与查找。由于本部分内容太多，所以关于节点之间通信的部分放到下一章节来讲解。下一章会结合之前讲到的RPC通信来实际部署一个分布式系统，然后观察节点之间的通信方式，以及路由规则是如何起作用的。
