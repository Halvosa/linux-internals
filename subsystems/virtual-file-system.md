# Virtual File System (VFS)

The linux virtual file system is a software layer in the kernel that provides a unified interface for interacting with devices. It implements system calls such as open, read, write, stat, chmod and so on.

It turns out that almost all device can be categorized into only a few different categories.  

Let us start with a simple system call: stat. What happens when we run the stat command from bash?
```bash

```


 We know it must call the stat system call, but what exact C code identfier is called inside the kernel on the perimiter between user space and kernel space? 


```bash
sudo trace-cmd record -p function_graph --max-graph-depth 10 -g __x64_sys_newfstat -F stat test.tx
```

```bash
echo "linux" > test
sudo trace-cmd record -p function_graph --max-graph-depth 2 -e syscalls -F stat test
sudo trace-cmd report | less
```

Reading the report, you will realize that the C identifiers for the system calls starts with "\_\_x64\_sys\_". By searching for the regex "\_sys\_.\*stat", we find:
```c
stat-105448 [003] 95503.957132: funcgraph_entry:                   |  __x64_sys_newfstat() {
stat-105448 [003] 95503.957132: funcgraph_entry:        2.219 us   |    vfs_fstat();
stat-105448 [003] 95503.957135: funcgraph_entry:        0.772 us   |    cp_new_stat();
stat-105448 [003] 95503.957136: funcgraph_exit:         3.960 us   |  }
stat-105448 [003] 95503.957136: funcgraph_entry:                   |  syscall_exit_work() {
stat-105448 [003] 95503.957136: sys_exit_newfstat:    0x0
stat-105448 [003] 95503.957137: funcgraph_exit:         0.504 us   |  }
stat-105448 [003] 95503.957137: funcgraph_entry:                   |  exit_to_user_mode_prepare() {
stat-105448 [003] 95503.957137: funcgraph_entry:        0.229 us   |    fpregs_assert_state_consistent();
stat-105448 [003] 95503.957137: funcgraph_exit:         0.683 us   |  }
```

We already get a reference to the VFS: the function `vfs\_fstat`. Now that we have found our entrypoint into the kernel, let us increase the max depth of the function graph tracer and see what we get.

```
sudo trace-cmd record -p function_graph --max-graph-depth 10 -F stat test.txt
```

Running "trace-cmd report" again now gives us:
```
            stat-137924 [005] 126653.586743: funcgraph_entry:                   |  __x64_sys_newfstat() {
            stat-137924 [005] 126653.586744: funcgraph_entry:                   |    vfs_fstat() {
            stat-137924 [005] 126653.586744: funcgraph_entry:                   |      __fdget_raw() {
            stat-137924 [005] 126653.586744: funcgraph_entry:        0.239 us   |        __fget_light();
            stat-137924 [005] 126653.586745: funcgraph_exit:         0.705 us   |      }
            stat-137924 [005] 126653.586745: funcgraph_entry:                   |      vfs_getattr() {
            stat-137924 [005] 126653.586745: funcgraph_entry:                   |        security_inode_getattr() {
            stat-137924 [005] 126653.586746: funcgraph_entry:                   |          apparmor_inode_getattr() {
            stat-137924 [005] 126653.586746: funcgraph_entry:                   |            common_perm_cond() {
            stat-137924 [005] 126653.586746: funcgraph_entry:        0.236 us   |              common_perm();
            stat-137924 [005] 126653.586747: funcgraph_exit:         0.731 us   |            }
            stat-137924 [005] 126653.586747: funcgraph_exit:         1.307 us   |          }
            stat-137924 [005] 126653.586747: funcgraph_exit:         1.804 us   |        }
            stat-137924 [005] 126653.586747: funcgraph_entry:                   |        vfs_getattr_nosec() {
            stat-137924 [005] 126653.586748: funcgraph_entry:                   |          ext4_file_getattr() {
            stat-137924 [005] 126653.586748: funcgraph_entry:                   |            ext4_getattr() {
            stat-137924 [005] 126653.586749: funcgraph_entry:        0.261 us   |              generic_fillattr();
            stat-137924 [005] 126653.586749: funcgraph_exit:         0.882 us   |            }
            stat-137924 [005] 126653.586749: funcgraph_exit:         1.510 us   |          }
            stat-137924 [005] 126653.586750: funcgraph_exit:         2.227 us   |        }
            stat-137924 [005] 126653.586750: funcgraph_exit:         4.727 us   |      }
            stat-137924 [005] 126653.586750: funcgraph_exit:         6.134 us   |    }
            stat-137924 [005] 126653.586750: funcgraph_entry:                   |    cp_new_stat() {
            stat-137924 [005] 126653.586751: funcgraph_entry:                   |      from_kuid_munged() {
            stat-137924 [005] 126653.586751: funcgraph_entry:        0.258 us   |        map_id_up();
            stat-137924 [005] 126653.586751: funcgraph_exit:         0.707 us   |      }
            stat-137924 [005] 126653.586752: funcgraph_entry:                   |      from_kgid_munged() {
            stat-137924 [005] 126653.586752: funcgraph_entry:        0.248 us   |        map_id_up();
            stat-137924 [005] 126653.586752: funcgraph_exit:         0.691 us   |      }
            stat-137924 [005] 126653.586753: funcgraph_exit:         2.177 us   |    }
            stat-137924 [005] 126653.586753: funcgraph_exit:         9.430 us   |  }
```

This is very helpful, because it gives us a quick overview of what is going on. We can even see where the VFS hands over responsibility to the particular file system implementation that our file "test" resides on, i.e ext4; ext4\_file\_getattr gets called by vfs\_getattr\_nosec. But how does the VFS know that our file is stored on an ext4 type file system and that it should call ext4\_file\_getattr? Let us take a look at definition for vfs\_getattr\_nosec in the linux source code by running the Vim command ":tag vfs\_getattr\_nosec".

```

```

The definition for `struct path` is in `include/linux/path.h`.
The definition for `struct file` and `struct file_operations` is in `include/linux/fs.h`.
The definition for `struct fd` is in `include/linux/file.h` and is very simple:
```
struct fd {
        struct file *file;
        unsigned int flags;
};
```
A file descriptor is a struct that holds a pointer to a file struct and a flags. (What are the flags??)


The file struct has a bunch of members, but I believe the most interesting one for now are the ones shown below:

> NOTE: The code snippets below have all been trimmed, to make it easier to see the the important pieces for this discussion.

```c
struct file {
        
	struct path             f_path;
        struct inode            *f_inode;       /* cached value */
        const struct file_operations    *f_op;
        
	unsigned int            f_flags;
        fmode_t                 f_mode;
        struct fown_struct      f_owner;
        const struct cred       *f_cred;
        struct file_ra_state    f_ra;

#ifdef CONFIG_SECURITY
        void                    *f_security;
#endif
        struct address_space    *f_mapping;

} __randomize_layout
```

We can see that, amongst other things, a file struct holds a path, an inode, and a file\_operations struct. If you have ever looked out how to develop kernel modules, you will recognize the file\_operations struct.

```c
struct file_operations {
        struct module *owner;
        loff_t (*llseek) (struct file *, loff_t, int);
        ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
        ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
        ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
        ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
        int (*iopoll)(struct kiocb *kiocb, struct io_comp_batch *,
                        unsigned int flags);
        int (*iterate) (struct file *, struct dir_context *);
        int (*iterate_shared) (struct file *, struct dir_context *);
        __poll_t (*poll) (struct file *, struct poll_table_struct *);
        long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
        long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
        int (*mmap) (struct file *, struct vm_area_struct *);
        unsigned long mmap_supported_flags;
        int (*open) (struct inode *, struct file *);

	...output omitted...
```

The inode definition starts with:

"include/linux/fs.h"
```c 
struct inode {
        umode_t                 i_mode;
        unsigned short          i_opflags;
        kuid_t                  i_uid;
        kgid_t                  i_gid;
        unsigned int            i_flags;

#ifdef CONFIG_FS_POSIX_ACL
        struct posix_acl        *i_acl;
        struct posix_acl        *i_default_acl;
#endif

        const struct inode_operations   *i_op;
        struct super_block      *i_sb;
        struct address_space    *i_mapping;
```

"include/linux/path.h"
```
struct path {
        struct vfsmount *mnt;
        struct dentry *dentry;
} __randomize_layout;
```

"include/linux/mount.h"
```
struct vfsmount {
        struct dentry *mnt_root;        /* root of the mounted tree */
        struct super_block *mnt_sb;     /* pointer to superblock */
        int mnt_flags;
        struct user_namespace *mnt_userns;
} __randomize_layout;
```

## Read

`fs/read_write.c`:
```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
	return ksys_read(fd, buf, count);
}
```

```c
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos, *ppos = file_ppos(f.file);
		if (ppos) {
			pos = *ppos;
			ppos = &pos;
		}
		ret = vfs_read(f.file, buf, count, ppos);
		if (ret >= 0 && ppos)
			f.file->f_pos = pos;
		fdput_pos(f);
	}
	return ret;
}
```


```c
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
	ssize_t ret;

	if (!(file->f_mode & FMODE_READ))
		return -EBADF;
	if (!(file->f_mode & FMODE_CAN_READ))
		return -EINVAL;
	if (unlikely(!access_ok(buf, count)))
		return -EFAULT;

	ret = rw_verify_area(READ, file, pos, count);
	if (ret)
		return ret;
	if (count > MAX_RW_COUNT)
		count =  MAX_RW_COUNT;

	if (file->f_op->read)
		ret = file->f_op->read(file, buf, count, pos);
	else if (file->f_op->read_iter)
		ret = new_sync_read(file, buf, count, pos);
	else
		ret = -EINVAL;
	if (ret > 0) {
		fsnotify_access(file);
		add_rchar(current, ret);
	}
	inc_syscr(current);
	return ret;
}
```
