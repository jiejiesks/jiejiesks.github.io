# 6.s081_lab11


# Network

## ethernet（以太网地址，MAC地址）

让我从最底层开始，我们先来看一下一个以太网packet的结构是什么。当两个主机非常靠近时，或许是通过相同的线缆连接，或许连接在同一个wifi网络，或许连接到同一个以太网交换机。当局域网中的两个主机彼此间要通信时，最底层的协议是以太网协议。你可以认为Host1通过以太网将Frame发送给Host2。Frame是以太网中用来描述packet的单词，本质上这就是两个主机在以太网上传输的一个个的数据Byte。以太网协议会在Frame中放入足够的信息让主机能够识别彼此，并且识别这是不是发送给自己的Frame。每个以太网packet在最开始都有一个Header，其中包含了3个数据。Header之后才是payload数据。Header中的3个数据是：目的以太网地址，源以太网地址，以及packet的类型。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241332230.png)

每一个以太网地址都是48bit的数字，这个数字唯一识别了一个网卡。packet的类型会告诉接收端的主机该如何处理这个packet。接收端主机侧更高层级的网络协议会按照packet的类型检查并处理以太网packet中的payload。

整个以太网packet，包括了48bit+48bit的以太网地址，16bit的类型，以及任意长度的payload这些都是通过线路传输。除此之外，虽然对于软件来说是不可见的，但是在packet的开头还有被硬件识别的表明packet起始的数据（注，Preamble + SFD），在packet的结束位置还有几个bit表明packet的结束（注，FCS）。packet的开头和结束的标志不会被系统内核所看到，其他的部分会从网卡送到系统内核。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241332980.png)

如果你们查看了这门课程的最后一个lab，你们可以发现我们提供的代码里面包括了一些新的文件，其中包括了kernel/net.h，这个文件中包含了大量不同网络协议的packet header的定义。上图中的代码包含了以太网协议的定义。我们提供的代码使用了这里结构体的定义来解析收到的以太网packet，进而获得目的地址和类型值（注，实际中只需要对收到的raw data指针强制类型转换成结构体指针就可以完成解析）。

> 学生提问：硬件用来识别以太网packet的开头和结束的标志是不是类似于lab中的End of Packets？
>
> Robert教授：并不是的，EOP是帮助驱动和网卡之间通信的机制。这里的开头和结束的标志是在线缆中传输的电信号或者光信号，这些标志位通常在一个packet中是不可能出现的。以结束的FCS为例，它的值通常是packet header和payload的校验和，可以用来判断packet是否合法。

有关以太网48bit地址，是为了给每一个制造出来的网卡分配一个唯一的ID，所以这里有大量的可用数字。这里48bit地址中，前24bit表示的是制造商，每个网卡制造商都有自己唯一的编号，并且会出现在前24bit中。后24bit是由网卡制造商提供的任意唯一数字，通常网卡制造商是递增的分配数字。所以，如果你从一个网卡制造商买了一批网卡，每个网卡都会被写入属于自己的地址，并且如果你查看这些地址，你可以发现，这批网卡的高24bit是一样的，而低24bit极有可能是一些连续的数字。

虽然以太网地址是唯一的，但是出了局域网，它们对于定位目的主机的位置是没有帮助的。如果网络通信的目的主机在同一个局域网，那么目的主机会监听发给自己的地址的packet。但是如果网络通信发生在两个国家的主机之间，你需要使用一个不同的寻址方法，这就是IP地址的作用。

在实际中，你可以使用tcpdump来查看以太网packet。这将会是lab的一部分。下图是tcpdump的一个输出：

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241333883.png)

tcpdump输出了很多信息，其中包括：

- 

  接收packet的时间

- 

  第一行的剩下部分是可读的packet的数据

- 

  接下来的3行是收到packet的16进制数

如果按照前面以太网header的格式，可以发现packet中：

- 

  前48bit是一个广播地址，0xffffffffffff。广播地址是指packet需要发送给局域网中的所有主机。

- 

  之后的48bit是发送主机的以太网地址，我们并不能从这个地址发现什么，实际上这个地址是运行在QEMU下的XV6生成的地址，所以地址中的前24bit并不是网卡制造商的编号，而是QEMU编造的地址。

- 

  接下来的16bit是以太网packet的类型，这里的类型是0x0806，对应的协议是ARP。

- 

  剩下的部分是ARP packet的payload。

## ARP

下一个与以太网通信相关的协议是ARP。在以太网层面，每个主机都有一个以太网地址。但是为了能在互联网上通信，你需要有32bit的IP地址。为什么需要IP地址呢？因为IP地址有额外的含义。IP地址的高位bit包含了在整个互联网中，这个packet的目的地在哪。所以IP地址的高位bit对应的是网络号，虽然实际上要更复杂一些，但是你可以认为互联网上的每一个网络都有一个唯一的网络号。路由器会检查IP地址的高bit位，并决定将这个packet转发给互联网上的哪个路由器。IP地址的低bit位代表了在局域网中特定的主机。当一个经过互联网转发的packet到达了局域以太网，我们需要从32bit的IP地址，找到对应主机的48bit以太网地址。这里是通过一个动态解析协议完成的，也就是Address Resolution Protocol，ARP协议。

当一个packet到达路由器并且需要转发给同一个以太网中的另一个主机，或者一个主机将packet发送给同一个以太网中的另一个主机时，发送方首先会在局域网中广播一个ARP packet，来表示任何拥有了这个32bit的IP地址的主机，请将你的48bit以太网地址返回过来。如果相应的主机存在并且开机了，它会向发送方发送一个ARP response packet。

下图是一个ARP packet的格式：

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241333072.png)

它会出现在一个以太网packet的payload中。所以你们看到的将会是这样的结构：首先是以太网header，它包含了48bit的目的以太网地址，48bit的源以太网地址，16bit的类型；之后的以太网的payload会是ARP packet，包含了上图的内容。

接收到packet的主机通过查看以太网header中的16bit类型可以知道这是一个ARP packet。在ARP中类型值是0x0806。通过识别类型，接收到packet的主机就知道可以将这个packet发送给ARP协议处理代码。

有关ARP packet的内容，包含了不少信息，但是基本上就是在说，现在有一个IP地址，我想将它转换成以太网地址，如果你拥有这个IP地址，请响应我。

同样的，我们也可以通过tcpdump来查看这些packet。在网络的lab中，XV6会在QEMU模拟的环境下发送IP packet。所以你们可以看到在XV6和其他主机之间有ARP的交互。下图中第一个packet是我的主机想要知道XV6主机的以太网地址，第二个packet是XV6在收到了第一个packet之后，并意识到自己是IP地址的拥有者，然后返回response。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241333146.png)

tcpdump能够解析出ARP packet，并将数据打印在第一行。对应ARP packet的格式，在第一个packet中，10.0.2.2是SIP，10.0.2.15是DIP。在第二个packet中，52:54:00:12:34:56对应SHA。

同时，我们也可以自己分析packet的原始数据。对于第一个packet：

- 

  前14个字节是以太网header，包括了48bit目的以太网地址，48bit源以太网地址，16bit类型。

- 

  从后往前看，倒数4个字节是TIP，也就是发送方想要找出对应以太网地址的IP地址。每个字节对应了IP地址的一块，所以0a00 020f对应了IP地址10.0.2.15。

- 

  再向前数6个字节，是THA，也就是目的地的以太网地址，现在还不知道所以是全0。

- 

  再向前数4个字节是SIP，也就是发送方的IP地址，0a000202对应了IP地址10.0.2.2。

- 

  再向前数6个字节是SHA，也就是发送方的以太网地址。

- 

  剩下的8个字节表明了我们感兴趣的是以太网和IP地址格式。

第二个packet是第一个packet的响应。

> 学生提问：ethernet header中已经包括了发送方的以太网地址，为什么ARP packet里面还要包含发送方的以太网地址？
>
> Robert教授：我并不清楚为什么ARP packet里面包含了这些数据，我认为如果你想的话是可以精简一下ARP packet。或许可以这么理解，ARP协议被设计成也可以用在其他非以太网的网络中，所以它被设计成独立且不依赖其他信息，所以ARP packet中包含了以太网地址。现在我们是在以太网中发送ARP packet，以太网packet也包含了以太网地址，所以，如果在以太网上运行ARP，这些信息是冗余的。但是如果在其他的网络上运行ARP，你或许需要这些信息，因为其他网络的packet中并没有包含以太网地址。
>
> 学生提问：tcpdump中原始数据的右侧是什么内容？
>
> Robert教授：这些是原始数据对应的ASCII码，“.”对应了一个字节并没有相应的ASCII码，0x52对应了R，0x55对应了U。当我们发送的packet包含了ASCII字符时，这里的信息会更加有趣。

我希望你们在刚刚的讨论中注意到这一点，网络协议和网络协议header是嵌套的。我们刚刚看到的是一个packet拥有了ethernet header和ethernet payload。在ethernet payload中，首先出现的是ARP header，对于ARP来说并没有的payload。但是在ethernet packet中还可以包含其他更复杂的结构，比如说ethernet payload中包含一个IP packet，IP packet中又包含了一个UDP packet，所以IP header之后是UDP header。如果在UDP中包含另一个协议，那么UDP payload中又可能包含其他的packet，例如DNS packet。所以发送packet的主机会按照这样的方式构建packet：DNS相关软件想要在UDP协议之上构建一个packet；UDP相关软件会将UDP header挂在DNS packet之前，并在IP协议之上构建另一个packet；IP相关的软件会将IP heade挂在UDP packet之前；最后Ethernet相关的软件会将Ethernet header挂在IP header之前。所以整个packet是在发送过程中逐渐构建起来的。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241333965.png)

类似的，当一个操作系统收到了一个packet，它会先解析第一个header并知道这是Ethernet，经过一些合法性检查之后，Ethernet header会被剥离，操作系统会解析下一个header。在Ethernet  header中包含了一个类型字段，它表明了该如何解析下一个header。同样的在IP header中包含了一个protocol字段，它也表明了该如何解析下一个header。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241333308.png)

软件会解析每个header，做校验，剥离header，并得到下一个header。一直重复这个过程直到得到最后的数据。这就是嵌套的packet header。

## Internet

Ethernet header足够在一个局域网中将packet发送到一个host。如果你想在局域网发送一个IP packet，那么你可以使用ARP获得以太网地址。但是IP协议更加的通用，IP协议能帮助你向互联网上任意位置发送packet。下图是一个IP packet的header，你们可以在lab配套的代码中的net.h文件找到。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241333140.png)

如果IP packet是通过以太网传输，那么你可以看到，在一个以太网packet中，最开始是目的以太网地址，源以太网地址，以太网类型是0x0800，之后是IP header，最后是IP payload。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241333154.png)

在一个packet发送到世界另一端的网络的过程中，IP header会被一直保留，而Ethernet header在离开本地的以太网之后会被剥离。或许packet在被路由的过程中，在每一跳（hop）会加上一个新的Ethernet header。但是IP header从源主机到目的主机的过程中会一直保留。

IP header具有全局的意义，而Ethernet header只在单个局域网有意义。所以IP header必须包含足够的信息，这样才能将packet传输给互联网上遥远的另一端。对于我们来说，关键的信息是三个部分，目的IP地址（ip_dst），源IP地址（ip_src）和协议（ip_p）。目的IP地址是我们想要将packet送到的目的主机的IP地址。地址中的高bit位是网络号，它会帮助路由器完成路由。IP header中的协议字段会告诉目的主机如何处理IP payload。

如果你们看到过MIT的IP地址，你们可以看到IP地址是18.x.x.x，虽然最近有些变化，但是在很长一段时间18是MIT的网络号。所以MIT的大部分主机的IP地址最高字节就是18。全世界的路由器在看到网络号18的时候，就知道应该将packet路由到离MIT更近的地方。

接下来我们看一下包含了IP packet的tcpdump输出。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241333342.png)

因为这个IP packet是在以太网上传输，所以它包含了以太网header。呃……，实际上这个packet里面有点问题，我不太确定具体的原因是什么，但是Ethernet header中目的以太网地址不应该是全f，因为全f是广播地址，它会导致packet被发送到所有的主机上。一个真实网络中两个主机之间的packet，不可能出现这样的以太网地址。所以我提供的针对network lab的方案，在QEMU上运行有点问题。不管怎么样，我们可以看到以太网目的地址，以太网源地址，以及以太网类型0x0800。0x0800表明了Ethernet payload是一个IP packet。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241333338.png)

IP header的长度是20个字节，所以中括号内的是IP header，

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241333699.png)

从后向前看：

- 

  目的IP地址是0x0a000202，也就是10.0.2.2。

- 

  源IP地址是0x0a00020f，也就是10.0.2.15。

- 

  再向前有16bit的checksum，也就是0x3eae。IP相关的软件需要检查这个校验和，如果结果不匹配应该丢包。

- 

  再向前一个字节是protocol，0x11对应的是10进制17，表明了下一层协议是UDP

- 

  其他的就是我们不太关心的一些字段了，例如packet的长度。

IP header中的protocol字段告诉了目的主机的网络协议栈，这个packet应该被UDP软件处理。

## UDP

IP header足够让一个packet传输到互联网上的任意一个主机，但是我们希望做的更好一些。每一个主机都运行了大量需要使用网络的应用程序，所以我们需要有一种方式能区分一个packet应该传递给目的主机的哪一个应用程序，而IP header明显不包含这种区分方式。有一些其他的协议完成了这里的区分工作，其中一个是TCP，它比较复杂，而另一个是UDP。TCP不仅帮助你将packet发送到了正确的应用程序，同时也包含了序列号等用来检测丢包并重传的功能，这样即使网络出现问题，数据也能完整有序的传输。相比之下，UDP就要简单的多，它以一种“尽力而为”的方式将packet发送到目的主机，除此之外不提供任何其他功能。

UDP header中最关键的两个字段是sport源端口和dport目的端口。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241334397.png)

当你的应用程序需要发送或者接受packet，它会使用socket API，这包含了一系列的系统调用。一个进程可以使用socket API来表明应用程序对于特定目的端口的packet感兴趣。当应用程序调用这里的系统调用，操作系统会返回一个文件描述符。每当主机收到了一个目的端口匹配的packet，这个packet会出现在文件描述符中，之后应用程序就可以通过文件描述符读取packet。

这里的端口分为两类，一类是常见的端口，例如53对应的是DNS服务的端口，如果你想向一个DNS server发请求，你可以发送一个UDP packet并且目的端口是53。除此之外，很多常见的服务都占用了特定的端口。除了常见端口，16bit数的剩下部分被用来作为匿名客户端的源端口。比如说，我想向一个DNS server的53端口发送一个packet，目的端口会是53，但是源端口会是一个本地随机选择的端口，这个随机端口会与本地的应用程序的socket关联。所以当DNS server向本地服务器发送一个回复packet，它会将请求中的源端口拷贝到回复packet的目的端口，再将回复packet发送回本地的服务器。本地服务器会使用这个端口来确定应该将packet发送给哪个应用程序。

接下来我们看一下UDP packet的tcpdump输出。首先，我们同样会有一个以太网Header，以及20字节的IP header。IP header中的0x11表明这个packet的IP协议号是17，这样packet的接收主机就知道应该使用UDP软件来处理这个packet。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241334665.png)

接下来的8个字节是UDP header。这里的packet是由lab代码生成的packet，所以它并没有包含常见的端口，源端口是0x0700，目的端口是0x6403。第4-5个字节是长度，第6-7个字节是校验和。XV6的UDP软件并没有生成UDP的校验和。

![img](https://jiejiesks.oss-cn-beijing.aliyuncs.com/Note/202305241334597.png)

UDP header之后就是UDP的payload。在这个packet中，应用程序发送的是ASCII文本，所以我们可以从右边的ASCII码看到，内容是“a.message.from.xv6”。所以ASCII文本放在了一个UDP packet中，然后又放到了一个IP packet中，然后又放到了一个Ethernet packet中。最后发布到以太网上。

> 学生提问：当你发送一个packet给一个主机，但是你又不知道它的以太网地址，这个packet是不是会被送到路由器，之后再由路由器来找到以太网地址？
>
> Robert教授：如果你发送packet到一个特定的IP地址，你的主机会先检查packet的目的IP地址来判断目的主机是否与你的主机在同一个局域网中。如果是的话，你的主机会直接使用ARP来将IP地址翻译成以太网地址，再将packet通过以太网送到目的主机。更多的场景是，我们将一个packet发送到互联网上某个主机。这时，你的主机会将packet发送到局域网上的路由器，路由器会检查packet的目的IP地址，并根据路由表选择下一个路由器，将packet转发给这个路由器。这样packet一跳一跳的在路由器之间转发，最终离目的主机越来越近。
>
> 学生提问：对于packet的长度有限制吗？
>
> Robert教授：有的。这里有几个不同的限制，每一个底层的网络技术，例如以太网，都有能传输packet的上限。今天我们要讨论的论文基于以太网最大可传输的packet是1500字节。最新的以太网可以支持到9000或者10000字节的最大传输packet。为什么不支持传输无限长度的packet呢？这里有几个原因：
>
> - 
>
>   发送无限长度的packet的时间可能要很长，期间线路上会有信号噪音和干扰，所以在发送packet的时候可能会收到损坏的bit位。基本上每一种网络技术都会在packet中带上某种校验和或者纠错码，但是校验和也好，纠错码也好，只能在一定长度的bit位内稳定的检测错误。如果packet长度增加，遗漏错误的可能性就越来越大。所以一个校验和的长度，例如16bit或者32bit，限制了传输packet的最大长度。
>
> - 
>
>   另一个限制是，如果发送巨大的packet，传输路径上的路由器和主机需要准备大量的buffer来接收packet。这里的代价又比较高，因为较难管理一个可变长度的buffer，管理一个固定长度的buffer是最方便的。而固定长度的buffer要求packet的最大长度不会太大。
>
> 所以，以太网有1500或者9000字节的最大packet限制。除此之外，所有的协议都有长度字段，例如UDP的长度字段是16bit。所以即使以太网支持传输更大的packet，协议本身对于数据长度也有限制。

以上就是UDP的介绍。在lab的最后你们会通过实验提供的代码来向谷歌的DNS server发送一个查询，收到回复之后代码会打印输出。你们需要在设备驱动侧完成以太网数据的处理。

# lab11

YOUR JOB

您的工作是在***kernel/e1000.c\***中完成`e1000_transmit()`和`e1000_recv()`，以便驱动程序可以发送和接收数据包。当`make grade`表示您的解决方案通过了所有测试时，您就完成了。

[!TIP] 在编写代码时，您会发现自己参考了[《E1000软件开发人员手册》](https://pdos.csail.mit.edu/6.828/2020/readings/8254x_GBe_SDM.pdf)。以下部分可能特别有用：

- Section 2是必不可少的，它概述了整个设备。
- Section 3.2概述了数据包接收。
- Section 3.3与Section 3.4一起概述了数据包传输。
- Section 13概述了E1000使用的寄存器。
- Section 14可能会帮助您理解我们提供的init代码。

浏览[《E1000软件开发人员手册》](https://pdos.csail.mit.edu/6.828/2020/readings/8254x_GBe_SDM.pdf)。本手册涵盖了几个密切相关的以太网控制器。QEMU模拟82540EM。现在浏览第2章，了解该设备。要编写驱动程序，您需要熟悉第3章和第14章以及第4.1节（虽然不包括4.1的子节）。你还需要参考第13章。其他章节主要介绍你的驱动程序不必与之交互的E1000组件。一开始不要担心细节；只需了解文档的结构，就可以在以后找到内容。E1000具有许多高级功能，其中大部分您可以忽略。完成这个实验只需要一小部分基本功能。

我们在***e1000.c\***中提供的`e1000_init()`函数将E1000配置为读取要从RAM传输的数据包，并将接收到的数据包写入RAM。这种技术称为DMA，用于直接内存访问，指的是E1000硬件直接向RAM写入和读取数据包。

由于数据包突发到达的速度可能快于驱动程序处理数据包的速度，因此`e1000_init()`为E1000提供了多个缓冲区，E1000可以将数据包写入其中。E1000要求这些缓冲区由RAM中的“描述符”数组描述；每个描述符在RAM中都包含一个地址，E1000可以在其中写入接收到的数据包。`struct rx_desc`描述描述符格式。描述符数组称为接收环或接收队列。它是一个圆环，在这个意义上，当网卡或驱动程序到达队列的末尾时，它会绕回到数组的开头。`e1000_init()`使用`mbufalloc()`为要进行DMA的E1000分配`mbuf`数据包缓冲区。此外还有一个传输环，驱动程序将需要E1000发送的数据包放入其中。`e1000_init()`将两个环的大小配置为`RX_RING_SIZE`和`TX_RING_SIZE`。

当***net.c\***中的网络栈需要发送数据包时，它会调用`e1000_transmit()`，并使用一个保存要发送的数据包的`mbuf`作为参数。传输代码必须在TX（传输）环的描述符中放置指向数据包数据的指针。`struct tx_desc`描述了描述符的格式。您需要确保每个`mbuf`最终被释放，但只能在E1000完成数据包传输之后（E1000在描述符中设置`E1000_TXD_STAT_DD`位以指示此情况）。

当当E1000从以太网接收到每个包时，它首先将包DMA到下一个RX(接收)环描述符指向的`mbuf`，然后产生一个中断。`e1000_recv()`代码必须扫描RX环，并通过调用`net_rx()`将每个新数据包的`mbuf`发送到网络栈（在***net.c\***中）。然后，您需要分配一个新的`mbuf`并将其放入描述符中，以便当E1000再次到达RX环中的该点时，它会找到一个新的缓冲区，以便DMA新数据包。

除了在RAM中读取和写入描述符环外，您的驱动程序还需要通过其内存映射控制寄存器与E1000交互，以检测接收到数据包何时可用，并通知E1000驱动程序已经用要发送的数据包填充了一些TX描述符。全局变量`regs`包含指向E1000第一个控制寄存器的指针；您的驱动程序可以通过将`regs`索引为数组来获取其他寄存器。您需要特别使用索引`E1000_RDT`和`E1000_TDT`。

要测试驱动程序，请在一个窗口中运行`make server`，在另一个窗口中运行`make qemu`，然后在xv6中运行`nettests`。`nettests`中的第一个测试尝试将UDP数据包发送到主机操作系统，地址是`make server`运行的程序。如果您还没有完成实验，E1000驱动程序实际上不会发送数据包，也不会发生什么事情。

完成实验后，E1000驱动程序将发送数据包，qemu将其发送到主机，`make server`将看到它并发送响应数据包，然后E1000驱动程序和`nettests`将看到响应数据包。但是，在主机发送应答之前，它会向xv6发送一个“ARP”请求包，以找出其48位以太网地址，并期望xv6以ARP应答进行响应。一旦您完成了对E1000驱动程序的工作，***kernel/net.c\***就会处理这个问题。如果一切顺利，`nettests`将打印`testing ping: OK`，`make server`将打印`a message from xv6!`。

`tcpdump -XXnr packets.pcap`应该生成这样的输出:

```
reading from file packets.pcap, link-type EN10MB (Ethernet)
15:27:40.861988 IP 10.0.2.15.2000 > 10.0.2.2.25603: UDP, length 19
        0x0000:  ffff ffff ffff 5254 0012 3456 0800 4500  ......RT..4V..E.
        0x0010:  002f 0000 0000 6411 3eae 0a00 020f 0a00  ./....d.>.......
        0x0020:  0202 07d0 6403 001b 0000 6120 6d65 7373  ....d.....a.mess
        0x0030:  6167 6520 6672 6f6d 2078 7636 21         age.from.xv6!
15:27:40.862370 ARP, Request who-has 10.0.2.15 tell 10.0.2.2, length 28
        0x0000:  ffff ffff ffff 5255 0a00 0202 0806 0001  ......RU........
        0x0010:  0800 0604 0001 5255 0a00 0202 0a00 0202  ......RU........
        0x0020:  0000 0000 0000 0a00 020f                 ..........
15:27:40.862844 ARP, Reply 10.0.2.15 is-at 52:54:00:12:34:56, length 28
        0x0000:  ffff ffff ffff 5254 0012 3456 0806 0001  ......RT..4V....
        0x0010:  0800 0604 0002 5254 0012 3456 0a00 020f  ......RT..4V....
        0x0020:  5255 0a00 0202 0a00 0202                 RU........
15:27:40.863036 IP 10.0.2.2.25603 > 10.0.2.15.2000: UDP, length 17
        0x0000:  5254 0012 3456 5255 0a00 0202 0800 4500  RT..4VRU......E.
        0x0010:  002d 0000 0000 4011 62b0 0a00 0202 0a00  .-....@.b.......
        0x0020:  020f 6403 07d0 0019 3406 7468 6973 2069  ..d.....4.this.i
        0x0030:  7320 7468 6520 686f 7374 21              s.the.host!
```

您的输出看起来会有些不同，但它应该包含字符串“ARP, Request”，“ARP, Reply”，“UDP”，“a.message.from.xv6”和“this.is.the.host”。

`nettests`执行一些其他测试，最终通过（真实的）互联网将DNS请求发送到谷歌的一个名称服务器。您应该确保您的代码通过所有这些测试，然后您应该看到以下输出：

```bash
$ nettests
nettests running on port 25603
testing ping: OK
testing single-process pings: OK
testing multi-process pings: OK
testing DNS
DNS arecord for pdos.csail.mit.edu. is 128.52.129.126
DNS OK
all tests passed.
```

您应该确保`make grade`同意您的解决方案通过。

## 提示

首先，将打印语句添加到`e1000_transmit()`和`e1000_recv()`，然后运行`make server`和（在xv6中）`nettests`。您应该从打印语句中看到，`nettests`生成对`e1000_transmit`的调用。

**实现`e1000_transmit`的一些提示：**

- 首先，通过读取`E1000_TDT`控制寄存器，向E1000询问等待下一个数据包的TX环索引。
- 然后检查环是否溢出。如果`E1000_TXD_STAT_DD`未在`E1000_TDT`索引的描述符中设置，则E1000尚未完成先前相应的传输请求，因此返回错误。
- 否则，使用`mbuffree()`释放从该描述符传输的最后一个`mbuf`（如果有）。
- 然后填写描述符。`m->head`指向内存中数据包的内容，`m->len`是数据包的长度。设置必要的cmd标志（请参阅E1000手册的第3.3节），并保存指向`mbuf`的指针，以便稍后释放。
- 最后，通过将一加到`E1000_TDT再对TX_RING_SIZE`取模来更新环位置。
- 如果`e1000_transmit()`成功地将`mbuf`添加到环中，则返回0。如果失败（例如，没有可用的描述符来传输`mbuf`），则返回-1，以便调用方知道应该释放`mbuf`。

**实现`e1000_recv`的一些提示：**

- 首先通过提取`E1000_RDT`控制寄存器并加一对`RX_RING_SIZE`取模，向E1000询问下一个等待接收数据包（如果有）所在的环索引。
- 然后通过检查描述符`status`部分中的`E1000_RXD_STAT_DD`位来检查新数据包是否可用。如果不可用，请停止。
- 否则，将`mbuf`的`m->len`更新为描述符中报告的长度。使用`net_rx()`将`mbuf`传送到网络栈。
- 然后使用`mbufalloc()`分配一个新的`mbuf`，以替换刚刚给`net_rx()`的`mbuf`。将其数据指针（`m->head`）编程到描述符中。将描述符的状态位清除为零。
- 最后，将`E1000_RDT`寄存器更新为最后处理的环描述符的索引。
- `e1000_init()`使用mbufs初始化RX环，您需要通过浏览代码来了解它是如何做到这一点的。
- 在某刻，曾经到达的数据包总数将超过环大小（16）；确保你的代码可以处理这个问题。

```c
int
e1000_transmit(struct mbuf *m)
{
  //
  // 把m的信息赋值给desc,并把数据放入tx_mbufs中
  //
  // the mbuf contains an ethernet frame; program it into
  // the TX descriptor ring so that the e1000 sends it. Stash
  // a pointer so that it can be freed after sending.
  //
  acquire(&e1000_lock); // 获取 E1000 的锁，防止多进程同时发送数据出现 race

  uint32 ind = regs[E1000_TDT]; // 下一个可用的 buffer 的下标
  struct tx_desc *desc = &tx_ring[ind]; // 获取 buffer 的描述符，其中存储了关于该 buffer 的各种信息
  // 如果该 buffer 中的数据还未传输完，则代表我们已经将环形 buffer 列表全部用完，缓冲区不足，返回错误
  if(!(desc->status & E1000_TXD_STAT_DD)) {
    release(&e1000_lock);
    return -1;
  }
  
  // 如果该下标仍有之前发送完毕但未释放的 mbuf，则释放
  if(tx_mbufs[ind]) {
    mbuffree(tx_mbufs[ind]);
    tx_mbufs[ind] = 0;
  }

  // 将要发送的 mbuf 的内存地址与长度填写到发送描述符中
  desc->addr = (uint64)m->head;
  desc->length = m->len;
  // 设置参数，EOP 表示该 buffer 含有一个完整的 packet
  // RS 告诉网卡在发送完成后，设置 status 中的 E1000_TXD_STAT_DD 位，表示发送完成。
  desc->cmd = E1000_TXD_CMD_EOP | E1000_TXD_CMD_RS;
  // 保留新 mbuf 的指针，方便后续再次用到同一下标时释放。
  tx_mbufs[ind] = m;

  // 环形缓冲区内下标增加一。
  regs[E1000_TDT] = (regs[E1000_TDT] + 1) % TX_RING_SIZE;
  
  release(&e1000_lock);

  return 0;
}

static void
e1000_recv(void)
{
  //
  // 获取缓冲区的desc并设置rx_mbufs的length,然后将rx_mbufs传递给net_rx
  //
  // Check for packets that have arrived from the e1000
  // Create and deliver an mbuf for each packet (using net_rx()).
  //
  while(1) { // 每次 recv 可能接收多个包

    uint32 ind = (regs[E1000_RDT] + 1) % RX_RING_SIZE;
    
    struct rx_desc *desc = &rx_ring[ind];
    // 如果需要接收的包都已经接收完毕，则退出
    if(!(desc->status & E1000_RXD_STAT_DD)) {
      return;
    }

    rx_mbufs[ind]->len = desc->length;
    
    net_rx(rx_mbufs[ind]); // 传递给上层网络栈。上层负责释放 mbuf

    // 分配并设置新的 mbuf，供给下一次轮到该下标时使用
    rx_mbufs[ind] = mbufalloc(0); 
    desc->addr = (uint64)rx_mbufs[ind]->head;
    desc->status = 0;

    regs[E1000_RDT] = ind;
  }

}
```


