# DFS与BFS介绍

## 对比

**DFS和BFS的总结结论**：

数据结构上

* DFS：用**递归**的形式和**栈**结构，先进后出。
* BFS：选取状态用**队列**的形式，先进先出。

复杂度上： DFS与BFS的复杂度大体一致，不同之处在于遍历的方式和对于问题的解决出发点不同，**DFS适用于目标明确**，**BFS适用于大范围的寻找**。(DFS通常是可以知道我们能不能做到某事，而BFS一般还能知道做到某事的最小路径)

## DFS（深度优先）

**深度优先搜索的步骤：**

1. 递归下去
2. 回溯上来

顾名思义深度优先是以**深度为准则，先一条路走到底，直到达到目标**。这里称之为递归下去。当没有达到目标又无路可走时，进行**回溯退回到上一步的状态，走其他的路**。

​	比如：假如有一个迷宫，其方式是一层门一层门的往里走，目标是最终走出这个迷宫。具体的游戏方式起始房间有9个门，我们可以任选其一进入，下一个房间也会有9个门，直到最后一层会要么有出去的大门，要么是一堵墙。一个个试，一层层往下随机走，直到最后一层如果出去了就出去了，但是如果没出去那么返回上一层。

如图所示，在一个迷宫中，黑色块代表玩家所在位置，红色块代表终点，问是否有一条到终点的路径

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/dfs-1.png)

我们用深度优先搜索的方法去解这道题，由图可知，我们可以走上下左右四个方向，我们规定按照左下右上的方向顺序走，即，如果左边可以走，我们先走左边。然后递归下去，没达到终点，我们再回溯上来，等又回溯到这个位置时，左边已经走过了，那么我们就走下边，按照这样的顺序与方向走。并且我们把走过的路标记一下代表走过，不能再走。

所以我们从黑色起点首先向左走，然后发现还可以向左走，最后我们到达图示位置

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/dfs-2.png)

已经连续向左走到左上角的位置了，这时发现左边不能走了，这时我们就考虑往下走，发现也不能走，同理，上边也不能走，右边已经走过了也不能走，这时候无路可走了，代表这条路是死路不能帮我们解决问题，所以现在要回溯上去，回溯到上一个位置，

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/dfs-3.png)

在这个位置我们由上可知走左边是死路不行，上下是墙壁不能走，而右边又是走过的路，已经被标记过了，不能走。所以只能再度回溯到上一个位置寻找别的出路。

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/dfs-4.png)

最终我们回溯到最初的位置，同理，左边证明是死路不必走了，上和右是墙壁，所以我们走下边。然后递归下去

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/dfs-5.png)

到了这个格子，因为按照左下右上的顺序，我们走左边，递归下去

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/dfs-6.png)

一直递归下去到最左边的格子，然后左边行不通，走下边。然后达到目标。DFS的重要点在于状态回溯。

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/dfs-7.png)

## BFS（广度优先）

