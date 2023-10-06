---
title: File system
date: 2023-10-05 14:56:42
tags: 操作系统，计算机基础, C
categories: 操作系统
---

1. file abstraction

2. Crash safety：遭遇某种宕机或者突然关机，原文件还存在

3. Disk layout：文件是如何在硬盘上存储的

4. Performance

   持久化的存储设备读取速率通常很慢，需要buffer和concurrency

   

## API Example

fd = open("x/y", _)，文件名是人类可读的

write(fd, "abc", 3)，写入文件的时候，是不存在偏移量的，偏移量是隐式的，需要文件抽象来存储

link("x/y", "x/z"), 建立链接，相当于对同一个文件有多个命名

unlink("x/y")，删除之前的引用

write(fd, "def", 3)，此时仍然可以写入

所以，fd是和文件名无关的对象，

存储信息的组织方式不只是本次课的一种，还有很多方法，比如B+, 红黑树，但本节课是最简单实现的一种



## Inode

Inode <- file info, independent of name

Inode #(仅仅是个数字)

对Inode必须有链接计数，当链接计数为0时且打开文件的fd为0时才能删除对应文件，fd必须维护每次写入后的偏移量

FS layers

<img src="C:\Users\Mmjiang\Desktop\oslearning\mit-6.s081\filesys\fs-layers.png" alt="fs-layers" style="zoom:50%;" />



## Storage Device

从下面开始讲：

1. 存储设备

   SSD 和HDD

对于存储设备有个存储划分的概念，且存储设备有对应的分区：扇区(512)和块区(1024B)，这两个是对存储设备不同的划分单位

CPU <- memory

write /read PCI(驱动)

SSD HDD



disk layout：xv6按照块对磁盘进行划分，一个块1024字节

<img src="C:\Users\Mmjiang\Desktop\oslearning\mit-6.s081\filesys\disk-layout.png" alt="disk-layout" style="zoom:50%;" />

给定inode，可以计算出在哪个块里，eg:10 -> 32 + 10 * 64 / 1024 = 32，所以在编号为32的块中
inode 17呢 => 33





<img src="C:\Users\Mmjiang\Desktop\oslearning\mit-6.s081\filesys\无标题-2023-08-26-1159.png" alt="无标题-2023-08-26-1159" style="zoom:50%;" />

block 12可以存放(1024/4 = 256，4是block index大学)个block index，所以xv6里硬盘最大存储(256 + 12) * 1024 bytes大小

如果想要扩展存储最大空间的话，可以加double indirect block index:

index => block index => block index

eg: 给定字节，怎么知道是哪个inode
read(8000) = 8000 / 1024 = 7
8000 % 1024  = 832

由于7小于direct index，所以直接可以查到，然后取偏移量为832处的数据



目录：xv6中目录即文件

目录由很多entry组成，每个entry由：entry: inode num(2 bytes) + filename (14 bytes)，

eg: 查找y/x

从root inode(1)开始查找，在block 32这里，第64~128(32 + 1 * 64 / 1024 = 32, offset = 1*64 % 1024 = 64) =>scans block for name 'y' => 如果是目录然后遍历它的block是寻找文件名为y的entry，其inode为251，继续向下寻找

![image-20230909201628672](C:\Users\Mmjiang\AppData\Roaming\Typora\typora-user-images\image-20230909201628672.png)



## 代码演示

```
create file
write: 33 (block 33)
write: 33 (filling inode)
write: 46 (data block, root目录的一个块)
write: 32 (update inode 1)
write: 33  (update new inode)

hi
write: 45 (bitmap)
write: 595 (write bitmap to 595)
write: 595 (write into block)
write: 33 (update new inode)

\n
write: 33
write: 33
```





你可以使用名叫E1000 的网络设备处理网络通信。对于xv6来说，E1000看起来像是连接到真实局域网的一块硬件设备。实际上，E1000在quemu中是一种软件模拟的形式。在QEMU模拟的局域网里，xv6作为客户端它的IP地址10.0.2.15，同时还有另一台与之通信的IP为10.0.2.2的主机。当xv6使用E1000给10.0.2.2发送数据包，qemu传递该数据包给合适的应用	



## Crash Safety

Crash can lead the on-disk fs to be incorrect adn inconsistent state.

解决方案：logging



### RIsks： fs operation are multipl steps disk operation. crash may leave fs invariants violated(破坏文件系统的不变性)

重启后：crash  again, or no crash but w/w data incorrectly



case: xv6

bitmap: 用于记录哪些块是空闲的，哪些不是

echo "hi" > x

```
write: 33 (allocate inode for x)
write: 33 (init inode x)
write: 46 (record x in / directory's data block)
write: 32 (update root inode)
write: 33  (update inode x)

单个文件的创建由多步组成


```

<img src="C:\Users\Mmjiang\Desktop\oslearning\mit-6.s081\filesys\write1.png" alt="write1" style="zoom:50%;" />

可以不看通过调整顺序呢？

46 32 33 33 33

可以避免上面的问题，但是会有新的问题，比如在32处发生崩溃，此时OS可见的是未分配的Inode，所以导致该inode后续被用在了其他的文件里，即一个inode被多个文件共享。造成的直观影响是一个用户读写一个文件会影响其他用户。



45(set alloc bitmap block)

595(update h to allocate data block)

595(update i to allocate data block)

33(size update)

修改顺序：33 45，在33后发生崩溃，size发生改变，但实际大小却不是，且在bitmap中595没有被记录属于当前文件的；所以该block后续也会被分配到其他文件，导致共享。

出现问题的本质原因在于：fs operation不是以原子性对磁盘进行操作，并不是由于操作的顺序所导致的。



解决方法：logging，用日志延迟更新磁盘的操作，将多次写操作组成一个原子性操作，称为事务。

1. atomic fs calls
2. fast recovery
3. high performance

(本次课程只提到了write，不涉及文件的append操作)

磁盘：



内存：





有了logging之后，磁盘的更新步骤如下：

1. log writes (不做实际写入操作)
2. commit (提交一次事务到磁盘，更新磁盘的logging block)
3. install (更新磁盘的data block)
4. clean log



事务帮助多个写入操作成为原子性的，

```
事务的开始(begin_op())

事务的结束(end_op())

结束之后才会生成commit，xv6中的文件系统调用都有这样的代码结构
```



bitmap：用于记录哪些data block在被使用中



三个注意点：

1. bcache不能驱逐
2. 单次commit不能太大，超过logging block大小，需要分割
3. 并发的情况下，如果多个事务同时参与commit，如果本次commit的大小太大，需要睡眠部分进程，等待上一个事务结束之后再恢复



## 文件系统

### 概述

出于以下原因，文件系统背后的机制还比较有意思：

- 文件系统对硬件的抽象较为有用，所以理解文件系统对于硬件的抽象是如何实现的还是有点意思的。
- 除此之外，还有个关键且有趣的地方就是crash safety。有可能在文件系统的操作过程中，计算机崩溃了，在重启之后你的文件系统仍然能保持完好，文件系统的数据仍然存在，并且你可以继续使用你的大部分文件。如果文件系统操作过程中计算机崩溃了，然后你重启之后文件系统不存在了或者磁盘上的数据变了，那么崩溃的将会是你。所以crash safety是一个非常重要且经常出现的话题，我们下节课会专门介绍它。
- 之后是一个通用的问题，如何在磁盘上排布文件系统。例如目录和文件，它们都需要以某种形式在磁盘上存在，这样当你重启计算机时，所有的数据都能恢复。所以在磁盘上有一些数据结构表示了文件系统的结构和内容。在XV6中，使用的数据结构非常简单，因为XV6是专门为教学目的创建的。真实的文件系统通常会更加复杂。但是它们都是磁盘上保存的数据结构，我们在今天的课程会重点看这部分。
- 最后一个有趣的话题是性能。文件系统所在的硬件设备通常都较慢，比如说向一个SSD磁盘写数据将会是毫秒级别的操作，而在一个毫秒内，计算机可以做大量的工作，所以尽量避免写磁盘很重要，我们将在几个地方看到提升性能的代码。比如说，所有的文件系统都有buffer cache或者叫block cache。同时这里会有更多的并发，比如说你正在查找文件路径名，这是一个多次交互的操作，首先要找到文件结构，然后查找一个目录的文件名，之后再去查找下一个目录等等。你会期望当一个进程在做路径名查找时，另一个进程可以并行的运行。这样的并行运行在文件系统中将会是一个大的话题。

文件系统究竟维护了什么样的结构？

- 首先，最重要的可能就是inode，这是代表一个文件的对象，并且它不依赖于文件名。实际上，inode是通过自身的编号来进行区分的，这里的编号就是个整数。所以文件系统内部通过一个数字，而不是通过文件路径名引用inode。同时，基于之前的讨论，inode必须有一个link count来跟踪指向这个inode的文件名的数量。一个文件（inode）只能在link count为0的时候被删除。实际的过程可能会更加复杂，实际中还有一个openfd count，也就是当前打开了文件的文件描述符计数。一个文件只能在这两个计数器都为0的时候才能被删除。

文件系统中核心的数据结构就是inode和file descriptor。后者主要与用户进程进行交互。

尽管文件系统的API很相近并且内部实现可能非常不一样。但是很多文件系统都有类似的结构。因为文件系统还挺复杂的，所以最好按照分层的方式进行理解(图3-2)。可以这样看：

- 在最底层是磁盘，也就是一些实际保存数据的存储设备，正是这些设备提供了持久化存储。
- 在这之上是buffer cache或者说block cache，这些cache可以避免频繁的读写磁盘。这里我们将磁盘中的数据保存在了内存中。
- 为了保证持久性，再往上通常会有一个logging层。许多文件系统都有某种形式的logging，我们下节课会讨论这部分内容，所以今天我就跳过它的介绍。
- 在logging层之上，XV6有inode cache，这主要是为了同步（synchronization），我们稍后会介绍。inode通常小于一个disk block，所以多个inode通常会打包存储在一个disk block中。为了向单个inode提供同步操作，XV6维护了inode cache。
- 再往上就是inode本身了。它实现了read/write。
- 再往上，就是文件名，和文件描述符操作。

![img](https://tnrhmtc0g9.feishu.cn/space/api/box/stream/download/asynccode/?code=NWQ0YjFkYzFlNDg4NTk4NDIyMDc3NTBhNjM3ZDgwZmFfZXBqRmJIeWp4d2x5V3dpODBKcVZpYTliVXpYSkIzSEVfVG9rZW46SHlDRmJFb1hHb01GSDl4a3Q2N2NoOHRqbnpmXzE2OTY1NzQ1OTc6MTY5NjU3ODE5N19WNA)

(图3-1: filesystem layering)

### inode

当今的Unix文件系统(Unix File System, UFS)起源于Berkeley Fast File System。和所有的文件系统一样，Unix文件系统是以块(Block)为单位对磁盘进行读写的。一般而言，一个块的大小为512bytes或者1024bytes。文件系统的所有数据结构都以块为单位存储在硬盘上，一些典型的数据块包括：

- Super block：Superblock包含了关于整个文件系统的元信息(metadata)，比如文件系统的类型、大小、状态和关于其他文件系统数据结构的信息。Superblock对文件系统是非常重要的，因此Unix文件系统的实现会保存多个Superblock的副本。
- Inode block：inode是Unix文件系统中用于表示文件的抽象数据结构。inode不仅是指抽象了一组硬盘上的数据的”文件”，目录和外部IO设备等也会用inode数据结构来表示。inode包含了一个文件的元信息，比如拥有者、访问权限、文件类型等等。对于一个文件系统里的所有文件，文件系统会维护一个inode列表，这个列表可能会占据一个或者多个磁盘块（见下图3-1）。
- Data block：Data block用于存储实际的文件数据。一些文件系统中可能会存在用于存放目录的Directory Block和Indirection Block，但是在Unix文件系统中这些文件块都被视为数据，上层文件系统通过inode对其加以操作，他们唯一的区别是inode里记录的属性有所不同。
- Directory block
- Indirection block

![img](https://tnrhmtc0g9.feishu.cn/space/api/box/stream/download/asynccode/?code=MmZmY2I0ZDgxYjE4MTNiMzQ0NGU1ZGM3ODM1YzJkMzJfSERudVhNNjVQRlZYeE1xRUVIN1lQVVhoUjNJTXJRRWhfVG9rZW46VVhwamJOS1NRb0NyQzF4MElyU2N2cGxlblpnXzE2OTY1NzQ1OTc6MTY5NjU3ODE5N19WNA)

(3-2: inode)

block 12可以存放(1024/4 = 256，4是block index本身大小)个block index，所以xv6里硬盘最大存储(256 + 12) * 1024 bytes大小

如果想要扩展存储最大空间的话，可以加double indirect block index

### 存储设备与硬盘上的文件系统布局

通常来说，xv6的硬件上文件布局如下：

- block0要么没有用，要么被用作boot sector来启动操作系统。
- block1通常被称为super block，它描述了文件系统。它可能包含磁盘上有多少个block共同构成了文件系统这样的信息。我们之后会看到XV6在里面会存更多的信息，你可以通过block1构造出大部分的文件系统信息。
- 在XV6中，log从block2开始，到block32结束。实际上log的大小可能不同，这里在super block中会定义log就是30个block。
- 接下来在block32到block45之间，XV6存储了inode。我之前说过多个inode会打包存在一个block中，一个inode是64字节。
- 之后是bitmap block，这是我们构建文件系统的默认方法，它只占据一个block。它记录了数据block是否空闲。
- 之后就全是数据block了，数据block存储了文件的内容和目录的内容。

通常来说，bitmap block，inode blocks和log blocks被统称为metadata block。它们虽然不存储实际的数据，但是它们存储了能帮助文件系统完成工作的元数据。

> 学生提问：boot block是不是包含了操作系统启动的代码？
>
> Frans教授：完全正确，它里面通常包含了足够启动操作系统的代码。之后再从文件系统中加载操作系统的更多内容。
>
> 学生提问：所以XV6是存储在虚拟磁盘上？
>
> Frans教授：在QEMU中，我们实际上走了捷径。QEMU中有个标志位-kernel，它指向了内核的镜像文件，QEMU会将这个镜像的内容加载到了物理内存的0x80000000。所以当我们使用QEMU时，我们不需要考虑boot sector。
>
> 学生提问：所以当你运行QEMU时，你就是将程序通过命令行传入，然后直接就运行传入的程序，然后就不需要从虚拟磁盘上读取数据了？
>
> Frans教授：完全正确。

![img](https://tnrhmtc0g9.feishu.cn/space/api/box/stream/download/asynccode/?code=YmZjODk4YzY2YmVjMGFmZjU4ZTk5YzRiYWI5N2M0ZDFfakV5eWFEMnhONEROUmdKYVMwRGszeDRmZ2FibmJUMnZfVG9rZW46T3pmaWJzdUxSb1N4aFJ4RDBuR2NZMzNwbkc5XzE2OTY1NzQ1OTc6MTY5NjU3ODE5N19WNA)

给定字节，怎么知道是哪个inode？ read(8000) = 8000 / 1024 = 7 8000 % 1024  = 832

由于7小于direct index，所以直接可以查到，然后取偏移量为832处的数据。

**目录**：xv6中目录即文件

目录由很多entry组成，每个entry由：entry: inode num(2 bytes) + filename (14 bytes)，

eg: 查找y/x

从root inode(1)开始查找，在block 32这里，第64~128(32 + 1 * 64 / 1024 = 32, offset = 1*64 % 1024 = 64) =>scans block for name 'y' => 如果是目录然后遍历它的block是寻找文件名为y的entry，其inode为251，继续向下寻找。

### 代码

下面以open为例看下大致的与磁盘交互的过程：

#### sys_open 

首先是sys_open函数：就是参数校验调用begin_op

```C++
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  if((n = argstr(0, path, MAXPATH)) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }
```

#### begin_op

判断是否已经处于committing status，如果是则睡眠，除此之外日志太大也睡眠

```C++
// file.c

// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    if(log.committing){
      sleep(&log, &log.lock);
      // MAXOPBLOCKS为10，为什么呢？
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```

#### Create

接下来是create函数：主要就是查找父级目录，查找文件是否存在，如果存在直接返回，否则调用ialloc分配空间

```C++
// fs.c
struct inode*
namei(char *path)
{
  char name[DIRSIZ
  return namex(path, 0, name);
}

struct inode*
nameiparent(char *path, char *name)
{
  return namex(path, 1, name); // 默认从1，root目录开始
}
static struct inode*
namex(char *path, int nameiparent, char *name) // nameiparent是父级目录的inode编号
{
  struct inode *ip, *next;

  if(*path == '/')
    // 如果创建目录是root
    ip = iget(ROOTDEV, ROOTINO);
  else 
    ip = idup(myproc()->cwd); // 获取当前所在目录，如果是在/下创建的，就是/的inode num

  while((path = skipelem(path, name)) != 0){
    ilock(ip);
    if(ip->type != T_DIR){
      iunlockput(ip);
      return 0;
    }
    if(nameiparent && *path == '\0'){
      // Stop one level early.
      iunlock(ip);
      return ip;
    }
    if((next = dirlookup(ip, name, 0)) == 0){
      iunlockput(ip);
      return 0;
    }
    iunlockput(ip);
    ip = next;
  }
  if(nameiparent){
    iput(ip);
    return 0;
  }
  return ip; // 返回/(父级)的inode
}

// Look for a directory entry in a directory.
// If found, set *poff to byte offset of entry.
struct inode*
dirlookup(struct inode *dp, char *name, uint *poff)
{
  uint off, inum;
  struct dirent de; 

  if(dp->type != T_DIR)
    // 父级目录不是目录
    panic("dirlookup not DIR");
  // de是16byte，对应一个entry的大小
  for(off = 0; off < dp->size; off += sizeof(de)){
    // dir entry
    if(readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlookup read");
    if(de.inum == 0)
      continue;
    if(namecmp(name, de.name) == 0){ 
      // entry matches path element，有一个entry匹配到了，说明找到了，目录的data就是entry
      if(poff)
        *poff = off;
      inum = de.inum;
      return iget(dp->dev, inum);
    }   
  }

  return 0;
}

static struct inode*
create(char *path, short type, short major, short minor)
{
  struct inode *ip, *dp;
  char name[DIRSIZ];

  if((dp = nameiparent(path, name)) == 0)
    // 查找父级目录，这里说明没找到，返回父级目录的inode
    return 0;

  ilock(dp); // 加锁，其他进程无法访问父级目录的inode，如果不加锁，其他进程操作该文件，可能会引起异常

  if((ip = dirlookup(dp, name, 0)) != 0){ 
    // 在父级目录里查找该文件，如果文件存在，返回找到的indoe
    iunlockput(dp); // 释放锁
    ilock(ip); // 加锁，其他进程无法访问目标文件inode
    if(type == T_FILE && (ip->type == T_FILE || ip->type == T_DEVICE))
      return ip; 
    iunlockput(ip); // 释放锁
    return 0;
  }

  if((ip = ialloc(dp->dev, type)) == 0)
    panic("create: ialloc");

  ilock(ip); // 加锁，其他进程无法访问目标文件inode
  ip->major = major;
  ip->minor = minor;
  ip->nlink = 1;
  iupdate(ip);

  if(type == T_DIR){  // Create . and .. entries.
    dp->nlink++;  // for ".."
    iupdate(dp);
    // No ip->nlink++ for ".": avoid cyclic ref count.
    if(dirlink(ip, ".", ip->inum) < 0 || dirlink(ip, "..", dp->inum) < 0)
      panic("create dots");
  }

  if(dirlink(dp, name, ip->inum) < 0)
    panic("create: dirlink");

  iunlockput(dp);

  return ip; 
}
```

#### Ialloc

首先调用bread获取缓存inode，如果缓存没有就分配最近未使用的（LRU链表）（这里会加一个睡眠锁，仅允许当前进程使用），找到后返回一个block对应的buf节点，在分配的缓存节点里找适合的位置（合适的位置=inode#/64 + start_addr），然后log_write就可以写日志了，在日志里记录当前正在写的block。

它很简单，但是又不是很高效。它会遍历所有可能的inode编号，找到inode所在的block，再看位于block中的inode数据的type字段。如果这是一个空闲的inode，那么将其type字段设置为文件，这会将inode标记为已被分配。

以上就是第一次写磁盘涉及到的函数调用。这里有个有趣的问题，如果有多个进程同时调用create函数会发生什么？对于一个多核的计算机，进程可能并行运行，两个进程可能同时会调用到ialloc函数，然后进而调用bread（block read）函数。所以必须要有一些机制确保这两个进程不会互相影响。

> 是的，我们这里看一下目标block no的cache是否存在，如果存在的话，将block对象的引用计数（refcnt）加1，之后再释放bcache锁，因为现在我们已经完成了对于cache的检查并找到了block cache。之后，代码会尝试获取block cache的锁。
>
> 所以，如果有多个进程同时调用bget的话，**其中一个可以获取bcache的锁并扫描buffer cache。此时，其他进程是没有办法修改buffer cache的（注，因为bacche的锁被占住了）。**之后，进程会查找block number是否在cache中，如果在的话将block cache的引用计数加1，表明当前进程对block cache有引用，之后再释放bcache的锁。如果有第二个进程也想扫描buffer cache，那么这时它就可以获取bcache的锁。假设第二个进程也要获取该block的cache，那么它也会对相应的block cache的引用计数加1。最后这两个进程都会尝试对block 33的block cache调用acquiresleep函数。
>
> acquiresleep是另一种锁，我们称之为sleep lock，本质上来说它获取目标 block  cache的锁。其中一个进程获取锁之后函数返回。

如果buffer cache中有两份block 33的cache将会出现问题。假设一个进程要更新inode19，另一个进程要更新inode20。如果它们都在处理block 33的cache，并且cache有两份，那么第一个进程可能持有一份cache并先将inode19写回到磁盘中，而另一个进程持有另一份cache会将inode20写回到磁盘中，并将inode19的更新覆盖掉。所以一个block只能在buffer cache中出现一次。你们在完成File system lab时，必须要维持buffer cache的这个属性。

```JavaScript
struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE]; BSIZE = 1024
};

// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;
  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}


// 分配inode
struct inode*
ialloc(uint dev, short type)
{
  int inum;
  struct buf *bp;
  struct dinode *dip;

  for(inum = 1; inum < sb.ninodes; inum++){
    bp = bread(dev, IBLOCK(inum, sb));
    dip = (struct dinode*)bp->data + inum%IPB; 
    if(dip->type == 0){  // a free inode
      memset(dip, 0, sizeof(*dip));
      dip->type = type;
      log_write(bp);   // mark it allocated on the disk
      brelse(bp); // 释放睡眠锁
      return iget(dev, inum);
    }   
    brelse(bp);
  }
  panic("ialloc: no inodes");
}

// Return a locked buf with the contents of the indicated block.
// bread每次只能拿到一个block
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b; 

  b = bget(dev, blockno); // 根据block获取缓存的对应的buf节点
  if(!b->valid) { // 如果buf上之前没有数据
    virtio_disk_rw(b, 0); // 从disk上获取数据，完成一次真正的读写
    b->valid = 1;
  }
  return b;
}


关于brelese：
这个函数会对refcnt减1，并释放sleep lock。这意味着，如果有任何一个其他进程正在等待使用这个block cache，现在它就能获得这个block cache的sleep lock，并发现刚刚做的改动。
假设两个进程都需要分配一个新的inode，且新的inode都位于block 33。如果第一个进程分配到了inode18并完成了更新，那么它对于inode18的更新是可见的。另一个进程就只能分配到inode19，因为inode18已经被标记为已使用，任何之后的进程都可以看到第一个进程对它的更新。
这正是我们想看到的结果，如果一个进程创建了一个inode或者创建了一个文件，之后的进程执行读就应该看到那个文件
```

#### log_write

log_write主要是在log缓存里记录了当前正在写的block（所以一次commit仅一个block，也仅一个文件）

```JavaScript
// 记录本次操作的日志
log_write(struct buf *b)
{
  int i;

  acquire(&log.lock); // 操作日志需要加锁
  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
      // 日志的数量最大30，或者大于本身该日志所能承受的大小
    panic("too big a transaction");
  if (log.outstanding < 1) 
    panic("log_write outside of trans");

  for (i = 0; i < log.lh.n; i++) {
    // 寻找目标block
    if (log.lh.block[i] == b->blockno)   // log absorption
      break;
  }
  在header的数组里记录block#
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // 如果是新增的日志，则n++
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}
```

#### commit

最后回到create函数，调用end_op，进而调用commit，commit会连续调用不同的函数，分别是用 header 缓存更新log缓存，接着用更新后的log缓存做真正的提交；用log缓存更新 header 缓存；清空log

- write_log：基本上就是将所有存在于内存中的log header中的block编号对应的block写入到磁盘上的log区域中（注，也就是将变化先从内存拷贝到log中）。函数中依次遍历log中记录的block，并写入到log中。它首先读出log block，将cache中的block拷贝到log block，最后再将log block写回到磁盘中。这样可以确保需要写入的block都记录在log中。但是在这个位置，我们还没有commit，现在我们只是将block存放在了log中。如果我们在这个位置也就是在write_head之前crash了，**那么最终的表现就像是transaction从来没有发生过，文件不会有任何变化同步到disk上。**
- write_head：会将内存中的log header写入到磁盘的log header中。首先读取log的header block。将n拷贝到block中，将所有的block编号拷贝到header的列表中。最后再将header block写回到磁盘。函数中的倒数第2行，bwrite是实际的commit point吗？如果crash发生在这个bwrite之前，会发生什么？**这时虽然我们写了log的header block，但是数据并没有落盘。所以crash并重启恢复时，并不会发生任何事情。**那crash发生在bwrite之后会发生什么呢？**这时header会写入到磁盘中，当重启恢复相应的文件系统操作会被恢复。在恢复过程的某个时间点，恢复程序可以读到log header并发现比如说有5个log还没有install，恢复程序可以将这5个log拷贝到实际的位置。所以这里的bwrite就是实际的commit point。在commit point之前，transaction并没有发生，在commit point之后，只要恢复程序正确运行，transaction必然可以完成。**
- install_trans：这里先读取log block，再读取文件系统对应的block。将数据从log拷贝到文件系统，最后将文件系统block缓存落盘。这里实际上就是将block数据从log中拷贝到了实际的文件系统block中。当然，可能在这里代码的某个位置会出现问题，但是这应该也没问题，因为在恢复的时候，我们会从最开始重新执行过。
- 最后write_log：会将log header中的n设置为0，再将log header写回到磁盘中。将n设置为0的效果就是清除log，晴空磁盘额的log。

```JavaScript
void
end_op(void)
{
  int do_commit = 0;

  // 枷锁
  acquire(&log.lock); 
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;
    log.committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}

static void
write_log(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *to = bread(log.dev, log.start+tail+1); // log block
    struct buf *from = bread(log.dev, log.lh.block[tail]); // cache block
    memmove(to->data, from->data, BSIZE);
    bwrite(to);  // write the log to disk
    brelse(from);
    brelse(to);
  }
}

static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // 用 header 缓存更新log缓存
    write_head();    // 用更新后的log缓存 -- the real commit
    install_trans(0); // 用log缓存更新 header 缓存
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}


// file.h
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
    
// Contents of the header block, used for both the on-disk header block
// and to keep track in memory of logged block# before commit.
struct logheader {
  int n; 一般用于判断是否真的需要往disk里写东西
  int block[LOGSIZE]; 30
};

BSIZE 1024
LOGSIZE 30
MAXOPBLOCKS 10

struct log {
  struct spinlock lock;
  int start;
  int size;
  int outstanding; // how many FS sys calls are executing，可以理解为往disk输送的调用有几个.
  int committing;  // in commit(), please wait.
  int dev;
  struct logheader lh; 
};
struct log log;

// Read the log header from disk into the in-memory log header
static void
read_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *lh = (struct logheader *) (buf->data);
  int i;
  log.lh.n = lh->n;
  for (i = 0; i < log.lh.n; i++) {
    log.lh.block[i] = lh->block[i];
  }
  brelse(buf);
}

// Write in-memory log header to disk.
// This is the true point at which the
// current transaction commits.
static void
write_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *hb = (struct logheader *) (buf->data);
  int i;
  hb->n = log.lh.n;
  for (i = 0; i < log.lh.n; i++) {
    hb->block[i] = log.lh.block[i];
  }
  bwrite(buf);
  brelse(buf);
}

// 更新缓存里的header
static void
install_trans(int recovering)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block，这里的log.start+tail+1是log block no
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst，因为这里的log.lh.block[tail]就是文件系统数据block no
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    if(recovering == 0)
      bunpin(dbuf);
    brelse(lbuf);
    brelse(dbuf);
  }
}
```

看下write_head，bread会往该缓存blokc buf加睡眠锁，该睡眠锁持续时间挺长，然后bwrite正在写入buf数据到disk，最好释放该睡眠锁。

如果这里不用睡眠锁，使用自旋锁可以吗？

不可以，首先这里锁的加的时间挺长，和磁盘交互时间一般是比较大的，也就是说这段时间不会让出cpu，但cpu下导致其他进程被迫等待；且自旋锁加锁期间不允许中断，这意味着bread和bwrite将永远无法完成和disk的交互

```JavaScript
static void
write_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *hb = (struct logheader *) (buf->data);
  int i;
  hb->n = log.lh.n;
  for (i = 0; i < log.lh.n; i++) {
    hb->block[i] = log.lh.block[i];
  }
  bwrite(buf);
  brelse(buf);
}
```

关于缓存链表：block cache的实现，这对于性能来说是至关重要的，因为读写磁盘是代价较高的操作，可能要消耗数百毫秒，而block cache确保了如果我们最近从磁盘读取了一个block，那么我们将不会再从磁盘读取相同的block。

brelese函数中首先释放了sleep lock；之后获取了bcache的锁；之后减少了block cache的引用计数，表明一个进程不再对block cache感兴趣；最后如果引用计数为0，那么它会修改buffer cache的linked-list，将block cache移到linked-list的头部，这样表示这个block cache是最近使用过的block cache。这一点很重要，当我们在bget函数中不能找到block cache时，我们需要在buffer cache中腾出空间来存放新的block cache，这时会使用LRU（Least Recent Used）算法找出最不常使用的block cache，并撤回它（注，而将刚刚使用过的block cache放在linked-list的头部就可以直接更新linked-list的tail来完成LRU操作）。为什么这是一个好的策略呢？因为通常系统都遵循temporal locality策略，也就是说如果一个block cache最近被使用过，那么很有可能它很快会再被使用，所以最好不要撤回这样的block cache。

以上就是对于block cache代码的介绍。这里有几件事情需要注意：

- **首先在内存中，对于一个block只能有一份缓存。这是block cache必须维护的特性。**
- 其次，这里使用了与之前的spinlock略微不同的sleep lock。与spinlock不同的是，可以在I/O操作的过程中持有sleep lock。
- 第三，它采用了LRU作为cache替换策略。
- 第四，**它有两层锁。第一层锁用来保护buffer cache的内部数据；第二层锁也就是sleep lock用来保护单个block的cache。**

![img](https://tnrhmtc0g9.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDM5MmY4ZmJlOGQyZmY2YmQ5ZGQzMDVhNjk0ODkzYjhfbFZmNHpuUnZ4QXVRT0V2akJuUmlIeVJ6OVEwSnNUeGJfVG9rZW46UDZ6TmJTeGJkb0JMMEF4eGt3cmNxWnJVbnhjXzE2OTY1NzQ1OTc6MTY5NjU3ODE5N19WNA)

以创建为例，其整体的调用链路如下：

![img](https://tnrhmtc0g9.feishu.cn/space/api/box/stream/download/asynccode/?code=MzYzZWQ0NTkwN2M2MWQ0NmVlMjYwYmY5NmY4OGMzOWFfUEVaWEVMTFFoNWVJemszRVVLdzE5dERhV1BEeDBLRWlfVG9rZW46SllybmJSY2Fnb3FUMlF4eTg2S2NQM2c0blVkXzE2OTY1NzQ1OTc6MTY5NjU3ODE5N19WNA)

关键在于

1. 每一个系统调用之间都会有begin_op和end_op，begin_op表明想要开始一个事务，在最后有end_op表示事务的结束。并且事务中的所有写block操作具备原子性，这意味着这些写block操作要么全写入，要么全不写入。
2. 和文件相关的和inode的代码都在bio.c里，比如创建文件是ialloc，写入文件是walloc
3. inode对应buffer cache linklist里的一个buf节点，其数据存在data成员上，其存储的block再blockno上
4. 每一次commit都是commit一个inode（即一个文件，一个blockno，根据一个inode即可算出对应的blockno）以及相关的logheader

