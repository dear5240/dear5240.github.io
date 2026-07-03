# 4. norfs文件系统

Norfs是一个循环管理的可读写文件系统，内部空间环形管理，空间不足时新写入的文件会覆盖之前的文件；

norfs文件系统只通过文件号进行管理，创建新文件后文件号会累加；

该文件系统一般录音在内置flash或外置flash时使用；

该文件系统能使用的接口如图所示：

![](img\41.png)

​		**图4.1 norfs文件系统支持的接口**

其中ioctl支持的cmd命令有：

①FS_IOCTL_FILE_TOTAL，同上

②FS_IOCTL_FS_TOTAL，同上

③FS_IOCTL_FILE_ATTR，同上

④FS_IOCTL_FILE_SYNC:

```
示例：
    u32 flen = 0;
    vfs_ioctl(pfile, FS_IOCTL_FILE_SYNC, (int)&flen);
    该命令一般在录音结束时调用，用于更新norfs录音文件头，flen传入时为需要舍弃的录音末尾字节数，调用结束后会flen会被赋值为录音文件的字节数；
```

④FS_IOCTL_FILE_INDEX:

```
示例：
    int file_index;
    vfs_ioctl(pfile, FS_IOCTL_FILE_INDEX, (int)&file_index);
    file_index即为pfile文件所在norfs文件系统的最新文件序号；
```

⑤FS_IOCTL_FS_INDEX:

```
示例：
    int file_index;
    vfs_ioctl(pfs, FS_IOCTL_FS_INDEX, (int)&file_index);
    file_index即为pfs所指的norfs文件系统的最新文件序号；
```