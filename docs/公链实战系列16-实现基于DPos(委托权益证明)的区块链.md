公链实战系列16-实现基于DPos(委托权益证明)的区块链

理论介绍
应用
代码实现
架构设计
代码实现
BlockChain模块实现
P2P模块实现
dpos模块代码实现
测试
总结
源码地址
参考
理论介绍
DPoS(Delegated Proof of Stake) 委托权益证明，是应用在区块链技术领域的一种新的共识算法。在DPoS算法里面区块由固定数量的生产者生产，这些生产者叫做producers 或者 witnesses. 现在经常听到的BP(Block Producer)或者超级节点其实就是DPoS算法中的区块生产者。在网络中的用户，可以将自己的票投给这些生产者，让其代替自己行事区块生产的权利，其机制类似与议会制度或人民代表大会制度。

应用
DPoS算法自BM提出之后，先后应用与BitShares(比特股)、 Steemit、Lisk、Ark、EOS、NAS等区块链项目中。可以说是目前证明有效的为数不多的分布式共识算法之一。

代码实现
在开始代码实现之前，首先梳理下代码的架构以及设计逻辑。需要强调的是，在这里实现的DPoS算法，也仅仅是一个简易实现。

架构设计
BlockChain模块
Block生产
Block校验
P2P模块
创建P2P节点
处理节点连接
同步区块信息
Tools模块
投票
Dpos模块
构建超级节点
选择节点(从超级节点中选择节点来生产Block)
代码实现
根据上面的架构设计，我们来依次实现上述模块，并进行测试。

BlockChain模块实现
// implement a simple p2p blockchain which use dpos algorithm.
// this file just for a simple block generate and validate.
// This file is created by magic at 2018-9-2

package dpos

import (
    "crypto/sha256"
    "encoding/hex"
    "time"
)

var BlockChain []Block

type Block struct {
    Index     int
    Timestamp string
    BPM       int
    Hash      string
    PrevHash  string
    validator string
}

// 计算string的hash值
func CaculateHash(s string) string {
    h := sha256.New()
    h.Write([]byte(s))
    hashed := h.Sum(nil)
    return hex.EncodeToString(hashed)
}

func CaculateBlockHash(block Block) string {
    record := string(block.Index) + block.Timestamp + string(block.BPM) + block.PrevHash
    return CaculateHash(record)
}

func GenerateBlock(oldBlock Block, BPM int, address string) (Block, error) {
    var newBlock Block
    t := time.Now()

    newBlock.Index = oldBlock.Index + 1
    newBlock.BPM = BPM
    newBlock.Timestamp = t.String()
    newBlock.PrevHash = oldBlock.Hash
    newBlock.Hash = CaculateBlockHash(newBlock)
    newBlock.validator = address

    return newBlock, nil
}

func IsBlockValid(newBlock, oldBlock Block) bool {
    if oldBlock.Index+1 != newBlock.Index {
        return false
    }

    if oldBlock.Hash != newBlock.PrevHash {
        return false
    }

    if CaculateBlockHash(newBlock) != newBlock.Hash {
        return false
    }
    return true
}
BlockChain模块逻辑比较简单，我们看到上述代码总共实现了四个方法。CaculateHash、CaculateBlockHash、GenerateBlock、IsBlockValid 分别是计算字符串hash，计算区块hash、生成区块以及区块校验。需要注意的是在Block里面的validator字段我们用于记录产生此区块的节点信息，这里是节点名称。

P2P模块实现
P2P模块的实现我们采用go语言的libp2p 这个库来实现，这一块如果之前没了解过，可能相对较难理解。对于之前没有接触过的同学，建议首先查看前面的文章P2P对等网络。

// This is the p2p network, handler the conn and communicate with nodes each other.
// this file is created by magic at 2018-9-2

package dpos

import (
    "io"
    "fmt"
    "os"
    "strings"
    "strconv"
    "time"
    "crypto/rand"
    "flag"
    "log"
    "bufio"
    "sync"
    "encoding/json"
    "context"
    mrand "math/rand"
    "github.com/libp2p/go-libp2p-net"
    "github.com/libp2p/go-libp2p-host"
    "github.com/libp2p/go-libp2p-crypto"
    "github.com/libp2p/go-libp2p"
    "github.com/libp2p/go-libp2p-peer"
    ma "github.com/multiformats/go-multiaddr"
    pstore "github.com/libp2p/go-libp2p-peerstore"
    "github.com/davecgh/go-spew/spew"
)

const DefaultVote = 10
const FileName = "config.ini"

var mutex = &sync.Mutex{}

type Validator struct {
    name string
    vote int
}

func MakeBasicHost(listenPort int, secio bool, randseed int64) (host.Host, error) {
    var r io.Reader

    if randseed == 0 {
        r = rand.Reader
    } else {
        r = mrand.New(mrand.NewSource(randseed))
    }

    // 生产一对公私钥
    priv, _, err := crypto.GenerateKeyPairWithReader(crypto.RSA, 2048, r)
    if err != nil {
        return nil, err
    }

    opts := []libp2p.Option{
        libp2p.ListenAddrStrings(fmt.Sprintf("/ip4/127.0.0.1/tcp/%d", listenPort)),
        libp2p.Identity(priv),
    }

    if !secio {
        opts = append(opts, libp2p.NoSecurity)
    }
    basicHost, err := libp2p.New(context.Background(), opts...)
    if err != nil {
        return nil, err
    }

    // Build host multiaddress
    hostAddr, _ := ma.NewMultiaddr(fmt.Sprintf("/ipfs/%s", basicHost.ID().Pretty()))

    // Now we can build a full multiaddress to reach this host
    // by encapsulating both addresses;
    addr := basicHost.Addrs()[0]
    fullAddr := addr.Encapsulate(hostAddr)

    log.Printf("我是: %s\n", fullAddr)
    SavePeer(basicHost.ID().Pretty())

    if secio {
        log.Printf("现在在一个新终端运行命令: 'go run main/dpos.go -l %d -d %s -secio'\n ", listenPort+1, fullAddr)
    } else {
        log.Printf("现在在一个新的终端运行命令: 'go run main/dpos.go -l %d -d %s '", listenPort+1, fullAddr)
    }
    return basicHost, nil
}

func HandleStream(s net.Stream) {
    log.Println("得到一个新的连接!", s.Conn().RemotePeer().Pretty())
    // 将连接加入到
    rw := bufio.NewReadWriter(bufio.NewReader(s), bufio.NewWriter(s))
    go readData(rw)
    go writeData(rw)
}

func readData(rw *bufio.ReadWriter) {
    for {
        str, err := rw.ReadString('\n')
        if err != nil {
            log.Fatal(err)
        }

        if str == "" {
            return
        }
        if str != "\n" {
            chain := make([]Block, 0)

            if err := json.Unmarshal([]byte(str), &chain); err != nil {
                log.Fatal(err)
            }

            mutex.Lock()
            if len(chain) > len(BlockChain) {
                BlockChain = chain
                bytes, err := json.MarshalIndent(BlockChain, "", " ")
                if err != nil {
                    log.Fatal(err)
                }

                fmt.Printf("\x1b[32m%s\x1b[0m> ", string(bytes))
            }
            mutex.Unlock()
        }
    }
}

func writeData(rw *bufio.ReadWriter) {
    // 启动一个协程处理终端同步
    go func() {
        for {
            time.Sleep(2 * time.Second)
            mutex.Lock()
            bytes, err := json.Marshal(BlockChain)
            if err != nil {
                log.Println(err)
            }
            mutex.Unlock()

            mutex.Lock()
            rw.WriteString(fmt.Sprintf("%s\n", string(bytes)))
            rw.Flush()
            mutex.Unlock()
        }
    }()

    stdReader := bufio.NewReader(os.Stdin)
    for {
        fmt.Print(">")
        sendData, err := stdReader.ReadString('\n')
        if err != nil {
            log.Fatal(err)
        }

        sendData = strings.Replace(sendData, "\n", "", -1)
        bpm, err := strconv.Atoi(sendData)
        if err != nil {
            log.Fatal(err)
        }

        // pick选择block生产者
        address := PickWinner()
        log.Printf("******节点%s获得了记账权利******", address)
        lastBlock := BlockChain[len(BlockChain)-1]
        newBlock, err := GenerateBlock(lastBlock, bpm, address)
        if err != nil{
            log.Fatal(err)
        }

        if IsBlockValid(newBlock, lastBlock){
            mutex.Lock()
            BlockChain = append(BlockChain, newBlock)
            mutex.Unlock()
        }

        spew.Dump(BlockChain)

        bytes, err := json.Marshal(BlockChain)
        if err != nil {
            log.Println(err)
        }
        mutex.Lock()
        rw.WriteString(fmt.Sprintf("%s\n", string(bytes)))
        rw.Flush()
        mutex.Unlock()
    }
}

func Run() {

    t := time.Now()
    genesisBlock := Block{}
    genesisBlock = Block{0, t.String(), 0, CaculateBlockHash(genesisBlock), "", ""}
    BlockChain = append(BlockChain, genesisBlock)

    // 命令行传参
    listenF := flag.Int("l", 0, "等待节点加入")
    target := flag.String("d", "", "连接目标节点")
    secio := flag.Bool("secio", false, "打开secio")
    seed := flag.Int64("seed", 0, "生产随机数")
    flag.Parse()

    if *listenF == 0 {
        log.Fatal("请提供一个端口号")
    }

    // 构造一个host 监听地址
    ha, err := MakeBasicHost(*listenF, *secio, *seed)
    if err != nil {
        log.Fatal(err)
    }

    if *target == "" {
        log.Println("等待节点连接...")
        ha.SetStreamHandler("/p2p/1.0.0", HandleStream)
        select {}
    } else {
        ha.SetStreamHandler("/p2p/1.0.0", HandleStream)
        ipfsaddr, err := ma.NewMultiaddr(*target)
        if err != nil {
            log.Fatal(err)
        }
        pid, err := ipfsaddr.ValueForProtocol(ma.P_IPFS)
        if err != nil {
            log.Fatal(err)
        }

        peerid, err := peer.IDB58Decode(pid)
        if err != nil {
            log.Fatal(err)
        }

        targetPeerAddr, _ := ma.NewMultiaddr(
            fmt.Sprintf("/ipfs/%s", peer.IDB58Encode(peerid)))
        targetAddr := ipfsaddr.Decapsulate(targetPeerAddr)

        // 现在我们有一个peerID和一个targetaddr，所以我们添加它到peerstore中。 让libP2P知道如何连接到它。
        ha.Peerstore().AddAddr(peerid, targetAddr, pstore.PermanentAddrTTL)
        log.Println("打开Stream")

        // 构建一个新的stream从hostB到hostA
        // 使用了相同的/p2p/1.0.0 协议
        s, err := ha.NewStream(context.Background(), peerid, "/p2p/1.0.0")
        if err != nil {
            log.Fatal(err)
        }

        rw := bufio.NewReadWriter(bufio.NewReader(s), bufio.NewWriter(s))

        go writeData(rw)
        go readData(rw)
        select {}
    }
}

func SavePeer(name string) {
    vote := DefaultVote // 默认的投票数目
    f, err := os.OpenFile(FileName, os.O_WRONLY|os.O_APPEND|os.O_CREATE, 0666)
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()


    f.WriteString(name + ":" + strconv.Itoa(vote) + "\n")

}
在这里我们实现了MakeBasicHost、HandleStream、readData、writeData、Run、SavePeer 其中MakeBasicHost、HandleStream、readData、writeData、Run 四个方法是使用p2plib时需要实现的方法，
所以这里不过多介绍，如果有不理解的同学可以查看libp2p文档或者P2P对等网络来了解。 这里主要介绍SavePeer, 在这里为了能够实现后面的节点投票与BP选择，我们将每个连接进来的P2P节点信息保存到了文件中，文件名称为config.ini。这里需要注意，我们是追加写入，也就是说，每加入一个节点，就多记录一条信息。 另外，如果你仔细查看了以上代码，或许你已经注意到了下面的代码:

// pick选择block生产者
address := PickWinner()
log.Printf("******节点%s获得了记账权利******", address)
lastBlock := BlockChain[len(BlockChain)-1]
newBlock, err := GenerateBlock(lastBlock, bpm, address)
if err != nil{
    log.Fatal(err)
}
在Block生成的时候，我们用PickWinner方法来选择区块的生成者。关于PickWinner的实现，我们放到了Dpos模块中，接下来我们来看看Dpos模块代码实现。

dpos模块代码实现
package dpos

import (
    "io/ioutil"
    "log"
    "strings"
    "strconv"
    "sort"
)

const BPCount = 5

func PickWinner() (bp string) {
    // 选择BlockProducer

    f, err := ioutil.ReadFile(FileName)
    if err != nil {
        log.Fatal(err)
    }

    res := strings.Split(string(f), "\n")

    voteList := make([]int, len(res))
    voteMap := make(map[string]int)
    for _, node := range res {
        nodeSplit := strings.Split(node, ":")
        if len(nodeSplit) > 1 {
            vote, err := strconv.Atoi(nodeSplit[1])
            if err != nil {
                log.Fatal(err)
            }
            voteList = append(voteList, vote)
            voteMap[nodeSplit[0]] = vote
        }
    }
    sort.Slice(voteList, func(i, j int) bool {
        return voteList[i] > voteList[j]
    })

    if len(voteList) > BPCount {
        voteList = voteList[0:BPCount] // 选择前面的5个节点作为Block producer
    }

    for k, v := range voteMap {
        if v > voteList[len(voteList)-1] {
            bp = k
        }
    }
    return
}
在代码中我们定义了一个BPCount的参数，这个参数表示BP的数量，也就是超级节点的数量。当然这个数字由你自己定义，我这里由于测试中，为了不启动过多的节点，所以这里设置了5，你完全可以像EOS一样设置成21个，然后启动100个节点，投票产生此21个节点。了解了这点，我们来看看超级节点的投票逻辑。之前讲到超级节点的信息我们存在了文件中，所以这里我们从文件中读取所有加入到网络中的节点信息，然后根据vote(票数)对节点进行排序，排列在最前面的5个节点作为我们的BP(区块生产者)，由他们来负责区块的生成。

### 投票
实现以上部分，一个简易的DPoS链基本可以运行起来了。但是我们还差最关键的一步——投票。 既然DPoS算法类似与人民代表大会制度，显然不能少了投票环节。

package tools

import (
    "flag"
    "log"
    "fmt"
    "os"
    "io/ioutil"
    "strings"
    "strconv"
    "github.com/csunny/argo/src/dpos"

)

func Vote() {
    name := flag.String("name", "", "节点名称")
    vote := flag.Int("v", 0, "投票数量")
    flag.Parse()

    if *name == "" {
        log.Fatal("节点名称不能为空")
    }

    if *vote < 1 {
        log.Fatal("最小投票数目为1")
    }

    f, err := ioutil.ReadFile(dpos.FileName)
    if err != nil {
        log.Fatal(err)
    }

    res := strings.Split(string(f), "\n")

    voteMap := make(map[string]string)
    for _, node := range res {
        nodeSplit := strings.Split(node, ":")
        if len(nodeSplit) > 1 {
            voteMap[nodeSplit[0]] = fmt.Sprintf("%s", nodeSplit[1])
        }
    }

    originVote, err := strconv.Atoi(voteMap[*name])
    if err != nil {
        log.Fatal(err)
    }
    votes := originVote + *vote
    voteMap[*name] = fmt.Sprintf("%d", votes)

    log.Printf("节点%s新增票数%d", *name, votes)
    str := ""
    for k, v := range voteMap {
        str += k + ":" + v + "\n"
    }

    file, err := os.OpenFile(dpos.FileName, os.O_RDWR, 0666)
    file.WriteString(str)
}
投票的逻辑也很简单，就是用户可以指定节点信息，然后对其进行投票。节点票数遵守以下数学公式。

节点的票数 = 原来票数 + 用户投票数
到这里，一个完整的DPoS算法的区块链就完成了，接下来我们测试一下。是不是跟我们预想的效果一样。

测试
运行如下命令，启动P2P网络

go run main/dpos.go -l 3000 -secio
输出如下

2018/09/04 19:18:39 我是: /ip4/127.0.0.1/tcp/3000/ipfs/QmP6d6Wx3V4goPjVCKMT5HwuDdWd3NtQ9ESrW1fNR6bk3B
2018/09/04 19:18:39 现在在一个新终端运行命令: 'go run main/dpos.go -l 3001 -d /ip4/127.0.0.1/tcp/3000/ipfs/QmP6d6Wx3V4goPjVCKMT5HwuDdWd3NtQ9ESrW1fNR6bk3B -secio'

2018/09/04 19:18:39 等待节点连接...

新开一个窗口，运行提示的命令:

go run main/dpos.go -l 3001 -d /ip4/127.0.0.1/tcp/3000/ipfs/QmP6d6Wx3V4goPjVCKMT5HwuDdWd3NtQ9ESrW1fNR6bk3B -secio
以此运行命令，多启动几个节点，我这里启动8个节点用来测试。


随意找一个终端，输入bpm值，比如5回车，查看输出。


测试投票

go run main/vote.go -name QmWnLMG4sVkvzKpShamvwZXefaaj6HowD2dXHawQSrFdS2 -v 90



总结
本篇内容主要介绍了DPoS算法，以及如何实现一条简单的基于DPoS算法的区块链。DPoS算法是相当重要的一种共识算法，为了充分模拟它的实现，我们利用了P2P网络，模拟了投票以及区块同步，可以算是比较完整的实现了。 当然离生产还差的比较远。

源码地址
https://github.com/csunny/dpos
