gRPC 'Error: 14 UNAVAILABLE: TCP Write failed' issue
最近搞nodejs grpc程序部在k8s，遇到了TCP Write failed问题，查遍了各种issue，终于找到了一个靠谱的解决：http://www.longfan.me/post/devops/2020-07-09。
防止丢失，这里复制一份

## 排查
打开 grpc 的 debug 
GRPC_TRACE = all
GRPC_VERBOSITY = DEBUG
生产加上这个重新部署之后在偶然的错误里面发现了一些细节，每个Error: 14 UNAVAILABLE: TCP Write failed错误后面的 grpc 的 log 都会出现这种类型的 debug 日志:
```
I0709 13:20:21.758757992      16 chttp2_transport.cc:839]    W:0x39fe7b0 CLIENT [ipv4:10.98.85.83:80] state WRITING -> IDLE [finish writing]
I0709 13:20:21.758762763      16 chttp2_transport.cc:2866]   transport 0x39fe7b0 set connectivity_state=4
I0709 13:20:21.758768114      16 connectivity_state.cc:147]  SET: 0x39fea58 client_transport: READY --> SHUTDOWN [close_transport]
I0709 13:20:21.758772872      16 connectivity_state.cc:160]  NOTIFY: 0x39fea58 client_transport: 0x3645a68
I0709 13:20:21.758791048      16 tcp_custom.cc:287]          TCP 0x3772c30 shutdown why={"created":"@1594272021.758670258","description":"Delayed close due to in-progress write","file":"../deps/grpc/src/core/ext/transport/chttp2/transport/chttp2_transport.cc","file_line":593,"referenced_errors":[{"created":"@1594272021.758639866","description":"TCP Write failed","file":"../deps/grpc/src/core/lib/iomgr/tcp_uv.cc","file_line":72,"grpc_status":14,"os_error":"connection reset by peer"},{"created":"@1594272021.758743251","description":"TCP Write failed","file":"../deps/grpc/src/core/lib/iomgr/tcp_uv.cc","file_line":72,"grpc_status":14,"os_error":"broken pipe"}]}
```
error 里面的第一个原因是一个"os_error":"connection reset by peer"这样的 error，但是具体什么会造成这样的错误呢？
研究了一下之前所有服务的 log，发现我们的测试服完全没有Error: 14 UNAVAILABLE: TCP Write failed，出现的仅仅在 UAT 和生产服里面，而测试服和生产服最大的区别就是测试服使用的 kube-proxy 是 iptables, UAT 和生产是 ipvs.
尝试寻找了一下网上看有没有人踩过这样的坑，结果一搜就搜出来一篇相关的文章，还是中文的[Kubernetes IPVS 模式下服务间长连接通讯的优化，解决 Connection reset by peer 问题 ---- 青蛙小白][https://blog.frognew.com/2018/12/kubernetes-ipvs-long-connection-optimize.html]. 看作者的问题描述，基本判断是遇到的同样的问题。

## 原因

通过阅读文章，原因锁定到了文章开头的一段话:_我们知道 gRPC 是基于 HTTP/2 协议的，gRPC 的 client 和 server 在交互时会建立多条连接，为了性能，这些连接都是长连接并且是一直保活的。 这段环境中不管是客户端服务还是 gRPC 服务都被调度到各个相同配置信息的 Kubernetes 节点上，这些 k8s 节点的 keep-alive 是一致的，如果出现连接主动关闭的问题，因为从 client 到 server 经历了一层 ipvs，所以最大的可能就是 ipvs 出将连接主动断开，而 client 端还不知情_
因为我们并没有改动过 k8s 节点的net.ipv4.tcp_keepalive_time，所以经过检查我们发现系统符合出现 issue 的描述
```
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_probes net.ipv4.tcp_keepalive_intvl
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_intvl = 75
```
ipvsadm -l --timeout
Timeout (tcp tcpfin udp): 900 120 300
通过研究 net.ipv4.tcp_keepalive 的具体用途和设置方法之后，找到了解决方法
[How to Configure Linux TCP keepalive Setting]

Solution
```
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_intvl = 30
```
600 + 9 * 30 = 870 < 900
尝试把net.ipv4.tcp_keepalive_time + net.ipv4.tcp_keepalive_probes * net.ipv4.tcp_keepalive_intvl改成了小于 900 的值以后，经过观察和测试，貌似出现Error: 14 UNAVAILABLE: TCP Write failed错误信息的概率变小了, 问题看起来暂时解决了一部分，准备再观察一段时间看看情况。
Solution+
观察了一晚上，发现Error: 14 UNAVAILABLE: TCP Write failed还是存在，如果理论上出现错误的原因是找正确了的，那么问题就出在解决问题的方法是否有效上面了。
在所有宿主机上面观察了net.ipv4.tcp_keepalive*的设置都是正确的
```
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_probes net.ipv4.tcp_keepalive_intvl
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_intvl = 30
```
但是容器里面真实的情况呢？
进入容器之后运行同样的命令，发现令人沮丧的消息
```
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_probes net.ipv4.tcp_keepalive_intvl
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_intvl = 75
```
宿主机的sysctl设置并没有进入容器，看来 namespace 的隔离还是非常彻底的，查阅了一些资料发现，如果要设置容器内部的net.ipv4.tcp_keepalive*，会涉及到使用非安全的内核设置，需要使用priviliegedPOD 设置，不太安全[Reference][https://stackoverflow.com/questions/54552379/tcp-keepalive-time-in-docker-container/54564456#54564456].看看能不能另辟蹊径。
通过阅读grpc库的[C++文档][https://grpc.github.io/grpc/cpp/md_doc_keepalive.html]我们发现可以使用一些设置来注入grpc库，来改变库本身的对于 tcp keepalive 的频率设置。
GRPC_ARG_KEEPALIVE_TIME_MS可以设置库对于 keepalive 探针时间的频率，我们可以设置得小于等于 900 秒，同时打开设置GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS允许没有请求的时候也发送 keepalive 探针，GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA设为 0 避免服务端认为过高频率的探针是恶意探针而屏蔽掉。以 nodejs 为例
```js
const Client = grpc.makeGenericClientConstructor({}, null, null);
const grpcClient = new Client(
  process.env.GRPC_SERVER_URL,
  grpc.credentials.createInsecure(),
  {
    'grpc.keepalive_time_ms': parseInt(process.env.GRPC_ARG_KEEPALIVE_TIME_MS || '7200000'),
    'grpc.keepalive_permit_without_calls': parseInt(
      process.env.GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS || '0',
    ),
    'grpc.http2.max_pings_without_data': parseInt(
      process.env.GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA || '2',
    ),
  }
);
```
又观察了一晚上发现没有任何报错了，问题解决。
