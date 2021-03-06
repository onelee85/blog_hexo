title: 区块链解谜
author: James
tags:
  - 区块链
categories:
  - 区块链
date: 2017-12-29 11:58:00
---

# 什么是区块链

区块链是一个分布式的，去中心化的，点对点的，不可篡改的，共享账本系统。

<!-- more -->

![jvm_struct2](/images/blockchain/blockchain.png)

#  区块链架构

区块链整体架构上划分为网络层、共识层、数据层、智能合约层和应用层。共识层和数据层之间的联系比较密切，需要“通力合作”完成数据的存取。

- 网络层，几乎都是基于P2P的网络，它主要完成节点之间的交互和数据块的传输。
- 共识层，提供在不可信、分布式环境中的共识协议，通过它保证节点之间的数据一致性。
- 数据层，提供数据保存的数据结构，它要承担防篡改的任务，所以技术上用了区块链表、Hash、Merkle Tree来实现。
- 智能合约层，相当于数据访问接口，对区块链上的数据读写都是通过智能合约进行的，任何人的数据读写动作都是被明确定义的。
- 应用层，各种各样的应用程序。


![jvm_struct2](/images/blockchain/arch.png)


# 区块链和比特币的关系

每个区块数据由谁来记录或者说打包，是可以设置一个游戏规则的，比如说掷骰子，大家约定谁能连续3次掷出6，那就让他来记这个数据，为了补偿 一下他的劳动投入，奖励给他一些收益， 
比特币正是使用了这样的原理来不断的发行新的比特币出来，奖励给打包的那个人的比特币就是新发行的比特币。

# 挖矿和区块链

作为一个“数据库应用”比特币必须要“做点什么”。于是它就设计了一个小游戏环节——挖矿。游戏设计师中本聪在他大作《比特币：一种点对点式的电子现金系统》详细讲述的设计内幕和游戏方法。

# 区块链的技术组合

## 公开密钥算法

属于计算机密码学里传统的技术，公开密钥算法是一种不对称的加密算法，拥有两个密钥，可以互相加解密，通常其中的一个密钥是公开的称之为公钥，另外一个密钥是保密的称之为私钥

## 哈希算法

也是属于计算机密码学中传统的技术，应用就更广泛了，主要用来对一段数据进行计算，得出一个摘要信息，通俗点说就是给一段数据生成1个身份证号；不同的消息生成的摘要数据是不一样的（某些抗碰撞能力弱的哈希算法可能在这里面会有些问题，但是使用广泛的一些知名的哈希算法，发生碰撞的概率很低），相当于给⼀段数据生成了一个身份证号这么个意思，在区块链系统中，哈希算法的使用很多，例如如区块与区块之间，就是通过区块头的哈希关联起来的，在区块中的每一笔交易事务也都会生成一个哈希值作为交易易数据的ID，通过这些身份证号可以方便的检索或者关联区块，也能方便的指定某一笔交易事务。 

## 网络共识算法

在很久前就有计算机科学家研究过，并且提出过一些模型，如拜占庭容错算法之类，比特币、以太坊这些使用的是一种工作量证明算法，其它的一些区块链系统有使用其他各种衍生的算法，算法的原理都很简单，就是约一个规则，通过共同执行这个规则，让每个分布式的节点数据都保持最终一致；

## 梅克尔数据证明

这是利用哈希算法将一组数据创建为一棵哈希树结构，用于验证数据完整性的一种结构，同时也应用在了轻量级钱包中。不同的区块链系统对梅克尔树的应用不尽相同，比特币中是二叉梅克尔树，比较简单，通过交易事务的哈希值两两配合生成一棵树，以太坊这种就复杂的多了，称之为梅克尔.帕特里夏树，这里暂时不赘述。

## 可编程脚本合约

什么叫合约？就是一组约定的规则，例如银行的结算系统，小明转账100给小王，在这个过程中，银系统就会根据一组规则自动执行，规则包含如检测小明的密码是否正确，余额是否足够，小王的账号是否正确，检测通过则分别更改两者的账户余额并写入事务日志。是的，这就是合约的意思了，当然，如果范围再广泛些，各种商业合约也都是这么个意思，因此可编程脚本合约也没什么好稀奇的，如果将这种编程合约放到区块链的环境中，比较有趣了，看两个特点：

第一个，区块链系统是无中新的分布式网络，没有边界

第二个，区块链系统通过一系列的技术实现了可信任网络

加起来，就是无边界的可信任网络，在这样的网络中执行既定的合约，成本低而且安全，比特币在本质上也是属于这种脚本合约，只不过在比特币软件中，合约中处理的事情是比特币的转账