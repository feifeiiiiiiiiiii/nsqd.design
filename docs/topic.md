# topic #

源文件 nsqd/topic.go

# 数据结构 #

定义

    struct Topic {
        messageCount    int64

        sync.RWMutex

        name            string                  // topic的名称

        channelMap      map[string]*Channel     // 订阅topic的所有channel实例
        backend         BackendQueue            // [磁盘队列](/docs/diskqueue.md)
        memoryMsgChan   chan *Message           // 内存队列

        exitChan        chan int
        channelUpdateChan   chan int
        waitGroup       util.WaitGroupWrapper
        exitFlag        int32
        idFactory       *guidFactory            // 生成消息的唯一id

        ephemeral       bool                    // topic是临时的,不会写入磁盘,仅保留在内存中
        deleteCallback  func(*Topic)
        deleter         sync.Once

        paused          int32
        pauseChan       chan bool

        ctx             *context                // 存储上下文， 在topic中存储的是NSQD实例
    }

# 对外提供的接口 #

主要API列表

    NewTopic(...)           // 创建一个topic

    GetChannel(channelName string)            // 获取channelName实例 没有则创建 并且会把新增的channel用channelUpdateChan(messagePump函数更新channel列表)通知更新

    PutMessages(...)        // 批量写入消息 

    PutMessage(...)         // 单条消息写入

    GetExistingChannel      // 获取已经存在的channel实例

    Depth()                 // 当前topic消息数量 内存+磁盘


# 处理流程 #

核心的处理流程是(nsqd/topic.go)messagePump处理

    这个函数主要做以下几件事

        for {
            select {
            // 内存和磁盘读取消息 进行解码
            case msg = <-memoryMsgChan:
            case buf = <-backendChan:
                msg, err = decodeMessage(buf)
                if err != nil {
                    t.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
                    continue
                }

            // 如果新增订阅channel则更新chans列表
            case <-t.channelUpdateChan:
                chans = chans[:0]
                t.RLock()
                for _, c := range t.channelMap {
                    chans = append(chans, c)
                }
                t.RUnlock()
                if len(chans) == 0 || t.IsPaused() {
                    memoryMsgChan = nil
                    backendChan = nil
                } else {
                    memoryMsgChan = t.memoryMsgChan
                    backendChan = t.backend.ReadChan()
                }
                continue
            
            // 暂停接受消息
            case pause := <-t.pauseChan:
                if pause || len(chans) == 0 {
                    memoryMsgChan = nil
                    backendChan = nil
                } else {
                    memoryMsgChan = t.memoryMsgChan
                    backendChan = t.backend.ReadChan()
                }
                continue
            case <-t.exitChan:
                goto exit
            }

            // 将消息放入每个channel中进行处理
            for i, channel := range chans {
                chanMsg := msg
                if i > 0 {
                    chanMsg = NewMessage(msg.ID, msg.Body)
                    chanMsg.Timestamp = msg.Timestamp
                    chanMsg.deferred = msg.deferred
                }
                if chanMsg.deferred != 0 {
                    // 处理延迟消息
                    channel.PutMessageDeferred(chanMsg, chanMsg.deferred)
                    continue
                }
                // 处理普通消息
                err := channel.PutMessage(chanMsg)
                if err != nil {
                    t.ctx.nsqd.logf(LOG_ERROR,
                        "TOPIC(%s) ERROR: failed to put msg(%s) to channel(%s) - %s",
                        t.name, msg.ID, channel.name, err)
                }
            }
	    }



