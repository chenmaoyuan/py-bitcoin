公链实战系列9-用Transactions、MerkleTree、Wallet以及POW生成完整区块

理论介绍
代码实战
Python实现
Block类实现
POW实现
生产创世区块
交易实现
测试代码
项目地址：
参考：
理论介绍
先前我们有讲，在一个区块中至少要包含以下信息：

Timestamp
Hash
PrevHash
Transactions
今天我们回头来看，其实之前的实现并不完整， 前面我们也有讲到，我们为了让区块看着简单易懂，我们将交易信息简单的用一个随机的字符串data来替代了。 现在，当我们在前面的章节中系统的了解了Base58、Hash、MerkleTree、Transaction、POW、Wallet 等知识点之后，我们就可以生成一个完整的区块了，这个区块用到了以上提到的所有的内容， 并且在我们的代码当中会产生如下的输出

{

    "TimeStamp":1531800034,

    "Transactions":[

        {

            "ID":"aaf1cf4ab2675df2df75ce9828f719568aadaec61de1b8cb1217b92ce9e19e2e",

            "Vin":[

                {

                    "txid":"",

                    "vout":-1,

                    "signature":null,

                    "pubkey":"Base Info, Magic"

                }

            ],

            "Vout":[

                {

                    "value":10,

                    "pub_key_hash":"7AgP8z7XYyZ2sdnVJ6HCiE5X2reJDf"

                }

            ]

        },

        {

            "ID":"aaf1cf4ab2675df2df75ce9828f719568aadaec61de1b8cb1217b92ce9e19e2e",

            "Vin":[

                {

                    "txid":"",

                    "vout":-1,

                    "signature":null,

                    "pubkey":"Base Info, Magic"

                }

            ],

            "Vout":[

                {

                    "value":10,

                    "pub_key_hash":"7AgP8z7XYyZ2sdnVJ6HCiE5X2reJDf"

                }

            ]

        }

    ],

    "PrevBlockHash":"",

    "Nonce":60137,

    "Height":0,

    "Hash":"00004c593701710e45c81238fd9f034f14f1fe523f5760f2e085b4a384e53d66"

}
我们看到在这个区块信息当中，除了显示了完整的交易信息之外，还有两个我们之前没有提到的字段

Nonce # 随机数
Height # 区块高度， 因为我们这个是创世区块，所以区块高度为0
那么，现在我们就知道了，在一个完整的区块当中，其实包含了一下信息

Timestamp
PrevHash
Transactions
Nonce
Height
代码实战
认识了一个完整的区块之后，现在让我们来看看，在代码里面是如何将前面提到的这些知识点串联起来，并且生成一个完整的区块的。

Python实现
Block类实现
首先我们来看看Block类

# block.py
import time
import hashlib
from consensus.proof_of_work import ProofOfWork

class Block:
    """

    Todo add index for block.

    doc  区块

    block = {
        "Timestamp": "",
        "Data": "",
        "PrevBlockHash": "",
        "Hash": "",
        "Nonce": ""
    }
    """

    # todo update data to transaction information,  and then the transaction look like
    '''
        block = {
            "Timestamp": datetime.now(),
            "Transactions": [
                {
                    "ID": "",    # 交易ID
                    "Vin": "",    # 交易输入
                    "Vout": ""   # 交易输出
                    },
                {
                    "ID": "",
                    "Vin"" "",
                    "Vout"" "",
                }
            ],
            "PrevBlockHash": "",
            "Hash": "",
            "Nonce": "",
            "Height": ""
        }
    '''

    def __init__(self):
        self.block = dict()

    def new_block(self, transactions, prev_hash, height):
        block = {
            "TimeStamp": int(time.time()),
            "Transactions": transactions,
            "PrevBlockHash": prev_hash,
            "Nonce": 0,
            "Height": height
        }

        pow = ProofOfWork(block)

        b_hash, nonce = pow.run()
        block["Hash"] = b_hash
        block["Nonce"] = nonce

        self.block = block
        return block
可以看到，这里的new_block 方法发生了变化，我们加入了Transactions、Nonce和Height

我们知道，POW是根据前一个区块来生成后一个区块的，所以当区块信息发生变化时，POW也需要做相应的更改，不过，我们这里只需要更改prepare_data 这一个方法，并添加一个新的hash Transactions的方法就ok了， 详细看代码实现

POW实现
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
This Document is Created by  At 2018/6/29 
"""
import hashlib
from settings import maxNonce, targetBits
from utils.merkletree import MerkleTree


class ProofOfWork:
    """
    工作量证明算法
    """

    def __init__(self, block, nonce=0):
        self.nonce = nonce

        if not isinstance(block, dict):
            raise TypeError

        self.block = block
        self.target = 1 << (256-targetBits)

    def hash_txs(self):
        """
        将block的transactions 信息hash加密, 并且用merkle tree来存储交易信息。
        :return:
        """

        tree = MerkleTree(self.block["Transactions"])
        tree.new_tree()

        return tree.root.data

    def prepare_data(self):
        """
        准备计算的数据
        :return:
        """

        timestamp = hex(self.block["TimeStamp"])[0]

        data = "".join([
            self.block["PrevBlockHash"],
            self.hash_txs(),
            timestamp,
            hex(targetBits),
            hex(self.nonce)
        ])

        return data

    def run(self):
        print("Mining  a new block...")

        hash_v = ""
        while self.nonce < maxNonce:
            data = self.prepare_data()

            hash_v = hashlib.sha256(data.encode("utf-8")).hexdigest()

            # print("-----> is mining ... %s" % hash_v)

            if int(hash_v, 16) <= self.target:
                break
            else:
                self.nonce += 1
        print("\n")

        return hash_v, self.nonce

    def validate(self):
        data = self.prepare_data()
        hash_v = hashlib.sha256(data.encode('utf-8')).hexdigest()

        print(int(hash_v, 16), self.target)

        if int(hash_v, 16) <= self.target:
            return True
        return False
这里有个需要关注的地方，那就是在我们的ProofOfWork类当中，我们新加入的hash_txs方法，我们看到，我们加密交易数据的时候，是用的MerkleTree，不知道关于MerkleTree的实现大家还记不记得（不记得的可以翻翻前面的教程，然后回来看这部分实现）可以看到我们交易hash，其实只返回了MerkleTree根节点的hash值，然后根据此hash来计算下一个区块。看到这里，我相信你已经迫不及待想看看，我们到底如何生成一个区块了，好了，我们直接上代码，需要注意的是，这里我们需要用到之前讲到的Transaction的代码，所以如果对前面的知识没有印象，可以翻开前面的教程一起来看。

生产创世区块
# block.py

def new_genesis_block(coinbase):
    """
    根据coinbase交易创建一个创世区块
    :param coinbase:
    :return:
    """
    b = Block()

    transaction = Transaction(coinbase.ID, coinbase.Vin, coinbase.Vout)

    tx = transaction.serialize()

    genesis_block = b.new_block([str(tx).encode()], "", 0)
    return genesis_block
交易实现
# transaction.py

def new_coinbase_tx(to, data="Base Info, Magic"):
    txin = Input("", -1, None, data)
    txout = Output(subsidy, to)

    tx = Transaction(None, [txin], [txout])
    tx.hash()

    return tx
测试代码
# block_test.py

from core.blockchain.block import new_genesis_block
from core.transactions.transaction import new_coinbase_tx


# def test_v0_1_0_version():
#     b = Block()
#
#     b.new_block("Magic", "")
#
#     block = b.block
#
#     new_block = b.new_block("This is a test block", b.block["Hash"])
#
#     assert new_block["PrevBlockHash"] == block["Hash"]


def test_genesis_block():
    coinbase = new_coinbase_tx("7AgP8z7XYyZ2sdnVJ6HCiE5X2reJDf")

    genesis_block = new_genesis_block(coinbase)
    print(genesis_block)

    # we will get the genesis block like this, but there many work need to do

if __name__ == '__main__':
    test_genesis_block()
到这里，你当你运行测试程序的时候，就可以输出一个在开篇我们看到的block了， 为了能让大家更好的找到代码，在本章之前的代码我刻意打了tagv0.1.0 ， 所以在学习前面的教程的时候可以切到tagv0.1.0 去查看，本节以及之后的内容，目前放到了新分支utxo上面。源码

注意： 如果有对git版本管理不熟悉的同学，可以自己看看教程，我这里推荐一个教程 git

本章内容理论知识并不多，只是将之前讲到的知识点串联起来了。重点还是代码的实现部分，所以建议大家多看看代码，多理解，相信会有收获。

