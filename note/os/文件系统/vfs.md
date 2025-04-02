/home/yunjian/work_2.6/linux2.6/Documentation/filesystems/vfs.txt
## 目录条目缓存（dcache）
VFS 实现了 open(2)、stat(2)、chmod(2) 等系统调用。传递给它们的路径名参数由 VFS 用于搜索目录条目缓存（也称为 dentry 缓存或 dcache）。这提供了一种非常快速的查找机制，将路径名（文件名）转换为特定的 dentry。dentries 存储在 RAM 中，永远不会保存到磁盘：它们仅用于提高性能。

dentry 缓存旨在提供对整个文件空间的视图。由于大多数计算机无法同时将所有的 dentry 存放在 RAM 中，因此缓存中会缺少一些条目。为了将路径名解析为 dentry，VFS 可能需要在此过程中创建 dentry，然后加载 inode。这是通过查找 inode 来完成的。

## Inode 对象
单个 dentry 通常有一个指向 inode 的指针。Inodes 是文件系统对象，如常规文件、目录、FIFO 及其他类型。它们可以存储在磁盘上（对于块设备文件系统）或内存中（对于伪文件系统）。存储在磁盘上的 inode 会在需要时复制到内存中，并且`对 inode 的更改会写回到磁盘`。`一个 inode 可以被多个 dentry` 指向（例如，硬链接就是这样）。

查找 inode 需要 VFS 调用父目录 inode 的 `lookup()`方法。这个方法由存储 `inode` 的具体文件系统实现提供。一旦 VFS 获得了所需的 dentry（因此也获得了 inode），我们可以进行一些常规操作，比如 open(2) 文件或 stat(2) 来查看 inode 数据。stat(2) 操作相对简单：一旦 VFS 拥有 `dentry`，它就会查看 inode 数据并将其中的一些信息传递回用户空间。

##  文件对象 file
打开文件需要另一个操作：分配一个文件结构（这是文件描述符在内核中的实现）。新分配的文件结构会初始化一个指向 dentry 的指针以及一组文件操作成员函数。这些操作函数来自 inode 数据。然后调用 open() 文件方法，以便特定的文件系统实现可以执行其工作。可以看出，这是 VFS 进行的另一个切换。文件结构被放置到进程的文件描述符表中。

读取、写入和关闭文件（以及其他各种 VFS 操作）是通过使用用户空间的文件描述符来获取相应的文件结构，然后调用所需的文件结构方法来完成所需的操作。在文件打开期间，它会保持 dentry 处于使用状态，这反过来意味着 VFS inode 仍然处于使用状态。


## 注册和挂载文件系统
要注册和注销文件系统，可以使用以下 API 函数：

``` c
#include <linux/fs.h>

extern int register_filesystem(struct file_system_type *);
extern int unregister_filesystem(struct file_system_type *);
```

传递的 struct file_system_type 描述了您的文件系统。当请求将设备挂载到文件空间中的某个目录时，VFS 将调用特定文件系统的相应 get_sb() 方法。挂载点的 dentry 将更新为指向新文件系统的根 inode。

您可以在文件 /proc/filesystems 中查看所有注册到内核的文件系统。



### struct file_system_type

此结构描述文件系统。自内核 2.6.22 起，定义了以下成员：
``` c
struct file_system_type {
    const char *name;
    int fs_flags;
    int (*get_sb) (struct file_system_type *, int,
                   const char *, void *, struct vfsmount *);
    void (*kill_sb) (struct super_block *); // 俩个函数指针 ，关于
    struct module *owner;
    struct file_system_type *next;
    struct list_head fs_supers;
    struct lock_class_key s_lock_key;
    struct lock_class_key s_umount_key;
};
```
- name：文件系统类型的名称，如 "ext2"、"iso9660"、"msdos" 等。
- fs_flags：各种标志（例如 FS_REQUIRES_DEV、FS_NO_DCACHE 等）。
- get_sb：当需要挂载此文件系统的新实例时调用的方法。
- kill_sb：当需要卸载此文件系统的实例时调用的方法。
- owner：用于内部 VFS 使用：在大多数情况下应将其初始化为 THIS_MODULE。
- next：用于内部 VFS 使用：应将其初始化为 NULL。
- s_lock_key, s_umount_key：与锁依赖性相关的特定键。


####  get_sb() 方法具有以下参数：

- *struct file_system_type fs_type：描述文件系统，部分由特定文件系统代码初始化。
- int flags：挂载标志。
- *const char dev_name：我们要挂载的设备名称。
- *void data：任意挂载选项，通常为 ASCII 字符串（见“挂载选项”部分）。
- *struct vfsmount mnt：一个 VFS 内部表示的挂载点。

get_sb() 方法必须确定 dev_name 和 fs_type 指定的块设备是否包含该方法支持的文件系统类型。如果成功打开指定的块设备，它会为块设备中包含的文件系统初始化一个 struct super_block 描述符。如果失败，则返回错误。

get_sb() 方法填充的超级块结构中最有趣的成员是 "s_op" 字段。这是指向 struct super_operations 的指针，描述了文件系统实现的下一个层次。

通常，文件系统使用其中一个通用的 get_sb() 实现，并提供 fill_super() 方法作为替代。通用方法包括：

get_sb_bdev：挂载驻留在块设备上的文件系统。
get_sb_nodev：挂载不依赖于设备的文件系统。
get_sb_single：挂载在所有挂载之间共享实例的文件系统。

#### fill_super() 方法的实现具有以下参数：

*struct super_block sb：超级块结构。fill_super() 方法必须正确初始化此结构。
*void data：任意挂载选项，通常为 ASCII 字符串（见“挂载选项”部分）。
int silent：在错误发生时是否保持静默。


## 超级块对象

超级块对象表示已挂载的文件系统。

### 超级块对象表示已挂载的文件系统。（super_operations超级块方法）
``` c
struct super_operations {
    struct inode *(*alloc_inode)(struct super_block *sb); // alloc_inode ,分配struct inode,且初始化 ,传一个super_block 
    void (*destroy_inode)(struct inode *);  // destroy_inode  释放inode 资源
    void (*dirty_inode) (struct inode *);  // dirty_inode dirty的时候？？？？
    int (*write_inode) (struct inode *, int); // write_inode 把inode 写入磁盘
    void (*drop_inode) (struct inode *); //当最后一次对 inode 的访问被删除时调用，持有 inode_lock 自旋锁
    void (*delete_inode) (struct inode *); // 当 VFS 希望删除 inode 时调用。
    void (*put_super) (struct super_block *);  //put_super当 VFS 希望释放超级块（即卸载）时调用。此方法在持有超级块锁时调用。
    void (*write_super) (struct super_block *); //write_super当 VFS 超级块需要写入磁盘时调用。此方法是可选的。
    int (*sync_fs)(struct super_block *sb, int wait); //写出与超级块相关的所有脏数据时调用 （什么是脏数据）
    int (*freeze_fs) (struct super_block *); // 当 VFS 锁定文件系统并强制其进入一致状态时调用
    int (*unfreeze_fs) (struct super_block *); // unfreeze_fs：当 VFS 解锁文件系统并再次使其可写时调用
    int (*statfs) (struct dentry *, struct kstatfs *); // statfs：当 VFS 需要获取文件系统统计信息时调用。 
    int (*remount_fs) (struct super_block *, int *, char *); // 
    void (*clear_inode) (struct inode *); // 当 VFS 清除 inode 时调用
    void (*umount_begin) (struct super_block *); // 当 VFS 正在卸载文件系统时调用。
    int (*show_options)(struct seq_file *, struct vfsmount *); //由 VFS 调用以显示 /proc/<pid>/mounts 的挂载选项（见“挂载选项”部分）。
    ssize_t (*quota_read)(struct super_block *, int, char *, size_t, loff_t); // 由 VFS 调用以从文件系统配额文件中读取数据。
    ssize_t (*quota_write)(struct super_block *, int, const char *, size_t, loff_t); // VFS 调用以向文件系统配额文件中写入数据。

};
```
所有方法在没有持有任何锁的情况下调用，除非另有说明。这意味着大多数方法可以安全地阻塞。
所有方法仅在进程上下文中调用（即，不在中断处理程序或底半部中）。

- alloc_inode：由 inode_alloc() 调用以分配 struct inode 的内存并初始化它。如果未定义此函数，将分配一个简单的 struct inode。通常，alloc_inode 会用于分配一个更大的结构，其中嵌入了 struct inode。
- destroy_inode：由 destroy_inode() 调用以释放为 struct inode 分配的资源。仅在定义了 ->alloc_inode 时需要，并简单地撤销 ->alloc_inode 所做的任何操作。
dirty_inode：当 VFS 标记 inode 为脏时调用此方法。this method is called by the VFS to mark an inode dirty.
write_inode：当 VFS 需要将 inode 写入磁盘时调用此方法。第二个参数指示写入是否应为同步，不是所有文件系统都会检查此标志。
drop_inode：当最后一次对 inode 的访问被删除时调用，持有 inode_lock 自旋锁。此方法应为 NULL（正常 UNIX 文件系统语义）或 generic_delete_inode（对于不希望缓存 inode 的文件系统）。
delete_inode：当 VFS 希望删除 inode 时调用。
put_super：当 VFS 希望释放超级块（即卸载）时调用。此方法在持有超级块锁时调用。
write_super：当 VFS 超级块需要写入磁盘时调用。此方法是可选的。
sync_fs：当 VFS 写出与超级块相关的所有脏数据时调用。第二个参数指示方法是否应等待写出完成。可选。
freeze_fs：当 VFS 锁定文件系统并强制其进入一致状态时调用。此方法目前由逻辑卷管理器（LVM）使用。
unfreeze_fs：当 VFS 解锁文件系统并再次使其可写时调用。
statfs：当 VFS 需要获取文件系统统计信息时调用。
remount_fs：当文件系统被重新挂载时调用。此方法在持有内核锁时调用。
clear_inode：当 VFS 清除 inode 时调用。可选。
umount_begin：当 VFS 正在卸载文件系统时调用。
show_options：由 VFS 调用以显示 /proc/<pid>/mounts 的挂载选项（见“挂载选项”部分）。
quota_read：由 VFS 调用以从文件系统配额文件中读取数据。
quota_write：由 VFS 调用以向文件系统配额文件中写入数据。


## Inode 对象

一个 inode 对象表示文件系统中的一个对象。

### struct inode_operations

此结构描述了 VFS 如何操作您文件系统中的 inode。自内核 2.6.22 起，定义了以下成员：

``` c

struct inode_operations {
    int (*create) (struct inode *, struct dentry *, int, struct nameidata *); // create 
    struct dentry * (*lookup) (struct inode *, struct dentry *, struct nameidata *); //  lookup 当 VFS 需要在父目录中查找 inode 时调用。
    int (*link) (struct dentry *, struct inode *, struct dentry *); // link 
    int (*unlink) (struct inode *, struct dentry *);
    int (*symlink) (struct inode *, struct dentry *, const char *); // 
    int (*mkdir) (struct inode *, struct dentry *, int);
    int (*rmdir) (struct inode *, struct dentry *);
    int (*mknod) (struct inode *, struct dentry *, int, dev_t);
    int (*rename) (struct inode *, struct dentry *, struct inode *, struct dentry *);
    int (*readlink) (struct dentry *, char __user *, int);
    void * (*follow_link) (struct dentry *, struct nameidata *);
    void (*put_link) (struct dentry *, struct nameidata *, void *);
    void (*truncate) (struct inode *);
    int (*permission) (struct inode *, int, struct nameidata *);
    int (*setattr) (struct dentry *, struct iattr *);
    int (*getattr) (struct vfsmount *mnt, struct dentry *, struct kstat *);
    int (*setxattr) (struct dentry *, const char *, const void *, size_t, int);
    ssize_t (*getxattr) (struct dentry *, const char *, void *, size_t);
    ssize_t (*listxattr) (struct dentry *, char *, size_t);
    int (*removexattr) (struct dentry *, const char *);
    void (*truncate_range)(struct inode *, loff_t, loff_t);
};

```

所有方法在没有持有任何锁的情况下调用，除非另有说明。

- create：由 open(2) 和 creat(2) 系统调用调用。仅在您希望支持常规文件时需要。您获取的 dentry 不应有 inode（即应为负 dentry）。此处您可能会调用 d_instantiate() 以将 dentry 和新创建的 inode 关联起来。
- lookup：当 VFS 需要在父目录中查找 inode 时调用。要查找的名称存储在 dentry 中。此方法必须调用 d_add() 将找到的 inode 插入到 dentry 中。inode 结构中的 "i_count" 字段应增加。如果命名的 inode 不存在，则应在 dentry 中插入 NULL inode（称为负 dentry）。从此例程返回错误代码必须仅在真实错误情况下进行，否则将导致使用 create(2)、mknod(2)、mkdir(2) 等系统调用时创建 inode 失败。如果希望重载 dentry 方法，则应初始化 dentry 中的 "d_dop" 字段；这是指向 struct dentry_operations 的指针。此方法在持有目录 inode 信号量时调用。
- link：由 link(2) 系统调用调用。仅在您希望支持硬链接时需要。您可能需要像在 create() 方法中那样调用 d_instantiate()。
- unlink：由 unlink(2) 系统调用调用。仅在您希望支持删除 inode 时需要。
- symlink：由 symlink(2) 系统调用调用。仅在您希望支持符号链接时需要。您可能需要像在 create() 方法中那样调用 d_instantiate()。
- mkdir：由 mkdir(2) 系统调用调用。仅在您希望支持创建子目录时需要。您可能需要像在 create() 方法中那样调用 d_instantiate()。
- rmdir：由 rmdir(2) 系统调用调用。仅在您希望支持删除子目录时需要。
- mknod：由 mknod(2) 系统调用调用，以创建设备（字符、块）inode 或命名管道（FIFO）或套接字。仅在您希望支持创建这些类型的 inode 时需要。您可能需要像在 create() 方法中那样调用 d_instantiate()。
- rename：由 rename(2) 系统调用调用，以重命名对象，使其具有第二个 inode 和 dentry 指定的父级和名称。
- readlink：由 readlink(2) 系统调用调用。仅在您希望支持读取符号链接时需要。
- follow_link：由 VFS 调用以跟随符号链接到它指向的 inode。仅在您希望支持符号链接时需要。此方法返回一个 void 指针 cookie，传递给 put_link()。
- put_link：由 VFS 调用以释放 follow_link() 分配的资源。follow_link() 返回的 cookie 作为最后一个参数传递给此方法。它在诸如 NFS 的文件系统中使用，其中页面缓存不稳定（即，在符号链接遍历开始时安装的页面在遍历结束时可能不在页面缓存中）。
- truncate：由 VFS 调用以更改文件的大小。在此方法被调用之前，inode 的 "i_size" 字段由 VFS 设置为所需大小。此方法由 truncate(2) 系统调用和相关功能调用。
- permission：由 VFS 调用以检查 POSIX 风格文件系统的访问权限。
- setattr：由 VFS 调用以设置文件的属性。此方法由 chmod(2) 和相关系统调用调用。
- getattr：由 VFS 调用以获取文件的属性。此方法由 stat(2) 和相关系统调用调用。
- setxattr：由 VFS 调用以设置文件的扩展属性。扩展属性是与 inode 关联的名称：值对。此方法由 setxattr(2) 系统调用调用。
- getxattr：由 VFS 调用以检索扩展属性名称的值。此方法由 getxattr(2) 函数调用。
- listxattr：由 VFS 调用以列出给定文件的所有扩展属性。此方法由 listxattr(2) 系统调用调用。
- removexattr：由 VFS 调用以从文件中删除扩展属性。此方法由 removexattr(2) 系统调用调用。
- truncate_range：由底层文件系统提供的方法，用于截断一系列块，即在文件中打一个洞。


### 地址空间对象
地址空间对象用于分组和管理页面缓存中的页面。它可以用来跟踪文件中的页面（或其他任何内容），同时跟踪文件部分在进程地址空间中的映射。

地址空间可以提供多种不同但相关的服务，包括内存压力通信、按地址查找页面，以及跟踪标记为脏（Dirty）或写回（Writeback）的页面。

#### 功能与操作
1. 内存压力通信：
虚拟内存（VM）可以尝试写入脏页面以清理它们，或者释放干净页面以便重用。可以在脏页面上调用 ->writepage 方法，在标记为 PagePrivate 的干净页面上调用 ->releasepage。没有 PagePrivate 且没有外部引用的干净页面会在没有通知地址空间的情况下被释放。
页面管理：
页面通常在一个基于 ->index 的树状索引中保持，这个树维护每个页面的 PG_Dirty 和 PG_Writeback 状态信息，以便快速查找带有这些标志的页面。
脏标签（Dirty Tag）：
主要由 mpage_writepages（默认的 ->writepages 方法）使用。它使用该标签找到脏页面并调用 ->writepage。如果不使用 mpage_writepages，则 PAGECACHE_TAG_DIRTY 标签几乎未被使用，write_inode_now 和 sync_inode 通过 __sync_single_inode 使用它来检查 ->writepages 是否成功写出整个地址空间。
写回标签（Writeback Tag）：
由 filemap*wait* 和 sync_page* 函数使用，通过 filemap_fdatawait_range 等待所有写回操作完成。在等待期间，如果找到需要写回的页面，将调用 ->sync_page（如果定义）。
页面附加信息
地址空间处理程序可以将额外信息附加到页面，通常使用 struct page 中的 private 字段。如果附加了此类信息，则应设置 PG_Private 标志。这将导致各种 VM 例程对地址空间处理程序进行额外调用，以处理该数据。

数据读写
数据以整页为单位读取到地址空间中，并通过复制页面或内存映射页面提供给应用程序。
数据由应用程序写入地址空间，然后通常以整页写回存储，但地址空间对写入大小有更细的控制。
读取过程：

仅需要调用 readpage。
写入过程：

更复杂，使用 write_begin/write_end 或 set_page_dirty 将数据写入地址空间，使用 writepage、sync_page 和 writepages 将数据写回存储。
页面状态管理
添加和移除页面到/从地址空间由 inode 的 i_mutex 保护。

当数据写入页面时，应设置 PG_Dirty 标志。通常在 writepage 请求写入之前保持设置。写入请求后，应清除 PG_Dirty 并设置 PG_Writeback。一旦确定安全，PG_Writeback 被清除。
写回控制
写回操作利用一个 writeback_control 结构体进行管理。











# 超级块对象的结构
``` c
// 超级块
struct super_block {
	struct list_head	s_list;		/* Keep this first */		/* 指向所有超级块的链表，该结构会插入到super_blocks链表 */
	dev_t			s_dev;		/* search index; _not_ kdev_t */	/* 设备标识符 */
	unsigned char		s_dirt;	/* 修改（脏）标志，用于判断超级块对象中数据是否脏了即被修改过了，即与磁盘上的超级块区域是否一致。 */
	unsigned char		s_blocksize_bits;			/* 以位为单位的块大小 */
	unsigned long		s_blocksize;		/* 以字节为单位的块大小 */
	loff_t			s_maxbytes;	/* Max file size */		/* 文件大小上限 */
	// 指向file_system_type类型的指针，file_system_type结构体用于保存具体的文件系统的信息。
	struct file_system_type	*s_type;		/* 文件系统类型 */
	// super_operations结构体类型的指针，因为一个超级块对应一种文件系统，
	// 而每种文件系统的操作函数可能是不同的。super_operations结构体由一些函数指针组成，
	// 这些函数指针用特定文件系统的超级块区域操作函数来初始化。
	// 比如里边会有函数实现获取和返回底层文件系统inode的方法。
	const struct super_operations	*s_op;	/* 超级块方法（对超级块操作的方法） */
	const struct dquot_operations	*dq_op;	/* 磁盘限额方法 */
	const struct quotactl_ops	*s_qcop;		/* 限额控制方法 */
	const struct export_operations *s_export_op;	/* 导出方法 */
	unsigned long		s_flags;		/* 挂载标志 */
	unsigned long		s_magic;		/* 文件系统的幻数 */
	struct dentry		*s_root;		/* 目录挂载点 */
	struct rw_semaphore	s_umount;	/* 卸载信号量 */
	struct mutex		s_lock;				/* 超级块锁 */
	int			s_count;							/* 超级块引用计数 */
	int			s_need_sync;					/* 尚未同步标志 */
	atomic_t		s_active;					/* 活动引用计数 */
#ifdef CONFIG_SECURITY
	void                    *s_security;		/* 安全模块 */
#endif
	struct xattr_handler	**s_xattr;				/* 拓展的属性操作 */
	// 指向超级块对应文件系统中的所有inode索引节点的链表。
	struct list_head	s_inodes;	/* all inodes */		/* inodes链表 */
	struct hlist_head	s_anon;		/* anonymous dentries for (nfs) exporting */
	// 该超级块表是的文件系统中所有被打开的文件。
	struct list_head	s_files;	/* 被分配文件链表 */
	/* s_dentry_lru and s_nr_dentry_unused are protected by dcache_lock */
	struct list_head	s_dentry_lru;	/* unused dentry lru */		/* 未被使用目录项链表 */
	int			s_nr_dentry_unused;	/* # of dentry on lru */			/* 链表中目录项的数目 */
	struct block_device	*s_bdev;		/* 相关的块设备 */
	struct backing_dev_info *s_bdi;
	struct mtd_info		*s_mtd;				/* 存储磁盘信息 */
	struct list_head	s_instances;	/* 该类型的文件系统 */
	struct quota_info	s_dquot;	/* Diskquota specific options */	/* 限额相关选项 */

	int			s_frozen;		/* frozen标志位 */
	wait_queue_head_t	s_wait_unfrozen;	/* 冻结的等待队列 */

	char s_id[32];				/* Informational name */	/* 文本名字 */

	// 指向指定文件系统的super_block比如，ext4_sb_info
	void 			*s_fs_info;	/* Filesystem private info */	/* 文件系统特殊信息 */
	fmode_t			s_mode;		/* 安装权限 */

	/* Granularity of c/m/atime in ns.
	   Cannot be worse than a second */
	u32		   s_time_gran;		/* 时间戳粒度 */

	/*
	 * The next field is for VFS *only*. No filesystems have any business
	 * even looking at it. You had been warned.
	 */
	struct mutex s_vfs_rename_mutex;	/* Kludge */	/* 重命名锁 */

	/*
	 * Filesystem subtype.  If non-empty the filesystem type field
	 * in /proc/mounts will be "type.subtype"
	 */
	char *s_subtype;		/* 子类型名称 */

	/*
	 * Saved mount options for lazy filesystems using
	 * generic_show_options()
	 */
	char *s_options;		/* 已存安装选项 */
}
```


# inode对象的结构

``` c
// 索引节点对象，该对象包含了内核在操作文件或目录时需要的全部信息。
struct inode {
	// 放在inode_unused或inode_in_use中
	// 指向哈希链表指针，用于查询，已经inode号码和对应超级块的时候，通过哈希表来快速查询地址。
	// 为了加快查找效率，将正在使用的和脏的inode放入一个哈希表中，
	// 但是不同的inode的哈希值可能相等，hash值相等的inode通过过i_hash成员连接。
	struct hlist_node	i_hash;		/* 散列表 */
	struct list_head	i_list;		/* backing dev IO list */	/* 索引节点链表 */
	struct list_head	i_sb_list;	/* 超级块链表 */
	// 一个给定的inode可能由多个链接（如/a/b/c，就存在/和目录a和目录b），所以就可能有多个目录项对象，因此用一个链表连接它们
	// 指向目录项链表指针，因为一个inode可以对应多个dentry(a/b和b/b是同一个硬链接的情况)，
	// 因此用一个链表将于本inode关联的目录项都连在一起。
	struct list_head	i_dentry;		/* 目录项链表 */
	unsigned long		i_ino;				/* 节点号 */
	// 访问该inode结构体对象的进程数
	atomic_t		i_count;					/* 引用计数 */
	// 硬链接计数，等于0时将文件从磁盘移除
	unsigned int		i_nlink;			/* 硬链接数 */
	uid_t			i_uid;							/* 使用者id */
	gid_t			i_gid;							/* 使用组的id */
	dev_t			i_rdev;							/* 实际设备标识符 */
	unsigned int		i_blkbits;		/* 以位为单位的块大小 */
	u64			i_version;						/* 版本号 */
	loff_t			i_size;						/* 以字节为单位的文件大小 */
#ifdef __NEED_I_SIZE_ORDERED
	seqcount_t		i_size_seqcount;	/* 对i_size进行串行计数 */
#endif
	struct timespec		i_atime;			/* 最后访问时间 */
	struct timespec		i_mtime;			/* 最后修改时间 */
	struct timespec		i_ctime;			/* 最后改变时间 */
	blkcnt_t		i_blocks;						/* 文件的块数 */
	unsigned short          i_bytes;	/* 使用的字节数 */
	umode_t			i_mode;							/* 访问权限 */
	spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */		/* 自旋锁 */
	struct mutex		i_mutex;					/* 索引节点自旋锁 */
	struct rw_semaphore	i_alloc_sem;	/* 嵌入i_sem内部 */
	// 描述了VFS用以操作索引节点对象的所有方法，这些方法由文件系统实现。调用方式如下i->i_op->truncate(i)
	// 索引节点操作函数指针，指向了inode_operation结构体，提供与inode相关的操作
	const struct inode_operations	*i_op;	/* 索引节点操作表 */
	// 指向file_operations结构提供文件操作，在file结构体中也有指向file_operations结构的指针。
	const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */	/*缺省的索引节点操作*/
	// inode所属文件系统的超级块指针
	struct super_block	*i_sb;				/* 相关的超级块 */
	struct file_lock	*i_flock;				/* 文件锁链表 */
	struct address_space	*i_mapping;	/* 相关的地址映射 */
	struct address_space	i_data;			/* 设备地址映射 */
#ifdef CONFIG_QUOTA
	struct dquot		*i_dquot[MAXQUOTAS];	/* 索引结点的磁盘限额 */
#endif
	struct list_head	i_devices;					/* 块设备链表 */
	union {
		// 下面三种结构互斥，所以放在一个union中。inode可以表示下面三者之一或三者都不是。
		struct pipe_inode_info	*i_pipe;		/* 管道信息 */			// i_pipe指向一个代表用名管道的数据结构
		struct block_device	*i_bdev;				/* 块设备驱动 */		// i_bdev指向块设备结构体
		struct cdev		*i_cdev;							/* 字符设备驱动 */	// i_cdev指向字符设备结构体
	};

	__u32			i_generation;		// 生成号，用于文件系统的唯一性验证。

#ifdef CONFIG_FSNOTIFY
	// 事件掩码，指定inode关心的事件。
	__u32			i_fsnotify_mask; /* all events this inode cares about */
	// fsnotify标记项链表。
	struct hlist_head	i_fsnotify_mark_entries; /* fsnotify mark entries */
#endif
#ifdef CONFIG_INOTIFY
	struct list_head	inotify_watches; /* watches on this inode */	/* 索引节点通知监测链表 */
	struct mutex		inotify_mutex;	/* protects the watches list */	/* 保护inotify_watches */
#endif

	unsigned long		i_state;		/* 状态标志 */
	unsigned long		dirtied_when;	/* jiffies of first dirtying */	/* 第一次弄脏数据的时间 */

	unsigned int		i_flags;		/* 文件系统标志 */

	atomic_t		i_writecount;		/* 写者计数 */
#ifdef CONFIG_SECURITY
	void			*i_security;			/* 安全模块 */
#endif
#ifdef CONFIG_FS_POSIX_ACL
	// POSIX访问控制列表。
	struct posix_acl	*i_acl;
	// 默认POSIX访问控制列表。
	struct posix_acl	*i_default_acl;
#endif
	void			*i_private; /* fs or device private pointer */	/* fs私有指针 */
};
```

# 目录项对象

```c

/* 目录项对象结构 */
struct dentry {
	/**
	 * 每个目录项对象都有3种状态：被使用，未使用和负状态
	 * 被使用：对应一个有效的索引节点（d_inode指向相应的索引节点），并且该对象由一个或多个使用者(d_count为正值)
	 * 未使用：对应一个有效的索引节点，但是VFS当前并没有使用这个目录项(d_count为0)
	 * 负状态：没有对应的有效索引节点（d_inode为NULL），因为索引节点被删除或者路径不存在了，但目录项仍然保留，以便快速解析以后的路径查询。
	 */
	// 目录项对象引用计数器  
	atomic_t d_count;		/* 使用计数 */
	unsigned int d_flags;		/* protected by d_lock */		/* 目录项标识 */
	spinlock_t d_lock;		/* per dentry lock */		/* 单目录项锁 */
	/* 表示dentry是否是一个挂载点，如果是挂载点，该成员不为0 */
	int d_mounted;		/* 是否是挂载点 */
	// inode节点的指针，便于快速找到对应的索引节点
	struct inode *d_inode;		/* Where the name belongs to - NULL is
					 * negative */		/* 相关联的索引节点 */
	/*
	 * The next three fields are touched by __d_lookup.  Place them here
	 * so they all fit in a cache line.
	 */
	/* 链接到dentry_hashtable的hash链表 */
	// dentry_hashtable哈希表维护在内存中的所有目录项，哈希表中每个元素都是一个双向循环链表，
	// 用于维护哈希值相等的目录项，这个双向循环链表是通过dentry中的d_hash成员来链接在一起的。
	// 利用d_lookup函数查找散列表，如果该函数在dcache中发现了匹配的目录项对象，则匹配对象被返回，
	// 否则，返回NULL指针。
	struct hlist_node d_hash;	/* lookup hash list */		/* 散列表 */
	/* 指向父dentry结构的指针 */  
	struct dentry *d_parent;	/* parent directory */		/* 父目录的目录项对象 */
	// 文件名  
	struct qstr d_name;		/* 目录项名称 */

	struct list_head d_lru;		/* LRU list */	/* 未使用的链表 */
	/*
	 * d_child and d_rcu can share memory
	 */
	union {
		struct list_head d_child;	/* child of parent list */	/* 目录项内部形成的链表 */
	 	struct rcu_head d_rcu;		/* RCU加锁 */
	} d_u;
	/* 是子项的链表头，子项可能是目录也可能是文件，所有子项都要链接到这个链表， */ 
	// 某目录的d_subdirs与该目录下所有文件的d_child成员一起形成一个双向循环链表，
	// 将该目录下的所有文件连接在一起，目的是保留文件的目录结构，即一个d_subdirs和
	// 多个d_child一起形成链表，d_subdirs对应文件在d_child对应文件的上一层目录。
	struct list_head d_subdirs;	/* our children */		/* 子目录链表 */
	// d_alias会插入到对应inode的i_dentry链表中
	struct list_head d_alias;	/* inode alias list */	/* 索引节点别名链表 */
	unsigned long d_time;		/* used by d_revalidate */	/* 重置时间 */
	// 指向dentry对应的操作函数集
	const struct dentry_operations *d_op;	/* 目录项操作相关函数 */
	// 指向对应超级块的指针
	struct super_block *d_sb;	/* The root of the dentry tree */	/* 文件的超级块 */
	void *d_fsdata;			/* fs-specific data */	/* 文件系统特有数据 */

	unsigned char d_iname[DNAME_INLINE_LEN_MIN];	/* small names */	/* 短文件名 */
};


```