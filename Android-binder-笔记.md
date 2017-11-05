# 一些基础概念

kernel space：内核，驱动 等，一般是系统必须程序，所在的内存地址

user spspace：非kernel的，一般的Application, Library 运行的内存地址

# 关于进程及内存空间的一些规则:
1. 每个进程所在的 user space 是相互隔离的。
2. user space 的进程，可以调用系统api（system calls），间接操作 kernel space。
3. kernel space 的进程，可以直接操作 user space。通过`uacess`接口。

我有几个地方是有疑问的：

1. kernel space 存在进程吗？还是说，只有 kernel 本身？

    kernel space 包括了 init 进程，及 fork 衍生出的其他进程。fork()->process, clone()->thread。

    Linux 下 thread 相当于 共享资源的 process；因为kerenl process 是 light weight process, 所以一般称为 kernel thread, 而不是 kernel  process

2. kernel space 的进程也是相互隔离的吗？IPC 是怎么样的？

    进程间的IPC是统一的，并没有为 kernel space 的进程使用其他机制。

3. process isolation 是怎么实现的？

    (From Wikipedia: Process isolation)[https://en.wikipedia.org/wiki/Process_isolation]

    > Process isolation can be implemented with virtual address space,
    > where process A's address space is different from process B's
    > address space – preventing A from writing onto B.

    通过 *virtual address space* 控制进程的内存映射到物理地址的过程，避免不同进程的*越权*读写。
    
    另外，内存共享IPC，是kernel可以通过控制不同的虚拟内存映射到相同的物理内存实现。

# Android Binder 的角色

因为 user space 是相互隔离的，所以 user space 的两个 process, 假设为 A, B, 要进行交互&通信，必须存在一个负责连接 A, B 的中间人。

而且，如果不采用传统的IPC，那这个中间人一定是要可以跟user space 的进程通信的，所以这个中间人要是运行在 kernel space 的进程。

Android Binder 就是这个中间人。也就是说 Binder Driver 是运行在 kernel space 的。

至于为什么Android要额外写一个 Binder Framework ，而不是复用Linux 下已有的IPC框架。
这里 (Embedded Linux: Android Binder)[http://elinux.org/Android_Binder] 及文章内的外链有很详细&底层的解释，值得深读。

# Android binder 调用流程及解释

marshaling: convert an object to multi primitive types
unmarshaling: convert multi primitive types to an object

我先约定以下概念及名称：

1. **api**：对外接口
2. **service**：提供api的程序&进程。运行在 user space
3. **client**: service api 的使用者。运行在 user space

client calls `api.hello(msg)` workflow:

```
             (1)                             (2)
userspace  ------->     kernel space      -------->  userspace
client               Binder Framewrok                service
           <-------                       <---------
             (4)                             (3)
```

(1): client call service api through binder,
    binder driver marshaling objects to primitive data, then stores primitive data in kernel space

(2): binder driver unmarshaling primitive data to object, then  call service api passing object params

(3) service api finish, pass result back to binder, binder marshaling object to primitive data, pass primitive datas to kernel space

(4) binder driver read primitive data from kernel space, then marshaling back to object, pass object to client

so we can define api result or params in complex data types, not only primitive types

## 核心原理

1. Binder 机制的 marshaling 及 unmarshaling

    marshalling: convert user space object to primitive types. 然后，Binder Driver 把 primitive types data 保存到 kernel space
    unmarshalling: Composite kernel space primitive types to object. 然后，把 object copy 到 user space.

    大概理解为，Binder 把 process A 需传输的对象 复制，保存到 kernel space， 再从 kernel space 复制 到 process B 的内存空间。

    kernel space 作为一个数据传输的中转站。

2. user space process <-- data --> kernel space 

    `ioct`: user space 传输数据到 kernel space 的 Binder Driver

    `uacess`： kernel space 的 Binder Driver 把数据传给 user space

# 参考文档

1. [必读-Deep Dive into Android IPC/Binder Framework at Android Builders Summit 2013](https://events.linuxfoundation.org/images/stories/slides/abs2013_gargentas.pdf)
10. [必读-User space memory access from the Linux kernel](http://www.ibm.com/developerworks/library/l-kernel-memory-access/)
3. [必读-Android Binder && Android Interprocess Communication](http://www.nds.rub.de/media/attachments/files/2012/03/binder.pdf)
9. [推荐-Gityuan: Binder 系列](http://gityuan.com/2015/10/31/binder-prepare/)
2. [基础概念-ELinux: Android Binder](http://elinux.org/Android_Binder)
4. [基础概念-Marshalling](https://en.wikipedia.org/wiki/Marshalling_%28computer_science%29)
5. [基础概念-Kernel space](http://www.linfo.org/kernel_space.html)
6. [基础概念-User space](http://www.linfo.org/user_space.html)
7. [基础概念-Kernel space vs User space](http://www.ruanyifeng.com/blog/2016/12/user_space_vs_kernel_space.html)
8. [基础概念-IOCT](https://en.wikipedia.org/wiki/Ioctl)