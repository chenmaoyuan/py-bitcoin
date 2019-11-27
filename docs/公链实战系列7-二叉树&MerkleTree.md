公链实战系列7-二叉树&MerkleTree

二叉树
二叉搜索树
代码实战
python实现
go语言实现
测试代码
Merkle Tree介绍
特点
应用
MerkleTree实现
python 实现
测试代码
go语言实现
项目地址
参考
二叉树
在我们实现MerkleTree之前，我们首先来熟悉一下数据结构与算法里面的二叉树。这里我们主要以BinarySearchTree为例子。另外，为了突出数据结构与算法的重要性，以及语言的无关性。我这里提供两种语言的实现，python、go

二叉搜索树
二叉查找树（Binary Search Tree)， 又叫二叉搜索树、有序二叉树(ordered binary tree) 或排序二叉树(sorted binary tree), 是指一颗空树或者具有下列性质的二叉树

若任意节点的左子树不空， 则左子树上所有节点的值均小于它的根节点的值。
若任意节点的右子树不空，则右边子树上所有节点的值均大与它的根节点的值。
任意节点的左、右子树也分别为二叉查找树。
没有键、值相等的节点。
代码实战
下图为一个二叉搜索树的表示，下面我们来以此树为例，编写代码


python实现
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
This Document is Created by  At 2018/7/14 
"""


class Node:
    def __init__(self, key, value, left=None, right=None):
        self.key = key
        self.value = value
        self.left = left
        self.right = right


class BinarySearchTree:
    """
        实现一个二叉搜索树。
    """

    def __init__(self, root=None):
        self.root = root

    def insert(self, key, value):
        """
        插入元素
        :param key: 
        :param value: 
        :return: 
        """
        n = Node(key, value)

        if not self.root:
            self.root = n
        else:
            self._insert_node(self.root, n)

    def _insert_node(self, node, new_node):
        if new_node.key < node.key:
            if not node.left:
                node.left = new_node
            else:
                self._insert_node(node.left, new_node)
        else:
            if not node.right:
                node.right = new_node
            else:
                self._insert_node(node.right, new_node)

    def search(self, key):
        """
        查找元素
        :param key: 
        :return: 
        """
        return self._search(self.root, key)

    def _search(self, node, key):
        if not node:
            return False
        if key < node.key:
            return self._search(node.left, key)

        if key > node.key:
            return self._search(node.right, key)

        return True

    def string(self):
        print("=======================================")
        self._stringify(self.root, 0)
        print("=======================================")

    def _stringify(self, node, level):
        if node:
            format = ""
            for i in range(level):
                format += "                     "

            format += "----["
            level += 1
            self._stringify(node.left, level)

            print(format + "%s\n" % node.key)
            self._stringify(node.right, level)


if __name__ == '__main__':
    bst = BinarySearchTree()

    bst.insert(8, "8")
    bst.insert(3, "3")
    bst.insert(1, "1")
    bst.insert(10, "10")
    bst.insert(6, "6")
    bst.insert(4, "4")
    bst.insert(7, "7")
    bst.insert(13, "13")
    bst.insert(14, "14")

    bst.string()
go语言实现
# tree.go

type Item interface {}


// Node s single node that compose the tree
type Node struct {
   Key int
   Value Item
   left *Node
   right *Node
}
// ItemBinarySearchTree the binary search tree of the Items
type ItemBinarySearchTree struct {
   Root *Node
   Lock *sync.Mutex
}

// Insert inserts the item t in the tree
func (bst *ItemBinarySearchTree) Insert(key int, value Item){
   //bst.Lock.Lock()
   //defer bst.Lock.Unlock()
   n := &Node{key, value, nil, nil}
   if bst.Root == nil{
      bst.Root = n
   }else {
      insertNode(bst.Root, n)
   }
}

func insertNode(node, newNode *Node)  {
   if newNode.Key < node.Key{
      // 左边插入
      if node.left == nil{
         node.left = newNode
      }else {
         insertNode(node.left, newNode)
      }
   }else {
      if node.right == nil{
         node.right = newNode
      }else {
         insertNode(node.right, newNode)
      }
   }
}


func (bst *ItemBinarySearchTree) String()  {
   //bst.Lock.Lock()
   //defer bst.Lock.Unlock()

   fmt.Println("---------------------------------------------")
   stringify(bst.Root, 0)
   fmt.Println("---------------------------------------------")

}

func stringify(n *Node, level int)  {
   if n != nil{
      format := ""
      for i := 0; i< level; i++{
         format += "         "
      }
      format += "---["
      level ++
      stringify(n.left, level)
      fmt.Printf(format+"%d\n", n.Key)
      stringify(n.right, level)
   }
}
利用上面的代码，我们就可以生成一颗二叉搜索树，在这里我们采用的是递归实现，根据二叉树的性质。

测试代码
package tree

import (
   "testing"
   "fmt"
)

var bst ItemBinarySearchTree

func fillTree(bst *ItemBinarySearchTree)  {
   bst.Insert(8, "8")
   bst.Insert(3, "3")
   bst.Insert(1, "1")
   bst.Insert(10, "10")
   bst.Insert(6, "6")
   bst.Insert(4, "4")
   bst.Insert(7, "7")
   bst.Insert(13, "13")
   bst.Insert(14, "14")
}


func TestItemBinarySearchTree_InOrderTraverse(t *testing.T) {
   fillTree(&bst)
   bst.String()

   var result []string
   bst.InOrderTraverse(func(i Item) {
      result = append(result, fmt.Sprintf("%s", i))
   })

   t.Error(result)
}
# 运行go test 查看


Merkle Tree介绍
在这里，我们只是实现了二叉搜索树的插入和搜索， 对于二叉搜索树，还有很多其他的操作，比如遍历、删除、求最大元素、最小元素等等。另外二叉树拓展开来还有很多类型，这里我们就不展开了，既然已经了解了二叉树，那么下面我们就趁热打铁，来看看MerkleTree，其实MerkleTree也是一种二叉树结构。

MerkleTree是一种典型的二叉树结构，由一个根节点、一组中间节点和一组叶子节点组成。MerkleTree最早由Merkle Ralf在1980年提出，层广泛应用于文件系统和P2P系统中。

特点
最下面的叶子节点包含存储数据或hash值。
非叶子节点（包括中间节点和根节点）都是它的两个孩子节点内容的hash值。
Merkle树逐层记录hash值的特点，让它具有了一些独特的性质。例如，底层数据的任何变动，都会传递到其父节点，一层层沿着路径一直到树根。这意味着树根的值实际上代表了对底层所有数据的“数字摘要”。

应用
快速比较大量数据
快速定位修改
零和知识证明
接下来，我们看看Merkle树的代码实现， 这里在强调下，本系列注重代码的实现，关注实战，以talk is cheap， show me the code 为宗旨！

MerkleTree实现
python 实现
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
This Document is Created by  At 2018/6/30 
This module is merkletree implement
"""

import hashlib


class MerkleTree:
    """
    - doc  Merkle tree
    """

    def __init__(self, data):
        if not isinstance(data, list):
            raise TypeError("data must be a bytes list")

        self.data = data
        self.nodes = []
        self.root = None

    def new_tree(self):

        if len(self.data) % 2 != 0:
            self.data.append(self.data[-1])

        for d in self.data:

            if isinstance(d, bytes) or isinstance(d, bytearray):

                node = MerkleNode(None, None, d)
                node.new_node()
                self.nodes.append(node)
            else:
                raise TypeError("数据类型错误！")

        for i in range(int(len(self.data)/2)):
            new_level = []

            for j in range(0, len(self.nodes), 2):
                node = MerkleNode(self.nodes[j], self.nodes[j + 1], b'')
                node.new_node()
                new_level.append(node)
            self.nodes = new_level

        self.root = self.nodes[0]


class MerkleNode:
    """
    - doc
    """

    def __init__(self, left, right, data):
        self.Left = left
        self.Right = right

        if isinstance(data, bytes) or isinstance(data, bytearray):
            self.data = data
        else:
            raise TypeError("data 不是byte类型!")

    def new_node(self):
        if not self.Left and not self.Right:
            hash_value = hashlib.sha256(bytes(self.data)).hexdigest()
            self.data = hash_value
        else:
            prehash = self.Left.data + self.Right.data
            hash_value = hashlib.sha256(prehash.encode()).hexdigest()
            self.data = hash_value
测试代码
#!/usr/bin/env python
# -*- coding:utf-8 -*-

"""
This Document is Created by  At 2018/7/6 
"""
from utils.merkletree import MerkleNode, MerkleTree

if __name__ == "__main__":
    m1 = MerkleNode(None, None, b"Node1")
    m1.new_node()

    m2 = MerkleNode(None, None, b"Node2")
    m2.new_node()

    m3 = MerkleNode(None, None, b"Node3")
    m3.new_node()

    m4 = MerkleNode(None, None, b"Node3")
    m4.new_node()

    m5 = MerkleNode(m1, m2, b'')
    m5.new_node()
    print(m5.Left, m5.Right, m5.data)

    tree = MerkleTree([b'Node1', b'Node2', b'Node3'])
    tree.new_tree()

    print(tree.root.data)
go语言实现
import (
   "crypto/sha256"
)

// MerkleTree represent a Merkle tree
type MerkleTree struct {
   RootNode *MerkleNode
}

// MerkleNode represent a Merkle tree node
type MerkleNode struct {
   Left  *MerkleNode
   Right *MerkleNode
   Data  []byte
}

// NewMerkleTree creates a new Merkle tree from a sequence of data
func NewMerkleTree(data [][]byte) *MerkleTree {
   var nodes []MerkleNode

   if len(data)%2 != 0 {
      data = append(data, data[len(data)-1])
   }

   for _, datum := range data {
      node := NewMerkleNode(nil, nil, datum)
      nodes = append(nodes, *node)
   }

   for i := 0; i < len(data)/2; i++ {
      var newLevel []MerkleNode

      for j := 0; j < len(nodes); j += 2 {
         node := NewMerkleNode(&nodes[j], &nodes[j+1], nil)
         newLevel = append(newLevel, *node)
      }

      nodes = newLevel
   }

   mTree := MerkleTree{&nodes[0]}

   return &mTree
}

// NewMerkleNode creates a new Merkle tree node
func NewMerkleNode(left, right *MerkleNode, data []byte) *MerkleNode {
   mNode := MerkleNode{}

   if left == nil && right == nil {
      hash := sha256.Sum256(data)
      mNode.Data = hash[:]
   } else {
      prevHashes := append(left.Data, right.Data...)
      hash := sha256.Sum256(prevHashes)
      mNode.Data = hash[:]
   }

   mNode.Left = left
   mNode.Right = right

   return &mNode
}


