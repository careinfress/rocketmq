# RocketMq消息存储源码跟踪

我们可以从RocketMq源码中测试消息落地的测试类开始进去

~~~java
	//org.apache.rocketmq.store.DefaultMessageStoreTest#init
	@Before
    public void init() throws Exception {
        StoreHost = new InetSocketAddress(InetAddress.getLocalHost(), 8123);
        BornHost = new InetSocketAddress(InetAddress.getByName("127.0.0.1"), 0);

        messageStore = buildMessageStore();
        boolean load = messageStore.load();
        assertTrue(load);
        messageStore.start();
    }

private MessageStore buildMessageStore() throws Exception {
        MessageStoreConfig messageStoreConfig = new MessageStoreConfig();
        messageStoreConfig.setMapedFileSizeCommitLog(1024 * 1024 * 10);
        messageStoreConfig.setMapedFileSizeConsumeQueue(1024 * 1024 * 10);
        messageStoreConfig.setMaxHashSlotNum(10000);
        messageStoreConfig.setMaxIndexNum(100 * 100);
        messageStoreConfig.setFlushDiskType(FlushDiskType.SYNC_FLUSH);
        messageStoreConfig.setFlushIntervalConsumeQueue(1);
        return new DefaultMessageStore(messageStoreConfig, new BrokerStatsManager("simpleTest"), new MyMessageArrivingListener(), new BrokerConfig());
    }
~~~

可以看到，这个测试类在进行真正的测试样例的之前需要进行一些初始化操作，比如关于Broker这一块的参数等等。

~~~java
	//org.apache.rocketmq.store.DefaultMessageStoreTest#testWriteAndRead
	@Test
    public void testWriteAndRead() {
        long totalMsgs = 10;
        QUEUE_TOTAL = 1;
        MessageBody = StoreMessage.getBytes();
        for (long i = 0; i < totalMsgs; i++) {
            messageStore.putMessage(buildMessage());//入口位置
        }

        for (long i = 0; i < totalMsgs; i++) {
            GetMessageResult result = messageStore.getMessage("GROUP_A", "TOPIC_A", 0, i, 1024 * 1024, null);
            assertThat(result).isNotNull();
            result.release();
        }
        verifyThatMasterIsFunctional(totalMsgs, messageStore);
    }

private MessageExtBrokerInner buildMessage() {
        MessageExtBrokerInner msg = new MessageExtBrokerInner();
        msg.setTopic("FooBar");
        msg.setTags("TAG1");
        msg.setKeys("Hello");
        msg.setBody(MessageBody);
        msg.setKeys(String.valueOf(System.currentTimeMillis()));
        msg.setQueueId(Math.abs(QueueId.getAndIncrement()) % QUEUE_TOTAL);
        msg.setSysFlag(0);
        msg.setBornTimestamp(System.currentTimeMillis());
        msg.setStoreHost(StoreHost);
        msg.setBornHost(BornHost);
        return msg;
    }
~~~

构建好一条存储的MessageExtBrokerInner，然后调用messageStore.putMessage方法进行存储。我们进入到putMessage方法中

~~~java
//org.apache.rocketmq.store.DefaultMessageStore#putMessage
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
        if (this.shutdown) {
            log.warn("message store has shutdown, so putMessage is forbidden");
            return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
        }
		//Slave Broker 不处理客户端发送的消息
        if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
            long value = this.printTimes.getAndIncrement();
            if ((value % 50000) == 0) {
                log.warn("message store is slave mode, so putMessage is forbidden ");
            }

            return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
        }
		//Broker不可写时，也不处理客户端发送的消息
        if (!this.runningFlags.isWriteable()) {
            long value = this.printTimes.getAndIncrement();
            if ((value % 50000) == 0) {
                log.warn("message store is not writeable, so putMessage is forbidden " + this.runningFlags.getFlagBits());
            }

            return new PutMessageResult(PutMessageStatus.SERVICE_NOT_AVAILABLE, null);
        } else {
            this.printTimes.set(0);
        }
		//Topic长度限制
        if (msg.getTopic().length() > Byte.MAX_VALUE) {
            log.warn("putMessage message topic length too long " + msg.getTopic().length());
            return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, null);
        }
		//Message的扩展属性的长度限制
        if (msg.getPropertiesString() != null && msg.getPropertiesString().length() > Short.MAX_VALUE) {
            log.warn("putMessage message properties length too long " + msg.getPropertiesString().length());
            return new PutMessageResult(PutMessageStatus.PROPERTIES_SIZE_EXCEEDED, null);
        }
		//如果Messsage刷新OSPageCache耗时太长的话，broker降级
        if (this.isOSPageCacheBusy()) {
            return new PutMessageResult(PutMessageStatus.OS_PAGECACHE_BUSY, null);
        }
		//从业务层到逻辑层，putMessage时咱们存储的核心过程
        long beginTime = this.getSystemClock().now();
        PutMessageResult result = this.commitLog.putMessage(msg);

        long eclipseTime = this.getSystemClock().now() - beginTime;
        if (eclipseTime > 500) {
            log.warn("putMessage not in lock eclipse time(ms)={}, bodyLength={}", eclipseTime, msg.getBody().length);
        }
        this.storeStatsService.setPutMessageEntireTimeMax(eclipseTime);

        if (null == result || !result.isOk()) {
            this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
        }

        return result;
    }
~~~

从注释中我们看到，DefaultMessageStore的putMessage方法基本都是在进行一些前序的校验工作，我们进去到

this.commitLog.putMessage(msg);这个方法。

~~~java
//org.apache.rocketmq.store.CommitLog#putMessage
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
        // Set the storage time 设置储存时间
        msg.setStoreTimestamp(System.currentTimeMillis());
        // Set the message body BODY CRC (consider the most appropriate setting
        // on the client)
    	// Message body的CRC校验码，防止消息内容被篡改和破坏
        msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
        // Back to Results
        AppendMessageResult result = null;
    
		// 存储耗时相关的Metric，可以统计这些指标信息，上报给监控系统
        StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

        String topic = msg.getTopic();
        int queueId = msg.getQueueId();

    	// 处理延迟消息（定时消息）
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
		
    	// 获取最近一个CommitLog文件的内存映射文件（零拷贝）
    	// mappedFileQueue管理这些连续的CommitLog文件
   		// MappedFile和MappedFileQueue后面再分享，大家先可以理解为一个高性能的磁盘IO接口
        putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
        try {
            long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
            this.beginTimeInLock = beginLockTimestamp;

            // Here settings are stored timestamp, in order to ensure an orderly
            // global
            // 拿到锁之后，在设置一次，这样可以保证全局有序
            msg.setStoreTimestamp(beginLockTimestamp);

            // 最近的CommitLog文件写满时，在创建一个
            if (null == mappedFile || mappedFile.isFull()) {
                mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
            }
            if (null == mappedFile) {
                log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
            }
			//把Broker内部的这个Message刷盘到MappedFile的内存，还没有刷盘
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
    
		// 刷盘（同步刷盘或者异步刷盘）
        handleDiskFlush(result, putMessageResult, msg);
    	// 主从同步
        handleHA(result, putMessageResult, msg);

        return putMessageResult;
    }
~~~

在CommitLog#putMessage方法中我们看到了message的进一步处理，首先是设置一些参数，比如存储事件跟CRC，开启监控服务，然后申请一把锁，等拿到锁之后，判断最近的CommitLog是否存在或者文件已经被写满，如果是的话就再重新创建一个新的，然后开启写到这个mappedFile这个内存文件中，最后开启刷盘和同步。

我们进入到刷盘的代码中

~~~java
//org.apache.rocketmq.store.CommitLog#handleDiskFlush
public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
        // Synchronization flush
    	// 同步刷盘
        if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
            // 如果客户端要确定刷盘是否成功 
            if (messageExt.isWaitStoreMsgOK()) {
                // nextOffset:当前内存的写位置 + 本次要写入的字节数
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                service.putRequest(request);
                boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                if (!flushOK) {
                    log.error("do groupcommit, wait for flush failed, topic: " + messageExt.getTopic() + " tags: " + messageExt.getTags()
                        + " client address: " + messageExt.getBornHostString());
                    putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
                }
            } else {
                // broker虽然配置成同步刷盘，但客户端不关心是否已经刷盘成功
                // 同步刷盘策略退化成了异步刷盘策略
                service.wakeup();
            }
        }
        // Asynchronous flush
        else {
            if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                flushCommitLogService.wakeup();
            } else {
                commitLogService.wakeup();
            }
        }
    }

	//org.apache.rocketmq.store.CommitLog.GroupCommitRequest
	public static class GroupCommitRequest {
        //MappedFile文件的缓存区的位置
        private final long nextOffset;
        //等待磁盘是否完成的标志
        private final CountDownLatch countDownLatch = new CountDownLatch(1);
        //刷盘是否成功
        private volatile boolean flushOK = false;

        public GroupCommitRequest(long nextOffset) {
            this.nextOffset = nextOffset;
        }

        public long getNextOffset() {
            return nextOffset;
        }

        public void wakeupCustomer(final boolean flushOK) {
            this.flushOK = flushOK;
            this.countDownLatch.countDown();
        }

        public boolean waitForFlush(long timeout) {
            try {
                this.countDownLatch.await(timeout, TimeUnit.MILLISECONDS);
                return this.flushOK;
            } catch (InterruptedException e) {
                log.error("Interrupted", e);
                return false;
            }
        }
    }

//org.apache.rocketmq.store.CommitLog.GroupCommitService#putRequest
public synchronized void putRequest(final GroupCommitRequest request) {
    		private volatile List<GroupCommitRequest> requestsWrite = new ArrayList<GroupCommitRequest>();
            synchronized (this.requestsWrite) {
                this.requestsWrite.add(request);
            }
            if (hasNotified.compareAndSet(false, true)) {
                // 如果通知已经传达了，但是工作线程还没有处理结束吗，新的请求不用通知了
                // 如果工作线程在等待，就提前唤醒工作线程
                waitPoint.countDown(); // notify
            }
        }
~~~

