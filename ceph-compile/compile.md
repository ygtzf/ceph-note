# ceph源码编译问题总结

* 当前编译选项:
  --with-jemalloc --without-tcmalloc --without-librocksdb-static --with-rados  --with-radosgw  --with-radosstriper

* 编译Hammer版本，是要关掉rocksdb这个option的，也就是编译的时候带option: --without-librocksdb-static
编译Jewel版本，是要打开rocksdb，好像是说bluestor要用， 编译的时候要将上年的option改为: --with-librocksdb-static

* 在编译10.2.5的时候出现下列问题：
  root@ygt-ceph:/mnt/ceph-compile/ceph# dpkg-buildpackage -j4
  dpkg-buildpackage: source package ceph
  dpkg-buildpackage: source version 10.2.5-1
  dpkg-buildpackage: source distribution stable
  dpkg-buildpackage: source changed by Alfredo Deza <adeza@redhat.com>
  dpkg-buildpackage: host architecture amd64
  dpkg-source --before-build ceph
  debian/rules clean
  dh_testdir
  dh_testroot
  ......
  make[2]: Leaving directory `/mnt/ceph-compile/ceph'
  Making distclean in src
  make[2]: Entering directory `/mnt/ceph-compile/ceph/src'
  Making distclean in gmock
  make[3]: Entering directory `/mnt/ceph-compile/ceph/src/gmock'
  make[3]: *** No rule to make target `distclean'.  Stop.
  make[3]: Leaving directory `/mnt/ceph-compile/ceph/src/gmock'
  make[2]: *** [distclean-recursive] Error 1
  make[2]: Leaving directory `/mnt/ceph-compile/ceph/src'
  make[1]: *** [distclean-recursive] Error 1
  make[1]: Leaving directory `/mnt/ceph-compile/ceph'
  make: *** [clean] Error 2

  解决：
  1. ceph/src# git clone  https://github.com/ceph/gmock.git 【但是在进入编译的
  时候， 会git subtree update其他git代码源里的代码的，这个为什么在最开始的时候
  就需要，肯定有问题】
  2. ceph/src# cd gmock/
  3. autoreconf -fvi 【为了生成./configure】
  4. ./configure  【为了生成 Makefile】
  5. OK

* 制作ceph deb包的时候, hack包版本
直接修改debian/changelog: (在开头加上如下几行， 这个是debian做包时，需要查看的文件，本文件是对发布版本的记录， 我们hack的改变一下版本（10.2.6-2就是hack的，默认是10.2.6-1）)

```
ceph (10.2.6-2) stable; urgency=medium

  * New upstream release

 -- Ceph Release Team <ceph-maintainers@ceph.com>  Tue, 05 Apr 2017 17:29:39 +0000

ceph (10.2.6-1) stable; urgency=medium

  * New upstream release

 -- Ceph Release Team <ceph-maintainers@ceph.com>  Tue, 01 Apr 2017 13:29:39 +0000
```