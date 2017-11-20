# 学习Binder

> Binder机制的目的**是实现IPC(Inter-Process Communication)，即实现进程间通信**。

## 知识储备

### 1.Linux系统中内存的划分

​	在Linux系统中，应用程序都运行在用户空间，而Kernel和驱动都运行在内核空间。用户空间和内核空间若涉及到通信(即数据交互)，两者不能简单地使用指针传递数据，而必须在"内核"中copy_from_user(),copy_to_user(),get_user()或put_user()等函数传递数据。copy_from_user()和get_user()是将内核空间的数据拷贝到用户空间，而copy_to_user()和put_user()则是将用户空间的数据拷贝到内核空间。

### 2.进程的概念

​	进程可以说是程序运行在上面的抽象CPU，它拥有独立的内存单位，是系统进行资源分配和调度的基本单位。对Linux系统来说，每个用户空间运行的应用程序都可以看成一个进程。不同进程在不同内存当中。

### 3.代理模式-远程代理

​	远程代理是最经典的代理模式之一，远程代理负责与远程JVM通信，以实现本地调用者与远程被调用者之间的正常交互。

![img](http://images.cnitblog.com/blog/610080/201410/042203005033153.jpg)

> 具体流程是这样的：
>
> 1. Client向Stub发送方法调用请求（Client以为Stub就是Server）
> 2. Stub接到请求，通过Socket与服务端的Skeleton通信，把调用请求传递给Skeleton
> 3. Skeleton接到请求，调用本地Server（听起来有点奇怪，这里Server相当于Service）
> 4. Server作出对应动作，把结果返回给调用者Skeleton
> 5. Skeleton接到结果之后通过Socket发送给Stub
> 6. Stub把结果传递给Client

### 4.内存映射

> ​	内存映射，简而言之就是将用户空间的一段内存区域映射到内核空间，映射成功后，用户对这段内存区域的修改可以直接反映到内核空间，相反，内核空间对这段区域的修改也直接反映用户空间。那么对于内核空间<—->用户空间两者之间需要大量数据传输等操作的话效率是非常高的。



## Binder的通信模型

![img](http://ww1.sinaimg.cn/large/006tKfTcjw1f7adz7zozlj31eq0vowo0.jpg)

上图中涉及到Binder模型的4类角色：**Binder驱动**，**ServiceManager**，**Server**和**Client**。

### Binder驱动

> 两个不同进程的通信必须要内核进行中转，对于Android而言，在内核中起中转作用便是Binder驱动。

​	Android的通信是基于Client-Server架构的，进程间的通信无非就是Client向Server发起请求，Server响应Client的请求。这里以发起请求为例：当Client向Server发起请求(例如，MediaPlayer向MediaPlayerService发起请求)，Client会先将请求数据从用户空间拷贝到内核空间(将数据从MediaPlayer发给Binder驱动)；数据被拷贝到内核空间之后，再通过驱动程序，将内核空间中的数据拷贝到Server位于用户空间的缓存中(Binder驱动将数据发给MediaPlayerService)。这样，就成功的将Client进程中的请求数据传递到了Server进程中。

​	实际上，Binder驱动是整个Binder机制的核心。除了实现上面所说的数据传输之外，Binder驱动还是实现线程控制(通过中断等待队列实现线程的等待/唤醒)，以及UID/PID等安全机制的保证。



### ServiceManager

> ​	Binder是要实现Android的C-S架构的，即Client-Server架构。而ServiceManager，是以服务管理者的身份存在的。ServiceManager也是运行在用户空间的一个独立进程。



​	对于Binder驱动而言，**ServiceManager是一个守护进程，更是Android系统各个服务的管理者**。Android系统中的各个服务，都是添加到ServiceManager中进行管理的，而且每个服务都对应一个服务名。当Client获取某个服务时，则**通过服务名**来从ServiceManager中获取相应的服务。



​	 对于MediaPlayerService和MediaPlayer而言，**ServiceManager是一个Server服务端，是一个服务器**。当要将MediaPlayerService等服务添加到ServiceManager中进行管理时，ServiceManager是服务器，它会收到MediaPlayerService进程的添加服务请求。当MediaPlayer等客户端要获取MediaPlayerService等服务时，它会向ServiceManager发起获取服务请求。



​	当MediaPlayer和MediaPlayerService通信时，MediaPlayerService是服务端；而当MediaPlayerService与ServiceManager通信时，ServiceManager则是服务端。这样，就造就了ServiceManager的特殊性。于是，在Binder驱动中，将句柄0指定为ServiceManager对应的句柄，通过这个特殊的句柄就能获取ServiceManager对象。



### Client、Server

> ​	这两个便是要进行通信的两个不同进程，比如MediaPlayerService和MediaPlayer



## 为什么选择Binder

> ​	Android是在Linux内核的基础上设计的。但是在Linux中，拥有"**管道**/**消息队列**/**共享内存**/**信号量**/**Socket**等等"众多的IPC通信手段。为什么不选择其它的IPC通信方式，而要设计出Binder？



### 1.从传输效率上看，Binder更能实现C-S架构

​	Linux支持的"传统的管道/消息队列/共享内存/信号量/Socket等"IPC通信手段中，只有Socket是Client-Server的通信方式。**但是** ， socket作为一款通用接口，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信。



### 2.从传输效率和可操作性上看

> ​	**管道，消息队列**采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程。**效率太低！**



> ​	对于**共享内存**来说，虽然使用它进行IPC通信时进行的内存拷贝次数是0。但是，共享内存操作复杂，并不适合用。



​	采用Binder机制的话，则只需要经过**1次**内存拷贝即可！ 即，从发送方的缓存区拷贝到内核的缓存区，而接收方的缓存区与内核的缓存区是映射到同一块物理地址的（内存映射），因此只需要1次拷贝即可。



### 从安全性来看，Binder更安全

> ​	传统IPC没有任何安全措施，完全依赖上层协议来确保。传统IPC的接收方无法获得对方进程可靠的UID/PID(用户ID/进程ID)，从而无法鉴别对方身份。而Binder机制则为每个进程分配了UID/PID来作为鉴别身份的标示，并且在Binder通信时会根据UID/PID进行有效性检测。



## Binder中的各个联系

![img](https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_4relationship.jpg)



### 重要概念说明

#### 1.Binder实体

> Binder实体，是各个Server以及ServiceManager在内核中的存在形式。

​	Binder实体实际上是内核中binder_node结构体的对象，它的作用是在内核中保存Server和ServiceManager的信息(例如，Binder实体中保存了Server对象在用户空间的地址)。简言之，Binder实体是Server在Binder驱动中的存在形式，内核通过Binder实体可以找到用户空间的Server对象。

​	在上图中，Server和ServiceManager在Binder驱动中都对应的存在一个Binder实体。

#### 2.Binder引用

> 所谓Binder引用，实际上是内核中binder_ref结构体的对象，它的作用是在表示"Binder实体"的引用。

​	每一个Binder引用都是某一个Binder实体的引用，通过Binder引用可以在内核中找到它对应的Binder实体。如果将Server看作是Binder实体的话，那么Client就好比Binder引用。Client要和Server通信，它就是通过保存一个Server对象的Binder引用，再通过该Binder引用在内核中找到对应的Binder实体，进而找到Server对象，然后将通信内容发送给Server对象。

​	Binder实体和Binder引用都是内核(即，Binder驱动)中的数据结构。每一个Server在内核中就表现为一个Binder实体，而每一个Client则表现为一个Binder引用。这样，每个Binder引用都对应一个Binder实体，而每个Binder实体则可以多个Binder引用。

#### 3.远程服务

​	Server都是以服务的形式注册到ServiceManager中进行管理的。如果将Server本身看作是"本地服务"的话，那么Client中的"远程服务"就是本地服务的代理。远程服务就是本地服务的一个代理，通过该远程服务Client就能和Server进行通信。



### 简要流程分析

#### ServiceManager守护进程

​	ServiceManager是用户空间的一个守护进程。当该应用程序启动时，它会和Binder驱动进行通信，告诉Binder驱动它是服务管理者；对Binder驱动而言，它则会新建ServiceManager对应的Binder实体，并将该Binder实体设为全局变量。

​	为什么要将它设为全局变量呢？因为Client和Server都需和ServiceManager进行通信，不将它设为全局变量的话，无法找到ServiceManager



#### Server注册到ServiceManager

​	Server首先会向Binder驱动发起注册请求，而Binder驱动在收到该请求之后就将该请求转发给ServiceManager进程。但是Binder驱动怎么才能知道该请求是要转发给ServiceManager的呢？这是因为Server在发送请求的时候，会告诉Binder驱动这个请求是交给0号Binder引用对应的进程来进行处理的。而Binder驱动中指定了0号引用是与ServiceManager对应的。

​	在Binder驱动转发该请求之前，它其实还做了两件很重要的事：

> **(01) 当它知道该请求是由一个Server发送的时候，它会新建该Server对应的Binder实体。**
>
> 
>
> **(02) 它在ServiceManager的"保存Binder引用的红黑树"中查找是否存在该Server的Binder引用；找不到的话，就新建该Server对应的Binder引用，并将其添加到"ServiceManager的保存Binder引用的红黑树"中。**



​	当ServiceManager收到Binder驱动转发的注册请求之后，它就将该Server的相关信息注册到"Binder引用组成的单链表"中。这里所说的Server相关信息主要包括两部分：**Server对应的服务名** + **Server对应的Binder实体的一个Binder引用**。

#### Client获取远程服务

> Client要和某个Server通信，需要先获取到该Server的远程服务。

​	Client首先会向Binder驱动发起获取服务的请求。Binder驱动在收到该请求之后也是该请求转发给ServiceManager进程。ServiceManager在收到Binder驱动转发的请求之后，会从"Binder引用组成的单链表"中找到要获取的Server的相关信息。

​	至于ServiceManager是如何从单链表中找到需要的Server的呢？答案是Client发送的请求数据中，会包括它要获取的Server的服务名；而ServiceManager正是根据这个**服务名**来找到Server的。

​	接下来，ServiceManager通过Binder驱动将Server信息反馈给Client的。它反馈的信息是Server对应的Binder实体的Binder引用信息。而Client在收到该Server的Binder引用信息之后，就根据该Binder引用信息创建一个Server对应的远程服务。这个远程服务就是Server的代理，Client通过调用该远程服务的接口，就相当于在调用Server的服务接口一样；因为Client调用该Server的远程服务接口时，该远程服务会对应的通过Binder驱动和真正的Server进行交互，从而执行相应的动作。



## Binder的设计

> 通过上面，已经有了Binder模型的理论基础。现在开始学习它的设计思路。

### 两个中心思想

#### 一、Server提供接入点

> ​	如果C-S架构中的Client和Server属于同一进程的话，那么Client和Server之间的通信将非常容易。只需要在Client端先获取相应的Server端对象；然后，再通过Server对象调用Server的相应接口即可。但是，Binder机制中涉及到的Client和Server是位于不同的进程中的，这也就意味着，不可能直接获取到Server对象。**那就需要Server提供一个接入点给Client。**

> **而**这个接入点就是**"Server的远程服务代理"**！

​	Client能够获取到Server的远程服务，它就相当于Server的代理。Client要和Server通信时，它只需要调用该远程服务的相应接口即可，其他的工作都交给远程服务来处理。远程服务收到Client请求之后，会和Binder驱动通信；因为远程服务中有Server在Binder驱动中的Binder引用信息，因此远程服务就能轻易的找到对应的Server，进而将Client的请求内容发送Server。



#### 二、通信协议

> Binder机制中，涉及到大量的"内核的Binder驱动 和 用户空间的引用程序"之间的通信。需要指定对应的通信协议，确保通信的安全和正常。



### 内核空间的设计

> ​	内核空间的Binder设计涉及到3个非常重要的结构体：binder_proc，binder_node和binder_ref。



| binder_proc | 描述进程上下文信息的，每一个用户空间的进程都对应一个binder_proc结构体。 |
| :---------: | :--------------------------------------: |
| binder_node |   Binder实体对应的结构体，它是Server在Binder驱动中的体现   |
| binder_ref  |   Binder引用对应的结构体，它是Client在Binder驱动中的体现   |



![img](https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_kernel_ds01.jpg)



如上图所示，binder_proc中包含了3棵红黑树。
​	(01) Binder实体红黑树是保存"binder_proc对应的进程"所包含的Binder实体的，而Binder实体是与Server的服务对应的。可以将Binder实体红黑树理解为Server进程中包行的Server服务的红黑树。
​	(02) 图中有两棵Binder引用红黑树，这两棵树所包含的Binder引用都是一样的。不同的是，红黑树的排序基准不同，一个是以Binder实体来排序，而另一个则是以Binder引用描述(Binder引用描述实际上就是一个32位的整型数)来排序。以Binder引用描述的红黑树是为了方便进行快速查找。

![img](https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_kernel_ds02.jpg)



​	上图是描述Binder驱动中Binder实体结构体的。如图所示，Binder实体中有一个Binder引用的哈希表，专门来存放该Binder实体的Binder引用。这也如我们之前所说，每个Binder实体则可以多个Binder引用，而每个Binder引用则都只对应一个Binder实体。



### 用户空间的Binder设计

![img](https://raw.githubusercontent.com/wangkuiwu/android_applets/master/os/pic/binder/binder_user_ds01.jpg)

> ​	Server是以服务的形式注册到ServiceManager中，而Server在Client中则是以远程服务的形式存在的。因此，这个图的主干就是理清楚本地服务和远程服务这两者之间的关系。

​	"本地服务"就是Server提供的服务本身，而"远程服务"就是服务的代理；"服务接口"则是抽象出了它们的通用接口。这3个角色都是通用的，对于不同的服务而言，它们的名称都不相同。例如，对于MediaPlayerService服务而言，本地服务就是MediaPlayerService自身，远程服务是BpMediaPlayerService，而服务接口是IMediaPlayerService。当Client需要向MediaPlayerService发送请求时，它需要先获取到服务的代理(即，远程服务对象)，也就是BpMediaPlayerService实例，然后通过该实例和MediaPlayerService进行通信。



图中的ProcessState和IPCThreadState都是采用单例模式实现的，它们的实例都是全局的，而且只有唯一一个。

​	(01) 当Server启动之后，它会先将自己注册到ServiceManager中。注册时，Binder驱动会创建Server对应的Binder实体，并将"Server对应的本地服务对象的地址"保存到Binder实体中。注册成功之后，Server就进入消息循环，等待Client的请求。
​	(02) 当Client需要和Server通信时，会先获取到Server接入点，即获取到远程服务对象；而且Client要获取的远程服务对象是"服务接口"类型的。Client向ServiceManager发送获取服务的请求时，会通过IPCThreadState和Binder驱动进行通信；当ServiceManager反馈之后，IPCThreadState会将ServiceManager反馈的"Server的Binder引用信息"保存BpBinder中(具体来说，BpBinder的mHandle成员保存的就是Server的Binder引用信息)。然后，会根据该BpBinder对象创建对应的远程服务。这样，Client就获取到了远程服务对象，而且远程服务对象的成员中保存了Server的Binder引用信息。 
​	(03) 当Client获取到远程服务对象之后，它就可以轻松的和Server进行通信了。当它需要向Server发送请求时，它会调用远程服务接口；远程服务能够获取到BpBinder对象，而BpBinder则通过IPCThreadState和Binder驱动进行通信。由于BpBinder中保存了Server在Binder驱动中的Binder引用；因此，IPCThreadState和Binder驱动通信时，是知道该请求是需要传给哪个Server的。Binder驱动通过Binder引用找到对应的Binder实体，然后将Binder实体中保存的"Server对应的本地服务对象的地址"返回给用户空间。当IPC收到Binder驱动反馈的内容之后，它从内容中找到"Server对应的本地服务对象"，然后调用该对象的onTransact()。不同的本地服务都可以实现自己的onTransact()；这样，不同的服务就可以按照自己的需求来处理请求。

### 