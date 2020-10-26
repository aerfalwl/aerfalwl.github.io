## Introduction to the Kernel

1. 参考书籍The Design and Implementation of the FreeBSD Operating System, 2nd
   Edition

# Introduction to the Zettabyte Filesystem

1. 开始于Solaries。
2. Solaris是Sun Microsystem开发的一个Unix版本的操作系统；
   - 它是以下两种操作系统的合并版：SunOs和System V4
3. Sun需要解决的问题是：传统的操作系统在处理大数据的时候，显现出了很大的缺点，因为大数据使得系统需要大量的磁盘。
4. 为解决以上问题，Sun写了ZFS:
   - Log Structured Filesystem，由Berkeley开发，展现出了它处理大问题大数据的一些优势
   - 文件系统不可重复写的特性，给了它很多比较好的特性。

# ZFS的演变过程

1. 首先出现于OpenSolaris,  该操作系统是一款Solaris开源的操作系统，包括核心科技：ZFS和DTrace
2. 随后，ZFS出现在了FreeBSD（一款Unix操作系统）中；
3. 之后，Sun被Oracle收购，但是Larry Ellison不是一个开源爱好者，他不同意ZFS继续开源，但是已经开源的早期版本还在流传；
4. 于是一些公司继续开发并维护OpenSolaris，并且将他作为一个开源项目；
5. 一些组件变得很流行，像ZFS；
6. 最后ZFS变为单独的OpenZFS项目；
7. 当前开发OpenZFS的主要有四个组，它们每个月都有一次会议来协同工作：
   - 致力于OpenSolaris开发的组；
   - FressBSD开发者；
   - 致力于将ZFS嵌入到Linux内核的Lawrence Livermore National Laboratory
   - 致力于将ZFS嵌入到MacOs和Windows的Jorgen Lundman
8. Sun Community Development License与GNU Public License不兼容：
   - ZFS代码不能嵌入到Linux内核中；
   - 所有将ZFS带入嵌入到Linux内核的人不能使用GPL interfaces（GPL Interface是Linux的一个特性，该特性说明只有GPL的代码之间才能相互使用和修改）
   - ZFS只能通过FUSE(用户态空间文件系统， Filesystem in Userspace)访问Linux Kernel。
9. 在2016年，Ubuntu开源了一版在Kernel态的ZFS，它没有使用GPL interfaces，因此，自己额外开发了一些函数。当前，Canonical（开发Ubuntu的公司）还没被起诉，更多详情访问https://wiki.ubuntu.com/ZFS
10. 当前，OpenZFS代码已经能在大多数Linux系统中配置使用。

ZFS相关资料：https://openzfs.org/wiki/Main_Page

Github网址：https://github.com/openzfs/zfs

# Kernel I/O Structure

1. 关于kernel的系统调用：
   - 一切在硬件和系统调用之间的都是操作系统(everything between this and the hardware is the operating system)。
   - 当我们和subsystem进行交互时，其实是在和file descriptor、socket、kqueue交互

![image-20201014101848952](C:\Users\CCDC\AppData\Roaming\Typora\typora-user-images\image-20201014101848952.png)

# FileSystem Consistency

1. 必须维护一些元数据信息：directories, inodes, bitmaps
2. 维护元数据的一些规则：
   - 在一个结构被初始化之前不要指向它
   - 在将一个对象的所有指针置为空之前，不要使新指针指向它
   - 在新指针被设置之前，不要讲老指针设置为一个可用的资源(例如，rename foo to bar，则在将foo的旧指针指向bar之后，再删除老的foo)

# Keeping Metadata Consistent

实现一致性的三种方法：

1. 同步写
2. 使用非易失性RAM
3. 原子更新(journaling and logging)

