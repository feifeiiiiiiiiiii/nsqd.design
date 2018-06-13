# 消息持久化 #

源码地址

    [go-diskqueu源码地址](https://github.com/nsqio/go-diskqueue)

    自己学习用[c版本](https://github.com/feifeiiiiiiiiiii/cnsq/blob/master/src/diskqueue/)

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
        emptyChan         chan int      // 清空队列 对应接口 Empty()
        emptyResponseChan chan error    
        exitChan          chan int      // 关闭 对应接口是 Close
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
    
    数据存储的内容:
        dataLen(消息的大小,转换大端存储),消息内容(data)

    文件命名:
        name = dataPath + name + ".diskqueue.%06d.dat" + fileNum

    存储格式:
        BigEndian(dataLen)+Binary(data)

# 对外提供的接口 #

API列表

    New(...)                // 创建diskQueue实例

    Put([]byte) error       // 队列中放入消息

	ReadChan() chan []byte  // 读取消息
	
    Close() error           // 关闭diskQueue
	
    Delete() error          // 删除队列
	
    Depth() int64           // 获取当前队列的消息数量
	
    Empty() error           // 清空队列

# 私有接口 #

API接口

    exit(deleted bool)      // 清空和关闭调用

    deleteAllFiles()        // 删除队列所有文件

    skipToNextRWFile()      // 确定下一个要读写的文件的位置和编号

    readOne()               // 读取消息

    writeOne()              // 写入消息

    sync()                  // flush数据

    retrieveMetaData()      // 获取meta数据

    persistMetaData()       // 持久化数据

    metaDataFileName()      // 获取meta文件名

    fileName(fileNum int64) // 获取文件编号名字

    checkTailCorruption()   // 检查是否正常

    moveForward()           // 读取下个消息

    handleReadError()       // 处理读取消息错误

    ioLoop()                // 核心函数


# 处理流程 #

创建队列

    调用 New(...) 创建队列
        
        1. 创建 diskQueue实例 (d)

        2. 调用函数 `retrieveMetaData` 读取元数据信息
        
        3. 生成一个协程 处理函数是 `ioLoop`, diskQueue最核心的处理

        4. ioLoop 主要逻辑

            var dataRead []byte     // 存储消息
            var err error
            var count int64         // 操作(写入、读取)消息的计数器 和 syncEvery 一起决定是否需要flush磁盘的脏数据
            var r chan []byte       // 用来操作读取消息的chan,所有读取的消息都用chan（管道）的send出去

            syncTicker := time.NewTicker(d.syncTimeout)     // 创建一个syncTimeout的定时器，用来flush磁盘的脏数据

            for {
                if count == d.syncEvery {
                    d.needSync = true
                }

                if d.needSync {
                    d.sync() // flush磁盘脏数据
                    count = 0 // 计数器归零
                }

                // 读取消息 判断消息是否可读
                if (d.readFileNum < d.writeFileNum) || (d.readPos < d.writePos) {
                    if d.nextReadPos == d.readPos {
                        dataRead, err = d.readOne() // 读取一条消息
                        if err != nil {
                            d.handleReadError() // 出现读取异常的处理函数
                            continue
                        }
                    }
                    r = d.readChan
                } else {
                    r = nil
                }

                select {
                case r <- dataRead:
                    count++
                    d.moveForward()
                case <-d.emptyChan:
                    // 清空消息的指令
                    d.emptyResponseChan <- d.deleteAllFiles()
                    count = 0
                case dataWrite := <-d.writeChan:
                    // 处理写入消息的指令
                    count++
                    d.writeResponseChan <- d.writeOne(dataWrite)
                case <-syncTicker.C:
                    if count == 0 {
                        // avoid sync when there's no activity
                        continue
                    }
                    d.needSync = true
                case <-d.exitChan:
                    goto exit
                }
            }

        Tips:
            ioLoop实现了个CSP(Communicating Sequential Processes)并发模型，所有的操作都是基于chan通讯的,其实nsqd将go的chan玩的最溜了

读取消息

    更简单，直接从 readChan 中读取，有数据则读取 无数据则阻塞直到有数据可读

写入消息

    很简单 就是 把待写入的消息放入 writeChan 中, 写入的处理逻辑在 ioLoop 中

