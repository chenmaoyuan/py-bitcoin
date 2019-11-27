公链实战系列6-API层接口实现&区块浏览

理论介绍
Flask框架介绍
实现一个基于Flask的restful API
实现一个TCP服务器
实现一个TCP客户端
Flask源码分析
实现基于Flask的API层
查看响应结果
项目地址
参考
理论介绍
前面我们详细的讲解了，如何利用POW算法来生成区块，并将这些区块存储在内存中或者是做持久化存储。也就是说，到这里我们已经可以生成一条完整的链了。然并卵！ 因为一开始在讲整个架构的时候，我们就有讲，在比特币中，一个区块里面有四个很重要的信息。

Timestamp
PrevHash
Hash
Transactions
想必你也知道了，在我们的链上，实际上是没有Transactions的，也就是缺失了交易信息。 比特币作为一种电子货币，交易是其本质的属性，没有了交易，“币” 也就无从谈起，当然了其他一切的什么信任啊，工作量证明啊什么的，也就都毫无意义。 所以交易是核心，不管是在传统金融领域，还是在数字货币领域，都是围绕着交易这两个词来展开的。

而在比特币中，关于交易的设计又是一个值得好好讲讲的话题，大概的介绍在开篇的架构介绍里面也有所提及。今天这里我们先引出来，但是不会放在这里讲，在正式进入核心之前，我们先缓一缓，先聊点其他的话题——API层。

提到API相信大家对这个词都不会陌生，因为有一种很流行的设计模式——面向接口设计。相信大家都看了《Head First设计模式》（没看的同学可以考虑看一看）

这里我们的API层是我们的系统跟外界通信的窗口，由API层负责跟外界交互，以及提供一些必要的查询、交易等基础的功能。由于目前我们还没有实现P2P对等网络，同时也没有实现RPC（远程过程调用），所以在这里我们采用传统的C/S的架构，来实现API层的功能。 后面会更新为基于gRPC的方式。 当然这里也只是留个引子，大家知道后面我们会这么做就ok啦！

前面介绍了那么多，接下来我们进入正题，来讲一讲如何利用C/S的架构实现一个API层以及区块浏览的功能。因为在本教程里面，我们使用的语言是python，为了尽可能的少引入新概念，以及最大限度的降低教程的难度，同时又能让大家了解到更多的知识，在这里我采用Flask的web框架，

使用框架的原因有以下几点

降低开发难度
提高开发效率
尽可能少的关注底层的协议
但是为什么要采用flask？
正如前面提到的

Flask使用足够简单
Flask框架非常轻量
Flask采用模块化的设计，你需要什么就引入什么，而不用一开始就初始化一大堆的东西，其实最后发现很多东西都没什么用，在python里面最典型的就是django！

既然选择了Flask，那首先让我们看看它怎么用，另外顺便介绍一下Flask的框架的一些深入的东西。

首先我们来看一个最简单的Hello world 程序！

Flask框架介绍
实现一个基于Flask的restful API
# server.py
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello World!”

if  __name__ == “__main__”:
    app.run(port=8888, debug=True)
是不是足够简单？ 这感觉要比用Java或者C++写个Hello world还简单！是的，有时候python就是这么任性！

说完废话，我们来看看在这一个简单的Hello World里面flask都做了哪些事情？

在进入flask的源码之前，我们首先回顾下，一个web应用的本质是什么？

简单理解就是client基于Http请求发送一个request，服务器收到request请求之后经过一系列的处理生成一个response.

这样解释看着是一个很简单的操作，那么我们不妨想一想，如果我们不用框架，我们自己写一个服务器，需要哪些工作要做？

一般来说，我们通常使用到的服务器，绝大多数都是采用的TCP或者是UDP协议，当然不同的业务场景需求选择有所差别。(至于如何选择，大家可以自己了解下） 这里我们为了举例子来实现一个TCP协议的服务器。

实现一个TCP服务器
from socketserver import BaseRequestHandler, TCPServer

class EchoHandler(BaseRequestHandler):
    def handle(self):
        print('Got connection from', self.client_address)
        while True:

            msg = self.request.recv(8192)
            if not msg:
                break
            self.request.send(msg)

if __name__ == '__main__':
    serv = TCPServer(('', 20000), EchoHandler)
    serv.serve_forever()
像这样我们就启动了一个TCP服务器， 那么我们怎么跟它进行通信呢？

实现一个TCP客户端
>>> from socket import socket, AF_INET, SOCK_STREAM
>>> s = socket(AF_INET, SOCK_STREAM)
>>> s.connect(('localhost', 20000))
>>> s.send(b'Hello')
5
>>> s.recv(8192)
b'Hello'
>>>
看了这两段代码你是不是想吐了？而且这还没完，这仅仅只是个单线程的程序，如果要正式使用，server端肯定不能是单线程，一定要考虑并发，然后什么多线程、协程什么的在搞一搞，然后项目PM在催催项目进度，我相信你就真吐了!

如果在平时的工作中，还这么写代码的话，我相信你一定忍不住要自己编写一个标准，写一套框架，干掉这些繁杂的工作，让工作变得轻松，自己变得幸福。是的，没错，有很多人都有这样的想法，所以也就诞生了WSGI（Web Server Gateway Interface) 有了WSGI标准之后，在此标准之上诞生了很多框架，减少了很多繁杂的工作，程序员们的工作效率也一日千里， 这个一定程度促进了互联网行业的飞速发展。当然，也有人不开心的，最显著的就是很多同学抱怨，自己变成了CRUD boy.

好了，我们开始来看看Flask的实现吧。在Hello World里面，我们看到，首先引入了Flask这个类，然后实例化这个类，生成一个app对象。 我们来翻翻代码，看看Flask这个类都干了啥？

Flask源码分析
class Flask(_PackageBoundObject):
    """The flask object implements a WSGI application and acts as the central
    object.  It is passed the name of the module or package of the
    application.  Once it is created it will act as a central registry for
    the view functions, the URL rules, template configuration and much more.

    The name of the package is used to resolve resources from inside the
    package or the folder the module is contained in depending on if the
    package parameter resolves to an actual python package (a folder with
    an :file:`__init__.py` file inside) or a standard module (just a ``.py`` file).

    For more information about resource loading, see :func:`open_resource`.

    Usually you create a :class:`Flask` instance in your main module or
    in the :file:`__init__.py` file of your package like this::

        from flask import Flask
        app = Flask(__name__)

    .. admonition:: About the First Parameter

        The idea of the first parameter is to give Flask an idea of what
        belongs to your application.  This name is used to find resources
        on the filesystem, can be used by extensions to improve debugging
        information and a lot more.

        So it's important what you provide there.  If you are using a single
        module, `__name__` is always the correct value.  If you however are
        using a package, it's usually recommended to hardcode the name of
        your package there.

        For example if your application is defined in :file:`yourapplication/app.py`
        you should create it with one of the two versions below::

            app = Flask('yourapplication')
            app = Flask(__name__.split('.')[0])

        Why is that?  The application will work even with `__name__`, thanks
        to how resources are looked up.  However it will make debugging more
        painful.  Certain extensions can make assumptions based on the
        import name of your application.  For example the Flask-SQLAlchemy
        extension will look for the code in your application that triggered
        an SQL query in debug mode.  If the import name is not properly set
        up, that debugging information is lost.  (For example it would only
        pick up SQL queries in `yourapplication.app` and not
        `yourapplication.views.frontend`)
    """
    def __init__(self):
        pass

  def run(self, host=None, port=None, debug=None, **options):
    """Runs the application on a local development server.

    Do not use ``run()`` in a production setting. It is not intended to
    meet security and performance requirements for a production server.
    Instead, see :ref:`deployment` for WSGI server recommendations.

    If the :attr:`debug` flag is set the server will automatically reload
    for code changes and show a debugger in case an exception happened.

    If you want to run the application in debug mode, but disable the
    code execution on the interactive debugger, you can pass
    ``use_evalex=False`` as parameter.  This will keep the debugger's
    traceback screen active, but disable code execution.

    It is not recommended to use this function for development with
    automatic reloading as this is badly supported.  Instead you should
    be using the :command:`flask` command line script's ``run`` support.

    .. admonition:: Keep in Mind

       Flask will suppress any server error with a generic error page
       unless it is in debug mode.  As such to enable just the
       interactive debugger without the code reloading, you have to
       invoke :meth:`run` with ``debug=True`` and ``use_reloader=False``.
       Setting ``use_debugger`` to ``True`` without being in debug mode
       won't catch any exceptions because there won't be any to
       catch.
    """
    pass
这是从Flask源码里面拿出来的，一个是Flask这个类的注解，一个是他的run方法的注解。 我们可以看到，它也是采用了WSGI的标准，在WSGI标准中，需要实现一个application函数，这个函数可以接受响应，并处理请求。所以，在这里我们实例化的app就是这个application. 因为在源码中，我们清楚的可以看到，在run函数中有显示如下：

from werkzeug.serving import run_simple
if host is None:
    host = '127.0.0.1'
if port is None:
    server_name = self.config['SERVER_NAME']
    if server_name and ':' in server_name:
        port = int(server_name.rsplit(':', 1)[1])
    else:
        port = 5000
if debug is not None:
    self.debug = bool(debug)
options.setdefault('use_reloader', self.debug)
options.setdefault('use_debugger', self.debug)
try:
    run_simple(host, port, self, **options)
finally:
    # reset the first request information if the development server
    # reset normally.  This makes it possible to restart the server
    # without reloader and that stuff from an interactive shell.
    self._got_first_request = False
是的，我们看到在run_simple 函数当中，application参数传的是self. ok，这样就很明确了，至于后面在run_simple 函数中，提供了多种启动服务器的方式，有多线程、多进程等等。 这些都是为了提高服务器的并发性能。到这里，我们已经看完了flask框架最核心的部分，也就是它如何启动一个服务器，然后接受请求、处理请求。 当然，在整个过程中，还有很多处理，很多中间件，做了很多事情。这里我们不展开讲，有兴趣的同学可以读读源码。

所以写一个框架也没那么难对不对？ 当然这里也只是说说，一个成熟的框架包含方方面面的内容，相对来说，根据业务场景定制化的框架实现起来要相对简单，像这种开源的，面向很多场景的框架来说，实现难度也蛮大，毕竟有很多需要考虑的因素， 不是仅仅实现某几个特定的功能就ok的。

实现基于Flask的API层
了解完这些之后CRUD就变的非常简单了，下面我们就直接贴代码：

# server.py
import json
from flask import Flask

app = Flask(__name__)

from core.blockchain.block import Block
from core.blockchain.blockchain import BlockChain


@app.route('/')
def hello():
    return "Hello World!"


@app.route('/add/block')
def add_block():
    b = Block()
    b.new_block("Magic", "")
    block = b.block
    return json.dumps(block)


@app.route('/blockchain')
def blockchain():
    bc = BlockChain()

    blocks = bc.print_blockchain()

    blocks = sorted(blocks, key=lambda s: s['Data'])

    return json.dumps(blocks)


if __name__ == "__main__":
    app.run(port=8888, debug=True)
在这里，仅仅有三个接口，一个是hello world，做测试，一个是添加block，一个是查看整条链。

我们来测试一下吧, 本来打算连同前端代码也写完的，但实在太懒， 所以这里提供一个我之前写的前后端的一个项目， 网站跟源码地址我贴出来，有兴趣的同学可以看看。网站 源码：https://github.com/csunny/design

首先我们py-bitcoin 的源码里面源码地址在net下面找到server.py文件，然后运行，启动server.

查看响应结果
# 得到响应结果
[
    {
        "TimeStamp": 1531293208,
        "Data": "Magic Test 0",
        "PrevBlockHash": "",
        "Hash": "000021aa3af3d22f9a56e2bb7c34883724132bff1e2503588674d10c6fa17048",
        "Nonce": 17868
    },
    {
        "TimeStamp": 1531293208,
        "Data": "Magic Test 1",
        "PrevBlockHash": "000021aa3af3d22f9a56e2bb7c34883724132bff1e2503588674d10c6fa17048",
        "Hash": "0000d1c8dc0077505f1d80fa6c5e6b168cef3f515c56a3ca5c2c4f58476b9224",
        "Nonce": 58177
    },
    {
        "TimeStamp": 1531293209,
        "Data": "Magic Test 2",
        "PrevBlockHash": "0000d1c8dc0077505f1d80fa6c5e6b168cef3f515c56a3ca5c2c4f58476b9224",
        "Hash": "0000d1fa16fc4918e6a1feb5c8e5862914cce633cc626f33f2aa03a2cf573a9b",
        "Nonce": 33954
    },
    {
        "TimeStamp": 1531293209,
        "Data": "Magic Test 3",
        "PrevBlockHash": "0000d1fa16fc4918e6a1feb5c8e5862914cce633cc626f33f2aa03a2cf573a9b",
        "Hash": "000028b99fa62cbdddd9d4d4fddfd102bc50246643a05bc520d7e366bac11cdf",
        "Nonce": 52866
    },
    {
        "TimeStamp": 1531293209,
        "Data": "Magic Test 4",
        "PrevBlockHash": "000028b99fa62cbdddd9d4d4fddfd102bc50246643a05bc520d7e366bac11cdf",
        "Hash": "00005f4df0649cd1e89d2d2f3598ff2c6f8d9327c96a5573f5ab991cdf491d33",
        "Nonce": 2946
    },
    {
        "TimeStamp": 1531293209,
        "Data": "Magic Test 5",
        "PrevBlockHash": "00005f4df0649cd1e89d2d2f3598ff2c6f8d9327c96a5573f5ab991cdf491d33",
        "Hash": "00008af22d7933f498412b7373aff39d9cd159783b143bf1a3857cfcde9b50c5",
        "Nonce": 49161
    },
    {
        "TimeStamp": 1531293209,
        "Data": "Magic Test 6",
        "PrevBlockHash": "00008af22d7933f498412b7373aff39d9cd159783b143bf1a3857cfcde9b50c5",
        "Hash": "000070dafb1a9e78d9caa3e297270194a755309276198d20235cb8d442fabf34",
        "Nonce": 12467
    },
    {
        "TimeStamp": 1531293209,
        "Data": "Magic Test 7",
        "PrevBlockHash": "000070dafb1a9e78d9caa3e297270194a755309276198d20235cb8d442fabf34",
        "Hash": "000055faed15280e764e72e9300783173ffaea95f694d44478c3df054f6d43e5",
        "Nonce": 57270
    },
    {
        "TimeStamp": 1531293210,
        "Data": "Magic Test 8",
        "PrevBlockHash": "000055faed15280e764e72e9300783173ffaea95f694d44478c3df054f6d43e5",
        "Hash": "0000fc3657fee5f282b240781e0a04bc7bb80a66953f78b449dfa4c6f07b6035",
        "Nonce": 139445
    },
    {
        "TimeStamp": 1531293211,
        "Data": "Magic Test 9",
        "PrevBlockHash": "0000fc3657fee5f282b240781e0a04bc7bb80a66953f78b449dfa4c6f07b6035",
        "Hash": "00001db890be8c428f35c445fb965175c4c7895b4bd83ffc67580429ef9dc51a",
        "Nonce": 136057
    }
]
今天的知识比较传统，并没有太多涉及到区块链的知识，所以相对来说，应该好理解。
