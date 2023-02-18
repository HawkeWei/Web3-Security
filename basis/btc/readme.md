# 比特币模型详解

有人说，Web3就像《三体》中的“黑暗森林”，那么该课程将带你进入“猎人”的门。

**Twitter:** [@brycewai](https://twitter.com/brycewai)  
**Mirror:** [@Bryce.W](https://mirror.xyz/brycew.eth)  
**所有代码和教程开源在Github:** [https://github.com/brycewai/Web3-Security](https://github.com/brycewai/Web3-Security)

## 1 数据层

区块链是由Hash指针链接的单向链表。这个链表连接一系列的区块，每个区块包含数据和指向前一区块的指针。区块链中，前一个区块指针使用哈希指针替代了传统指针，因此每个区块不止告知了前一个区块的位置，也提供一个哈希值保证了前一个区块的所有内容。

下面是区块链中区块的基本结构：

![Untitled](img/Untitled.png)

其中最重要的部分是Blockheader、Transactions，下面将会对其进行分别介绍：

### 1.1 Blockheader

![Untitled](img/Untitled%201.png)

**1.1.1 Merkle tree root**

Merkle tree是一种哈希二叉树（即用Hash指针代替了普通指针），它是一种用作快速归纳和校验大规模数据完整性的数据结构。它被用来归纳一个区块中的所有交易，同时生成整个交易集合的数字指纹。

![Untitled](img/Untitled%202.png)

涉及到的相关概念如下：

- **Merkle proof**：判断某交易是否属于Merkle tree上所需的数据，包含叶子节点、路径、根；
- **SPV（Simplified Payment Verification，简单支付验证）**：判断用于“支付”的那笔交易是否已经被验证过（即是否已经上链），并得到了多少的算力保护（多少确认数）；
- **Merkle tree root** ：Merkle tree的root值；

**Merkle tree在比特币系统中的应用为：**

1） **提供Merkle proof**：由区块结构可知，区块头部仅存放Merkle root，而Merkle tree的其他部分存放在block body中。全节点拥有block header和body，但是轻节点仅拥有header，此时需要向轻节点证明某个交易是写入到区块链中的，其具体证明过程如下：

![Untitled](img/Untitled%203.png)

```
a. 轻节点首先向某个全节点请求包含该交易的Merkle proof

b. 全节点会返回上图中的红色Hash值

c. 轻节点在本地可进行SPV验证，具体计算过程为：首先计算TxB的Hash值，接着将该Hash值与红色Ha进行拼接计算出更上一层的Hash值Hab，这样一层层计算，最终计算出Merkle root。轻节点将这个Hash与block header中的Merkle root进行比较，验证该交易是否存在。

```

2） **仅用Merkle root可保护整棵树，即整个交易集合**

因为Merkle tree的性质决定，任何树中节点的修改都会导致root值的改变。

**Tips**:

- Merkle tree是自底向上构建的，底层的每个数据结构是一个个交易tx；
- 因为Merkle树是二叉树，所以它具有偶数个叶子节点。如果仅有奇数个交易需要归纳，那最后的交易就会被复制一份以构成偶数个叶子节点，这种偶数个叶子节点的树也被称为平衡树。
- Merkle proof 的时间复杂度为θ(logn)
- 比特币中如果需要证明一个交易不在Merkle tree中，如果交易不排序，时间复杂度为线性遍历，即θ(n)。如果交易排序则与证明一个交易存在时即Merkle proof相同，都为θ(logn)。
- Merkle tree的叶子节点是交易hash值

### 1.2 Transactions交易

比特币中的交易流程具体如下：

```
1. 源节点创建交易，并验证目的节点的地址；
2. 源节点对交易进行签名加密；
3. 源节点将该交易广播至全网其他节点；
4. 全网节点接收交易并验证其有效性，直到该交易被全网大多数节点验证和接受；
5. 交易暂存于节点内存池，并判断是否是孤立交易；
6. 交易被打包至节点本地区块中;
7. 全网共识结束后，获胜节点将其本地区块追加到主链；
8. 交易在主链上被越来越多的后续区块确认；

```

**coinbase交易：**

![Untitled](img/Untitled%204.png)

**普通交易：**

![Untitled](img/Untitled%205.png)

**孤立交易：**
交易并不总是能够按照顺序达到目的节点，有时子交易会在父交易之前到达，从而产生暂时缺失父交易的孤立交易。这些交易将会被暂存在孤立交易池中，一旦接收到父交易之后，所有与这个交易创建的UTXO有关的孤立交易都将从孤立交易池中释放出来，经过验证之后添加到内存池中，形成一条完整的交易链。

**1.2.1 UTXO交易模型**

**UTXO定义：** Unspent Transaction Output。面值为“聪”的离散（不连续）且不可分割的价值单元，一个UTXO只能在一次交易中作为一个整体被消耗。一笔交易会消耗先前的已被记录（存在）的UTXO，并创建新的UTXO以备未来的交易消耗。

- **UTXO交易模型——交易的输出**

    除“数据输出操作符”（OP_RETURN）外，每一笔比特币交易都会创造输出。

    交易的输出包含两部分：

    1、一定量的比特币，面值为“聪”（satoshis） ，是最小的比特币单位；

    2、确定花费输出所需条件的加密难题（cryptographic puzzle），也称为**锁定脚本**(locking script)，或脚本公钥 (scriptPubKey)。

    下面是具体的一个交易输出实例：

    ```json
    "vout": [
      {
        "value": 0.01500000,
        "scriptPubKey": "OP_DUP OP_HASH160 ab68025513c3dbd2f7b92a94e0581f5d50f654e7 OP_EQUALVERIFY OP_CHECKSIG"
      },
      {
        "value": 0.08450000,
        "scriptPubKey": "OP_DUP OP_HASH160 7f9b1a7fb68d60c536c2fd8aeaa53a8f3cc025a8 OP_EQUALVERIFY OP_CHECKSIG",
      }
    ]
    
    ```

    由上述交易实例可知，该笔交易包含两个输出，其中第一部分的value值代表输出的金额，是一个以“聪”为单位的整数。第二部分是锁定脚本，即设定这些UTXO未来被消费时需要满足的条件。

- **UTXO交易模型——交易的输入**

    交易的输入代表比特币的来源，标识哪些UTXO将被消费，并通过解锁脚本提供所有权证明。

    交易的输入可以包含如下几个部分：

    1、一个交易ID，引用包含将要消费的UTXO的交易

    2、一个输出索引（vout），用于标识来自该交易的哪个UTXO被引用（第一个为零）

    3、一个 **scriptSig（解锁脚本）**，满足UTXO的消费条件，解锁用于支出

    4、一个序列号（后续查该字段含义和意义）

    以下是一个具体的交易实例：

    ```json
    "vin": [
    {
      "txid": "7957a35fe64f80d234d76d83a2a8f1a0d8149a41d81de548f0a65a8a999f6f18",
      "vout": 0,
      "scriptSig" : "3045022100884d142d86652a3f47ba4746ec719bbfbd040a570b1deccbb6498c75c4ae24cb02204b9f039ff08df09cbe9f6addac960298cad530a863ea8f53982c09db8f6e3813[ALL] 0484ecc0d46f1918b30928fa0e4ed99f16a0fb4fde0735e7ade8416ab9fe423cc5412336376789d172787ec3457eee41c04f4938de5cc17b4a10fa336a8d752adf",
      "sequence": 4294967295
    }
    ]
    
    ```

    由上述交易实例可知，vin仅包含一个json对象，代表其仅包含一个输入（多个输入需要多个签名）。因为一个UTXO的面值足够付款；输出索vout=0，代表该UTXO是txid对应交易创建的第一个UTXO；解锁脚本由钱包创建，首先检索引用的UTXO，接着检查其锁定脚本，最后用它来构造满足要求的解锁脚本。

- **UTXO交易模型——输入输出脚本**

    当我们要支付给某个人比特币时，我们需要确保，只有接收者能花费这些比特币。在比特币系统中，会使用输出脚本来锁定交易输出，只有接收者拥有秘钥，当需要花费这笔交易输出时，提供解锁脚本即可。

    比特币系统中，另一个巧妙的设计时，任何人都可以通过验证解锁脚本是否能打开锁定脚本，来判断这个交易是否合法。而在验证的过程中，支付者只需提供解锁脚本，即可完成验证,无需对外提供自己的私钥。

    下图所示为一个小型区块链，左图TX中存在一个A→ B的转账交易，在链上隔了2个区块后右边的TX中存在一个B → C的交易。则左边的交易是右边交易的比特币来源。下面将对输入和输出脚本的执行进行具体的介绍。

    ![Untitled](img/Untitled%206.png)

    A → B的交易中，生成的B可支配的UTXO被A→ B的vout输出中的锁定脚本outscript锁定。

    B → C的交易中，需要首先用上述A→ B交易中的vout锁定脚本与B → C交易中的vin解锁脚本拼接，接着验证拼接的两个脚本，验证通过后，最后用B → C交易中的vout锁定脚本对新生成的C可支配的UTXO进行重新锁定。

    早期是inputscript和outscript拼接在一起执行验证，之后处于安全的考虑先执行inputscript，再执行outscript。

**💡 注意：理解时不要以交易来理解，应该以交易的本质UTXO为核心进行理解。**

**1.2.2 交易类型**

一共有三种交易类型，包括：P2PK、P2PKH、P2SH。交易中主要包括地址、脚本两部分。其中，地址用来表示交易中比特币的来源或目的地。

**1）地址生成**

**私钥**：私钥是一个**32字节**的数，私钥可以由伪随机算法(PRNG)产生。其实0也是一个合法的私钥，只不过这是一个特殊私钥，以太坊的创世区块就是这个私钥生成的。

**公钥**：非压缩公钥是一个65字节的数，其中32字节表示椭圆曲线的X坐标，32字节表示椭圆曲线的Y坐标，还有1字节是前缀标识。这个XY坐标是私钥通过ECDSA-secp256k1推导出来的。所以说，**椭圆曲线算法的公钥是通过私钥计算出来的。而反过来，用公钥推导私钥，以现有计算机的计算几乎是不可能的。**

![Untitled](img/Untitled%207.png)

**具体计算流程如下：**

```
0. 私钥 (private key)产生：伪随机数产生的256bit私钥
1. 私钥经secp256k1椭圆曲线推出的公钥（前缀04+X公钥+Y公钥）
2. 计算公钥的 SHA-256 哈希值（32bytes）
3. 取上一步结果，计算 RIPEMD-160 哈希值(20bytes)
4. 取上一步结果，前面加入地址版本号（比特币主网版本号“0x00”）
5. 取上一步结果，计算两次 SHA-256 哈希值(32bytes)
6. 取上一步结果的前4个字节（8位十六进制）
7. hash of public key，把这4个字节加在第4步的结果后面，作为校验（这就是比特币地址的16进制形态）
8. 最终给用户使用的：用base58编码变换一下地址（这就是最常见的比特币地址形态）
```

**不同交易类型对应不同的地址格式：**

- **P2PK格式**

    直接以公钥作为地址进行交易

- **Legacy (P2PKH)格式**

    范例：1Fh7ajXabJBpZPZw8bjD3QU4CuQ3pRty9u

    地址以“1”开头，是比特币最初的地址格式，至今仍在使用。

- **Nested SegWit (P2SH)格式**

    范例：3KF9nXowQ4asSGxRRzeiTpDjMuwM2nypAN

    地址以"3"开头，支持比Legacy 地址更复杂的功能。

**2）输入/输出脚本**

**1、P2PK（Pay to Public Key）**

![Untitled](img/Untitled%208.png)

将解锁脚本（输入）和锁定脚本（输出）拼接，结果如下图所示：

![Untitled](img/Untitled%209.png)

由上图可知，执行脚本中第一行来自交易的输入（该UTXO的解锁脚本），第二行来自交易的输出（该UTXO的锁定脚本），第三行代表签名验证。右边蓝色小框为执行过程形成的堆栈结构，即首先将解锁脚本的签名Sig压栈，之后将该UTXO的锁定脚本的公钥PubKey压栈，最后进行签名验证，通过则返回TURE代表验证通过。

**2、P2PKH（Pay to Public Hash）最常用**

![Untitled](img/Untitled%2010.png)

该方式与P2PK方式的区别为：交易的输出中没有直接给出收款人的公钥，给出的是公钥的hash值。而公钥在交易的输入中给出。

![Untitled](img/Untitled%2011.png)

由上图可知，执行脚本的前两行来自交易的输入（该UTXO的解锁脚本），后面所有行来自于交易的输出（该UTXO的锁定脚本）。右边蓝色框为执行时的堆栈结构。即首先将解锁脚本中的签名Sig和对应的公钥PubKey压入栈，接着DUP操作表示将此时的栈顶元素公钥PubKey复制并压栈，之后进行Hash操作生成160bit的公钥Hash值PubHash并压栈，随后使用EQUALVERIFY比较两个Hash值是否相等，最后进行CHECKSIG签名验证。

**3、P2SH（Pay to Script Key）**

![Untitled](img/Untitled%2012.png)

该方式与上述两种方式区别为：交易的输出中给出的不是收款人的公钥或公钥Hash，而是收款人提供的redeemScript赎回脚本Hash。

![Untitled](img/Untitled%2013.png)

由上述描述可知，该方式的验证分为两步，第一步是验证输入脚本给出的赎回脚本redeemScript是否正确，第二步是执行赎回脚本。下面是P2PKH形式的P2SH方式：

![Untitled](img/Untitled%2014.png)

上述方式中存在三种类型的脚本：赎回脚本redeemScript，交易的输入即解锁脚本，交易的输出即锁定脚本。下图为第一步的验证：

![Untitled](img/Untitled%2015.png)

由上图可知，前两行来自该UTXO的解锁脚本，后面三行来自该UTXO的锁定脚本。首先将解锁脚本中的签名Sig压栈，接着将序列化的赎回脚本redeemScript压栈，随后计算赎回脚本的Hash值并压栈，之后将该UTXO锁定脚本中的赎回脚本Hash值压栈，最后比较两个Hash值是否相等。下图为验证成功的结果：

![Untitled](img/Untitled%2016.png)

如果第一阶段验证成功，将进行第二阶段的验证，具体如下图所示：

![Untitled](img/Untitled%2017.png)

由上图可知，首先需要将提供的序列化的赎回脚本进行反序列化，之后再执行赎回脚本。即首先将公钥PubKey压栈，最后进行签名验证。

**4、多重签名——-P2SH的应用场景**

比特币系统中，一个输出（锁定的UTXO）可能要求多个签名才能解锁使用，如一个公司可能要求5个合伙人中至少有3个合伙人的签名，才能把钱取走。这种方式的目的是为私钥提供保护：如出现某个合伙人的私钥泄露了，或者丢失了。

下图为使用P2SH方式实现的多重签名，目的是为了在当收款人（交易的输出）采取多重签名的情况下，付款方（交易的输入）无需知道收款方采取的多重签名细节。细节都存储在redeemScript赎回脚本中，其中M、N表示该UTXO需要在满足N个公钥中至少存在M个对应签名才能解锁。

![Untitled](img/Untitled%2018.png)

💡 注意：给出的公钥和签名的相对顺序必须一致。

![Untitled](img/Untitled%2019.png)

由上图可知，前四行为交易的输入（UTXO的解锁脚本），后三行为交易的输出（UTXO的锁定脚本）。其中第一行的FALSE为最早的多重签名中CHECKSIG存在一个bug，会从堆栈中多弹出一个元素，所以为了暂时修复这个bug，会在堆栈顶压入一个无意义的FALSE。脚本执行时，首先将FALSE、Sig_1、Sig_2、序列化的赎回脚本依次压栈，接着计算序列化脚本的Hash值，随后将交易输出（UTXO的锁定脚本）中的赎回脚本Hash值RSH也压栈，最后判断赎回脚本Hash值是否相等。相等则进入下一阶段验证，具体如下：

![Untitled](img/Untitled%2020.png)

由上图可知，第二阶段的验证将执行赎回脚本。依次将赎回脚本中的M、公钥、N值压栈，最后执行签名验证，确认堆栈中是否满足需要验证的签名个数。

## 2 网络层

区块链的网络层封装了区块链系统的组网方式、数据传输协议。

### 2.1 网络架构

比特币采用了基于互联网的点对点（P2P：peer-to-peer）分布式网络架构。

![Untitled](img/Untitled%2021.png)

一般来说，分为授权链和非授权链：

- 非授权链：诸如比特币、以太坊为代表的非授权链大多采用去中心化网络，其网络节点一般具有海量、分布式、自治、开放可自由进出等特性，因而大多采用对等网络（P2P网络）来组织散布全球的参与数据验证和验证的节点；
- 授权链：大多采用中心化星型网络或多中心网络结构，如联盟链大多采用多中心化网络，而私有链则可能采用完全中心化的星型网络；

### 2.2 网络节点

比特币中网络中的节点总共包含以下四种类型的功能：

![Untitled](img/Untitled%2022.png)

网络节点主要分为标准**客户端节点、全节点、轻节点、矿工节点**，具体如下：

![Untitled](img/Untitled%2023.png)

### 2.3 数据传播协议

节点通常采用TCP协议，使用8333端口与其他对等节点交互。一个通用的区块链系统通常包含以下应用场景：

- 节点入网建立初始连接
- 节点地址传播发现
- 矿工、全节点同步区块数据
- 交易验证与传播

**2.3.1 节点入网建立初始连接**

![Untitled](img/Untitled%2024.png)

1、节点A首先发送nVersion来给节点B确定版本是否兼容

2、如果节点B兼容，则B发送Verack消息给节点A来建立连接

3、接着节点B也会发送nVersion来给节点A确定版本是否兼容

4、如果节点A兼容，则A发送Verack消息给节点B来建立连接

**2.3.2 节点地址传播发现**

一旦建立连接，新节点将向其相邻节点发送包含自身IP地址的addr消息。相邻节点则将此addr消息再度转发给各自的相邻节点，进而保证新节点被更多节点获知。此外，新节点还向其相邻节点发送getaddr消息，获取邻居节点可以连接的节点列表。整个过程如下所示：

![Untitled](img/Untitled%2025.png)

**2.3.2 矿工、全节点同步区块数据**

新入网节点只知道内置的创世区块，因此需要同步最新区块。同步过程始于发送version消息，该消息含有节点当前区块高度。具体而言，连接建立后，双方会互相发送同步消息getblocks，其包含各自本地区块链的顶端区块哈希值。通过比较，区块数较多的一方向区块较少的一方发送inv消息。注意，inv消息只是一个清单，并不是实际的数据。落后方收到inv消息后，开始发送getdata消息请求数据，具体如下图所示：

![Untitled](img/Untitled%2026.png)

需要注意的是，SPV节点同步的不是区块数据，而是区块头，所以使用getheaders()消息，具体如下图所示：

![Untitled](img/Untitled%2027.png)

**2.3.4 交易传播**

比特币的交易数据传输协议包括如下步骤：

1. 比特币交易节点将新生成的交易数据向全网所有节点进行广播；
2. 每个节点都将收集到的交易数据存储到一个区块中;
3. 每个节点基于自身算力在区块中找到一个具有足够难度的工作量证明；
4. 当节点找到区块的工作量证明后，就向全网所有节点广播此区块（block消息）；
5. 仅当包含在区块中的所有交易都是有效的且之前未存在过的，其他节点才认同该区块的有效性；
6. 其他节点接受该数据区块，并在该区块的末尾制造新的区块以延长该链条；

## 3 共识层

共识层主要是为了解决分布式节点达成一致的过程及其算法，即如何确保各个节点在网络层提供的网络环境和通信模式中可以共享同一份有效的区块链视图。具体地说，即**谁拥有记账权可以生成合法区块，而区块链上的其他节点又如何验证记账节点生成区块的合法性并最终达成共识。**

其中，区块链**通用的共识过程如下：**

第一阶段：选主 （从全体矿工节点集中选举出记账节点的过程）；

第二阶段：造块 （记账节点根据特定的策略将当前时间段内全体节点P生成的交易或者数据打包到一个区块中，并将生成的新区块广播给全体矿工节点）；

第三阶段：验证 （矿工节点收到广播的新区块后将各自验证区块中包含的交易和数据的合法性，如果新区块获得大多数验证节点的认可，则将该区块作为下一区块更新到区块链）；

第四阶段：上链 （记账节点将新区块添加到主链）；

### 3.1 共识问题

分布式系统中的共识问题，根据是否存在故障节点又分为两军问题和拜占庭将军问题两类，具体如下所示：

1. **两军问题**

- 问题描述

    两只由不同将军领导的军队，正准备进攻一座坚固的城市。军队在城市附近的两个山谷扎营。由于有另一个山谷将两个山丘隔开，两个将军交流的唯一方法是派遣信使穿越山谷。然而，这座山谷被城市的守卫者占领，并且有可能会俘虏途径该山谷传递信息的任意信使。

    尽管两位将军已经约定要同时发起进攻，但尚未约定进攻的具体时间点。要使攻击顺利，两支军队必须同时进攻城市。如果同一时间仅一组军队进攻，将会战败。因此，两位将军须通过沟通约定攻击时间，并且他们都必须确保另一位将军知道自己已同意了进攻计划。但由于传递确认讯息的信使与传递原始消息的信使一样，都可能被俘虏造成消息丢失，即使双方不断确认已收到对方的上一条信息，也无法确保对方已与自己达成共识。

- 问题实质

    因此，该问题本质上是如何找到一个可行的算法，使得系统能够在**节点可信而信道不可信**的情况下达成共识。

2. **拜占庭将军问题**

- 问题描述

    拜占庭帝国的将军们各自率领自己的军队围攻敌方的一座城池，各军队间通过信使传递消息，以期达成共识制定一个统一的合理作战计划。这个过程中可能存在叛徒故意发送虚假消息，以破坏正确共识的建立。

- 问题实质

    因此，该问题本质上是如何找到一个可行的算法，使得系统能够在**存在不可信节点而信道可信**的情况下达成共识。具体地说，忠诚的将军必须按照算法要求来行动，而叛徒则可以任意行动。此时，拜占庭将军算法必须保证忠诚的将军的通信和决策同时满足**一致性条件**和**正确性条件**。

    **一致性条件**：所有忠诚的将军必须按照相同的行动计划行事；

    **正确性条件**：少数叛徒不能使得忠诚的将军采纳错误的行动计划；

    💡 拜占庭将军问题解决的是同步网络和任意故障模型情况下的共识问题；
    **拜占庭将军问题解决的是信道可靠而节点不可靠的问题，而两军问题解决的是节点可靠但是信道不可靠的问题。**

### 3.2 常见共识算法

**3.2.1 工作量证明 (Proof of work, PoW)**

定义：比特币系统中 , 各节点 ( 即矿工 ) 基于各自的算力相互竞争来共同解决一个求解复杂但验证容易的 SHA256 数学难题 ( 即挖矿 ), 最快解决该难题的节点将获得区块记账权和系统自动生成的比特币奖励。PoW 共识的随机数搜索过程如下：

1. 搜集当前时间段的全网未确认交易 , 并增加一个用于发行新比特币奖励的 Coinbase 交易，形成当前区块体的交易集合。
2. 计算区块体交易集合的 Merkle 根记入区块头 , 并填写区块头的其他元数据 , 其中随机数Nonce 置零。
3. 随机数 Nonce 加 1; 计算当前区块头的双 SHA256 哈希值 , 如果小于或等于目标哈希值 ,则成功搜索到合适的随机数并获得该区块的记账权 ;否则继续步骤 3 直到任一节点搜索到合适的随机数为止。
4. 如果一定时间内未成功 , 则更新时间戳和未确认交易集合、重新计算 Merkle 根后继续搜索。

**3.2.2 权益证明 (Proof of stake, PoS)**

定义：PoS 共识是为解决 PoW 共识机制的资源浪费和安全性缺陷而提出的替代方案。是由系统中具有最高权益而非最高算力的节点获得区块记账权。权益体现为节点对特定数量货币的所有权，称为币龄或币天数 (Coin days)。币龄是特定数量的币与其最后一次交易的时间长度的乘积，每次交易都将会消耗掉特定数量的币龄。

币龄 = 持有的币数 * 持有币的天数

**💡 PoS缺点：**

**1.更容易被垄断**：因为持币越多，持有的越久，币龄就越高，越容易挖到区块并得到激励，持币少的人基本上没有机会，这样整个系统的安全性实际上会被持币数量较大的一部分人（大股东们）掌握；而PoW理论上则不存在这个问题，因为理论上任何人都可以购买矿机获得提高自己的算力（甚至可以联合起来），提升自己挖矿成功的概率；

**2.很难应对分叉的情况**：当出现分叉时，PoS可以在两条链上同时挖矿并获得收益。而PoW则不存在这个问题，因为当出现分叉以后，**PoW总是选择工作量大的链做为主链（即最长合法链）**。

但是在实际应用中，纯PoS的共识机制是不可行的，通常会和PoW混合一起用或者通过PoS升级改进PoW。这样会更好的发挥各自的优点，减小双方缺点带来的影响。

**3.2.3 实用拜占庭容错算法（Practical Byzantine Fault Tolerance，PBFT）**

PBFT算法假设共识过程是在一个异步分布式网络，该网络中可能发生消息传输失败、延迟、重复发送或乱序等，并且可能存在蓄意发送错误消息的**恶意节点**。

PBFT是一种基于状态机复制的实用共识算法：服务被建模为状态机，在分布式系统的不同节点上进行复制；每个状态机副本都维护了服务状态，并实施服务的相应操作。该算法可以在不高于33%的拜占庭错误节点的系统中保持活性和安全性。所有副本的状态变迁通过成为视图（view）的配置更换进行，每个view中只有一个副本是主节点，其余副本作为备份。

与其他状态机复制技术一样，PBFT对每个副本节点提出了**两个限定条件**：

    1. 所有节点必须是确定的，即在给定状态和参数的情况下，操作执行的结果必须相同；
    2. 所有节点必须从相同的状态开始执行；

在这两个限定条件下，即使失效的副本节点存在，PBFT算法对所有非失效副本节点的请求执行总顺序达成一致，从而保证安全性。

为了保证PBFT算法的正确性，节点总数量N和作恶节点数量f必须满足N>3f。

具体证明过程为：
a) 因为我们知道有f个作恶节点，所以我们必须在n-f个状态复制机的沟通内，就要做出决定。
为什么呢？因为我们在设计异步通信算法的时候，我们不知道那f个节点是恶意节点还是故障节点，这f个节点可以不发送消息，也可以发送错误的消息，所以在设计阈值的时候，我们要保证必须在n-f个状态复制机的沟通内，就要做出决定，因为如果阈值设置为需要n-f+1个消息，那么假设这f个作恶节点全部不回应，那这个系统根本无法运作下去。所以。考虑最大容错，必须要求n-f个节点就能做出决定。
b) 而在n-f个状态复制机的沟通内，我们无法预测这f个作恶节点做了什么（错误消息/不发送），所以这里也考虑最大容错，假设(n-f)个里面有f个是作恶节点。而我们必须保证正常的节点大于作恶节点数。所以有(n-f) - f > f，从而得出了n > 3f。

PBFT的完整流程共分为**五个阶段**：**请求（Request）、预准备（Pre-prepare）、准备（Prepare）、确认（Commit）和回复（Reply）**。其中，预准备和准备两个阶段用来确保同一视图请求发送的时序性（使对请求进行排序的主节点失效）；准备和确认两个阶段用来确保在不同视图之间的确认请求时严格排序的。

![Untitled](img/Untitled%2028.png)

- **请求（Request）**

    客户端C向主节点0发送<REQUEST, o, t, c>请求，其中o为请求的具体操作；t为请求时客户端追加的时间戳，用于保证请求只被执行一次，并且由于时间戳递增，可以用来比较操作顺序；c为客户端编号

- **预准备（Pre-prepare）**

    主节点0收到客户端的请求后，首先校验客户端请求消息的签名是否正确。如果是非法请求则丢弃，否则就分配一个编号n给收到的客户端请求消息，然后向所有副本节点广播预准备消息<<PRE-PREPARE, v, n, d>, m>，同时将该消息追加到其日志中。这里m为客户端发送的原始请求消息；v为视图编号；n是主节点为该请求分配的编号；d是m的摘要；

- **准备（Prepare）**

    副本节点i只接受满足如下条件的预准备消息：

    1. 首先请求和预准备消息的签名正确，并且d与m的摘要一致；
    2. 其次当前视图编号是v；
    3. 该副本节点从未在视图v中接受过序号为n但是摘要d不同的消息m；
    4. 预准备消息的序号n必须在水线（watermark）上限H和下限h之间；水线存在的意义在于防止一个失效节点使用一个很大的序号消耗序号空间。

    验证通过之后则进入准备阶段并向其他节点广播准备消息<PREPARE, v, n, d, i>，其中v、n、d与预准备消息相同，i为副本节点编号，同时将预准备消息和准备消息写入自己的消息日志；包括主节点在内的所有副本节点在收到准备消息后，验证其有效性，如果验证通过则将准备消息写入日志。

- **确认（Commit）**

    主节点和副本节点收到Prepare消息，需要进行如下校验：

    1. 副本节点Prepare消息签名是否正确；
    2. 当前副本节点是否已经收到了同一视图v下的n；
    3. n是否在区间[h,H]内；
    4. d是否和当前已收到的预准备消息中的d相同；

    如果副本节点i收到收到了2f+1个验证通过的确认，则向其他节点包括主节点发送<COMMIT, v, n, d, i>消息，v, n, d, i与上述PREPARE消息内容相同。记录COMMIT消息到日志中，用于View Change过程中恢复未完成的请求操作。记录其他副本节点发送的PREPARE消息到log中。

- **回复（Reply）**

    主节点和副本节点收到COMMIT消息，需要进行以下交验：

    1. 副本节点COMMIT消息签名是否正确。
    2. 当前副本节点是否已经收到了同一视图v下的n。
    3. d与m的摘要是否一致。
    4. n是否在区间[h, H]内。

    如果副本节点i收到了2f+1个验证通过的COMMIT消息，说明当前网络中的大部分节点已经达成共识，则运行客户端的请求操作o，并返回<REPLY, v, t, c, i, r>给客户端，r：是请求操作的结果，客户端如果收到f+1个相同的REPLY消息，说明客户端发起的请求已经达成全网共识，否则客户端需要判断是否重新发送请求给主节点。记录其他副本节点发送的COMMIT消息到log中。

    该算法可以参考fisco bcos的实现案例，具体参考链接为：

    [PBFT基础流程 - FISCO BCOS v2.9.0 文档](https://fisco-bcos-documentation.readthedocs.io/zh_CN/latest/docs/design/consensus/pbft.html)

    [深入浅出PBFT算法原理](https://www.jianshu.com/p/78e2b3d3af62)

### 3.3 出块过程

- **基本概念：**

    **交易池**：用于存储待确认交易，每个比特币挖矿节点均有自己独立的交易池。因为每个交易池体积、最低交易费比例（每B字节交易的手续费为x聪比特币）限制等不同，各节点的交易池也不相同。矿工在构造预备区块时，需要从交易池中选择要打包的交易。

    **挖矿节点：** 在比特币网络中，参与记录和验证比特币交易和区块的是一个个保存比特币数据的节点。其中有一部分节点，不仅参与记录和验证的工作，还参与比特币新区块的创建工作，他们构造新区块，并通过PoW工作量证明竞争记账权，进而获得创建新区块的权限，这部分节点是挖矿节点。

    **UTXO：** 比特币节点通过扫描节点所有交易信息，构建的UTXO集群。它包含所有未被消耗的UTXO。

    **Coinbase奖励：** 比特币协议规定，每产生一个新的比特币区块，比特币网络就会产生N个比特币，作为维护比特币网络的奖励支付给创建这个区块的矿工。

    **出块过程：（记账节点才能出块）**

    1. **待确认交易加入交易池**

    当我们要发起一笔比特币交易时，交易发起方构造好交易信息，此时的交易信息是**待确认的交易。**

    待确认交易在经过验证后，由交易发起方向比特币网络广播，比特币网络中的节点，均可验证和收录广播的信息。其中，挖矿节点会在收到广播后，验证待确认交易信息，验证通过后，挖矿节点会将待确认交易加入到自己的交易池中。

    ![Untitled](img/Untitled%2029.png)

    💡 注意：比特币节点只能验证交易是否在区块链上存在，而挖矿节点才能对交易本身进行验证。

    2. **构建预备区块**

    当挖矿节点要构造预备区块，准备生成新区块时，会按照优先级排序，从交易池中取待确认交易。预备区块通常会预留一定空间给高优先级的交易，剩下的空间会按照交易费比例（Sat/B）由高到低顺序一直把区块加满或者把交易池的交易用光。

    3. **生成合法区块**

    挖矿节点构建好预备区块后，就会将区块头信息下发给矿工，矿工通过不断调整区块头中的随机数来变更预备区块的哈希值，当预备区块的哈希值低于比特币网络当前目标哈希值时，这个区块就是一个合法新区块。

    4. **确认区块**

    挖矿节点会及时地向比特币网络广播新区块，比特币网络中其他比特币节点在接到广播信息后，对新区块进行验证，验证通过后，将新区块加入本地，并延长节点的区块链。此时，新区块创建并确认完毕，对应交易也完成了。其中，不同的共识算法涉及到的区块验证方式不同，本文将在3.4节中介绍比特币中使用到的区块验证算法。

### 3.4 比特币区块验证

相邻节点收到新区块后，立即做以下检查：

1. 验证PoW的nonce值是否小于难度值
2. 检查时间戳是否小于当前时间2小时（10分钟一个块）
3. 检查Merkle tree root值是否正确
4. 检查区块size要小于区块size上限
5. 第一笔交易必须是coinbase交易
6. 验证每一笔交易

- 出块时间

    10分钟

### 3.5 分叉

**state fork**：当有两个节点同时挖到矿，则该两个节点都可以发布区块，此时会出现临时性分叉。此时造成整个分叉的原因是未对区块链的状态达成共识。

**protocol fork**：在一个去中心化的系统中升级软件（即区块链上运行的协议）时，没法保证所有节点同时都升级软件。本质上是此时新旧节点对区块链中的协议产生了分歧，旧节点未升级可能存在如何两个原因：没来得及、不同意。根据对协议修改的内容的不同，又可将protocol fork分为**硬分叉**和**软分叉**。

**硬分叉：** 永久性分叉，不向前兼容。旧版本不会接受新版本创建的合法区块，认为新版本的合法区块是不合法的。所以很明显硬分叉是不向前兼容。

最终造成比特币区块链分叉为平行的两条链，此时实际上分裂为两个币种。

💡 此时上下两条链账户余额也不再相同，因为两条链彼此分离了。形成了上下两条链账户一样，私钥一样只是运行协议不一致了，这种情况下需要上下链各自记账。
真实案例：BTC和BCH

**软分叉：** 临时性分叉，向前兼容，即旧版本兼容新版本。所有被新版本认为是合法的区块也会被以前旧版本认为是合法的。旧版本会接受新版本创建的区块。

**软分叉和硬分叉的区别：**

硬分叉修改数据结构后，新节点生产的区块会被旧节点拒绝掉，如果旧节点拒绝升级软件而坚持按照旧数据结构继续挖矿，那比特币就会产生两条链。

软分叉修改数据结构后，新节点生产的交易和区块能够被旧节点验证并接受，所以不会出现两条链。

💡 **注意：在理解软硬分叉时，注意前提是系统中大部分节点已经更新到新版本，即新版本具有系统中至少51%的算力。**

## 4. 激励层

去中心化系统中的共识节点本身是自利的，最大化自身收益是其参与数据验证和记账的根本目标。因此，必须设计激励相容的合理众包机制，使得共识节点最大化自身收益的个体理性行为与保障去中心化区块链系统的安全和有效性的整体目标相吻合。

**发行机制：**

**coinbase奖励**：比特币系统中每个区块发行比特币的数量是随着时间阶梯性递减的。

**手续费**：交易输入和输出的差值。

💡 手续费还可以防止大量微额交易对比特币网络发起粉尘攻击，起到保障安全的作用。

**分配机制：** 主要是由于小算力节点为了收益会采取加入矿池的方式进行挖矿，而不同的矿池会采取不同的分配制度进行分配。

## 5. 应用层

### 5.1 私钥存储—钱包

- **钱包分类**

    按照钱包**是否联网**进行划分：

    1）**冷钱包**：冷钱包其实可以翻译成“离线钱包”，采取分散存储方式，每个地址存确定数量的比特币；安全性高。**（不联网使用）**

    常见的几种冷钱包：硬件钱包、纸钱包

    **硬件钱包**：是用于存储加密资产的专用电子设备，其功能就是将私钥存储在内部的芯片内，私钥永远都不会离开设备，因此很安全。私钥通过助记词生成。（Ledger和Trezor是使用最广泛的硬件钱包）

    2）**热钱包**：热钱包是指互联网能够访问你私钥的钱包；安全性比冷钱包差。**（实时联网在线）**

    常见的几种热钱包：网页钱包、电脑钱包、手机钱包

    **软件钱包**：通过软件存储私钥。安全性比硬件钱包差。

    💡 软件钱包成本非常低，使用起来非常便利，但是用户的私钥一般是通过加密等方式保存在本地，然后在使用的时候解密。这样就导致一个比较严重的安全问题：**至少在某一段时间内，私钥会以明文的方式出现在本地，可能存在泄露或者被攻击者利用**，这样就对数字资产的安全造成了直接的不可忽视的威胁。这也是软件钱包在安全上最大软肋。

- **助记词**

    助记词是派生确定性钱包的种子的随机数的单词序列。一般由12、15、18、21个单词构成, 这些单词都取自一个固定词库, 其生成顺序也是按照一定算法而来。采取BIP-39标准。

- **keystore**

    这是一个JSON编码的文件，其中包含一个（随机生成的）私钥，由一个密码加密以提高安全性。用于派生确定型钱包。

    keystore格式使用_Key派生函数（KDF），也称为密码扩展算法，该算法可防止对密码加密的暴力破解，字典或彩虹表攻击。简而言之，私钥没有直接由密码短语加密。相反，通过反复对它进行哈希，密码被拉长。

💡 虽然助记词和 Keystore 都可以作为私钥的另一种表现形式，但与 Keystore 不同的是, 助记词是未经加密的私钥, 没有任何安全性可言。

### 5.2 比特币客户端

用户可以从bitcoin.org下载标准客户端，即比特币核心，也叫“中本聪客户端”（satoshi client）。它实现了比特币系统的所有方面，包括钱包、对整个交易账簿（区块链）完整拷贝的交易确认引擎，和点对点比特币网络中的一个完整网络节点。

详情可以参考[比特币客户端](https://www.8btc.com/books/261/master_bitcoin/_book/3/3.html)