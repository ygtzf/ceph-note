# 在ubuntu上升级kernel

## 内核升级说明
* 使用apt-get来直接安装kernel的几个包。（这种方式最简单，但是ubuntu的官方源里一般不会有最新的kernel，都是比较老的kernel版本。）
* 从官方下载kernel所需的几个deb包， 自行安装。（这种方式也很简单，版本可控）
* 下载kernel源码，自行编译。（这个难度最大， 如果不是进行kernel开发，没有这个必要）

## Step 1. 了解内核版本信息

Linux kernel 版本信息，在linux kernel的官网上有明确的详细说明: https://www.kernel.org/

其中， 可以看到最新的版本，有稳定版（stable），有长期维护版（longterm）。我们一般会选择最新的longterm版本的kernel。

## Step 2. 下载kernel deb包
ubutnu 有自己的kernel包的仓库： http://kernel.ubuntu.com/~kernel-ppa/mainline/

所有的kernel包都在这里，我们需要进入之前选定的包的目录，这里有一堆deb包。

升级kernel, 我们需要3个包:

* linux-headers-{kernel version}_{kernel version}.{packge generate time}_all.deb
* linux-image-{kernel version}-generic_{kernel version}.{packge generate time}_{system architecture}.deb
* linux-headers-{kernel version}-generic_{kernel version}.{packge generate time}_{system architecture}.deb

其中，

kernel version就是第一步选择的kernel版本;

package generate time就是deb包生成时间，这个不用关心；

system architecture 是指你的操作系统架构，也就是多少位的系统，32位还是64位， 32位就选i386，64位就选amd64.

**举例**:

要升级4.4.52版本的内核， 就要下载如下包：

> linux-headers-4.4.52-040452_4.4.52-040452.201702260631_all.deb
>
> linux-headers-4.4.52-040452-generic_4.4.52-040452.201702260631_amd64.deb
>
> linux-image-4.4.52-040452-generic_4.4.52-040452.201702260631_amd64.deb

**注**:

包名中带lowlatency的是低延迟应用需要下载的包，比如：主要用来做视频处理的系统等。

## Step 3. 安装kernel包:

>sudo dpkg -i *.deb

## Step 4. 重启系统

> sudo reboot

## Step 5. 验证内核是否升级到指定版本:

> uname -r
>
> 4.4.52-30-generic
