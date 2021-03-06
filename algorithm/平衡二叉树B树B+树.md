# 平衡二叉树、B树、B+树

#### 目录

* [二叉树](#2tree)

* [平衡二叉树](#avg-tree)
* [B-tree](#b-tree)
* [B+-tree](#b+-tree)

## <span id="2Tree">二叉查找树</span>(BST)

特殊的二叉树，其他称谓排序二叉树、二叉搜索树、二叉排序树。

二叉查找树实际上是数据域有序的二叉树，即树上的每个节点，都满足**左子树**上的所有数据域的值**小于或等于根节点**，右子树上的所有数据域的值均**大于根节点**。

### 查找

对于二叉查找树的查找有递归模式和非递归两种。

* 递归方式：递归边界为树的终止节点；
* 非递归方式：分为 **DFS**（深度优先）和**BFS**（广度优先） 的遍历方式。 <a href="https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/algorithm/dfs-bfs.md">DFS与BFS介绍</a>

**DFS** 根据以结点（Node）的访问顺序，定义了三种不同的搜索策略：（**记忆点：**前、中、后 标达的时结点的访问顺序，其中左子树优先右子树遍历）

* 前序遍历：结点 -> 左子树 -> 右子树 
* 中序遍历：左子树-> 结点 -> 右子树
* 后续遍历：左子树-> 右子树 -> 结点

如图：

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/tree-search.png)

* 前序: F - B - A - D - C - E - G - I - H
* 中序: A - B - C - D - E - F - G - H - I
* 后序: A - C - E - D - B - H - I - G - F

## <span id="avg-tree">平衡二叉树</span>(AVL)

基于二分法的策略调高查询速度的二叉树的数据结构，

#### 主要特点：

* 非叶子节点最多有两个子节点；
* 非叶子节点值大于左边子节点、小于右边子节点；
* 树的左右两边的层级数相差不会大于1；（左树的高度减去右数的高度的差，即平衡因子）
* 没有值相等的重复节点；

AVL树的查询性能与树的层级（树高度 h) 成反比，**h值越小查询越快**。通过保证数据的左右边的节点高度相差不大于1，这样避免树形结构的由于**删除、增加导致树变成线性链表**的结构影响了查询效率，保证数据平衡查询速度近似于二分法查找。（为了保证左右端数据大致平衡，降低二叉树的查询难度，一般采用算法机制实现节点的数据结构平衡，如 <a href="">Treap</a>、<a href="">红黑树</a>）

#### 平衡的调整

分为四种情况：LL、LR、RR、RL

下面我们通过不断插入数据来说明几种不同的旋转方式:

<font color="red">注意：橘黄色的结点为旋转中心，黑色结点的为离插入结点最近的失衡结点。</font>

##### LR型

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/LR.png)

最开始插入数据16，3，7后的结构如上图所示，结点16失去了平衡，3为16的左孩子，7为失衡结点的左孩子的右孩子，所以为LR型，接下来通过两次旋转操作复衡，先通过以3为旋转中心，进行左旋转，结果如图所示，然后再以7为旋转中心进行右旋转，旋转后恢复平衡了。

##### LL型

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/LL.png)

在上面恢复平衡后我们再次插入数据11和9,发现又失去平衡了，这次失衡结点是16，11是其左孩子，9为其失衡结点的左孩子的左孩子，所以是LL型，以失衡结点的左孩子为旋转中心进行一次右旋转即可。

##### RR型

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/RR.png)

进一步插入数据26后又再次失衡了，失衡结点为7,很明显这是RR型，以失衡结点的右孩子为旋转中心左旋转一次即可。

##### RL型

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/RL.png)

再插入18后又再次失衡了，失衡结点为16，26为其右孩子，18为其右孩子的左孩子，为RL型，以失衡结点的右孩子为旋转中心，进行一次右旋转，然后再次已失衡结点的右孩子为旋转中心进行一次左旋转变恢复了平衡。

这就是4中旋转方式，其实只有两种，RR和LL，RL和LR本质上是一样的。下面我们再次插入数据14，15，完成我们最后数据的插入操作：

![avatar](https://github.com/craftlook/Hello-World/blob/craftlook-Hello-World/image/LLRR.png)

又是一次LR型，按前面操作就可以了。

## <span id="b-tree">B-tree</span>



## <span id="b+-tree">B+-tree</span>

