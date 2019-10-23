# CSE复习笔记

## File System

### FileSystem Layers

#### Symbolic Link Layer

![abusolute-find-by-name](./images/simbolic-link-in-memory.png)



如果你是新接入的设备，你将会在内存中农记录一个inode（并不代表你这个设备里面也是inode组织形式的file system）

引入新的问题，如果我们想要在文件系统A里面做一个Link访问文件系统B的一个文件。实现这个功能的背景是：

- Inode is different on other disks
- supports to attach new disks to the name space

我们提出的两个方案：

- make inodes unique all disks
- create synonyms for the files on the other disks

symbolic link的实现方式就是创建一个新的inode类型是link，我们读一个link类型的inode的时候，进行的操作是读取对应block的内容，内容即为我们指向的文件（夹）的absolute path

#### Absolute path name layer

![abusolute-find-by-name](./images/abusolute-find-by-name.png)

这个inode table是书上这种集中式处理的方式里面的具体数据其实只展示了blocks这个部分，有关inode其他部分的信息只是没有在上图中展示出来。

#### Path Name Layer

#### Link

需要在inode层级上添加refcnt进行计数，当refcnt变为0时，则正式删除文件，即去除掉的inode table中的记录。这个地方的link指hard link。每次unlink就是找到文件名对应的inode，然后通过inodetable找到具体的inode所在磁盘的offset，之后修改refcnt，并在该文件夹中消除该条entry。No Cycle for Link主要是类似于cpp中指针refcnt永远不会是0的情况。

##### renaming-1

1 UNLINK(to_name)

2 LINK(from_name)

3 UNLINK(from_name) 

如果在1、2步中出现fail，等同于是把源文件给删了，但是tmp文件还是在的（这里的语境是Text Edit状态下），应该是可以进行恢复。

##### renaming-2

1 LINK(from_name, to_name)

2 UNLINK(from_name)

如果1和2中间出现fail，则会出现refcnt出现2。

#### File Name Layer

inode中添加一个type，是这一层的需求需要能够区分inode具体的数据的内容是什么，进行的读取的相应操作会不同，比如文件夹就是我进行dump之后直接写入，然后读出来之后进buffer后我会进行反序列化的过程(load/dump)。

#### Inode Number Layer 

inode结构的索引，需要使用数字定位到具体磁盘的定位。存在在磁盘的前部。我们在lab中的实现inode是自增的，但是这样会爆炸，更好的方式是记录一个bitmap for inode number free。根目录的inode number 始终为1。

#### File Layer

##### File Requirements

能存储超过一个block

可能会增长或减小

文件是一个block的数组

需要记录文件的长度

```c++
structure inode {
    integer block_numbers[N];
    integer size;
    integer type;
    integer refcnt;
}
```

一个inode对应多个block

![inode-structure](./images/inode-structure.png)

一个inode假设是512Bit，指针是4Bit一个指针的话，能够指向的数据为(126+128+128*128) * 512Bit的数据，使用的组织结构的大小为(1+1+128) * 512 Bit

inode table最普通的是说在磁盘的super block和bitmap for free block的后面的一块区域，可以通过inode number获取到inode在磁盘中的offset，我们也可以使用算法，让这个inode table不是一个中心式的，比如hash到我们的磁盘的各个部分，这样我们可做到inode和具体的数据block更加贴近，能够提升性能(locality)

#### Block Layer

一个block number对应着一块block data，一个block data的大小是一定的。

super block是在磁盘前面的一个部分主要是存储整个disk的一些metadata。

kernel会在FS mount的时候读取superblock

–What will happen if the block size is too small? What if too big?

主要是考虑到如果block size 太小了，会导致block number会很大，为了记录block number都会消耗掉很多的空间，且查询相对更消耗时间，但是资源利用率得到了提高，如果一个block size 过大super block会比较小，但是对于空间的利用率就没有什么保证。

### FAT(File Allocation Table)

![fat-structure](./images/fat-structure.jpg)

**文件分配表：**FAT 文件系统的数据存储单元称为“簇”。簇的标准大小范围： 一个“簇”由一组连续的扇区组成，簇所包含的扇区个数必须是 2 的整数次幂， 如： 1、2、4、8、16、32 或 64 。 “簇” 的最大值为64个扇区，即32kb。

**目录项：** FAT 文件系统内的每个文件和文件夹都被分配一个目录项， 这个目录项中记录了该文件或文件夹的，文件名、大小、创建时间、文件内容起始地址以及其他一些“元数据”，说明对应的文件的“起始簇号”。

如上图所示整体结构中的“FAT 区” 由文件分区所具有的两个“(大小、结构内容相同的)FAT 表”组成，“FAT 区”紧跟在“保留区”之后。“FAT 表” 用以描述 “数据区”中的数据存储单元的分配状态 以及 为文件或目录内容分配的存储单元的前后连接关系。

文件的元信息只包含name&size，并且这些源信息存储在文件夹中，文件夹也是一种文件，文件夹里面有很多的entries，entries可能是文件，文件就包含文件名和大小，也可能是子文件夹，但是size不知道是不是统计量。

FAT不支持soft link和hard link，其上也不支持权限控制，就非常的简单。

### File Descriptor

使用File Descriptor的本因是在于我们要在操作系统通使用system call，比如write read等等，这个时候，我们需要有一个参数作为的连接操作系统和文件系统的桥梁。我们最开始会有两种选择：

- 直接返回操作系统中对应的inode的指针
- 直接返回所有的对应文件的所有block numbers

我们最终创建出File Descriptor这样的一个参数有以下两个原因：

- 安全，用户不会如同一一样能够有机会访问到内核的数据结构。
- 不会穿透让用户 直接操作文件系统，而是所有的文件操作均由操作系统代为处理。

#### FD的使用场景

##### cursor

1. 父子进程可以share cursor，通过父进程向子进程传递fd
2. 两个进程不share file cursor的是通过两个fd指向同一个文件来实现的。

##### fd_table & file_table

整个系统只有一个file_table，每个进程会有一个独立的fd_table，fd_table主要是记录map of fd to index of the file_table。

每一个process 都有自己的fd name space，即fd_table是process的context在进程间切换的时候会被切换点，即如果在进程1中存在把fd=1 重定向到一个文件，切换到另一个继承进行cout，对应的文件并没有输入。

![file-cursor-and-fd](./images/file-cursor-and-fd.png)

#### 有关于atime mtime ctime的实验

Time stamps

- Last access (by READ)

- Last modification (by WRITE)

- Last change of inode (by LINK)

atime对于ls第一次是可以的，第二次似乎在bash里面是有cache的，是不会再修改的了

### System Call OPEN READ CREATION

![FS-creation](./images/FS-creation.png)

![FS-open&write](./images/FS-open&write.png)

**When writing, which order is preferred?**

- Allocate new blocks, write new data, update size

- Allocate new blocks, update size, write new data

- Update size, allocate new blocks, write new data

第一个就是对的，因为如果size先给update了的话，一旦出现断电再恢复，就会发现读取到脏的数据，但是如果先allocate blocks然后断电了就会说只是磁盘自己内部有一块不见了。

##### 有关多进程删除文件

如果一个进程打开了某文件，但是另一个进程删除了该文件，这个时候unlink会删除掉文件夹里面的entry，inode的refcnt变为0，但是现在另一个进程手中的fd是对应的是inode（在file table）。

![FS-shadow](./images/FS-shadow.png)

#### Pooling & Interupt

Polling模式就是 OS 等待device做完操作之后再回到kernel态，这样浪费CPU太多的时间。

Interrupt指 OS提交一个task，在task完成操作之后，device给OS发送一个信号量，OS开始处理相关的数据，这样会存在一个livelock的问题，CPU会经常进行interrupt而不会回到user-level process。

采用混合模式，默认情况下使用interrupt，在interrupt发生后，启用polling

#### Interrupt Coalescing for Optimization

这个方面是和上一个优化是相互呼应的，上面的优化是OS层面的，本段的优化是针对device层面的，会存在在准备触发一次Interrupt的情况下，先等待一个指定时间，再打包整个interrupt回去。

#### DMA

Memory和Disk的交互原来需要CPU持续操作，占用CPU时间，现在出现一块硬件，可以让Memory和Disk的交互经过DMA，不需要占用CPU时间。

benefit：

- 减轻CPU load的调用
- 减少一次穿透
- 扩大总线支持long message的优势
- 摊销bus在protocol的overhead

#### Methods of Device Interaction
- PIO 通过in & out的汇编指令让CPU跟device进行交互，只能在kernel mode被调用
- Memroy-mapped I/O，使用LOAD & STORE，可以在用户态被使用，比如mmap的调用

#### 关于memory

出了cpu，所有的东西都是physical memory，我们原有的memory的physical memory被扩展到了system bus address（只是其中有一段是给了memory）

#### IDE Protocol

![IDE-protocol](./images/IDE-protocol.png)

![IDE-protocol--2](./images/IDE-protocol--2.png)

### Bus

**A set of wires**

Comprising address, data, control lines that connect to a bus interface on each module

**Broadcast link**

Every module hears every message

Bus address: identify the intended recipient, as the name

**Bus arbitration protocol**

Decide which module may send or receive message at any particular timea

Bus arbiter (optional): a circuit to choose which modules can use the bus

#### Bus Transaction

1. 源模块申请一个transaction，用于发送信息
2. 源模块设置目标模块的地址
3. 源模块发出READY信号通知其他模块
4. 目标模块发出ACKNOWLEDGE指令，在copy完数据之后，如果是同步的，只需要每个cycle去check
5. 源模块释放bus的独占

#### Sync & Async

同步数据传输则目标和源用的同一个锁，异步数据传输指的是数据在传输，但是两块硬件仍自行工作。

#### DMA运行方式

![DMA-step1](./images/DMA-step1.png)

![DMA-step2](./images/DMA-step2.png)

![DMA-step3](./images/DMA-step3.png)

整体的思维方式就是processor把map的任务下放到DMA，让Disk和Mmeory自行进行传输。

![IO-structure](./images/IO-structure.png)

#### FFS

##### 第一层次优化

修改意见：增大block size

##### 第二层优化

修改意见：使用bitmap替代freelist，尝试对文件进行连续空间的分配，保留10%的空间不被使用，因为已经够碎片化了。为了应对文件的增长，我们为了预留了一点点大的range，下一个文件就不要来占这个range，但是这样当前的磁盘使用率就很低。

##### 第三层优化

