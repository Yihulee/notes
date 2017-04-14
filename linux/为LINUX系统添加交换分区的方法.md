# 为`LINUX`系统添加交换分区的方法

 有时，有必要在操作系统安装完成之后添加更多的交换空间。例如：把系统内存从`64MB`升级到`128MB`，但是原有的交换空间只有`128MB`。如果在系统中执行的是大量使用内存的操作或运行需要大量的内存的程序，把交换空间增加到`256MB`会更有利。       

添加交换空间有两中选择：**添加交换分区或者是添加交换文件**。在这里我们推荐添加一个交换分区。

## 一、交换分区简介

      ` Linux`系统中的交换分区是当物理内存（`RAM`）被充满时，作为物理内存的缓存来使用。当系统需要更多的内存资源，而物理内存已经充满，内存中不活跃的页就会被移动到交换分区上。交换分区位于硬盘上，所以它的存取速度比物理内存要慢。

       一般情况下，交换分区的大小应当相当于计算机内存的两倍，但不能超过`2048MB`（`2GB`）。    

 ## 二、实验场景

### 资源配置：

       主机：`Virtual Server 2K5 R2`

       内存：`512M`

       硬盘：`hda 6G`

              ` hdb  1G`

       操作系统：`Fedora 2`

      

### 实验要求：

在第二块硬盘中创建交换分区，并添加到系统中。

### 实验步骤：

 1. 启动文件系统，删除当前的交换分区。

  2．使用`fdisk`创建交换分区。

  3．使用`mkswap`命令设置交换分区。

  4．启动（`swapon`）交换分区。

5.   编辑`/etc/fstab`文件，使交换分区在引导时启用。

      


# 三、实验过程

       在实验之前我们先检查一下系统交换空间的配置情况，使用命令可以查看在硬盘上哪个分区作为交换分区使用。

```shell
fdisk -l
```

 得到的结果如下：

```shell
[root@zheng root]#fdisk –l 

Disk /dev/had:8589 MB,858990124 bytes

16 heads,63 sectors/track,16644 cylinders

Units = culinders of 1008 * 512 = 516096 bytes 

Device    Boot      Start         End               Blocks            Id           System

/dev/hda1   *         1            203              102280+         83           Linux

/dev/hda2             204         14564              7237944         83           Linux

/dev/hda3            14565       16644               1048320         82           Linux swap

[root@zheng root]#
```

 其中`/dev/hda3`是正在使用的交换分区，为了添加新的交换分区，需要将现用的交换分区删除掉，然后再添加新的交换分区。 

      1． 启动文件系统，删除当前的交换分区

       在`Fedora 2`操作系统中，可以直接使用命令删除交换分区。由我们提前知道的信息，可以做如下的操作：用命令`swapoff`卸载交换分区。

```shell
swapoff /dev/hda3
```

  2． 使用`fdisk`命令创建交换分区

      使用`fdisk`命令创建一个是当前物理内存总容量`2`倍的交换分区，使用如下的命令：

```shell
[root@zheng root]# fdisk /dev/hdb

	Command ( m for help):n  #创建新的分区

	Command action

		E     extend

		P     primary partition(1-4)

	P                                        #分区类型为主分区

	Partition number(1-4):1
	First cylinder (1-2080,default 1):
	Using default value 1
	Last cylinder or + size or +sizeM or +sizeK(1-2080,default 2080):
	Using default value 2080



	Command(m for help):t         #更改分区格式
	Selected partition 1
	Hex code (type L to list codes):82        #Linux swap 的16进制编码为82 

	Command(m for help):p      
	Disk /dev/hab:1073 MB,1073479680 bytes
	16 heads,63 sectors/track,2080 cylinders
	Units = culinders of 1008 * 512 = 516096 bytes 

	Device    Boot      Start         End               Blocks            Id           System

	/dev/hdb1   *         1          2080           1048288+       82           Linux swap      

	Command(m for help):w                     #保存分区的信息
	Command(m for help):q      
```

通过使用`fdisk`命令，我们已经创建了一个交换分区格式的分区`/dev/hdb1`。

      3． 使用`mkswap`命令设置交换分区

       命令为：

```shell
[root@zheng root]# mkswap /dev/hdb1            #将分区格式化为交换分区格式

Setting up swapspace version 1, size = 1073442 kB      
```

4． 启动（`swapon`）交换分区

```shell
 [root@zheng root]# swapon /dev/hdb1             #启用交换分区 
```

5． 编辑`/etc/fstab`文件，使交换分区在引导时启用

       编辑后的`fstab`文件关于交换分区的内容为：

```shell
 /deb/hdb1                     swap              swap      defaults          0     0 
```

 这样，我们已经创建了一个新的交换分区。重新启动系统，验证交换分区添加是否正确。

在系统重新启动时会显示如下的信息：

```shell
  … …
 Activiting swap partition :    [OK]             #激活交换分区
 … …
 Enabling swap space :           [OK]             #创建交换空间
 … …
```

 当验证没有错误之后，可以删除原有的交换分区`/dev/hda3`。最后，我们的到的分区信息如下：

```shell
[root@zheng root]#fdisk –l 
Disk /dev/had:8589 MB,858990124 bytes
16 heads,63 sectors/track,16644 cylinders
Units = culinders of 1008 * 512 = 516096 bytes 

Device    Boot      Start         End               Blocks            Id           System
/dev/hda1    *        1            203               102280+         83           Linux
/dev/hda2             204       14564                7237944         83           Linux 

Disk /dev/hab:1073 MB,1073479680 bytes
16 heads,63 sectors/track,2080 cylinders
Units = culinders of 1008 * 512 = 516096 bytes


Device    Boot      Start         End               Blocks         Id           System
/dev/hdb1   *         1            2080             1048288+       82           Linux swap 
```



      该信息表明，交换分区添加已经成功。      

       说明：参考资料中关于必须使用**救援模式启动**在`Fedora 2`系统中没有要求，并且使用**救援模式**也不能添加交换分区，这一点与参考资料中的表述完全不同。 

参考资料：

1.《红帽Linux9 从入门到精通》                美 Michael Jang 著，邱仲潘 译

2.《Red Hat Linux定制指南》                            