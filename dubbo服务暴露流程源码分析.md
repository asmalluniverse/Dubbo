### dubbo����¶Դ�����

����ı�¶������Ҫ��Ϊ�������֣�
1. �����ע��
1.1 ��������  
����Ҫ����һ�������ʱ�����Ҫ�Ѹ÷������Ϣע�ᵽע�������У�������˵������Ҫ�����������ߵ��õ�Э��ע
�ᵽע�������С���zookeeperΪ������provider������ʱ��ͻ���zookeeper��providers�ڵ�����д��dubboЭ
�飬�������Խӿ�������Ϊ���񷢲��Ļ�׼������Ҫ����һ��cn.enjoy.service.UserService�������
��/dubbo/cn.enjoy.service.UserService/providers����д��dubboЭ�飬Э���������£�
```shell
dubbo%3A%2F%2F192.168.8.32%3A20990%2Fcn.enjoy.service.UserService%3Fanyhost%3Dtrue%26appl
ication%3Ddubbo_provider%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26gener
ic%3Dfalse%26interface%3Dcn.enjoy.service.UserService%26metadatatype%3Dremote%26methods%3DdoKill%2CqueryUser%26pid%3D10760%26release%3D3.0.2.1%26retri
es%3D4%26revision%3D1.0-SNAPSHOT%26service-namemapping%3Dtrue%26side%3Dprovider%26threadpool%3Dfixed%26threads%3D100%26timeout%3D50
00%26timestamp%3D1634537762362
```
1.2Դ�������  
����������Ҫ֪�������ע���Ǻ�ʱ�����ģ���spring����������ɺ󣬻ᷢ���¼�����DubboBootstrapApplicationListener�����¼�����������ķ�����
�̡�DubboBootstrapApplicationListener��ʵ����ApplicationListener�ӿڣ�ʵ���˽ӿ��е�onApplicationEvent��������spring����������ɺ��������¼���
```java 
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (isOriginalEventSource(event)) {
            if (event instanceof DubboAnnotationInitedEvent) {
                // This event will be notified at AbstractApplicationContext.registerListeners(),
                // init dubbo config beans before spring singleton beans
                initDubboConfigBeans();
            } else if (event instanceof ApplicationContextEvent) {
			    //����ע������
                this.onApplicationContextEvent((ApplicationContextEvent) event);
            }
        }
    }
	
	private void onApplicationContextEvent(ApplicationContextEvent event) {
        if (DubboBootstrapStartStopListenerSpringAdapter.applicationContext == null) {
            DubboBootstrapStartStopListenerSpringAdapter.applicationContext = event.getApplicationContext();
        }

        if (event instanceof ContextRefreshedEvent) {
			//���ķ���
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

				//���õĽ������������Ƿ������ص�
                initialize();

                if (logger.isInfoEnabled()) {
                    logger.info(NAME + " is starting...");
                }
                //��������
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
		//����¶�ĺ�������
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
		//ǰ�����dubbo�к�spring���ϵ�ʱ������˵��configManager�а���������@DubboServiceע����࣬���Ұ�����ע������ת��ΪServiceBean����ServiceBean����ServiceConfigBase��֮��
        for (ServiceConfigBase sc : configManager.getServices()) {
            // TODO, compatible with ServiceConfig.export()
			//ǿתΪServiceConfig ����
            ServiceConfig<?> serviceConfig = (ServiceConfig<?>) sc;
            serviceConfig.setBootstrap(this);
            if (!serviceConfig.isRefreshed()) {
                serviceConfig.refresh();
            }
            if (sc.isExported()) {
                continue;
            }
			//�ж��Ƿ����ӳٷ������������ն�����õ�export�����н��з���ķ�������
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
			//�����ӳٷ�������η����Ѿ������Ͳ���Ҫ�ٴη���
                if (!sc.isExported()) {
					//����ServiceConfig �����export����
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

			//������ӳٷ����������񶪵��ӳ��̳߳���
            if (shouldDelay()) {
                DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
            } else {
			    //����ķ���
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
		//����¶�ĺ��Ĵ��룬�ص㿴
        doExportUrls();
        exported();
    }
	
	private void doExportUrls() {
		//����ֿ�
        ServiceRepository repository = ApplicationModel.getServiceRepository();
		//�ѷ���ע�ᵽ���صķ���ֿ���  getInterfaceClass()����ȡʵ�����Ӧ�Ľӿ�ȫ·�� �� repository.registerService(getInterfaceClass())����ȡ�ӿڵ������Ϣ������Ϣ�ͷ�����Ϣ�� ���ע�ᵽ���ط���Ĳֿ���
        ServiceDescriptor serviceDescriptor = repository.registerService(getInterfaceClass());
        repository.registerProvider(
                getUniqueServiceName(),
                ref,
                serviceDescriptor,
                this,
                serviceMetadata
        );
 
		//ͨ�������ϲ����Ľ�������ȡ��Ҫע���Э�飬������ȡ������Э��
		//1��service-discovery-registry ע�ᵽ�����ڴ�  service-discovery-registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo_provider&dubbo=2.0.2&pid=17468&registry=zookeeper&release=3.0.2.1&timestamp=1646099262348
		//2��registry ע�ᵽע������  registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo_provider&dubbo=2.0.2&pid=17468&registry=zookeeper&release=3.0.2.1&timestamp=1646099262348
        List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);

        for (ProtocolConfig protocolConfig : protocols) {
			//pathKey = cn.enjoy.validation.ValidationService (�ӿ�ȫ·��)
            String pathKey = URL.buildKey(getContextPath(protocolConfig)
                    .map(p -> p + "/" + path)
                    .orElse(path), group, version);
            // In case user specified path, register service one more time to map it to path.
			//�ѽӿڷ��뵽�ֿ���
            repository.registerService(pathKey, interfaceClass);
			//���Ĵ���
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
	
	private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
		//����������ת��Ϊmap �����磺interface -> cn.enjoy.validation.ValidationService  dubbo -> 2.0.2   release -> 3.0.2.1  revision -> 1.0-SNAPSHOT  timeout -> 5000
        Map<String, String> map = buildAttributes(protocolConfig);

        //init serviceMetadata attachments ����Ϣ���뵽 serviceMetadata��
        serviceMetadata.getAttachments().putAll(map);

		// ��ȡurl�� dubbo://10.22.80.23:20990/cn.enjoy.validation.ValidationService?anyhost=true&application=dubbo_provider&bind.ip=10.22.80.23&bind.port=20990&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=cn.enjoy.validation.ValidationService&metadata-type=remote&methods=save,update,delete&pid=17468&release=3.0.2.1&revision=1.0-SNAPSHOT&side=provider&threadpool=fixed&threads=100&timeout=5000&timestamp=1646099765623
        URL url = buildUrl(protocolConfig, registryURLs, map);

		//����װ��url������Э�鴫�ݵ�exportUrl������
        exportUrl(url, registryURLs);
    }
	
	private void exportUrl(URL url, List<URL> registryURLs) {
        String scope = url.getParameter(SCOPE_KEY);
        // don't export when none is configured
        if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

            // export to local if the config is not remote (export to remote only when config is remote)
			//�����Ǳ��ط������Ѿ���ʱ�˲�������
            if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
                exportLocal(url);
            }

            // export to remote if the config is not local (export to local only when config is local)
			//Զ�̷������ص㿴
            if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
                url = exportRemote(url, registryURLs);
                MetadataUtils.publishServiceDefinition(url);
            }

        }
        this.urls.add(url);
    }
	
	private URL exportRemote(URL url, List<URL> registryURLs) {
        if (CollectionUtils.isNotEmpty(registryURLs)) {
		    //������Э����б���
            for (URL registryURL : registryURLs) {
			    //�Ƚ��Բ�����ƴ�ӣ�Ȼ�����doExportUrl�������з����ע�᣺����Ǹ�Э��ͷservice-discovery-registry ע�ᵽ�����ڴ棬 ��Э��ͷ registry ע�ᵽע������
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
				//��ʱ��url:dubbo://10.22.80.23:20990/cn.enjoy.validation.ValidationService?anyhost=true&application=dubbo_provider&bind.ip=10.22.80.23&bind.port=20990&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=cn.enjoy.validation.ValidationService&metadata-type=remote&methods=save,update,delete&pid=18712&release=3.0.2.1&revision=1.0-SNAPSHOT&service-name-mapping=true&side=provider&threadpool=fixed&threads=100&timeout=5000&timestamp=1646104603380
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
	    //private static final ProxyFactory PROXY_FACTORY = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension(); ������spi��Ӧ��
		//��ȡ���ñ���������invoke�����invoke���ջ��������˵�ҵ���߼�����
        Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
        if (withMetaData) {
            invoker = new DelegateProviderMetaDataInvoker(invoker, this);
        }
		//    private static final Protocol PROTOCOL = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension(); ������spi��Ӧ��
        Exporter<?> exporter = PROTOCOL.export(invoker);
        exporters.add(exporter);
    }
	
	
````

- PROXY_FACTORY�������
```java 
private static final ProxyFactory PROXY_FACTORY = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

�������ǲ鿴/META-INF/dubbo/internal/org.apache.dubbo.rpc.ProxyFactory �ļ���
stub=org.apache.dubbo.rpc.proxy.wrapper.StubProxyFactoryWrapper
jdk=org.apache.dubbo.rpc.proxy.jdk.JdkProxyFactory
javassist=org.apache.dubbo.rpc.proxy.javassist.JavassistProxyFactory

���Կ������������ж�û��@Adaptiveע�⣬����dubbo�л����������һ��������� 
```

- PROTOCOL�������
```java 
�������ǲ鿴/META-INF/dubbo/internal/org.apache.dubbo.rpc.Protocol �ļ���
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

���������а�װ���

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

#####��װ�����ת���� 

- QosProtocolWrapper
```java 
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
		//�����Registry://xx ��Э�飬����qos����
        if (UrlUtils.isRegistry(invoker.getUrl())) {
            startQosServer(invoker.getUrl());
            return protocol.export(invoker);
        }
		//����������� dubbo://xx Э��
        return protocol.export(invoker);
    }
```

- ProtocolSerializationWrapper
```java 

```

- ProtocolFilterWrapper
registryЭ��ͷû�����⴦��	
```java 

````

- ProtocolListenerWrapper
```java 

```
	
- InterfaceCompatibleRegistryProtoco
ͨ����װ�����ת�����մ�����ߵ�InterfaceCompatibleRegistryProtocol�У�������RegistryProtocol��һ������
```java 
@Override
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
	//��ȡע��Э�� zookeeper://xx ����registryЭ��ͷ��ȡ��
	//registry --> zookeeper://
	//service-discovery-registry --> service-discovery-registry
	URL registryUrl = getRegistryUrl(originInvoker);
	// url to export locally
	//��ȡע�ᵽע�����ĵ�dubboЭ��
	// url to export locally
	URL providerUrl = getProviderUrl(originInvoker);

	// Subscribe the override data
	// FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
	//  the same service. Because the subscribed is cached key with the name of the service, it causes the
	//  subscription information to cover.
	
	//�����߼��Ƕ�configurators�ڵ�ע���¼�����������޸���������Ḳ�ǿͻ��˵ĸýڵ������
	final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
	
	//zookeeper�¼����������ջص���listener
	final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
	overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

	//���overrideЭ�飬����¼�����
	providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
	//export invoker
	//����¶���������Ĵ���
	final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

	//��ȡע���߼���ʵ����
	// url to registry
	final Registry registry = getRegistry(registryUrl);
	final URL registeredProviderUrl = getUrlToRegistry(providerUrl, registryUrl);

	// decide if we need to delay publish
	boolean register = providerUrl.getParameter(REGISTER_KEY, true);
	if (register) {
		//��Э��ע�ᵽ�м���У�������zookeeperд�ڵ�
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
	
	
	
	
2. ����ķ�������
	������������������netty����Ŀ����͵������Ľ������̡�