#  pool xxx has too few pgs

* 问题：


```
root@node-3:~# ceph -s
    cluster 04bb3d14-acee-431e-88cd-7256bccd10a1
     health HEALTH_WARN
            pool images has too few pgs
     monmap e1: 1 mons at {node-3=10.10.10.2:6789/0}
            election epoch 1, quorum 0 node-3
     osdmap e63: 4 osds: 4 up, 4 in
      pgmap v299609: 1636 pgs, 13 pools, 82237 MB data, 11833 objects
            90099 MB used, 14804 GB / 14892 GB avail
                1636 active+clean
  client io 0 B/s rd, 3478 B/s wr, 2 op/s
root@node-3:~# ceph osd pool get images pg_num   
pg_num: 100
root@node-3:~# ceph df
GLOBAL:
    SIZE       AVAIL      RAW USED     %RAW USED 
    14892G     14804G       90099M          0.59 
POOLS:
    NAME                   ID     USED       %USED     MAX AVAIL     OBJECTS
    rbd                    0           0         0        14791G           0
    .rgw                   1         394         0        14791G           2
    images                 2      64278M      0.42        14791G        8049
    volumes                3       3016M      0.02        14791G         814
    backups                4           0         0        14791G           0
    compute                5      14942M      0.10        14791G        2915
    .rgw.root              6         848         0        14791G           3
    .rgw.control           7           0         0        14791G           8
    .rgw.gc                8           0         0        14791G          32
    .users.uid             9         778         0        14791G           4
    .users                 10         14         0        14791G           1
    .rgw.buckets.index     11          0         0        14791G           1
    .rgw.buckets           12       300k         0        14791G           4
root@node-3:~# ceph osd pool ls detail
pool 0 'rbd' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 1 flags hashpspool stripe_width 0
pool 1 '.rgw' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 128 pgp_num 128 last_change 2 flags hashpspool stripe_width 0
pool 2 'images' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 100 pgp_num 100 last_change 62 flags hashpspool stripe_width 0
        removed_snaps [1~3]
pool 3 'volumes' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 512 pgp_num 512 last_change 50 flags hashpspool stripe_width 0
        removed_snaps [1~3]
pool 4 'backups' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 128 pgp_num 128 last_change 24 flags hashpspool stripe_width 0
pool 5 'compute' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 42 flags hashpspool stripe_width 0
        removed_snaps [1~3]
pool 6 '.rgw.root' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 28 owner 18446744073709551615 flags hashpspool stripe_width 0
pool 7 '.rgw.control' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 30 owner 18446744073709551615 flags hashpspool stripe_width 0
pool 8 '.rgw.gc' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 32 owner 18446744073709551615 flags hashpspool stripe_width 0
pool 9 '.users.uid' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 34 owner 18446744073709551615 flags hashpspool stripe_width 0
pool 10 '.users' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 39 owner 18446744073709551615 flags hashpspool stripe_width 0
pool 11 '.rgw.buckets.index' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 51 owner 18446744073709551615 flags hashpspool stripe_width 0
pool 12 '.rgw.buckets' replicated size 1 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 53 owner 18446744073709551615 flags hashpspool stripe_width 0
```

* 分析解决：
	1. 源码分析：
	在源码中搜索has too few pgs就定位到了mon/PGMonitor.cc中的get_health函数，代码块如下。*<font set color="red">在master版本中没有搜到，后来切换到0.94.6中，搜索到了；master中是“ pool images has many more objects per pg than average (too few pgs?)”</font>*
    2. 源码流程：
	 1. 先从pg_map中取所有的pool的总的objects num，然后取所有pool的pg num, 然后计算average_objects_per_pg,这个是指整个集群中的所有objects和所有的pg的比率，也就是整个集群中每个pg有多少个objects；
     2. 取每个pool的objects num，然后取每个pool的pg num，然后计算objects_per_pg，这个值是每个pool中objects和pg的比率，也就是这个pool中每个pg有多少个objects.
     3. 计算ratio=objects_per_pg / average_objects_per_pg，这个比率是pool和集群平均值的比率。
     4. 取配置文件中的mon_pg_warn_max_object_skew这个参数的值，（默认为10）。
     5. 比较ratio和mon_pg_warn_max_object_skew这个参数值，然后就会出现了问题中的Health_warn警告。


* 源码：
```
if (!pg_map.pg_stat.empty()) 
  for (ceph::unordered_map<int,pool_stat_t>::const_iterator p = pg_map.pg_pool_sum.begin();
       p != pg_map.pg_pool_sum.end();
       ++p) {
    const pg_pool_t *pi = mon->osdmon()->osdmap.get_pg_pool(p->first);
    if (!pi)
      continue;   // in case osdmap changes haven't propagated to PGMap yet
    if (pi->get_pg_num() > pi->get_pgp_num()) {
      ostringstream ss;
      ss << "pool " << mon->osdmon()->osdmap.get_pool_name(p->first) << " pg_num "
         << pi->get_pg_num() << " > pgp_num " << pi->get_pgp_num();
      summary.push_back(make_pair(HEALTH_WARN, ss.str()));
      if (detail)
        detail->push_back(make_pair(HEALTH_WARN, ss.str()));
    }
    int average_objects_per_pg = pg_map.pg_sum.stats.sum.num_objects / pg_map.pg_stat.size();
    if (average_objects_per_pg > 0 &&
        pg_map.pg_sum.stats.sum.num_objects >= g_conf->mon_pg_warn_min_objects &&
        p->second.stats.sum.num_objects >= g_conf->mon_pg_warn_min_pool_objects) {
      int objects_per_pg = p->second.stats.sum.num_objects / pi->get_pg_num();
      float ratio = (float)objects_per_pg / (float)average_objects_per_pg;
      if (g_conf->mon_pg_warn_max_object_skew > 0 &&
          ratio > g_conf->mon_pg_warn_max_object_skew) {
        ostringstream ss;
        ss << "pool " << mon->osdmon()->osdmap.get_pool_name(p->first) << " has too few pgs";
        summary.push_back(make_pair(HEALTH_WARN, ss.str()));
        if (detail) {
          ostringstream ss;
          ss << "pool " << mon->osdmon()->osdmap.get_pool_name(p->first) << " objects per pg ("
             << objects_per_pg << ") is more than " << ratio << " times cluster average ("
             << average_objects_per_pg << ")";
          detail->push_back(make_pair(HEALTH_WARN, ss.str()));
        }
      }
    }
```

解决：
1. 增加相应pool的pg数

2. 删除多余的无用的pool，所以在ceph cluster中创建测试pool的时候，不要给太多的pg。

3. 修改mon_pg_warn_max_object_skew参数值，增加这个值，提高阈值。
