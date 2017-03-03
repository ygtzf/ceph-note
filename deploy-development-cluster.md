# 开发者的虚拟ceph cluster

## 在开发环境中搭建了一个虚拟的ceph cluster
### 搭建过程:
很简单，官网有说明: http://docs.ceph.com/docs/master/dev/quick_guide/

1. 肯定是先得有ceph源码了，ceph源码在github上管理，可以直接clone:

   git clone https://github.com/ceph/ceph.git

2. 在源码顶级目录下, 直接先执行 run-make-check.sh

   ygt@ygt:~/work/ceph/source/ceph-source$ ./run-make-check.sh
   这个脚本主要就是一个任务: 编译ceph:
   1. 检查依赖，安装依赖，install-dep.sh脚本负责。
   2. 编译ceph，所有编译出的ceph组件都是带debug选项编译的。本人还加了-g3 -O0（CFLAGS="-Wall -g3 -O0" CXXFLAGS="-Wall -g3 -O0"）,以方便gdb跟踪。
      在编译的时候，该脚本会看看所在pc的cpu核数，然后多核编译: -jX (X为CPU核数)
   3. 跑很多单元测试，以保证ceph源码的函数都OK
   这个脚本跑完挺长的，一方面是编译的时间，另外就是unittest花的时间更长。

3. 现在就可以直接跑虚拟ceph cluster：

   ygt@ygt:~/work/ceph/source/ceph-source/src$ MON=1 MDS=1 ./vstart.sh -d -n -x
   其中MON MDS都是环境变量，环境变量可以是: OSD,MDS,MON,RGW，这些环境变量的设置是指相应的服务实例个数(也就是该虚拟环境中每种组件跑几个服务）
   -d(debug): 以debug模式运行，很多都debug level都设置为: 20/20
   -n(new): 创建一个新的集群
   -x: 使用cephx认证
   还有更多的参数, 可以直接通过./vstart.sh --help来查看。
   这样虚拟环境就起来了，下面先看看这个集群的样子(新媳妇要见人了 :) )

4. ygt@ygt:~/work/ceph/source/ceph-source/src$ ./ceph -s

(注意路径，我的ceph编译完直接在src下，这样不太好，应该新建一个build目录，最后这些都在build下)
```
    cluster 23eef462-815c-4fbc-b33f-95ae25b3e16a
     health HEALTH_OK
     monmap e1: 1 mons at {a=127.0.0.1:6789/0}
            election epoch 3, quorum 0 a
      fsmap e5: 1/1/1 up {0=a=up:active}
     osdmap e22: 3 osds: 3 up, 3 in
            flags sortbitwise,require_jewel_osds
      pgmap v929: 24 pgs, 3 pools, 2068 bytes data, 20 objects
            289 GB used, 2298 GB / 2726 GB avail
                  24 active+clean
```
5. 停止这个虚拟集群:

   ygt@ygt:~/work/ceph/source/ceph-source/src$  ./stop.sh

6. 再次启动这个集群:

   ygt@ygt:~/work/ceph/source/ceph-source/src$ MON=1 MDS=1 ./vstart.sh -d -x
   注意: 不要加-n，不要new，不过也没啥关系，就是开发环境，随便搞。

7. 要想摧毁整个环境，需要将dev和out两个目录删除了。

【注】

1. 配置文件在src/ceph.conf, 可以看见把相关的debug level都调到最高了
2. 所有的log都放在了src/out目录下
3. 所有的mon、osd、mds、rgw的数据都方在了src/dev下，相当于真实环境中的/var/lib/ceph
4. 虚拟环境中的所有进程都要手动控制，没有service，因为它不是服务:
   ./ceph-mon -i a -c /home/ygt/work/ceph/source/ceph-source/src/ceph.conf
   ./ceph-osd -i 0 -c /home/ygt/work/ceph/source/ceph-source/src/ceph.conf
   ./ceph-osd -i 1 -c /home/ygt/work/ceph/source/ceph-source/src/ceph.conf
   ./ceph-osd -i 2 -c /home/ygt/work/ceph/source/ceph-source/src/ceph.conf
   ./ceph-mds -i a -c /home/ygt/work/ceph/source/ceph-source/src/ceph.conf

### 问题
1. 三个osd都起不来: log输出为:(ceph/src/out/xxx.log)
```
2017-03-03 15:18:34.960368 7fe100de5800 -1 osd.0 0 backend (filestore) is unable to support max object name[space] len
2017-03-03 15:18:34.960372 7fe100de5800 -1 osd.0 0    osd max object name len = 2048
2017-03-03 15:18:34.960373 7fe100de5800 -1 osd.0 0    osd max object namespace len = 256
2017-03-03 15:18:34.960374 7fe100de5800 -1 osd.0 0 (36) File name too long
2017-03-03 15:18:34.981221 7fe100de5800 -1  ** ERROR: osd init failed: (36) File name too long
```
> 解决:
> Starting with the Jewel release, the ceph-osd daemon will refuse to start if the configured max object name cannot be safely stored on ext4. If the cluster is only being used with short object names (e.g., RBD only), you can continue using ext4 by setting the following configuration option:
> 
> osd max object name len = 256
> osd max object namespace len = 64
