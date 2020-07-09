---
title: "Step by Step写一个简易的RPC"
date: 2019-07-09 11:06:43
tags:
- infra
- rpc
categories:
- Technology
---

## 背景
在Cloud Native大火的今天，我们想来讲讲在这个体系下比较重要的东西RPC-framework,为什么需要有这个东西呢，需要来讲讲历史了；
最开始代码组织的方式，最原始就是所有code都在一个repo里面，高内聚模式，在业务比较小的时候，这没什么问题！但是到了业务快速发展的时候就会
带来致命的缺陷了：
1. 敏捷性
    1. code全部写在一个repo里面，导致每次开发/测试/上线，编译会花很多时间
    2. 不同模块间的相互影响，比如模块A需要上线，依赖模块B，模块B由于比较复杂需要很多时间进行测试，导致功能很简单的A迭代速度强依赖于B，无法隔离
2. 扩展性
    1. 不同的模块，承载的业务不同，模块A需要很多机器来承载流量，而模块B却不需要这么多机器，导致很多无用的code跑在机器上占用资源
    2. 不同模块的SLA要求，不一样，一个999的模块和一个99的模块放在一起显然不合适
    3. 不同模块的属性不同，有的模块无状态，可以水平扩容，有的模块有状态，无法水平扩容，揉在一起，增加复杂性

现阶段的解决方案是，将一整块的拼图，打碎，划分为，一个个独立的个体，他们相互不影响，各自负责好自己的事情即可，引出来的架构叫SOA(`S`ervice-`O`riented `A`rchitecture),
里面很重要的一个能力就是支持，服务间调用，走远端的模式就是RPC，下面讲讲，常见RPC的架构体系

## INFRA
大部分的RPC主要分为三个部分，客户端，服务端，注册中心
![](/image/rpc/easy_rpc.svg)
* 客户端
    * 客户端主要是服务的调用方，他有几种operation:1.从注册中心拉去配置 2.和服务端建立连接 3.向服务端发起request
* 注册中心
    * 注册中心主要用来保存服务的元数据，比如1.现在有多少个service 2.每个service的provider有哪些 3.检测存活的provider 4.提供配置信息给客户端
* 服务端
    * 服务端是service的提供方，他主要做 1.启动服务并注册到注册中心 2.发送心跳，证明自己还活着 3.提供channel接收用户的request

## 实现
接下来是干活，整个rpc的实现部分，还是按照客户端，注册中心，服务端来讲，
整个repo`https://github.com/learn2Pro/easyrpc` 在此处
1. 注册中心
   * 注册中心的接口部分，提供一系列meta信息的管理
    ```$java
    public interface RpcConfigServer {
     RemoteAddr sense(String service) throws Throwable;
     void register(RemoteAddr addr, String service, Class<?> klass);
     void unregister(RemoteAddr addr);
     default CodecType fetchCodec() {
       return CodecType.JSON;
     }
    }
    主要有两种实现
    //本地注册中心（测试用）
    Class LocalConfigServer extends RpcConfigServer
    //Zk注册中心
    Class ZkConfigServer extends RpcConfigServer
    ```
2. 服务端
   * 注册到config server
   ```java
    public class ProviderRepository implements Repository, ApplicationContextAware, InitializingBean {
    ...
      //寻找provider注解，并注册到注册中心
      private void registerProvider(ApplicationContext app) {
        for (Map.Entry<String, Object> entry : app.getBeansWithAnnotation(Provider.class).entrySet()) {
          Class<?> klass = entry.getValue().getClass();
          Provider ann = klass.getAnnotation(Provider.class);
          String name =
              StringUtils.isEmpty(ann.value()) ? StringUtils.uncapitalize(klass.getSimpleName())
                  : ann.value();
          rpcConfigServer.register(RemoteAddr.local(), name, klass);
        }
      }
    
    ...
    }
    ```
   * 启动socket端口监听请求，使用netty-io
   ```java

   public class ProviderRouter implements InitializingBean {
   
     /**
      * logger instance
      */
     private static final Logger LOGGER = LoggerFactory.getLogger(ProviderRouter.class);
     @Autowired
     private RpcProviderHandler rpcProviderHandler;
     @Autowired
     private CodecEncodeHandler codecEncodeHandler;
     @Autowired
     private RpcConfigServer rpcConfigServer;
     private EventLoopGroup master;
     private EventLoopGroup workers;
   
     /**
      * start server for rpc provider
      *
      * @throws Exception init failed
      */
     @Override
     public void afterPropertiesSet() throws Exception {
       master = new NioEventLoopGroup(1);
       workers = new NioEventLoopGroup(10);
       RemoteAddr addr = RemoteAddr.local();
       new Thread(() -> {
         try {
           ServerBootstrap server = new ServerBootstrap();
           server.group(master, workers)
               .channel(NioServerSocketChannel.class)
               .localAddress(new InetSocketAddress(Integer.parseInt(addr.getPort())))
               .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 protected void initChannel(SocketChannel ch) throws Exception {
                   ch.pipeline()
                       .addLast("prepend", new LengthFieldPrepender(2))
                       .addLast("remove", new LengthFieldBasedFrameDecoder(65536, 0, 2, 0, 2))
                       .addLast("decoder", new CodecDecodeHandler(rpcConfigServer.fetchCodec(),
                           RpcRequest.class))
                       .addLast("encoder", codecEncodeHandler)
                       .addLast("rpc_provider", rpcProviderHandler);
                 }
               });
           ChannelFuture channelFuture = server.bind().sync();
           LOGGER.info(RpcProviderHandler.class.getName() +
               " started and listening for connections on " + channelFuture.channel().localAddress());
           channelFuture.channel().closeFuture().sync();
         } catch (Exception e) {
           master.shutdownGracefully();
           workers.shutdownGracefully();
         }
       }).start();
     }
   
     @PreDestroy
     public void destroy() throws InterruptedException {
       LOGGER.info("server shutdown going...");
       master.shutdownGracefully().sync();
       workers.shutdownGracefully().sync();
     }
   }
    ```
3. 消费端
    * 判断是call走近端还是远端,创建对应的proxy，整个过程是且契合到spring的lifecycle中
    ```java
    public class ConsumerBeanPostProcessor implements BeanPostProcessor,
        ApplicationContextAware {
    
      /**
       * the app instance
       */
      private ApplicationContext app;
      /**
       * remote rpc services
       */
      private Set<String> remoteServices = Sets.newHashSet();
    
      /**
       * register consumer for class
       *
       * @param targetName the instance name
       * @param inner      the inner class
       * @return
       */
      private Object createProxy(String targetName, Class<?> inner, ProviderType typo) {
        if (this.app.containsBean(targetName) && typo == ProviderType.LOCAL) {
          return Proxy
              .newProxyInstance(Thread.currentThread().getContextClassLoader(), new Class[]{inner},
                  new LocalInvocationWrapper(targetName, inner));
        } else {
          return Proxy
              .newProxyInstance(Thread.currentThread().getContextClassLoader(), new Class[]{inner},
                  new RemoteInvocationWrapper(targetName, inner));
        }
      }
    
      @Override
      public Object postProcessBeforeInitialization(Object bean, String beanName)
          throws BeansException {
        Class<?> target = bean.getClass();
        for (Field field : target.getDeclaredFields()) {
          //inject consumer
          if (field.getAnnotation(Consumer.class) != null && field.getType().isInterface()) {
            Consumer ann = field.getAnnotation(Consumer.class);
            String name = ann.value();
            String targetName =
                StringUtils.isEmpty(name) ? StringUtils.uncapitalize(field.getType().getSimpleName())
                    : name;
            if (ann.typo() == ProviderType.REMOTE || !this.app.containsBean(targetName)) {
              remoteServices.add(targetName);
            }
            Object instance = this.createProxy(targetName, field.getType(), ann.typo());
            field.setAccessible(true);
            ReflectionUtils.setField(field, bean, instance);
          }
        }
        return bean;
      }
    
      @Override
      public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.app = applicationContext;
      }
    
      /**
       * get all remote services
       *
       * @return service set
       */
      public Set<String> getRemoteServices() {
        return remoteServices;
      }
    }
    ```
   * 请求处理，此处使用了生产者消费者模型，通过队列的方式异步化，增加吞吐量，可以基于此做到控速和反压(backpush)
   ```java
   public interface RpcMsgPool {
   
     ExecutorService PROCESSORS = new ThreadPoolExecutor(5, 5, 0,
         TimeUnit.MICROSECONDS, new LinkedBlockingDeque<>(500));
     void answer(RpcResponse response);
     Future<RpcResponse> send(RpcRequest request);
   }
   public class AsyncRpcMsgPool implements RpcMsgPool, InitializingBean, DisposableBean {
   
     private static final AsyncRpcMsgPool INSTANCE = new AsyncRpcMsgPool();
     private static final BlockingQueue<RpcRequest> REQUEST_QUEUE = new LinkedBlockingQueue<>(
         10000);
     protected static final Map<String, RpcRequest> REQUEST_MAP = Maps.newConcurrentMap();
     protected static final Map<String, RpcResponse> RESPONSE_POOL = Maps
         .newConcurrentMap();
     private static final Logger LOGGER = LoggerFactory.getLogger(AsyncRpcMsgPool.class);
     /**
      * Lock held by take, poll, etc
      */
     private final ReentrantLock takeLock = new ReentrantLock();
   
     /**
      * Wait queue for waiting takes
      */
     private final Condition updated = takeLock.newCondition();
     private volatile boolean stop = false;
   
     @Autowired
     private ConsumerRouter consumerRouter;
   
     public static AsyncRpcMsgPool getInstance() {
       return INSTANCE;
     }
   
     private RpcResponse fetch(String requestId) throws InterruptedException {
       takeLock.lockInterruptibly();
       try {
         while (!RESPONSE_POOL.containsKey(requestId)) {
           updated.await();
         }
         RpcResponse response = RESPONSE_POOL.get(requestId);
         RESPONSE_POOL.remove(requestId);
         return response;
       } finally {
         takeLock.unlock();
       }
     }
   
     /**
      * fetch the specified response of request id
      *
      * @param requestId the request id
      * @return the response
      */
     private RpcResponse fetch(String requestId, long timeout)
         throws InterruptedException, TimeoutException {
       takeLock.lockInterruptibly();
       try {
         while (!RESPONSE_POOL.containsKey(requestId)) {
           if (!updated.await(timeout, TimeUnit.MILLISECONDS)) {
             throw new TimeoutException("fetch result timeout!");
           }
         }
         RpcResponse response = RESPONSE_POOL.get(requestId);
         RESPONSE_POOL.remove(requestId);
         REQUEST_MAP.remove(requestId);
         return response;
       } finally {
         takeLock.unlock();
       }
     }
   
     /**
      * send requst to server
      *
      * @param request the request info
      * @return the future of response
      */
     public Future<RpcResponse> send(RpcRequest request) {
       REQUEST_QUEUE.add(request);
       REQUEST_MAP.put(request.getSessionId(), request);
       return new RpcFuture(request.getSessionId());
     }
   
     @Override
     public void answer(RpcResponse response) {
       RESPONSE_POOL.put(response.getId(), response);
       signalNotEmpty();
     }
   
     public RpcRequest search(String sessionId) {
       return REQUEST_MAP.get(sessionId);
     }
   
     /**
      * Signals a waiting take. Called only from put/offer (which do not otherwise ordinarily lock
      * takeLock.)
      */
     private void signalNotEmpty() {
       final ReentrantLock takeLock = this.takeLock;
       takeLock.lock();
       try {
         updated.signalAll();
       } finally {
         takeLock.unlock();
       }
     }
   
     @Override
     @SuppressWarnings("unchecked")
     public void afterPropertiesSet() throws Exception {
       PROCESSORS.execute(() -> {
         try {
           for (; ; ) {
             if (stop) {
               return;
             }
             RpcRequest request = REQUEST_QUEUE.take();
             consumerRouter.choose(request.getServiceId()).writeAndFlush(request);
           }
         } catch (InterruptedException e) {
           LOGGER.error("this request fetch failed!", e);
         }
       });
     }
   
     @Override
     public void destroy() throws Exception {
       stop = true;
     }
   
     static final class RpcFuture implements Future<RpcResponse> {
   
       /**
        * the request id
        */
       private String requestId;
   
       public RpcFuture(String requestId) {
         this.requestId = requestId;
       }
   
       @Override
       public boolean cancel(boolean mayInterruptIfRunning) {
         return true;
       }
   
       @Override
       public boolean isCancelled() {
         return true;
       }
   
       @Override
       public boolean isDone() {
         return AsyncRpcMsgPool.RESPONSE_POOL.containsKey(requestId);
       }
   
       @Override
       public RpcResponse get() throws InterruptedException {
         return getInstance().fetch(requestId);
       }
   
       @Override
       public RpcResponse get(long timeout, TimeUnit unit)
           throws InterruptedException, ExecutionException, TimeoutException {
         long to = 0;
         switch (unit) {
           case DAYS:
             to = TimeUnit.DAYS.toMillis(timeout);
             break;
           case HOURS:
             to = TimeUnit.HOURS.toMillis(timeout);
             break;
           case MINUTES:
             to = TimeUnit.MINUTES.toMillis(timeout);
             break;
           case SECONDS:
             to = TimeUnit.SECONDS.toMillis(timeout);
             break;
           case MILLISECONDS:
             to = TimeUnit.MILLISECONDS.toMillis(timeout);
             break;
           case MICROSECONDS:
             to = TimeUnit.MICROSECONDS.toMillis(timeout);
             break;
           case NANOSECONDS:
             to = TimeUnit.NANOSECONDS.toMillis(timeout);
             break;
         }
         return getInstance().fetch(requestId, to);
       }
     }
   }
   ```
4. 工具类
    * encoding工具，实现了json和pb的序列化,因为网络传输的都是bytes
    ```java
    public interface CodecProcessor<T extends RpcSerModel> {
      byte[] encode(T data);
      T decode(byte[] bytes);
    }
    class JSONCodecProcessor extends CodecProcessor;
    class ProtoCodecProcessor extends CodecProcessor;
    ```
    * id生号器，每个请求会有唯一的标识，我们通过的是生号器去解决这个问题,uuid和snowflake两种算法
    ```java
    public interface Generator {
       String generateId();
    }
    class UUIDGenerator extends Generator;
    class SnowflakeGenerator extends Generator;
    ```
## 总结
通过上面一系列的工作，我们的简易RPC框架就完成了，实践是学习很好的途径，每个人在做的过程中，会提出不同的问题，有不同的创新；
分享出来是很好的成长！