# 无人值守自动安装linux操作系统

![image-20240507142842115](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507142842115.png)

## 1. 无人值守流程



![image-20240507194522662](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507194522662.png)

## 2. 组件安装

### 2.1 安装DHCP

```
# yum -y install dhcp
 
 #vim /etc/dhcp/dhcpd.conf
default-lease-time 600;
max-lease-time 7200;
log-facility local7;
subnet 192.168.0.0 netmask 255.255.255.0 {
  range 192.168.0.247 192.168.0.248;
  option routers 192.168.0.1;
  option domain-name-servers 114.114.114.114;
  next-server 192.168.0.203; #指定tftp的ip 
  filename "pxelinux.0"; #启动引导文件
}
```

* 启动DHCP

```
[root@localhost ~]# systemctl start dhcpd
[root@localhost ~]# systemctl enable dhcpd
```

### 2.2 安装TFTP服务

tftp简单文件共享服务，disable设置为no，server_args为共享目录，存放镜像及内核文件

```
 #yum -y install tftp-server
vim /etc/xinetd.d/tftp
# default: off
# description: The tftp server serves files using the trivial file transfer \
#       protocol.  The tftp protocol is often used to boot diskless \
#       workstations, download configuration files to network-aware printers, \
#       and to start the installation process for some operating systems.
service tftp
{
        socket_type             = dgram
        protocol                = udp
        wait                    = yes
        user                    = root
        server                  = /usr/sbin/in.tftpd
        server_args             = -s /var/lib/tftpboot
        disable                 = no
        per_source              = 11
        cps                     = 100 2
        flags                   = IPv4
}

```

* 安装syslinux,要 pxelinux.0 引导加载程序

```
yum install syslinux -y
```

* 拷贝镜像包的配置文件

```
umount /dev/cdrom
mount /dev/cdrom /media/
cp /media/isolinux/{vmlinuz,initrd.img} /var/lib/tftpboot/
mkdir /var/lib/tftpboot/pxelinux.cfg
cat /media/isolinux/isolinux.cfg > /var/lib/tftpboot/pxelinux.cfg/default
```

* default文件

```
default linux
timeout 600
display boot.msg
menu title CentOS 7
label linux
  menu label ^Install CentOS 7
  menu default
  kernel vmlinuz
  append initrd=initrd.img ks=nfs:192.168.0.203:/data/nfs/ks.cfg #应答文件存放路径也可以是http
label rescue
  menu label ^Rescue a CentOS system
  kernel vmlinuz
  append initrd=initrd.img rescue
```

* 启动TFTP

因为 tftp 服务是挂载在超级进程 xinetd 下的，所以通过启动 xinetd 来启动 tftp 服务。

```
[root@localhost ~]# systemctl start xinetd
[root@localhost ~]# systemctl enable xinetd
```



### 2.3 kickstart无人应答

* 生成ks.cfg 文件需要system-config-kickstart 工具，而此工具依赖于X Windows，所以我们需要安装X Windows 和Desktop 并重启系统，操作如下：

```
yum groupinstall "Server with GUI" -y
[root@localhost ~]# systemctl set-default graphical.target  // 设置默认启动到图形界面或者init 5
[root@localhost ~]# reboot      // 重启机器
```

* 安装system-config-kickstart工具

```
yum -y install system-config-kickstart
```

#### 2.3.1 生成ks.cfg

![image-20240507155338443](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507155338443.png)

![image-20240507155436989](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507155436989.png)

![image-20240507155557885](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507155557885.png)

![image-20240507155912716](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507155912716.png)

![image-20240507160002089](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507160002089.png)

![image-20240507160048320](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507160048320.png)

![image-20240507160105088](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507160105088.png)

![image-20240507160921270](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507160921270.png)

注意：必须要配置本地yum源，要不然ickstart配置会出现软件包选择出现没有软件包信息

```
[root@harbor pxelinux.cfg]# cat /etc/yum.repos.d/dvd.repo
[development] #名称必须要是development
name=rehat
baseurl=file:///media
gpgcheck=0
```

ks.cfg文件内容

```
#platform=x86, AMD64, 或 Intel EM64T
#version=DEVEL
# Install OS instead of upgrade
install
# Keyboard layouts
keyboard 'us'
# Root password
rootpw --iscrypted $1$HJcRTlOW$sDJg0DFWLjf5NlaQov2Y..
# System language
lang en_US
# System authorization information
auth  --useshadow  --passalgo=sha512
# Use graphical install
graphical
# SELinux configuration
selinux --disabled
# Do not configure the X Window System
skipx

# Use NFS installation media
nfs --server=192.168.0.203 --dir=/media

# Firewall configuration
firewall --disabled
# Network information
network  --bootproto=dhcp --device=ens33
# Reboot after installation
reboot
# System timezone
timezone Asia/Shanghai
# System bootloader configuration
bootloader --location=mbr
# Partition clearing information
clearpart --all --initlabel
# Disk partitioning information
ignoredisk --only-use=sda
part biosboot --fstype="biosboot" --size=1
part /boot --fstype="ext4"  --size=1024
part /boot/efi --fstype="ext4" --size=1024
part swap  --fstype="swap" --size=2048
part / --fstype="ext4" --size=1 --grow


%packages
@base
-abrt-addon-ccpp
-abrt-addon-python
-abrt-cli
-abrt-console-notification
-bash-completion
-blktrace
-bpftool
-bridge-utils
-bzip2
-chrony
-cryptsetup
-dmraid
-dosfstools
-ethtool
-fprintd-pam
-gnupg2
-hunspell
-hunspell-en
-kmod-kvdo
-kpatch
-ledmon
-libaio
-libreport-plugin-mailx
-libstoragemgmt
-lvm2
-man-pages
-man-pages-overrides
-mdadm
-mlocate
-mtr
-nano
-ntpdate
-pinfo
-plymouth
-pm-utils
-rdate
-rfkill
-rng-tools
-rsync
-scl-utils
-setuptool
-smartmontools
-sos
-sssd-client
-strace
-sysstat
-systemtap-runtime
-tcpdump
-tcsh
-teamd
-time
-unzip
-usbutils
-vdo
-vim-enhanced
-virt-what
-wget
-which
-words
-xfsdump
-xz
-yum-langpacks
-yum-utils
-zip

%end

```

#### 2.3.2 启动vmware虚拟机

![image-20240507205222098](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507205222098.png)

![image-20240507205307069](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507205307069.png)





# 存储管理

存储设备类型： IDE设备（hd）、SATA USB或SCSI(sd)

## 1. 磁盘分区

传统的MBR分区只有4个，解释磁盘容量较大，4个分区分完后也无法使用。

![image-20240507214050289](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507214050289.png)



如何获得多个分区呢，从4开始使用扩展分区，在扩展分区中创建逻辑分区，所有逻辑分区一定从5开始划分。

![image-20240507214849320](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240507214849320.png)

### 1.1 查看磁盘

```
[root@harbor nfs]# fdisk -l

Disk /dev/nvme0n1: 21.5 GB, 21474836480 bytes, 41943040 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x445b8664

        Device Boot      Start         End      Blocks   Id  System
/dev/nvme0n1p1            2048    41943039    20970496   83  Linux

Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000a7dff

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    41943039    19921920   8e  Linux LVM

Disk /dev/mapper/centos-root: 18.2 GB, 18249416704 bytes, 35643392 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

```

### 1.2 分区

#### 1.2.1 MBR分区 有分区数量限制

```
[root@harbor ~]# fdisk /dev/sdb
欢迎使用 fdisk (util-linux 2.23.2)。

更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。

Device does not contain a recognized partition table
使用磁盘标识符 0xae5df9b5 创建新的 DOS 磁盘标签。

命令(输入 m 获取帮助)：m
命令操作
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition #删除分区
   g   create a new empty GPT partition table
   G   create an IRIX (SGI) partition table
   l   list known partition types
   m   print this menu
   n   add a new partition #创建分区
   o   create a new empty DOS partition table
   p   print the partition table #打印分区
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)

命令(输入 m 获取帮助)：

```

#### 1.2.2 加载分区表

```
# partprobe /dev/sdb
```

#### 1.2.3 GPT分区不受分区数量限制

MBR分区大于2T或者超过4个分区限制，GPT分区不受这个影响。

```
parted /dev/sdc mklabel gpt #修改分区表格式
```

#### 1.2.4 gpt分区表查看

```
# parted /dev/sdc print

Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdc: 5369MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start  End  Size  File system  Name  标志

```

#### 1.2.5 gtp分区

```
# parted /dev/sdc mkpart primary ext4 1 2G
#parted /dev/sdc mkpart primary ext4 2G 3G

[root@harbor ~]# parted /dev/sdc print
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdc: 5369MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     标志
 1      1049kB  2000MB  1999MB               primary
 2      2000MB  3000MB  1000MB               primary

```

#### 1.2.6 删除分区

```
 parted /dev/sdc rm 1 指定分区号
```

### 1.3 格式化和挂载

#### 1.3.1 格式化

```
# mkfs.ext4 /dev/sdc1
mke2fs 1.42.9 (28-Dec-2013)
文件系统标签=
OS type: Linux
块大小=4096 (log=2)
分块大小=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
122160 inodes, 487936 blocks
24396 blocks (5.00%) reserved for the super user
第一个数据块=0
Maximum filesystem blocks=501219328
15 block groups
32768 blocks per group, 32768 fragments per group
8144 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: 完成
正在写入inode表: 完成
Creating journal (8192 blocks): 完成
Writing superblocks and filesystem accounting information: 完成
```

#### 1.3.2 挂载文件系统

```
 挂载
 mount /dev/sdc1 /mnt
 卸载
 umount /dev/sdc1 或 umount /mnt
```

* 开机自动挂载

```
[root@harbor ~]# cat /etc/fstab
# /etc/fstab
# Created by anaconda on Mon Nov 13 15:46:19 2023
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=a6c341ce-f6ef-443a-b71c-76cf6452f705 /boot                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
/dev/nvme0n1p1 /data ext4 defaults        0 0

```



## 2. LVM逻辑卷

lvm逻辑卷可实现文件系统的动态扩容，缺点是一旦vg中一个磁盘或分区损坏，数据有丢失风险。

![image-20240508111450900](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240508111450900.png)

### 2.1 创建pv

* 准备两个不同的磁盘的分区sdb1、sdc1

```
pvcreate /dev/sdb1 /dev/sdc1
```

* 查看pvdisplay

```
 "/dev/sdb1" is a new physical volume of "953.00 MiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb1
  VG Name
  PV Size               953.00 MiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               bwcoSX-v5mR-0Lxz-s0Js-Txci-ME4a-UrBMJv

  "/dev/sdc1" is a new physical volume of "1.86 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdc1
  VG Name
  PV Size               1.86 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               GCrH0m-0jgK-5vmI-33CO-9p41-wNZc-Zp8kNj

```

### 2.2 创建vg

```
vgcreate test-vg /dev/sdb1 /dev/sdc1
  Volume group "test-vg" successfully created
```

* 查看vgdisplay

```
--- Volume group ---
  VG Name               test-vg
  System ID
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               <2.79 GiB
  PE Size               4.00 MiB
  Total PE              714
  Alloc PE / Size       0 / 0
  Free  PE / Size       714 / <2.79 GiB
  VG UUID               yrUQWs-oBtI-WgTp-bnG3-95yf-7fBt-OL0ref


```

### 2.3 创建lv

* 创建1G的逻辑卷

```
[root@harbor ~]# lvcreate -L 1G -n test-lv1 test-vg
  Logical volume "test-lv1" created.
```

* 查看

```
 --- Logical volume ---
  LV Path                /dev/test-vg/test-lv1
  LV Name                test-lv1
  VG Name                test-vg
  LV UUID                yexDcG-DQeT-AbTJ-CR60-Ms5o-cCKn-T0XUnU
  LV Write Access        read/write
  LV Creation host, time harbor, 2024-05-08 11:21:37 +0800
  LV Status              available
  # open                 0
  LV Size                1.00 GiB
  Current LE             256
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2
```

* 格式化并挂载

```
# mkfs.ext4 /dev/test-vg/test-lv1
#mount /dev/test-vg/test-lv1 /mnt
[root@harbor ~]# df -h
文件系统                        容量  已用  可用 已用% 挂载点
/dev/mapper/test--vg-test--lv1  976M  2.6M  907M    1% /mnt
```

### 2.4 扩容lv

* test-lv1 扩容1G

```
# lvextend -L +1G /dev/test-vg/test-lv1
  Size of logical volume test-vg/test-lv1 changed from 1.00 GiB (256 extents) to 2.00 GiB (512 extents).
  Logical volume test-vg/test-lv1 successfully resized.
```

* 更新文件系统大小

```
# resize2fs /dev/test-vg/test-lv1
# df -h
文件系统                        容量  已用  可用 已用% 挂载点
/dev/mapper/test--vg-test--lv1  2.0G  3.0M  1.9G    1% /mnt

```

* 当vg资源不足时，扩容vg容量

```
#先创建pv
pvcreate /dev/sdc2
#扩容至vg
vgextend test-vg /dev/sdc2
#再次扩容lv 1G
lvextend -L +1G /dev/test-vg/test-lv1
```

### 2.5 删除lvm

一定要按以下顺序删除

* 先卸载分区

```
umount /dev/mapper/test--vg-test--lv1
```

* 删除lv

```
lvremove /dev/test-vg/test-lv1
```

* 删除vg

```
 vgremove test-vg
```

* 删除pv

```
pvremove /dev/sdb1 /dev/sdc1 /dev/sdc2
```

## 3. RAID级别

raid常用级别

* raid 0:  `不含校验和冗余的条带存储`，数据分隔后分别写入存储，最少两块磁盘，存储和性能较高。安全性最低。
* raid 1: `不含校验的镜像存储`， 数据写入时将被复制到每块磁盘中，安全性高，容量利用率低。

* raid 5: `数据块级别的分布式校验条带存储`,数据以块单位分别存储到不同的磁盘上，并对数据进行海明码运算，并写入到不同磁盘，放到同一条带，如果一块磁盘故障会根据校验码重建。
* raid 10: 先将磁盘做raid1，再对不同raid1的分组做raid0，安全性较高，容量利用率50%。



# 性能监控

待续。。。





# Shell简介

## 1. 命令序列

* & 

开启一个子shell，并在后台执行

* ;

命令组合，仅按顺序执行

* && 

且关系，前面命名执行成功后才会执行下面的命令

* ||

或关系，前面命令执行失败才能执行

## 2. 花括号{}的使用技巧

* 多个项目

```
[root@harbor ~]# echo {1,2,3}
1 2 3
```

* 连续的序列

```
[root@harbor ~]# echo {1..3}
1 2 3
```

## 3. 变量

### 3.1 自定义变量

`格式：NAME=[VALUNE]`,建议变量名为大写或首字母大写

```
# NAME="MARK"
# export  NAME="MARK" #添加至环境变量,子进程才能会被引用
# bash
# echo "${NAME}"
MARK
```

* read 标准输入变量

```
[root@harbor ~]# read name
11
[root@harbor ~]# echo "$name"
11
```

### 3.2 位置变量

* $0代表当前shell程序的文件名称
* $1代表运行shell程序的第一个参数,$2...9 依次类推
* $*代表所有参数内容，作为一个整体
* $@代表所有参数内容，作为个体
* $#代表参数的个数
* $?代表程序的退出码，正常为0，非零的代表执行失败
* $$代表程序的PID

执行shell bash a.sh 1 2

```
echo $#  参数个数2
echo $1  第一个参数 1
echo $2  第二个参数 2
echo $*  所有参数整体‘1 2’
echo $@  所有参数个体 ‘1’ ‘2’
```

## 4. 数组

* 定义数组

```
name=(v1 v2 v3)
declare -a name 定义空素组
```

* 数组调用

数组调用通过索引，从0开始

```
name=(1 2 3)
[root@harbor ~]# echo ${name[0]}
1
[root@harbor ~]# echo ${name[1]}
2
[root@harbor ~]# echo ${name[2]}
3
```

## 5. 算术运算及测试

### 5.1 算术运算

```
[root@harbor ~]# echo $(( 1+2 )) 加
3
[root@harbor ~]# echo $((1-1)) 减
0
[root@harbor ~]# echo $((1*2)) 乘
2
[root@harbor ~]# echo $((1%2)) 取余
1
[root@harbor ~]# echo $((1/2)) 除法，取整数位
0
x=2
[root@harbor ~]# echo $((x--)) 自减 -1
1
[root@harbor ~]# echo $((x++)) 自增 +1
1
[root@harbor ~]# echo $((x**3)) 幂运算2**3=8
8
```

### 5.2 常用的测试表达式

`格式： [ 测试表达式 ]，中括号内两边必须要留空格哦`

* -d file 判断目录是否存在 -f 文件且普通文件 -e 判断文件
* -s file 判断文件是否存在且非空
* -n str 字符串是否非0 -z 字符串长度为0
* 字符串是否相等 = 或 ！=
* 整数比较 -eq 相等  -ne 不等于 ；-gt 大于 -ge 大于等于 ； -le 小于等于 -lt 小于

## 6. 正则表达式

### 6.1 基础正则

* .代表任意单个字符
* *代表0次或多次
* [] 集合里任意单个字符，\[^a\]取反
* ^已开头、 $结尾
* \\{n,m\\} 匹配字符n到m
* \\{n,\\} 匹配字符至少n次
* \\{n\\} 匹配字符n次



### 6.2 扩展正则

* \{n,m\} 匹配字符n到m

* ?匹配0或1次
* \+匹配1或多次
* | 逻辑或
* () 匹配正则集合，eg:（root|admin） 匹配root或者admin的行

## 7. 文本工具

### 7.1 grep

* -A n 匹配当前行的后n行
* -C n 匹配当前行的前后n行
* -E 等于egre 扩展正则
* -v 取反
* -i 忽略大小写

```
#匹配ro后面的任意字符的行
[root@k8s-master01 ~]# grep ro.* /etc/passwd
root:x:0:0:root:/root:/bin/bash
operator:x:11:0:operator:/root:/sbin/nologin
chrony:x:998:996::/var/lib/chrony:/sbin/nologin
haproxy:x:188:188:haproxy:/var/lib/haproxy:/sbin/nologin

#匹配roo后面的字符行
[root@k8s-master01 ~]# egrep roo+ /etc/passwd
root:x:0:0:root:/root:/bin/bash
operator:x:11:0:operator:/root:/sbin/nologin
```

### 7.2 sed

sed是一款流编辑工具，对大数据文本更加合适

`语法格式：sed [option] {脚本指令}`

`基础选项`

* -n 静默输出配合p使用
* -e 多脚本指令使用 用；也可以
* -f 指定脚本指令文件

`常用指令`

* i 插入
* a 追加
* d 删除
* p 打印
* s 替换



以下操作不会修改源文件，仅测试指令可用性，修改文件需要加-i 选项

```
#准备文件
[root@harbor ~]# cat test.txt
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes


#第1行后追加PROXY_METHOD=none
#sed '1a PROXY_METHOD=none' test.txt

#第1行前面插入PROXY_METHOD=none
#sed '1i PROXY_METHOD=none' test.txt

#删除#开头的行
#sed '/^#/ d' test.txt

#替换所有的行yes--》no
 sed 's/yes/no/' test.txt 或者 sed 's/yes/no/g' test.txt 
 
#替换行或者匹配特定行 sed '2s/yes/no/' sed '/xxxx/s/yes/no/'
```

### 7.3 awk

Awk是一种编程语言，十分强大，本次结合实例进行学习

* 多个分隔符文本，如何打印不同字段

```
root@harbor ~]# echo "i love,you:too" |awk -F[' ',:] '{print $1,$2,$3,$4}'
i love you too
```

* 两种办法指定分隔符

```
[root@harbor ~]# awk -F ':' '{print $1}' /etc/passwd
[root@harbor ~]# awk  'BEGIN{FS=":"}{print $1}' /etc/passwd
root
bin
...
```

* 设置分隔符输出OFS

分隔符默认是空格和制表符，如果文本内非默认，需要指定

```
[root@harbor ~]# awk  'BEGIN{FS=":"; OFS="-"}{print $1,$2}' /etc/passwd
root-x
bin-x
...
```

* 使用正则匹配行

```
[root@harbor ~]# awk '/(root|makun)/ {print $0}' /etc/passwd
root:x:0:0:root:/root:/bin/bash
operator:x:11:0:operator:/root:/sbin/nologin
makun:x:1000:1000:makun:/home/makun:/bin/bash
```

* 计算内存使用率

```
[root@harbor ~]# free -m |awk '/Mem/{print (($2-$NF)/$2)*100}'
62.4519
```

* if判断

```
[root@harbor ~]# df -h |awk '{if ($2>10000) print $NF;else print "ok"}'
挂载点
/dev
/dev/shm
...
```

## 8. shell脚本编写

部署Dhcp服务脚本

```shell
#!/bin/bash
SUBNET="192.168.0.0"
NETMASK="255.255.255.0"
RANGE="192.168.0.247 192.168.0.248"
ROUTER="192.168.0.1"
DNS="114.114.114.114"

#TEST yum repo exist dhcp？
function test_yum(){
  if ! yum list dhcp &>/dev/null;then 
      echo "yum repo not found dhcp,please add"
      exit
  fi
}
#dhcp config
functon create_conf(){
cat > /etc/dhcp/dhcpd.conf <<EOF
default-lease-time 600;
max-lease-time 7200;
log-facility local7;
subnet $SUBNET netmask $NETMASK {
  range $RANGE;
  option routers $ROUTER;
  option domain-name-servers $DNS;
}
EOF
}

if ! rpm -qa|grep dhcp >/dev/null 2&>1;then 
	test_yum()
	yum -y install dhcp >/dev/null 2&>1
fi

 create_conf
 systemctl restart dhcpd #开启服务
 systemctl enable dhcpd #设置开机自启动
```

# 网络服务

## 1. NFS网络文件共享服务

### 1.1 安装

```
yum -y install nfs-utils rpcbind
```

### 1.2 配置

* /etc/exports

共享目录 客户端主机 选项

```
[root@harbor dhcp]# cat /etc/exports
/data/nfs/ 192.168.0.0/24(rw,no_root_squash,no_subtree_check,sync)
/media 192.168.0.0/24(rw,no_root_squash,no_subtree_check,sync)
```

`ro rw`: 客户端对远程挂载目录的权限是只读还是可读写

`no_root_squash`：不屏蔽远程的root权限

`sync async`: 同步就是写入到硬盘才返回client成功消息, 异步就数据未完全写入磁盘就返回成功

`no_subtree_check`:如果共享/usr/bin之类的子目录时，强制NFS检查父目录的权限

* 客户端挂载

```
mount -t nfs 192.168.0.203:/data/nfs /mnt/nfs
```

## 2. DHCP服务

该服务为了给局域网动态分配ip，端口udp 67

### 2.1 配置文件解析

```
#全局默认租期，单位s
default-lease-time 600;

#全局最大租期
max-lease-time 7200;


# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
log-facility local7;


# A slightly different configuration for an internal subnet.
#地址段
subnet 10.5.5.0 netmask 255.255.255.224 {
  #分配的ip地址池
  range 10.5.5.26 10.5.5.30;
  #客户端分配的dns
  option domain-name-servers ns1.internal.example.org;
  #域名
  option domain-name "internal.example.org";
  #网关
  option routers 10.5.5.1;
  #广播地址
  option broadcast-address 10.5.5.31;
  default-lease-time 600;
  max-lease-time 7200;
  next-server 下一个服务地址ip;
  filename 下个服务访问的文件;
}



#根据client的mac 固定分配ip
host fantasia {
  hardware ethernet 08:00:07:26:c0:a5;
  fixed-address 192.168.0.10
}
```

## 3. DNS

域名查询分类

* 递归： 客户端本次缓存没有的话，请求本地dns服务器。
* 迭代： 如果本地dns服务器没有对应记录，则会依次向根域、顶级域、二级域等迭代查询

![image-20240509211220645](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240509211220645.png)

### 3.1 安装

`bind-chroot:` chroot模式下运行在相对根路径下如/var/named/chroot下

`bind-utils: ` DNS查询工具，dig nslookup host等

```
yum -y install bind bind-chroot bind-utils
```

### 3.2 配置文件解析

配置文件主要分为主配置文件和域数据记录文件，主配置配置文件主要告知客户端去哪里找对应的域数据记录文件；域数据记录文件只要是记录域名与ip的解析记录。



一般配置文件主配置文件在/etc/named.conf,在chroot下路径则在/var/named/chroot/etc/named.conf。

* directory: 工作目录
* dump-file： 运行rndc dumpdn备份缓存资料后保持的文件路径和名称
*  listen-on port： 监听的端口
* allow-query: 允许哪些主机可以查询
* allow-query-cache： 允许哪些主机可以查询非权威解析数据，就是递归查询
* blackhole: 黑名单
* forwards: 转发，对本服务器将指向设置的IP进行解析
* max-cache-size: 缓存文件最大容量
*  allow-transfer： 执行下一个dns服务器

zone语句选项

| 选项            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| type            | hint 当本地找不到解析后，去找根域<br/>	master 权威域<br/>	slave 辅助域<br/>	forward: 定义转发域名服务器 |
| file 域数据文件 | 域数据文件                                                   |
| notify          | 域数据文件更新后，是否主动通知其他域名服务器                 |
| masters         | 定义主域名服务器，当type slave才有效                         |
| allow-update    | 允许哪些主机动态更新域数据信息                               |
| allow-transfer  | 哪些从服务器可以下载主服务器的数据文件                       |



```
options
{
        // Put files that named is allowed to write in the data/ directory:
        directory               "/var/named";           // "Working" directory
        dump-file               "data/cache_dump.db";
        statistics-file         "data/named_stats.txt";
        memstatistics-file      "data/named_mem_stats.txt";
        recursing-file          "data/named.recursing";
        secroots-file           "data/named.secroots";


        listen-on port 53       { any; };
        allow-query             { any; };
        allow-query-cache       { any; };
        recursion yes;

};

acl secondserver {
  192.168.0.0/24;
};


zone "." IN {
    type hint;
    file "/var/named/named.ca";
};


zone "abc.com" IN {
    type master;
    allow-transfer { secondserver };
    file "abc.com.zone";
};


zone "0.168.192.in-addr-arpa" IN {
    type master;
    allow-transfer { secondserver };
    file "192.168.0.zone";
};
```



域数据记录类型

| 记录选项 | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| SOA      | 域权威记录，说明本机服务器为该域的管理服务器                 |
| NS       | 域名服务器记录                                               |
| A        | 正向解析 ipv4                                                |
| AAA      | 正向解析 ipv6                                                |
| PTR      | 反向解析 ip和域名的                                          |
| CNAME    | 别名                                                         |
| MX       | 邮件记录，指定域内的邮寄服务器，指定优先级，数值越大优先级越低 |

正向解析域数据记录文件：/var/named/chroot/var/named/abc.com.zone

注意：主机名前面不能有空格哦

```
$TTL 1D
@       IN SOA  dns1.abc.com. jacob.abc.com (
                                        10      ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      dns1.abc.com.
        NS      dns2.abc.com.
        MX  10  mail.abc.com.
dns1 IN A 192.168.0.203
dns2 IN A 192.168.0.202
fileserver IN A 192.168.0.203
www IN A 192.168.0.200
            IN A 192.168.0.201
```

反向解析域数据记录文件：/var/named/chroot/var/named/192.168.0.zone

```
$TTL 1D
@       IN SOA  dns1.abc.com. jacob.abc.com (
                                        10      ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      dns1.abc.com.
        NS      dns2.abc.com.
        203 IN PTR dns1.abc.com.
        202 IN PTR dns2.abc.com.
        203 IN PTR fileserver.abc.com.
```

重启named服务

```
# systemctl restart named
#开机自启动
#systemctl enable named
Created symlink from /etc/systemd/system/multi-user.target.wants/named.service to /usr/lib/systemd/system/named.service.
```

### 3.3 DNS从服务器

#### 3.3.1 安装

```
yum -y install bind bind-chroot bind-utils
```

#### 3.3.2 从DNS配置文件

从服务器配置需要注意：type为slave ，masters为主dns的ip，可以单独创建个slaves目录存放同步下来的域数据记录文件。权限给775

```
options
{
        // Put files that named is allowed to write in the data/ directory:
        directory               "/var/named";           // "Working" directory
        dump-file               "data/cache_dump.db";
        statistics-file         "data/named_stats.txt";
        memstatistics-file      "data/named_mem_stats.txt";
        recursing-file          "data/named.recursing";
        secroots-file           "data/named.secroots";


        listen-on port 53       { any; };
        allow-query             { any; };
        allow-query-cache       { any; };
        recursion yes;

};


zone "." IN {
    type hint;
    file "/var/named/named.ca";
};


zone "mark.com" IN {
    type slave;
    masters { 192.168.0.203; };
    file "slaves/mark.com.zone";
};


zone "0.168.192.in-addr-arpa" IN {
    type slave;
    masters { 192.168.0.203; };
    file "slaves/192.168.0.zone";
};
```

可能出现的报错？需要移动named.ca到"/var/named/named.ca"

* 重启named-chroot

```
systemctl restart  named-chroot

systemctl enable named-chroot
```



## 4. Ngnix







# 高级应用

## 1. LVS

lvs虚拟服务器被集成到Linux内核模块中，基于IP的数据请求负载均衡调度方案。

`三种工作模式`

* NAT模式

网络地址转换，通过对数据报头的修改，使企业私有IP可以访问外网

* TUN模式
* DR模式
