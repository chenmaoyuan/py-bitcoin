公链实战系列8-Transaction（交易）

前言
理论介绍
代码实战
代码解读
代码测试
项目地址
参考
前言
在介绍比特币交易之前，这里有两个视频推荐下，为了能够更好的理解比特币交易，建议大家先观看推荐的视频。 如果你已经理解了交易的理论，那么可以直接跳过，看下面的代码部分。

经济机器是怎么运作的？

中文
英文
理论介绍
在比特币中，一个交易包含有一系列的输入、输出。也就是说，输入、输出构成了比特币交易。 如下图所示

输出可以不指向另一个输入
一个输入往往对应对个输出
一个输入必须来自一个输出


在单个交易中，通过公、私钥对交易进行加、解密。 我们来根据比特币白皮书中的图片对单个交易信息进行解读，看下图


从图中我们可以看出A用户通过自己的私钥对交易信息进行加密，也就是图片中sign的过程，经过加密的信息作为输入发送给B用户，B用户接受到交易输入后，用A用户的公钥对交易进行验证，也就是verify的过程。 不知道大家还记不记得之前我们在讲椭圆加密算法的时候提到的两种用法，这里采用的恰恰就是第二种，验证交易输入的有效性。

理解了前面讲的这两个知识点

交易包含输入、输出
交易信息是通过公、私钥在交易地址之间进行加、解密
接下来，我们就可以直接上代码了。

代码实战
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
This Document is Created by  At 2018/7/9

1. 输出可以不指向另一个输入
2. 一次交易中，一个输入往往对应与多个输出
3. 一个输入必须来自一个输出

If the data is correct, the output can be unlocked, and its value can be used to generate new outputs;
 if it’s not correct, the output cannot be referenced in the input.
  This is the mechanism that guarantees that users cannot spend coins belonging to other people.

"""
import hashlib
import json
from core.transactions.input import Input
from core.transactions.output import Output
from fastecdsa import ecdsa

subsidy = 10   # 交易额


class Transaction:
    """
    doc  交易

    # todo
    """

    def __init__(self, ID, vin, vout):
        """
        ID 交易的hash值
        :param ID:
        :param vin:  vin 所有的交易输入
        :param vout:  vout 所有的交易输出
        """
        self.ID = ID
        self.Vin = vin
        self.Vout = vout

    def is_coinbase(self):
        """
        只有一个输入，并且输入没有任何输出。
        :return:
        """
        return len(self.Vin) == 0 and len(self.Vin[0].txid) == 0 and self.Vin[0].vout == -1

    def serialize(self):

        vins = []
        for vin in self.Vin:
            k = dict()
            k['txid'] = vin.txid
            k['vout'] = vin.vout
            k['signature'] = vin.signature
            k['pubkey'] = vin.pubkey

            vins.append(k)

        vouts = []
        for vout in self.Vout:
            k = dict()
            k['value'] = vout.value
            k['pub_key_hash'] = vout.pub_key_hash
            vouts.append(k)

        data = {
            "ID": self.ID,
            "Vin": vins,
            "Vout": vouts
        }

        return json.dumps(data)

    def hash(self):

        hv = hashlib.sha256(self.serialize().encode('utf-8')).hexdigest()

        self.ID = hv
        return hv

    def sign(self, priv_key, prev_txs):
        """
        sign each input of transaction

        :param priv_key: 私钥
        :param prev_txs: 之前的所有交易
        :return:
        """
        if self.is_coinbase():
            return
        for vin in self.Vin:
            if not prev_txs[vin.txid].ID:
                raise ValueError(
                    "Error: previous transaction is not correct."
                )

        tx_copy = self.trimmed_copy()
        for inId, vin in enumerate(tx_copy.Vin):
            prev_tx = prev_txs[vin.txid]
            tx_copy.Vin[inId].signature = ""
            tx_copy.Vin[inId].pubkey = prev_tx.Vout[vin.vout].pub_key_hash

            r, s = ecdsa.sign(tx_copy, priv_key)
            signature = "".join([str(r), str(s)])
            self.Vin[inId].signature = signature
            tx_copy.Vin[inId].pubkey = ""

    def string(self):
        """
        打印交易的输入和输出
        :return:
        """
        pass

    def trimmed_copy(self):
        inputs = []
        outputs = []

        for vin in self.Vin:
            inpt = Input(vin.txid, vin.vout, "", "")
            inputs.append(inpt)

        for vout in self.Vout:
            opt = Output(vout.value, vout.pub_key_hash)
            outputs.append(opt)

        tx_copy = Transaction(self.ID, inputs, outputs)
        return tx_copy

    def verify(self, prev_txs):
        """
        校验先前的所有输入是否合法
        :param prev_txs:
        :return:
        """
        if self.is_coinbase():
            return True

        for vin in self.Vin:
            if not prev_txs[vin.txid].ID:
                raise ValueError(
                    "Error: previous transaction is not correct."
                )

        tx_copy = self.trimmed_copy()
        for inId, vin in enumerate(tx_copy.Vin):
            prev_tx = prev_txs[vin.txid]
            tx_copy.Vin[inId].signature = ""
            tx_copy.Vin[inId].pubkey = prev_tx.Vout[vin.vout].pub_key_hash

            sign_len = len(vin.signature)
            # x = vin.pubkey[:(sign_len/2)]
            # y = vin.pubkey[(sign_len/2):]
            #
            # raw_pub_key = ecdsa.verify()
            pass

        return True


def new_coinbase_tx(to, data="Base Info, Magic"):
    txin = Input("", -1, None, data)
    txout = Output(subsidy, to)

    tx = Transaction(None, [txin], [txout])
    tx.hash()

    return tx
代码解读
在代码中，我们实现了以下几个方法，is_coinbase、sign、verify、hash、serialise、trimmed_copy, 在前面的讲解中，我们了解到，在一次交易当中sign根verify是两个非常重要的过程，它是验证交易有效性的必要条件。 最下面有个单独的方法，生成coinbase交易的方法。也就是生成第一笔交易。创世区块的产生就根coinbase交易有关系，至于创世区块与coinbase以及交易验证的内容，我们放在下一节中，下面我们看看基本的测试代码

代码测试
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
This Document is Created by  At 2018/7/10 
"""

from core.wallet.wallet import Wallet
from core.transactions.transaction import Transaction, new_coinbase_tx


# 首先生成n个备用的地址
def get_address(n):
    wallet = Wallet()

    return [wallet.get_address() for i in range(n)]


# 测试生成coinbase交易
def build_coinbase_tx(to, data=None):
    tx = new_coinbase_tx(to, data) if data else new_coinbase_tx(to)
    return tx


def new_coinbase_test():
    addresses = get_address(10)
    tx = build_coinbase_tx(addresses[0])
    print(tx)


def main():
    new_coinbase_test()


if __name__ == '__main__':
    main()
由于交易的过程中涉及到交易地址，所以这一节中我们前面实现的wallet就配上用场了，详细的过程大家可以看看测试代码，这里就不赘述了，有问题的可以给我留言。

