---
layout: post
title: Chrome Native Client 原理
date: 2011-09-13 15:59:00
categories: [Tuhuolong, 浏览器]
tags: [chrome, javascript, 浏览器, service, socket, 存储]
---
[Native
 Client:A Sandbox for Portable, Untrusted x86 Native Code](http://nativeclient.googlecode.com/svn/trunk/nacl/googleclient/native_client/documentation/nacl_paper.pdf)

**系统架构**
      一个NaCl应用程序由许多可信和不可信NaCl模块组成，每个模块都在一个进程中单独运行。假想一个基于NaCL实现的，用于管理和分享图片的应用，它由两个组件构成，一个用javascript实现的用户界面接口，运行在web浏览器中，另一个是图片处理库，作为一个Native
 Client模块实现。在这个场景，两者都是不可信的。浏览器组件通过浏览器执行环境进行限制，而图片处理库通过Native Client容器进行限制。在运行这个图片应用前，用户必须将Native
 Client作为浏览器插件进行安装，我们将它视为一个可信的NaCl模块。
      当用户访问我们这个图片应用的web站点时，浏览器会加载并执行应用程序的javascript组件部分，后者会通过Native
 Client插件去加载图片处理库到Native Client容器中来。这个加载过程不会跳出提示窗口的，Native
 Client负责对本地应用模块的行为进行限制。
      每个组件都运行在它私有的地址空间内。组件间通信是通过NaCl的可靠数据包服务---模块间通信(IMC)来进行的。NaCl为浏览器和NaCl模块之间的通信提供了两种方式：简单远程过程调用(SRPC),Netscape插件应用程序接口(NPAPI)，这两者都基于IMC实现。为了避免访高频率的通信带来的消息负担，IMC还提供了共享内存段和共享同步对象。
      NaCl模块还可以访问”服务运行时”，它提供了内存管理操作，创建线程和其他系统服务。这个接口和操作系统的系统调用接口类似。应用程序中可以包含多个Native
 Client模块，并且可信和不可信模块都可以使用IMC。例如，上面说的图片应用的用户可以选择安装一个可信的NaCl服务，用于图片的本地存储。由于它能访问本地硬盘，因此这个存储服务必须作为一个本地的浏览器插件安装，而不能作为一个Native
 Client模块实现。在应用程序初始化时，用户接口会检查这个存储服务是否可用，若可用，则会与它建立一个IMC通信通道，并把这个通信通道的描述符传递给图片处理库，从而是两者可以通过基于IMC的服务（SRPC,共享内存等）进行直接通信。这种情况下，NaCl模块一般会静态链接到一个负责提供过程接口来访问存储服务的库，这个库对底层的IMC层面的通信细节实现进行封装，从而不用理会底层到底使用的是SRPC还是共享内存。存储服务组件假定图片处理库是不可信的，并确保只执行符合接口契约的服务。
      类似上面存储服务这样的组件一般在NaCl内核以外实现，这类似操作系统的微内核模型，简单，稳定，松耦合。
      下面来详细介绍NaCl系统的各个组成部分的设计。
**内外两层沙箱**
      Native Client基于x86进程内“内层沙箱“构建而成。而为了进一步防御攻击，还构建了一个”外层沙箱“用于进程边界的隔离。
      内层沙箱使用代码静态分析技术来在不可信的x86代码中检查安全隐患。以前这样的分析受到了很多技术的挑战，比如说可自我修改的代码，重叠指令。为了解决这样的问题，在Native
 Client中为代码制定了一系列契约和结构规则，从而确保本地代码模块能可靠地进行分解，然后通过代码验证器来确保可执行文件只包含合法指令集中的指令。
      内层沙箱还利用了x86内存分段机制来限制数据和指令的内存引用。
内层沙箱用于在一个本地进程中创建一个安全的子域。在这个子域中我们可以将一个可信的运行时服务(Service Runtime)子系统和不可信模块放置在同一个进程中。通过一个安全的跳跃/计分板机制来允许可信代码和不可信代码之间的控制转移。
内层沙箱不仅将系统与本地模块进行分离，而且还使得本地模块与操作系统进行了分离。
外层沙箱是第二道防御机制。它会对运行NaCl模块的进程的所有系统调用，通过与一个允许的系统调用白名单进行比对来拒绝或通过此调用。目前白名单上允许的系统调用有46个。
**运行时机制**
**      **IMC允许可信和不可信模块之间越过进程边界发送/接收包含无类型字节数组的数据包，还可以包含用于文件共享，共享内存对象，通信通道的”NaCl资源描述符“。第一种方式是SRPC，它用于定义和使用子过程来跨模块边界访问，包括从浏览器中的javascript调用NaCl代码。第二种是NPAPI,它提供了与浏览器状态交互的接口，包括打开URL,访问DOM。这两种机制都可以用来与浏览器进行交互，包含页面内容修改，处理鼠标和键盘事件，获取其他站点内容，因为这些都可以通过javascript访问。
      如上所说，运行时服务负责为NaCl模块之间，以及和浏览器之间交互提供容器。它一般会提供应用程序编程环境相适应的一系列系统服务。为了支持malloc/free接口或其他内存分配抽象机制，它提供了sbrk()和mmap()系统调用。它提供了POSIX线程接口的一个子集，用于线程的创建和销毁，条件变量，互斥量，信号量，线程本地存储。它还提供了常用的POSIX文件I/O接口，用于通信通道以及基于web的只读内容的操作。为了防止恶意的网络访问，connect()和accept()这样的系统调用不可使用。
**攻击层面**
      从内而外，有以下几个层面会受到攻击者关注：
      1)内层沙箱：二进制验证
      2)外层沙箱：OS系统调用截取
      3)服务运行时二进制模块加载器
      4)服务运行时跳转接口
      5)IMC通信接口
      6)NPAPI接口

      除了内外层沙箱，系统还包含了CPU和NaCl模块白名单机制。
 下面来看各个组件详细的实现细节
**内层沙箱**
**      **这层仅限于机器代码中明显的控制流（即调用和跳转）。其他类型的控制流（如异常）在NaCl运行时服务中处理。
      内层沙箱定义了一系列用于可靠分解的规则，一个用于监视这些规则的编译工具链，以及一个确保代码符合规则的静态代码分析器，通过这三者来实现代码的可信检查。
NaCl模块二进制代码规则：
C1 Once  loaded  into  the  memory,  the  binary  is  not  writable, 
enforced by OS-level protection mechanisms during execution. 
C2 The  binary  is  statically  linked  at  a  start  address  of  zero,with the first byte of text at 64K.  
C3 All  indirect control transfers use  a nacljmp  pseudo-  instruction (defined below). 
C4 The  binary  is  padded  up  to  the  nearest  page  with  at  
one hlt instruction (0xf4).   
C5 The binary contains no instructions or pseudo-instructions  overlapping a 32-byte boundary.   
C6 All  valid instruction addresses are  reachable by  a  fall-  through disassembly that starts at the load (base) address.     
C7 All direct control transfers target valid instructions.                                                                            

       使用80386段来限制在虚拟32位地址空间中的连续子段中的数据引用。同时禁止一些操作码的使用,比如syscall和int（不可信代码就无法直接调用系统调用），所有修改x86段状态的指令（如lds等），ret。允许hlt指令，但它会导致模块立即终结。处于安全考虑，还禁止所有其他的特权0级的指令，因为在一个正确的用户态指令流中不该使用。（其他的细节不翻译了，太晦涩了）
       我们发现使用的验证器能以30MB/s的速度对代码进行检查，这与下载数据相比非常小，因此并不会造成性能上的问题。
**外层沙箱**
      若内存沙箱被攻破，不可信代码就可以对运行时服务进行访问，但此时外层沙箱可以对进程边界的系统调用进行控制来阻止大部分危险行为。Linux上的实现使用了ptrace接口，在windows上将会采用访问控制列表（目前只有Linux上的实现,Mac和Windows还在开发）。
      Linux上的实现将NaCl容器作为一个子进程启动，然后使用ptrace对进程发出的所有系统调用进行检查。外层沙箱维护有一个允许的系统调用白名单，试图执行任何白名单以外的系统调用都会导致NaCl模块立即结束。
      但目前使用ptrace的这个实现增加了每个系统调用的负担，因为即使它在白名单中，也会进行两次上下文切换以及一次查表，从而确定此调用是否被允许。因此对系统性能会有一定影响，也就要求开发者在使用系统调用方面谨慎，并尽量减少模块间通信频度，可以考虑使用共享的内存和同步对象，这对于高访问频率，高带宽通信的应用获得最优性能非常重要。
**异常**
      不允许使用硬件异常（段错误，浮点数异常）和外部中断。为了隔离异常，每个NaCl模块都在一个单独进程中运行，无法使用异常处理来从硬件异常中恢复过来，而是这个进程会被关闭。虽然不支持硬件异常，但确实支持c++异常，但不支持Windows结构化异常处理(WSEH).
**服务运行时**
      服务运行时是一个本地可执行代码，它被一个NPAPI插件调用，这个插件也用来支持服务运行时和浏览器之间的交互。它实现了dynamic
 enforcement机制，这个机制维护了内层沙箱的完整性，并提供了一个资源抽象层，从而使得NaCl应用程序与主机资源以及操作系统接口分离开。它包含了可信的代码和数据，但与包含的NaCl模块共享同一个进程，且后者可以通过一个受控的接口去访问服务运行时。服务运行时通过x86的内存分段和分页机制相结合的方式来阻止不可信代码的非法内存访问行为。
      当一个NaCl模块被加载进来，它就被置于一个服务运行时地址空间中段分隔的256M区域内。NaCl模块地址空间（NaCl”用户”空间）的前64Kb被服务运行时保留，用于初始化。前4kb受读写保护，用于检查空指针。剩下的60Kb包含了实现“跳跃”调用门和”记分板“返回门的可信代码段。不可信的NaCl模块紧随64kb后的区域加载进来。
(笔记：在每个native plug-in process的内部都有一段代码叫Service
 Runtime，用来模拟传统的system call，plug-in的system
 call会被重定向到这个进程内的Service Runtime，即从untrust
 code切换到trust code。在进入Service Runtime之后，系统恢复我们熟悉的flat
 memory addressing，来访问操作系统的资源。当然，只有少数安全的system call才会被Service
 Runtime处理。first level sandbox中的static
 analysis会保证unstruct native code不能随意进入Service
 Runtime.)
**通信**
      IMC是NaCl模块之间通信的基础。它基于NaCl
 socket而实现，提供了一个类似Unix domain socket的双向，可靠，有序的数据包服务。当一个不可信NaCl模块被创建时（通过DOM和javascript来创建），它会接收到第一个NaCl
 socket。Javascript使用这个socket向NaCl模块发送消息，也可以将其共享给其他NaCl模块。JavaScript还可以将NaCl
 socket共享为NaCl 描述符，从而选择将模块连接到其他服务上去。NaCl描述符还可以用来创建共享的内存段。
      通过使用NaCl消息，NaCl的SRPC完全在不可信代码中实现。使用SRPC可以定义JavaScript和NaCl模块之间，或两个NaCl模块之间的过程接口，它支持一些基本类型（int,float,char），数组以及NaCl描述符。像XDR或Protocol
 Buffers这些外部数据表示层也可以很轻松地置于NaCl消息或SRPC之上
      NPAPI的实现也基于IMC，并支持常用NPAPI接口的一个子集。目前支持的操作包括对浏览器中脚本对象的属性和方法的读，修改和调用，简单光栅图形的绘制，提供createArray方法创建数组以及像文件描述符一样打开和使用URL。后续的改进还在考虑中。
