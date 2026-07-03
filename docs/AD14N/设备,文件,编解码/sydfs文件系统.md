# 3. sydfs文件系统

  Sydfs是系统数据运行的 **只读文件系统** ，可以通过sdk里下载目录里的download_bat.c工具加上 -res 命令来下载存放资源文件，并于系统运行时读取；

![](img\311.png)

**图3.1.1 下载目录下载内置flash资源**

该文件系统能使用的接口图1.1所示：

![](img\312.png)

**图3.1.2 syd 文件系统接口**

**其中ioctl支持的cmd命令有：**

①FS_IOCTL_FILE_TOTAL, 示例:

```
int file_total;
vfs_ioctl(pfile, FS_IOCTL_FILE_TOTAL, (int)&file_total);
// file_total即为pfile文件所在文件系统的总文件数；
```

②FS_IOCTL_FS_TOTAL, 示例:

```
int file_total;
vfs_ioctl(pfs, FS_IOCTL_FILE_TOTAL, (int)&file_total);
// file_total即为pfs文件系统的总文件数；
```

③FS_IOCTL_FILE_ATTR, 示例:

```
    struct vfs_attr attr;
    vfs_ioctl(pfile, FS_IOCTL_FILE_ATTR, (int)&attr);
    // attr即可获取pfile文件的位置和大小；
```

**备注**

- **注意：通过ioctl获取的文件位置仅为该pfile文件在其上层目录下的相对偏移；若要获取该文件相对于flash的偏移地址，需要调用vfs_get_fsize()；**

## 3.1. 函数int vfs_file_crc(void *pvfile)

此函数实现计算文件的CRC16校验值，目前仅SYD可使用此功能，其中参数：

```
1. pvfile：已打开的文件句柄；
2. 返回值：CRC校验值，计算错误则返回0；
```

**备注**

注意：V1.4.0以及之后的SDK，可以通过openbypath方式打开资源文件进行校验，也可打开dir文件用于校验；打开dir文件方式仅用于校验！

示例1:

```
vfs_openbypath(pfs, &pfile, "/dir_song");       //校验打包后的dir文件
u16 crc16 = vfs_file_crc(pfile);
```

示例2：

```
vfs_openbypath(pfs, &pfile, "/dir_song/so001.f1a");  // 校验资源文件
u16 crc16 = vfs_file_crc(pfile);
```