公链实战系列14-分布式系统-分布式Hash表(DHT)-下篇

理论介绍
协议消息
加入网络
代码实现
节点通信协议实现
服务端实现
客户端实现
启动一个Node
测试启动节点
查看测试输出
服务端输出
客户端输出
测试Ping检查
总结
项目源码
参考
理论介绍
在上篇中我们已经实现了节点跟路由表部分的代码。在本篇中，我们来完成剩下的部分。在开始之前，首先来重新详细的认识下上篇中提到的kademlia算法。

Kademlia是一种通过分布式散列表实现的协议算法，它是由Petar和David为P2P计算网络而设计的。在Kademlia算法中规定了网络的结构，同时也规定了通过节点查询进行信息交换的方式。其追求的主要目标是能够快速的定位期望的节点。上篇中我们也提到在Kademlia算法中通过异或操作来计算两个节点之间的距离，以此来表征节点的亲密度。

由于路由表我们在上篇中已经实现了，所以在这里就不赘述了。 我们直接进入本篇相关的内容。

协议消息
Kademlia协议共有四种消息

PING消息 测试节点是否在线
STORE 在某个节点中进行存储
FIND_NODE 寻找离自己最近的k个节点
FIND_VALUE 获取值
加入网络
节点的加入需要一个引导过程，节点需要知道其他已经在网络中的节点的IP地址跟端口号，最初的时候，节点只有一个K桶，当有新节点加入该K桶时，如果k桶已满，K桶就会开始分裂。

关于更多的理论知识的讲解大家可以看下面的参考文章，或者网上也有一些blog讲解关于DHT的理论知识，建议在实现代码前好好阅读相关的理论，然后回头来看代码实现。当然反过来我认为也可以。另外，关于此部分的内容，代码也只是实现大概。由于分布式系统的设计哲学与复杂性，很多细节不会在代码中体现，如果确实有感兴趣的同学，我后面会推荐一些比较好的开源项目的实现，大家可以直接看源码学习。

代码实现
前面大概讲了本节需要实现的内容，其核心就是协议消息与节点如何加入到网络中。对于DHT的实现，目前有好几种算法可以选择，我们这里选择Kademlia算法来实现。

节点通信协议实现
既然是分布式系统，那么节点之间通信就是绕不开的话题，在这里我们采用RPC来实现，对于RPC不了解的同学可以翻翻前面的教程，前面我们有一节内容专门介绍关于RPC&gRPC的使用。

服务端实现
package kademlia

import (
    "net/rpc"
    "net"
    "fmt"
)

type Kademlia struct {
    routers   *RoutingTable
    NetworkId string
}

type KademliaCore struct {
    kad *Kademlia
}

// rpc server
func (k *Kademlia) Serve() error {
    rpc.RegisterName("KademliaCore", &KademliaCore{k})

    listener, err := net.Listen("tcp", k.routers.node.Address)

    if err != nil {
        return err
    }

    conn, err := listener.Accept()
    fmt.Println(conn.LocalAddr())
    if err != nil {
        return err
    }
    go rpc.ServeConn(conn)

    go fmt.Println(k.routers)
    select{}
    return nil
}

// rpc server
func (k *Kademlia) Serve() error {
    rpc.RegisterName("KademliaCore", &KademliaCore{k})

    listener, err := net.Listen("tcp", k.routers.node.Address)

    if err != nil {
        return err
    }

    conn, err := listener.Accept()
    fmt.Println(conn.LocalAddr())
    if err != nil {
        return err
    }
    go rpc.ServeConn(conn)

    go fmt.Println(k.routers)
    select{}
    return nil
}
在上面的代码中我们实现了节点通信RPC协议的server端代码。 在分布式系统中，尤其是P2P网络中，节点之间是等价的。也就是没有中心服务器的概念，每个节点既是服务器也是客户端。

客户端实现
客户端的代码比较简单，如果看了前面的rpc教程，相信大家一看就能明白，核心就两个，一个是建立链接。另一个是调用远程服务器上的方法。

// rpc client
func (k *Kademlia) Call(contract *Contract, method string, args, reply interface{}) error {
    client, err := rpc.Dial("tcp", contract.Address)
    if err != nil {
        return err
    }

    fmt.Println(method, args)
    err = client.Call(method, args, reply)
    if err != nil {
        return err
    }

    fmt.Println("reply:", reply)

    k.routers.Update(contract)
    return nil
}
除此之外，在DHT中，我们需要快速定位目标节点，也就是查询，所以我们定义了一个sendQuery的方法

func (k *Kademlia) sendQuery(node *Contract, target NodeID, done chan []Contract) {
    args := FindNodeRequest{RPCHeader{&k.routers.node, k.NetworkId}, target}
    reply := FindNodeResponse{}

    err := k.Call(node, "KademliaCore.FindNode", &args, &reply)
    if err != nil {
        done <- []Contract{}
    }

    done <- reply.contacts

}
在分布式hash表中，每个节点都维护一个hash表，其中记录着临近节点的信息。在分布式系统中，节点发生故障，也就是节点失效是大概率事件，所以我们需要保证hash表中记录的节点信息是有效的节点。所以需要对节点进行健康检查，也就是检测节点心跳。对于心跳异常的节点从hash表中剔除出去，避免我们的消息处理异常或者丢失。下面通过Ping方法来检查节点健康状况。

func (kc *KademliaCore) Ping(args *PingRequest, response *PingResponce) error {
    err := kc.kad.HandleRPC(&args.RPCHeader, &response.RPCHeader)
    if err != nil {
        return err
    }

    fmt.Printf("ping from %s", args.RPCHeader)
    return nil
}

func (k *Kademlia) HandleRPC(request, response *RPCHeader) error {
    //if request.networkId != k.NetworkId {
    //  return fmt.Errorf("Excepted networkID %s, got %s", k.NetworkId, request.networkId)
    //}

    if request.Sender != nil {
        k.routers.Update(request.Sender)
    }

    response.Sender = &k.routers.node
    return nil
}
另外，我们还需要实现FIND_NODE方法，当节点加入网络之后，需要找出离自己最近的节点，然后更新hash表。

func (kc *KademliaCore) FindNode(args *FindNodeRequest, response *FindNodeResponse) error {
    err := kc.kad.HandleRPC(&args.RPCHeader, &response.RPCHeader)

    if err != nil {
        return err
    }
    contancts := kc.kad.routers.FindClosest(args.target, BucketSize)
    response.contacts = make([]Contract, len(contancts))

    //for i := 0; i < len(contancts); i++ {
    //  // todo there is some error，need to update
    //  // hashtable need to rewrite.
    //  response.contacts[i] = *contancts[i].(*ContractRecord).node
    //}

    return nil
}
到这里，核心的代码实现就差不多了，所以接下来，我们看看测试代码吧。因为需要测试节点接入网络，所以在这里还需要额外写一些代码。

启动一个Node
// node.go

package kademlia

import "fmt"

func RunServer()  {
    nodeOneId := NewNodeID("FFFFFFFF")
    currentNode := Contract{nodeOneId, "localhost:9999"}
    k := NewKademlia(&currentNode, "NodeOne")
    fmt.Println("server is running at localhost:9999")
    k.Serve()
}


func Client()  {
    nodeOneId := NewNodeID("FFFFFFFF")
    currentNode := Contract{nodeOneId, "localhost:9999"}

    nodeTwoId := NewNodeID("FFFFFFF0")
    other := Contract{nodeTwoId, "localhost:8888"}

    k2 := NewKademlia(&other, "NodeTwo")

    err := k2.Call(
        &currentNode, "KademliaCore.Ping", &PingRequest{RPCHeader{&other, "NodeTwo"}},
        &PingResponce{},
    )

    if err != nil{
        fmt.Println(err)
    }
}
我们上面提到，一个node既是server又是client，所以在这里我们需要实现两个函数，一个是RunServer函数，一个是Client函数，


测试启动节点
新建一个server.go文件

package main

import (
    "github.com/csunny/argo/src/libs/kademlia"
)

func main()  {
    kademlia.RunServer()
}
运行命令

go run server.go

这样我们就启动了一个server。接下来，我们让其他节点链接进来。

// client.go
package main

import (
    "github.com/csunny/argo/src/libs/kademlia"
)

func main() {
    kademlia.Client()
}
运行命令

go run client.go

查看测试输出
测试输出包含客户端跟服务端的输出，在这里我们起一个节点测试下，当然，更好的方式是起多个节点进行测试。

服务端输出


客户端输出


测试Ping检查
func TestKademliaCore_FindNode(t *testing.T) {
    currentNode := Contract{NewRandomNodeID(), "localhost:9999"}
    k := NewKademlia(&currentNode, "Test")
    kc := KademliaCore{k}

    var contacts [100]Contract

    for i:=0; i<len(contacts); i++{
        contacts[i] = Contract{NewRandomNodeID(), "localhost:9999"}
        err := kc.Ping(&PingRequest{RPCHeader{&contacts[i], k.NetworkId}},
            &PingResponce{},
        )

        if err != nil{
            t.Error(err)
        }
    }

    args := FindNodeRequest{RPCHeader{&contacts[0], k.NetworkId}, contacts[0].Id}
    response := FindNodeResponse{}
    err := kc.FindNode(&args, &response)
    if err != nil{
        t.Error(err)
    }
总结
综上，我们采用Go语言实现了一个基于Kademalia算法的Demo版本的DHT(分布式Hash表), 整个实现我们分了两部分，每部分都很长，内容也很多。 即使如此，还是有很多内容没有涵盖到，理论知识涉及很少。此外，即使我们实现的代码，目前也只是帮助大家理解DHT，要用于实际的生产还需要做大量的工作。另外，目前也有很多比较好的开源项目实现了DHT算法，感兴趣的同学可以自己找找代码来阅读。

