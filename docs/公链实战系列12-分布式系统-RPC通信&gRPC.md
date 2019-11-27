公链实战系列12-分布式系统-RPC通信&gRPC

简介
理论介绍
代码实战
RPC通信实现
Server端实现
Client端实现
gRPC实战
定义hello.proto文件
服务端实现
客户端实现
REST和Open APIs之上的gRPC
编辑hello.proto文件
生成reverse-proxy代码
proxy
测试
总结
参考
简介
在前面的章节中，我们讲解了如何实现一条公链。 当然，仅凭前面的实现，我们的代码离上生产环境还有很长的路要走。我这里为了让知识点更加集中，也为了能让大家了解到更多的内容，我把分布式系统相关的内容单独拿出来，放到后面来讲。在承接前面内容的同时，也适当为后续内容做铺垫。

因为后续的内容更多的是分布式系统相关的知识，在语言选择上会更偏向于用go语言。除了go语言在性能方面的优势之外，同时也因为go语言在分布式系统领域有更完善的技术方案与第三方库。 当然，语言始终只是个工具，在项目源码里面，python的源码很大概率都是会实现的。

理论介绍
通常不管是在目前流行的微服务还是在分布式系统中，系统间的调用方式主要以以下两种方式为主：

RPC
REST API
相信REST API大家都比较熟悉了，同时在我们之前的章节中，我们也基于flask实现了REST API层，负责外界与系统间的通信。

所以我们这里重点谈谈RPC调用，随着近年来分布式技术的发展，目前也已经有很多基于RPC通信的框架了，其中代表性的有：

hprose
grpc
thrift
Dubbo/Dubbox
其实不管是REST AP还是RPC其底层都是socket网络编程，都是基于传输层之上的应用层的协议。在REST API中是使用的HTTP/1.1的协议，在以gRPC中用到的是HTTP/2 的协议。同时，不同与REST API中采用json与XML处理数据，gRPC基于protobuf对数据进行序列化与反序列化，是一种更为高效的通信方式。经过最近这几年的发展gRPC已经支持了多种语言，并且也已经官方应用与下列场景中

低延迟、高扩展性、分布式系统
云端与客户端通信
设计语言独立、高效、精确的新协议
便于扩展的分层设计 如认证、负载均衡、日记记录、监控等
代码实战
在代码实现中，我们首先来看看在不使用任何框架的时候，我们如何实现一个RPC通信。


RPC通信实现
在go语言中官方提供了一个RPC库，所以在不使用框架的情况下，我们用go官方的net/rpc库来实现一个RPC调用。

在实现之前我们首先来熟悉下RPC远程调用的条件，如果一个对象的方法要实现RPC调用，那么必须满足以下条件：

方法的类型可输出
方法本身可输出
方法必须由两个参数
方法第二个参数是指针类型
方法返回类型为error
基于以上条件我们来看一个Hello world的代码

Server端实现
package rpc_example

type Greeter struct {

}

func (p *Greeter) Greet(request string, response *string) error  {
    *response = "Hello: " + request
    return nil
}

func Server()  {
    rpc.RegisterName("Greeter", new(Greeter))
    listener, err := net.Listen("tcp", ":8888")
    fmt.Println("Server is running at localhost:8888...")

    if err != nil{
        log.Fatal("ListenTCP error:", err)
    }

    conn, err := listener.Accept()
    if err != nil{
        log.Fatal("Accept error", err)
    }

    rpc.ServeConn(conn)
}
在服务端， 我们将对象方法注册为RPC函数，并在8888端口启动了一个TCP服务

Client端实现
func Client()  {
    client, err := rpc.Dial("tcp", "localhost:8888")
    if err != nil{
        log.Fatal("dialing:", err)
    }

    var response string
    err = client.Call("Greeter.Greet", "magic", &response)
    if err != nil{
        log.Fatal(err)
    }

    fmt.Println(response)
}
客户端首先建立链接，然后通过Call方法与服务端进行通信。首先运行服务端代码启动服务，然后运行客户端代码，观察输出

Hello: magic
这样我们就实现了一个简单的RPC通信，相信实现之后大家对RPC概念已经有大概的认识了，接下来我们看看框架的使用，这里我们重点介绍gRPC。

gRPC实战
作为入门，首先我们来看一个Hello World, 在前面介绍gRPC的时候，gRPC框架是用protobuf对数据进行序列化的，所以首先我们需要编写protobuf文件

定义hello.proto文件
# hello.proto 

syntax = "proto3";

package hello;

service Greeter {
    // Sends a greeting
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.

message HelloRequest {
    string name = 1;
}

// The response message containing the greetings
message HelloReply {
    string message = 1;
}
编写好proto文件之后, 接下来使用protoc生成相应代码

protoc -I grpc grpc/hello.proto  --go_out=plugins=grpc:grpc
如果命令执行成功，会发现生成了一个hello.pb.go 文件

服务端实现
package grpc_example

import (
    "log"
    "net"
    "os"
    "golang.org/x/net/context"
    pb "github.com/csunny/argo/examples/grpc"
    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
)

const (
    port        = ":9090"
    defaultName = "Magic"
)

// server
type server struct{}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
    return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

func RunServer() {
    log.Println("server is running at 127.0.0.1:9090... ")

    lis, err := net.Listen("tcp", port)
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    s := grpc.NewServer()
    pb.RegisterGreeterServer(s, &server{})
    // Register reflection service on gRPC server.
    reflection.Register(s)
    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
客户端实现
func Client() {
    // set up a connection to the server
    conn, err := grpc.Dial("localhost:9090", grpc.WithInsecure())
    if err != nil {
        log.Fatal(err)
    }

    defer conn.Close()
    c := pb.NewGreeterClient(conn)

    // Contract the server and print response
    name := defaultName
    if len(os.Args) > 1 {
        name = os.Args[1]
    }

    r, err := c.SayHello(context.Background(), &pb.HelloRequest{Name:name})
    if err != nil{
        log.Fatal("could not greet:", err)
    }
    log.Println("Greeting: ", r.Message)
}
代码没有什么具体可讲的地方，了解前面RPC调用满足的条件之后，gRPC也挺好理解的。如果有不清楚的可以参考官方案例

REST和Open APIs之上的gRPC
前面我们看到，不管是原生的RPC还是gRPC通信方式，都需要同时实现服务端跟客户端代码，感觉没有REST API来的方便，所以有没有可能将gRPC Server开放出来像REST API一样来访问呢？ 答案是肯定的。 在这里具体介绍下基于grpc-gateway库来实现此功能。

编辑hello.proto文件
syntax = "proto3";

package hello;

import "google/api/annotations.proto";

// The greeting service definution
service Greeter{
    // Send a greeting
    rpc SayHello(HelloRequest) returns (HelloReply){
        option(google.api.http) = {
            post: '/v1/example/hello'
            body: "*"
        };
    }
}

// The request message containing the user's name
message HelloRequest {
    string name = 1;
}

// the response message containing the greetings
message HelloReply{
    string message = 1;
}
代码也很简单，我们做的动作这是引入了google.api 然后在SayHello中定义了option, 我们定义的是一个post请求。

生成reverse-proxy代码
protoc -I/usr/local/include -I. \
  -I$GOPATH/src \
  -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
  --grpc-gateway_out=logtostderr=true:. \
  ./hello.proto
执行完以上命令之后，会生成一个hello.pb.gw.go的文件，我们通过此文件来完成proxy的代码。

proxy
var (
    helloEndPoint = flag.String("hello_endpoint", "localhost:9090", "endpoint of your service")
)
func Proxy() error  {
    ctx := context.Background()
    ctx, cancel := context.WithCancel(ctx)

    defer cancel()

    mux := runtime.NewServeMux()
    opts := []grpc.DialOption{grpc.WithInsecure()}

    err := pb.RegisterGreeterHandlerFromEndpoint(ctx, mux, *helloEndPoint, opts)
    if err != nil{
        return err
    }

    log.Println("The Server is running at 127.0.0.1:8888")
    return http.ListenAndServe(":8888", mux)
}
在代码中我们可以看到，我们在8888端口启动了proxy，然后将请求转发到了endpoint 9090，也就是我们gRPC的服务端端口，让服务端处理我们的请求逻辑，然后得到响应。

测试
首先我们启动服务端，然后启动proxy
通过postman工具来查看请求结果, 输出如下：


源码地址

总结
近年来，随着互联网、大数据以及区块链等行业的快速发展，分布式技术也越来越重要与流行。掌握分布式系统已经是一个优秀程序员所具备的必要技能(限领域),而RPC调用作为分布式系统间的调用方式也被越来越多的重视与使用。虽然在本篇中，我们用go语言介绍了如何实现RPC以及如何基于gRPC来实现RPC，但是以上内容还仅仅算分布式系统入门知识。要实现一个复杂的分布式系统，还有很多的路要走。本篇内容就到这里，后续会有更多关于分布式系统的知识，如CAP理论、DHT、一致性Hash、Raft等等，但这里我们并不是基于java技术栈。虽然Java很完善，也很强大，但这里我们聚焦于Go，聚焦与python。
