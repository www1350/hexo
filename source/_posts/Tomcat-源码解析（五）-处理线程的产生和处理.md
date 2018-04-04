---
title: Tomcat 源码解析（五）--处理线程的产生和处理
abbrlink: e3943dbd
date: 2018-03-12 22:43:23
tags: [tomcat]
categories: 源码
---

Catalina

 ```
       digester.addRule("Server/Service/Connector",
                         new ConnectorCreateRule());
        digester.addRule("Server/Service/Connector",
                         new SetAllPropertiesRule(new String[]{"executor", "sslImplementationName"}));
        digester.addSetNext("Server/Service/Connector",
                            "addConnector",
                            "org.apache.catalina.connector.Connector");
 ```

ConnectorCreateRule
```
@Override
    public void begin(String namespace, String name, Attributes attributes)
            throws Exception {
        Service svc = (Service)digester.peek();
        Executor ex = null;
        if ( attributes.getValue("executor")!=null ) {
            ex = svc.getExecutor(attributes.getValue("executor"));
        }
//根据protocol
        Connector con = new Connector(attributes.getValue("protocol"));
        if (ex != null) {
            setExecutor(con, ex);
        }
        String sslImplementationName = attributes.getValue("sslImplementationName");
        if (sslImplementationName != null) {
            setSSLImplementationName(con, sslImplementationName);
        }
        digester.push(con);
    }
```

Connector
```
    public Connector(String protocol) {
        setProtocol(protocol);
        // Instantiate protocol handler
        ProtocolHandler p = null;
        try {
//加载ProtocolHandlerClassName 
            Class<?> clazz = Class.forName(protocolHandlerClassName);
            p = (ProtocolHandler) clazz.getConstructor().newInstance();
        } catch (Exception e) {
            log.error(sm.getString(
                    "coyoteConnector.protocolHandlerInstantiationFailed"), e);
        } finally {
            this.protocolHandler = p;
        }

        if (Globals.STRICT_SERVLET_COMPLIANCE) {
            uriCharset = StandardCharsets.ISO_8859_1;
        } else {
            uriCharset = StandardCharsets.UTF_8;
        }
    }


    @Deprecated
    public void setProtocol(String protocol) {

        boolean aprConnector = AprLifecycleListener.isAprAvailable() &&
                AprLifecycleListener.getUseAprConnector();

//如果是Connector protocol="HTTP/1.1"
        if ("HTTP/1.1".equals(protocol) || protocol == null) {
            if (aprConnector) {
//设置ProtocolHandlerClassName
                setProtocolHandlerClassName("org.apache.coyote.http11.Http11AprProtocol");
            } else {
                setProtocolHandlerClassName("org.apache.coyote.http11.Http11NioProtocol");
            }
        } else if ("AJP/1.3".equals(protocol)) {
            if (aprConnector) {
                setProtocolHandlerClassName("org.apache.coyote.ajp.AjpAprProtocol");
            } else {
                setProtocolHandlerClassName("org.apache.coyote.ajp.AjpNioProtocol");
            }
        } else {
            setProtocolHandlerClassName(protocol);
        }
    }

```

Http11AprProtocol

```
public Http11AprProtocol() {
        super(new AprEndpoint());
    }

    public AbstractHttp11Protocol(AbstractEndpoint<S> endpoint) {
        super(endpoint);
        setConnectionTimeout(Constants.DEFAULT_CONNECTION_TIMEOUT);
//初始化ConnectionHandler
        ConnectionHandler<S> cHandler = new ConnectionHandler<>(this);
        setHandler(cHandler);
        getEndpoint().setHandler(cHandler);
    }

    public AbstractProtocol(AbstractEndpoint<S> endpoint) {
        this.endpoint = endpoint;
        setSoLinger(Constants.DEFAULT_CONNECTION_LINGER);
        setTcpNoDelay(Constants.DEFAULT_TCP_NO_DELAY);
    }


    public AprEndpoint() {
        // Need to override the default for maxConnections to align it with what
        // was pollerSize (before the two were merged)
        setMaxConnections(8 * 1024);
    }
```

![image](https://user-images.githubusercontent.com/7789698/37265784-af4e45f8-25f0-11e8-84bc-03971bf6d410.png)

AbstractProtocol的start

```
    @Override
    public void start() throws Exception {

//AbstractEndpoint
        endpoint.start();

        // Start async timeout thread
        asyncTimeout = new AsyncTimeout();
//创建http-apr-8080-AsyncTimeout线程（守护进程）
//作用：检测超时的请求，并将该请求再转发到工作线程池处理
        Thread timeoutThread = new Thread(asyncTimeout, getNameInternal() + "-AsyncTimeout");
        int priority = endpoint.getThreadPriority();
        if (priority < Thread.MIN_PRIORITY || priority > Thread.MAX_PRIORITY) {
            priority = Thread.NORM_PRIORITY;
        }
        timeoutThread.setPriority(priority);
        timeoutThread.setDaemon(true);
//启动
        timeoutThread.start();
    }

```

AprEndpoint（AbstractEndpoint）的start
```
    public final void start() throws Exception {
        if (bindState == BindState.UNBOUND) {
//bind
            bind();
            bindState = BindState.BOUND_ON_START;
        }
        startInternal();
    }

  



 @Override
    public void startInternal() throws Exception {

        if (!running) {
            running = true;
            paused = false;

            processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                    socketProperties.getProcessorCache());

            // Create worker collection
            if (getExecutor() == null) {
                createExecutor();
            }

            initializeConnectionLatch();

            // Start poller thread
            poller = new Poller();
            poller.init();
            Thread pollerThread = new Thread(poller, getName() + "-Poller");
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();

            // Start sendfile thread
            if (getUseSendfile()) {
                sendfile = new Sendfile();
                sendfile.init();
                Thread sendfileThread =
                        new Thread(sendfile, getName() + "-Sendfile");
                sendfileThread.setPriority(threadPriority);
                sendfileThread.setDaemon(true);
                sendfileThread.start();
            }
//创建Acceptor线程
            startAcceptorThreads();
        }
    }


    public void createExecutor() {
        internalExecutor = true;
        TaskQueue taskqueue = new TaskQueue();
//http-apr-8080-exec-
        TaskThreadFactory tf = new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
        executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), 60, TimeUnit.SECONDS,taskqueue, tf);
        taskqueue.setParent( (ThreadPoolExecutor) executor);
    }


    protected final void startAcceptorThreads() {
//accept队列的长度；当accept队列中连接的个数达到acceptCount时，队列满，进来的请求一律被拒绝。默认值是100。
        int count = getAcceptorThreadCount();
        acceptors = new Acceptor[count];

        for (int i = 0; i < count; i++) {
//new Acceptor()
            acceptors[i] = createAcceptor();
//http-apr-8080-Acceptor-0
            String threadName = getName() + "-Acceptor-" + i;
            acceptors[i].setThreadName(threadName);
            Thread t = new Thread(acceptors[i], threadName);
            t.setPriority(getAcceptorThreadPriority());
            t.setDaemon(getDaemon());
            t.start();
        }
    }
```

到这里就启动了一个http-apr-8080-AsyncTimeout线程和多个http-apr-8080-Acceptor-线程

这里我们着重看下Acceptor线程

```
protected class Acceptor extends AbstractEndpoint.Acceptor {

        private final Log log = LogFactory.getLog(AprEndpoint.Acceptor.class);

        @Override
        public void run() {

            int errorDelay = 0;

            // 除非收到shutdown命令否则一直循环
            while (running) {

                // Loop if endpoint is paused
                while (paused && running) {
                    state = AcceptorState.PAUSED;
                    try {
                        Thread.sleep(50);
                    } catch (InterruptedException e) {
                        // Ignore
                    }
                }

                if (!running) {
                    break;
                }
                state = AcceptorState.RUNNING;

                try {
                    //if we have reached max connections, wait
//使用了connectionLimitLatch限制最大连接数，到达上限就wait
                    countUpOrAwaitConnection();

                    long socket = 0;
                    try {
                        // Accept the next incoming connection from the server
                        // socket 监听到客户端的连接
                        socket = Socket.accept(serverSock);

                    } catch (Exception e) {
                        // We didn't get a socket
                        countDownConnection();
                        if (running) {
                            // Introduce delay if necessary
                            errorDelay = handleExceptionWithDelay(errorDelay);
                            // re-throw
                            throw e;
                        } else {
                            break;
                        }
                    }
                    // Successful accept, reset the error delay
                    errorDelay = 0;

                    if (running && !paused) {
                        // 把这个socket给一个合适的处理器处理
                        if (!processSocketWithOptions(socket)) {
                            // Close socket right away
                            closeSocket(socket);
                        }
                    } else {
                        // Close socket right away
                        // No code path could have added the socket to the
                        // Poller so use destroySocket()
                        destroySocket(socket);
                    }
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    String msg = sm.getString("endpoint.accept.fail");
                    if (t instanceof Error) {
                        Error e = (Error) t;
                        if (e.getError() == 233) {
                            // Not an error on HP-UX so log as a warning
                            // so it can be filtered out on that platform
                            // See bug 50273
                            log.warn(msg, t);
                        } else {
                            log.error(msg, t);
                        }
                    } else {
                            log.error(msg, t);
                    }
                }
                // The processor will recycle itself when it finishes
            }
            state = AcceptorState.ENDED;
        }
    }
```


```
protected boolean processSocketWithOptions(long socket) {
        try {
            // During shutdown, executor may be null - avoid NPE
            if (running) {
  //包装成AprSocketWrapper
                AprSocketWrapper wrapper = new AprSocketWrapper(Long.valueOf(socket), this);
                wrapper.setKeepAliveLeft(getMaxKeepAliveRequests());
                wrapper.setSecure(isSSLEnabled());
                wrapper.setReadTimeout(getConnectionTimeout());
                wrapper.setWriteTimeout(getConnectionTimeout());
                connections.put(Long.valueOf(socket), wrapper);
//SocketWithOptionsProcessor丢到线程池执行
                getExecutor().execute(new SocketWithOptionsProcessor(wrapper));
            }
        } catch (RejectedExecutionException x) {
            log.warn("Socket processing request was rejected for:"+socket,x);
            return false;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            // This means we got an OOM or similar creating a thread, or that
            // the pool and its queue are full
            log.error(sm.getString("endpoint.process.fail"), t);
            return false;
        }
        return true;
    }

```

SocketWithOptionsProcessor
```
protected class SocketWithOptionsProcessor implements Runnable {

        protected SocketWrapperBase<Long> socket = null;

        public SocketWithOptionsProcessor(SocketWrapperBase<Long> socket) {
            this.socket = socket;
        }

        @Override
        public void run() {

            synchronized (socket) {
                if (!deferAccept) {
                    if (setSocketOptions(socket)) {
                        getPoller().add(socket.getSocket().longValue(),
                                getConnectionTimeout(), Poll.APR_POLLIN);
                    } else {
                        // Close socket and pool
                        closeSocket(socket.getSocket().longValue());
                        socket = null;
                    }
                } else {
                    // Process the request from this socket
                    if (!setSocketOptions(socket)) {
                        // Close socket and pool
                        closeSocket(socket.getSocket().longValue());
                        socket = null;
                        return;
                    }
                    // Process the request from this socket
//用Http11AprProtocol 处理
                    Handler.SocketState state = getHandler().process(socket,
                            SocketEvent.OPEN_READ);
                    if (state == Handler.SocketState.CLOSED) {
                        // Close socket and pool
                        closeSocket(socket.getSocket().longValue());
                        socket = null;
                    }
                }
            }
        }
    }
```



```
@Override
        public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {
            if (wrapper == null) {
                // Nothing to do. Socket has been closed.
                return SocketState.CLOSED;
            }

            S socket = wrapper.getSocket();
//缓存一个map
            Processor processor = connections.get(socket);

            if (processor != null) {
                // Make sure an async timeout doesn't fire
                getProtocol().removeWaitingProcessor(processor);
            } else if (status == SocketEvent.DISCONNECT || status == SocketEvent.ERROR) {
                // Nothing to do. Endpoint requested a close and there is no
                // longer a processor associated with this socket.
                return SocketState.CLOSED;
            }

            ContainerThreadMarker.set();

            try {
                if (processor == null) {
                    String negotiatedProtocol = wrapper.getNegotiatedProtocol();
                    if (negotiatedProtocol != null) {
                        UpgradeProtocol upgradeProtocol =
                                getProtocol().getNegotiatedProtocol(negotiatedProtocol);
                        if (upgradeProtocol != null) {
                            processor = upgradeProtocol.getProcessor(
                                    wrapper, getProtocol().getAdapter());
                        } else if (negotiatedProtocol.equals("http/1.1")) {
                            // Explicitly negotiated the default protocol.
                            // Obtain a processor below.
                        } else {
                            // TODO:
                            // OpenSSL 1.0.2's ALPN callback doesn't support
                            // failing the handshake with an error if no
                            // protocol can be negotiated. Therefore, we need to
                            // fail the connection here. Once this is fixed,
                            // replace the code below with the commented out
                            // block.
                            return SocketState.CLOSED;
                            /*
                             * To replace the code above once OpenSSL 1.1.0 is
                             * used.
                            // Failed to create processor. This is a bug.
                            throw new IllegalStateException(sm.getString(
                                    "abstractConnectionHandler.negotiatedProcessor.fail",
                                    negotiatedProtocol));
                            */
                        }
                    }
                }
                if (processor == null) {
                    processor = recycledProcessors.pop();
                }
                if (processor == null) {
//创建processor，这里不贴代码了。实际上就是Http11AprProtocol（AbstractHttp11Protocol）的createProcessor，最后构造的是Http11Processor类。
                    processor = getProtocol().createProcessor();
                    register(processor);
                }

                processor.setSslSupport(
                        wrapper.getSslSupport(getProtocol().getClientCertProvider()));

                // Associate the processor with the connection
                connections.put(socket, processor);

                SocketState state = SocketState.CLOSED;
                do {
//最关键的地方！ 
                    state = processor.process(wrapper, status);

                    if (state == SocketState.UPGRADING) {
...
                       //UPGRADING
                    }
                } while ( state == SocketState.UPGRADING);

                if (state == SocketState.LONG) {
                    // In the middle of processing a request/response. Keep the
                    // socket associated with the processor. Exact requirements
                    // depend on type of long poll
                    longPoll(wrapper, processor);
                    if (processor.isAsync()) {
                        getProtocol().addWaitingProcessor(processor);
                    }
                } else if (state == SocketState.OPEN) {
                    // In keep-alive but between requests. OK to recycle
                    // processor. Continue to poll for the next request.
                    connections.remove(socket);
                    release(processor);
                    wrapper.registerReadInterest();
                } else if (state == SocketState.SENDFILE) {
                    // Sendfile in progress. If it fails, the socket will be
                    // closed. If it works, the socket either be added to the
                    // poller (or equivalent) to await more data or processed
                    // if there are any pipe-lined requests remaining.
                } else if (state == SocketState.UPGRADED) {
                    // Don't add sockets back to the poller if this was a
                    // non-blocking write otherwise the poller may trigger
                    // multiple read events which may lead to thread starvation
                    // in the connector. The write() method will add this socket
                    // to the poller if necessary.
                    if (status != SocketEvent.OPEN_WRITE) {
                        longPoll(wrapper, processor);
                    }
                } else if (state == SocketState.SUSPENDED) {
                    // Don't add sockets back to the poller.
                    // The resumeProcessing() method will add this socket
                    // to the poller.
                } else {
...
//Upgrade
                }
                return state;
            }  finally {
                ContainerThreadMarker.clear();
            }

            // Make sure socket/processor is removed from the list of current
            // connections
            connections.remove(socket);
            release(processor);
            return SocketState.CLOSED;
        }
```

![image](https://user-images.githubusercontent.com/7789698/37266564-00072780-25f6-11e8-8295-a142cb4e9f5f.png)

AbstractProcessorLight的process
```
 @Override
    public SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status)
            throws IOException {

        SocketState state = SocketState.CLOSED;
        Iterator<DispatchType> dispatches = null;
        do {
            if (dispatches != null) {
                DispatchType nextDispatch = dispatches.next();
                state = dispatch(nextDispatch.getSocketStatus());
            } else if (status == SocketEvent.DISCONNECT) {
                // Do nothing here, just wait for it to get recycled
            } else if (isAsync() || isUpgrade() || state == SocketState.ASYNC_END) {
                state = dispatch(status);
                if (state == SocketState.OPEN) {
                    // There may be pipe-lined data to read. If the data isn't
                    // processed now, execution will exit this loop and call
                    // release() which will recycle the processor (and input
                    // buffer) deleting any pipe-lined data. To avoid this,
                    // process it now.
                    state = service(socketWrapper);
                }
            } else if (status == SocketEvent.OPEN_WRITE) {
                // Extra write event likely after async, ignore
                state = SocketState.LONG;
            } else if (status == SocketEvent.OPEN_READ){
                state = service(socketWrapper);
            } else {
                // Default to closing the socket if the SocketEvent passed in
                // is not consistent with the current state of the Processor
                state = SocketState.CLOSED;
            }

            if (state != SocketState.CLOSED && isAsync()) {
                state = asyncPostProcess();
            }


            if (dispatches == null || !dispatches.hasNext()) {
                // Only returns non-null iterator if there are
                // dispatches to process.
                dispatches = getIteratorAndClearDispatches();
            }
        } while (state == SocketState.ASYNC_END ||
                dispatches != null && state != SocketState.CLOSED);

        return state;
    }
```



```
@Override
    public SocketState service(SocketWrapperBase<?> socketWrapper)
        throws IOException {
        RequestInfo rp = request.getRequestProcessor();
        rp.setStage(org.apache.coyote.Constants.STAGE_PARSE);

        // Setting up the I/O
        setSocketWrapper(socketWrapper);
        inputBuffer.init(socketWrapper);
        outputBuffer.init(socketWrapper);

        // Flags
        keepAlive = true;
        openSocket = false;
        readComplete = true;
        boolean keptAlive = false;
        SendfileState sendfileState = SendfileState.DONE;

        while (!getErrorState().isError() && keepAlive && !isAsync() && upgradeToken == null &&
                sendfileState == SendfileState.DONE && !endpoint.isPaused()) {

            // Parsing the request header
            try {
//解析请求头，并装入inputBuffer里的私有字段org.apache.coyote.Request的属性里
                if (!inputBuffer.parseRequestLine(keptAlive)) {
                    if (inputBuffer.getParsingRequestLinePhase() == -1) {
                        return SocketState.UPGRADING;
                    } else if (handleIncompleteRequestLineRead()) {
                        break;
                    }
                }

                if (endpoint.isPaused()) {
                    // 503 - Service unavailable
                    response.setStatus(503);
                    setErrorState(ErrorState.CLOSE_CLEAN, null);
                } else {
                    keptAlive = true;
                    // Set this every time in case limit has been changed via JMX
                    request.getMimeHeaders().setLimit(endpoint.getMaxHeaderCount());
                    if (!inputBuffer.parseHeaders()) {
                        // We've read part of the request, don't recycle it
                        // instead associate it with the socket
                        openSocket = true;
                        readComplete = false;
                        break;
                    }
                    if (!disableUploadTimeout) {
                        socketWrapper.setReadTimeout(connectionUploadTimeout);
                    }
                }
            } catch (IOException e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("http11processor.header.parse"), e);
                }
                setErrorState(ErrorState.CLOSE_CONNECTION_NOW, e);
                break;
            } catch (Throwable t) {
...           
                // 400 - Bad Request
                response.setStatus(400);
                setErrorState(ErrorState.CLOSE_CLEAN, t);
                getAdapter().log(request, response, 0);
            }

            // Has an upgrade been requested?
            Enumeration<String> connectionValues = request.getMimeHeaders().values("Connection");
            boolean foundUpgrade = false;
            while (connectionValues.hasMoreElements() && !foundUpgrade) {
                foundUpgrade = connectionValues.nextElement().toLowerCase(
                        Locale.ENGLISH).contains("upgrade");
            }

            if (foundUpgrade) {
                // Check the protocol
                String requestedProtocol = request.getHeader("Upgrade");

                UpgradeProtocol upgradeProtocol = httpUpgradeProtocols.get(requestedProtocol);
//转换传输协议
                if (upgradeProtocol != null) {
                    if (upgradeProtocol.accept(request)) {
                        // TODO Figure out how to handle request bodies at this
                        // point.
                        response.setStatus(HttpServletResponse.SC_SWITCHING_PROTOCOLS);
                        response.setHeader("Connection", "Upgrade");
                        response.setHeader("Upgrade", requestedProtocol);
                        action(ActionCode.CLOSE,  null);
                        getAdapter().log(request, response, 0);

                        InternalHttpUpgradeHandler upgradeHandler =
                                upgradeProtocol.getInternalUpgradeHandler(
                                        getAdapter(), cloneRequest(request));
                        UpgradeToken upgradeToken = new UpgradeToken(upgradeHandler, null, null);
                        action(ActionCode.UPGRADE, upgradeToken);
                        return SocketState.UPGRADING;
                    }
                }
            }

            if (!getErrorState().isError()) {
                // Setting up filters, and parse some request headers
                rp.setStage(org.apache.coyote.Constants.STAGE_PREPARE);
                try {
                    prepareRequest();
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    if (log.isDebugEnabled()) {
                        log.debug(sm.getString("http11processor.request.prepare"), t);
                    }
                    // 500 - Internal Server Error
                    response.setStatus(500);
                    setErrorState(ErrorState.CLOSE_CLEAN, t);
                    getAdapter().log(request, response, 0);
                }
            }

            if (maxKeepAliveRequests == 1) {
                keepAlive = false;
            } else if (maxKeepAliveRequests > 0 &&
                    socketWrapper.decrementKeepAlive() <= 0) {
                keepAlive = false;
            }

            // Process the request in the adapter
//用adapter处理请求
            if (!getErrorState().isError()) {
                try {
                    rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                    getAdapter().service(request, response);
                    // Handle when the response was committed before a serious
                    // error occurred.  Throwing a ServletException should both
                    // set the status to 500 and set the errorException.
                    // If we fail here, then the response is likely already
                    // committed, so we can't try and set headers.
                    if(keepAlive && !getErrorState().isError() && !isAsync() &&
                            statusDropsConnection(response.getStatus())) {
                        setErrorState(ErrorState.CLOSE_CLEAN, null);
                    }
                } catch (InterruptedIOException e) {
                    setErrorState(ErrorState.CLOSE_CONNECTION_NOW, e);
                } catch (HeadersTooLargeException e) {
                    log.error(sm.getString("http11processor.request.process"), e);
                    // The response should not have been committed but check it
                    // anyway to be safe
                    if (response.isCommitted()) {
                        setErrorState(ErrorState.CLOSE_NOW, e);
                    } else {
                        response.reset();
                        response.setStatus(500);
                        setErrorState(ErrorState.CLOSE_CLEAN, e);
                        response.setHeader("Connection", "close"); // TODO: Remove
                    }
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    log.error(sm.getString("http11processor.request.process"), t);
                    // 500 - Internal Server Error
                    response.setStatus(500);
                    setErrorState(ErrorState.CLOSE_CLEAN, t);
                    getAdapter().log(request, response, 0);
                }
            }

            // Finish the handling of the request
            rp.setStage(org.apache.coyote.Constants.STAGE_ENDINPUT);
            if (!isAsync()) {
                // If this is an async request then the request ends when it has
                // been completed. The AsyncContext is responsible for calling
                // endRequest() in that case.
                endRequest();
            }
            rp.setStage(org.apache.coyote.Constants.STAGE_ENDOUTPUT);

            // If there was an error, make sure the request is counted as
            // and error, and update the statistics counter
            if (getErrorState().isError()) {
                response.setStatus(500);
            }

            if (!isAsync() || getErrorState().isError()) {
                request.updateCounters();
                if (getErrorState().isIoAllowed()) {
                    inputBuffer.nextRequest();
                    outputBuffer.nextRequest();
                }
            }

            if (!disableUploadTimeout) {
                int soTimeout = endpoint.getConnectionTimeout();
                if(soTimeout > 0) {
                    socketWrapper.setReadTimeout(soTimeout);
                } else {
                    socketWrapper.setReadTimeout(0);
                }
            }

            rp.setStage(org.apache.coyote.Constants.STAGE_KEEPALIVE);

            sendfileState = processSendfile(socketWrapper);
        }

        rp.setStage(org.apache.coyote.Constants.STAGE_ENDED);

        if (getErrorState().isError() || endpoint.isPaused()) {
            return SocketState.CLOSED;
        } else if (isAsync()) {
            return SocketState.LONG;
        } else if (isUpgrade()) {
            return SocketState.UPGRADING;
        } else {
            if (sendfileState == SendfileState.PENDING) {
                return SocketState.SENDFILE;
            } else {
                if (openSocket) {
                    if (readComplete) {
                        return SocketState.OPEN;
                    } else {
                        return SocketState.LONG;
                    }
                } else {
                    return SocketState.CLOSED;
                }
            }
        }
    }
```


所以很重要的一句`getAdapter().service(request, response);`，而这里的adapter就是CoyoteAdapter

```
@Override
    public void service(org.apache.coyote.Request req, org.apache.coyote.Response res)
            throws Exception {
//org.apache.coyote.Request转化成org.apache.catalina.connector.Request 
        Request request = (Request) req.getNote(ADAPTER_NOTES);
//类似
        Response response = (Response) res.getNote(ADAPTER_NOTES);

        if (request == null) {
            // Create objects
            request = connector.createRequest();
            request.setCoyoteRequest(req);
            response = connector.createResponse();
            response.setCoyoteResponse(res);

            // Link objects
            request.setResponse(response);
            response.setRequest(request);

            // Set as notes
            req.setNote(ADAPTER_NOTES, request);
            res.setNote(ADAPTER_NOTES, response);

            // Set query string encoding
            req.getParameters().setQueryStringCharset(connector.getURICharset());
        }

        if (connector.getXpoweredBy()) {
            response.addHeader("X-Powered-By", POWERED_BY);
        }

        boolean async = false;
        boolean postParseSuccess = false;

        req.getRequestProcessor().setWorkerThreadName(THREAD_NAME.get());

        try {
            // Parse and set Catalina and configuration specific
            // request parameters
//很关键，请求转化为 Host、Context、Wrapper组件
            postParseSuccess = postParseRequest(req, request, res, response);
            if (postParseSuccess) {
                //check valves if we support async
                request.setAsyncSupported(
                        connector.getService().getContainer().getPipeline().isAsyncSupported());
                // Calling the container
//connector.getService()获取StandardService
//getContainer()获取StandardEngine
//getPipeline()获取StandardPipeline
//getFirst() 获取第一个阀 Valve
//这里的基础阀其实就是StandardContextValve，见下面
                connector.getService().getContainer().getPipeline().getFirst().invoke(
                        request, response);
            }
            if (request.isAsync()) {
                async = true;
                ReadListener readListener = req.getReadListener();
                if (readListener != null && request.isFinished()) {
                    // Possible the all data may have been read during service()
                    // method so this needs to be checked here
                    ClassLoader oldCL = null;
                    try {
                        oldCL = request.getContext().bind(false, null);
                        if (req.sendAllDataReadEvent()) {
                            req.getReadListener().onAllDataRead();
                        }
                    } finally {
                        request.getContext().unbind(false, oldCL);
                    }
                }

                Throwable throwable =
                        (Throwable) request.getAttribute(RequestDispatcher.ERROR_EXCEPTION);

                // If an async request was started, is not going to end once
                // this container thread finishes and an error occurred, trigger
                // the async error process
                if (!request.isAsyncCompleting() && throwable != null) {
                    request.getAsyncContextInternal().setErrorState(throwable, true);
                }
            } else {
                request.finishRequest();
                response.finishResponse();
            }

        } catch (IOException e) {
            // Ignore
        } finally {
            AtomicBoolean error = new AtomicBoolean(false);
            res.action(ActionCode.IS_ERROR, error);

            if (request.isAsyncCompleting() && error.get()) {
                // Connection will be forcibly closed which will prevent
                // completion happening at the usual point. Need to trigger
                // call to onComplete() here.
                res.action(ActionCode.ASYNC_POST_PROCESS,  null);
                async = false;
            }

            // Access log
            if (!async && postParseSuccess) {
                // Log only if processing was invoked.
                // If postParseRequest() failed, it has already logged it.
                Context context = request.getContext();
                // If the context is null, it is likely that the endpoint was
                // shutdown, this connection closed and the request recycled in
                // a different thread. That thread will have updated the access
                // log so it is OK not to update the access log here in that
                // case.
                if (context != null) {
                    context.logAccess(request, response,
                            System.currentTimeMillis() - req.getStartTime(), false);
                }
            }

            req.getRequestProcessor().setWorkerThreadName(null);

            // Recycle the wrapper request and response
            if (!async) {
                request.recycle();
                response.recycle();
            }
        }
    }
```


postParseRequest
```
protected boolean postParseRequest(org.apache.coyote.Request req, Request request,
            org.apache.coyote.Response res, Response response) throws IOException, ServletException {

        // If the processor has set the scheme (AJP does this, HTTP does this if
        // SSL is enabled) use this to set the secure flag as well. If the
        // processor hasn't set it, use the settings from the connector
        if (req.scheme().isNull()) {
            // Use connector scheme and secure configuration, (defaults to
            // "http" and false respectively)
            req.scheme().setString(connector.getScheme());
            request.setSecure(connector.getSecure());
        } else {
            // Use processor specified scheme to determine secure state
            request.setSecure(req.scheme().equals("https"));
        }

        // At this point the Host header has been processed.
        // Override if the proxyPort/proxyHost are set
        String proxyName = connector.getProxyName();
        int proxyPort = connector.getProxyPort();
        if (proxyPort != 0) {
            req.setServerPort(proxyPort);
        } else if (req.getServerPort() == -1) {
            // Not explicitly set. Use default ports based on the scheme
            if (req.scheme().equals("https")) {
                req.setServerPort(443);
            } else {
                req.setServerPort(80);
            }
        }
        if (proxyName != null) {
            req.serverName().setString(proxyName);
        }

//uri
        MessageBytes undecodedURI = req.requestURI();

        // Check for ping OPTIONS * request
        if (undecodedURI.equals("*")) {
            if (req.method().equalsIgnoreCase("OPTIONS")) {
                StringBuilder allow = new StringBuilder();
                allow.append("GET, HEAD, POST, PUT, DELETE");
                // Trace if allowed
                if (connector.getAllowTrace()) {
                    allow.append(", TRACE");
                }
                // Always allow options
                allow.append(", OPTIONS");
                res.setHeader("Allow", allow.toString());
            } else {
                res.setStatus(404);
                res.setMessage("Not found");
            }
            connector.getService().getContainer().logAccess(
                    request, response, 0, true);
            return false;
        }

        MessageBytes decodedURI = req.decodedURI();

        if (undecodedURI.getType() == MessageBytes.T_BYTES) {
            // Copy the raw URI to the decodedURI
            decodedURI.duplicate(undecodedURI);

            // Parse the path parameters. This will:
            //   - strip out the path parameters
            //   - convert the decodedURI to bytes
            parsePathParameters(req, request);

            // URI decoding
            // %xx decoding of the URL
            try {
                req.getURLDecoder().convert(decodedURI, false);
            } catch (IOException ioe) {
                res.setStatus(400);
                res.setMessage("Invalid URI: " + ioe.getMessage());
                connector.getService().getContainer().logAccess(
                        request, response, 0, true);
                return false;
            }
            // Normalization
            if (!normalize(req.decodedURI())) {
                res.setStatus(400);
                res.setMessage("Invalid URI");
                connector.getService().getContainer().logAccess(
                        request, response, 0, true);
                return false;
            }
            // Character decoding
            convertURI(decodedURI, request);
            // Check that the URI is still normalized
            if (!checkNormalize(req.decodedURI())) {
                res.setStatus(400);
                res.setMessage("Invalid URI character encoding");
                connector.getService().getContainer().logAccess(
                        request, response, 0, true);
                return false;
            }
        } else {
            /* The URI is chars or String, and has been sent using an in-memory
             * protocol handler. The following assumptions are made:
             * - req.requestURI() has been set to the 'original' non-decoded,
             *   non-normalized URI
             * - req.decodedURI() has been set to the decoded, normalized form
             *   of req.requestURI()
             */
            decodedURI.toChars();
            // Remove all path parameters; any needed path parameter should be set
            // using the request object rather than passing it in the URL
            CharChunk uriCC = decodedURI.getCharChunk();
            int semicolon = uriCC.indexOf(';');
            if (semicolon > 0) {
                decodedURI.setChars
                    (uriCC.getBuffer(), uriCC.getStart(), semicolon);
            }
        }

        // Request mapping.
        MessageBytes serverName;
        if (connector.getUseIPVHosts()) {
            serverName = req.localName();
            if (serverName.isNull()) {
                // well, they did ask for it
                res.action(ActionCode.REQ_LOCAL_NAME_ATTRIBUTE, null);
            }
        } else {
            serverName = req.serverName();
        }

        // Version for the second mapping loop and
        // Context that we expect to get for that version
        String version = null;
        Context versionContext = null;
        boolean mapRequired = true;

        while (mapRequired) {
            // This will map the the latest version by default
//请求转化成Host、Context、Wrapper
            connector.getService().getMapper().map(serverName, decodedURI,
                    version, request.getMappingData());

            // If there is no context at this point, it is likely no ROOT context
            // has been deployed
            if (request.getContext() == null) {
                res.setStatus(404);
                res.setMessage("Not found");
                // No context, so use host
                Host host = request.getHost();
                // Make sure there is a host (might not be during shutdown)
                if (host != null) {
                    host.logAccess(request, response, 0, true);
                }
                return false;
            }

            // Now we have the context, we can parse the session ID from the URL
            // (if any). Need to do this before we redirect in case we need to
            // include the session id in the redirect
            String sessionID;
            if (request.getServletContext().getEffectiveSessionTrackingModes()
                    .contains(SessionTrackingMode.URL)) {

                // Get the session ID if there was one
                sessionID = request.getPathParameter(
                        SessionConfig.getSessionUriParamName(
                                request.getContext()));
                if (sessionID != null) {
                    request.setRequestedSessionId(sessionID);
                    request.setRequestedSessionURL(true);
                }
            }

            // Look for session ID in cookies and SSL session
            parseSessionCookiesId(request);
            parseSessionSslId(request);

            sessionID = request.getRequestedSessionId();

            mapRequired = false;
            if (version != null && request.getContext() == versionContext) {
                // We got the version that we asked for. That is it.
            } else {
                version = null;
                versionContext = null;

                Context[] contexts = request.getMappingData().contexts;
                // Single contextVersion means no need to remap
                // No session ID means no possibility of remap
                if (contexts != null && sessionID != null) {
                    // Find the context associated with the session
                    for (int i = (contexts.length); i > 0; i--) {
                        Context ctxt = contexts[i - 1];
                        if (ctxt.getManager().findSession(sessionID) != null) {
                            // We found a context. Is it the one that has
                            // already been mapped?
                            if (!ctxt.equals(request.getMappingData().context)) {
                                // Set version so second time through mapping
                                // the correct context is found
                                version = ctxt.getWebappVersion();
                                versionContext = ctxt;
                                // Reset mapping
                                request.getMappingData().recycle();
                                mapRequired = true;
                                // Recycle cookies and session info in case the
                                // correct context is configured with different
                                // settings
                                request.recycleSessionInfo();
                                request.recycleCookieInfo(true);
                            }
                            break;
                        }
                    }
                }
            }

            if (!mapRequired && request.getContext().getPaused()) {
                // Found a matching context but it is paused. Mapping data will
                // be wrong since some Wrappers may not be registered at this
                // point.
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    // Should never happen
                }
                // Reset mapping
                request.getMappingData().recycle();
                mapRequired = true;
            }
        }

        // Possible redirect
        MessageBytes redirectPathMB = request.getMappingData().redirectPath;
        if (!redirectPathMB.isNull()) {
            String redirectPath = URLEncoder.DEFAULT.encode(
                    redirectPathMB.toString(), StandardCharsets.UTF_8);
            String query = request.getQueryString();
            if (request.isRequestedSessionIdFromURL()) {
                // This is not optimal, but as this is not very common, it
                // shouldn't matter
                redirectPath = redirectPath + ";" +
                        SessionConfig.getSessionUriParamName(
                            request.getContext()) +
                    "=" + request.getRequestedSessionId();
            }
            if (query != null) {
                // This is not optimal, but as this is not very common, it
                // shouldn't matter
                redirectPath = redirectPath + "?" + query;
            }
            response.sendRedirect(redirectPath);
            request.getContext().logAccess(request, response, 0, true);
            return false;
        }

        // Filter trace method
        if (!connector.getAllowTrace()
                && req.method().equalsIgnoreCase("TRACE")) {
            Wrapper wrapper = request.getWrapper();
            String header = null;
            if (wrapper != null) {
                String[] methods = wrapper.getServletMethods();
                if (methods != null) {
                    for (int i=0; i<methods.length; i++) {
                        if ("TRACE".equals(methods[i])) {
                            continue;
                        }
                        if (header == null) {
                            header = methods[i];
                        } else {
                            header += ", " + methods[i];
                        }
                    }
                }
            }
            res.setStatus(405);
            res.addHeader("Allow", header);
            res.setMessage("TRACE method is not allowed");
            request.getContext().logAccess(request, response, 0, true);
            return false;
        }

        doConnectorAuthenticationAuthorization(req, request);

        return true;
    }
```



在 Service 内只有一个 Engine ，但可能有多个 Connector ，在 Engine 内部 Engine 和 Host ，Host 和 Context，Context 和 Wrapper 都是一对多的关系。但浏览器发出一次请求连接并不需要也不可能让部署在 Tomcat 中的所有 Web 应用的所有 Servlet 类都执行一遍，本文所说的 Map 机制就是为了 Connector 在接收到一次 Socket 连接时转化成请求后，能够找到 Engine 下具体哪个 Host、哪个 Context、哪个 Wrapper来执行这个请求。

StandardPipeline内部有三个成员变量basic（Valve）、container（Container）、first（Valve）

StandardPipeline
```
@Override
    public void addValve(Valve valve) {

        // Validate that we can add this Valve
        if (valve instanceof Contained)
            ((Contained) valve).setContainer(this.container);

        // Start the new component if necessary
        if (getState().isAvailable()) {
            if (valve instanceof Lifecycle) {
                try {
                    ((Lifecycle) valve).start();
                } catch (LifecycleException e) {
                    log.error("StandardPipeline.addValve: start: ", e);
                }
            }
        }

        // Add this Valve to the set associated with this Pipeline
//每次给管道添加一个普通阀的时候如果管道内原来没有普通阀则将新添加的阀作为该管道的成员变量 first 的引用，如果管道内已有普通阀，则把新加的阀加到所有普通阀链条末端，并且将该阀的下一个阀的引用设置为管道的基础阀
        if (first == null) {
            first = valve;
            valve.setNext(basic);
        } else {
            Valve current = first;
            while (current != null) {
                if (current.getNext() == basic) {
                    current.setNext(valve);
                    valve.setNext(basic);
                    break;
                }
                current = current.getNext();
            }
        }

        container.fireContainerEvent(Container.ADD_VALVE_EVENT, valve);
    }

//如果管道中有普通阀则返回普通阀链条最开始的那个，否则就返回基础阀。
    @Override
    public Valve getFirst() {
        if (first != null) {
            return first;
        }

        return basic;
    }
```
Valve共分为两类，一类叫基础阀（通过 getBasic、setBasic 方法调用），一类是普通阀（通过 addValve、removeValve 调用）。管道都是包含在一个容器当中，所以 API 里还有 getContainer 和 setContainer 方法。一个管道一般有一个基础阀（通过 setBasic 添加），可以有 0 到多个普通阀（通过 addValve 添加）。

如果想加普通阀`<Valve className="org.apache.catalina.authenticator.SingleSignOn" />`


所以上面的`connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);`
就是StandardContextValve
```
@Override
    public final void invoke(Request request, Response response)
        throws IOException, ServletException {

        // Disallow any direct access to resources under WEB-INF or META-INF
        MessageBytes requestPathMB = request.getRequestPathMB();
        if ((requestPathMB.startsWithIgnoreCase("/META-INF/", 0))
                || (requestPathMB.equalsIgnoreCase("/META-INF"))
                || (requestPathMB.startsWithIgnoreCase("/WEB-INF/", 0))
                || (requestPathMB.equalsIgnoreCase("/WEB-INF"))) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        // Select the Wrapper to be used for this Request
//StandardWrapper
        Wrapper wrapper = request.getWrapper();
        if (wrapper == null || wrapper.isUnavailable()) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        // Acknowledge the request
        try {
            response.sendAcknowledgement();
        } catch (IOException ioe) {
            container.getLogger().error(sm.getString(
                    "standardContextValve.acknowledgeException"), ioe);
            request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, ioe);
            response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            return;
        }

        if (request.isAsyncSupported()) {
            request.setAsyncSupported(wrapper.getPipeline().isAsyncSupported());
        }
//这里又链式调用了StandardWrapperValve的invoke
        wrapper.getPipeline().getFirst().invoke(request, response);
    }
```



StandardWrapperValve

```
@Override
    public final void invoke(Request request, Response response)
        throws IOException, ServletException {

        // Initialize local variables we may need
        boolean unavailable = false;
        Throwable throwable = null;
        // This should be a Request attribute...
        long t1=System.currentTimeMillis();
        requestCount.incrementAndGet();
        StandardWrapper wrapper = (StandardWrapper) getContainer();
        Servlet servlet = null;
        Context context = (Context) wrapper.getParent();

...

      
//分配一个servlet实例
            if (!unavailable) {
                servlet = wrapper.allocate();
            }
...

        MessageBytes requestPathMB = request.getRequestPathMB();
        DispatcherType dispatcherType = DispatcherType.REQUEST;
        if (request.getDispatcherType()==DispatcherType.ASYNC) dispatcherType = DispatcherType.ASYNC;
        request.setAttribute(Globals.DISPATCHER_TYPE_ATTR,dispatcherType);
        request.setAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR,
                requestPathMB);
        // Create the filter chain for this request    
//ApplicationFilterChain filterChain = ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);
//最后调用了servlet 的FilterChain，然后是各个过滤器的doFilter
        ApplicationFilterChain filterChain =
                ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

...
//dofilter & servlet.service
          filterChain.doFilter(request.getRequest(),
                                    response.getResponse());
...
        // Release the filter chain (if any) for this request
        if (filterChain != null) {
            filterChain.release();
        }

//解除
            if (servlet != null) {
                wrapper.deallocate(servlet);
            }

        // 如果这个servlet永久不可用,卸载和释放这个实例
        // unload it and release this instance
...

    }
```


ApplicationFilterFactory
```
public static ApplicationFilterChain createFilterChain(ServletRequest request,
            Wrapper wrapper, Servlet servlet) {
...
        filterChain.setServlet(servlet);
        filterChain.setServletSupportsAsync(wrapper.isAsyncSupported());
...
        return filterChain;
    }
```

ApplicationFilterChain
```
    @Override
    public void doFilter(ServletRequest request, ServletResponse response)
        throws IOException, ServletException {
...
internalDoFilter(req,res);
...
}

private void internalDoFilter(ServletRequest request,
                                  ServletResponse response)
        throws IOException, ServletException {
...
filter.doFilter(request, response, this);
...
            if ((request instanceof HttpServletRequest) &&
                    (response instanceof HttpServletResponse) &&
                    Globals.IS_SECURITY_ENABLED ) {
                final ServletRequest req = request;
                final ServletResponse res = response;
                Principal principal =
                    ((HttpServletRequest) req).getUserPrincipal();
                Object[] args = new Object[]{req, res};
                SecurityUtil.doAsPrivilege("service",
                                           servlet,
                                           classTypeUsedInService,
                                           args,
                                           principal);
            } else {
                servlet.service(request, response);
            }
...
}
```

很明显最后就调用到service了，后续大家应该很了解


![sequencediagram22](https://user-images.githubusercontent.com/7789698/37269633-5e197dc2-2606-11e8-8dbb-74a52180de09.png)