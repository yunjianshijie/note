从**start_kernel(void)**开始，

##  debug函数

``` c


static inline void __noreturn BUG(void)
{
	__asm__ __volatile__("break %0" : : "i" (BRK_BUG));
	unreachable();
}

#define BUG_ON(C) __BUG_ON((unsigned long)(C)) // 如果C为真，则调用__BUG_ON

static inline void  __BUG_ON(unsigned long condition)
{
	if (__builtin_constant_p(condition)) {
		if (condition)
			BUG();
		else
			return;
	}
	__asm__ __volatile__("tne $0, %0, %1"
			     : : "r" (condition), "i" (BRK_BUG));
}

```


## 初始化结点 
上一个指自己，下一个也指自己
``` c
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
```


## 文件系统类型（描述）结构体
```c
// 用来描述各种特定文件系统类型，如ext3、ext4或UDF。
// 每个注册的文件系统都由该结构体来表示,该结构体描述了文件系统、功能、行为及其性能
// 每种文件系统不管被安装多少个实例到系统中，都只有一个file_system_type结构。
// 当文件系统被实际安装时，有一个vfsmount结构体在安装点被创建。该结构体用来代表文件系统的实例，即一个安装点
struct file_system_type {
	const char *name;		// 文件系统名字
	int fs_flags;		// 文件系统类型标志
	// get_sb()函数从磁盘读取超级块，并且在文件系统被安装时，在内存中组装超级块对象，剩余的函数描述文件系统属性。
	int (*get_sb) (struct file_system_type *, int,
		       const char *, void *, struct vfsmount *);		// 从磁盘中读取超级块
	// 终止访问超级块
	void (*kill_sb) (struct super_block *);
	struct module *owner;		// 文件系统模块
	struct file_system_type * next;		// 链表中下一个文件系统类型
	struct list_head fs_supers;				// 超级块对象链表

	// 以下字段运行时使锁生效
	struct lock_class_key s_lock_key;
	struct lock_class_key s_umount_key;
	struct lock_class_key i_lock_key;
	struct lock_class_key i_mutex_key;
	struct lock_class_key i_mutex_dir_key;
	struct lock_class_key i_alloc_sem_key;
};
```


这样定义文件系统结构体 

![alt text](./image/image0.png)
![alt text](./image/image-1.png)

结束后，调用 register_filesystem() 函数注册文件系统类型，该函数将文件系统类型添加到全局链表file_systems中。
``` c 
static struct file_system_type *file_systems;
//在file_systems链表中添加文件系统类型
```


## register_filesystem() 函数

``` c
int register_filesystem(struct file_system_type * fs)
{
	int res = 0;
	struct file_system_type ** p; //定义一个指向文件系统结构体的指针

	BUG_ON(strchr(fs->name, '.')); // 判断是不是 第一个是不是. 
	if (fs->next)
		return -EBUSY; // 如果文件下一个是NULL那就ruturn 
	INIT_LIST_HEAD(&fs->fs_supers); //初始化(超级块)链表
	write_lock(&file_systems_lock);//加锁
	p = find_filesystem(fs->name, strlen(fs->name));//寻找一个空闲的文件系统结构体
	if (*p)
		res = -EBUSY;
	else
		*p = fs;
	write_unlock(&file_systems_lock);//解锁，多核 
	return res;
}

```

中有个是find_filesystem()函数 ，用来寻找一个空闲的文件系统结构体


（）可以看出来，linux的文件系统类型是用一个链表连在一起的，所以find_filesystem()函数就是遍历这个链表，寻找一个空闲的文件系统结构体

![alt text](image/image3.png)

``` c 
extern int register_filesystem(struct file_system_type *); // 注册文件系统
extern int unregister_filesystem(struct file_system_type *); 
```
注册注销函数，在每个文件系统自身init时使用，

register_filesystem（）函数中find_filesystem()函数，如果已有文件系统则返回busy，如果没有则加入链表返回

 

前超级块
 ```c
 typedef struct SuperBlock {
  u32 first_data_sec;
  u32 data_sec_cnt;
  u32 data_clus_cnt;
  u32 bytes_per_clus;
  union {
    struct bpb bpb;
    struct ext4_sblock ext4_sblock;
  };
} SuperBlock;
 ```




 



