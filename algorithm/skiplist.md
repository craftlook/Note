## 跳表(skiplist)

## 数据结构简介

skiplist 跳表全称跳跃列表，本质上是一种查找结构，允许快速查询。插入和删除一个有序连续元素的数据链表，跳跃列表的平均查找和插入时间复杂度都是O(logn)。**快速查询是通过维护一个多层次的链表，且每一层链表中的元素是前一层链表元素的子集(如下图)**。开始时，算法在最稀疏的层次进行搜索，直至需要查找的元素在该层两个相邻的元素中间。这时，算法将跳转到下一个层次，重复刚才的搜索，直到找到需要查找的元素为止。

![avatar](https://github.com/craftlook/Note/blob/master/image/skiplist1.jpg)

## 演化过程

1. skiplist，首先是一个list，实际上在有序链表的基础上发展出来的，首先看有序链表，如图:

![avatar](https://github.com/craftlook/Note/blob/master/image/skiplist2.png)

上图的有序链表，如果需要查找数据，需要从头开始进行比较直到找到**目标数据节点**或者找到第一个比目标数据大的节点为止**（没找到）**。顺序查找的时间复杂度O(n)。插入也要经历查找过程，从而确定保存位置。

2. 如果我们每相邻的两个节点增加一个指针，让指针指向下下一个接单，如图：

![avatar](https://github.com/craftlook/Note/blob/master/image/skiplist3.png)

新增加的指针形成了新的链表，但只包含7，19，26(如上图)。

3. 现在当想查找数据时，可以先沿着新链表进行查找。当碰到**比目标数据大的节点时，再回到原来的链表进行查找**。如图，我们进行23的查找，图中红线是我们的查找路径：

![avatar](https://github.com/craftlook/Note/blob/master/image/skiplist4.png)

