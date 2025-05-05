---
title: "【Linux】Linux 文件系统相关概念记录"
description: ""
date: "2025-04-28T19:25:53+08:00"
thumbnail: ""
categories:
  - "操作系统"
tags:
  - "Linux Filesystem"
draft: true
---
记录一下 Linux 文件系统的相关概念
<!--more-->
## 1. Linux Filesystem 结构
![Linux Filesystem](https://developer.ibm.com/developer/default/tutorials/l-linux-filesystem/images/figure1.gif)   
Linux 文件系统的结构如图所示，可以分为：
- 系统调用接口
- 虚拟文件系统（VFS）
- 真实文件系统（比如 EXT4、NFS、FAT32 等）
- 文件缓存（Buffer Cache 和 Page Cache）
- 设备驱动
#### 1.1 VFS
VFS 对各种类型的真实文件系统做了一层抽象，提供统一的文件系统操作接口，其核心结构包括：
1. **super_block**：描述被挂载的文件系统的元信息
```c
struct super_block {
	struct list_head	s_list;		/* 指向前后的 super_block 组成链表*/
	unsigned char		s_blocksize_bits;
	unsigned long		s_blocksize;
	loff_t			s_maxbytes;	/* Max file size */
	struct file_system_type	*s_type;
	const struct super_operations	*s_op;
	const struct dquot_operations	*dq_op;
	const struct quotactl_ops	*s_qcop;
	const struct export_operations *s_export_op;
	struct dentry		*s_root; /* 根目录 */
    int			s_count; /* 引用计数 */
    struct list_head	s_inodes;	/* 全部 inodes 组成的链表 */
    ...
}
```
2. **inode**：描述具体文件或目录的元信息
3. **dentry**：描述目录项，即目录下的文件或子目录
4. **file**：描述被进程打开的文件和进程间交互的信息
#### 1.2 文件缓存
- 扇区（sector）是磁盘的基本读写单位，比如下面这块磁盘的 sector size 为 512 bytes
```shell
# fdisk -l
Disk /dev/sdc: 3.7 TiB, 4000787030016 bytes, 7814037168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: xxxxxxxxxxxxxxxxxxxxxxxxxx

Device     Start        End    Sectors  Size Type
/dev/sdc1   2048 7814035455 7814033408  3.7T Microsoft basic data
```
- 块（block）是操作系统读写磁盘的基本单位，一个 block 可以包含 2^n 个在磁盘上连续的 sector，每个 block 只能用于一个文件的保存，下面是上面磁盘挂载后的文件系统信息，block size 为 4096 bytes

    通过计算可以发现 (7814033408*512-961389308*4096)/1024/1024/1024 = 58.6 GB，说明这块 3.7 TB 的磁盘在挂载后有 1.5% 左右的空间被文件系统预留了
```shell
# stat -f /home/disk2
  File: "/home/disk2"
    ID: xxxxxxxxxxxxxx Namelen: 255     Type: ext2/ext3
Block size: 4096       Fundamental block size: 4096
Blocks: Total: 961389308  Free: 960577421  Available: 960573325
Inodes: Total: 244195328  Free: 244193850
``` 
- 页（page）是内存管理的基本单位，`page cache` 是为了加速磁盘的读写速度引入的文件缓存，其逻辑在文件层面，另外还有 `buffer cache` 对应磁盘 sector 的缓存，其逻辑在磁盘层面，Linux 系统早期版本中两个缓存是互相独立的，存在冗余缓存，后来将两者合并，如果是通过文件系统访问磁盘则缓存在 `page cache` 中，如果是绕过文件系统直接访问磁盘则缓存到 `buffer cache` 中。
```shell
# getconf PAGE_SIZE
4096
```
## 2. Linux Filesystem 相关的技术
#### 2.1 mmap 和 sendFile
#### 2.2 Fuse

## 参考
https://www.ruanyifeng.com/blog/2011/12/inode.html
https://lan-cyl.github.io/linux%20kernel/Linux-kernel-05-vfs.html