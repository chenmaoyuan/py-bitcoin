第一部分 EOS快速入门

简介
EOS架构简介
快速安装
Docker安装
Docker-Compose
Docker设置
构建EOS镜像
启动服务
测试
参考
简介
EOS(Enterprise Operating System)是由Block.One公司开发的一款分布式应用操作系统。它的愿景是成为最强大的分布式应用基础设施。要达到企业级应用系统的要求，性能是非常重要的一环。所以一开始其主要开发者BM就宣称EOS将实现跨CPU进行并行计算，最终达到每秒数百万的TPS。从2017年开发到18年6月上线，可谓一时风头无两。截止目前EOS已经稳定运行了两个多月，虽然目前为止TPS最高点到4000+tps/s, 但EOS不管是架构设计，还是技术细节都值得我们去探索，去学习。

EOS架构简介
如图所示，EOS主要有三部分组成

nodeos （node+eos=nodeos）是EOSIO操作系统的核心守护进程，可以配置插件来运行节点。 例如: 块生产者(BP或超级节点)、API节点或用于本地开发
cleos (cli+eos=cleos) 与区块链交互和管理钱包的命令行工具
keosd (key+eos=keosd) EOSIO钱包

此三者之间的关系，如架构图所示。当然，这仅仅是一个非常简单的架构图，关于EOSIO, 架构图拆开了有很多东西。当然关于EOS的完整架构图以及它的设计哲学我们放在后面慢慢梳理。这里先从简单的教程来上手。
快速安装
这里特别说明下，后面的安装以及测试学习我这里打算更多的用Docker来进行。一个原因是方便，另一个核心的原因是整个教程后面，我们会使用docker来启动一个完整的EOS公链，节点数目可能会超过50个, 所以这里采用docker来启动的原因也是希望大家一方面可以熟悉docker，另一方面，我们后面可以用docker来模拟一个完整的EOS公链架构的部署。

接下来我们看看如何利用Docker来安装EOSIO

Docker安装
关于docker的安装我这里不过多的讲述，如果有问题的同学可以通过Docker安装官方教程来进行。 如果这个教程看着有问题或者安装出问题，可以给我留言。

Docker-Compose
早前的Docker版本在非Mac系统下，安装完Docker不会有Docker-compose，我们需要用到Docker-compose。所以如果没有docker-compose命令的话。

pip install docker-compose 
就可以安装docker-compose
安装成功测试如下:


Docker设置
由于在整个过程中，需要构建镜像，对docker内存有一定要求。官方建议设置为7GB。
Mac下设置较为简单。打开Docker依次选择Preferences -> Advanced -> Memory -> 7GB或者更大值


其他系统可以参考这里

构建EOS镜像
下载源代码并构建镜像。

git clone https://github.com/EOSIO/eos.git --recursive  --depth 1
cd eos/Docker
docker build . -t eosio/eos
如果大家遇到构建镜像比较慢，或者构建失败的情况，可以直接下载镜像。命令如下:

docker pull eosio/eos   
当然如果需要特定的版本的话直接pull特定版本就可以，我们这里都默认为最新版本。

启动服务
做完前面的准备工作，接下来我们来实际运行一个单节点的EOS。

docker volume create --name=nodeos-data-volume
docker volume create --name=keosd-data-volume
docker-compose up 
注意，此命令需要在具有docker-compose.yml的路径下才有效。


测试
docker-compose exec keosd /opt/eosio/bin/cleos -u http://nodeosd:8888 --wallet-url http://localhost:8900 get info


输入如上命令进行验证，输出如下说明启动成功。另外，我们还可以通过浏览器来进行验证。打开浏览器输入如下链接：http://localhost:8888/v1/chain/get_info


在我们使用命令行进行交互的时候，为了少敲一点内容，我们做一个别名吧。

alias cleos='docker-compose exec keosd /opt/eosio/bin/cleos -u http://nodeosd:8888 --wallet-url http://localhost:8900'
好了，现在我们不需要敲那么长的命令了，只需要

cleos get info
就可以验证了。

截止这里，我们的单节点Docker启动就完成了，接下来大家可以自己尝试下各种命令。熟悉cleos的用法。

第二部分 EOS钱包&账号


理论介绍
cleos 命令使用
钱包创建
创建一个默认钱包
创建一个名字为Magic的钱包
查看钱包
钱包加锁
钱包解锁
生成EOSIO Keys
查看启动的容器服务
创建账号
总结
参考
理论介绍
提起钱包，我相信大家肯定都听过这个词汇，也一定对它有自己的认识和理解。但是在我们这里的钱包跟宽泛的钱包概念有所区别，严格来说它是数字钱包，不是纸钱包也不是皮钱包。

如果大家看过我前面的教程或者对比特币有所了解，相信大家也或多或少了解过数字钱包。因为这是你能够拥有比特币的先决条件。当然，EOS作为一款公链，要解决交易的问题，那么钱包无疑是必不可少的一个基础设施。在区块链里，用户身份或者用户的钱包是利用密码学原理，由一对公私钥来唯一确定的。可以简单认为一对公、私钥就组成了一个数字钱包，且是唯一的。由于目前市面上绝大多数的钱包都是利用椭圆加密算法来生成的，所以在数字钱包上都大同小异。

当然，跟我们之前接触的比特币或以太坊不同，EOS在钱包的基础上进一步提出来账号体系。而且，就我所知，EOS账号像网站域名一样，已经出现了价值不菲的账户，最高的甚至达到了数百上千万。那么账号到底是什么？

一个账号是存储在区块链上的方便人们读的名称。它可以被个人或者是一个组织所拥有，这主要是根据权限配置来决定的。账号可以用于在区块链上发送和接收交易。

一个EOS账号由两个密钥组成, active密钥和owner密钥匙。active密钥可以用来交易、投票、买卖内存等等。owner密钥表示账户的所有权。owner密钥是非常重要，最好离线保存。

cleos 命令使用
提醒: 在下面的钱包创建过程中，生成的密码或者密钥对请做好记录，方便后续操作。

钱包创建
创建一个默认钱包
cleos wallet create --to-console

$ cleos wallet create --to-console
Creating wallet: default
Save password to use in the future to unlock this wallet.
Without password imported keys will not be retrievable.
"There is your password, please note!"
创建一个名字为Magic的钱包
cleos wallet create -n magic --to-console

$ cleos wallet create -n periwinkle --to-console
Creating wallet: magic
Save password to use in the future to unlock this wallet.
Without password imported keys will not be retrievable.
"这是你的钱包密码，请记录！"
查看钱包
cleos wallet list

Wallets:
[
  "default *",
  "magic *"
]
钱包加锁
cleos wallet lock -n magic

钱包解锁
cleos wallet unlock -n magic
pwd: ******

这里需要注意, 当我们重新启动keosd服务之后，我们首先需要打开钱包, 然后才能查看钱包。
也就是首先需要执行命令：
cleos wallet open 否则将看不到钱包。

生成EOSIO Keys
我们前面提到一个账户由active key和owner key。 接下来我们生成这两组密钥对。执行命令
cleos create key --to-console

$ cleos create key --to-console
Private key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Public key: EOSXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
$ cleos create key --to-console
Private key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Public key: EOSXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
如上所示，我们会得到两组密钥对，将其复制保存下来。 当然，也可以直接使用保存到文件的命令
cleos create key --file active_key.txt
现在我们拥有两对EOSIO密钥对，但是截止目前还没有被认证。接下来，我们导入私钥到默认钱包中。为了完成此步骤，我们需要执行以下命令。
cleos wallet import --private-key 1
cleos wallet import --private-key 2
如果提示你的钱包处于锁定状态，执行解锁命令解锁即可。


接下来，我们检查是否引入key成功了，执行下面的命令进行检查

$ cleos wallet keys
[
    "EOS6....",
    "EOS3...."
]
$ cleos wallet private_keys --password YOUR WALLET PASSWORD
password:
[[
    "EOS6....",
    "5KQwr..."
  ],
  [
    "EOS3....",
    "5Ks0e..."
  ]
]
如果观察到上面的输出，说明成功了，接下来对钱包做一个备份吧。 对钱包文件做备份是相当重要的。
由于我们是用的docker，所以这里有一点特殊。 首先我们来看看源代码里面的docker-compose.yml文件。

version: "3"
...
    volumes:
      - keosd-data-volume:/opt/eosio/bin/data-dir
...      
这一步我们只想备份钱包文件，所以其他的信息我们暂时不看，我们只查看钱包文件在什么地方。从文件中我们可以看到，docker通过volume将文件映射到了容器中的/opt/eosio/bin/data-dir地址。接下来我们来看看data-dir文件夹下都有哪些文件？

查看启动的容器服务
docker ps -a


可以看到我们启动了两个容器，nodeosd、keosd。在我们前面架构介绍中，我们已经分别讲述了这两个服务的作用。下面我们来看看钱包文件。

docker exec -it docker_keosd_1 ls /opt/eosio/bin/data-dir

我们可以看到default.wallet 跟magic.wallet 两个文件。如果要备份的话，将此两个文件备份即可。

创建账号
上面我们讲述了钱包的创建以及操作，接下来我们来看看如何创建账号。 当然，这里有一点需要说明，我们目前的操作都是在本地的测试环境，并没有链接到EOS主网。如果要在EOS主网创建账号是需要一定的费用的。当然费用主要是用于钱包文件的存储。

在测试环境创建账号之前，我们首先需要将eosio的私钥导入到我们的钱包
cleos wallet import --private-key 5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3
接下来执行创建账号的命令
cleos create account eosio NEW_ACCOUNT OWNER_KEY ACTIVE_KEY

 cleos create account eosio myaccount  EOS6Enq97tcgJixTPt9Fk7hCxUhG2HJyckJnvVYhpBbfWgweiadJm EOS828qVkEn4jdMYQWxrByjgNmB8oJHnZfpsgPG66fYnwF59Z3gLv
如果看到如下输出，说明创建账号成功了。


接下来我们来查看我们创建的账户
cleos get account myaccount


总结
本篇对EOS的钱包和账号进行了一个简单的介绍，以及如何在自己的测试网络中创建钱包以及账号。当然在生产环境中的创建也大同小异，只不过你需要连接到生产网络中。另外关于账号的创建有很多途径，可以通过我们文章中提到的命令行的方式，也可以通过web钱包或者app钱包抑或是浏览器插件钱包进行创建。EOS账号是使用EOS系统的关键，没有EOS账号也就不能在EOS系统上进行交易或者其他操作。当然EOS人性化的设计是一个亮点，至于它让人眼花缭乱的玩法就仁者见仁，智者见智了。

第三部分 连接EOS主网并创建主网账号

理论介绍
主网启动
EOS主网
启动一个本地钱包节点
连接到EOS主网
创建EOS主网账号
在主网上熟悉一些常用的操作
总结
参考
理论介绍
EOS作为区块链公链领域的新星，争议一直不断。 尽管前面我们已经花了很多的内容介绍EOS，但仅凭前面的内容还远远不够。今天我们来谈谈EOS主网、EOS主网上线风波、以及如何利用本地钱包连接EOS主网并创建账号。

主网启动
谈到EOS，不可避免的我们要讲讲Block.one公司。Block.one公司是由CEO Brendan Blumer和CTO Daniel Larimer领导的一家高科技软件公司。他们以“建立一个更加安全和互联的世界”为愿景(Building a more secure and connected world)。EOS正是由Block.one公司开发的一款开源软件系统。由于Brendan Blumer 和BM的知名度，EOS从一开始就备受关注。但block.one公司宣称，Block.one公司只是EOS的开发者，Block.one不会启动任何一条主网。EOS主网可以由任何人启动，也就意味着市面上可以存在很多条EOS主网。Block.one一开始就将EOS的命运交给了社区。当然，也正是因此，EOS主网上线经历了很多风波与挑战。

2018年6月，是一个燥热的月份。这份燥热不只是天气，还有扎堆上线的公链，还有躁动的币民，还有意料之外的大佬的声音。6月EOS开源版本v1.0.0发布, Block.one声明EOS完成上线前测试，可以正式上线。

新事物的发展，总是不会一帆风顺，EOS主网启动也不例外。在EOS上线之际，社区一开始也出现了好多声音。最典型的就是以BM、佳能、starteos、helloeos、eosnewyork为代表的社区与eos原力为代表的社区之间的矛盾。 后来众多社区达成一致，最后宣布只启动一条主网，标志着EOS主网启动正式开始。 （当然之后EOS原力单独启动一条主网就是后话了。）

在社区达成共识不久，以360为首的安全公司，在社区披露了EOS重大漏洞。由于区块链技术的高安全性要求以及360的影响力，一时间人心惶惶。毫无疑问，由于安全事件，EOS主网上线时间有所推迟，但最终经过社区以及BP(block producer)的共同努力，在经过几天的奋战之后，EOS主网正式上线了。

但，对于EOS主网来讲，仅仅做到这里还不够。 在EOS的系统里采用的是Dpos的共识算法。在前面我们介绍了POW算法、Pos算法。这里简单介绍下Dpos算法。 Dpos(Delegated proof of stake) 委托权益证明。如果看了前面的教程或许对这个词不是那么陌生，因为其比Pos(proof of stake)就多了一个Delegated（委托）。 顾名思义，既然Pos是持有更多token的人说了算，那么Dpos是就是普通用户将自己的token委托给一些代表，让其有话语权，代表自己发声。所以为了实现委托的目的，EOS系统设计了投票机制。EOS主网设置了21个BP（block producer），普通EOS持有者可以利用自己的Token对众多的候选人进行投票，最终产生21个超级节点，也就是21个BP来出块，并且EOS主网规定投票率需要超过总token的15%才能开始出块。

经过两周的投票，投票率终于超过了15%，前面的21个超级节点也如期产生。EOS第一个区块的产生，标志着EOS系统正式运行，主网上线成功，用户可以在EOS上进行交易、合约开发、账号创建等一系列活动。


EOS主网
在连接EOS主网之前，需要启动一个本地钱包，在这里我们依旧用docker来做演示。

启动一个本地钱包节点
在我们之前的教程中，我们启动了两个容器，一个是keosd, 一个是nodeosd. keosd就是一个钱包节点。所以，接着我们之前的内容，我们将nodeos容器关掉, 首先查看容器
docker ps -a


docker stop docker_nodeosd_1 
这样我们就只有一个keosd的钱包节点在运行了。当然，如果没有了解前面的内容，也没有关系，下面我演示下，如何在没有看前面教程的基础上启动一个本地keosd钱包。首先拉取eos源码

git clone https://github.com/EOSIO/eos.git
cd eos/Docker
mv docker-compose.yml docker-compose.yml.bak
docker volume create --name=keosd-data-volume
重新打开一个docker-compose.yml文件, 并将如下内容粘贴进去

version: "3"

services:
  keosd:
    image: eosio/eos
    command: /opt/eosio/bin/keosd --wallet-dir /opt/eosio/bin/data-dir --http-server-address=127.0.0.1:8900 --http-alias=keosd:8900 --http-alias=localhost:8900
    hostname: keosd
    volumes:
      - keosd-data-volume:/opt/eosio/bin/data-dir
    stop_grace_period: 10m

volumes:
  keosd-data-volume:
    external: true
接下来，运行docker-compose命令启动本地钱包容器。

docker-compose up -d
如上，我们就启动了一个本地钱包服务。


连接到EOS主网
接下来，我们来连接到主网上。跟主网进行通信。运行如下命令:

alias cleos='docker-compose exec keosd /opt/eosio/bin/cleos -u http://mainnet.genereos.io --wallet-url http://localhost:8900'
或许你已经看到了，在-u之后的参数，已经不是我们《EOS快速入门》里面的本地地址了，我们这个是主网的节点地址。
前面我们介绍了，主网有21个超级节点，所以这个地址不是唯一的，有很多可以选择。下面是一个列表:

Producer	Key	url
eoslaomaocom	EOS8QgURqo875qu3a8vgZ58qBeu2cTehe9zAWRfpdCXAQipicu1Fi	https://eoslaomao.com
starteosiobp	EOS4wZZXm994byKANLuwHD6tV3R3Mu3ktc41aSVXCBaGnXJZJ4pwF	https://www.starteos.io
eos42freedom	EOS4tw7vH62TcVtMgm2tjXzn9hTuHEBbGPUK2eos42ssY7ip4LTzu	https://www.eos42.io
zbeosbp11111	EOS7rhgVPWWyfMqjSbNdndtCK8Gkza3xnDbUupsPLMZ6gjfQ4nX81	http://www.zbeos.com
eoshuobipool	EOS5XKswW26cR5VQeDGwgNb5aixv1AMcKkdDNrC59KzNSBfnH6TR7	http://eoshuobipool.com
eosnewyorkio	EOS6GVX8eUqC1gN1293B3ivCNbifbr1BT6gzTFaQBXzWH9QNKVM4X	https://bp.eosnewyork.io
eosfishrocks	EOS4xAhqjLs5fsCzw7QsGYJtRDVo3t2xRFCHd7amGEWwhjSb5DPZT	https://bp.fish
bitfinexeos1	EOS4tkw7LgtURT3dvG3kQ4D1sg3aAtPDymmoatpuFkQMc7wzZdKxc	https://www.bitfinex.com
jedaaaaaaaaa	EOS6nB9Ar5sghWjqk27bszCiA9zxQtXZCaAaEkf2nwUm9iP5MEJTT	https://www.eosjapan.org
eosasia11111	EOS76gG6ATpqfVf5KrVjh3f4JAa4EKzAwWabTucNQ4Xv2TmVAj9bN	https://www.eosasia.one
eosauthority	EOS4va3CTmAcAAXsT26T3EBWqYHgQLshyxsozYRgxWm9tjmy17pVV	https://eosauthority.com
eoscannonchn	EOS73cTi9V7PNg4ujW5QzoTfRSdhH44MPiUJkUV6m3oGwj7RX7kML	https://eoscannon.io
eoscannonchn	EOS73cTi9V7PNg4ujW5QzoTfRSdhH44MPiUJkUV6m3oGwj7RX7kML	https://eoscannon.io
这里就列举这么多，更多的url后面教大家在命令行里面查看。

创建EOS主网账号
在创建账号之前，首先需要创建本地钱包，关于本地钱包跟账号的创建，这里就不赘述了。如果有不了解的朋友，可以移步《EOS快速入门》和《EOS 钱包&账号》进行了解。

到这一步大家都已经连接到主网，并且创建好自己的钱包了，那么接下来就是非常关键的步骤。EOS账号的创建，这里需要强调一点，在EOS上创建账号之前首先需要具有EOS账号。这里是不是感觉很困惑？ 我相信是有人有困惑，如果你没有EOS账号，打算创建账号的话，需要具有EOS账号的人帮忙创建。因为在EOS上创建账号并不是免费的。是需要花费一定的EOS token。如果你之前在EOS上没有账号，那么就没法付款，没法付款也就不能创建账号。嗯，大概理解就这么个意思。

所以如果事先没有EOS账号，需要创建的话，要其他具有EOS账号的人帮忙创建才可以。另外，创建账号的时候只需要提供自己本地生成钱包的公钥地址就可以了。
具体操作如下：

cleos system newaccount [已有账号] [新注册账户名] [账户公钥] --stake-net '0.0001 EOS' --stake-cpu '0.001 EOS' --buy-ram-kbytes 3 
上面的已有账号就是前面我们介绍的已经具有的账号名，新注册账号名是你要创建的账号名称（注意需要12位，并且由12345abcdefghijklmnopqrstuvwxyz字符构成）
账号公钥就是本地通过cleos create key 生成的公钥。另外在EOS上有三种资源，分别为net、cpu、ram. 所以一开始需要设定此三个参数。

当然也可以直接通过命令行来查看操作参数


以上就是在EOS主网上通过命令行创建账号，当然还有其他方式，这里就不一一介绍了。我相信作为一个geek，你一定最喜欢这种方式，一切尽在掌控之中。没有中间商赚差价！

在主网上熟悉一些常用的操作
既然我们链接到了EOS主网，那么来搞点事情吧。
首先我们来查看下主网信息
cleos get info

查看cleos system命令
cleos system 


列举所有的block producer
cleos system listproducers


查看账号信息


当然还有其他很多有趣的命令，大家可以自己尝试下，这里就不一一介绍了。

总结
今天的内容主要介绍了下EOS的主网上线风波以及如何本地启动一个节点，连接到EOS主网并创建账号，内容本身并不复杂。 这一节的内容相当有用，因为EOS账号是使用EOS系统的第一步。

