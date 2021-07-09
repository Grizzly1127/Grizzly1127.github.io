---
title: 挖矿与矿池
date: 2021-07-08 12:47:30
tags:
- blockchain
categories:
- blockchain
---



## 1. 挖矿

### 1.1 什么是挖矿

挖矿的本质就是构造符合规则（PoW共识机制）的区块并进行全网验证的过程。作为激励，矿工成功挖掘到区块后，可以从中获取区块奖励和交易费奖励。

下面以BTC为例，阐述挖矿的运作流程。

<!-- more -->

### 1.2 BTC如何挖矿？

![block_package.png](https://i.loli.net/2021/07/08/lgS1IYWGuPKbrTN.png)

上面这张图描述了如何构造一个比特币区块。区块构造流程：

1. 从未确定交易池中选取交易，通常尽可能多的优先选择手续费高的交易。
2. 构造coinbase交易，计算打包交易中的挖矿手续费和coinbase奖励。
3. 矿工将所有交易（包含coinbase交易）添加到区块体中。
4. 对所有交易（包含coinbase交易）进行hash计算，并构造出merkle树，得到merkle树根哈希值。
5. 根据当前环境填充区块头中的当前版本、父区块哈希、时间戳和难度，自行填写nNonce，与得到的merkle树根哈希值一同构成区块头。

完成以上步骤，一个比特币区块就构建好了。

### 1.3 BTC挖矿过程

![block_verify.png](https://i.loli.net/2021/07/08/Silnx2oewNPvURD.png)

下面我们来看一下比特币的挖矿验证过程：

1. 区块打包成功后，对区块中的区块头进行hash256运算并得到结果。
2. 将区块头哈希值与当前target（由nBits解压得来的）比对，这时比对结果会有两种情况：

   - 若区块头哈希值大于target，则表明不符合规则，需要重新构造区块头继续循环验证。
   - 若区块头哈希值小于等于target，则全网节点广播验证，节点验证成功后，成功加入到区块链中。

以上就是比特币挖矿的全过程。
在区块头中，当前版本、父区块哈希、难度是固定不可改变的，那么想要改变区块头哈希值，需要调整nNonce、nTime和coinbase交易，而交易merkle树根的哈希值会随着coinbase交易的改变而改变。

### 1.4 BTC的coinbase结构

我们来看一下coinbase交易的结构，在下面的表格中可以看到在coinbase交易中，唯一能改变的就是coinbase data这个字段。所以该字段数据可由矿工自定义，用于增加区块头哈希的搜索空间。

| Size              | Field              | Description                        |
| ----------------- | ------------------ | ---------------------------------- |
| 32 bytes          | Transaction Hash   | 32字节全部为0                      |
| 4 bytes           | Output Index       | 固定值：0xFFFFFFFF                 |
| 1-9 bytes(VarInt) | Coinbase Data size | coinbase数据的长度                 |
| Variable          | Coinbase Data      | 数据由矿工自定义，用于增加搜索空间 |
| 4 bytes           | Sequence Number    | 固定值：0xFFFFFFFF                 |

### 1.5 BTC中目标、难度概念理解

我们前面说过，比特币挖矿需要`区块头的哈希值`与`当前的目标`进行对比。所以我们需要理解一下什么是目标、什么是难度。

**目标（target）**：矿工计算的区块HASH值需要小于等于目标值才能有效出块。
**难度（difficulty）**：难度是对挖矿困难程度的度量，即指：计算符合给定目标的一个Hash值的困难程度。
**nBits**：目标值被压缩在区块头的nBits字段中，nBits字段以十六进制表示，总共有4个字节，前1个字节为`指数（exponent）`，后3个字节为`系数（coefficient）`。

目标可由nBits计算得到，目标计算公式如下：

> target = coefficient * 256^(exponent - 0x03)

而难度由目标计算得到，难度计算公式如下：

> difficulty = 创世区块target / 当前区块target

从比特币区块浏览器中可以查到，创世区块的nBits为`0x1D00FFFF`，其中指数为`0x1D`，系数为`0x00FFFF`，将其代入到目标计算公式中，可得创世区块的目标值：

> target = 0x00FFFF * 256^(0x1D - 0x03)
>
> target = 0x00000000FFFF0000000000000000000000000000000000000000000000000000

我们以高度为`686,580`这个高度区块为例子：

![block_case.png](https://i.loli.net/2021/07/08/hsrc1zMva2n9tkI.png)

通过公式，可以计算出：

> current_target = 0x0d5f7b * 256^(0x17 - 0x03)
>
> current_target = 0x0000000000000000000d5f7b0000000000000000000000000000000000000000

得到了当前的目标值与创世区块目标值，通过难度公式，可计算出高度`686,580`的难度为：

> difficulty = init_target / current_target = 21,047,730,572,451 ≈ 21.05T

### 1.6 能不能一次挖多个币呢？

**答案是肯定的。**

当前存在一种机制，可以使得矿工在不影响挖比特币的同时，还能顺带挖另外几种虚拟币，以增加额外的收入，该机制就是联合挖矿。

## 2. 联合挖矿

### 2.1 ViaBTC矿池联合挖矿赠币规则

目前许多主流的矿池都上线了挖矿赠币业务，例如在我们的矿池服务中，挖BTC可以额外获得NMC、EMC、SYS和ELA的赠币，挖LTC可以额外获得DOGE的赠币。

![merge_mining_rule.png](https://i.loli.net/2021/07/08/qQo8PCYU3BSwexy.png)

下面我们来看一下联合挖矿的实现原理。

### 2.2 联合挖矿实现

1. 通过 rpc 调用辅链节点的 `getauxblock` 或者 `createauxblock` 方法，创建新块。返回结果如下：

   ```json
   {
     "hash"                (string) hash of the created block
     "chainid"             (numeric) chain ID for this block
     "previousblockhash"   (string) hash of the previous block
     "coinbasevalue"       (numeric) value of the block's coinbase
     "bits"                (string) compressed target of the block
     "height"              (numeric) height of the block
     "_target"             (string) target in reversed byte order, deprecated
   }
   ```

2. 在Bitcoin的coinbase交易中，coinbase data字段可以写入任意自定义数据，那么通过写入规定格式的数据。我们在coinbase的coinbase data中添加以下信息：

   ![coinbase_structure.png](https://i.loli.net/2021/07/08/mV6sGop1d4iY2nQ.png)

   当有多条辅链时，需要用多条辅链的block_hash来构建出merkle树。那么就需要考虑辅链在merkle树底部的插槽位置。使用以下算法将chain ID转换为辅链的block_hash所在的merkle树的插槽位置：

   ```c
   static int cal_merkle_index(int chain_id, int nonce, int size)
   {
       unsigned int rand = nonce;
       rand = rand * 1103515245 + 12345;
       rand += chain_id;
       rand = rand * 1103515245 + 12345;
       return rand % size;
   }
   ```

3. 矿工照常以挖比特币的方式挖矿，但有一点不同的是，需要判断以下三种情况：

   - btc_hash_value <= btc_target：矿工广播区块，矿工可以得到BTC的挖矿奖励和NMC辅链的挖矿奖励。
   - btc_target < btc_hash_value <= aux_target：矿工广播区块，矿工不能获取BTC的挖矿奖励，但是能获取辅链的挖矿奖励。
   - btc_hash_value > aux_target：矿工不会广播。

4. 当矿工挖出区块后，通过 rpc 调用辅链节点的 `submitauxblock` 方法提交区块验证，参数与结果如下：

   ```
   Arguments:
   1. hash       (string, required) Hash of the block to submit
   2. auxpow     (string, required) Serialised auxpow found

   Result:
   xxxxx         (boolean) whether the submitted block was correct
   ```

   其中，auxpow包含以下内容：

| Size     | Field             | Description          |
| -------- | ----------------- | -------------------- |
| Variable | coinbase          | 主链的coinbase信息   |
| 32 bytes | block_hash        | 主链区块hash值       |
| Variable | coinbase_branch   | 主链的交易merkle分支 |
| Variable | blockchain_branch | 多个辅链的merkle分支 |
| 80 bytes | parent_block      | 主链区块头信息       |

### 2.3 个人（solo）挖矿的劣势

![solo_mining.png](https://i.loli.net/2021/07/08/YrjJdNhtVQcRFn2.png)

随着挖矿的人越来越多，挖矿设备也从cpu到显卡gpu再到专业矿机，算力越来越大，独立矿工挖到区块的概率越来越小，收益也越来越低，如果继续投入资金购买电脑，显卡、矿机等硬件设备，会大大提高成本，风险也越来越大，所以很多矿工不会选择 Solo 挖矿这种投资方式。

劣势：

1. 硬盘和带宽要求大
2. 挖矿成功概率极低

于是就有人提出把大家的算力集中在一起挖矿，算力大了，挖到区块的概率就会大大提高，然后再根据每个参与的矿工所占算力配额来进行奖励分配。
使用这种方法建立的特殊节点，就是矿池。

## 3 矿池

### 3.1 简介

我们来看一下这张图，左边是区块链网络，上面这个是solo矿工，相比于全网算力来说自身的算力非常小，挖矿成功的概率也极小，下面的是矿池。

![pool.png](https://i.loli.net/2021/07/08/dyp6fKmPYNXLSCu.png)

矿池是矿工的**集合地**，任何矿工都可以加入，无论个体还是组织，无论专业还是业余，都能为矿池提供自己的一份算力。加入到矿池的矿工挖到矿后，获得的奖励会被分配到矿池，然后矿池再根据预先设定的分红规则并结合各个矿工的算力进行奖励发放。
加入到矿池的矿工，只需要做一件事，那就是不停的计算矿池下发的任务，算出符合矿池难度的区块就进行提交。
矿池的工作就是需要对矿工进行管理，统计矿工算力和贡献，挖矿任务管理等。
矿池与矿工之间使用的是stratum矿池协议进行交互的，这是很重要的交互协议。

### 3.2 stratum矿池协议

stratum矿池协议有很多方法，最重要的就是以下几种方法：

- **订阅消息（mining.subscribe）**

  矿机想参与挖矿，需要主动连接到矿池，向矿池申请加入挖矿，一般会附带自己挖矿软件的版本。

  ```json
  矿机消息：
  {"id":1,"method":"mining.subscribe","params":[version]}\n
  矿池响应消息：
  {"id":1,"result":[[["mining.set_difficulty","1"],["mining.notify","1"]],"{extraNonce1}",{extraNonce2_len}],“error”:null}\n
  ```

  **extraNonce1**：该字段由矿池为矿工分配，为了确保每个矿工不会做重复工作。
  **extraNonce2_len**：设定extraNonce2字段的长度，一般为4或8个字节，矿工挖矿时，通过该字段增加搜索空间。
  现在，我们至少有了8个字节的搜索空间，即`nNonce`的4个字节，以及`extraNonce2`的4个字节。

- **授权消息（mining.authorize）**

  矿机使用帐号和密码登录到矿池，矿池返回true则登录成功。该方法必须是在初始化连接之后马上进行，否则矿机得不到矿池任务。

  ```json
  矿机消息：
  {"id":2,"method":"mining.authorize","params":["{account}","{password}"]}\n
  矿池响应消息：
  {"id":2,"result":true,"error":null}\n
  ```

- **下发难度（mining.set_difficulty）**

  矿机被授权后，那么矿池会下发任务难度给矿机，后面矿机的挖矿，必须达到任务难度要后，矿池才登记贡献。

  ```json
  矿池发送消息：
  {"id":null,"method":"mining.set_difficulty","params":[{难度值}]}\n
  ```

- **下发任务（mining.notify）**

  矿池会不断向矿机下发任务，矿机接到任务后，根据任务的标示，可以继续挖上一个任务，还是立即开始新的任务挖矿。

  ```json
  矿池发送消息：
  {"id":null,"method":"mining.notify","params":["{jobId}","{prev hash}","{coinbase1}","{coinbase2}",[{区块交易merkle_branch}],"{version}","{nBits}","{nTime}",{是否立即更换任务}]}\n
  ```

- **提交消息（mining.submit）**

  矿机根据挖矿任务开始挖矿，直到发现了满足矿池难度的区块后，将构造区块头必要的信息提交给矿池验证并广播。

  ```json
  矿机消息：
  {"id":3,"method":"mining.submit","params":["{account}","{jobId}","{extraNonce2}","{time}","{nonce}"]}\n
  ```

### 3.3 如何增加矿工的搜索空间

如果仅仅给矿工可以改变nNonce（4字节，搜索空间4294967296大约43亿）和nTime（可变动范围很小）字段，则交互的数据量很少，但这点搜索空间肯定是不够的。
想要增加搜索，只能从`hashMerkleRoot`下功夫，如果让矿工自己构造coinbase，进而影响hashMerkleRoot值，那么搜索空间的问题将迎刃而解。
而构造Merkle树并不需要所有的交易数据，只需要某几个节点上的hash值，即可计算出修改coinbase后的hashMerkleRoot值。如下图：

![merkle_branch.png](https://i.loli.net/2021/07/08/7mdoVCrY1W8xTOk.png)

我们将交易列表顺序固定下来后，只需要将merkle树中的一小部分节点的哈希值发给矿工，矿工就可以根据这些节点进行计算得到merkle树根。

### 3.4 矿工的挖矿步骤

1. 构造coinbase交易

   > coinbase = coinbase1 + extraNonce1 + extraNonce2 + coinbase2

   对于矿工来说，`coinbase1`、`extraNonce1`、`coinbase2`都是固定的，这是由矿池来填写，矿工只关心 `extraNonce2` 的长度。矿工生成指定长度的`extraNonce2`来增加搜索空间。

2. 构建hashMerkleRoot

   利用coinbase和merkle_branch，生成hashMerkleRoot。

3. 构建区块头

   填充余下的5个字段，现在，矿工可以在 nTime、nNonce 和 extraNonce2 里搜索进行挖矿，如果嫌搜索空间还不够，只要增加Extranonce2_size为多几个字节就可轻而易举解决。

## 4. ViaBTC矿池实现

项目路径：https://github.com/viabtc/viabtc_mining_server

### 4.1 ViaBTC矿池架构图

![mining_pool_arch.png](https://i.loli.net/2021/07/08/cawiHMVkBtgWI4N.png)

这张图是viabtc矿池的server层架构图，接下来我将讲解每个服务的作用与功能。

### 4.2 bitpeer

![bitpeer.png](https://i.loli.net/2021/07/08/AyrhiqYwXGoP15k.png)

- bitpeer实现了btc的p2p网络协议，可以把它当成是一个简化版的BTC节点，实现了p2p的部分功能。
- 部署多个bitpeer，可加速区块在btc网络中的广播。
- 当区块高度更新时，需要通知jobmaster放弃旧块的挖掘转而开始新块任务

### 4.3 blockmaster

![blockmaster.png](https://i.loli.net/2021/07/08/cnFzt4Khwe8HU7g.png)

blockmaster的主要功能有：

1. 将矿工挖掘到的区块发送给bitpeer进行广播。

2. 为了更快的进行区块广播，blockmaster将区块处理成瘦块广播。

**瘦块原理：**

为了更快的进行区块广播，blockmaster将区块处理成瘦块，原理是剔除掉区块交易（不包括coinbase交易）中除了交易哈希值之外的交易信息，因为交易详细信息可通过交易哈希值向节点查询，所以大大减少了区块的大小，瘦块在网络上传输将会加快，将瘦块发送给其他地区的blockmaster组装成完整的区块进行广播。

![thin_block.png](https://i.loli.net/2021/07/08/cxTVYO489NPHmnF.png)

`bitpeer` 和 `blockmaster` 这两个服务的主要功能是为了加速区块的广播，毕竟比别人先一步将区块进行广播到全网验证，就能抢先一步将区块加到链中，获得奖励。

### 4.3 jobmaster

主要功能：

1. 构造区块信息给gateway发送挖矿任务。

2. 接收gateway提交的区块，并进行广播。

### 4.4 gateway

主要功能：

1. 管理矿工、矿场管理员。
2. 矿工难度值动态调整。
3. 实现stratum矿池协议与矿工交互。
4. 测算各个矿工的算力。

5. 验证矿工提交的share，并进行验证统计。

### 4.5 poolbench

监控其他矿池的状态，如果其他矿池的区块高度有更新，则通知jobmaster开始挖新块。

### 4.6 mineragent

1. gateway代理，从gateway接收挖块任务。
2. 实现stratum协议，为矿工分配任务。

3. 主要用于矿机数量庞大的矿场，部署在矿场中，可以有效节省带宽，提升性能。

### 4.7 算力统计

为统计矿工的算力贡献，矿池会为每个矿工设置难度值，当矿工挖掘的区块达到难度值要求时，就提交一次share，矿池可通过矿工难度测算出矿工的算力。
矿池要求矿工大约每4秒提交一次share，那么，如何控制矿工难度来达到该要求呢？

1. 新的矿工连接到矿池，矿池会为矿工分配一个默认难度值（1024）。
2. 矿池给矿工下发任务和该难度值，矿工开始挖矿。
3. 矿工挖出区块达到该难度值的要求后，提交share给矿池。
4. 矿池收到后，判断矿工此次提交的share是否有效，有效则记录此次提交的时间间隔，计算此次矿工的工作量与挖矿难度的比例（goal），矿池计算矿工多次提交share的平均时间（avg）后，进行如下判断：
   - 如果avg大于4s，并且（avg / 4s）大于1.5，则矿工难度值需要**减半**。
   - 如果avg小于4s，并且（avg / 4s）小于0.7，则矿工难度值需要**增加一倍**。
5. 每次有效提交share，矿池会将矿工的`难度值`、`有效提交share次数`和 `goal` 收集起来，每隔60秒将数据提交给metawriter，metawriter会将数据记录到redis，用于计算矿工的收益。

### 4.8 收益结算

矿池常用的分红规则如下：

- **PPS（pay per share）**

  矿池对矿工提交的每一个工作量证明 (share) 按照理论收益付费，相当于矿工给矿池打工，根据提供的算力稳定获取收益。 由于矿工不承担风险，矿池承担了运气风险和孤块风险，所以收取相对较高的费用。

- **PPLNS（Pay Per Last N Shares）**

  矿池每发现有效的区块， 根据过去 N 个难度周期中用户算力占矿池算力的比例进行分配，矿工费也参与分配。 矿池收取相对较少的费用，用于矿池的运营和维护。 这种方式下矿工的收益和矿池的出块相关，矿工收益不稳定，但长期平均收益更高。

- **PPS+（Pay Per Share Plus）**

  对传统 PPS 结算方式的一种改进，在传统的 PPS 结算方式基础上，增加了矿工费的分配。 在这种方式下，矿池对矿工提交的每一个工作量证明 (share) 按照理论收益付费，相当于矿工给矿池打工，根据提供的算力稳定获取收益。 由于矿工不承担风险，矿池承担了运气风险和孤块风险，所以收取相对较高的费用。 矿工费另外通过下述的 PPLNS 方式分配给矿工。

- **SOLO**

  全部收益分配给挖出该块的矿工，其他矿工不参与分配，矿池收取极少手续费，用于矿池运营和维护。

ViaBTC矿池支持三种结算方式：PPS+、PPLNS和SOLO。默认为PPS+结算方式。

**PPS+模式：**

| 挖矿收益 | 块奖励                                  | 交易费（不固定）                                             |
| -------- | --------------------------------------- | ------------------------------------------------------------ |
| 结算方式 | PPS                                     | PPLNS                                                        |
| 费率     | 4%                                      | 2%                                                           |
| 计算公式 | 工作量 / 挖矿难度 * 块奖励 * (1 - 费率) | 用户算力 / 矿池算力 * 块收益 * (1 - 费率)                    |
| 分配规则 | 每个小时根据当前难度结算一次            | 区块得到6次确认后，根据过去5个难度周期内用户算力占矿池算力的比例进行分配 |

**PPLNS 模式：**

| 挖矿收益 | 块奖励 + 交易费（不固定）                                    |
| -------- | ------------------------------------------------------------ |
| 费率     | 2%                                                           |
| 计算方式 | 用户算力／矿池算力 * 挖矿收益 *（1 - 费率）                  |
| 分配规则 | 区块得到6次确认后，根据过去5个难度周期内用户算力占矿池算力的比例进行分配 |

**SOLO 模式：**

| 挖矿收益 | 块奖励 + 交易费（不固定） |
| -------- | ------------------------- |
| 费率     | 1%                        |

**联合挖矿：**

| 币   | 规则                              |
| ---- | --------------------------------- |
| BTC  | 每挖 1 个 BTC 赠送 1 个 NMC       |
| BTC  | 每挖 1 个 BTC 赠送 5 个 SYS       |
| BTC  | 每挖 1 个 BTC 赠送 0.1 个 EMC     |
| BTC  | 每挖 1 个 BTC 赠送 1 个 ELA       |
| BCH  | 每挖 1 个 BCH 赠送 1 个 SYS       |
| LTC  | 挖 LTC 送 DOGE，按 PPLNS 模式结算 |

