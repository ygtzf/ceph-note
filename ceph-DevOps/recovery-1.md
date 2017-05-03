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
日志盘的修复其实就是更换日志盘, 见另一篇文章: links

## osd重启
启动osd服务, 但是有10个osd起不来, 接下来就逐个分析原因, 个个击破.

## osd恢复
1. 有两个osd的leveldb出错, 具体log没保留下来, 很遗憾.
   解决:
     download leveldb python 模块, 可以使用pip, 可以下载源码安装, 链接: https://pypi.python.org/pypi/leveldb
     修复leveldb: RepairDB的参数是leveldb的具体存储路径, 对于osd, 就是current/omap
     ```
     root@host3:~/leveldb-0.20# python
	 Python 2.7.6 (default, Oct 26 2016, 20:30:19)
     [GCC 4.8.4] on linux2
     Type "help", "copyright", "credits" or "license" for more information.
     >>> import leveldb
     >>>
     >>>
     >>> leveldb.RepairDB("/var/lib/ceph/osd/ceph-44/current/omap/")
     >>>
     ```

**接下来的几个问题一直没有解决, 最重要的是osd直接crash, 无从谈及后续的pg recovery, 虽然尝试了很多方法, 包括把问题从正常的sod中导出, 然后加入进来, 都不起作用. 好在这是一个测试环境, 时间紧张, 我们只能先放下, 转而使用多cephfs的特性来继续测试cephfs, 后续还要跟代码来看看问题所在.**
2. osd log如下:
```
    -7> 2017-03-31 12:27:34.025244 7fdaeb424800 15 filestore(/var/lib/ceph/osd/ceph-47) omap_get_values 1.1daa_head/#1:55b80000::::head#
    -6> 2017-03-31 12:27:34.025350 7fdaeb424800 15 filestore(/var/lib/ceph/osd/ceph-47) omap_get_values 1.1daa_head/#1:55b80000::::head# = -1
    -5> 2017-03-31 12:27:34.025356 7fdaeb424800 15 filestore(/var/lib/ceph/osd/ceph-47) collection_getattr /var/lib/ceph/osd/ceph-47/current/1.1daa_head 'remove' len 1
    -4> 2017-03-31 12:27:34.025378 7fdaeb424800 10 filestore(/var/lib/ceph/osd/ceph-47) collection_getattr /var/lib/ceph/osd/ceph-47/current/1.1daa_head 'remove' len 1 = -61
    -3> 2017-03-31 12:27:34.025381 7fdaeb424800 10 osd.47 8625 pgid 1.1daa coll 1.1daa_head
    -2> 2017-03-31 12:27:34.025385 7fdaeb424800 15 filestore(/var/lib/ceph/osd/ceph-47) omap_get_values 1.1daa_head/#1:55b80000::::head#
    -1> 2017-03-31 12:27:34.025447 7fdaeb424800 15 filestore(/var/lib/ceph/osd/ceph-47) omap_get_values 1.1daa_head/#1:55b80000::::head# = -1
     0> 2017-03-31 12:27:34.027079 7fdaeb424800 -1 osd/PG.cc: In function 'static int PG::peek_map_epoch(ObjectStore*, spg_t, epoch_t*, ceph::bufferlist*)' thread 7fdaeb424800 time 2017-03-31 12:27:34.025452
osd/PG.cc: 2947: FAILED assert(0 == "unable to open pg metadata")

 ceph version 10.2.6 (656b5b63ed7c43bd014bcafd81b001959d5f089f)
 1: (ceph::__ceph_assert_fail(char const*, char const*, int, char const*)+0x8b) [0x7fdaeaeba26b]
 2: (PG::peek_map_epoch(ObjectStore*, spg_t, unsigned int*, ceph::buffer::list*)+0x727) [0x7fdaea906c97]
 3: (OSD::load_pgs()+0x8bd) [0x7fdaea86840d]
 4: (OSD::init()+0x1f74) [0x7fdaea87a734]
 5: (main()+0x29d1) [0x7fdaea7e1d71]
 6: (__libc_start_main()+0xf5) [0x7fdae72a9ec5]
 7: (()+0x36a957) [0x7fdaea82a957]
 NOTE: a copy of the executable, or `objdump -rdS <executable>` is needed to interpret this.
```
   未解决......

3. osd log如下:
```
    -3> 2017-04-01 10:24:08.243324 7f10393eb700  0 log_channel(cluster) log [INF] : 1.1fb5 starting backfill to osd.51 from (0'0,0'0] MAX to 8625'202097
    -2> 2017-04-01 10:24:08.244198 7f10393eb700  5 osd.65 pg_epoch: 14087 pg[1.1fb5( v 8625'202097 (8612'199066,8625'202097] local-les=14087 n=87794 ec=350 les/c/f 12937/12937/0 14082/14086/14086) [65,51]/[65,57] r=0 lpr=14086 pi=9325-14085/17 bft=51 crt=8625'202097 lcod 0'0 mlcod 0'0 activating+remapped] enter Started/Primary/Active/Activating
    -1> 2017-04-01 10:24:08.244211 7f1036be6700  0 log_channel(cluster) log [INF] : 1.14 starting backfill to osd.51 from (0'0,0'0] MAX to 8625'201676
     0> 2017-04-01 10:24:08.246532 7f1037be8700 -1 osd/PGLog.h: In function 'void PGLog::IndexedLog::claim_log_and_clear_rollback_info(const pg_log_t&)' thread 7f1037be8700 time 2017-04-01 10:24:08.243637
osd/PGLog.h: 111: FAILED assert(rollback_info_trimmed_to_riter == log.rbegin())

 ceph version 10.2.6 (656b5b63ed7c43bd014bcafd81b001959d5f089f)
 1: (ceph::__ceph_assert_fail(char const*, char const*, int, char const*)+0x8b) [0x7f10824d026b]
 2: (PG::RecoveryState::Stray::react(PG::MLogRec const&)+0x63e) [0x7f1081f497ae]
 3: (boost::statechart::simple_state<PG::RecoveryState::Stray, PG::RecoveryState::Started, boost::mpl::list<mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na, mpl_::na>, (boost::statechart::history_mode)0>::react_impl(boost::statechart::event_base const&, void const*)+0x1f4) [0x7f1081f84044]
 4: (boost::statechart::state_machine<PG::RecoveryState::RecoveryMachine, PG::RecoveryState::Initial, std::allocator<void>, boost::statechart::null_exception_translator>::send_event(boost::statechart::event_base const&)+0x5b) [0x7f1081f6e38b]
 5: (PG::handle_peering_event(std::shared_ptr<PG::CephPeeringEvt>, PG::RecoveryCtx*)+0x1d5) [0x7f1081f367b5]
 6: (OSD::process_peering_events(std::list<PG*, std::allocator<PG*> > const&, ThreadPool::TPHandle&)+0x249) [0x7f1081e953a9]
 7: (OSD::PeeringWQ::_process(std::list<PG*, std::allocator<PG*> > const&, ThreadPool::TPHandle&)+0x12) [0x7f1081edd242]
 8: (ThreadPool::worker(ThreadPool::WorkThread*)+0xa5e) [0x7f10824c17ce]
 9: (ThreadPool::WorkThread::entry()+0x10) [0x7f10824c26b0]
 10: (()+0x8184) [0x7f10809be184]
 11: (clone()+0x6d) [0x7f107e998bed]
 NOTE: a copy of the executable, or `objdump -rdS <executable>` is needed to interpret this.
```
	未解决......

4. osd log如下:
```
    -3> 2017-04-01 09:07:03.256009 7f087f7e8700  1 osd.10 pg_epoch: 13637 pg[1.498( v 8625'201557 (8612'198458,8625'201557] local-les=13512 n=87583 ec=350 les/c/f 13512/13512/0 13636/13637/9027) [39,10] r=1 lpr=13637 pi=981-13636/70 crt=8625'201557 lcod 0'0 inactive NOTIFY] state<Start>: transitioning to Stray
    -2> 2017-04-01 09:07:03.256570 7f087f7e8700  1 osd.10 pg_epoch: 13637 pg[1.ae4( v 8625'201280 (8612'198271,8625'201280] local-les=13531 n=87307 ec=350 les/c/f 13531/13532/0 13636/13637/9026) [15,10] r=1 lpr=13637 pi=951-13636/70 crt=8625'201280 lcod 0'0 inactive NOTIFY] state<Start>: transitioning to Stray
    -1> 2017-04-01 09:07:03.257173 7f087efe7700  1 osd.10 pg_epoch: 13637 pg[1.4e7( v 8625'200805 (8612'197750,8625'200805] local-les=13631 n=87570 ec=350 les/c/f 13512/9635/0 13636/13637/13636) [10,73]/[10] r=0 lpr=13637 pi=9634-13636/42 crt=8625'200805 lcod 0'0 mlcod 0'0 remapped] state<Start>: transitioning to Primary
     0> 2017-04-01 09:07:03.267047 7f0874fff700 -1 osd/ReplicatedPG.cc: In function 'virtual void ReplicatedPG::on_local_recover(const hobject_t&, const ObjectRecoveryInfo&, ObjectContextRef, ObjectStore::Transaction*)' thread 7f0874fff700 time 2017-04-01 09:07:03.263494
osd/ReplicatedPG.cc: 209: FAILED assert(is_primary())

 ceph version 10.2.6 (656b5b63ed7c43bd014bcafd81b001959d5f089f)
 1: (ceph::__ceph_assert_fail(char const*, char const*, int, char const*)+0x8b) [0x7f08cc6d726b]
 2: (ReplicatedPG::on_local_recover(hobject_t const&, ObjectRecoveryInfo const&, std::shared_ptr<ObjectContext>, ObjectStore::Transaction*)+0x6c1) [0x7f08cc1acda1]
 3: (ReplicatedBackend::handle_push(pg_shard_t, PushOp&, PushReplyOp*, ObjectStore::Transaction*)+0x1f2) [0x7f08cc251d52]
 4: (ReplicatedBackend::_do_push(std::shared_ptr<OpRequest>)+0x11c) [0x7f08cc25202c]
 5: (ReplicatedBackend::handle_message(std::shared_ptr<OpRequest>)+0x3f6) [0x7f08cc260e66]
 6: (ReplicatedPG::do_request(std::shared_ptr<OpRequest>&, ThreadPool::TPHandle&)+0xed) [0x7f08cc1bd28d]
 7: (OSD::dequeue_op(boost::intrusive_ptr<PG>, std::shared_ptr<OpRequest>, ThreadPool::TPHandle&)+0x3f5) [0x7f08cc07bc85]
 8: (PGQueueable::RunVis::operator()(std::shared_ptr<OpRequest>&)+0x5d) [0x7f08cc07bead]
 9: (OSD::ShardedOpWQ::_process(unsigned int, ceph::heartbeat_handle_d*)+0x869) [0x7f08cc0808c9]
 10: (ShardedThreadPool::shardedthreadpool_worker(unsigned int)+0x877) [0x7f08cc6c7767]
 11: (ShardedThreadPool::WorkThreadSharded::entry()+0x10) [0x7f08cc6c9690]
 12: (()+0x8184) [0x7f08cabc5184]
 13: (clone()+0x6d) [0x7f08c8b9fbed]
 NOTE: a copy of the executable, or `objdump -rdS <executable>` is needed to interpret this.
```
	未解决......

## 尝试解决
1. 找到问题pg, 从它的所在正常的osd中来导出pg, 再导入有问题的osd.
2. 直接force_create问题pg
