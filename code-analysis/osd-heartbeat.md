osd heartbeat流程分析:
从ceph_osd.cc开始, 也就是ceph-osd服务启动开始:
ms_hbclient/ms_hb_back_server/ms_hb_front_server [Messenger]先定义了3个Messenger对象, 其中, hb client是osd发送ping心跳的messenger, hb_back_server是集群网络服务,接收ping心跳, hb_front_server是公开网络接收ping.
|
Messenger::create() [Messenger]
|
SimpleMessenger() [SimpleMessenger] 现在默认的消息系统是SimpleMessenger
|
init_local_connection() [SimpleMessenger] 设置了peer_addr为自己的ip, 设置了peer_type, 在msg/msg_types.h中定义, type包括:mon/osd/mds/client
|
ms_deliver_handle_fast_connect() [Messenger] 通知fast dispatcher有新的connection, 这个函数在一个新的链接初始化或者重新链接的时候调用; 会循环fast_dispatcher list, 然后分别通知, 10.2.5中看到只有osd, objecter, heartbeat实现了这个函数; fast_dispatcher这个list会在add_dispatcher_tail和add_dispatcher_head中更新, 更新方法是调用dispatcher(所有继承Dispatcher的类)自己的ms_can_fast_dispatch_any()函数, 这个函数由dispatcher自己实现, 就是返回true或false; 所以要想某个msg需要fast_dispatch, 需要实现ms_can_fast_dispatch_any()及ms_fast_dispatch(); 但不是所有的msg都可以fast, 要满足3个要求: 1. 能够快速处理消息, 不会长时间拿某个lock; 后面两点还没弄明白 [后续分析, msg/Dispatcher.h 45行左右 ???] 
|
ms_handle_fast_connect() [Dispatcher] 虚函数
|
ms_handle_fast_connect() [OSD] 具体实现
|
new Session() 并设置了session的一些属性 这块应该还有一些其他动作, 但是没有找到, 我觉得应该建立一个connection [????]

\
定义了几个hb messenger之后, 会建立网络链接:
ms_hb_back_server/ms_hb_front_server->bind() [SimpleMessenger] 绑定ip port, 进而走accepter那一套, 我们在这里不再分析
\
osd = new OSD() [OSD] new OSD对象
\
ms_hbclient/ms_hb_back_server/ms_hb_front_server->start()  SimpleMessenger的start(), 也没干啥, 就是起了一个回收pipe的线程, 是一直在回收吗???
\
osd->init() [OSD] 这个是OSD服务真正初始化的地方, 会做很多处理, 我们在这里先只关注heartbeat的几个初始化
|
heartbeat_thread.create() [T_Heartbeat] 该类继承的Thread类, 也就是创建了一个单独的heartbeat线程, 该线程的入口函数是heartbeat_entry()
|
heartbeat_entry() [OSD] 一直轮训的发送heartbeat, 时间间隔是wait = .5 + ((float)(rand() % 10)/10.0) * (float)cct->_conf->osd_heartbeat_interval [这个算法有什么含义吗? 没搞懂???]
|
heartbeat() [OSD] 
| | | |
| | | getloadavg() 先获取一下当前系统CPU 1m load, system CPU load有1m, 5m, 15m
| | con_back/con_front->send_message() [SimpleMessenger] for循环, 分别向heartbeat_peers中的元素发送MOSDPING::PING message, 就是走messenger的send_message发送到目标osd上.
| heartbeat_check() [OSD] 检查peer osd的heartbeat info
| |
| is_unhealthy() [HeatbeatInfo] 检查ping reply是否超时, 超时: False, 否则: True
| |如果超时, 或者没有收到ping reply
| failure_queue会更新, 也就是将该osd都加入failure queue.  
heartbeat_peers.empty() 还会检查heartbeat_peers是否为空, 如果为空, 并且, 和monitor的心跳超过了配置的interval, 但是自己还是active的, 就会去更新一下osdmap


下面分析osd对于ping心跳的消息分发及处理:
osd->init() [OSD]
|
hbclient_messenger->add_dispatcher_head(&heartbeat_dispatcher) heartbeat采用单独的heartbeat dispatcher, 并不是和osd用同一个dispatcher
|
ms_dispatch() [HeartbeatDispatcher]
|
osd->heartbeat_dispatch() [OSD] 进入MSG_OSD_PING分支
|
handle_osd_ping() [OSD]
| |
| is_healthy() [HeartbeatMap] 检查本osd内部状态, 不正常的时候, 直接丢弃心跳消息, 处理心跳已经没有意义了; 这个函数后续再研究一下, 在common/HeartbeatMap.cc中
send_message() 发送MOSDPing::PING_REPLY的消息, 但是并没有结束, 还会查看当前osdmap, 如果发现osdmap中已经标记收到消息的那个osd为down了, 要去告诉那个osd, 也就是会再发一个MOSDPing::YOU_DIED的message

下面分析收到ping reply消息的处理:
更新peer osd的那些last_rx_front/back到reply的时间戳, 然后检查健康状况:
is_healthy() [HeartbeatInfo], 因为要看收到reply的时间是否超时, 如果健康, 并且之前已经将这个osd加入了failure队列或者正在等待加入failure队列, 会去擦除这个osd

下面分析收到MOSDPing::YOU_DIED的消息处理:
osdmap_subscribe() 自己去订阅osdmap, 我理解的就是更新一下osdmap, 告诉monitor我活了


下面来分析osd将timeout的peer osd加入failure queue后的处理:
(通过查找failure_queue的调用来追踪)

osd->init() [OSD]
|
tick_timer_without_osd_lock.add_event_after(new C_Tick_WithoutOSDLock) 启动timer, 回调函数是C_Tick_WithoutOSDLock类, 该类中实现finish()函数, 
|
finish() [C_Tick_WithoutOSDLock]
|
tick_without_osd_lock() [OSD] 这个函数中会在最后又起一个timer, 和上面的timer一模一样, timer就这样有了; 
|
send_failures() [OSD] 除了上面那个函数有调用send_failures(), 还有ms_handle_connect()也调用了, 后续再分析这个分支 [????]
|
monc->send_mon_message() [MonClient] 从failure_queue中遍历fail osd, 向monitor发送osd id失败的消息, 然后将osd id从failure queue删除

下面来分析peer osd list的更新:
heartbeat_peers是一个osd id和heartbeat info的map关系, 该map会更新的情况: tick()/_committed_osd_maps()/handle_pg_create(), 这几函数都会调用maybe_update_heartbeat_peers(), 进而来更新heatbeat_peers

tick() / _committed_osd_maps() / handle_pg_create() 什么时候需要更新peers集合，也即这个函数什么时候会被调用？从实现看，影响peers集合主要是pgmap的变化，那什么时候pgmap可能改变呢？ 1. tick线程中周期性的检查，主要是因为osd启动过程中，会load_pg 2. osdmap变更的时候，osd承载的pg可能需要重新peering，导致osd状态可能会变为STATE_WAITING_FOR_HEALTHY 3. pg创建的时候，参考函数handle_pg_create
|
maybe_update_heartbeat_peers() [OSD]
|
heartbeat_peers_need_update() [OSD] 先判断一下是否需要更新, 这个是否需要更新, 是看OSD是否处于STATE_WAITING_FOR_HEALTHY状态, 是的话, 如果last_heartbeat_resample(utime_t类型的)是0, 就设置为True(需要更新); 还有一种情况是: last_heartbeat_resample这个时间在osd_heartbeat_grace(默认值20s)范围外, 就去强制更新; 注意前提都是osd处于waiting_for_healthy状态; 什么时候会处于这种状态呢? 一般有: a. 在osd启动的过程中; b.在osd收到更新osdmap的消息，osd状态可能变为waiting
接下来有几个不同维度的更新osd peers, 我们一个一个分析:

1. 遍历pg, 然后遍历pg的heartbeat peers, 然后从osdmap中检查该osd是否up, 如果up, 就加入本osd的heartbeat peers
2. 遍历pg, 遍历pg的probe targets peers, 然后检查osd是否up, 然后加入peers
3. 定义了两个集合, 一个是want, 一个是extras, 然后更新want和extras, want集合中都是什么呢? want中放的是osd的相邻osd id, 那什么算相邻osd呢? 有两个函数: get_next_up_osd_after(whoami),get_previous_up_osd_before(whoami), 分别是比当前osd id大的处于up的第一个osd(for循环, 查osd id, 从osd_id + 1开始找; 并且, osd id的大小判断标准是一个循环的, 最大的osd id的后一个是osd id最小的那个); 比osd id小的第一个up osd, want集合中的osd都是extras(额外的), 然后将额外的加入heartbeat peers中.

   不是简单的加入heartbeat peers就结束了, 还要做一轮检查:
1. 遍历heartbeat peers, 如果peer osd down了, 就从peers中剔出.
2. 如果heartbeat peers的总数小于osd_heartbeat_min_peers(配置文件中的, 默认值是10), 就继续将比当前osd id大的up osd加入extras集合中, 并且加入heartbeat peers
3. 但是如果heartbeat peers的总数大于osd_heartbeat_min_peers(还是那个参数, 也就是没有max, 这个min硬性指标), 会从heartbeat peers中剔除那些, 是extras集合中的, 但不是want集合(相邻的)的osd
4. 当然, 如果osd一共就比osd_heartbeat_min_peers少, 那也是没办法了.


综上分析:
1. osd heartbeat是单独的线程,单独的dispatcher来分发, 处理的.
2. osd的heartbeat是一直保持的, 时间间隔是osd_heartbeat_interval(默认是6s)
3. heartbeat peers是分别从pg heartbeat peers, 相邻osd, 及其他额外相邻的osd中更新的, 但都是up的.
4. heartbeat发送的是Ping message, peers收到后需要回复ping reply
5. heartbeat peers更新的时机包括: osd处于waiting_for_healthy的状态时, 也就是osd启动时, 或osdmap有更新时; 新创建pg时;  

调优:
1. 大规模部署情况下，压测的时候（比如1000 vm 跑fio)，心跳可能会出问题，可能需要将grace时间调大，避免误报， 如果调大后，interval也应相应增大，避免发送频率太高，保证至少发送过3次心跳后没有回包才上报，比如目前grace为20， interval为6，如果grace为30，则interval建议为9比较合适.
2. grace调大，也有副作用，如果某个osd异常退出，等待其他osd上报的时间必须为grace，在这段时间段内，这个osd负责的pg的io会hang住。 可以采用我之前的优化的patch，尽量不要将grace调的太大。
3. 如果集群存在大规模的顺序读写，网络成为瓶颈的时候，可以通过下面这个参数调高心跳消息在内核网络层的优先级: osd_heartbeat_use_min_delay_socket 

