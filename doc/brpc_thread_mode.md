brpc源码分析——线程模型

saddlesad

于 2022-03-14 16:27:42 发布

2859
 收藏 1
分类专栏： 开源项目阅读 文章标签： 网络协议
版权

开源项目阅读
专栏收录该内容
7 篇文章0 订阅
订阅专栏
brpc线程模型
从一个server的启动过程谈起，我们这里以echo server为例：

int main(int argc, char* argv[]) {
    // gflags介绍：https://blog.csdn.net/lezardfu/article/details/23753741
    // Parse gflags. We recommend you to use gflags as well.
    GFLAGS_NS::ParseCommandLineFlags(&argc, &argv, true);

    // Generally you only need one Server.
    brpc::Server server;
    
    // Instance of your service.
    example::EchoServiceImpl echo_service_impl;


    // Add the service into server. Notice the second parameter, because the
    // service is put on stack, we don't want server to delete it, otherwise
    // use brpc::SERVER_OWNS_SERVICE.
    if (server.AddService(&echo_service_impl, 
                          brpc::SERVER_DOESNT_OWN_SERVICE) != 0) {
        LOG(ERROR) << "Fail to add service";
        return -1;
    }
    
    // Start the server.
    brpc::ServerOptions options;
    options.idle_timeout_sec = FLAGS_idle_timeout_s;
    if (server.Start(FLAGS_port, &options) != 0) {
        LOG(ERROR) << "Fail to start EchoServer";
        return -1;
    }
    
    // Wait until Ctrl-C is pressed, then Stop() and Join() the server.
    server.RunUntilAskedToQuit();
    return 0;
}


GFLAGS_NS::ParseCommandLineFlags用于按照预定义规则解析命令行参数，Server::AddService会注册服务，而Server::Start真正启动服务器，Server::RunUntilAskedToQuit阻塞等待服务结束。

int Server::AddService(google::protobuf::Service* service,
                       ServiceOwnership ownership) {
    ServiceOptions options;
    options.ownership = ownership;
    return AddServiceInternal(service, false, options);
}

Server::AddService()会调用Server::AddServiceInternal()方法：

检查service是否有效以及service是否定义了至少一个方法，如果没有则报错。
调用Server::InitializeOnce()，以初始化server配置，此初始化过程由pthread_once保障只执行一次。
如果同名服务已存在则报错返回。
将service_name与service实体绑定保存在map中。
int Server::Start(const butil::EndPoint& endpoint, const ServerOptions* opt) {
    return StartInternal(
        endpoint.ip, PortRange(endpoint.port, endpoint.port), opt);
}

Server::Start()会调用Server::StartInternal()方法：

调用Server::InitializeOnce()。
检查状态，只有server处于READY态时继续启动，否则输出信息返回。
根据配置准备bthread，SSL等，开启若干个线程服务于bthread。
从min_port到max_port依次尝试调用bind() + listen()，直到有一个端口可用；如果没有可用端口则报错返回，如果port为0，表示让内核自行选择一个端口，那么brpc会使用getsockname()来获取内核选择的端口。
调用Acceptor::StartAccept()来注册accept事件。
InitializeOnce()
熟悉了大概流程后，我们来看一下Server::InitializeOnce()的实现细节：

int Server::InitializeOnce() {
    if (_status != UNINITIALIZED) {
        return 0;
    }
    GlobalInitializeOrDie();

    if (_status != UNINITIALIZED) {
        return 0;
    }
    if (_fullname_service_map.init(INITIAL_SERVICE_CAP) != 0) {
        LOG(ERROR) << "Fail to init _fullname_service_map";
        return -1;
    }
    if (_service_map.init(INITIAL_SERVICE_CAP) != 0) {
        LOG(ERROR) << "Fail to init _service_map";
        return -1;
    }
    if (_method_map.init(INITIAL_SERVICE_CAP * 2) != 0) {
        LOG(ERROR) << "Fail to init _method_map";
        return -1;
    }
    if (_ssl_ctx_map.init(INITIAL_CERT_MAP) != 0) {
        LOG(ERROR) << "Fail to init _ssl_ctx_map";
        return -1;
    }
    _status = READY;
    return 0;
}


InitializeOnce()在确保status处于UNINITIALIZED状态后调用GlobalInitializeOrDie()，然后初始化service_map、method_map等多个map，最后将status设为READY，确保GlobalInitializeOrDie()只被调用一次。

void GlobalInitializeOrDie() {
    if (pthread_once(&register_extensions_once,
                     GlobalInitializeOrDieImpl) != 0) {
        LOG(FATAL) << "Fail to pthread_once";
        exit(1);
    }
}


GlobalInitializeOrDie()函数使用pthread_once来调用GlobalInitializeOrDieImpl()，保证多个线程也一次进行一次初始化。

static void GlobalInitializeOrDieImpl() {
    // Ignore SIGPIPE.
    struct sigaction oldact;
    if (sigaction(SIGPIPE, NULL, &oldact) != 0 ||
            (oldact.sa_handler == NULL && oldact.sa_sigaction == NULL)) {
        CHECK(NULL == signal(SIGPIPE, SIG_IGN));
    }
    // ...
    // Initialize openssl library
    SSL_library_init();
    // RPC doesn't require openssl.cnf, users can load it by themselves if needed
    SSL_load_error_strings();
    if (SSLThreadInit() != 0 || SSLDHInit() != 0) {
        exit(1);
    }
    // ...
    // Naming Services
    NamingServiceExtension()->RegisterOrDie("http", &g_ext->dns);
    NamingServiceExtension()->RegisterOrDie("https", &g_ext->dns_with_ssl);
    // ...
    // Load Balancers
    LoadBalancerExtension()->RegisterOrDie("rr", &g_ext->rr_lb);
    LoadBalancerExtension()->RegisterOrDie("wrr", &g_ext->wrr_lb);
    LoadBalancerExtension()->RegisterOrDie("random", &g_ext->randomized_lb);
    // Compress Handlers
    const CompressHandler gzip_compress =
        { GzipCompress, GzipDecompress, "gzip" };
    if (RegisterCompressHandler(COMPRESS_TYPE_GZIP, gzip_compress) != 0) {
        exit(1);
    }
    // ...
    
    // Protocols
    Protocol baidu_protocol = { ParseRpcMessage,
                                SerializeRequestDefault, PackRpcRequest,
                                ProcessRpcRequest, ProcessRpcResponse,
                                VerifyRpcRequest, NULL, NULL,
                                CONNECTION_TYPE_ALL, "baidu_std" };
    if (RegisterProtocol(PROTOCOL_BAIDU_STD, baidu_protocol) != 0) {
        exit(1);
    }
    Protocol http_protocol = { ParseHttpMessage,
                               SerializeHttpRequest, PackHttpRequest,
                               ProcessHttpRequest, ProcessHttpResponse,
                               VerifyHttpRequest, ParseHttpServerAddress,
                               GetHttpMethodName,
                               CONNECTION_TYPE_POOLED_AND_SHORT,
                               "http" };
    if (RegisterProtocol(PROTOCOL_HTTP, http_protocol) != 0) {
        exit(1);
    }
    // ...
    std::vector<Protocol> protocols;
    ListProtocols(&protocols);
    for (size_t i = 0; i < protocols.size(); ++i) {
        if (protocols[i].process_response) {
            InputMessageHandler handler;
            // `process_response' is required at client side
            handler.parse = protocols[i].parse;
            handler.process = protocols[i].process_response;
            // No need to verify at client side
            handler.verify = NULL;
            handler.arg = NULL;
            handler.name = protocols[i].name;
            if (get_or_new_client_side_messenger()->AddHandler(handler) != 0) {
                exit(1);
            }
        }
    }
    // ...
}


在GlobalInitializeOrDieImpl()内首先忽略掉SIGPIPE，然后初始化SSL，注册各种NamingService，LoadBalancer，CompressHandler；然后注册各种协议的回调函数簇，具体每个回调函数做什么以及如何交互的将在下篇讲解。之后，GlobalInitializeOrDieImpl()还会再次遍历所有先前注册的协议，将支持process_response的协议注册到客户端的协议map中。

StartInternal()
现在让我们把重点放在server端的socket() + bind() + listen() + accept()过程上，下面是Server::StartInternal()中的部分代码：

_listen_addr.ip = ip;
for (int port = port_range.min_port; port <= port_range.max_port; ++port) {
    _listen_addr.port = port;
    butil::fd_guard sockfd(tcp_listen(_listen_addr));
    if (sockfd < 0) {
        if (port != port_range.max_port) { // not the last port, try next
            continue;
        }
        if (port_range.min_port != port_range.max_port) {
            LOG(ERROR) << "Fail to listen " << ip
                << ":[" << port_range.min_port << '-'
                << port_range.max_port << ']';
        } else {
            LOG(ERROR) << "Fail to listen " << _listen_addr;
        }
        return -1;
    }
    if (_listen_addr.port == 0) {
        // port=0 makes kernel dynamically select a port from
        // https://en.wikipedia.org/wiki/Ephemeral_port
        _listen_addr.port = get_port_from_fd(sockfd);
        if (_listen_addr.port <= 0) {
            LOG(ERROR) << "Fail to get port from fd=" << sockfd;
            return -1;
        }
    }
    if (_am == NULL) {
        _am = BuildAcceptor();
        if (NULL == _am) {
            LOG(ERROR) << "Fail to build acceptor";
            return -1;
        }
    }
    // Set `_status' to RUNNING before accepting connections
    // to prevent requests being rejected as ELOGOFF
    _status = RUNNING;
    time(&_last_start_time);
    GenerateVersionIfNeeded();
    g_running_server_count.fetch_add(1, butil::memory_order_relaxed);

    // Pass ownership of `sockfd' to `_am'
    if (_am->StartAccept(sockfd, _options.idle_timeout_sec,
                         _default_ssl_ctx) != 0) {
        LOG(ERROR) << "Fail to start acceptor";
        return -1;
    }
    sockfd.release();
    break; // stop trying
}



其中，butil::tcp_listen()会执行对listened_fd的socket() + bind() + listen()系统调用：

int tcp_listen(EndPoint point) {
    fd_guard sockfd(socket(AF_INET, SOCK_STREAM, 0));
    if (sockfd < 0) {
        return -1;
    }

    if (FLAGS_reuse_addr) {
        const int on = 1;
        if (setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR,
                       &on, sizeof(on)) != 0) {
            return -1;
        }
    }
    
    if (FLAGS_reuse_port) {
        const int on = 1;
        if (setsockopt(sockfd, SOL_SOCKET, SO_REUSEPORT,
                       &on, sizeof(on)) != 0) {
            LOG(WARNING) << "Fail to setsockopt SO_REUSEPORT of sockfd=" << sockfd;
        }
    }
    
    struct sockaddr_in serv_addr;
    bzero((char*)&serv_addr, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr = point.ip;
    serv_addr.sin_port = htons(point.port);
    if (bind(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) != 0) {
        return -1;
    }
    if (listen(sockfd, 65535) != 0) {
        //             ^^^ kernel would silently truncate backlog to the value
        //             defined in /proc/sys/net/core/somaxconn if it is less
        //             than 65535
        return -1;
    }
    return sockfd.release();
}



可以看到tcp_listen首先调用socket()创建一个IPV4的TCP套接字，默认情况下还会开启SO_REUSEADDR套接字选项，但是因为FLAGS_reuse_port默认值为false，所以不会开启SO_REUSEPORT套接字选项，随后调用bind()为套接字指定监听地址和端口，最后调用listen(sockfd, 65535)指定尽可能大的连接队列。当当前端口可用时（bind()不返回-1）tcp_listen将返回处于LISTEN态的监听套接字。

Server::StartInternal()还会调用Server::BuildAcceptor()：

// ...
std::vector<Protocol> protocols;
ListProtocols(&protocols);
for (size_t i = 0; i < protocols.size(); ++i) {
    if (protocols[i].process_request == NULL) {
        // The protocol does not support server-side.
        continue;
    }
    if (has_whitelist &&
        !is_http_protocol(protocols[i].name) &&
        !whitelist.erase(protocols[i].name)) {
        // the protocol is not allowed to serve.
        RPC_VLOG << "Skip protocol=" << protocols[i].name;
        continue;
    }
    // `process_request' is required at server side
    handler.parse = protocols[i].parse;
    handler.process = protocols[i].process_request;
    handler.verify = protocols[i].verify;
    handler.arg = this;
    handler.name = protocols[i].name;
    if (acceptor->AddHandler(handler) != 0) {
        LOG(ERROR) << "Fail to add handler into Acceptor("
            << acceptor << ')';
        delete acceptor;
        return NULL;
    }
}


BuildAcceptor扫描所有协议，将支持process_request功能的协议注册入服务端的协议map中。

执行完上述过程之后，Server::StartInternal()会调用Acceptor::StartAccept()：

int Acceptor::StartAccept(int listened_fd, int idle_timeout_sec,
                          const std::shared_ptr<SocketSSLContext>& ssl_ctx) {
    if (listened_fd < 0) {
        LOG(FATAL) << "Invalid listened_fd=" << listened_fd;
        return -1;
    }
    
    BAIDU_SCOPED_LOCK(_map_mutex);
    if (_status == UNINITIALIZED) {
        if (Initialize() != 0) {
            LOG(FATAL) << "Fail to initialize Acceptor";
            return -1;
        }
        _status = READY;
    }
    if (_status != READY) {
        LOG(FATAL) << "Acceptor hasn't stopped yet: status=" << status();
        return -1;
    }
    if (idle_timeout_sec > 0) {
        if (bthread_start_background(&_close_idle_tid, NULL,
                                     CloseIdleConnections, this) != 0) {
            LOG(FATAL) << "Fail to start bthread";
            return -1;
        }
    }
    _idle_timeout_sec = idle_timeout_sec;
    _ssl_ctx = ssl_ctx;
    
    // Creation of _acception_id is inside lock so that OnNewConnections
    // (which may run immediately) should see sane fields set below.
    SocketOptions options;
    options.fd = listened_fd;
    options.user = this;
    options.on_edge_triggered_events = OnNewConnections;
    if (Socket::Create(options, &_acception_id) != 0) {
        // Close-idle-socket thread will be stopped inside destructor
        LOG(FATAL) << "Fail to create _acception_id";
        return -1;
    }
    
    _listened_fd = listened_fd;
    _status = RUNNING;
    return 0;
}



可以看到StartAccept照例检查状态是否为READY，另外如果idle_timeout_sec > 0，那么还要为当前Acceptor开启一个bthread用来检测该连接是否空闲（空闲一段时间后就要关闭），然后注册新事件的回调函数为options.on_edge_triggered_events = OnNewConnections，调用Socket::Create来将listened_fd加入epoll监听队列中，最后设置_status = RUNNING，服务器正式启动。

int Socket::Create(const SocketOptions& options, SocketId* id) {
    butil::ResourceId<Socket> slot;
    Socket* const m = butil::get_resource(&slot, Forbidden());
    if (m == NULL) {
        LOG(FATAL) << "Fail to get_resource<Socket>";
        return -1;
    }
    g_vars->nsocket << 1;
    CHECK(NULL == m->_shared_part.load(butil::memory_order_relaxed));
    m->_nevent.store(0, butil::memory_order_relaxed);
    m->_keytable_pool = options.keytable_pool;
    // ...
    const int rc2 = bthread_id_create(&m->_auth_id, NULL, NULL);
    if (rc2) {
        LOG(ERROR) << "Fail to create auth_id: " << berror(rc2);
        m->SetFailed(rc2, "Fail to create auth_id: %s", berror(rc2));
        return -1;
    }
    // ...
    const int rc = bthread_id_list_init(&m->_id_wait_list, 512, 512);
    if (rc) {
        LOG(ERROR) << "Fail to init _id_wait_list: " << berror(rc);
        m->SetFailed(rc, "Fail to init _id_wait_list: %s", berror(rc));
        return -1;
    }
    m->_last_writetime_us.store(cpuwide_now, butil::memory_order_relaxed);
    m->_unwritten_bytes.store(0, butil::memory_order_relaxed);
    CHECK(NULL == m->_write_head.load(butil::memory_order_relaxed));
    // Must be last one! Internal fields of this Socket may be access
    // just after calling ResetFileDescriptor.
    if (m->ResetFileDescriptor(options.fd) != 0) {
        const int saved_errno = errno;
        PLOG(ERROR) << "Fail to ResetFileDescriptor";
        m->SetFailed(saved_errno, "Fail to ResetFileDescriptor: %s", 
                     berror(saved_errno));
        return -1;
    }
    *id = m->_this_id;
    return 0;
}


Socket::Create()详尽地设置了Socket类型参数m的各种字段，然后调用Socket::ResetFileDescriptor()：

int Socket::ResetFileDescriptor(int fd) {
	// ...
    // FIXME : close-on-exec should be set by new syscalls or worse: set right
    // after fd-creation syscall. Setting at here has higher probabilities of
    // race condition.
    butil::make_close_on_exec(fd);

    // Make the fd non-blocking.
    if (butil::make_non_blocking(fd) != 0) {
        PLOG(ERROR) << "Fail to set fd=" << fd << " to non-blocking";
        return -1;
    }
    // turn off nagling.
    // OK to fail, namely unix domain socket does not support this.
    butil::make_no_delay(fd);
    if (_tos > 0 &&
        setsockopt(fd, IPPROTO_IP, IP_TOS, &_tos, sizeof(_tos)) < 0) {
        PLOG(FATAL) << "Fail to set tos of fd=" << fd << " to " << _tos;
    }
    
    if (FLAGS_socket_send_buffer_size > 0) {
        int buff_size = FLAGS_socket_send_buffer_size;
        socklen_t size = sizeof(buff_size);
        if (setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &buff_size, size) != 0) {
            PLOG(FATAL) << "Fail to set sndbuf of fd=" << fd << " to " 
                        << buff_size;
        }
    }
    
    if (FLAGS_socket_recv_buffer_size > 0) {
        int buff_size = FLAGS_socket_recv_buffer_size;
        socklen_t size = sizeof(buff_size);
        if (setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &buff_size, size) != 0) {
            PLOG(FATAL) << "Fail to set rcvbuf of fd=" << fd << " to " 
                        << buff_size;
        }
    }
    
    if (_on_edge_triggered_events) {
        if (GetGlobalEventDispatcher(fd).AddConsumer(id(), fd) != 0) {
            PLOG(ERROR) << "Fail to add SocketId=" << id() 
                        << " into EventDispatcher";
            _fd.store(-1, butil::memory_order_release);
            return -1;
        }
    }
    return 0;
}



ResetFileDescriptor会为当前套接字设置close-on-exec、non-blocking、no-delay（禁用nagle算法），如果用户指定那么还将设置当前套接字的接受缓冲区和发送缓冲区大小，最后调用GetGlobalEventDispatcher(fd).AddConsumer(id(), fd)。

void InitializeGlobalDispatchers() {
    g_edisp = new EventDispatcher[FLAGS_event_dispatcher_num];
    for (int i = 0; i < FLAGS_event_dispatcher_num; ++i) {
        const bthread_attr_t attr = FLAGS_usercode_in_pthread ?
            BTHREAD_ATTR_PTHREAD : BTHREAD_ATTR_NORMAL;
        CHECK_EQ(0, g_edisp[i].Start(&attr));
    }
    // This atexit is will be run before g_task_control.stop() because above
    // Start() initializes g_task_control by creating bthread (to run epoll/kqueue).
    CHECK_EQ(0, atexit(StopAndJoinGlobalDispatchers));
}

EventDispatcher& GetGlobalEventDispatcher(int fd) {
    pthread_once(&g_edisp_once, InitializeGlobalDispatchers);
    if (FLAGS_event_dispatcher_num == 1) {
        return g_edisp[0];
    }
    int index = butil::fmix32(fd) % FLAGS_event_dispatcher_num;
    return g_edisp[index];
}

第一次调用GetGlobalEventDispatcher时会初始化FLAGS_event_dispatcher_num个EventDispatcher全局对象和相同数量的bthread，每个bthread中都执行事件循环——EventDispatcher::Run()。

EventDispatcher的构造函数会调用epoll_create，可见每个event_dispatcher都有一个epollfd。

显然这里根据fd的哈希值来随机选择一个event_dispatcher。

而AddConsumer()将fd添加入epoll监听队列中，并注册EPOLLIN事件（边缘触发）：

int EventDispatcher::AddConsumer(SocketId socket_id, int fd) {
    if (_epfd < 0) {
        errno = EINVAL;
        return -1;
    }
    epoll_event evt;
    evt.events = EPOLLIN | EPOLLET;
    evt.data.u64 = socket_id;
    return epoll_ctl(_epfd, EPOLL_CTL_ADD, fd, &evt);
}

注意这里将evt.data绑定到socket_id上

文章知识点与官方知识档案匹配，可进一步学习相关知识
网络技能树首页概览32550 人正在系统学习中
————————————————
版权声明：本文为CSDN博主「saddlesad」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/saddlesad/article/details/123481856