# nqsd #

    消息队列核心的服务，可以单机运行

# 名词解释 #

1.[topic](/docs/topic.md) 主题
    
    用来发布消息

2.[channel](/docs/channel.md) 频道
    
    用来订阅 topic 的消息 同一个topic的消息会广播给改topic下的所有channel
    这里也能看出来topic和channel的关系

                    |-------- channel-1
            topic------------ channel-2
                    |--------- ......
                    |-------- channel-n

3.message 消息
    https://nsq.io/clients/tcp_protocol_spec.html (Data Format)

    消息格式

    数据格式

        [x][x][x][x][x][x][x][x][x][x][x][x]...
        |  (int32) ||  (int32) || (binary)
        |  4-byte  ||  4-byte  || N-byte
        ------------------------------------...
            size     frame type     data

        FrameTypeResponse int32 = 0
        FrameTypeError    int32 = 1
        FrameTypeMessage  int32 = 2

    完整消息格式

        [x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x]...
        |       (int64)        ||    ||      (hex string encoded in ASCII)           || (binary)
        |       8-byte         ||    ||                 16-byte                      || N-byte
        ------------------------------------------------------------------------------------------...
        nanosecond timestamp    ^^                   message ID                       message body
                            (uint16)
                                2-byte
                            attempts

# TCP协议 # 

只做SUB/PUB/FIN协议逻辑 

详细见 https://nsq.io/clients/tcp_protocol_spec.html

1.订阅 SUB

    SUB <topic_name> <channel_name>\n

    成功:
        OK
    失败:
        E_INVALID
        E_BAD_TOPIC
        E_BAD_CHANNEL

2.发布 PUB

    PUB <topic_name>\n
    [ 4-byte size in bytes ][ N-byte binary data ]

    成功:
        OK
    失败:
        E_INVALID
        E_BAD_TOPIC
        E_BAD_MESSAGE
        E_PUB_FAILED

3.完成 FIN

    FIN <message_id>\n

    成功:
        无响应
    失败:
        E_INVALID
        E_FIN_FAILED

# HTTP 协议 #

    参见 nsqd/http.go 定义rest api
    

# 数据结构 #

NSQD定义

    struct NSQD {
        clientIDSequence    int64

        sync.RWMutex

        opts                atomic.Value            // NSQD启动的配置参数

        dl                  *dirlock.DirLock        // 文件锁,多个进程同时操作同一份文件的过程中，很容易导致文件中的数据混乱，需要锁操作来保证数据的完整性,window忽略

        isLoading           int32               
        errValue            atomic.Value
        startTime           time.Time

        topicMap            map[string]*Topic       // topic 实例map

        lookupPeers         atomic.Value

        tcpListener         net.Listener            // 提供 tcp 服务, 针对tcp协议
        httpListener        net.Listener            // 提供 http 服务  rest api接口
        httpsListener       net.Listener            // https 服务
        tlsConfig           *tls.Config             // 证书配置

        poolSize            int

        notifyChan          chan interface{}
        optsNotificationChan    chan struct{}
        exitChan            chan int
        waitGroup           util.WaitGroupWrapper

        ci                  *clusterinfo.ClusterInfo    
    }

