# I/O 模型

### I/O 概念

I/O (input and output) 输入和输出。I/O的可以分类为网络I/O和磁盘I/O，I/O的模型就是操作系统执行I/O指令的方法。

### I/O模型

#### 阻塞I/O(blocking I/O)

A去柜台买限量款包包，柜姐B说限量款需要排队。A专心的站在柜台前等包包，其他事情不做专项等待。直到包包有货并够资格购买时，结束等待动作进行包包购买。

![avatar](https://github.com/craftlook/Note/blob/master/image/io/b-io.png)

#### 非阻塞I/O(noblocking I/O)

A又去买包包，这次A不想把时间都花在站在柜台前排队等待，在包包可以购买前想做些其他的事情（比如逛街、喝茶、吃饭、打豆豆）。A在做这些事情时，每隔固定时间过来检查询问柜姐B是否可以购买。一旦包包到货，就停下所做的事情，飞奔过来把包包买到手。

![avatar](https://github.com/craftlook/Note/blob/master/image/io/nb-io.png)

#### 信号驱动I/O (signal blocking I/O)

A经过两次的购买已经跟柜姐B混熟，再次购买包包时柜姐B说你加点钱我有你的微信等货到了我通知你。A就可以做其他的事情，直到B通知A，A来的店里把包购买到手。

![avatar](https://github.com/craftlook/Note/blob/master/image/io/s-io.png)

#### I/O多路复用(I/O multiplexing)

A在柜台购买多次的包包，有了店里大部分柜姐的联系方式。柜姐们都有自己的渠道调取包包（多渠道提高获取效率），A就拜托多个柜姐帮忙留意。A每隔一固定时间发给各个柜姐是否可以购买，任意一个柜姐反馈可以，就进行购买操作。

![avatar](https://github.com/craftlook/Note/blob/master/image/io/m-io.png)

#### 异步I/O (asynchronous I/O)

A有事情没有精力关注包包，于是雇佣B就全权负责购买，直到B购买完送到A的手上。
![avatar](https://github.com/craftlook/Note/blob/master/image/io/a-io.png)
