# 第一次大型recovery经历

## 背景

1. 这是一个测试环境。
2. 该环境中是cephfs
3. 一共12个节点， 2个client、2个mds、8个osd
4. mds： 2颗CPU，每个4核，一共是8核。 128G内存， 单独的两个节点，只作为mds
   cpu型号： Intel(R) Xeon(R) CPU E5-1620 v3 @ 3.50GHz
5. osd节点， 每个24核， 8 × 4T SATA盘， 1 SSD：INTEL SSD SC2BB48 (480G) 64G内存
   cpu型号: Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz
   其中，有两个osd各有3块nvme SSD，

6. 3个nvme SSD，每个分4个分区，两个journal，两个osd，一共是6个osd来做为metadata pool

## 测试任务

1. 10亿个小文件（2M-4M），最终我们选择了256K的文件， 因为为了达到文件数量， 只能选择小文件， 否则容量不够。
2. 到出现问题的时候， ceph cluster有7亿多的文件。cluster正常。


## 意外

1. 由于测试环境的物理条件限制，温度过高，跳闸了。悲剧发生，我们的raid卡用的cache没带电池，物理机开启后，8台osd节点，86个osd，一共有40块左右磁盘都故障了， 无法mount。

## 磁盘恢复

1. 问题： 磁盘文件系统损坏, mount不成功(error message: log mount/recovery failed: error -117); 即使mount成功, 进入挂载目录, ls会显示input/output error.

   解决: 最初想使用如下的步骤来修复disk:
        1. xfs_check /dev/sdb1
        2. xfs_metadump /dev/sdb1 sdb1.meta
        3. xfs_mdrestore sdb1.meta sdb1.img
        4. xfs_repair sdb1.img

   结果, 在第二步就报错过不去, 所以只能强制修复了:
```
root@host2:~# xfs_repair /dev/sdb1
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
ERROR: The filesystem has valuable metadata changes in a log which needs to
be replayed.  Mount the filesystem to replay the log, and unmount it before
re-running xfs_repair.  If you are unable to mount the filesystem, then use
the -L option to destroy the log and attempt a repair.
Note that destroying the log may cause corruption -- please attempt a mount
of the filesystem before doing this.
// -L选项 表示强制修复, 如果遇到dirty log, 也要强制设为0, 很危险, 容易丢数据, 由于是测试环境, 没办法, 只能这样.
# xfs_repair -L /dev/sdb1
```
2. 当时断电后, 有3台机器, 一直可以ping通, 但是就是不能ssh连接, log如下:
```
[Thu Mar 30 12:05:00 2017] INFO: task mount:1984 blocked for more than 120 seconds.
[Thu Mar 30 12:05:00 2017]       Not tainted 3.19.0-25-generic #26~14.04.1-Ubuntu
[Thu Mar 30 12:05:00 2017] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[Thu Mar 30 12:05:00 2017] mount           D ffff881059927c28     0  1984   1889 0x00000000
[Thu Mar 30 12:05:00 2017]  ffff881059927c28 ffff88085af16bf0 0000000000013e80 ffff881059927fd8
[Thu Mar 30 12:05:00 2017]  0000000000013e80 ffff88105c265850 ffff88085af16bf0 ffff88084b57c000
[Thu Mar 30 12:05:00 2017]  ffff88085b302900 ffff88084b57c000 ffff88085b302940 ffff88085b302968
[Thu Mar 30 12:05:00 2017] Call Trace:
[Thu Mar 30 12:05:00 2017]  [<ffffffff817b22e9>] schedule+0x29/0x70
[Thu Mar 30 12:05:00 2017]  [<ffffffffc0927a59>] xfs_ail_push_all_sync+0xa9/0xe0 [xfs]
[Thu Mar 30 12:05:00 2017]  [<ffffffff810b4e10>] ? prepare_to_wait_event+0x110/0x110
[Thu Mar 30 12:05:00 2017]  [<ffffffffc091d6e7>] xfs_log_quiesce+0x37/0x70 [xfs]
[Thu Mar 30 12:05:00 2017]  [<ffffffffc091d73a>] xfs_log_unmount+0x1a/0x70 [xfs]
[Thu Mar 30 12:05:00 2017]  [<ffffffffc0912c45>] xfs_mountfs+0x5e5/0x760 [xfs]
[Thu Mar 30 12:05:00 2017]  [<ffffffffc0915f06>] xfs_fs_fill_super+0x2c6/0x340 [xfs]
[Thu Mar 30 12:05:00 2017]  [<ffffffff811ef8a0>] mount_bdev+0x1b0/0x1f0
[Thu Mar 30 12:05:00 2017]  [<ffffffffc0915c40>] ? xfs_parseargs+0xbe0/0xbe0 [xfs]
[Thu Mar 30 12:05:00 2017]  [<ffffffffc0913ed5>] xfs_fs_mount+0x15/0x20 [xfs]
[Thu Mar 30 12:05:00 2017]  [<ffffffff811f01f9>] mount_fs+0x39/0x1b0
[Thu Mar 30 12:05:00 2017]  [<ffffffff8120b6bb>] vfs_kern_mount+0x6b/0x120
[Thu Mar 30 12:05:00 2017]  [<ffffffff8120e444>] do_mount+0x204/0xb30
[Thu Mar 30 12:05:00 2017]  [<ffffffff8120e0ea>] ? copy_mount_options+0x3a/0x160
[Thu Mar 30 12:05:00 2017]  [<ffffffff8120f06b>] SyS_mount+0x8b/0xe0
[Thu Mar 30 12:05:00 2017]  [<ffffffff817b668d>] system_call_fastpath+0x16/0x1b
[Thu Mar 30 12:07:01 2017] INFO: task mount:1984 blocked for more than 120 seconds.
[Thu Mar 30 12:07:01 2017]       Not tainted 3.19.0-25-generic #26~14.04.1-Ubuntu
[Thu Mar 30 12:07:01 2017] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[Thu Mar 30 12:07:01 2017] mount           D ffff881059927c28     0  1984   1889 0x00000000
[Thu Mar 30 12:07:01 2017]  ffff881059927c28 ffff88085af16bf0 0000000000013e80 ffff881059927fd8
[Thu Mar 30 12:07:01 2017]  0000000000013e80 ffff88105c265850 ffff88085af16bf0 ffff88084b57c000
[Thu Mar 30 12:07:01 2017]  ffff88085b302900 ffff88084b57c000 ffff88085b302940 ffff88085b302968
[Thu Mar 30 12:07:01 2017] Call Trace:
[Thu Mar 30 12:07:01 2017]  [<ffffffff817b22e9>] schedule+0x29/0x70
[Thu Mar 30 12:07:01 2017]  [<ffffffffc0927a59>] xfs_ail_push_all_sync+0xa9/0xe0 [xfs]
[Thu Mar 30 12:07:01 2017]  [<ffffffff810b4e10>] ? prepare_to_wait_event+0x110/0x110
[Thu Mar 30 12:07:01 2017]  [<ffffffffc091d6e7>] xfs_log_quiesce+0x37/0x70 [xfs]
[Thu Mar 30 12:07:01 2017]  [<ffffffffc091d73a>] xfs_log_unmount+0x1a/0x70 [xfs]
[Thu Mar 30 12:07:01 2017]  [<ffffffffc0912c45>] xfs_mountfs+0x5e5/0x760 [xfs]
[Thu Mar 30 12:07:01 2017]  [<ffffffffc0915f06>] xfs_fs_fill_super+0x2c6/0x340 [xfs]
[Thu Mar 30 12:07:01 2017]  [<ffffffff811ef8a0>] mount_bdev+0x1b0/0x1f0
[Thu Mar 30 12:07:01 2017]  [<ffffffffc0915c40>] ? xfs_parseargs+0xbe0/0xbe0 [xfs]
[Thu Mar 30 12:07:01 2017]  [<ffffffffc0913ed5>] xfs_fs_mount+0x15/0x20 [xfs]
[Thu Mar 30 12:07:01 2017]  [<ffffffff811f01f9>] mount_fs+0x39/0x1b0
[Thu Mar 30 12:07:01 2017]  [<ffffffff8120b6bb>] vfs_kern_mount+0x6b/0x120
[Thu Mar 30 12:07:01 2017]  [<ffffffff8120e444>] do_mount+0x204/0xb30
[Thu Mar 30 12:07:01 2017]  [<ffffffff8120e0ea>] ? copy_mount_options+0x3a/0x160
[Thu Mar 30 12:07:01 2017]  [<ffffffff8120f06b>] SyS_mount+0x8b/0xe0
[Thu Mar 30 12:07:01 2017]  [<ffffffff817b668d>] system_call_fastpath+0x16/0x1b
```

可能是磁盘损坏, 而ceph-osd进程一直在试图mount, mount不成功, 阻塞, 操作系统发现进程超时, 只能调整hung_task_timeout_secs,
我们再深入的研究这个问题, 但基本上断定: 这个时候系统的CPU忙于这些, 连ssh都不处理. 我们认为这是操作系统的问题, 后续需要仔细研究一下这个问题.

最后我们只能到现场或通过IPMI, 强制重启机器, 进入安全模式, 来修复损坏的磁盘.
之所以要进入安全模式, 是因为正常模式进入后, 还是上面的问题, 我们在其他可以ssh进入的机器上, 先禁止了ceph服务的自启动:
```
root@host2:~# update-rc.d ceph disable
 Disabling system startup links for /etc/init.d/ceph ...
 Removing any system startup links for /etc/init.d/ceph ...
   /etc/rc0.d/K20ceph
   /etc/rc1.d/K20ceph
   /etc/rc2.d/S20ceph
   /etc/rc3.d/S20ceph
   /etc/rc4.d/S20ceph
   /etc/rc5.d/S20ceph
   /etc/rc6.d/K20ceph
 Adding system startup for /etc/init.d/ceph ...
   /etc/rc0.d/K20ceph -> ../init.d/ceph
   /etc/rc1.d/K20ceph -> ../init.d/ceph
   /etc/rc6.d/K20ceph -> ../init.d/ceph
   /etc/rc2.d/K80ceph -> ../init.d/ceph
   /etc/rc3.d/K80ceph -> ../init.d/ceph
   /etc/rc4.d/K80ceph -> ../init.d/ceph
   /etc/rc5.d/K80ceph -> ../init.d/ceph
```

## 日志盘修复

## osd重启

## osd恢复

