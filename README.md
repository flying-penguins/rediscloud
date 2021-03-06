## 文档内容提要
 1. 系统设计及工作流程
 2. HA 机制设计与实现
 3. 在线扩容设计与实现
 4. 其他相关设计与实现
 5. 系统配置文件说明
 6. 服务编译运行与部署
 7. Redis命令运行示例
 8. Telnet控制台命令说明
 9. Redis主备切换示例-主动
 10. Redis主备切换示例-被动
 11. 在线平滑扩容升级示例
 12. 系统监控与状态统计

RedisProxy 是一套基于GO语言实现的Redis分布式解决方案, 对于使用方来说, 使用RedisProxy 与使用原生的 RedisServer 没有区别 (有些Redis原生命令在RedisProxy 作为命令黑名单不予支持), 上层应用可以像使用单机的 Redis 一样使用多个Redis服务器；RedisProxy会处理请求的转发, 在线扩容、缩容, 以及Redis服务器的主备故障切换等工作, 对于客户端来说是完全透明的。设计原则是尽量精简、高效简洁；便于部署运维。
## 系统设计及工作流程
RedisProxy 作为一套分布式系统，由一个配置节点、多个访问代理节点、以及一系列redis主备服务器组成，配置节点我们称其为config-server，访问代理节点为proxy-server，一套redis主备服务器为一个redis-group。
![RedisProxy系统框架图](https://github.com/flying-penguins/redisproxy/raw/master/doc/20161114145802859.png)

 - config-server： 负责管理所有redis-server，维护proxy-server的路由分发规则，以及定时收集proxy-server的状态、统计信息等。从架构上看，config-server类似于传统系统的中心节点，但实际上对于RedisProxy 这套系统来说是非常轻量级的，客户端的请求是完全不经过config-server的；就算config-server挂掉在路由规则无变化的情况下不影响系统提供正常服务，换句话说在config-server挂掉的同时redis-server也有挂掉，才会影响部分请求。我们还提供主备模式的部署，主备之间通过keepalive探测对方的状态，进行状态切换【部署主备个人感觉没多大必要，尽量在硬件上面做好隔离比较好】。
 - proxy-server： 实现了redis协议提供客户端的访问功能；每个客户端连接上来会开一个协程专门处理，获取请求校验命令合法后，根据KEY路由到相应的后端redis服务器进行执行；proxy-server跟后端redis的一个数据库一条TCP连接，通过pipeline模式与redis交互。
 - redis-group： redis主备服务器组，一个组内包含一主多备作容灾备份；为了确保数据的一致性读写请求都是发往主服务器执行。
 - 路由规则： 根据操作的KEY值HASH到1024个slot，每个redis-group会负责一系列slot的存储，每个slot对应到redis中的一个数据库，slot的序号就是数据库的索引号；逻辑上分成1024个slot的目的是为了便于扩容等方面的管理。

proxy-server在启动的时候会通过config-server获取最新的路由规则；客户端的请求直接发往proxy-server，proxy-server根据路由规则分发到后端redis服务器，并将响应转发给客户端。config-server提供管理功能(扩容、数据迁移、redis主备切换等)，路由规则变更通知等。
##  HA 机制设计与实现
 - proxy-server访问层的容灾：proxy-server是无状态的，随意增删完全水平扩展；可以通过客户端失败重试或者前面架设LVS进行容灾，
 -  redis-group存储层容灾： 对于可靠性要求较高的服务，要配置redis的主从模式；config-server每5s探测一下redis-group中master的状态，如果在指定时间内一直探测失败，会选择一个数据相对全的slave提升为主；如果对数据的一致性要求极高，可以将探测失败的时间设置为很长，这样不会自动进行主从切换，可以通知管理人员上来处理。需要注意，config-server 将其中一个 slave 升级为 master 时，该组内其他 slave 实例是不会自动改变同步节点；因此当出现主从切换时，需要管理员手动创建新的主从关系，不提供自动的原因是风险太高。
## 在线扩容设计与实现
扩容的实现方式就是将一系列slot从旧的redis-group移动到一组新的上面（缩容也是类似迁移slot即可）。

 - config-server发起扩容操作给proxy-server，config-server根据proxy-server的应答决定是否继续进行。
 - config-server遍历待迁移的slot对应的redis数据库，采用migrate命令原子的迁移到新的redis上面。
 - 迁移完成后，发送新的路由规则给proxy-server。

发生数据迁移的时候proxy-server如何对外提供服务：
config-server在数据迁移的过程中，proxy-server的路由规则并没有变化，两者是并行进行的；如果还是按照原先逻辑会出现行为不一致，从而导致数据不一致。
我们采取的方式是：proxy-server接受到的请求，如果正好落在迁移的slot上面，proxy-server首先发起migrate操作优先迁移过去，然后再到新的redis上面执行请求。

注意：migrate操作的时候采取忽略“NO KEY”响应的情况，以防止proxy-server与config-server操作同一个KEY(migrate同一个key只会有一个成功)；同时采取替换目标实例已存在KEY的方式。
##  其他相关设计与实现
 - 支持pipeline模式，大小是1024(命令个数)；每个客户端连接上来会分配一个1024的缓冲区。
 - 支持LUA脚本EVAL、EVALSHA命令；只会根据第一个KEY路由到一台后端redis（这点要注意）。
 - Redis命令流量控制；每个slot限制等待redis回复响应的大小是4096(proxy-server与redis-server之间的pipeline大小是4096)，超过就等待。
 - 完全水平扩展，所以不存在单机性能问题，当然最大支持后端1024个redis限制，proxy之间无通信交互、无状态的，路由信息通过config-server获取。


## 系统配置文件说明
配置文件采用yaml格式，yaml是一种非常简洁、功能强大的配置格式，绝对胜过ini、xml、json等格式；yaml解析库是系统唯一依赖的一个第三方库。

 - config-server、proxy-server配置说明
```
# 日志级别，从0——5分别是：panic、error、warn、notic、info、debug
log_level: 4
# 日志位置，日志每小时分割一次，并进行压缩，保留最近三天
log_file: ./logs/configserver.log
cluster:
  # 服务bind的地址
  addr: 192.168.200.129:19601
  # 如果配置是给config-server使用，则为对端config-server地址（config-server主备部署时使用，否则配置为空）
  # 如果配置是给proxy-server使用，则为config-server的地址，config-server为主备时配置两个
  cfg_servers: [""]
  # config-server之间keepalive时间，服务内部是30s探测一次，超时对端无响应提升自己为主
  # proxy-server可以不配置
  keepalive_time: 600
  # proxy-server允许客户端的最大连接数
  # 对于config-server不起作用，可不配置
  max_clients: 200
  # config-server使用，redis主备切换时间间隔，服务内部是5s探测一次
  # proxy-server可不配置
  redis_down_time: 60
  # 配置变更的版本号，对于config-server非"0"即可；如果需要手动修改配置文件发布，随意修改一下值就可以
  # 对于proxy-server不需要关心，不配置也可
  groups_ver: "135"
  # 根据slot的路由规则，只需要配置主redis服务器即可，从服务器信息通过主获取到，migrate无需配置
  # 只需要在config-server中配置；proxy-server无需配置
  groups:
  - slots: [0, 340]
    master: 192.168.200.129:9001
    migrate: ""
  - slots: [341, 680]
    master: 192.168.200.129:9002
    migrate: ""
  - slots: [681, 1023]
    master: 192.168.200.129:9003
    migrate: ""
```
 - redis-server配置说明：
```
databases 1024 #数据库设置为1024或者以上
采用3.0以上版本，最好采用最新的3.2
不要设置密码，不然数据迁移的时候migrate失败
```
## 服务编译运行与部署
 - 编译运行
 编译redisproxy /servers/config-serve得到config-server
 编译redisproxy /servers/proxy-server得到proxy-server
 通过命令行参数-c  指定配置文件运行程序
 - 部署方式：proxy-server是无状态的可以随意启动多个；config-server采用主备的方式，也可以只启动一个作为主(推荐这个方式，一主一备必要性不大)，尽量在机器上面隔离就好。
 - config-server配置例子
```
log_level: 4
log_file: ./logs/configserver.log
cluster:
  addr: 0.0.0.0:19601
  cfg_servers: [""]
  keepalive_time: 600
  redis_down_time: 60
  groups_ver: "1111"
  groups:
  - slots: [0, 340]
    master: 0.0.0.0:9001
    migrate: ""
  - slots: [341, 680]
    master: 0.0.0.0:9002
    migrate: ""
  - slots: [681, 1023]
    master: 0.0.0.0:9003
    migrate: ""
```
 - proxy-server配置例子

```
log_level: 4
log_file: ./logs/proxyserver.log
cluster:
  addr: 0.0.0.0:9601
  cfg_servers: ["127.0.0.1:19601"]
  max_clients: 20480
```
注意：在本地运行的时候修改最大文件描述符，通常限制是1024，需要修改大一点可以为10240
## Redis命令运行示例
使用redis-cli连上proxy
```
127.0.0.1:9601> set k1 v1
OK
127.0.0.1:9601> get k1
"v1"
127.0.0.1:9601> set k2 v2
OK
127.0.0.1:9601> mset k3 v3 k4 v4
OK
127.0.0.1:9601> mget k1 k2 k3 k4
1) "v1"
2) "v2"
3) "v3"
4) "v4"
127.0.0.1:9601> del k1 k2
(integer) 2
127.0.0.1:9601> mget k1 k2 k3 k4
1) (nil)
2) (nil)
3) "v3"
4) "v4"
127.0.0.1:9601> keys *
Error: Server closed the connection
127.0.0.1:9601> 
```
特殊命令说明：
 1. mget、mset、del操作多个key的时候是分别分发到多个后端redis执行，不是一个原子操作，如果有些成功有些失败，会响应失败。
 2. 像keys还有一些管理命令等作为黑名单，不允许执行。
 3. select、quit、auth命令proxy-server直接响应OK
 4. ping命令proxy-server直接响应PONG
## Telnet控制台命令说明
可以通过telnet进入config-server，通过发送命令给config-server控制系统的行为。telnet以后通过输入冒号跟英文字母cmd与config-server握手(:cmd)；握手成功后会进入命令模式运行。如下：
```
$ telnet 0 19601
Trying 0.0.0.0...
Connected to 0.
Escape character is '^]'.
:cmd

CMD> 
2016-11-14 04:36:16.178191452 -0800 PST
=======================================================================
exit                   --- exit telnet cmd mode or fresh screen
group                  --- view groups info: slotid keys;proxy(qps hand-time succ-rate); rediss...
proxy                  --- view proxys info: addr conns qps hand-time succ-rate
keyslot key            --- get slotid by key: slotid
migrate 000 555 master --- process
promote addr1 as addr2 --- process
-----------------------------------------------------------------------
2016-11-14 04:36:16.178253374 -0800 PST
```
 1. exit：退出telnet命令控制模式或者group、proxy等命令实时信息刷新
 2. group：查看group相关的信息；按照group维度实时查看请求量、处理耗时、成功率、redis服务器相关信息
 3. proxy：查看proxy相关信息；按照proxy维度实时查看请求量、处理耗时、成功率等信息
 4. keyslot：查看key对应的slot-id
 5. migrate：slot迁移命令，用于扩容、缩容
 6. promote：主备切换，提升一个备为主(主动主备切换使用)
## Redis主备切换示例-主动
主动状态的主从切换：是人工发起，比如机器上下线进行替换等情况下面使用。这种情况下面是无损的，无缝切换不影响客户端正常请求(会有3s内的卡顿)。通过命令promote new-redis as old-redis进行，new-redis将升级为master，old-redis下线；前提是它们是主从关系，并且处于数据基本一致的状态，命令运行的时候会进行校验。

 -  通过group命令看到在[0000-0340] 中master是：0.0.0.0:9001，slave是：192.168.200.129:9004（3319这个数字是同步的offset）；如下
```
CMD> group
2016-11-11 08:13:46.779861645 -0800 PST
2016-11-11 08:14:01.795930415 -0800 PST
[0000-0340] KEYS:2;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9001,3319[192.168.200.129:9004,3319 ]
[0341-0680] KEYS:1;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9002,0[]
[0681-1023] KEYS:0;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9003,0[]
```
 - 发送promote命令，以及之后的结果如下：
```
CMD> promote 192.168.200.129:9004 as 0.0.0.0:9001


2016-11-11 08:14:27.968722572 -0800 PST
update to:  0.0.0.0:9601, ver:1112
count vote: 0.0.0.0:9601, ver:1112,accept
0.0.0.0:9001->192.168.200.129:9004: wait for sync, success!
192.168.200.129:9004: slaveof no one, success!
commit to:  0.0.0.0:9601, ver:1112(done) [total:1,accept:1,reject:0]
Promote Success! 
2016-11-11 08:14:28.117824741 -0800 PST


CMD> group
2016-11-11 08:15:04.269827604 -0800 PST
2016-11-11 08:15:08.273852676 -0800 PST
[0000-0340] KEYS:2;0.0.0.0:9601(Q:0 T:0us R:100.00);192.168.200.129:9004,3361[]
[0341-0680] KEYS:1;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9002,0[]
[0681-1023] KEYS:0;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9003,0[]
```
可以看到
192.168.200.129:9004已经升级为主，0.0.0.0:9001已经下线。
中间的输出文本是切换的流程信息。
## Redis主备切换示例-被动
被动状态的主从切换：redis服务器故障引起的切换，config-server检测到master挂掉后会提升slave为新的master。检测失败的时长在config-server的配置文件中进行配置。
如下：127.0.0.1:9004作为0.0.0.0:9001的从节点存在，将9001对应的进程kill掉，大约过一分钟会看到master已经变为127.0.0.1:9004

```
CMD> group
2016-11-13 23:52:16.675478818 -0800 PST
[0000-0340] KEYS:-341;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9001,9130[127.0.0.1:9004,9130 ]
[0341-0680] KEYS:1;0.0.0.0:9601(Q:0 T:0us R:100.00;0.0.0.0:9002,0[]
[0681-1023] KEYS:0;0.0.0.0:9601(Q:0 T:0us R:100.00;0.0.0.0:9003,0[]

CMD> group
2016-11-13 23:52:52.912122163 -0800 PST
[0000-0340] KEYS:2;0.0.0.0:9601(Q:0 T:0us R:100.00);127.0.0.1:9004,0[]
[0341-0680] KEYS:1;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9002,0[]
[0681-1023] KEYS:0;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9003,0[]
```
## 在线平滑扩容升级示例
扩容的实现方式就是将一系列slot移动到新的redis服务器，数据迁移过程中不影响系统向外提供正常服务。
如下：将[1001-1023]迁移到新服务0.0.0.0:9004扩容的命令（migrate 1001 1023 0.0.0.0:9004），中间信息是数据迁移的过程。
```
CMD> group
2016-11-14 00:05:50.404632453 -0800 PST
[0000-0340] KEYS:2;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9001,0[]
[0341-0680] KEYS:1;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9002,0[]
[0681-1023] KEYS:0;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9003,0[]
2016-11-14 00:05:51.001707723 -0800 PST

CMD> migrate 1001 1023 0.0.0.0:9004
2016-11-14 00:06:21.608087113 -0800 PST
update to:  0.0.0.0:9601, ver:1112
count vote: 0.0.0.0:9601, ver:1112,accept
commit to:  0.0.0.0:9601, ver:1112(done) [total:1,accept:1,reject:0]
Setslot Success! Migrate...
migrate slot-1001: [####################################################################################################] 0/0 = 100.00
migrate slot-1002: [####################################################################################################] 0/0 = 100.00
migrate slot-1003: [####################################################################################################] 0/0 = 100.00
migrate slot-1004: [####################################################################################################] 0/0 = 100.00
migrate slot-1005: [####################################################################################################] 0/0 = 100.00
migrate slot-1006: [####################################################################################################] 0/0 = 100.00
migrate slot-1007: [####################################################################################################] 0/0 = 100.00
migrate slot-1008: [####################################################################################################] 0/0 = 100.00
migrate slot-1009: [####################################################################################################] 0/0 = 100.00
migrate slot-1010: [####################################################################################################] 0/0 = 100.00
migrate slot-1011: [####################################################################################################] 0/0 = 100.00
migrate slot-1012: [####################################################################################################] 0/0 = 100.00
migrate slot-1013: [####################################################################################################] 0/0 = 100.00
migrate slot-1014: [####################################################################################################] 0/0 = 100.00
migrate slot-1015: [####################################################################################################] 0/0 = 100.00
migrate slot-1016: [####################################################################################################] 0/0 = 100.00
migrate slot-1017: [####################################################################################################] 0/0 = 100.00
migrate slot-1018: [####################################################################################################] 0/0 = 100.00
migrate slot-1019: [####################################################################################################] 0/0 = 100.00
migrate slot-1020: [####################################################################################################] 0/0 = 100.00
migrate slot-1021: [####################################################################################################] 0/0 = 100.00
migrate slot-1022: [####################################################################################################] 0/0 = 100.00
migrate slot-1023: [####################################################################################################] 0/0 = 100.00
update to:  0.0.0.0:9601, ver:1113
count vote: 0.0.0.0:9601, ver:1113,accept
commit to:  0.0.0.0:9601, ver:1113(done) [total:1,accept:1,reject:0]
Migrate Over, Success
2016-11-14 00:06:21.893774913 -0800 PST

CMD> group
2016-11-14 00:06:32.567374261 -0800 PST
2016-11-14 00:06:36.571325007 -0800 PST
[0000-0340] KEYS:2;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9001,0[]
[0341-0680] KEYS:1;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9002,0[]
[0681-1000] KEYS:0;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9003,0[]
[1001-1023] KEYS:0;0.0.0.0:9601(Q:0 T:0us R:100.00);0.0.0.0:9004,0[]
```
## 系统监控与状态统计
proxy-server对每个slot处理的请求量、命令耗时、成功数都会定时上报到config-server；config-server可以按照redis-server、proxy-server两个维度汇总统计，进行监控。

--------允