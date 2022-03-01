# 共识性算法-paxos

[toc]

## Basic Paxos

### 角色

Paxos 算法中分布式系统角色：

- **提议者（Proposer）**：发出提案（Proposal）。Proposal 信息包括提案编号 (Proposal ID) 和提议的值 (Value)。
- **决策者（Acceptor）**：对每个 Proposal 进行投票，若 Proposal 获得多数 Acceptor 的接受，则称该 Proposal 被批准。
- **学习者（Learner）**：不参与决策，从 Proposers/Acceptors 学习、记录最新达成共识的提案（Value）

### 算法流程

Paxos 算法的分布式系统里的，所有的节点都是平等的，它们都可以承担以上某一种或者多种的角色，不过为了便于确保有明确的多数派，**决策节点的数量应该被设定为奇数个**。在系统初始化时，网络中每个节点都知道整个网络所有决策节点的数量、地址等信息。

Paxos 算法包括两个阶段（Learn 阶段之前决议已经形成）：

* 第一阶段：(准备)Prepare 阶段。（提议者）Proposer 向（决策者）Acceptors 发出 Prepare 请求，Acceptors 针对收到的 Prepare 请求进行 Promise 承诺。

  提议节点的 Prepare 请求中会附带一个全局唯一的数字作为（提案）Proposal ID，决策节点收到后，将会给予提案节点两个承诺与一个应答。

  Promise 承诺包含**两个承诺，一个应答**：

  * 两个承诺是指：
    - 承诺不会再接受（提案）Proposal ID 小于或等于当前的 Prepare 请求。
    - 承诺不会再接受（提案）Proposal ID 小于 当前请求的 （提议）Propose 请求。
  * 应答是指：
    * 不违背以前作出的承诺的前提下，回复已经（批准）Accept过的提案中 Proposal ID 最大的那个提案的 Value 和 Proposal ID，如果该值从来没有被任何提案设定过，则返回空值。如果违反此前做出的承诺，即收到的提案Proposal ID 并不是（决策节点）Acceptor收到过的最大的，那允许直接对此 Prepare 请求不予理会。

* 第二阶段:  (批准) Accept批准阶段。（提议节点）Proposer 收到多数 （决策节点）Acceptors 承诺的 Promise 后，向 Acceptors 发出 （提议）Propose 请求，Acceptors 针对收到的 Propose 请求进行 （批准）Accept 处理。

  当提案节点收到了多数派决策节点的应答（称为 Promise 应答）后，可以开始第二阶段“批准”（Accept）过程，这时有如下两种可能的结果：

  * 如果提案节点发现所有响应的决策节点此前都没有批准过该值（即为空），那说明它是第一个设置值的节点，可以随意地决定要设定的值，将自己选定的值与提案 ID，构成一个二元组“(id, value)”，再次广播给全部的决策节点（称为 Accept 请求）
  * 如果提案节点发现响应的决策节点中，已经有至少一个节点的应答中包含有值了，那它就不能够随意取值了，必须无条件地从**应答中找出提案 ID 最大的那个值并接受**，构成一个二元组“(id, maxAcceptValue)”，再次广播给全部的决策节点（称为 Accept 请求）

* 第三阶段：Learn 阶段。Proposer 在收到多数 Acceptors 的 Accept 之后，标志着本次 Accept 成功，决议形成，将形成的决议发送给所有 Learners。

<img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxosq.png" width="80%" heigth="80%" >

### 工作实例

假设一个分布式系统有五个节点，分别命名为 S1、S2、S3、S4、S5，这个例子中只讨论正常通信的场景，不涉及网络分区。全部节点都同时扮演着提案节点和决策节点的身份。此时，有两个并发的请求分别希望将同一个值分别设定为 X（由 S1作为提案节点提出）和 Y（由 S5作为提案节点提出），以 P 代表准备阶段，以 A 代表批准阶段，这时候可能发生以下情况：

* 情况一：S1选定的(提案 ID) P 是 3.1（全局唯一 ID 加上节点编号），先取得了多数派决策节点的 Promise 和 Accepted 应答，此时 S5选定(提案 ID) P 是 4.5，发起 Prepare 请求，收到的多数派应答中至少会包含 1 个此前应答过 S1的决策节点，假设是 S3，那么 S3提供的 Promise 中必将包含 S1已设定好的值 X，S5就必须无条件地用 X 代替 Y 作为自己提案的值，由此整个系统对“取值为 X”这个事实达成一致。

  <img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxos1.png" heigth="70%" width="70%"/>

  <center>整个系统对“取值为 X”达成一致</center>

* 情况二：对于情况一，X 被选定为最终值是必然结果，但从上图中可以看出，X 被选定为最终值并不是必定需要多数派的共同批准，只取决于 S5提案时 Promise 应答中是否已包含了批准过 X 的决策节点，譬如图 所示，S5发起提案的 Prepare 请求时，X 并未获得多数派批准，但由于 S3已经批准的关系，最终共识的结果仍然是 X。

  <img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxos2.png" heigth="70%" width="70%"/>

  <center>X 被选定只取决于 Promise 应答中是否已批准</center>

* 情况三：另外一种可能的结果是 S5提案时 Promise 应答中并未包含批准过 X 的决策节点，譬如应答 S5提案时，节点 S1已经批准了 X，节点 S2、S3未批准但返回了 Promise 应答，此时 S5以更大的提案 ID 获得了 S3、S4、S5的 Promise，这三个节点均未批准过任何值，那么 S3将不会再接收来自 S1的 Accept 请求，因为它的提案 ID 已经不是最大的了，这三个节点将批准 Y 的取值，整个系统最终会对“取值为 Y”达成一致，如图所示

  <img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxos3.png" heigth="70%" width="70%"/>

  <center>整个系统最终会对“取值为 Y”达成一致</center>

* 情况四：从情况三可以推导出另一种极端的情况，如果两个提案节点交替使用更大的提案 ID 使得准备阶段成功，但是批准阶段失败的话，这个过程理论上可以无限持续下去，形成活锁（Live Lock），如图所示。在算法实现中会引入随机超时时间来避免活锁的产生。

  <img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxos4.png" heigth="70%" width="70%"/>

  <center>批准阶段失败，形成活锁</center>

**总结**：Basic Paxos 只能对单个值形成决议，并且决议的形成至少需要两次网络请求和应答（准备和批准阶段各一次），高并发情况下将产生较大的网络开销，极端情况下甚至可能形成活锁。Basic Paxos 是一种很学术化但对工业化并不友好的算法，现在几乎只用来做理论研究。实际的应用都是基于**Multi Paxos 和 Fast Paxos ** 算法的。

## Multi Paxos



## Paxos 推导过程



## 协助记忆总结
