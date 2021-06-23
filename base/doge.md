#### ES

**order主集群**

* master gw d 三个节点公用物理机 (gw 10个节点 d 50个节点 m 3个节点) 50台物理机
* 内存 32 * 8G
* 磁盘 300GB SAS HDD * 2 + 960GB SATA SSD * 16
* CPU  物理核心 16C * 2 逻辑核心 32 * 2
* 网卡 10G * 2

**order归档** 与主集群公用物理机  (gw 10个节点 d 32个节点 m 3个节点) 32物理机

* master gw d 三个节点公用物理机
* 内存 32 * 8G
* 磁盘 300GB SAS HDD * 2 + 960GB SATA SSD * 16
* CPU  物理核心 16C * 2 逻辑核心 32 * 2
* 网卡 10G * 2

**order备集群**

* docker
* 16C 32G
* 磁盘 300GB SAS HDD * 2 + 960GB SATA SSD * 16

#### 数据库

**订单物理机 主/备 （1主 4从 3docker 1物理）16个物理机**

* 内存 8 * 8G
* 磁盘 120GB SATA SSD * 2 + 800GB SATA SSD * 6
* CPU  物理核心 8C * 2 逻辑核心 16C * 2
* 网卡 10G * 2

**JED**
![image](https://user-images.githubusercontent.com/17558555/123053737-c8602a00-d436-11eb-9c92-bb2d25fd82dc.png)
8	一主两从	24	192	768GB	12.3T
6144C	24576GB	393.6T
#### JimDB

**主订单512分片**

* 10G 内存
* 磁盘 120GB SATA SSD * 2 + 800GB SATA SSD * 6 公用
* CPU  物理核心 8C * 2 逻辑核心 16C * 2 公用
* 网卡 10G * 2 公用
