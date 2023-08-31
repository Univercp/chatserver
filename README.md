# chatserver
 集群聊天系统 

项目简介：该项目基于 C/S 架构实现的聊天系统，包括注册、登录、一对一/群组聊天、好友管理、群组创建等功能。通过数据、 业务、网络模块的解耦，采用 Json 格式传输数据。服务器集群使用 Nginx 进行负载均衡，借助 Redis 发布-订阅实现服务器间通信。该聊天系统实现了稳定、高效、灵活的通信聊天。 

技术选型：Json、Muduo、Nginx、Redis、MySqL、CMake 等。 

任务职责： 

⚫ 使用 Json 序列化和反序列化将复杂结构数据转化为字符串，便于客户端和服务端的数据传输，提升系统的互操作性。

⚫ 使用 Muduo 网络框架实现高性能的网络通信、多线程处理、定时器管理等功能，构建稳定、高效的聊天系统。 

⚫ 使用 Nginx 源码编译安装并进行环境部署，配置 TCP 负载均衡，实现将传入的 TCP 连接分发，提高系统的负载能力。 

⚫ 使用基于发布-订阅的服务器中间件，实现 Redis 消息队列处理实时消息的发布、订阅和分发，增强系统实时通信能力。


效果展示：
![替代文本](https://github.com/Univercp/chatserver/blob/main/src/github.png)


上图创建了两台服务端程序，一个工作在6000端口，一个工作在6002端口，每新增一台服务端程序，需要将服务端程序的IP地址和端口写入nginx：stream中进行配置。

#整体流程：
#1.首先是安装mysql 和 mysql-client ；创建数据库：chat，在数据库chat建立了5张表

user

Offlinemsg

Allgroup

Groupuser

Friend


每张表的用途如下：

User表：设置userid作为表主键，通过主键，可以存储用户的name,密码password，state用来存放User在线信息。

Friend表：是一个存放用户id 和 friended 的表，这个表是将userid和friendid设置为联合主键。因为朋友之间的关系组合是唯一的。

Offlinemsg表：Userid，就是哪个用户是否有离线信息，这里不设置为主键（即不唯一），让message可以存在多条。

Allgroup表：Id:为群组id，唯一表示，用于给groupuser表的用户组进行管理。groupname：群组名称Groipdesc:描述群主

Groupuser表：Groupid 是群主id 的联合查询，下面接着是userid,grouprole为角色。用于查找userid转发给groupid中的所有userid。


#2.使用cmake创建工程

安装cmake(apt install cmake , apt install g++)

Cmake工程建立：

CMaleLists.txt
{
	#设置最小版本
	#设置编译选项
	#设置工程目录
	#设置可执行文件路径（输出的）
	#设置头文件搜索路径
	#设置子文件目录
}



建立工程：
bin(存放可执行程序)
      build(存放编译过程文件)
	  include
	  src(资源文件)
	  thirdparty
	  CMakeLsits.txt

Json 序列化和反序列化：
#include "json.hpp"
using json = nlohmann::json;

json js ; //生成js对象
js[“{key}”] = {“value”} js对象加入键值对
js.dump() js序列化成字符串

反序列化
Json js ;
Js = json::parse({对象 js.dump()});
通过键来查值
Js[“key”]返回value;

#2.网络模块

网络模块完成的任务就是使用muduo建立client 和 serverd端的链接，并能通过发送和介绍js字符串。Client与Server建立连接后，Client序列化各种信息成js字符串。Server反序列化接受到的字符串成为各种信息，比如操作指令，用户id

安装muduo库网络通信框架
{

安装 boost库
sudo apt install libboost-all-dev

安装 muduo库
git clone https://github.com/chenshuo/muduo.git
sudo apt install cmake g++（如果没有cmake 和g++的话）
cd muduo
mkdir build
cd build
cmake ..
make
make install（安装在全局）
}

ChatServer.hpp
{
#include <muduo/net/TcpServer.h>
#include <muduo/net/EventLoop.h>
using namespace muduo;
using namespace muduo::net;
// 服务器类，基于muduo库开发
class ChatServer
{
public:
  // 初始化聊天服务器对象
  ChatServer(EventLoop *loop,
    const InetAddress &listenAddr,
    const std::string &nameArg);

    // 开启事件循环
  void start();

private:
    // 连接相关信息的回调函数（新连接到来/旧连接断开）
    void onConnection(const TcpConnectionPtr &);

    // 读写事件相关信息的回调函数
    void onMessage(const TcpConnectionPtr &,
                   Buffer *,
                   Timestamp);

  TcpServer _server;  
  EventLoop *_loop;
};
声明ChatServer对象，初始化时需要传给构造函数 Eventloop 对象，用于循环监听事件，还传入一个 listenAddress 对象用于绑定该服务器应用网络地址和端口；
在private:中 有 TcpServer _server;对象作为服务端接收端。事件循环其
并且在事件循环中，监听TcpConnectionPtr 一个智能指针的tcp socket链接，当监听到事件后，就建立tcp socket链接，将该事件分发给 worker线程，调用onMessage
{
	String buf = buffer ->retrieveALLAsString();
	Json js = json:parse(buf);
	//将该指令传给 ChatService对象，调用相应的回调函数，用来对数据库的增删改查和好友聊天等。
}
}

ChatServer.cpp
{
	实现 start()
	构造函数初始化ChatServer对象->给TcpServer 对象复制，Server（loop,ip和端口，name）
	监听事件setconnectioncall
	回调事件
	设置线程数量
}

#3.业务模块 
ChatService.hpp -> ChatService.cpp
业务模块是一个单例模式对象。一个类一个实例化对象。整个进程从生到死只有一个业务对象，同一对数据模块（mysql）这部分共享管理，同一管理，反正发生数据读写混乱。
业务模块对象构造只有一次，该对象只针对js发过的msgid操作指令进行回调
{
	class ChatService
{
public:
    // ChatService 单例模式
    // thread safe
    static ChatService* instance() {
        static ChatService service;
        return &service;
    }

    // 登录业务
    void loginHandler(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 注册业务
    void registerHandler(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 一对一聊天业务
    void oneChatHandler(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 添加好友业务
    void addFriendHandler(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 获取对应消息的处理器
    MsgHandler getHandler(int msgId);
    // 创建群组业务
    void createGroup(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 加入群组业务
    void addGroup(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 群组聊天业务
    void groupChat(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 处理客户端异常退出
    void clientCloseExceptionHandler(const TcpConnectionPtr &conn);
    // 服务端异常终止之后的操作
    void reset();
    //redis订阅消息触发的回调函数
    void redis_subscribe_message_handler(int channel, string message);

private:
    ChatService();
    ChatService(const ChatService&) = delete;
    ChatService& operator=(const ChatService&) = delete;

    // 存储消息id和其对应的业务处理方法
    std::unordered_map<int, MsgHandler> _msgHandlerMap;
    
    // 存储在线用户的通信连接
    std::unordered_map<int, TcpConnectionPtr> _userConnMap;

    // 定义互斥锁
    std::mutex _connMutex;

    //redis操作对象
    Redis _redis;

    // 数据操作类对象
    UserModel _userModel;
    OfflineMsgModel _offlineMsgModel;
    FriendModel _friendModel;
    GroupModel _groupModel;
};

}

回调机制
{
	Msghandle =  Function<void(TcpConnect,js& js,timesnap)>
	Unordered_map<int,msghandle> map; //用来存放指令
	ChatService.gethandle[int i]
	{Auto it = _sghanlermap.find(msgid) return msghandle}
}

业务应用（服务端中的事情）
{
	// 登录业务
    void loginHandler(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 注册业务
    void registerHandler(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 一对一聊天业务
    void oneChatHandler(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 添加好友业务
    void addFriendHandler(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 获取对应消息的处理器
    MsgHandler getHandler(int msgId);
    // 创建群组业务
    void createGroup(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 加入群组业务
    void addGroup(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 群组聊天业务
    void groupChat(const TcpConnectionPtr &conn, json &js, Timestamp time);
    // 处理客户端异常退出
    void clientCloseExceptionHandler(const TcpConnectionPtr &conn);
    // 服务端异常终止之后的操作
    void reset();
    //redis订阅消息触发的回调函数
    void redis_subscribe_message_handler(int channel, string message);

}

{
	登录业务：
判断是否在线，在线的话不允许登录。客户端与服务端建立TCP连接后，发送js字符串，其中包括操作指标msgid，通过回调器，调用login函数。使用一个类User ,记录User类的信息，更新User表。将该用户的userid更新在在线map中，并且将userid使用Redis订阅器订阅在channel为useid的频道上。
	注册业务：
	将js字符串解析，调用注册函数，将userid password 加入到user表
	
	添加好友业务：
	将js字符串解析，调用注册函数，将userid friendid加入到friend表,该表是联合主键，组合记录是唯一的，确保无重复的朋友关系。
	
一对一聊天业务：
查看对方是否在线，在线的话，通过用户在线的一个map,找到对放的TCP链接，然后转发信息给该用户。若是对方没在线，就将信息存放在离线信息表。表中有字段是userid和messages,当该用户再次上线时，会自动向该表提取信息，并删除离线信息。
}

{
数据处理模块
定义了一个db操作类，各各种数据表进行处理。首先是初始化数据库，然后链接数据库，执行sql语句更新操作。
}

#4.Nginx TCP负载均衡算法：会自动将Client的程序连接到不同的服务器的程序上
{
Nginx
安装sudo apt install nginx
配置nginx
vim /etc/nginx/nginx.conf
stream{
        upstream Myserver{

                server 127.0.0.1:6000 weight=1 max_fails=3 fail_timeout=30s;
                server 127.0.0.1:6002 weight=1 max_fails=3 fail_timeout=30s;
        }
        server{
                proxy_connect_timeout 1s;
                #proxy_timeout 3s;
                listen 8000;
                proxy_pass MyServer;
                tcp_nodelay on;

        }
}
将所有服务端程序加入到nginx负载均衡器上
}

#5.Redis 发布订阅中间件：基于观察者模式，每个服务器程序将自己的userid订阅subscribe到Redis中，充当一个观察者身份；其他通道充当被观察者身份。如果其他通道在该通道上有消息Publish，观察者就会接收到相应的信息，实现服务器之间的通信（跨服务器通信）
{
Redis 发布订阅中间件
sudo apt install redis-server
将userid作为每一个通道，链接不同服务器，实现服务器之间的低耦合链接，提供一个高质量的多服务器程序通信。
}

小结：Resdis发布订阅器和nginx的Tcp负载均衡，这两个中间件是提升系统的并发能力的。将一台服务器上的链接压力分发到多台（Nginx负载均衡器），分发后实现服务器通信（Redis）.网络通信模块使用 Muduo库，自动监听连接IO事件（EVentloop），监听到的链接事件分发给worker（回调指令->业务模块），业务模块根据指令实现：注册，登录，下线，一对一聊天，创建/加入群/群聊，离线聊天->数据模块（db.h数据库连接与更新）。
