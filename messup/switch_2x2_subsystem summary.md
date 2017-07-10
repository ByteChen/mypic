### 项目简介
* ![image](https://raw.githubusercontent.com/ByteChen/mypic/master/messup/switch_2x2_framework.png)
* 项目框图如上。实现的是一个2x2的交换结构，利用10G-subsystem IP核（包含PHY和MAC）的rx端从光模块接收数据，经过FIFO，进入switch结构进行交换，然后再次经过FIFO，从subsystem的tx端发送出去。

### 一、仿真成功前遇到的困难
##### 问题1. 时钟、复位非常复杂
* ![image](https://raw.githubusercontent.com/ByteChen/mypic/master/messup/CLK.png)
* subsystem的时钟连接关系如上图。由于10G-Subsystem IP核集成了10G-Ethernet-MAC 以及10G-PCS/PMA 两个IP核，因此它的时钟信号和复位信号极其复杂。再要考虑两个subsystem和4个FIFO核以及1个switch核连接时，大概总共有10个不同时钟信号，9个同步/异步复位信号。
##### 问题2. 采用block design连线方式的不足
* 以前使用10G-MAC核搭建系统时，由于时钟信号不多，这种方法还是可行的。但是由于subsystem比较复杂，使用图形连线的方法会使问题复杂化，比如说时钟分叉、同步复位的提供。
##### 问题3. 对AXI4-Lite控制总线不清楚
* AXI4-Lite控制总线利用状态机读写寄存器检查/修改系统运行状态，这一块比较复杂，估计要专门阅读AXI4-Lite控制总线的说明文档，如果有的话。后来我们干脆不用AXI4-Lite控制总线，使用configuration_vector来配置。
##### 问题4. IP核说明文档不详
* 能看懂，不会用。有些信号说得不明不白。
##### 总结
* 由于问题比较复杂，尝试了多种不同连接方式，都没有成功，仿真的时候都没有数据输出。主要问题是subsystem的时钟没给对，以及复位信号混乱。


### 二、仿真成功总结
* 能够仿真成功，主要还是借助了example design。前人总说要好好利用example design，现在算是深刻体会到了其价值。example design里有个时钟、复位模块，看懂之后稍微改一下就能用。最终我们采用代码的形式，在example design的基础上，增加switch核、FIFO核等，改出了一个完整的2x2交换系统。
##### 关于时钟
* 由于数量很多，需要细看文档，了解每个时钟应该怎么接。这里主要是利用subsystem0和时钟产生模块配合，产生时钟给其他模块使用。后期可以将subsystem0设为shared logic in core，相当于把时钟产生模块内置到subsystem0核内。这样代码会更简洁。
##### 关于同步复位
* 同步复位若是使用block design的方式设计的话，确实很麻烦，每个同步复位都要加入一个processor system reset IP核。使用代码的方式，只要利用example design里写的一个小模块，就能很方便地将复位信号同步到任意时钟。另外，值得注意的是，整个系统的所有复位信号，全部来自系统输入的reset信号，其他复位只是这个原始reset的一些衍生。并不需要给系统输入多个复位信号，否则还容易导致复位错误。
##### 关于tuser信号
* 在大多数地方，tuser信号都是为了告诉接收方，刚刚锁接收到的帧是一个好帧还是坏帧，若是坏帧则接收方会将其丢弃。但是在MAC_TX端，tuser信号并不是这个作用，此处当tuser置为1时，其作用是告诉MAC_TX端，刚传完的这个帧不正确，然后MAC会插入一些代码到xgmii中指示刚传输的数据是错误的。所以一般而言，MAC_TX端的tuser信号直接赋一个定值为1'b0。


### 三、下板子所遇到问题及解决
* 下板子时，首先遇到的时候参考时钟分叉问题，不过现场改了一下，两个subsystem共用一个时钟产生模块就行了。
* 约束文件比较麻烦，也是在example design的基础上，根据自己工程的需要，先查找电路图，确定要分配的管脚，然后约束绑定。我们调试时遇到比较多critical warning，根据警告，调整并删除非必需的管脚，多次尝试直到没有不可忽视的warning就好了。
* 将生成的比特文件download到板子后，使用testcenter输入数据帧。第一次去的时候，subsystem能正常接收到数据并输出到fifo，但是fifo无输出。然而仿真的时候是没有问题的，可见仿真的结果并不可靠，只是让我们知道逻辑大概正确了。之后我们调整了fifo IP核的配置，也没有成功。再后来我们使用example design中的FIFO模块来产生fifo，该fifo信号线如下图。由于这个fifo输出端没有tuser信号，因此其他IP核也要作相应调整。再次下板子，就成功了。
* ![image](https://raw.githubusercontent.com/ByteChen/mypic/master/messup/FIFO.png)
* 这个设计的不足之处有：① 不满足timing constraints，在极端情况下会出现大时延；②不能跑满10Gbs的速度，否则会出现丢包现象，不过能跑到10Gbs的99.99%的速率而不丢包。原因可能是fifo处引入了一些时延，这个fifo是异步fifo，要求fifo中至少存有一个完整的帧，才会往外发数据。以后在交换前拓宽数据位宽，提高交换速率，或许会好点。
* 下板子之前，可以将一些关键信号给**MARK_DEBUG**一下，然后就能在调试时观察这些信号。


### 总结
* 以后再入手新的且比较难的IP核时，仔细看完文档后不妨再仔细研究一下example design，可以少走很多弯路。