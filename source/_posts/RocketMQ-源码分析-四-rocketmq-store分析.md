---
title: RocketMQ 源码分析(四) ----rocketmq-store分析
tags:
  - RocketMQ
  - rpc
categories:
  - 中间件
  - 源码
abbrlink: 5909dfe9
date: 2018-04-02 18:42:19
---

## 存储层

rocketmq-store主要提供了消息存储及管理。主要针对CommitLog、ConsumeQueue的相关操作。 ConsumeQueue是消息的逻辑队列，队列的每一个元素是一个20字节的定长的数据，元素结构有三部分构成：commitLogOffset（8byte,消息所在CommitLog的实际偏移量），size(4byte,消息的大小),hashTag(8byte,消息Tag的哈希值)）。 实际上ConsumeQueue中存储的只是消息所在CommitLog的偏移量，可以理解为是CommitLog的索引文件。

### 主要功能

- 提供保存消息至CommitLog并持久化到物理文件(刷盘方式：同步、异步)

- 维护Topic与ConsumeQueue的关系

- 维护内存映射文件及队列（MapedFile、MapedFileQueue）

- 提供broker主从同步功能（同步双写、异步复制）

- 提供recover数据恢复功能（正常恢复、异常恢复（OSCRASH或者JVM CRASH或者机器掉电等））

- 提供数据索引服务

### BrokerRole

```java
public enum BrokerRole {
    ASYNC_MASTER,
    SYNC_MASTER,
    SLAVE;
}
```

### FlushDiskType

```java
public enum FlushDiskType {
    SYNC_FLUSH,//同步刷盘
    ASYNC_FLUSH//异步刷盘
}
```

### CommitLog

```java
// Message's MAGIC CODE daa320a7
//魔数
public final static int MESSAGE_MAGIC_CODE = 0xAABBCCDD ^ 1880681586 + 8;
// End of file empty MAGIC CODE cbd43194
private final static int BLANK_MAGIC_CODE = 0xBBCCDDEE ^ 1880681586 + 8;
//消息文件队列，包含所有保存在磁盘上的文件
private final MappedFileQueue mappedFileQueue;
//默认消息存储类对象
private final DefaultMessageStore defaultMessageStore;
//刷盘操作服务类
private final FlushCommitLogService flushCommitLogService;

//If TransientStorePool enabled, we must flush message to FileChannel at fixed periods
private final FlushCommitLogService commitLogService;
//添加消息的回调，在doAppend方法中追加消息到内存
private final AppendMessageCallback appendMessageCallback;
private final ThreadLocal<MessageExtBatchEncoder> batchEncoderThreadLocal;
//Topic与每个队列的偏移量关系
private HashMap<String/* topic-queueid */, Long/* offset */> topicQueueTable = new HashMap<String, Long>(1024);
private volatile long confirmOffset = -1L;

private volatile long beginTimeInLock = 0;
private final PutMessageLock putMessageLock;
```

putMessage

```java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
    // Set the storage time
    msg.setStoreTimestamp(System.currentTimeMillis());
    // Set the message body BODY CRC (consider the most appropriate setting
    // on the client)
    msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
    // Back to Results
    AppendMessageResult result = null;

    StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

    String topic = msg.getTopic();
    int queueId = msg.getQueueId();

    final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
    if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
        || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
        // Delay Delivery
        if (msg.getDelayTimeLevel() > 0) {
            if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
                msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
            }

            topic = ScheduleMessageService.SCHEDULE_TOPIC;
            queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

            // Backup real topic, queueId
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
            msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

            msg.setTopic(topic);
            msg.setQueueId(queueId);
        }
    }

    long eclipseTimeInLock = 0;
    MappedFile unlockMappedFile = null;
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

    putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
    try {
        long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
        this.beginTimeInLock = beginLockTimestamp;

        // Here settings are stored timestamp, in order to ensure an orderly
        // global
        msg.setStoreTimestamp(beginLockTimestamp);

        if (null == mappedFile || mappedFile.isFull()) {
            mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
        }
        if (null == mappedFile) {
            log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
        }

        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        switch (result.getStatus()) {
            case PUT_OK:
                break;
            case END_OF_FILE:
                unlockMappedFile = mappedFile;
                // Create a new file, re-write the message
                mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                if (null == mappedFile) {
                    // XXX: warn and notify me
                    log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
                }
                result = mappedFile.appendMessage(msg, this.appendMessageCallback);
                break;
            case MESSAGE_SIZE_EXCEEDED:
            case PROPERTIES_SIZE_EXCEEDED:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
            case UNKNOWN_ERROR:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
            default:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
        }

        eclipseTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
        beginTimeInLock = 0;
    } finally {
        putMessageLock.unlock();
    }

    if (eclipseTimeInLock > 500) {
        log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", eclipseTimeInLock, msg.getBody().length, result);
    }

    if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
        this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
    }

    PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

    // Statistics
    storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();
    storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());

    handleDiskFlush(result, putMessageResult, msg);
    handleHA(result, putMessageResult, msg);

    return putMessageResult;
}
```

- ConsumeQueue

```java
public static final int CQ_STORE_UNIT_SIZE = 20;
private final DefaultMessageStore defaultMessageStore;
//消息文件队列，包含所有保存在磁盘上的文件
private final MappedFileQueue mappedFileQueue;
//Topic名称
private final String topic;
//队列ID
private final int queueId;
//内存中索引位置
private final ByteBuffer byteBufferIndex;
//存储的位置
private final String storePath;
//映射文件的大小
private final int mappedFileSize;
//CommitLog最大的偏移
private long maxPhysicOffset = -1;
//CommitLog最小的偏移
private volatile long minLogicOffset = 0;
private ConsumeQueueExt consumeQueueExt = null;
```

- MappedFileQueue

```java
private static final int DELETE_FILES_BATCH_MAX = 10;
//存储的位置
private final String storePath;
//映射文件的大小
private final int mappedFileSize;

private final CopyOnWriteArrayList<MappedFile> mappedFiles = new CopyOnWriteArrayList<MappedFile>();
//预创建MapedFile服务，MapedFile
private final AllocateMappedFileService allocateMappedFileService;

private long flushedWhere = 0;
//已刷盘位置
private long committedWhere = 0;
//最后刷盘完成的时间
private volatile long storeTimestamp = 0;
```

- MappedFile

```java
//系统每页缓存大小为4K
public static final int OS_PAGE_SIZE = 1024 * 4;
private static final AtomicLong TOTAL_MAPPED_VIRTUAL_MEMORY = new AtomicLong(0);
//总共映射虚拟内存的大小
private static final AtomicInteger TOTAL_MAPPED_FILES = new AtomicInteger(0);
//文件开始写的位置
protected final AtomicInteger wrotePosition = new AtomicInteger(0);
//文件已经刷盘的位置
protected final AtomicInteger committedPosition = new AtomicInteger(0);
private final AtomicInteger flushedPosition = new AtomicInteger(0);
//文件大小
protected int fileSize;
protected FileChannel fileChannel;
/**
 * Message will put to here first, and then reput to FileChannel if writeBuffer is not null.
 */
protected ByteBuffer writeBuffer = null;
protected TransientStorePool transientStorePool = null;
//文件名称
private String fileName;
//文件的全局offset，也就是文件名的前缀
private long fileFromOffset;
//文件对象
private File file;
//文件映射的内存
private MappedByteBuffer mappedByteBuffer;
//最后刷盘完成的时间
private volatile long storeTimestamp = 0;
private boolean firstCreateInQueue = false;
```



接口MessageStore

```java
public interface MessageStore {

    //加载之前存储的消息
    boolean load();

    /**
     * Launch this message store.
     *
     * @throws Exception if there is any error.
     */
    void start() throws Exception;

    /**
     * Shutdown this message store.
     */
    void shutdown();

    /**
     * Destroy this message store. Generally, all persistent files should be removed after invocation.
     */
    void destroy();

    /**
     * Store a message into store.
     *
     * @param msg Message instance to store
     * @return result of store operation.
     */
    PutMessageResult putMessage(final MessageExtBrokerInner msg);

    /**
     * Store a batch of messages.
     *
     * @param messageExtBatch Message batch.
     * @return result of storing batch messages.
     */
    PutMessageResult putMessages(final MessageExtBatch messageExtBatch);

    /**
     * Query at most <code>maxMsgNums</code> messages belonging to <code>topic</code> at <code>queueId</code> starting
     * from given <code>offset</code>. Resulting messages will further be screened using provided message filter.
     *
     * @param group Consumer group that launches this query.
     * @param topic Topic to query.
     * @param queueId Queue ID to query.
     * @param offset Logical offset to start from.
     * @param maxMsgNums Maximum count of messages to query.
     * @param messageFilter Message filter used to screen desired messages.
     * @return Matched messages.
     */
    GetMessageResult getMessage(final String group, final String topic, final int queueId,
        final long offset, final int maxMsgNums, final MessageFilter messageFilter);

    /**
     * Get maximum offset of the topic queue.
     *
     * @param topic Topic name.
     * @param queueId Queue ID.
     * @return Maximum offset at present.
     */
    long getMaxOffsetInQueue(final String topic, final int queueId);

    /**
     * Get the minimum offset of the topic queue.
     *
     * @param topic Topic name.
     * @param queueId Queue ID.
     * @return Minimum offset at present.
     */
    long getMinOffsetInQueue(final String topic, final int queueId);

    /**
     * Get the offset of the message in the commit log, which is also known as physical offset.
     *
     * @param topic Topic of the message to lookup.
     * @param queueId Queue ID.
     * @param consumeQueueOffset offset of consume queue.
     * @return physical offset.
     */
    long getCommitLogOffsetInQueue(final String topic, final int queueId, final long consumeQueueOffset);

    /**
     * Look up the physical offset of the message whose store timestamp is as specified.
     *
     * @param topic Topic of the message.
     * @param queueId Queue ID.
     * @param timestamp Timestamp to look up.
     * @return physical offset which matches.
     */
    long getOffsetInQueueByTime(final String topic, final int queueId, final long timestamp);

    /**
     * Look up the message by given commit log offset.
     *
     * @param commitLogOffset physical offset.
     * @return Message whose physical offset is as specified.
     */
    MessageExt lookMessageByOffset(final long commitLogOffset);

    /**
     * Get one message from the specified commit log offset.
     *
     * @param commitLogOffset commit log offset.
     * @return wrapped result of the message.
     */
    SelectMappedBufferResult selectOneMessageByOffset(final long commitLogOffset);

    /**
     * Get one message from the specified commit log offset.
     *
     * @param commitLogOffset commit log offset.
     * @param msgSize message size.
     * @return wrapped result of the message.
     */
    SelectMappedBufferResult selectOneMessageByOffset(final long commitLogOffset, final int msgSize);

    /**
     * Get the running information of this store.
     *
     * @return message store running info.
     */
    String getRunningDataInfo();

    /**
     * Message store runtime information, which should generally contains various statistical information.
     *
     * @return runtime information of the message store in format of key-value pairs.
     */
    HashMap<String, String> getRuntimeInfo();

    /**
     * Get the maximum commit log offset.
     *
     * @return maximum commit log offset.
     */
    long getMaxPhyOffset();

    /**
     * Get the minimum commit log offset.
     *
     * @return minimum commit log offset.
     */
    long getMinPhyOffset();

    /**
     * Get the store time of the earliest message in the given queue.
     *
     * @param topic Topic of the messages to query.
     * @param queueId Queue ID to find.
     * @return store time of the earliest message.
     */
    long getEarliestMessageTime(final String topic, final int queueId);

    /**
     * Get the store time of the earliest message in this store.
     *
     * @return timestamp of the earliest message in this store.
     */
    long getEarliestMessageTime();

    /**
     * Get the store time of the message specified.
     *
     * @param topic message topic.
     * @param queueId queue ID.
     * @param consumeQueueOffset consume queue offset.
     * @return store timestamp of the message.
     */
    long getMessageStoreTimeStamp(final String topic, final int queueId, final long consumeQueueOffset);

    /**
     * Get the total number of the messages in the specified queue.
     *
     * @param topic Topic
     * @param queueId Queue ID.
     * @return total number.
     */
    long getMessageTotalInQueue(final String topic, final int queueId);

    /**
     * Get the raw commit log data starting from the given offset, which should used for replication purpose.
     *
     * @param offset starting offset.
     * @return commit log data.
     */
    SelectMappedBufferResult getCommitLogData(final long offset);

    /**
     * Append data to commit log.
     *
     * @param startOffset starting offset.
     * @param data data to append.
     * @return true if success; false otherwise.
     */
    boolean appendToCommitLog(final long startOffset, final byte[] data);

    /**
     * Execute file deletion manually.
     */
    void executeDeleteFilesManually();

    /**
     * Query messages by given key.
     *
     * @param topic topic of the message.
     * @param key message key.
     * @param maxNum maximum number of the messages possible.
     * @param begin begin timestamp.
     * @param end end timestamp.
     */
    QueryMessageResult queryMessage(final String topic, final String key, final int maxNum, final long begin,
        final long end);

    /**
     * Update HA master address.
     *
     * @param newAddr new address.
     */
    void updateHaMasterAddress(final String newAddr);

    /**
     * Return how much the slave falls behind.
     *
     * @return number of bytes that slave falls behind.
     */
    long slaveFallBehindMuch();

    /**
     * Return the current timestamp of the store.
     *
     * @return current time in milliseconds since 1970-01-01.
     */
    long now();

    /**
     * Clean unused topics.
     *
     * @param topics all valid topics.
     * @return number of the topics deleted.
     */
    int cleanUnusedTopic(final Set<String> topics);

    /**
     * Clean expired consume queues.
     */
    void cleanExpiredConsumerQueue();

    /**
     * Check if the given message has been swapped out of the memory.
     *
     * @param topic topic.
     * @param queueId queue ID.
     * @param consumeOffset consume queue offset.
     * @return true if the message is no longer in memory; false otherwise.
     */
    boolean checkInDiskByConsumeOffset(final String topic, final int queueId, long consumeOffset);

    /**
     * Get number of the bytes that have been stored in commit log and not yet dispatched to consume queue.
     *
     * @return number of the bytes to dispatch.
     */
    long dispatchBehindBytes();

    /**
     * Flush the message store to persist all data.
     *
     * @return maximum offset flushed to persistent storage device.
     */
    long flush();

    /**
     * Reset written offset.
     *
     * @param phyOffset new offset.
     * @return true if success; false otherwise.
     */
    boolean resetWriteOffset(long phyOffset);

    /**
     * Get confirm offset.
     *
     * @return confirm offset.
     */
    long getConfirmOffset();

    /**
     * Set confirm offset.
     *
     * @param phyOffset confirm offset to set.
     */
    void setConfirmOffset(long phyOffset);

    /**
     * Check if the operation system page cache is busy or not.
     *
     * @return true if the OS page cache is busy; false otherwise.
     */
    boolean isOSPageCacheBusy();

    /**
     * Get lock time in milliseconds of the store by far.
     *
     * @return lock time in milliseconds.
     */
    long lockTimeMills();

    /**
     * Check if the transient store pool is deficient.
     *
     * @return true if the transient store pool is running out; false otherwise.
     */
    boolean isTransientStorePoolDeficient();

    /**
     * Get the dispatcher list.
     *
     * @return list of the dispatcher.
     */
    LinkedList<CommitLogDispatcher> getDispatcherList();

    /**
     * Get consume queue of the topic/queue.
     *
     * @param topic Topic.
     * @param queueId Queue ID.
     * @return Consume queue.
     */
    ConsumeQueue getConsumeQueue(String topic, int queueId);
}
```





#### DefaultMessageStore

