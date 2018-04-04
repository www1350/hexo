---
title: RocketMQ 源码分析(二) ----通信模块分析
abbrlink: 58cc55f9
date: 2018-03-19 22:41:42
tags: [RocketMQ,rpc]
categories: [中间件,源码]
---

## rocketMQ通信模块
![image](https://user-images.githubusercontent.com/7789698/32897440-358e21d4-caab-11e7-8f66-76a302229a2d.png)

NettyRemotingClient
```
 @Override
    public void start() {
//业务线程池
        this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
            nettyClientConfig.getClientWorkerThreads(),
            new ThreadFactory() {

                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, "NettyClientWorkerThread_" + this.threadIndex.incrementAndGet());
                }
            });

        Bootstrap handler = this.bootstrap.group(this.eventLoopGroupWorker).channel(NioSocketChannel.class)
            .option(ChannelOption.TCP_NODELAY, true)
            .option(ChannelOption.SO_KEEPALIVE, false)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, nettyClientConfig.getConnectTimeoutMillis())
            .option(ChannelOption.SO_SNDBUF, nettyClientConfig.getClientSocketSndBufSize())
            .option(ChannelOption.SO_RCVBUF, nettyClientConfig.getClientSocketRcvBufSize())
            .handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(
                        defaultEventExecutorGroup,
//编码（RemotingCommand 的Header和Body依次写入）
                        new NettyEncoder(),
//解码（LengthFieldBasedFrameDecoder自定义长度解码器解决TCP粘包）
                        new NettyDecoder(),
//netty 4.X心跳检测 
//readerIdleTime：为读超时时间（即测试端一定时间内未接受到被测试端消息）、writerIdleTime：为写超时时间（即测试端一定时间内向被测试端发送消息）、allIdleTime：所有类型的超时时间 120
                        new IdleStateHandler(0, 0, nettyClientConfig.getClientChannelMaxIdleTimeSeconds()),
//连接管理handler,处理connect, disconnect, close等事件
                        new NettyConnectManageHandler(),
//处理接收到RemotingCommand消息后的事件, 收到服务器端响应后的相关操作
                        new NettyClientHandler());
                }
            });
//每次有消息需要发送, 就会生成resposneFuture用于接收消息回应, 但是如果始终没有收到回应, //Map(scanResponseTable)中的responseFuture就会堆积.
//这个时候就需要一个线程来专门做Map的清理回收,
        this.timer.scheduleAtFixedRate(new TimerTask() {
            @Override
            public void run() {
                try {
                    NettyRemotingClient.this.scanResponseTable();
                } catch (Throwable e) {
                    log.error("scanResponseTable exception", e);
                }
            }
        }, 1000 * 3, 1000);

        if (this.channelEventListener != null) {
            this.nettyEventExecutor.start();
        }
    }
```

```
public void scanResponseTable() {
        final List<ResponseFuture> rfList = new LinkedList<ResponseFuture>();
        Iterator<Entry<Integer, ResponseFuture>> it = this.responseTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<Integer, ResponseFuture> next = it.next();
            ResponseFuture rep = next.getValue();

            if ((rep.getBeginTimestamp() + rep.getTimeoutMillis() + 1000) <= System.currentTimeMillis()) {
                rep.release();
                it.remove();
                rfList.add(rep);
                log.warn("remove timeout request, " + rep);
            }
        }

        for (ResponseFuture rf : rfList) {
            try {
                executeInvokeCallback(rf);
            } catch (Throwable e) {
                log.warn("scanResponseTable, operationComplete Exception", e);
            }
        }
    }
```

## RocketMQ 通信编解码源码解析

编码
```
public class NettyEncoder extends MessageToByteEncoder<RemotingCommand> {
    private static final Logger log = LoggerFactory.getLogger(RemotingHelper.ROCKETMQ_REMOTING);

    @Override
    public void encode(ChannelHandlerContext ctx, RemotingCommand remotingCommand, ByteBuf out)
        throws Exception {
        try {
//nio ByteBuffer 具体见http://www1350.github.io/#post/115
            ByteBuffer header = remotingCommand.encodeHeader();
            out.writeBytes(header);
            byte[] body = remotingCommand.getBody();
            if (body != null) {
                out.writeBytes(body);
            }
        } catch (Exception e) {
            log.error("encode exception, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()), e);
            if (remotingCommand != null) {
                log.error(remotingCommand.toString());
            }
            RemotingUtil.closeChannel(ctx.channel());
        }
    }
}
```


解码
```
public class NettyDecoder extends LengthFieldBasedFrameDecoder {
    private static final Logger log = LoggerFactory.getLogger(RemotingHelper.ROCKETMQ_REMOTING);

    private static final int FRAME_MAX_LENGTH =
        Integer.parseInt(System.getProperty("com.rocketmq.remoting.frameMaxLength", "16777216"));

    public NettyDecoder() {
        super(FRAME_MAX_LENGTH, 0, 4, 0, 4);
    }

    @Override
    public Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        ByteBuf frame = null;
        try {
            frame = (ByteBuf) super.decode(ctx, in);
            if (null == frame) {
                return null;
            }
//netty的ByteBuf 转化为nio 的ByteBuffer
//  http://www.jianshu.com/p/0f93834f23de
            ByteBuffer byteBuffer = frame.nioBuffer();

            return RemotingCommand.decode(byteBuffer);
        } catch (Exception e) {
            log.error("decode exception, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()), e);
            RemotingUtil.closeChannel(ctx.channel());
        } finally {
            if (null != frame) {
                frame.release();
            }
        }

        return null;
    }
}
```
抓一下包：
三次握手
![b79ea69e-e0c3-4400-813b-66d201dab19d](https://user-images.githubusercontent.com/7789698/33132848-be28c574-cfd5-11e7-87d4-0630b2c14a4a.png)

内容
<img width="1253" alt="wx20171122-225830 2x" src="https://user-images.githubusercontent.com/7789698/33133766-c3a83662-cfd8-11e7-8f9a-043d4ceb7682.png">


我们来看看通信的时候编解码的包：
![main](https://user-images.githubusercontent.com/7789698/32903410-e1ca3d8c-cb2f-11e7-85dd-7cabb312482c.jpg)

处理header
```
public ByteBuffer encodeHeader() {
        return encodeHeader(this.body != null ? this.body.length : 0);
    }

    public ByteBuffer encodeHeader(final int bodyLength) {
        // 1> header length size
        int length = 4;

        // 2> header data length
        byte[] headerData;
        headerData = this.headerEncode();

        length += headerData.length;

        // 3> body data length
        length += bodyLength;

        ByteBuffer result = ByteBuffer.allocate(4 + length - bodyLength);

        // length
        result.putInt(length);

        // header length
        result.put(markProtocolType(headerData.length, serializeTypeCurrentRPC));

        // header data
        result.put(headerData);

        result.flip();

        return result;
    }


    public static byte[] markProtocolType(int source, SerializeType type) {
        byte[] result = new byte[4];

        result[0] = type.getCode();
        result[1] = (byte) ((source >> 16) & 0xFF);
        result[2] = (byte) ((source >> 8) & 0xFF);
        result[3] = (byte) (source & 0xFF);
        return result;
    }
```


### header内容：

| 字段名    |          类型           |                        Request                        |          Response          |                               描述 |
| --------- | :---------------------: | :---------------------------------------------------: | :------------------------: | ---------------------------------: |
| code      |          short          |     请求操作码，请求接收方根据不同代码做不同操作      | 应答结果代码，0成功非0失败 |                                    |
| language  |          byte           |               请求发起方语言，默认JAVA                |       应答接收方语言       |                   详见LanguageCode |
| version   |          short          |                    请求发起方版本                     |         应答接收方         | 环境变量 rocketmq.remoting.version |
| opaque    |           int           | 请求发起方在同一连接不同的请求标识符,多线程连接时复用 |   应答方不做处理直接返回   |                      RPC请求的序号 |
| flag      |           int           |                     通信层标志位                      |        通信层标志位        |   区分是普通RPC还是onewayRPC得标志 |
| remark    |         String          |                    传输自定义文本                     |      错误详细描述信息      |                                    |
| extFields | HashMap<String, String> |                    请求自定义字段                     |       应答自定义字段       |             customHeader的每个字段 |


```
 private static final int RPC_TYPE = 0; // 0, REQUEST_COMMAND rpc类型的标注，一种是普通的RPC请求
 private static final int RPC_ONEWAY = 1; // 0, 这种ONEWAY 是指单向RPC,比如心跳包

 private static final Map<Class<? extends CommandCustomHeader>, Field[]> clazzFieldsCache =
        new HashMap<Class<? extends CommandCustomHeader>, Field[]>();//**CommandCustomHader**是所有headerData都要实现的接口，后面的Field[]就是解析header所对应的成员属性，所以这个map就是解析时候的字段缓存，下面两个map也是分别对应类名缓存和注解缓存。
 private static final Map<Class, String> canonicalNameCache = new HashMap<Class, String>();
 // 1, RESPONSE_COMMAND
 private static final Map<Field, Annotation> notNullAnnotationCache = new HashMap<Field, Annotation>();
private static AtomicInteger requestId = new AtomicInteger(0);//这里的requestId是RPC请求的序号，每次请求的时候都会increment一下，同时后面会讲到的responseTable会用这个requestId作为key。   

private int code;//这里的code是用来区分request类型的
private LanguageCode language = LanguageCode.JAVA;//区分语言种类
private int version = 0;//RPC版本号
private int opaque = requestId.getAndIncrement();//这里的opaque就是requestId
private int flag = 0;//区分是普通RPC还是onewayRPC得标志
private String remark;//标注信息
private HashMap<String, String> extFields;//存放本次RPC通信中所有的extFeilds，extFeilds其实就可以理解成本次通信的包头数据
private transient CommandCustomHeader customHeader; //包头数据，注意transient标记，不会被序列化
private transient byte[] body; //body数据，注意transient标记，不会被序列化
```


 ```
   private byte[] headerEncode() {
        this.makeCustomHeaderToNet();
//基于rocketmq还是基于JSON的序列化方式
        if (SerializeType.ROCKETMQ == serializeTypeCurrentRPC) {
            return RocketMQSerializable.rocketMQProtocolEncode(this);
        } else {
            return RemotingSerializable.encode(this);
        }
    }


public void makeCustomHeaderToNet() {
        if (this.customHeader != null) {
            Field[] fields = getClazzFields(customHeader.getClass());
            if (null == this.extFields) {
                this.extFields = new HashMap<String, String>();
            }

            for (Field field : fields) {
                if (!Modifier.isStatic(field.getModifiers())) {
                    String name = field.getName();
                    if (!name.startsWith("this")) {
                        Object value = null;
                        try {
                            field.setAccessible(true);
                            value = field.get(this.customHeader);
                        } catch (Exception e) {
                            log.error("Failed to access field [{}]", name, e);
                        }

                        if (value != null) {
                            this.extFields.put(name, value.toString());
                        }
                    }
                }
            }
        }
    }
 ```

![ba3e2d6c-d79a-4148-9c88-facba76ff0e1](https://user-images.githubusercontent.com/7789698/32938009-b6449d98-cbb5-11e7-848a-cebb2631fc97.png)

### RocketMQ序列化
```
public static byte[] rocketMQProtocolEncode(RemotingCommand cmd) {
        // String remark
        byte[] remarkBytes = null;
        int remarkLen = 0;
        if (cmd.getRemark() != null && cmd.getRemark().length() > 0) {
            remarkBytes = cmd.getRemark().getBytes(CHARSET_UTF8);
            remarkLen = remarkBytes.length;
        }

        // HashMap<String, String> extFields
        byte[] extFieldsBytes = null;
        int extLen = 0;
        if (cmd.getExtFields() != null && !cmd.getExtFields().isEmpty()) {
            extFieldsBytes = mapSerialize(cmd.getExtFields());
            extLen = extFieldsBytes.length;
        }

        int totalLen = calTotalLen(remarkLen, extLen);

        ByteBuffer headerBuffer = ByteBuffer.allocate(totalLen);
        // int code(~32767)
        headerBuffer.putShort((short) cmd.getCode());
        // LanguageCode language
        headerBuffer.put(cmd.getLanguage().getCode());
        // int version(~32767)
        headerBuffer.putShort((short) cmd.getVersion());
        // int opaque
        headerBuffer.putInt(cmd.getOpaque());
        // int flag
        headerBuffer.putInt(cmd.getFlag());
        // String remark
        if (remarkBytes != null) {
            headerBuffer.putInt(remarkBytes.length);
            headerBuffer.put(remarkBytes);
        } else {
            headerBuffer.putInt(0);
        }
        // HashMap<String, String> extFields;
        if (extFieldsBytes != null) {
            headerBuffer.putInt(extFieldsBytes.length);
            headerBuffer.put(extFieldsBytes);
        } else {
            headerBuffer.putInt(0);
        }

        return headerBuffer.array();
    }
```

### JSON序列化
```
    public static byte[] encode(final Object obj) {
        final String json = toJson(obj, false);
        if (json != null) {
            return json.getBytes(CHARSET_UTF8);
        }
        return null;
    }
```

### OnewayRPC
```
    public void markOnewayRPC() {
        int bits = 1 << RPC_ONEWAY;
        this.flag |= bits;
    }

    @JSONField(serialize = false)
    public boolean isOnewayRPC() {
        int bits = 1 << RPC_ONEWAY;
        return (this.flag & bits) == bits;
    }

```

```
    @JSONField(serialize = false)
    public boolean isResponseType() {
        int bits = 1 << RPC_TYPE;
        return (this.flag & bits) == bits;
    }
```


```
    class NettyClientHandler extends SimpleChannelInboundHandler<RemotingCommand> {

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
            processMessageReceived(ctx, msg);
        }
    }

    public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
        final RemotingCommand cmd = msg;
        if (cmd != null) {
            switch (cmd.getType()) {
                case REQUEST_COMMAND:
                    processRequestCommand(ctx, cmd);
                    break;
                case RESPONSE_COMMAND:
                    processResponseCommand(ctx, cmd);
                    break;
                default:
                    break;
            }
        }
    }
```


### processRequestCommand
```
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
//缓存着每个请求的code对应的处理器
//MQClientAPIImpl构造器里面初始化了
//CHECK_TRANSACTION_STATE(39)、NOTIFY_CONSUMER_IDS_CHANGED(40)、RESET_CONSUMER_CLIENT_OFFSET(220)、GET_CONSUMER_STATUS_FROM_CLIENT(221)、GET_CONSUMER_RUNNING_INFO(307)、CONSUME_MESSAGE_DIRECTLY（309）
        final Pair<NettyRequestProcessor, ExecutorService> matched = this.processorTable.get(cmd.getCode());
        final Pair<NettyRequestProcessor, ExecutorService> pair = null == matched ? this.defaultRequestProcessor : matched;
        final int opaque = cmd.getOpaque();

        if (pair != null) {
            Runnable run = new Runnable() {
                @Override
                public void run() {
                    try {
                        RPCHook rpcHook = NettyRemotingAbstract.this.getRPCHook();
                        if (rpcHook != null) {
                            rpcHook.doBeforeRequest(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd);
                        }

                        final RemotingCommand response = pair.getObject1().processRequest(ctx, cmd);
                        if (rpcHook != null) {
                            rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
                        }

                        if (!cmd.isOnewayRPC()) {
                            if (response != null) {
                                response.setOpaque(opaque);
                                response.markResponseType();
                                try {
                                    ctx.writeAndFlush(response);
                                } catch (Throwable e) {
                                    log.error("process request over, but response failed", e);
                                    log.error(cmd.toString());
                                    log.error(response.toString());
                                }
                            } else {

                            }
                        }
                    } catch (Throwable e) {
                        log.error("process request exception", e);
                        log.error(cmd.toString());

                        if (!cmd.isOnewayRPC()) {
                            final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_ERROR,
                                RemotingHelper.exceptionSimpleDesc(e));
                            response.setOpaque(opaque);
                            ctx.writeAndFlush(response);
                        }
                    }
                }
            };

            if (pair.getObject1().rejectRequest()) {
                final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                    "[REJECTREQUEST]system busy, start flow control for a while");
                response.setOpaque(opaque);
                ctx.writeAndFlush(response);
                return;
            }

            try {
                final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
                pair.getObject2().submit(requestTask);
            } catch (RejectedExecutionException e) {
                if ((System.currentTimeMillis() % 10000) == 0) {
                    log.warn(RemotingHelper.parseChannelRemoteAddr(ctx.channel())
                        + ", too many requests and system thread pool busy, RejectedExecutionException "
                        + pair.getObject2().toString()
                        + " request code: " + cmd.getCode());
                }

                if (!cmd.isOnewayRPC()) {
                    final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                        "[OVERLOAD]system busy, start flow control for a while");
                    response.setOpaque(opaque);
                    ctx.writeAndFlush(response);
                }
            }
        } else {
            String error = " request type " + cmd.getCode() + " not supported";
            final RemotingCommand response =
                RemotingCommand.createResponseCommand(RemotingSysResponseCode.REQUEST_CODE_NOT_SUPPORTED, error);
            response.setOpaque(opaque);
            ctx.writeAndFlush(response);
            log.error(RemotingHelper.parseChannelRemoteAddr(ctx.channel()) + error);
        }
    }
```

## processRequestCommand
```
public void processResponseCommand(ChannelHandlerContext ctx, RemotingCommand cmd) {
        final int opaque = cmd.getOpaque();
       //获取请求对应的responseFuture
        final ResponseFuture responseFuture = responseTable.get(opaque);
        if (responseFuture != null) {
            responseFuture.setResponseCommand(cmd);

            responseFuture.release();

            responseTable.remove(opaque);

            if (responseFuture.getInvokeCallback() != null) {
                executeInvokeCallback(responseFuture);
            } else {
                responseFuture.putResponse(cmd);
            }
        } else {
            log.warn("receive response, but not matched any request, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()));
            log.warn(cmd.toString());
        }
    }
```

![sequencediagram1](https://user-images.githubusercontent.com/7789698/33025070-d7112ef8-ce47-11e7-913f-4893bde369be.jpg)


部分重点代码：
```
 private SendResult sendKernelImpl(final Message msg,
        final MessageQueue mq,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final TopicPublishInfo topicPublishInfo,
        final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
        if (null == brokerAddr) {
            tryToFindTopicPublishInfo(mq.getTopic());
            brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
        }

        SendMessageContext context = null;
        if (brokerAddr != null) {
            brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);

            byte[] prevBody = msg.getBody();
            try {
                //for MessageBatch,ID has been set in the generating process
                if (!(msg instanceof MessageBatch)) {
                    MessageClientIDSetter.setUniqID(msg);
                }

                int sysFlag = 0;
                if (this.tryToCompressMessage(msg)) {
                    sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
                }

                final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
                if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
                    sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
                }

                if (hasCheckForbiddenHook()) {
                    CheckForbiddenContext checkForbiddenContext = new CheckForbiddenContext();
                    checkForbiddenContext.setNameSrvAddr(this.defaultMQProducer.getNamesrvAddr());
                    checkForbiddenContext.setGroup(this.defaultMQProducer.getProducerGroup());
                    checkForbiddenContext.setCommunicationMode(communicationMode);
                    checkForbiddenContext.setBrokerAddr(brokerAddr);
                    checkForbiddenContext.setMessage(msg);
                    checkForbiddenContext.setMq(mq);
                    checkForbiddenContext.setUnitMode(this.isUnitMode());
                    this.executeCheckForbiddenHook(checkForbiddenContext);
                }

                if (this.hasSendMessageHook()) {
                    context = new SendMessageContext();
                    context.setProducer(this);
                    context.setProducerGroup(this.defaultMQProducer.getProducerGroup());
                    context.setCommunicationMode(communicationMode);
                    context.setBornHost(this.defaultMQProducer.getClientIP());
                    context.setBrokerAddr(brokerAddr);
                    context.setMessage(msg);
                    context.setMq(mq);
                    String isTrans = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
                    if (isTrans != null && isTrans.equals("true")) {
                        context.setMsgType(MessageType.Trans_Msg_Half);
                    }

                    if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
                        context.setMsgType(MessageType.Delay_Msg);
                    }
                    this.executeSendMessageHookBefore(context);
                }

              //构成cmd里面的customHeader
                SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
              //ProducerGroup
                requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
              //Topic
                requestHeader.setTopic(msg.getTopic());
              //DefaultTopic
                requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
  //DefaultTopicQueueNums在发送消息时，自动创建服务器不存在的topic，默认创建的队列数（默认4）
requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
              //QueueId
                requestHeader.setQueueId(mq.getQueueId());
              //参见MessageSysFlag 跟事务消息有关
                requestHeader.setSysFlag(sysFlag);
             //
                requestHeader.setBornTimestamp(System.currentTimeMillis());
               
                requestHeader.setFlag(msg.getFlag());
                requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
             //reconsumeTimes代表消息重试次数
                requestHeader.setReconsumeTimes(0);

                requestHeader.setUnitMode(this.isUnitMode());
              //如果是批处理消息，放入状态
                requestHeader.setBatch(msg instanceof MessageBatch);
                if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
                    if (reconsumeTimes != null) {
                        requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
                        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
                    }

                    String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
                    if (maxReconsumeTimes != null) {
                        requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
                        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
                    }
                }

                SendResult sendResult = null;
                switch (communicationMode) {
                    case ASYNC:
                        sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                            brokerAddr,
                            mq.getBrokerName(),
                            msg,
                            requestHeader,
                            timeout,
                            communicationMode,
                            sendCallback,
                            topicPublishInfo,
                            this.mQClientFactory,
                            this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(),
                            context,
                            this);
                        break;
                    case ONEWAY:
                    case SYNC:
                        sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                            brokerAddr,
                            mq.getBrokerName(),
                            msg,
                            requestHeader,
                            timeout,
                            communicationMode,
                            context,
                            this);
                        break;
                    default:
                        assert false;
                        break;
                }

                if (this.hasSendMessageHook()) {
                    context.setSendResult(sendResult);
                    this.executeSendMessageHookAfter(context);
                }

                return sendResult;
            } catch (RemotingException e) {
                if (this.hasSendMessageHook()) {
                    context.setException(e);
                    this.executeSendMessageHookAfter(context);
                }
                throw e;
            } catch (MQBrokerException e) {
                if (this.hasSendMessageHook()) {
                    context.setException(e);
                    this.executeSendMessageHookAfter(context);
                }
                throw e;
            } catch (InterruptedException e) {
                if (this.hasSendMessageHook()) {
                    context.setException(e);
                    this.executeSendMessageHookAfter(context);
                }
                throw e;
            } finally {
                msg.setBody(prevBody);
            }
        }

        throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
    }
```

```
//12
request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);

    public static RemotingCommand createRequestCommand(int code, CommandCustomHeader customHeader) {
        RemotingCommand cmd = new RemotingCommand();
        cmd.setCode(code);
        cmd.customHeader = customHeader;
        setCmdVersion(cmd);
        return cmd;
    }
```
发送端做了什么：
![invokesync-client png](https://user-images.githubusercontent.com/7789698/33022975-bcdea14c-ce41-11e7-83ec-895d892a8ce1.png)

- 构建ResponseFuture,设置opaque值，把ResponseFuture以opaque为键放入Map中保存
- netty发送请送请求
- 发送成功设置ResponseFuture发送状态为成功；发送失败设置ResponseFuture发送失败，并且从Map存中移除ResponseFuture
- responseFuture.waitResponse(timeoutMillis)获取响应(如果超时则抛出异常)
- 收到服务端的回应以后，从Map中根据opaque拿出responseFuture，将回应写入其中，并从Map中移除
- resposneFuture得到回应，并将返回给客户端

接收端做了什么：
- netty监听得到发送过来的消息，分拣给Server端进行处理
- 根据消息的code得到对应的处理器(Processor)
- 创建一个新的线程，在这个线程中让处理器去处理消息，并得到回应(Response)。判断如果消息不是单向的(one-way)，则把请求中的opaque放回response中，并把消息发回给请求端。
- 将线程放入线程池--这里注意 请求消息的code对应了一组(Processor，ExectorService)，处理器和线程池是对应的

![invokesync2](https://user-images.githubusercontent.com/7789698/33023053-fd89c410-ce41-11e7-9d66-b82a536b1d89.png)


源码：
```
public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis)
        throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
//根据地址获取channel
        final Channel channel = this.getAndCreateChannel(addr);
        if (channel != null && channel.isActive()) {
            try {
                if (this.rpcHook != null) {
                    this.rpcHook.doBeforeRequest(addr, request);
                }
//
                RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis);
                if (this.rpcHook != null) {
                    this.rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
                }
                return response;
            } catch (RemotingSendRequestException e) {
                log.warn("invokeSync: send request exception, so close the channel[{}]", addr);
                this.closeChannel(addr, channel);
                throw e;
            } catch (RemotingTimeoutException e) {
                if (nettyClientConfig.isClientCloseSocketIfTimeout()) {
                    this.closeChannel(addr, channel);
                    log.warn("invokeSync: close socket because of timeout, {}ms, {}", timeoutMillis, addr);
                }
                log.warn("invokeSync: wait response timeout exception, the channel[{}]", addr);
                throw e;
            }
        } else {
            this.closeChannel(addr, channel);
            throw new RemotingConnectException(addr);
        }
    }
```


```
 public RemotingCommand invokeSyncImpl(final Channel channel, final RemotingCommand request,
        final long timeoutMillis)
        throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException {
          //获取请求的序号
        final int opaque = request.getOpaque();

        try {
            final ResponseFuture responseFuture = new ResponseFuture(opaque, timeoutMillis, null, null);
          //根据opaque序号存入responseFuture
           this.responseTable.put(opaque, responseFuture);
          //获取addr
            final SocketAddress addr = channel.remoteAddress();
          //调用netty的channel发送请求，监听返回回写response结果到responseFuture
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    if (f.isSuccess()) {
                        responseFuture.setSendRequestOK(true);
                        return;
                    } else {
                        responseFuture.setSendRequestOK(false);
                    }
                    //移除相应opaque对应responseFuture
                    responseTable.remove(opaque);
                    responseFuture.setCause(f.cause());
                    responseFuture.putResponse(null);
                    log.warn("send a request command to channel <" + addr + "> failed.");
                }
            });
             //等待putResponse 返回结果 this.countDownLatch.await(timeoutMillis, TimeUnit.MILLISECONDS); 
            RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);
            if (null == responseCommand) {
                if (responseFuture.isSendRequestOK()) {
                    throw new RemotingTimeoutException(RemotingHelper.parseSocketAddressAddr(addr), timeoutMillis,
                        responseFuture.getCause());
                } else {
                    throw new RemotingSendRequestException(RemotingHelper.parseSocketAddressAddr(addr), responseFuture.getCause());
                }
            }

            return responseCommand;
        } finally {
            this.responseTable.remove(opaque);
        }
    }
```


SendResult [sendStatus=SEND_OK, msgId=AC1107140D7C3764951D701DA4250000, offsetMsgId=AC1128E100002A9F0000000000579CFC, messageQueue=MessageQueue [topic=TopicTest, brokerName=kickseed, queueId=0], queueOffset=7595]


<img width="1279" alt="wx20171122-231321 2x" src="https://user-images.githubusercontent.com/7789698/33134496-d6f9a352-cfda-11e7-9250-2e87e9b32275.png">

<img width="1246" alt="wx20171122-231450 2x" src="https://user-images.githubusercontent.com/7789698/33134824-b2d92ed8-cfdb-11e7-9e4b-dc92a98af1a8.png">

 <p style="display:none;">666</p>