1. ceph两个网络怎么规划:
public 和 cluster, public network是client和monitor, monitor和osd通信的网络,
cluster网络是osd与osd之间通信的网络.

2. client和osd之间怎么通信? 是通过public呢?还是cluster呢?
public, 这样的话, 岂不是public也得万兆? 因为client连接osd的话, 也要进行数据传输
的, 这样的话, 也很容易达到网络瓶颈????

3. public和cluster可以用同一个4万兆网络吗?
首先, cluster是可以不用连外网或者大网的,就是一个私有网络;
另外, cluster网络其实也起到一个安全的作用, 可以防止DOS攻击之类的.
