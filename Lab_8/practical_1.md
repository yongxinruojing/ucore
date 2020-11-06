[TOC]



#### **练习1: 完成读文件操作的实现（需要编码）**

首先了解打开文件的处理流程，然后参考本实验后续的文件读写操作的过程分析，编写在sfs_inode.c中sfs_io_nolock读文件中数据的实现代码。

请在实验报告中给出设计实现”UNIX的PIPE机制“的概要设方案，鼓励给出详细设计方案

#### **解析新增代码**

```c
/************************************************文件系统**************************************************/
// process增加了file_struct

/*
 * process's file related informaction
 */
struct files_struct {
    struct inode *pwd;      // inode of present working directory
    struct file *fd_array;  // opened files array 	打开的文件数组
    int files_count;        // the number of opened files	打开的文件数量
    semaphore_t files_sem;  // lock protect sem
};


// inode结构 是文件的一个抽象代表,也就是对靠近底层文件的抽象
// 是一个接口使得内核中独立于文件系统之外的代码可以去有效地与不同的文件系统代码交互
struct inode {
    // in_info是文件系统特有的数据
    union {	// union共用内存：大小是占用内存最长的成员占用的内存
        struct device __device_info;	// 设备文件系统的内存中inode信息
        struct sfs_inode __sfs_inode_info;// SFS文件系统的内存中inode信息
    } in_info;
    
    // in_type是inode所属文件系统的类型，e.g.,用来区分是设备的inode还是sfs的inode
    enum {
        inode_type_device_info = 0x1234,
        inode_type_sfs_inode_info,
    } in_type;
    
    int ref_count;	// 此inode的引用计数,也就是有访问我的进程我就+1,当没有人访问的时候我就是0
    int open_count;	// 记录针对这个inode有多少个打开操作(打开一个文件的时候,这个文件对应的inode->open_count需要+1)
    struct fs *in_fs;	// inode所在的文件系统
    const struct inode_ops *in_ops;	// 访问inode的函数指针
};


struct file {
    // 文件当前的执行状态：无、初始化、被打开、被关闭
    enum {
        FD_NONE, FD_INIT, FD_OPENED, FD_CLOSED,
    } status;
    
    // 可读、可写
    bool readable;
    bool writable;
    
    // 文件索引值,此proc打开文件数组的索引值
    int fd;
    
    // 访问文件的当前位置(int32)
    off_t pos;
    
    //此文件对应的内存指针
    struct inode *node;
    
    // 打开此文件的次数
    int open_count;
};


/***********************************************VFS*******************************************************/
struct inode;   // 硬盘上文件的抽象结构 (inode.h)
struct device;  // 设备文件系统的抽象结构 (dev.h)
struct iobuf;   // 内核空间or用户空间的IO缓存 (iobuf.h)

// 抽象文件系统！！(SFS、device、etc.)
struct fs {
    // 只实现了sfs但是没有device的吗、、、
    union {
        struct sfs_fs __sfs_info;                   
    } fs_info;                                     // filesystem-specific data 
    
    enum {
        fs_type_sfs_info,
    } fs_type;                                     // filesystem type
    
    int (*fs_sync)(struct fs *fs);                 // Flush all dirty buffers to disk 
    struct inode *(*fs_get_root)(struct fs *fs);   // Return root inode of filesystem.
    int (*fs_unmount)(struct fs *fs);              // Attempt unmount of filesystem.
   
    // Cleanup of filesystem.???根据具体sfs的cleanup知道是指当文件系统不能用的时候就通过sfs_sync上锁
    void (*fs_cleanup)(struct fs *fs);              
};

/*************************************************SFS*****************************************************/
/*
 * On-disk superblock  是对SFS的整体的描述
 */
struct sfs_super {
    uint32_t magic;                                 /* magic number, should be SFS_MAGIC */
    uint32_t blocks;                                /* # 整体的块数 in fs */
    uint32_t unused_blocks;                         /* # 未使用的块数 in fs */
    char info[SFS_MAX_INFO_LEN + 1];                /* infomation for sfs  */
};

/* inode (on disk) */
struct sfs_disk_inode {
    uint32_t size;                                  /* size of the file (in bytes) */
    uint16_t type;                                  /* one of SYS_TYPE_* above */
    
    // 这个文件是否删除就和这个硬链接数目有关
    // 硬链接就是通过索引节点对文件的链接,多个文件可以指向同一个索引节点,说白了就是：复制+同步更新
    // 或者说同一个文件有多个文件名,修改一个就全部修改,硬链接不创建inode,相当于引用
    uint16_t nlinks;                                /* # of hard links to this file */
    
    uint32_t blocks;                                /* # 所有块数 */
    uint32_t direct[SFS_NDIRECT];                   /* 直接索引块 */
    uint32_t indirect;                              /* 间接索引块 */
//    uint32_t db_indirect;                           /* double indirect blocks */
//   unused
};

/* file entry (on disk) 目录项 */
struct sfs_disk_entry {
    // 文件名所对应的的inode的索引值
    uint32_t ino;                                   /* inode number 索引节点所占数据块的索引值 */
    
    char name[SFS_MAX_FNAME_LEN + 1];               /* file name */
};

/* inode for sfs */
struct sfs_inode {
    struct sfs_disk_inode *din;                     /* on-disk inode */
    uint32_t ino;                                   /* inode number 索引 */
    bool dirty;                                     /* true if inode modified 脏位/是否被修改 */
    int reclaim_count;                              /* kill inode if it hits zero */
    semaphore_t sem;                                /* semaphore for din */
    list_entry_t inode_link;                        /* entry for linked-list in sfs_fs  */
    list_entry_t hash_link;                         /* entry for hash linked-list in sfs_fs  */
};

/* filesystem for sfs */
struct sfs_fs {
    struct sfs_super super;                         /* on-disk superblock */
    struct device *dev;                             /* device mounted on */
    struct bitmap *freemap;                         /* blocks in use are mared 0 */
    bool super_dirty;                               /* true if super/freemap modified */
    void *sfs_buffer;                               /* buffer for non-block aligned io */
    semaphore_t fs_sem;                             /* semaphore for fs */
    semaphore_t io_sem;                             /* semaphore for io */
    semaphore_t mutex_sem;                          /* semaphore for link/unlink and rename */
    list_entry_t inode_list;                        /* inode linked-list inode双向链表 */
    list_entry_t *hash_list;                        /* inode hash linked-list inode哈希链表 */
};



```



#### **练习一**

```c
/*
 * sfs_bmap_get_nolock - according sfs_inode and index of block, find the NO. of disk block
 *                       no lock protect
 * @sfs:      sfs file system
 * @sin:      sfs inode in memory
 * @index:    the index of block in inode
 * @create:   BOOL, if the block isn't allocated, if create = 1 the alloc a block,  otherwise just do nothing
 * @ino_store: 0 OR the index of already inused block or new allocated block.
 */
// 根据提供的块索引和sfs文件抽象结构找到未锁的这个硬盘块,找到的结果索引存到ino_store
static int
sfs_bmap_get_nolock
    (struct sfs_fs *sfs, struct sfs_inode *sin, uint32_t index, bool create, uint32_t *ino_store)
    
/*
 * sfs_bmap_load_nolock - according to the DIR's inode and the logical index of block in inode, find the NO. of disk block.然后存到ino_store
 * @sfs:      sfs file system
 * @sin:      sfs inode in memory
 * @index:    the logical index of disk block in inode
 * @ino_store:the NO. of disk block
 */
static int
sfs_bmap_load_nolock(struct sfs_fs *sfs, struct sfs_inode *sin, uint32_t index, uint32_t *ino_store) 
 
    
/* sfs_rbuf - The Basic block-level I/O routine for  Rd( non-block & non-aligned io 非阻塞非对齐IO ) one disk block(using sfs->sfs_buffer)  读(非阻塞非对齐)一个磁盘块到buf里
 *  利用了memcpy(buf, sfs->sfs_buffer + offset, len);实现非对齐的IO
 *            with lock protect for mutex process on Rd/Wr disk block
 * 注意The Basic block-level I/O routine, 这个blkno是disk的number block 我是因为理解错了 导致半天写的代码错误！！！！
 */
int
sfs_rbuf(struct sfs_fs *sfs, void *buf, size_t len, uint32_t blkno, off_t offset)
    
// The Basic block-level I/O routine!!!,往buf里面读nblks个块 
int
sfs_rblock(struct sfs_fs *sfs, void *buf, uint32_t blkno, uint32_t nblks) {
    return sfs_rwblock(sfs, buf, blkno, nblks, 0);
}
/*
sfs_bmap_load_nolock -> sfs_bmap_get_nolock -> sfs_block_alloc -> sfs_clear_block -> sfs_rwblock_nolock -> dop_io
*/
/*  
 * sfs_io_nolock - Rd/Wr a file content from offset position to offset+length  disk blocks<-->buffer (in memroy)
 * @sfs:      sfs file system			sfs文件系统
 * @sin:      sfs inode in memory		内存中的sys inode
 * @buf:      the buffer Rd/Wr			读写缓冲
 * @offset:   the offset of file		文件偏移	32位int
 * @alenp:    the length need to read (is a pointer). and will RETURN the really Rd/Wr lenght读或写的长度,此函数返回值是真正完成的读或写的长度,是个无符号32位int
 * @write:    BOOL, 0 read, 1 write		读写标志
 */	
static int
sfs_io_nolock(struct sfs_fs *sfs, struct sfs_inode *sin, void *buf, off_t offset, size_t *alenp, bool write) {
    // 先获取sfs_inode的din也就是sfs文件系统在磁盘上的inode具体对象
    struct sfs_disk_inode *din = sin->din;
    
    assert(din->type != SFS_TYPE_DIR);		// 不是文件目录类型
    
    // off_t是int32
    off_t endpos = offset + *alenp, blkoff;
    
    // 清0
    *alenp = 0;
    
	// calculate the Rd/Wr end position
    // offset不能小于0,也不能大于最大,offset > endpos
    if (offset < 0 || offset >= SFS_MAX_FILE_SIZE || offset > endpos) {
        return -E_INVAL;
    }
    if (offset == endpos) {// 如果正好是结尾 还读什么
        return 0;
    }
    if (endpos > SFS_MAX_FILE_SIZE) {	// endpos大于最大文件大小的话就只能读MAX_FILE_SIZE个大小
        endpos = SFS_MAX_FILE_SIZE;
    }
    if (!write) {
        if (offset >= din->size) {
            return 0;
        }
        if (endpos > din->size) {
            endpos = din->size;
        }
    }

    // 函数指针
    int (*sfs_buf_op)(struct sfs_fs *sfs, void *buf, size_t len, uint32_t blkno, off_t offset);
    int (*sfs_block_op)(struct sfs_fs *sfs, void *buf, uint32_t blkno, uint32_t nblks);
    if (write) {// 如果写
        sfs_buf_op = sfs_wbuf, sfs_block_op = sfs_wblock;
    }
    else {// 如果读
        sfs_buf_op = sfs_rbuf, sfs_block_op = sfs_rblock;
    }
    // 不会覆盖吗。。。
	// 到这里为止,硬盘的内容已经被放到到缓冲区
    
    int ret = 0;
    size_t size, alen = 0;
    uint32_t ino;			// inode
    uint32_t blkno = offset / SFS_BLKSIZE;          // The NO. of Rd/Wr begin block
    uint32_t nblks = endpos / SFS_BLKSIZE - blkno;  // The size of Rd/Wr blocks

  //LAB8:EXERCISE1 YOUR CODE HINT: call sfs_bmap_load_nolock, sfs_rbuf, sfs_rblock,etc. read different kind of blocks in file
	/*
	 * (1) If offset isn't aligned with the first block, Rd/Wr some content from offset to the end of the first block
	 *       NOTICE: useful function: sfs_bmap_load_nolock, sfs_buf_op
	 *               Rd/Wr size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset)
	 * (2) Rd/Wr aligned blocks 
	 *       NOTICE: useful function: sfs_bmap_load_nolock, sfs_block_op
     * (3) If end position isn't aligned with the last block, Rd/Wr some content from begin to the (endpos % SFS_BLKSIZE) of the last block
	 *       NOTICE: useful function: sfs_bmap_load_nolock, sfs_buf_op	
	*/
    // 根据参考答案做了部分调整
    // 1. 获取磁盘块序号存到ino、获取第一块内容大小size、读第一块的内容
    if ((blkoff = offset % SFS_BLKSIZE) != 0) {
        size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;		
            // 自己做的时候不知道ino有啥用,现在知道了,从磁盘读要用磁盘块号,然而这里的blkno是inode里面的逻辑号也就是那个数组的索引,因此需要先读取磁盘块号并且存到ino,然后才能根据ino读取磁盘
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0) {
            goto out;
        }
        alen += size;	// 记录已经读的大小
        // 以读为例
        // 先看nblks,如果不是0代表至少跨越了一个block,此时要先读第一个未对齐的block的指定内容
        //          如果是0代表一共就在一个block内读,读完我就返回
    	// 如果只是读那一个块中的内容 读完了就直接跳走了
        if (nblks == 0) {
            goto out;
        }
        buf += size, blkno ++, nblks --;// 老师的答案逻辑清晰,这里一起修改内容
    }
 	// 2.此时要读对齐的block了(如果有的话),如果nblks是1，那到这里就最多1个对齐的块,如果nblks>1那就有(nblks-1)个对齐的块
  	size = SFS_BLKSIZE;		
    while (nblks != 0) {// 答案进行了循环读 我自己实现的是根据nblks的值然后一次读完
       	/*  if( nblks > 1 ){
        		sfs_block_op(sfs, buf+blkoff, blkno+1, nblks-1);
        		if( (alen = endpos % SFS_BLKSIZE) == 0 ) 
            		sfs_buf_op(sfs, buf+blkoff+(nblks-1)*SFS_BLKSIZE, alen, blkno+nblks-1, 0);
    	}
    	*/
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_block_op(sfs, buf, ino, 1)) != 0) {
            goto out;
        }
        alen += size, buf += size, blkno ++, nblks --;
    }

    //3. 此时就是后一种情况了
    if ((size = endpos % SFS_BLKSIZE) != 0) {	// 最后一个块也不是对齐的
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) {
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, 0)) != 0) {
            goto out;
        }
        alen += size;
    }
out:
    *alenp = alen;
    if (offset + alen > sin->din->size) {
        sin->din->size = offset + alen;
        sin->dirty = 1;
    }
    return ret;
}

```

#### **指导书疑问**

为什么间接索引ucore 支持最大的文件大小为 12 * 4k + 1024 * 4k = 48k + 4m？

+   因为一个block有4k，然后存的全是数据块索引一个索引32位 ,4k一共1k个就是1024个然后 直接索引的48k加上1024*4k就是结果了

#### **UNIX的PIPE机制**

**参考**：https://blog.csdn.net/dingdingdodo/article/details/100625018

**概念：**管道可用于具有亲缘关系进程间的通信，管道是由内核管理的一个缓冲区，相当于我们放入内存中的一个纸条。管道的一端连接一个进程的输出。这个进程会向管道中放入信息。管道的另一端连接一个进程的输入，这个进程取出被放入管道的信息。一个缓冲区不需要很大，它**被设计成为环形的数据结构**，以便管道可以被循环利用。当管道中没有信息的话，从管道中读取的进程会等待，直到另一端的进程放入信息。当管道被放满信息的时候，尝试放入信息的进程会等待，直到另一端的进程取出信息。当两个进程都终结的时候，管道也自动消失。

-   首先我们需要在磁盘上保留一定的区域用来作为PIPE机制的缓冲区，或者创建一个文件为PIPE机制服务
-   对系统文件初始化时将PIPE也初始化并创建相应的inode
-   在内存中为PIPE留一块区域，以便高效完成缓存
-   当两个进程要建立管道时，那么可以在这两个进程的进程控制块上新增变量来记录进程的这种属性
-   当其中一个进程要对数据进行写操作时，通过进程控制块的信息，可以将其先对临时文件PIPE进行修改
-   当一个进行需要对数据进行读操作时，可以通过进程控制块的信息完成对临时文件PIPE的读取
-   增添一些相关的系统调用支持上述操作