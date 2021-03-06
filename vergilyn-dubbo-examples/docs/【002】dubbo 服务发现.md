# 【002】consumer 启动过程

+ [服务引用（服务发现）](http://dubbo.apache.org/zh-cn/docs/source_code_guide/refer-service.html)

```text
dubbo://127.0.0.1:20880/com.vergilyn.examples.api.ProviderFirstApi
    ?anyhost=true
    &application=dubbo-consumer-application
    &category=providers
    &check=false
    &codec=dubbo
    &deprecated=false
    &dubbo=2.0.2
    &dynamic=true
    &generic=false
    &heartbeat=60000
    &init=false
    &interface=com.vergilyn.examples.api.ProviderFirstApi
    &logger=log4j2
    &methods=sayHello,sayGoodbye
    &path=com.vergilyn.examples.api.ProviderFirstApi
    &pid=4572&protocol=dubbo
    &register.ip=127.0.0.1
    &release=2.7.6.RELEASE
    &remote.application=dubbo-provider-application
    &revision=1.0.0
    &side=consumer
    &sticky=false
    &timeout=500000
    &timestamp=1586742746618
    &version=1.0.0
```

## 2. `vergilyn-consumer-examples`

### 2.1 FAQ

#### 2.1.1 registry-center: NACOS, `No provider available...`
```
Caused by: java.lang.IllegalStateException: 
Failed to check the status of the service com.vergilyn.examples.api.ProviderServiceApi. 
No provider available for the service com.vergilyn.examples.api.ProviderServiceApi:1.0.0 
from the url nacos://127.0.0.1:8848/org.apache.dubbo.registry.RegistryService
    ?application=dubbo-consumer-application&dubbo=2.0.2&init=false
    &interface=com.vergilyn.examples.api.ProviderServiceApi
    &methods=sayHello,sayGoodbye&pid=7620&register.ip=127.0.0.1
    &revision=1.0.0&side=consumer
    &sticky=false&timeout=2000&timestamp=1583814014425
    &version=1.0.0 
to the consumer 127.0.0.1 use dubbo version 

	at org.apache.dubbo.config.ReferenceConfig.createProxy(ReferenceConfig.java:349) ~[classes/:na]
	at org.apache.dubbo.config.ReferenceConfig.init(ReferenceConfig.java:258) ~[classes/:na]
	at org.apache.dubbo.config.ReferenceConfig.get(ReferenceConfig.java:158) ~[classes/:na]
	at org.apache.dubbo.config.spring.beans.factory.annotation.ReferenceAnnotationBeanPostProcessor.getOrCreateProxy(ReferenceAnnotationBeanPostProcessor.java:274) ~[classes/:na]
	at org.apache.dubbo.config.spring.beans.factory.annotation.ReferenceAnnotationBeanPostProcessor.doGetInjectedBean(ReferenceAnnotationBeanPostProcessor.java:143) ~[classes/:na]
```

+ [issues#5871](https://github.com/apache/dubbo/issues/5871)
+ [issues#5885](https://github.com/apache/dubbo/issues/5885): 个人提的issues

dubbo中会有很多原因导致"no provider available"，个人实际遇到的是：**因为nacos与dubbo v2.7.6的bug导致。**

**异常原因分析过程**

1. 首先，确保provider已成功注册到nacos。

2. 根据异常堆栈信息找到异常位置 `ReferenceConfig#createProxy(...)`
```java
public class ReferenceConfig<T> extends ReferenceConfigBase<T> {

    private T createProxy(Map<String, String> map) {
        // 省略...

        if (urls.size() == 1) {
            // 获取 invoker 是相当重要的，也是造成"no provider available"的根本原因
            invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
        }
        // 省略...

        // check, 默认 true
        if (shouldCheck() && !invoker.isAvailable()) {
                    throw new IllegalStateException("Failed to check the status of the service "
                            + interfaceName
                            + ". No provider available for the service "
                            + (group == null ? "" : group + "/")
                            + interfaceName +
                            (version == null ? "" : ":" + version)
                            + " from the url "
                            + invoker.getUrl()
                            + " to the consumer "
                            + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
    }
}
```

由源码可知，造成该exception的原因：shouldCheck() 返回 true，且 invoker.isAvailable() 返回false。

尝试修改`check = false`（<dubbo:reference check="false"/>），此时仍然报"no provider available"，但异常堆栈信息不同。
```text
Caused by: org.apache.dubbo.rpc.RpcException: No provider available from registry localhost:8848 
for service com.vergilyn.examples.api.ProviderServiceApi:1.0.0 on consumer 127.0.0.1 use dubbo version 4.0.9, 
please check status of providers(disabled, not registered or in blacklist).
	at org.apache.dubbo.registry.integration.RegistryDirectory.doList(RegistryDirectory.java:599) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.directory.AbstractDirectory.list(AbstractDirectory.java:75) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker.list(AbstractClusterInvoker.java:291) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(AbstractClusterInvoker.java:256) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.interceptor.ClusterInterceptor.intercept(ClusterInterceptor.java:47) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.support.wrapper.AbstractCluster$InterceptorInvokerNode.invoke(AbstractCluster.java:92) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(MockClusterInvoker.java:82) ~[classes/:na]
	at org.apache.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(InvokerInvocationHandler.java:74) ~[classes/:na]
	at org.apache.dubbo.common.bytecode.proxy0.sayHello(proxy0.java) ~[classes/:na]
	at com.vergilyn.examples.ConsumerExamplesApplication.run(ConsumerExamplesApplication.java:30) [classes/:na]
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:784) ~[spring-boot-2.2.2.RELEASE.jar:2.2.2.RELEASE]
	... 3 common frames omitted
```

此时注意，通过provider/nacos可知，启动provider时registry的 serviceName = "com.vergilyn.examples.api.ProviderServiceApi:1.0.0:"。
但是堆栈信息中提到的 service = "com.vergilyn.examples.api.ProviderServiceApi:1.0.0"，末尾缺少一个":"。
这其实是很关键的信息，我也是在了解了具体原因后，再回来看这些异常信息发现这是如此重要！！！
继续往后分析...

因为"check = false"，所以现在跟踪查看 `RegistryDirectory#isAvailable()`：  
```java
package org.apache.dubbo.registry.integration;

public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener {

    // 返回false时，exception: No provider available        
    @Override
    public boolean isAvailable() {
        if (isDestroyed()) {
            return false;
        }
        // urlInvokerMap == null, return false
        Map<String, Invoker<T>> localUrlInvokerMap = urlInvokerMap;
        if (localUrlInvokerMap != null && localUrlInvokerMap.size() > 0) {
            for (Invoker<T> invoker : new ArrayList<>(localUrlInvokerMap.values())) {
                if (invoker.isAvailable()) {
                    return true;
                }
            }
        }
        return false;
    }
}
```

根据debug可知`urlInvokerMap == EMPTY`，所以`return false`。

**现在的问题，变成了为什么`urlInvokerMap == EMPTY`？**  

带着这个疑问继续往后看，回到修改成"check = false"时后的异常对战信息，查看为什么抛出异常：
```java
public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener {
    private volatile boolean forbidden = false;
    
    @Override
    public List<Invoker<T>> doList(Invocation invocation) {
        if (forbidden) {  // true
            // 1. No service provider 2. Service providers are disabled
            throw new RpcException(RpcException.FORBIDDEN_EXCEPTION, "No provider available from registry " +
                    getUrl().getAddress() + " for service " + getConsumerUrl().getServiceKey() + " on consumer " +
                    NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() +
                    ", please check status of providers(disabled, not registered or in blacklist).");
        }

        // 省略...
        return invokers == null ? Collections.emptyList() : invokers;
    }
}
```

由源码可知，是因为`forbidden == true`。  
那么，**为什么 forbidden 会被设置成 true？**  

通过源码可知（find usages），只有 1个 地方会修改 forbidden：  
```java
public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener {

    private void refreshInvoker(List<URL> invokerUrls) {
        if (invokerUrls.size() == 1
                    && invokerUrls.get(0) != null
                    && EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {

            this.forbidden = true; // Forbid to access
            this.invokers = Collections.emptyList();
            routerChain.setInvokers(this.invokers);
            destroyAllInvokers(); // Close all invokers

        } else {
            this.forbidden = false; // Allow to access
        
        }
    }
}
```

通过debug可知，**会出现`protocol = empty`***，例如 url：
```text
# url
empty://127.0.0.1/com.vergilyn.examples.api.ProviderServiceApi
    ?application=dubbo-consumer-application
    &category=providers
    &check=false&dubbo=2.0.2&init=false
    &interface=com.vergilyn.examples.api.ProviderServiceApi
    &methods=sayHello,sayGoodbye
    &pid=13780&release=4.0.9&revision=1.0.0
    &side=consumer&sticky=false
    &timeout=2000&timestamp=1584498455807&version=1.0.0
```

**为什么会出现 protocol "empty://..." 调用 refreshInvoker() ？**

这涉及到dubbo的服务调用过程，consumer 启动时扫描到`@Reference`，会创建proxy，即`org.apache.dubbo.config.ReferenceConfig#createProxy(Map)`。
```java
package org.apache.dubbo.config;
public class ReferenceConfig<T> extends ReferenceConfigBase<T> {

    private T createProxy(Map<String, String> map) {
        if (urls.size() == 1) {
            invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
        }
    }
}
```

其调用链：  
```text
ReferenceConfig.createProxy()
            ↓
RegistryProtocol.refer()
            ↓
RegistryProtocol.doRefer()
            ↓
RegistryDirectory.subscribe()
            ↓
FailbackRegistry.subscribe()
            ↓
NacosRegistry.doSubscribe()
```

关键代码：
```java
package org.apache.dubbo.registry.nacos;

public class NacosRegistry extends FailbackRegistry {
    @Override
    public void doSubscribe(final URL url, final NotifyListener listener) {
        Set<String> serviceNames = getServiceNames(url, listener);
        doSubscribe(url, listener, serviceNames);
    }

    private void doSubscribe(final URL url, final NotifyListener listener, final Set<String> serviceNames) {
        execute(namingService -> {
            List<Instance> instances = new LinkedList();
            for (String serviceName : serviceNames) {
                instances.addAll(namingService.getAllInstances(serviceName));
                subscribeEventListener(serviceName, url, listener);
            }
            notifySubscriber(url, listener, instances);
        });
    }

    private void subscribeEventListener(String serviceName, final URL url, final NotifyListener listener)
            throws NacosException {
        EventListener eventListener = event -> {
            if (event instanceof NamingEvent) {
                NamingEvent e = (NamingEvent) event;
                notifySubscriber(url, listener, e.getInstances());
            }
        };
        namingService.subscribe(serviceName, eventListener);
    }

    private void notifySubscriber(URL url, NotifyListener listener, Collection<Instance> instances) {
        List<Instance> healthyInstances = new LinkedList<>(instances);
        if (healthyInstances.size() > 0) {
            // Healthy Instances
            filterHealthyInstances(healthyInstances);
        }

        List<URL> urls = toUrlWithEmpty(url, healthyInstances);

        NacosRegistry.this.notify(url, listener, urls);
    }


    private Set<String> getServiceNames(URL url, NotifyListener listener) {
        if (isAdminProtocol(url)) {
            // ...
        } else {
            return getServiceNames0(url);
        }
    }

    private Set<String> getServiceNames0(URL url) {
        NacosServiceName serviceName = createServiceName(url);

        final Set<String> serviceNames;

        if (serviceName.isConcrete()) { // is the concrete service name
            serviceNames = new LinkedHashSet<>();
            serviceNames.add(serviceName.toString());

            /* vergilyn-question, 2020-03-18 >>>> 注释，由于"no provider "
             *   https://github.com/apache/dubbo/issues/5871
             *   https://github.com/apache/dubbo/issues/5885
             */
            // Add the legacy service name since 2.7.6
            // serviceNames.add(getLegacySubscribedServiceName(url));
        } else {
            serviceNames = filterServiceNames(serviceName);
        }

        return serviceNames;
    }
}
```

因为 legacy-subscribed 会创建一个serviceNames = "com.vergilyn.examples.api.ProviderServiceApi:1.0.0"。  
provider并未提供该service，其提供的是"com.vergilyn.examples.api.ProviderServiceApi:1.0.0:" （最后的 冒号）。 

因为subscribe了naocs，但是实际不存在"com.vergilyn.examples.api.ProviderServiceApi:1.0.0"，所以 `instances == EMPTY`。  
正式由于此，导致创建了一个empty的protocol，进而导致后面 refreshInvokers() 时满足特定导致 invokers#destroy() 并将 invokers 赋值 empty。  

...至此，前面提到的问题其实都得到了解决。  
2020-03-19 >>>> 暂时解决方案，把legacy-subscribe-services-name注释掉，让其不生成无效的 serviceName。  

> 创建 empty protocol 目的
> https://github.com/apache/dubbo/issues/5871
> 生成empty的protocol是正常的。否则会出现服务端全部下线，但是客户端还有一个订阅的服务端的代理一直存在的情况


备注：以上这个过程就是 consumer-side 创建 service-proxy的过程。

#### 2.1.2 "Invoke remote method timeout."
1. provider 已启动，并在nacos注册了正确的service。
2. `telnet 127.0.0.1 20880`正常（provider中配置的 dubbo/netty 通信端口）
3. 分析是 consumer-side 还是 provider-side 出现问题（个人遇到的是 provider忘记依赖 hessian2）

#### 2.1.3 netty 对象实例数/连接数？
consumer 在启动时 扫描到 `@Reference`时会创建 invoker，进而创建 netty-bootstrap。

```java
package org.apache.dubbo.config;

public class ReferenceConfig<T> extends ReferenceConfigBase<T> {
    
    private static final Protocol REF_PROTOCOL = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    
    private T createProxy(Map<String, String> map) {
        invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
    }
}
```

由声明可知，Protocol(DubboProtocol) 是 静态常量。  
每次调用 `refer()` 会 `new AsyncToSyncInvoker()`进而 `new DubboInvoker()`。  
但是，其中的核心`DubboProtocol#getClients()`，因为默认是 share-connection :  
```java
package org.apache.dubbo.rpc.protocol.dubbo;

public class DubboProtocol extends AbstractProtocol {
    /**
     * <host:port,Exchanger>
     */
    private final Map<String, List<ReferenceCountExchangeClient>> referenceClientMap = new ConcurrentHashMap<>();

    private List<ReferenceCountExchangeClient> getSharedClient(URL url, int connectNum) {
        String key = url.getAddress();

        // 获取带有“引用计数”功能的 ExchangeClient
        List<ReferenceCountExchangeClient> clients = referenceClientMap.get(key);

        if (checkClientCanUse(clients)) {
            batchClientRefIncr(clients);
            return clients;
        }
        // 省略...
    }
}
```

由上一步知道，其实 DubboProtocol 是同一个对象，所以可以`referenceClientMap`即client缓存。  
所以，针对不同 address 会创建多个 Netty.Bootstrap （并且会 立即connect，）。
（如果 lazy，Bootstrap 和 connect 都会在第一次请求时再执行）

**扩展：**  
如果 address 不存在，则会 initClient （DubboProtocol#initClient(url)）。  
如果 non-lazy，那么则会 constructor `NettyClient extends AbstractClient `。 

`AbstractClient` 的构造函数是一个 模版方法。
```java
package org.apache.dubbo.remoting.transport;

public abstract class AbstractClient extends AbstractEndpoint implements Client {

    public AbstractClient(URL url, ChannelHandler handler) throws RemotingException {
        super(url, handler);
        
        needReconnect = url.getParameter(Constants.SEND_RECONNECT_KEY, false);
        
        initExecutor(url);
        
        /* vergilyn-comment, 2020-03-20 >>>> 模版方法
         *   例如 netty4.NettyClient#doOpen() `new Netty.Bootstrap()`
         */
        doOpen();

        /* vergilyn-comment, 2020-03-20 >>>> 模版方法，实际调用子类的 #doConnect
         *   例如 netty4.NettyClient#doConnect()
         *   通过 doOpen() 构造的 NettyBootstrap，创建其连接 `bootstrap.connect(getConnectAddress())`。
         *   这一步，consumer 与 provider 已经创建了connect （通过 wireshark 可知 3次握手 已经完成）
         */
        connect();
    }
}
```

#### 2.1.4 "consumer invoke timeout"

+ [issues#6004, "first call time consumption too long"](https://github.com/apache/dubbo/issues/6004)

例如当provider启动成功后（成功注册服务到nacos），再启动consumer，`@Reference(class = ProviderFirstApi.class, timeout = 1000, retries = 0)`第一次调用会出现timeout：  
```
>>>> consumer
@Reference(class = ProviderFirstApi.class, timeout = 1000, retries = 0)

sayHello("vergilyn", 1000);

>>>> provider
@org.apache.dubbo.config.annotation.Service
public class ProviderFirstApiImpl implements ProviderFirstApi {
    @Override
    public String sayHello(String name, long sleepMs) {
        LocalTime begin = LocalTime.now();
        TimeUnit.MILLISECONDS.sleep(sleepMs);
        LocalTime end = LocalTime.now();

        String result = String.format("[%s][%s][%s][%s] >>>> Hello, %s",
                serviceName, this.getClass().getSimpleName(),
                begin.toString(), end.toString(), name);

        log.info("result >>>> {}", result);
        return result;
    }
}


>>>> log
Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2020-04-20 17:05:25.990  WARN 17980 --- [:20880-thread-1] o.a.d.r.exchange.support.DefaultFuture   :  [DUBBO] The timeout response finally returned at 2020-04-20 17:05:25.990, 
response Response [id=0, version=null, status=20, event=false, error=null, result=AppResponse [value=[dubbo-provider-application][ProviderFirstApiImpl] >>>>>>>> Hello, vergilyn, exception=null]], channel: /127.0.0.1:54284 -> /127.0.0.1:20880, dubbo version: 2.7.6.RELEASE, current host: 127.0.0.1

2020-04-20 17:05:26.388 ERROR 17980 --- [           main] o.s.boot.SpringApplication               : Application run failed

java.lang.IllegalStateException: Failed to execute CommandLineRunner
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:787) ~[spring-boot-2.2.2.RELEASE.jar:2.2.2.RELEASE]
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:768) ~[spring-boot-2.2.2.RELEASE.jar:2.2.2.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:322) ~[spring-boot-2.2.2.RELEASE.jar:2.2.2.RELEASE]
	at com.vergilyn.examples.ConsumerExamplesApplication.main(ConsumerExamplesApplication.java:28) [classes/:na]

Caused by: org.apache.dubbo.rpc.RpcException: Failed to invoke the method sayHello in the service com.vergilyn.examples.api.ProviderFirstApi. 
  Tried 1 times of the providers [127.0.0.1:20880] (1/1) from the registry localhost:8848 on the consumer 127.0.0.1 using the dubbo version 2.7.6.RELEASE. 
  Last error is: Invoke remote method timeout. method: sayHello, 
  provider: dubbo://127.0.0.1:20880/com.vergilyn.examples.api.ProviderFirstApi
  ?anyhost=true&application=dubbo-consumer-application&category=providers
  &check=false&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&init=false
  &interface=com.vergilyn.examples.api.ProviderFirstApi&logger=log4j2&methods=sayHello,sayGoodbye
  &path=com.vergilyn.examples.api.ProviderFirstApi&pid=17980&protocol=dubbo
  &register.ip=127.0.0.1&release=2.7.6.RELEASE&remote.application=dubbo-provider-application&retries=0
  &revision=1.0.0&side=consumer&sticky=false&timeout=1000&timestamp=1587372208069&version=1.0.0, 
cause: org.apache.dubbo.remoting.TimeoutException: Waiting server-side response timeout by scan timer. start time: 2020-04-20 17:05:24.874, end time: 2020-04-20 17:05:25.898, 
  client elapsed: 96 ms, server elapsed: 927 ms, timeout: 1000 ms, 
  request: Request [id=0, version=2.0.2, twoway=true, event=false, broken=false, data=null], channel: /127.0.0.1:54284 -> /127.0.0.1:20880
	at org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker.doInvoke(FailoverClusterInvoker.java:113) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(AbstractClusterInvoker.java:264) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.interceptor.ClusterInterceptor.intercept(ClusterInterceptor.java:51) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.support.wrapper.AbstractCluster$InterceptorInvokerNode.invoke(AbstractCluster.java:96) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(MockClusterInvoker.java:86) ~[classes/:na]
	at org.apache.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(InvokerInvocationHandler.java:96) ~[classes/:na]
	at org.apache.dubbo.common.bytecode.proxy0.sayHello(proxy0.java) ~[classes/:na]
	at com.vergilyn.examples.ConsumerExamplesApplication.run(ConsumerExamplesApplication.java:33) [classes/:na]
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:784) ~[spring-boot-2.2.2.RELEASE.jar:2.2.2.RELEASE]
	... 3 common frames omitted

Caused by: java.util.concurrent.ExecutionException: org.apache.dubbo.remoting.TimeoutException: 
  Waiting server-side response timeout by scan timer. start time: 2020-04-20 17:05:24.874, end time: 2020-04-20 17:05:25.898, 
  client elapsed: 96 ms, server elapsed: 927 ms, timeout: 1000 ms, 
  request: Request [id=0, version=2.0.2, twoway=true, event=false, broken=false, data=null], channel: /127.0.0.1:54284 -> /127.0.0.1:20880
	at java.util.concurrent.CompletableFuture.reportGet(CompletableFuture.java:357) ~[na:1.8.0_171]
	at java.util.concurrent.CompletableFuture.get(CompletableFuture.java:1915) ~[na:1.8.0_171]
	at org.apache.dubbo.rpc.AsyncRpcResult.get(AsyncRpcResult.java:181) ~[classes/:na]
	at org.apache.dubbo.rpc.protocol.AsyncToSyncInvoker.invoke(AsyncToSyncInvoker.java:79) ~[classes/:na]
	at org.apache.dubbo.monitor.support.MonitorFilter.invoke(MonitorFilter.java:89) ~[classes/:na]
	at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:81) ~[classes/:na]
	at org.apache.dubbo.rpc.protocol.dubbo.filter.FutureFilter.invoke(FutureFilter.java:51) ~[classes/:na]
	at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:81) ~[classes/:na]
	at org.apache.dubbo.rpc.filter.ConsumerContextFilter.invoke(ConsumerContextFilter.java:55) ~[classes/:na]
	at org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:81) ~[classes/:na]
	at org.apache.dubbo.rpc.listener.ListenerInvokerWrapper.invoke(ListenerInvokerWrapper.java:78) ~[classes/:na]
	at org.apache.dubbo.rpc.protocol.InvokerWrapper.invoke(InvokerWrapper.java:56) ~[classes/:na]
	at org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker.doInvoke(FailoverClusterInvoker.java:82) ~[classes/:na]
	... 11 common frames omitted

Caused by: org.apache.dubbo.remoting.TimeoutException: 
  Waiting server-side response timeout by scan timer. start time: 2020-04-20 17:05:24.874, end time: 2020-04-20 17:05:25.898, 
  client elapsed: 96 ms, server elapsed: 927 ms, timeout: 1000 ms, 
  request: Request [id=0, version=2.0.2, twoway=true, event=false, broken=false, data=null], channel: /127.0.0.1:54284 -> /127.0.0.1:20880
	at org.apache.dubbo.remoting.exchange.support.DefaultFuture.doReceived(DefaultFuture.java:223) ~[classes/:na]
	at org.apache.dubbo.remoting.exchange.support.DefaultFuture.received(DefaultFuture.java:188) ~[classes/:na]
	at org.apache.dubbo.remoting.exchange.support.DefaultFuture$TimeoutCheckTask.notifyTimeout(DefaultFuture.java:311) ~[classes/:na]
	at org.apache.dubbo.remoting.exchange.support.DefaultFuture$TimeoutCheckTask.lambda$run$0(DefaultFuture.java:298) ~[classes/:na]
	at org.apache.dubbo.common.threadpool.ThreadlessExecutor.waitAndDrain(ThreadlessExecutor.java:93) ~[classes/:na]
	at org.apache.dubbo.rpc.AsyncRpcResult.get(AsyncRpcResult.java:179) ~[classes/:na]
	... 21 common frames omitted
```

然而，如果允许重试`retries >= 1`，那么其实最终（**可能**）会成功！

**低级错误，调用的provider service 存在`TimeUnit.SECONDS.sleep(1);`，所以 elapsed-time > timeout!**

但是new-issues，为什么retries会成功？
如果 retries 是新的(netty)request，那么provider 执行花费也会是 elapsed-time > timeout！
通过查看 provider 代码，确实执行了2次method，并且 provider 确实sleep了1s。（本来猜测可能sleep(1s) 实际可能低于1s）

retries code:
- `FailoverClusterInvoker#doInvoke()`, retry logic
- `DefaultFuture#newFuture()` -> `DefaultFuture#timeoutCheck()`, timeout check
```
future.timeoutCheckTask = 
            new HashedWheelTimer(
                new NamedThreadFactory("dubbo-future-timeout", true),
                tickDuration:30,
                TimeUnit.MILLISECONDS
            )
            .newTimeout(new TimeoutCheckTask(future.getId()), delay: 1000, TimeUnit.MILLISECONDS)
```

client start-time:  
  `HeaderExchangeChannel#request(Object, int, ExecutorService)` 
  -> `DefaultFuture#newFuture()` 
  -> `DefaultFuture#timeoutCheck()` 
  -> DefaultFuture.start = System.currentTimeMillis();
  
client end-time:  
  `DefaultFuture#getTimeoutMessage()`
  -> end-time = System.currentTimeMillis();

client sent-time:  
  `netty4.NettyClientHandler#write()` 
  -listener-> `HeaderExchangeHandler#sent()` 
  -> `DefaultFuture#doSent()` 
  -> DefaultFuture.sent = System.currentTimeMillis();

注意：因为异步调用，所以个人打印的LocalTime 其实是一起输出的。
```text
>>>> first request
FailoverClusterInvoker#doInvoke() before `invoker.invoke()` >>>> 15:19:11.382
HeaderExchangeChannel#request() before `newFuture` >>>> 15:19:11.385
HeaderExchangeChannel#request() after `newFuture` >>>> 15:19:11.389             4 ms, <=> start-time
NettyClientHandler#Override() before `super.write()` >>>> 15:19:11.395
NettyClientHandler#Override() after `super.write()` >>>> 15:19:11.468           73 ms, 主要时间差
NettyClientHandler#Override() before `promise.addListener()` >>>> 15:19:11.469
NettyClientHandler#sent() sent before >>>> 15:19:11.473                         4 ms, <=> sent-time
NettyClientHandler#sent() sent after >>>> 15:19:11.473                          

>>>> first retry
FailoverClusterInvoker#doInvoke() before `invoker.invoke()` >>>> 15:19:12.412
HeaderExchangeChannel#request() before `newFuture` >>>> 15:19:12.412
HeaderExchangeChannel#request() after `newFuture` >>>> 15:19:12.412             0 ms, <=> start-time
NettyClientHandler#Override() before `super.write()` >>>> 15:19:12.412
NettyClientHandler#Override() after `super.write()` >>>> 15:19:12.413           1 ms, 主要时间差
NettyClientHandler#Override() before `promise.addListener()` >>>> 15:19:12.413
NettyClientHandler#sent() sent before >>>> 15:19:12.413                         0 ms, <=> sent-time
NettyClientHandler#sent() sent after >>>> 15:19:12.413

>>>> first request
Caused by: org.apache.dubbo.remoting.TimeoutException: 
  Waiting server-side response timeout by scan timer. 
  start time: 2020-04-23 15:19:11.388, end time: 2020-04-23 15:19:12.410, sent time: 2020-04-23 15:19:11.473, 
  client elapsed: 85 ms, server elapsed: 937 ms, timeout: 1000 ms, 
  request: ...

>>>> first retry
Caused by: org.apache.dubbo.remoting.TimeoutException: 
  Waiting server-side response timeout by scan timer. 
  start time: 2020-04-23 15:19:12.412, end time: 2020-04-23 15:19:13.429, sent time: 2020-04-23 15:19:12.413, 
  client elapsed: 1 ms, server elapsed: 1016 ms, timeout: 1000 ms, 
  request: ...

```

由上可知，最主要的时间差在：`netty4.NettyClientHandler#write()` 中的 `super.write()`，即netty.channel.write() 第1次 和 第2+次存在执行时间差距!
所以，**首次请求时**多消耗约84ms在client部分，**重试时**这部分时间被用于server部分!

