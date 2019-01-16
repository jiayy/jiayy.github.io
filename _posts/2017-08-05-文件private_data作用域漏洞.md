---
layout:     post
title: 文件 private_data 作用域漏洞
subtitle: 一种内核驱动竞态漏洞类型 
data: 2017-08-05
author: jiayy
header-img: img/post-android-bulletin-2017.jpg
catalog: true
tags:
    - android
    - vulneribility
    - bulletin
    - security
---
author : <a href="https://twitter.comengjia4574" target="_blank">jiayy(@chengjia4574)</a>  

```c 
 349 static int mdss_debug_base_open(struct inode *inode, struct file *file)
 350 {
 351         /* non-seekable */
 352         file->f_mode &= ~(FMODE_LSEEK | FMODE_PREAD | FMODE_PWRITE);
 353         file->private_data = inode->i_private; 
 354         return 0;
 355 }
 356
 357 static int mdss_debug_base_release(struct inode *inode, struct file *file)
 358 {
 359         struct mdss_debug_base *dbg = file->private_data;
 360         if (dbg && dbg->buf) {
 361                 kfree(dbg->buf);
 362                 dbg->buf_len = 0;
 363                 dbg->buf = NULL;
 364         }
 365         return 0;
 366 }
```

如上，“mdss_debug_base_open” 函数是一个驱动的 open 函数, “mdss_debug_base_release” 函数是同一个驱动的 release 函数

注意 353 行:

```c 
353         file->private_data = inode->i_private; 
```

private_data 是 struct file 的 私有数据, i_private 是 struct inode 的私有数据, 这行代码将 struct inode 上的 i_private 直接赋值给了struct file 上的 private_data, 而我们知道，struct file 和 struct inode 在内核的作用域（作用范围）是不一样的。


![1](https://www.ibm.com/developerworks/library/l-virtual-filesystem-switch/figure7.gif)


用户态进程用系统调用 open 打开一个文件会返回一个文件描述符 fd, 其在内核空间对应的实体就是一个 struct file 结构体, 一个进程打开的所有文件组成一个 struct file 数组， fd 其实就是这个数组的下标。当该进程对某个 fd 调用 open/read/write/close 系统调用时，内核会调用这个 struct file 结构上的 f_op 指针里的函数， 这个 f_op 就是驱动的底层函数。

每个file结构体都有一个指向dentry结构体的指针，“dentry”是directory entry（目录项）的缩写。

每个dentry结构体都有一个指针指向inode结构体, inode结构体保存着从磁盘inode读上来的信息。例如所有者、文件大小、文件类型和权限位等。


回头看那句代码:

```c 
353         file->private_data = inode->i_private; 
```

发现问题了吗？

这句代码表示，这个驱动文件的 file->private_data 被指向了 inode->i_private， 所以作用域被扩大了，这意味着多个进程打开这个文件后，用户空间不同进程内不同的 fd 在内核对应的 file->private_data 其实是一块数据！

当多个进程可以操纵同一块数据时， race condition 就是值得注意的问题 

```c 
 357 static int mdss_debug_base_release(struct inode *inode, struct file *file)
 358 {
 359         struct mdss_debug_base *dbg = file->private_data;
 360         if (dbg && dbg->buf) {
 361                 kfree(dbg->buf);
 362                 dbg->buf_len = 0;
 363                 dbg->buf = NULL;
 364         }
 365         return 0;
 366 }

 469 static ssize_t mdss_debug_base_reg_read(struct file *file,
 470                         char __user *user_buf, size_t count, loff_t *ppos)
 471 {
 472         struct mdss_debug_base *dbg = file->private_data;
 473         struct mdss_data_type *mdata = mdss_res;
 474         size_t len;
 ============== skip ===============
 505                         len = scnprintf(dbg->buf + tot, dbg->buf_len - tot,
 506                                         "0x%08x: %s\n",
 507                                         ((int) (unsigned long) ptr) -
 508                                         ((int) (unsigned long) dbg->base),
 509                                         dump_buf);
 ============== skip ===============
 534 }
```

比如，开头的例子中就有一个漏洞， 在驱动的 release 函数里，
line 359， 局部变量 dbg 指向了 file->private_data
line 361，调用 kfree 释放了 dbg->buf
在驱动的 read 函数里，
line 472, 局部变量 dbg 指向了 file->private_data
line 505, 调用 scnprintf 写数据进入 dbg->buf

当多个进程分别打开这个驱动文件，某些进程调用 read , 同时其他进程调用 close,  内核态会触发UAF 漏洞

基于这种思路收集内核驱动对象的多种作用域情况

```c 
1)

static int msm_pc_debug_counters_file_open(struct inode *inode,
701                 struct file *file)
702 {
703         struct msm_pc_debug_counters_buffer *buf;
704
705
706         if (!inode->i_private)
707                 return -EINVAL;
708
709         file->private_data = kzalloc( 
710                 sizeof(struct msm_pc_debug_counters_buffer), GFP_KERNEL);
711
712         if (!file->private_data) {
713                 pr_err("%s: ERROR kmalloc failed to allocate %zu bytes\n",
714                 __func__, sizeof(struct msm_pc_debug_counters_buffer));
715
716                 return -ENOMEM;
717         }
718
719         buf = file->private_data;
720         buf->reg = (long *)inode->i_private;
721
722         return 0;
723 }

725 static int msm_pc_debug_counters_file_close(struct inode *inode,
726                 struct file *file)
727 {
728         kfree(file->private_data);
729         return 0;
730 }
``` 

这是最普通的一种，file->private_data 在 open 函数里从 slab 里分配一块堆内存， 然后在 release 函数里释放堆, 这种没有竞态

```c 
2)

257 #define DEBUG_BUF_SIZE (2048)
258 static char *debug_buffer;
259 static u32 debug_data_size;

312 static int debug_open(struct inode *inode, struct file *file)
313 {
314         u32 buffer_size;
315
316         if (debug_buffer != NULL)
317                 return -EBUSY;
318         buffer_size = DEBUG_BUF_SIZE;
319         debug_buffer = kzalloc(buffer_size, GFP_KERNEL); // allocate buffer
320         if (debug_buffer == NULL)
321                 return -ENOMEM;
322         debug_data_size = fill_debug_info(debug_buffer, buffer_size);
323         return 0;
324 }


326 static int debug_close(struct inode *inode, struct file *file)
327 {
328         kfree(debug_buffer);
329         debug_buffer = NULL;
330         debug_data_size = 0;
331         return 0;
332 }
``` 

1）的变种，虽然同样是在 open 函数里分配一块堆内存，并在 release 函数里释放，然而，这块内存没有保存在 file->private_data 指针里，而是存放在一个全局变量 “debug_buffer” 里。 显然，多个进程打开这个驱动文件后，都能操作到这个全局变量，因而有 race condition 风险。

```c
3）

#define DEBUG_BUF_SIZE (2048)
static char debug_buffer[DEBUG_BUF_SIZE];
static u32 debug_data_size;

static int debug_open(struct inode *inode, struct file *file)
{

319         file->private_data = &debug_buffer[0];
320         if (debug_buffer == NULL)
321                 return -ENOMEM;
323         return 0;
324 }


326 static int debug_close(struct inode *inode, struct file *file)
327 {
328         
329         debug_buffer = file->private_data;
    	    kfree(debug_buffer);
330         debug_data_size = 0;
331         return 0;
332 }
```

file->private_data 指针指向全局数组，存在 race conditon 风险

基于作用域不当<a href="https://source.android.com/security/bulletin/2017-05-01" target="_blank">挖到cve:</a>  

* CVE-2016-10285, CVE-2016-10288, CVE-2016-10290
* CVE-2017-0624, CVE-2017-0616, CVE-2017-0617 
* CVE-2016-10294, CVE-2016-10295, CVE-2016-10296

总结：驱动工程师有必要了解vfs几个数据结构的关系，才能避免类似由于作用域混乱导致的安全风险
