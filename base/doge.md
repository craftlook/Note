#### ES

**order主集群**

* master gw d 三个节点公用物理机
* 内存 32 * 8G
* 磁盘 300GB SAS HDD * 2 + 960GB SATA SSD * 16
* CPU  物理核心 16C * 2 逻辑核心 32 * 2
* 网卡 10G * 2

**order归档** 与主集群公用物理机

* master gw d 三个节点公用物理机
* 内存 32 * 8G
* 磁盘 300GB SAS HDD * 2 + 960GB SATA SSD * 16
* CPU  物理核心 16C * 2 逻辑核心 32 * 2

**order备集群**

* docker
* 16C 32G
* 磁盘 300GB SAS HDD * 2 + 960GB SATA SSD * 16

#### 数据库

**主订单物理机**

* 内存 8 * 8G
* 磁盘 120GB SATA SSD * 2 + 800GB SATA SSD * 6
* CPU  物理核心 8C * 2 逻辑核心 16C * 2
* 网卡 10G * 2
