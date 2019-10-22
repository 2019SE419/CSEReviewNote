# CSE复习笔记

## File System

### FileSystem Layers

#### Symbolic Link Layer



#### Absolute path name layer

![abusolute-find-by-name](./images/abusolute-find-by-name.png)

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