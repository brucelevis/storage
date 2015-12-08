#可定制的私有代理服务器技术
##需求分析
中国的宽带运营商主要是电信和联通，存在着地域差异，跨电信运营商进行网络操作会有较大延迟。为了应对这种特有国情，游戏运营商会在电信机房和网通机房分别部署服务器，也有方案是部署在双线机房，但是成本比较高，一个可行的减少成本的方案就是，部署代理服务器在少量的双线机房，玩家选择代理服务器进行跨运营商网络访问时，可以显著减少延迟。该代理服务器必须满足高负载，低延迟。
##术语
client：游戏客户端
gameserver：游戏服务器
proxy：代理服务器
##数据流分析
client在代理模式下，连接proxy，连接成功后立即发送想要连接的gameserver，该数据包称之为IPMsg，proxy连接gameserver，连接成功后，client到gameserver的通道就完全建立起来。gameserver需要知道client的真实ip，而不是proxy的ip，proxy增加协议用于把client的真实ip告知gameserver，gameserver和client的通讯是加密通讯，所以proxy必须知道该加密方式，并模拟client的协议格式与gameserver通讯。
##设计要点

 1. proxy基本上是一个无状态服务器，几乎无逻辑，单进程，单线程，传统半同步办异步多线程模式不适合proxy，网络数据在网络层收到后如果再转移到逻辑层是一种浪费，应该是直接放到缓冲区供转发。
 2. 基于libev的事件驱动框架，所有套接字操作都需要异步无阻塞
 3. proxy到gameserver的连接需要超时机制
 4. 为了降低延迟，需要在收到client数据后立即转发，而不是放到缓冲区等待libev触发EV_WRITE后再转发
 5. 使用帧的概念，使用参数自定义帧数，可以模拟各种延迟，方便客户端测试
 
##代码详解
###缓冲区
```c
struct Buffer
{
	char buf_[64*1024];
	int  readpos_;
	int  writepos_;
	int  get_read_buffer(char*& ptr);
	int  get_write_buffer(char*& ptr);
	void add_read_pos(int add);
	void add_write_pos(int add);
}
```
###代理通道
proxy抽象的是client<--->proxy<--->gameserver，这条双向通道，所以需要两个缓冲区，两个套接字，4个ev_io对象，分别是客户端读，写，服务器读，写，用于libev进行事件注册。
```c
struct Proxy
{
	struct ev_loop* loop_;  // 事件循环体
	ev_timer conntimer_;    // 连接超时控制
	ev_io  clientr_;        // 注册客户端可读事件
	ev_io  clientw_;        // 注册客户端可写事件
	ev_io  serverr_;        // 注册服务器可读事件
	ev_io  serverw_;        // 注册服务器可写事件
	Buffer srv2cli_;        // server-->proxy-->client
	Buffer cli2srv_;        // client-->proxy-->server
	int    serverfd_;
	int    clientfd_;
	EClientState clistate_; // 管理client<-->proxy通道状态
	EServerState srvstate_; // 管理server<-->proxy通道状态
	char authmsg_[sizeof(MsgAuth)]; // client连接proxy后首先发送的MsgAuth，该数据不需要转发，用来做一些安全检查，以及告诉proxy需要连接的server的地址信息
	int  authsize_;
	char srvkey_[sizeof(SAllKey)];  // 解析server-->client的握手信息，用来获得通讯key，proxy模拟client发送客户端的真实ip
	int  keysize_;
}
```

##通道建立图
![enter image description here](https://raw.githubusercontent.com/doublefox1981/storage/master/handshake.png)

##数据流图
![enter image description here](https://raw.githubusercontent.com/doublefox1981/storage/master/transfer.png)