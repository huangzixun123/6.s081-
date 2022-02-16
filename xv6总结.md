# page fault

## Copy-on-Write Fork
### Problem
fork复制父进程的用户空间的内存到子进程中，然后子进程exec，当父进程很大时，十分浪费。

### Solution
COW fork只给子进程创建一个页表，同时用户内存的PTE指向父进程的物理页，即不给子进程份分配新的物理页，只增加物理页的ref cnt，共享同一份物理页。COW fork将父进程和子进程的PTE(page table entry)权限标记为不可写。当父进程或子进程写COW页时，CPU会触发page fault信号。trap handler会侦测到page fault，然后给发生fault的进程分配一页物理页，然后复制原先物理页的内容到新的物理页中，并修改faulting进程的页表的pte指向新的物理页，以及标志pte为可写。当page fault handler返回时，用户进程将能够写该复制页。

## mmap/munmap

### Problem
以往随机访问文件open，然后通过read/wirte系统调用以及lseek来移动到对应位置，两个系统调用。

open文件后，通过mmap将文件映射到内存中实现随机访问，减少1个系统调用。

对于随机访问，不再需要频繁lseek，减少系统调用次数，减少数据copy(不需要经过中间buffer)，访问的局部更好。

### Solution
给每个进程增加vm_area_struct数组，记录进程的vm_area，调用mmap时，只记录mmap的地址、大小、prot(PROT_READ/PROT_WRITE)、flag(MAP_SHARED/MAP_PRIVATE)，增加进程的vm_area的大小，不作实际分配(lazy allocation)，当进程访问该地址时，触发缺页中断，分配物理内存，读取文件对应inode内容到物理内存中并增加进程的页表映射。


## read/write

read/write ->  sys_read/sys_write -> readi/writei -> bread/bwrite

硬盘上的块在内核中有缓存


# lock

互斥锁 : 进程休眠
spinlock : 可能某个进程频繁的持有lock，进程不会休眠，一直旋转

ticker lock : spinlock + {next, owner}   按申请锁的顺序分配锁，公平

提高读的性能，允许读并行，即读的时候能够多个进程进入critical section :  rwlock/RCU(Read-Copy-Update)

rwlock :  让reader并行，只能一个writer进入critical section

两种实现方式: reader1 + writer1 + reader2
    1. reader2需要等待writer1完成后才能进入CS(对writer公平，但导致读者竞争)
    2. reader2不需要等待writer1完成，直接进入CS(对writer不公平，提高writer读延迟)

RCU : Read-Copy-Update
与rwlock相比，读者不需要锁，同时，写的时候也可以读
写 : 
    1. Copy原结构提到新的结构体
    2. Update在副本上修改新结构体
    3. 指向新结构体指针(writer publish，之后新结构体对writer可见)
    4. 待所有读完成，删除原结构体(grace period)

RCU如何确定是否有读者在引用结构体?

```
    #define rcu_read_lock()   preempt_disable() 禁止所有CPU上下文切换，即禁止切换进程
    #define rcu_read_unlock()   preempt_enable() 允许上下文切换
```

```
    synchronize_cpu() : 遍历每一个cpu，查看当前进程是否可以切换到指定cpu上，如果可以，说明所有的cpu已经经历过一次上下文切换，则读端已经结束
```


# 文件系统

## 基于inode的文件系统

ext2

## 基于table的文件系统

FAT32

NTFS(移动硬盘)