# mychatserver
基于muduo库实现的工作在nginx.tcp负载均衡环境中的集群服务聊天器。
一、技术栈
json序列化和反序列化，muduo网络库开发，sql，数据库编程，cmake构建编译环境，单例设计，ORM对象关系映射，多线程编程，线程安全，nginx的tcp负载均衡器配置，基于发布-订阅的服务器中间件redis消息队列编程实践.
二、编译方式：
运行脚本autobuild.sh:
cd build
rm -rf *
cmake ..
make
三、项目相关细节
1.应用场景：局域网的聊天。
2.服务器分层设计思想：
网络I/O模块：onmessage回调函数与onconnection回调函数。
业务模块：CHATSERVICE-USER:log in（记录连接<线程安全>，更新state,返回ack信息，查询离线消息<读取并删除>，返回好友列表，返回群组列表）,register,one_chat,clientconnexception，服务器异常(ctrl+c)重置user状态，添加好友业务。GROUP:创建群组。加入群组。注销业务。
数据模块:ORM映射表，model 进行相应更新。
分层的设计思想。
3.json-数据序列化方式。使用json就像使用stl容器一样，并且stl与json可以互相转换。
js.dump()：是 nlohmann::json 类型的一个成员函数，它会把 JSON 对象序列化为 字符串（std::string）。
parse(string str):将字符串反序列化为json对象。
4.muduo:网络库，实际上就是epoll+线程池的封装，优点是能够把网络I/O的代码（由网络库实现）与业务代码（用户的连接与断开，用户的读写事件）区分开。
在chatserver类中：
TcpServer成员_server，EventLoop指针_loop。
构造函数中先初始化_server与_loop，然后负责给服务器_server.setConnectionCallback注册用户连接的创建于断开回调_server.setMessageCallback以及给服务器注册用户读写事件回调。
onConnection与onMessage方法专门负责处理用户连接创建和断开，以及用户的读写事件。
setThreadNum=4(1个I/O线程（main reactor）负责新用户的连接 3个worker线程（sub reactor）负责已连接用户的读写事件)——muduo库会自己分配。
5.thirdparty:第三方库，这里是json.hpp
6.TcpServer::start函数作用：
void TcpServer::start() {
    // 1. 检查线程池是否已启动
    if (!started_.exchange(true)) {
        // 2. 启动 I/O 线程池（若有）
        threadPool_->start(threadInitCallback_);

        // 3. 确保主 EventLoop 已运行
        loop_->runInLoop([this]() {
            // 4. 启动 Acceptor 监听
            acceptor_->listen();
        });
    }
}
7.TcpConnection::shutdown函数：
函数	行为	适用场景
shutdown()	优雅关闭写端，等待输出缓冲区数据发送完毕，对端仍可发送数据。	需要半关闭（如 HTTP Keep-Alive）
forceClose()	立即关闭连接（调用 ::close(fd)），丢弃未发送数据。	错误处理或强制终止连接
8.chatservice类是单例设计，负责处理业务模块，包括登录与注册业务。
9.using MsgHandler = std::function<void(const TcpConnectionPtr &conn, json &js, Timestamp)>;--用一个接口封装不同类型的可调用对象，类型擦除。
定义名为MsgHandler的类型别名，等价于右边的标准库模板类std::function<>，提供一个接口，封装任意可调用对象:函数签名void(const TcpConnectionPtr &conn, json &js, Timestamp)，表示一个返回 void，接受三个参数的函数。
灵活性：
使用 std::function 替代传统函数指针，支持绑定到：普通函数（如 handleLogin）。Lambda 表达式（如 [&](auto...){...}）。成员函数（需结合 std::bind 或 this 捕获）。
类型安全：
明确参数类型（如 TcpConnectionPtr、json），避免低级错误。
解耦：
分离消息解析与处理逻辑，便于扩展和维护。
8.连接时的线程安全操作：定义互斥量_connmutex，使用lockguard<mutex> lock(_connmutex);自动管理加锁与解锁。数据库增删改查的多线程操作由mysqlserver自动管理，用户无需管理。
9.数据量：1-2w万级。
10.客户端实现：main线程用作发送线程，子线程readtaskhandler作为接收线程。
11.nginx负载均衡器：采用nginx的tcp负载均衡模块。1）负载算法派发client的requests给server2）采用心跳机制检测server健康3)发现添加新的server
12.解决跨服务器聊天问题：登录在server1上的client1与client2进行onechat时，server1上的_userconnmap查询不到client2的连接会默认client离线而发送离线消息给client2，但实际上client2已在server2上登录。如何解决：如果让所有server之间建立连接，设计会极其复杂，耦合度过高。解决方法：采用发布-订阅的redis消息队列。（subscribe,publish,notify）
13.配置nginx的tcp负载均衡模块时，要加上--with-stream来激活模块。
在nginx配置文件中指定监听8000端口号。
14.redis-include<hiredis.h> rediscontext对象publish_context,subscribe_context执行connect端口号为6379.
在单独的线程中调用observer函数用于观察监听channel上的事件，有消息就上报给业务层chatservice。
rediscommand=redisappencommand+redisbufferwrite+redisgetreply
subscribe本身会阻塞通道等待消息，但我们设计只subscribe不接受redis server的reply消息。因此只调用redisappendcommand与redisbufferwrite，消除阻塞。
而publish不会阻塞通道，因此可以直接用rediscommand。
