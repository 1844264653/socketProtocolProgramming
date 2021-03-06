## 追古溯源：TCP/IP和linux是如何改变世界的？

​		具体到互联网技术里，有两件事最为重要，一个是 TCP/IP 协议，它是万物互联的事实标准；另一个是 Linux 操作系统，它是推动互联网技术走向繁荣的基石



### TCP 发展历史

​		一般来说，我们认为互联网起源于阿帕网（ARPANET）。最早的阿帕网还是非常简陋的，网络控制协议（Network Control Protocol，缩写 NCP）是阿帕网中连接不同计算机的通信协议。



​		在构建阿帕网（ARPANET）之后，其他数据传输技术的研究又被摆上案头。NCP 诞生两年后，NCP 的开发者温特·瑟夫（Vinton Cerf）和罗伯特·卡恩（Robert E. Kahn）一起开发了一个阿帕网的下一代协议，并在 1974 年发表了以**分组、序列化、流量控制、超时和容错等为核心的一种新型的网络互联协议，一举奠定了 TCP/IP 协议的基础。**



### OSI & TCP/IP

​		在这之后，TCP/IP 逐渐发展。咱们话分两头说，一头是一个叫 ISO 的组织发现计算机设备的互联互通是一个值得研究的新领域，于是，这个组织出面和众多厂商游说，“我们一起定义，出一个网络互联互通的标准吧，这样大家都遵守这个标准，一起把这件事情做大，大家就有钱赚了”。众多厂商觉得可以啊，于是 ISO 组织就召集了一帮人，认真研究起了网络互联这件事情，还真的搞出来一个非常强悍的标准，这就是 OSI 参考模型。这里我不详细介绍 OSI 参考模型了，你可以阅读罗剑锋老师的专栏，他讲得很好。

​		这个标准发布的时候已经是 1984 年了，有点尴尬的是，OSI 搞得是很好，大家也都很满意，不过，等它发布的时候，ISO 组织却惊讶地发现满世界都在用一个叫做 TCP/IP 协议栈的东西，而且，跟 OSI 标准半毛钱关系都没有。

​		这就涉及到了另一头——TCP/IP 的发展。

​		事实上，我在前面提到的那两位牛人卡恩和瑟夫，一直都在不遗余力地推广 TCP/IP。当然，TCP/IP 的成功也不是偶然的，而是综合了几个因素后的结果：

1. TCP/IP 是免费或者是少量收费的，这样就扩大了使用人群；
2. TCP/IP 搭上了 UNIX 这辆时代快车，很快推出了基于套接字（socket）的实际编程接口；
3. 这是最重要的一点，TCP/IP 来源于实际需求，大家都在翘首盼望出一个统一标准，可是在此之前实际的问题总要解决啊，TCP/IP 解决了实际问题，并且在实际中不断完善。

回过来看，OSI 的七层模型定得过于复杂，并且没有参考实现，在一定程度上阻碍了普及。

​		不过，OSI 教科书般的层次模型，对后世的影响很深远，一般我们说的 4 层、7 层，也是遵从了 OSI 模型的定义，分别指代传输层和应用层。

​		我们说 TCP/IP 的应用层对应了 OSI 的应用层、表示层和会话层；TCP/IP 的网络接口层对应了 OSI 的数据链路层和物理层。

![](image\OSI七层四层模型.png)

​		



### UNIX 操作系统发展历史



​		前面我们提到了 TCP/IP 协议的成功，离不开 UNIX 操作系统的发展。接下来我们就看下 UNIX 操作系统是如何诞生和演变的。

​		下面这张图摘自维基百科，它将 UNIX 操作系统几十年的发展历史表述得非常清楚。

![](image\unix操作系统的发展.png)

UNIX 的各种版本和变体都起源于在 PDP-11 系统上运行的 UNIX 分时系统第 6 版（1976 年）和第 7 版（1979 年），它们通常分别被称为 V6 和 V7。这两个版本是在贝尔实验室以外首先得到广泛应用的 UNIX 系统。



这张图画得比较概括，我们主要从这张图上看 3 个分支：

- 图上标示的 **Research 橘黄色部分**，是由 AT&T **贝尔实验室不断开发的 UNIX 研究版本**，从此引出 UNIX 分时系统第 8 版、第 9 版，终止于 1990 年的第 10 版（10.5）。这个版本可以说是操作系统界的**少林派**。**天下武功皆出少林，世上 UNIX 皆出自贝尔实验室**。
- 图中**最上面所标识的操作系统版本**，是加州大学伯克利分校（**BSD**）研究出的分支，从此引出 4.xBSD 实现，以及后面的各种 BSD 版本。这个可以看做是**学院派**。在历史上，**学院派有力地推动了 UNIX 的发展，包括我们后面会谈到的 socket 套接字都是出自此派**。
- 图中最下面的那一个部分，是从 AT&T 分支的商业派，致力于从 UNIX 系统中谋取商业利润。从此引出了 System III 和 System V（被称为 UNIX 的商用版本），还有各大公司的 UNIX 商业版。

下面这张图也是源自维基百科，将 UNIX 的历史表达得更为详细。

![](image\详细的unix历史.png)