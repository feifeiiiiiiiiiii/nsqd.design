# Channel #

# 数据结构 #

定义

    struct Channel {
        requeueCount uint64
        messageCount uint64
        timeoutCount uint64

        sync.RWMutex

        topicName string
        name      string
        ctx       *context

        backend BackendQueue

        memoryMsgChan chan *Message
        exitFlag      int32
        exitMutex     sync.RWMutex

        // state tracking
        clients        map[int64]Consumer
        paused         int32
        ephemeral      bool
        deleteCallback func(*Channel)
        deleter        sync.Once

        // Stats tracking
        e2eProcessingLatencyStream *quantile.Quantile

        // TODO: these can be DRYd up
        deferredMessages map[MessageID]*pqueue.Item
        deferredPQ       pqueue.PriorityQueue
        deferredMutex    sync.Mutex
        inFlightMessages map[MessageID]*Message
        inFlightPQ       inFlightPqueue
        inFlightMutex    sync.Mutex
    }