### dubbo服务暴露源码分析

服务的暴露流程主要分为两个部分：
1. 服务的注册
1.1 流程描述  
当需要发布一个服务的时候就需要把该服务的信息注册到注册中心中，具体来说就是需要把用于消费者调用的协议注
册到注册中心中。以zookeeper为例，当provider启动的时候就会往zookeeper的providers节点下面写入dubbo协
议，具体是以接口粒度作为服务发布的基准。比如要发布一个cn.enjoy.service.UserService服务，则会
往/dubbo/cn.enjoy.service.UserService/providers下面写入dubbo协议，协议内容如下：
```shell
dubbo%3A%2F%2F192.168.8.32%3A20990%2Fcn.enjoy.service.UserService%3Fanyhost%3Dtrue%26appl
ication%3Ddubbo_provider%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26gener
ic%3Dfalse%26interface%3Dcn.enjoy.service.UserService%26metadatatype%3Dremote%26methods%3DdoKill%2CqueryUser%26pid%3D10760%26release%3D3.0.2.1%26retri
es%3D4%26revision%3D1.0-SNAPSHOT%26service-namemapping%3Dtrue%26side%3Dprovider%26threadpool%3Dfixed%26threads%3D100%26timeout%3D50
00%26timestamp%3D1634537762362
```
1.2源码分析：  
首先我们需要知道服务的注册是何时触发的，当spring容器启动完成后，会发布事件，由DubboBootstrapApplicationListener捕获到事件来触发服务的发布流
程。DubboBootstrapApplicationListener中实现了ApplicationListener接口，实现了接口中的onApplicationEvent方法，当spring容器启动完成后会监听该事件。
```java 
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (isOriginalEventSource(event)) {
            if (event instanceof DubboAnnotationInitedEvent) {
                // This event will be notified at AbstractApplicationContext.registerListeners(),
                // init dubbo config beans before spring singleton beans
                initDubboConfigBeans();
            } else if (event instanceof ApplicationContextEvent) {
			    //服务注册的入口
                this.onApplicationContextEvent((ApplicationContextEvent) event);
            }
        }
    }
	
	private void onApplicationContextEvent(ApplicationContextEvent event) {
        if (DubboBootstrapStartStopListenerSpringAdapter.applicationContext == null) {
            DubboBootstrapStartStopListenerSpringAdapter.applicationContext = event.getApplicationContext();
        }

        if (event instanceof ContextRefreshedEvent) {
			//核心方法
            onContextRefreshedEvent((ContextRefreshedEvent) event);
        } else if (event instanceof ContextClosedEvent) {
            onContextClosedEvent((ContextClosedEvent) event);
        }
    }
	
	if (dubboBootstrap.getTakeoverMode() == BootstrapTakeoverMode.SPRING) {
            dubboBootstrap.start();
        }
		
	/**
     * Start the bootstrap
     */
    public synchronized DubboBootstrap start() {
        // avoid re-entry start method multiple times in same thread
        if (isCurrentlyInStart){
            return this;
        }

        isCurrentlyInStart = true;
        try {
            if (started.compareAndSet(false, true)) {
                startup.set(false);
                shutdown.set(false);
                awaited.set(false);

				//配置的解析，不是我们分析的重点
                initialize();

                if (logger.isInfoEnabled()) {
                    logger.info(NAME + " is starting...");
                }
                //核心流程
                doStart();

                if (logger.isInfoEnabled()) {
                    logger.info(NAME + " has started.");
                }
            } else {
                if (logger.isInfoEnabled()) {
                    logger.info(NAME + " is started, export/refer new services.");
                }

                doStart();

                if (logger.isInfoEnabled()) {
                    logger.info(NAME + " finish export/refer new services.");
                }
            }
            return this;
        } finally {
            isCurrentlyInStart = false;
        }
    }
	
	    private void doStart() {
        // 1. export Dubbo Services
		//服务暴露的核心流程
        exportServices();

        // If register consumer instance or has exported services
        if (isRegisterConsumerInstance() || hasExportedServices()) {
            // 2. export MetadataService
            exportMetadataService();
            // 3. Register the local ServiceInstance if required
            registerServiceInstance();
        }

        referServices();

        // wait async export / refer finish if needed
        awaitFinish();

        if (isExportBackground() || isReferBackground()) {
            new Thread(() -> {
                while (!asyncExportFinish || !asyncReferFinish) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        logger.error(NAME + " waiting async export / refer occurred and error.", e);
                    }
                }
                onStarted();
            }).start();
        } else {
            onStarted();
        }
    }
	
	private void exportServices() {
		//前面分析dubbo中和spring整合的时候我们说过configManager中包含了所有@DubboService注解的类，并且包含该注解的类会转化为ServiceBean对象，ServiceBean又是ServiceConfigBase的之类
        for (ServiceConfigBase sc : configManager.getServices()) {
            // TODO, compatible with ServiceConfig.export()
			//强转为ServiceConfig 对象
            ServiceConfig<?> serviceConfig = (ServiceConfig<?>) sc;
            serviceConfig.setBootstrap(this);
            if (!serviceConfig.isRefreshed()) {
                serviceConfig.refresh();
            }
            if (sc.isExported()) {
                continue;
            }
			//判断是否是延迟发布，但是最终都会调用到export方法中进行服务的发布流程
            if (sc.shouldExportAsync()) {
                ExecutorService executor = executorRepository.getServiceExportExecutor();
                CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                    try {
                        if (!sc.isExported()) {
                            sc.export();
                            exportedServices.add(sc);
                        }
                    } catch (Throwable t) {
                        logger.error("export async catch error : " + t.getMessage(), t);
                    }
                }, executor);

                asyncExportingFutures.add(future);
            } else {
			//不是延迟发布，如何服务已经发布就不需要再次发布
                if (!sc.isExported()) {
					//调用ServiceConfig 对象的export方法
                    sc.export();
                    exportedServices.add(sc);
                }
            }
        }
    }
	
	@Override
    public synchronized void export() {
        if (this.shouldExport() && !this.exported) {
            this.init();

            // check bootstrap state
            if (!bootstrap.isInitialized()) {
                throw new IllegalStateException("DubboBootstrap is not initialized");
            }

            if (!this.isRefreshed()) {
                this.refresh();
            }

            if (!shouldExport()) {
                return;
            }

			//如果是延迟发布，把任务丢到延迟线程池中
            if (shouldDelay()) {
                DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
            } else {
			    //服务的发布
                doExport();
            }

            if (this.bootstrap.getTakeoverMode() == BootstrapTakeoverMode.AUTO) {
                this.bootstrap.start();
            }
        }
    }
	
	protected synchronized void doExport() {
        if (unexported) {
            throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
        }
        if (exported) {
            return;
        }

        if (StringUtils.isEmpty(path)) {
            path = interfaceName;
        }
		//服务暴露的核心代码，重点看
        doExportUrls();
        exported();
    }
	
	private void doExportUrls() {
		//服务仓库
        ServiceRepository repository = ApplicationModel.getServiceRepository();
		//把服务注册到本地的服务仓库中  getInterfaceClass()：获取实现类对应的接口全路径 ； repository.registerService(getInterfaceClass())：获取接口的相关信息（类信息和方法信息） 最后注册到本地服务的仓库中
        ServiceDescriptor serviceDescriptor = repository.registerService(getInterfaceClass());
        repository.registerProvider(
                getUniqueServiceName(),
                ref,
                serviceDescriptor,
                this,
                serviceMetadata
        );
 
		//通过对类上参数的解析，获取需要注册的协议，这里会获取到两种协议
		//1、service-discovery-registry 注册到本地内存  service-discovery-registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo_provider&dubbo=2.0.2&pid=17468&registry=zookeeper&release=3.0.2.1&timestamp=1646099262348
		//2、registry 注册到注册中心  registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo_provider&dubbo=2.0.2&pid=17468&registry=zookeeper&release=3.0.2.1&timestamp=1646099262348
        List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);

        for (ProtocolConfig protocolConfig : protocols) {
			//pathKey = cn.enjoy.validation.ValidationService (接口全路径)
            String pathKey = URL.buildKey(getContextPath(protocolConfig)
                    .map(p -> p + "/" + path)
                    .orElse(path), group, version);
            // In case user specified path, register service one more time to map it to path.
			//把接口放入到仓库中
            repository.registerService(pathKey, interfaceClass);
			//核心代码
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
	
	private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
		//把所有属性转化为map ，例如：interface -> cn.enjoy.validation.ValidationService  dubbo -> 2.0.2   release -> 3.0.2.1  revision -> 1.0-SNAPSHOT  timeout -> 5000
        Map<String, String> map = buildAttributes(protocolConfig);

        //init serviceMetadata attachments 把信息放入到 serviceMetadata中
        serviceMetadata.getAttachments().putAll(map);

		// 获取url： dubbo://10.22.80.23:20990/cn.enjoy.validation.ValidationService?anyhost=true&application=dubbo_provider&bind.ip=10.22.80.23&bind.port=20990&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=cn.enjoy.validation.ValidationService&metadata-type=remote&methods=save,update,delete&pid=17468&release=3.0.2.1&revision=1.0-SNAPSHOT&side=provider&threadpool=fixed&threads=100&timeout=5000&timestamp=1646099765623
        URL url = buildUrl(protocolConfig, registryURLs, map);

		//把组装的url和两种协议传递到exportUrl方法中
        exportUrl(url, registryURLs);
    }
	
	private void exportUrl(URL url, List<URL> registryURLs) {
        String scope = url.getParameter(SCOPE_KEY);
        // don't export when none is configured
        if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

            // export to local if the config is not remote (export to remote only when config is remote)
			//这里是本地发布，已经过时了不做分析
            if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
                exportLocal(url);
            }

            // export to remote if the config is not local (export to local only when config is local)
			//远程发布，重点看
            if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
                url = exportRemote(url, registryURLs);
                MetadataUtils.publishServiceDefinition(url);
            }

        }
        this.urls.add(url);
    }
	
	private URL exportRemote(URL url, List<URL> registryURLs) {
        if (CollectionUtils.isNotEmpty(registryURLs)) {
		    //对两种协议进行遍历
            for (URL registryURL : registryURLs) {
			    //先进性参数的拼接，然后调用doExportUrl方法进行服务的注册：如果是该协议头service-discovery-registry 注册到本地内存， 该协议头 registry 注册到注册中心
                if (SERVICE_REGISTRY_PROTOCOL.equals(registryURL.getProtocol())) {
                    url = url.addParameterIfAbsent(SERVICE_NAME_MAPPING_KEY, "true");
                }

                //if protocol is only injvm ,not register
                if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                    continue;
                }

                url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                URL monitorUrl = ConfigValidationUtils.loadMonitor(this, registryURL);
                if (monitorUrl != null) {
                    url = url.putAttribute(MONITOR_KEY, monitorUrl);
                }

                // For providers, this is used to enable custom proxy to generate invoker
                String proxy = url.getParameter(PROXY_KEY);
                if (StringUtils.isNotEmpty(proxy)) {
                    registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                }

                if (logger.isInfoEnabled()) {
                    if (url.getParameter(REGISTER_KEY, true)) {
                        logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url.getServiceKey() + " to registry " + registryURL.getAddress());
                    } else {
                        logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url.getServiceKey());
                    }
                }
				//此时的url:dubbo://10.22.80.23:20990/cn.enjoy.validation.ValidationService?anyhost=true&application=dubbo_provider&bind.ip=10.22.80.23&bind.port=20990&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=cn.enjoy.validation.ValidationService&metadata-type=remote&methods=save,update,delete&pid=18712&release=3.0.2.1&revision=1.0-SNAPSHOT&service-name-mapping=true&side=provider&threadpool=fixed&threads=100&timeout=5000&timestamp=1646104603380
				//registerURL: service-discovery-registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo_provider&dubbo=2.0.2&pid=18712&registry=zookeeper&release=3.0.2.1&timestamp=1646104586951
                doExportUrl(registryURL.putAttribute(EXPORT_KEY, url), true);
            }

        } else {

            if (MetadataService.class.getName().equals(url.getServiceInterface())) {
                MetadataUtils.saveMetadataURL(url);
            }

            if (logger.isInfoEnabled()) {
                logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
            }

            doExportUrl(url, true);
        }


        return url;
    }
	
	
	private void doExportUrl(URL url, boolean withMetaData) {
	    //private static final ProxyFactory PROXY_FACTORY = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension(); 这里是spi的应用
		//获取调用被代理对象的invoke，这个invoke最终会调到服务端的业务逻辑代码
        Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
        if (withMetaData) {
            invoker = new DelegateProviderMetaDataInvoker(invoker, this);
        }
		//    private static final Protocol PROTOCOL = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension(); 这里是spi的应用
        Exporter<?> exporter = PROTOCOL.export(invoker);
        exporters.add(exporter);
    }
	
	
````

- PROXY_FACTORY代码分析
```java 
private static final ProxyFactory PROXY_FACTORY = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

首先我们查看/META-INF/dubbo/internal/org.apache.dubbo.rpc.ProxyFactory 文件：
stub=org.apache.dubbo.rpc.proxy.wrapper.StubProxyFactoryWrapper
jdk=org.apache.dubbo.rpc.proxy.jdk.JdkProxyFactory
javassist=org.apache.dubbo.rpc.proxy.javassist.JavassistProxyFactory

可以看到这三个类中都没有@Adaptive注解，所以dubbo中会给我们生成一个代理对象 
```

- PROTOCOL代码分析
```java 
首先我们查看/META-INF/dubbo/internal/org.apache.dubbo.rpc.Protocol 文件：
filter=org.apache.dubbo.rpc.cluster.filter.ProtocolFilterWrapper
listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper
mock=org.apache.dubbo.rpc.support.MockProtocol
serializationwrapper=org.apache.dubbo.rpc.protocol.ProtocolSerializationWrapper

dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
injvm=org.apache.dubbo.rpc.protocol.injvm.InjvmProtocol
rest=org.apache.dubbo.rpc.protocol.rest.RestProtocol
grpc=org.apache.dubbo.rpc.protocol.grpc.GrpcProtocol
tri=org.apache.dubbo.rpc.protocol.tri.TripleProtocol
registry=org.apache.dubbo.registry.integration.InterfaceCompatibleRegistryProtocol
service-discovery-registry=org.apache.dubbo.registry.integration.RegistryProtocol
qos=org.apache.dubbo.qos.protocol.QosProtocolWrapper

这里面是有包装类的

package org.apache.dubbo.rpc;
import org.apache.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
public void destroy()  {
throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
}
public int getDefaultPort()  {
throw new UnsupportedOperationException("The method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
}
public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
if (arg0.getUrl() == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
org.apache.dubbo.common.URL url = arg0.getUrl();
String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
return extension.export(arg0);
}
public java.util.List getServers()  {
throw new UnsupportedOperationException("The method public default java.util.List org.apache.dubbo.rpc.Protocol.getServers() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
}
public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
if (arg1 == null) throw new IllegalArgumentException("url == null");
org.apache.dubbo.common.URL url = arg1;
String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
return extension.refer(arg0, arg1);
}
}
```	

#####包装类的流转分析 

- QosProtocolWrapper
```java 
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
		//如果是Registry://xx 的协议，则开启qos服务
        if (UrlUtils.isRegistry(invoker.getUrl())) {
            startQosServer(invoker.getUrl());
            return protocol.export(invoker);
        }
		//如果是类似于 dubbo://xx 协议
        return protocol.export(invoker);
    }
```

- ProtocolSerializationWrapper
```java 

```

- ProtocolFilterWrapper
registry协议头没做特殊处理	
```java 

````

- ProtocolListenerWrapper
```java 

```
	
- InterfaceCompatibleRegistryProtoco
通过包装类的流转后，最终代码会走到InterfaceCompatibleRegistryProtocol中，该类是RegistryProtocol的一个子类
```java 
@Override
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
	//获取注册协议 zookeeper://xx 根据registry协议头获取的
	//registry --> zookeeper://
	//service-discovery-registry --> service-discovery-registry
	URL registryUrl = getRegistryUrl(originInvoker);
	// url to export locally
	//获取注册到注册中心的dubbo协议
	// url to export locally
	URL providerUrl = getProviderUrl(originInvoker);

	// Subscribe the override data
	// FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
	//  the same service. Because the subscribed is cached key with the name of the service, it causes the
	//  subscription information to cover.
	
	//以下逻辑是对configurators节点注册事件监听，如果修改了属性则会覆盖客户端的该节点的数据
	final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
	
	//zookeeper事件触发后，最终回调的listener
	final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
	overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

	//添加override协议，添加事件监听
	providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
	//export invoker
	//服务暴露和启动核心代码
	final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

	//获取注册逻辑的实现类
	// url to registry
	final Registry registry = getRegistry(registryUrl);
	final URL registeredProviderUrl = getUrlToRegistry(providerUrl, registryUrl);

	// decide if we need to delay publish
	boolean register = providerUrl.getParameter(REGISTER_KEY, true);
	if (register) {
		//把协议注册到中间件中，比如往zookeeper写节点
		register(registry, registeredProviderUrl);
	}

	// register stated url on provider model
	registerStatedUrl(registryUrl, registeredProviderUrl, register);


	exporter.setRegisterUrl(registeredProviderUrl);
	exporter.setSubscribeUrl(overrideSubscribeUrl);

	// Deprecated! Subscribe to override rules in 2.6.x or before.
	registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

	notifyExport(exporter);
	//Ensure that a new exporter instance is returned every time export
	return new DestroyableExporter<>(exporter);
}
```	
	
	
	
	
2. 服务的发布启动
	服务的启动包含服务的netty服务的开启和调用链的建立过程。