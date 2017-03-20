* 通常情况下，想要将内存当做普通的磁盘使用，只需挂载一下tmpfs就可以:
> mount -t tmpfs -o size=10G tmpfs /mnt/ram

* 现在来试验一下将内存做为一块日志盘来用
```
# step 1. 创建一个目录
root@server3:~# mkdir /mnt/ram 
# step 2. 挂载内存到创建的目录
root@server3:~# mount -t tmpfs -o size=10G tmpfs /mnt/ram
# step 3. 创建一个指定大小的文件
root@server3:~# cd /mnt/ram/
root@server3:/mnt/ram# dd if=/dev/zero of=journal bs=10G count=1 
0+1 records in
0+1 records out
2147479552 bytes (2.1 GB) copied, 1.82554 s, 1.2 GB/s
# step 4. 创建一个块设备(mknod的用法可以看man手册)
root@server3:~# mknod /dev/fake-journal b 7 201 
# step 5. 修改该块设备权限为可读可写
root@server3:~# chmod 666 /dev/fake-journal 
# step 6. 创建一个loop设备
root@server3:~# losetup /dev/fake-journal /mnt/ram/journal
# 接下来就是ceph安装的步骤了，创建一个journal盘， 然后和osd关联起来
root@server3:~# ceph-osd -i 0 --mkfs --mkkey --osd-journal=/dev/fake-journal --mkjournal
 HDIO_DRIVE_CMD(identify) failed: Invalid argument
2017-03-16 16:20:32.727585 7fb58741c800 -1 journal check: ondisk fsid 00000000-0000-0000-0000-000000000000 doesn't match expected 538476d2-8344-4b90-8263-5309581809b6, invalid (someone else's?) journal
 HDIO_DRIVE_CMD(identify) failed: Invalid argument
 HDIO_DRIVE_CMD(identify) failed: Invalid argument
 HDIO_DRIVE_CMD(identify) failed: Invalid argument
2017-03-16 16:20:32.755011 7fb58741c800 -1 created object store /var/lib/ceph/osd/ceph-0 for osd.0 fsid cbc99ef9-fbc3-41ad-a726-47359f8d84b3
2017-03-16 16:20:32.755266 7fb58741c800 -1 already have key in keyring /var/lib/ceph/osd/ceph-0/keyring

root@server3:~# ln -s /dev/fake-journal /var/lib/ceph/osd/ceph-0/journal

root@server3:~# start ceph-osd id=0
ceph-osd (ceph/0) start/running, process 3298

root@server3:~# ceph -s
    cluster cbc99ef9-fbc3-41ad-a726-47359f8d84b3
     health HEALTH_OK
     monmap e1: 1 mons at {server3=192.168.36.201:6789/0}
            election epoch 3, quorum 0 server3
     osdmap e7: 1 osds: 1 up, 1 in
            flags sortbitwise,require_jewel_osds
      pgmap v9: 64 pgs, 1 pools, 0 bytes data, 0 objects
            100952 kB used, 3723 GB / 3723 GB avail
                  64 active+clean
```
* **说明:**
1.  tmpfs
tmpfs是一种虚拟内存文件系统，不是块设备，它是基于内存的文件系统，其最大特点是它的存储空间在VM（virtual memory 虚拟内存）。
linux下面VM的大小由RM(Real Memory)和swap组成,RM的大小就是物理内存的大小，而Swap的大小是由自己决定的。
&emsp;&emsp;Swap是通过硬盘虚拟出来的内存空间，因此它的读写速度相对RM(Real Memory）要慢许多，当一个进程申请一定数量的内存时，如内核的vm子系统发现没有足够的RM时，就会把RM里面的一些不常用的数据交换到Swap里面，如果需要重新使用这些数据再把它们从Swap交换到RM里面。如果有足够大的物理内存，可以不划分Swap分区。
&emsp;&emsp;VM由RM+Swap两部分组成，因此tmpfs最大的存储空间可达（The size of RM + The size of Swap）。 但是对于tmpfs本身而言，它并不知道自己使用的空间是RM还是Swap，这一切都是由内核的vm子系统管理的。
&emsp;&emsp;tmpfs默认的大小是RM的一半，假如你的物理内存是1024M，那么tmpfs默认的大小就是512M
一般情况下，是配置的小于物理内存大小的。
&emsp;&emsp;tmpfs配置的大小并不会真正的占用这块内存，如果/dev/shm/下没有任何文件，它占用的内存实际上就是0字节；如果它最大为1G，里头放有100M文件，那剩余的900M仍然可为其它应用程序所使用，但它所占用的100M内存，是不会被系统回收重新划分的。
&emsp;&emsp;当删除tmpfs中文件，tmpfs 文件系统驱动程序会动态地减小文件系统并释放 VM 资源。
&emsp;&emsp;我们看到通常的linux系统都有tmpfs：
	```
	ygt@ygt:~$ df
	文件系统       1K-blocks      已用      可用 已用% 挂载点
	/dev/sda1      953128852 105999832 798689828   12% /
	none                   4         0         4    0% /sys/fs/cgroup
	udev             8164660         4   8164656    1% /dev
	tmpfs            1635108      1288   1633820    1% /run
	```
	
2.  loop设备
&emsp;&emsp;在类 UNIX 系统里，loop 设备是一种伪设备(pseudo-device)，或者也可以说是仿真设备。它能使我们像块设备一样访问一个文件。
&emsp;&emsp;在使用之前，一个 loop 设备必须要和一个文件进行连接。这种结合方式给用户提供了一个替代块特殊文件的接口。因此，如果这个文件包含有一个完整的文件系统，那么这个文件就可以像一个磁盘设备一样被 mount 起来。
&emsp;&emsp;上面说的文件格式，我们经常见到的是 CD 或 DVD 的 ISO 光盘镜像文件或者是软盘(硬盘)的 *.img 镜像文件。通过这种 loop mount (回环mount)的方式，这些镜像文件就可以被 mount 到当前文件系统的一个目录下。
&emsp;&emsp;loop 含义：对于第一层文件系统，它直接安装在我们计算机的物理设备之上；而对于这种被 mount 起来的镜像文件(它也包含有文件系统)，它是建立在第一层文件系统之上，这样看来，它就像是在第一层文件系统之上再绕了一圈的文件系统，所以称为 loop。

3. losetup命令
&emsp;&emsp;此命令用来设置循环设备。循环设备可把文件虚拟成块设备，籍此来模拟整个文件系统，让用户得以将其视为硬盘驱动器，光驱或软驱等设备，并挂入当作目录来使用。

