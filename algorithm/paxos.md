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

<div align="center"><img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxosq.png" width="80%" heigth="80%" ></div>

### 工作实例

假设一个分布式系统有五个节点，分别命名为 S1、S2、S3、S4、S5，这个例子中只讨论正常通信的场景，不涉及网络分区。全部节点都同时扮演着提案节点和决策节点的身份。此时，有两个并发的请求分别希望将同一个值分别设定为 X（由 S1作为提案节点提出）和 Y（由 S5作为提案节点提出），以 P 代表准备阶段，以 A 代表批准阶段，这时候可能发生以下情况：

* 情况一：S1选定的(提案 ID) P 是 3.1（全局唯一 ID 加上节点编号），先取得了多数派决策节点的 Promise 和 Accepted 应答，此时 S5选定(提案 ID) P 是 4.5，发起 Prepare 请求，收到的多数派应答中至少会包含 1 个此前应答过 S1的决策节点，假设是 S3，那么 S3提供的 Promise 中必将包含 S1已设定好的值 X，S5就必须无条件地用 X 代替 Y 作为自己提案的值，由此整个系统对“取值为 X”这个事实达成一致。

  <div align="center"><img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxos1.png" heigth="70%" width="70%"/></div>
  <div align="center">整个系统对“取值为 X”达成一致</div>

  

* 情况二：对于情况一，X 被选定为最终值是必然结果，但从上图中可以看出，X 被选定为最终值并不是必定需要多数派的共同批准，只取决于 S5提案时 Promise 应答中是否已包含了批准过 X 的决策节点，譬如图 所示，S5发起提案的 Prepare 请求时，X 并未获得多数派批准，但由于 S3已经批准的关系，最终共识的结果仍然是 X。

  <div align="center"><img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxos2.png" heigth="70%" width="70%"/></div>

  <div align="center">X 被选定只取决于 Promise 应答中是否已批准</center></div>

* 情况三：另外一种可能的结果是 S5提案时 Promise 应答中并未包含批准过 X 的决策节点，譬如应答 S5提案时，节点 S1已经批准了 X，节点 S2、S3未批准但返回了 Promise 应答，此时 S5以更大的提案 ID 获得了 S3、S4、S5的 Promise，这三个节点均未批准过任何值，那么 S3将不会再接收来自 S1的 Accept 请求，因为它的提案 ID 已经不是最大的了，这三个节点将批准 Y 的取值，整个系统最终会对“取值为 Y”达成一致，如图所示

  <div align="center"><img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxos3.png" heigth="70%" width="70%"/></div>

  <div align="center">整个系统最终会对“取值为 Y”达成一致</div>

* 情况四：从情况三可以推导出另一种极端的情况，如果两个提案节点交替使用更大的提案 ID 使得准备阶段成功，但是批准阶段失败的话，这个过程理论上可以无限持续下去，形成活锁（Live Lock），如图所示。在算法实现中会引入随机超时时间来避免活锁的产生。

  <div align="center"><img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxos4.png" heigth="70%" width="70%"/></div>

  <div align="center">批准阶段失败，形成活锁</div>

**总结**：Basic Paxos 只能对单个值形成决议，并且决议的形成至少需要两次网络请求和应答（准备和批准阶段各一次），高并发情况下将产生较大的网络开销，极端情况下甚至可能形成活锁。Basic Paxos 是一种很学术化但对工业化并不友好的算法，现在几乎只用来做理论研究。实际的应用都是基于**Multi Paxos 和 Fast Paxos ** 算法的。

## Multi Paxos

### Basic Paxos 的问题

Basic Paxos 有以下问题，导致它不能应用于实际：

- **Basic Paxos 算法只能对一个值形成决议**。
- **Basic Paxos 算法会消耗大量网络带宽**。Basic Paxos 中，决议的形成至少需要两次网络通信，在高并发情况下可能需要更多的网络通信，极端情况下甚至可能形成活锁。如果想连续确定多个值，Basic Paxos 搞不定了。

### Multi Paxos 的改进

Multi Paxos 正是为解决以上问题而提出。Multi Paxos 基于 Basic Paxos 做了两点改进：

- 针对每一个要确定的值，运行一次 Paxos 算法实例（Instance），形成决议。每一个 Paxos 实例使用唯一的 Instance ID 标识。
- 在所有 Proposer 中选举一个 Leader，由 Leader 唯一地提交 Proposal 给 Acceptor 进行表决。这样没有 Proposer 竞争，解决了活锁问题。在系统中仅有一个 Leader 进行 Value 提交的情况下，Prepare 阶段就可以跳过，从而将两阶段变为一阶段，提高效率。

Multi Paxos 首先需要选举 Leader，Leader 的确定也是一次决议的形成，所以可执行一次 Basic Paxos 实例来选举出一个 Leader。选出 Leader 之后只能由 Leader 提交 Proposal，在 Leader 宕机之后服务临时不可用，需要重新选举 Leader 继续服务。在系统中仅有一个 Leader 进行 Proposal 提交的情况下，Prepare 阶段可以跳过。

Multi Paxos 通过改变 Prepare 阶段的作用范围至后面 Leader 提交的所有实例，从而使得 Leader 的连续提交只需要执行一次 Prepare 阶段，后续只需要执行 Accept 阶段，将两阶段变为一阶段，提高了效率。为了区分连续提交的多个实例，每个实例使用一个 Instance ID 标识，Instance ID 由 Leader 本地递增生成即可。

Multi Paxos 允许有多个自认为是 Leader 的节点并发提交 Proposal 而不影响其安全性，这样的场景即退化为 Basic Paxos。

Chubby 和 Boxwood 均使用 Multi Paxos。ZooKeeper 使用的 Zab 也是 Multi Paxos 的变形。

## Paxos 推导过程

### 最简单的方案——只有一个Acceptor

假设只有一个Acceptor（可以有多个Proposer），只要Acceptor接受它收到的第一个提案，则该提案被选定，该提案里的value就是被选定的value。这样就保证只有一个value会被选定。

但是，如果这个唯一的Acceptor宕机了，那么整个系统就**无法工作**了！

因此，必须要有**多个Acceptor**！
<div align="center"><img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxost1.png" heigth="70%" width="70%"/></div>

### 多个Acceptor

多个Acceptor的情况如下图。那么，如何保证在多个Proposer和多个Acceptor的情况下选定一个value呢？

<div align="center"><img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxost2.png" heigth="70%" width="70%"/></div>
下面开始寻找解决方案。

如果我们希望即使只有一个Proposer提出了一个value，该value也最终被选定。

那么，就得到下面的约束：

> P1：一个Acceptor必须接受它收到的第一个提案。

但是，这又会引出另一个问题：如果每个Proposer分别提出不同的value，发给不同的Acceptor。根据P1，Acceptor分别接受自己收到的value，就导致不同的value被选定。出现了不一致。如下图：
<div align="center"><img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxost3.png" heigth="70%" width="70%"/></div>
刚刚是因为『一个提案只要被一个Acceptor接受，则该提案的value就被选定了』才导致了出现上面不一致的问题。因此，我们需要加一个规定：

> 规定：一个提案被选定需要被**半数以上**的Acceptor接受

这个规定又暗示了：『一个Acceptor必须能够接受不止一个提案！』不然可能导致最终没有value被选定。比如上图的情况。v1、v2、v3都没有被选定，因为它们都只被一个Acceptor的接受。

最开始讲的『**提案=value**』已经不能满足需求了，于是重新设计提案，给每个提案加上一个提案编号，表示提案被提出的顺序。令『**提案=提案编号+value**』。

虽然允许多个提案被选定，但必须保证所有被选定的提案都具有相同的value值。否则又会出现不一致。

于是有了下面的约束：

> P2：如果某个value为v的提案被选定了，那么每个编号更高的被选定提案的value必须也是v。

一个提案只有被Acceptor接受才可能被选定，因此我们可以把P2约束改写成对Acceptor接受的提案的约束P2a。

> P2a：如果某个value为v的提案被选定了，那么每个编号更高的被Acceptor接受的提案的value必须也是v。

只要满足了P2a，就能满足P2。

但是，考虑如下的情况：假设总的有5个Acceptor。Proposer2提出[M1,V1]的提案，Acceptor25（半数以上）均接受了该提案，于是对于Acceptor25和Proposer2来讲，它们都认为V1被选定。Acceptor1刚刚从宕机状态恢复过来（之前Acceptor1没有收到过任何提案），此时Proposer1向Acceptor1发送了[M2,V2]的提案（V2≠V1且M2>M1），对于Acceptor1来讲，这是它收到的第一个提案。根据P1（一个Acceptor必须接受它收到的第一个提案。）,Acceptor1必须接受该提案！同时Acceptor1认为V2被选定。这就出现了两个问题：

1. Acceptor1认为V2被选定，Acceptor2~5和Proposer2认为V1被选定。出现了不一致。
2. V1被选定了，但是编号更高的被Acceptor1接受的提案[M2,V2]的value为V2，且V2≠V1。这就跟P2a（如果某个value为v的提案被选定了，那么每个编号更高的被Acceptor接受的提案的value必须也是v）矛盾了。

<div align="center"><img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxost4.png" heigth="70%" width="70%"/></div>

所以我们要对P2a约束进行强化！

P2a是对Acceptor接受的提案约束，但其实提案是Proposer提出来的，所有我们可以对Proposer提出的提案进行约束。得到P2b：

P2b：如果某个value为v的提案被选定了，那么之后任何Proposer提出的编号更高的提案的value必须也是v。

由P2b可以推出P2a进而推出P2。

那么，如何确保在某个value为v的提案被选定后，Proposer提出的编号更高的提案的value都是v呢？

只要满足P2c即可：

P2c：对于任意的N和V，如果提案[N, V]被提出，那么存在一个半数以上的Acceptor组成的集合S，满足以下两个条件中的任意一个：

S中每个Acceptor都没有接受过编号小于N的提案。
S中Acceptor接受过的最大编号的提案的value为V。
Proposer生成提案
为了满足P2b，这里有个比较重要的思想：Proposer生成提案之前，应该先去『学习』已经被选定或者可能被选定的value，然后以该value作为自己提出的提案的value。如果没有value被选定，Proposer才可以自己决定value的值。这样才能达成一致。这个学习的阶段是通过一个『Prepare请求』实现的。

于是我们得到了如下的提案生成算法：

Proposer选择一个新的提案编号N，然后向某个Acceptor集合（半数以上）发送请求，要求该集合中的每个Acceptor做出如下响应（response）。
(a) 向Proposer承诺保证不再接受任何编号小于N的提案。
(b) 如果Acceptor已经接受过提案，那么就向Proposer响应已经接受过的编号小于N的最大编号的提案。

我们将该请求称为编号为N的Prepare请求。

如果Proposer收到了半数以上的Acceptor的响应，那么它就可以生成编号为N，Value为V的提案[N,V]。这里的V是所有的响应中编号最大的提案的Value。如果所有的响应中都没有提案，那 么此时V就可以由Proposer自己选择。
生成提案后，Proposer将该提案发送给半数以上的Acceptor集合，并期望这些Acceptor能接受该提案。我们称该请求为Accept请求。（注意：此时接受Accept请求的Acceptor集合不一定是之前响应Prepare请求的Acceptor集合）

Acceptor接受提案
Acceptor可以忽略任何请求（包括Prepare请求和Accept请求）而不用担心破坏算法的安全性。因此，我们这里要讨论的是什么时候Acceptor可以响应一个请求。

我们对Acceptor接受提案给出如下约束：

P1a：一个Acceptor只要尚未响应过任何编号大于N的Prepare请求，那么他就可以接受这个编号为N的提案。

如果Acceptor收到一个编号为N的Prepare请求，在此之前它已经响应过编号大于N的Prepare请求。根据P1a，该Acceptor不可能接受编号为N的提案。因此，该Acceptor可以忽略编号为N的Prepare请求。当然，也可以回复一个error，让Proposer尽早知道自己的提案不会被接受。

因此，一个Acceptor只需记住：1. 已接受的编号最大的提案 2. 已响应的请求的最大编号。

<div align="center"><img src="https://github.com/craftlook/Note/blob/master/image/paxos/paxost5.png" heigth="70%" width="70%"/></div>


## 协助记忆总结

