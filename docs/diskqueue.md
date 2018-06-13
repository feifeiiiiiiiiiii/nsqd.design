# 消息持久化 #

# 数据结构 #

定义
    
    struct diskQueue {
        readPos      int64      //  当前读取文件的位置
        writePos     int64      //  当前写入文件文件位置
        readFileNum  int64      //  当前正在读取文件编号
        writeFileNum int64      //  当前正在写入文件的编号
        depth        int64      //  当前队列已有消息的总量

        name            string  //  队列名称
        dataPath        string  //  队列持久化存储位置
        maxBytesPerFile int64   //  消息存储文件的最大存储字节
        minMsgSize      int32   //  消息最小字节数
        maxMsgSize      int32   //  消息最大字节数
        syncEvery       int64         // 消息写入的数量需要fsync
        syncTimeout     time.Duration // 定时fsync
        exitFlag        int32         // 标志位
        needSync        bool          // 同步标志位

        nextReadPos     int64         // 下一个待读取文件的位置
	    nextReadFileNum               // 下一个待读取文件编号
        
        readFile  *os.File            // 当前正在读取文件的句柄
        writeFile *os.File            // 当前正在写入文件的句柄
        reader    *bufio.Reader       // 
        writeBuf  bytes.Buffer        // 写入缓冲区

        readChan chan []byte          // 消息读取channel

        writeChan         chan []byte   // 消息写入的channel
        writeResponseChan chan error    
        emptyChan         chan int
        emptyResponseChan chan error
        exitChan          chan int
        exitSyncChan      chan int

        logf AppLogFunc
    }


# 存储结构 #

元数据存储

    元数据记录五个信息:
        depth、readFileNum, readPos, writeFileNum, writePos, 具体意义看上面

    文件命名:
        metaName = name + ".diskqueue.meta.dat"

    存储格式:
        depth + "\n" +
        readFileNum + "," + readPos + "," + "\n" + 
        writeFileNum + "," + writePos + "," + "\n"


数据存储