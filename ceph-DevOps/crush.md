## ceph crush 基本操作

### 方法一:  命令行修改

```
root@ceph-client:~# ceph osd crush add-bucket test-bucket root
added bucket test-bucket type root to crush map
 
root@ceph-client:~# ceph osd tree
ID WEIGHT  TYPE NAME          UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-7       0 root test-bucket                                     
-1 4.00000 root default                                         
-2       0     host ceph-osd1                                   
-3       0     host ceph-osd2                                   
-4 1.00000     host ceph-mon2                                   
 2 1.00000         osd.2           up  1.00000          1.00000 
-5 2.00000     host ceph-mon1                                   
 3 1.00000         osd.3           up  1.00000          1.00000 
 1 1.00000         osd.1           up  1.00000          1.00000 
-6 1.00000     host ceph-mon                                    
 0 1.00000         osd.0           up  1.00000          1.00000 

root@ceph-client:~# ceph osd crush move ceph-osd1 root=test-bucket
moved item id -2 name 'ceph-osd1' to location {root=test-bucket} in crush map

root@ceph-client:~# ceph osd tree
ID WEIGHT  TYPE NAME          UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-7       0 root test-bucket                                     
-2       0     host ceph-osd1                                   
-1 4.00000 root default                                         
-3       0     host ceph-osd2                                   
-4 1.00000     host ceph-mon2                                   
 2 1.00000         osd.2           up  1.00000          1.00000 
-5 2.00000     host ceph-mon1                                   
 3 1.00000         osd.3           up  1.00000          1.00000 
 1 1.00000         osd.1           up  1.00000          1.00000 
-6 1.00000     host ceph-mon                                    
 0 1.00000         osd.0           up  1.00000          1.00000 

【如果新建了rule，并且将rule绑定了这个bucket下，需要先将rule删除；然后再删除这个bucket】
root@ceph-client:~# ceph osd crush remove test-bucket
removed item id -7 name 'test-bucket' from crush map

root@ceph-client:~# ceph osd tree
ID WEIGHT  TYPE NAME          UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 4.00000 root default                                         
-2       0     host ceph-osd1                                   
-3       0     host ceph-osd2                                   
-4 1.00000     host ceph-mon2                                   
 2 1.00000         osd.2           up  1.00000          1.00000 
-5 2.00000     host ceph-mon1                                   
 3 1.00000         osd.3           up  1.00000          1.00000 
 1 1.00000         osd.1           up  1.00000          1.00000 
-6 1.00000     host ceph-mon                                    
 0 1.00000         osd.0           up  1.00000          1.00000 
```

### 方法二: 修改crush map文件

ceph官网crush map操作说明：

**1. GET A CRUSH MAP**

To get the CRUSH map for your cluster, execute the following:
```
ceph osd getcrushmap -o {compiled-crushmap-filename}
```
  Ceph will output (-o) a compiled CRUSH map to the filename you specified. Since the CRUSH map is in a compiled form, you must decompile it first before you can edit it.
 
**2. DECOMPILE A CRUSH MAP**

To decompile a CRUSH map, execute the following:
```
crushtool -d {compiled-crushmap-filename} -o {decompiled-crushmap-filename}
```
Ceph will decompile (-d) the compiled CRUSH map and output (-o) it to the filename you specified.

**3. Edit crush map**

**4. COMPILE A CRUSH MAP**

To compile a CRUSH map, execute the following:
```
crushtool -c {decompiled-crush-map-filename} -o {compiled-crush-map-filename}
```
Ceph will store a compiled CRUSH map to the filename you specified.

**5. SET A CRUSH MAP**

To set the CRUSH map for your cluster, execute the following:
```
ceph osd setcrushmap -i  {compiled-crushmap-filename}
```
Ceph will input the compiled CRUSH map of the filename you specified as the CRUSH map for the cluster.

**具体用例：**

```
root@ceph-client:~# ceph osd getcrushmap -o crushmap.map
got crush map from osdmap epoch 114

root@ceph-client:~# crushtool -d crushmap.map -o crushmap.txt

root@ceph-client:~# vim crushmap.txt 
//编辑crushmap文件，修改成期望的crush结构

root@ceph-client:~# crushtool -c crushmap.txt -o crushmap.new

root@ceph-client:~# ceph osd setcrushmap -i crushmap.new 
set crush map
```

**crushmap文件修改:** 

* 增加一个bucket：
在bucket最后直接添加：（复制之前的一个bucket，然后修改bucket name、id（id继续增加就行（注意是负的））、把它下面的item去掉，目前还没有item）
```
root test-bucket {
        id -7           # do not change unnecessarily
        # weight 0.000
        alg straw
        hash 0  # rjenkins1
}
```

* 移动一个host bucket到root节点上：
直接将下面的两个节点copy到root test-bucket之前；再test-bucket里面加上
>         item ceph-osd1 weight 0.000
>         item ceph-osd2 weight 0.000

当然weight要按照host本身的weight；这个时候一定注意原来所在的root节点中的那些host删除。
【这些不用担心，在编译crushmap的时候，会报错的。】
```
host ceph-osd1 {
        id -2           # do not change unnecessarily
        # weight 0.000
        alg straw
        hash 0  # rjenkins1
}
host ceph-osd2 {
        id -3           # do not change unnecessarily
        # weight 0.000
        alg straw
        hash 0  # rjenkins1
}
```

* 删除一个bucket：
直接删除该bucket的字段，注意同时去掉关联的东西。

**crushmap.txt 内容：**
```
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable straw_calc_version 1

# devices
device 0 osd.0
device 1 osd.1
device 2 osd.2
device 3 osd.3

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host ceph-mon2 {
        id -4           # do not change unnecessarily
        # weight 1.000
        alg straw
        hash 0  # rjenkins1
        item osd.2 weight 1.000
}
host ceph-mon1 {
        id -5           # do not change unnecessarily
        # weight 2.000
        alg straw
        hash 0  # rjenkins1
        item osd.3 weight 1.000
        item osd.1 weight 1.000
}
host ceph-mon {
        id -6           # do not change unnecessarily
        # weight 1.000
        alg straw
        hash 0  # rjenkins1
        item osd.0 weight 1.000
}
root default {
        id -1           # do not change unnecessarily
        # weight 4.000
        alg straw
        hash 0  # rjenkins1
        item ceph-mon2 weight 1.000
        item ceph-mon1 weight 2.000
        item ceph-mon weight 1.000
}
host ceph-osd1 {
        id -2           # do not change unnecessarily
        # weight 0.000
        alg straw
        hash 0  # rjenkins1
}
host ceph-osd2 {
        id -3           # do not change unnecessarily
        # weight 0.000
        alg straw
        hash 0  # rjenkins1
}
root test-bucket {
        id -7           # do not change unnecessarily
        # weight 0.000
        alg straw
        hash 0  # rjenkins1
        item ceph-osd1 weight 0.000
        item ceph-osd2 weight 0.000
}

# rules
rule replicated_ruleset {
        ruleset 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}

# end crush map 
<<<<<<< HEAD
```
=======
```
>>>>>>> 860956eafcbca9c24320cd989d5bcd795453de15
