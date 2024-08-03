# VFS-FUSE
VFS

### 几个数据结构之间的关系
![image](https://github.com/lus-oa/VFS-FUSE/assets/122666739/cf0e7051-772d-4507-aaf3-ed214beaf59e)  


### 几个常用操作
可用`strace -o ls.txt ls`指令获取该操作调用了哪些内核函数

### 几个常见指令的函数调用流程图
![image](https://github.com/user-attachments/assets/d55182b1-8695-4af1-9d43-022492700c73)


#### 1.read操作
- 调用read系统调用
```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
    return ksys_read(fd, buf, count);
}
```
- 调用ksys_read
```c
struct fd f = fdget_pos(fd);//获取文件描述符
loff_t *ppos = file_ppos(f.file);//获取指向文件位置偏移量
ret = vfs_read(f.file, buf, count, ppos);
fdput_pos(f);//释放文件描述符
```
- 调用vfs_read
```c
ret = file->f_op->read(file, buf, count, pos);//根据file中定义的file_operation中的read操作调用实际文件系统中定义的read操作
```
#### 2.write操作
- 调用write系统调用
```c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t,count)
{
    return ksys_write(fd, buf, count);
}
```
- 调用ksys_write
```c
struct fd f = fdget_pos(fd);//获取文件描述符
loff_t *ppos = file_ppos(f.file);//获取文件偏移量
ret = vfs_write(f.file, buf, count, ppos);
fdput_pos(f);//释放文件描述符
```
- 调用vfs_write
```c
ret = file->f_op->write(file, buf, count, pos);//根据file中定义的file_operation中的write操作调用实际文件系统中定义的write操作
```
#### 3.ls操作
- strace打印信息
![image](https://github.com/lus-oa/VFS-FUSE/assets/122666739/2718ac79-4458-4082-8dfc-92c1f1bbb69b)

##### 3.1 调用fstat系统调用
```c
SYSCALL_DEFINE2(fstat, unsigned int, fd, struct __old_kernel_stat __user *, statbuf)
{
    struct kstat stat;
    int error = vfs_fstat(fd, &stat);//调用vfs_stat
    if (!error)
        error = cp_old_stat(&stat, statbuf);
    return error;
}
```
- 调用vfs_stat函数，用于获取文件或目录的状态信息的函数
```c
f = fdget_raw(fd);//获取文件描述符
error = vfs_getattr(&f.file->f_path, stat, STATX_BASIC_STATS, 0);
fdput(f);//释放文件描述符
```
- 调用vfs_getattr函数，获取给定路径或 inode 的文件或目录属性
```c
int retval;
retval = security_inode_getattr(path);
if (retval)
    return retval;
return vfs_getattr_nosec(path, stat, request_mask, query_flags);
```
##### 3.2 调用getdents64系统调用
```c
SYSCALL_DEFINE3(getdents64, unsigned int, fd, struct linux_dirent64 __user *,dirent, unsigned int, count)
//struct linux_dirent64 __user * dirent:用户空间缓冲区，用于存储读取的目录条目。
{
    f = fdget_pos(fd);
    error = iterate_dir(f.file, &buf.ctx);//遍历目录项
    fdput_pos(f);
}
```
- 调用iterate_dir遍历目录项
```c
struct inode *inode = file_inode(file);//获取inode
res = security_file_permission(file, MAY_READ);//检查当前进程是否有读取该文件的权限
if (shared)//调用file_operation中的iterate_shared/iterate操作调用实际文件系统中定义的目录遍历操作
    res = file->f_op->iterate_shared(file, ctx);
else
    res = file->f_op->iterate(file, ctx);
```
#### 4.mkdir操作
![image](https://github.com/lus-oa/VFS-FUSE/assets/122666739/a98d020f-441c-4f1d-ace6-d6596e75d9a4)

- 调用mkdir系统调用
```c
SYSCALL_DEFINE2(mkdir, const char __user *, pathname, umode_t, mode)
{
    return do_mkdirat(AT_FDCWD, getname(pathname), mode);
}
```
- 调用do_mkdirat函数
```c
dentry = filename_create(dfd, name, &path, lookup_flags);//创建路径以及目录项
error = security_path_mkdir(&path, dentry, mode);//安全性检查
error = vfs_mkdir(mnt_userns, path.dentry->d_inode, dentry,mode);//调用vfs_mkdir
```
- 调用vfs_mkdir函数
```c
int vfs_mkdir(struct user_namespace *mnt_userns, struct inode *dir,struct dentry *dentry, umode_t mode)
{
int error = may_create(mnt_userns, dir, dentry);//是否允许创建
error = security_inode_mkdir(dir, dentry, mode);//安全性检查
error = dir->i_op->mkdir(mnt_userns, dir, dentry, mode);//与read/write不同，mkdir是对inode操作而不是对file操作，因此这里会调用inode中定义的inode_operations中的mkdir操作
}
```

