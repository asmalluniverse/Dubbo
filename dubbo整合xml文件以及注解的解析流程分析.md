

源码分析

### dubbo 和spring 整合源码分析

1.基于xml的方式跟spring的整合，首先我们必须要知道spring的xml解析流程，只有知道这点才能清楚dubbo的自定义标签是如何解析的，
首先我们分析spring中是如何进行xml文件解析的。
```java
public class XmlProvider {

    public static void main(String[] args) throws InterruptedException {
        ClassPathXmlApplicationContext cxa = new ClassPathXmlApplicationContext("applicationProvider.xml");
        System.out.println("dubbo service started.");
        new CountDownLatch(1).await();
    }
}

从该类的构造方法进入，核心源码在refresh()方法里面：
public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
	
/*
	 *	该方法是spring容器初始化的核心方法。是spring容器初始化的核心流程，是一个典型的父类模板设计模式的运用
	 *	根据不同的上下文对象，会调到不同的上下文对象子类方法中
	 *
	 *	核心上下文子类有：
	 *	ClassPathXmlApplicationContext
	 *	FileSystemXmlApplicationContext
	 *	AnnotationConfigApplicationContext
	 *	EmbeddedWebApplicationContext(springboot)
	 *
	 * 方法重要程度：
	 *  0：不重要，可以不看
	 *  1：一般重要，可看可不看
	 *  5：非常重要，一定要看
	 *  必须读的 ：重要程度 5
	 * */
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			//为容器初始化做准备，重要程度：0
			// Prepare this context for refreshing.
			prepareRefresh();

			/*
			   重要程度：5
			  1、创建BeanFactory对象
			* 2、xml解析
			* 	传统标签解析：bean、import等
			* 	自定义标签解析 如：<context:component-scan base-package="com.xiangxue.jack"/>
			* 	自定义标签解析流程：
			* 		a、根据当前解析标签的头信息找到对应的namespaceUri
			* 		b、加载spring所以jar中的spring.handlers文件。并建立映射关系
			* 		c、根据namespaceUri从映射关系中找到对应的实现了NamespaceHandler接口的类
			* 		d、调用类的init方法，init方法是注册了各种自定义标签的解析类
			* 		e、根据namespaceUri找到对应的解析类，然后调用paser方法完成标签解析
			*
			* 3、把解析出来的xml标签封装成BeanDefinition对象
			* */
			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			/*
			 * 给beanFactory设置一些属性值，可以不看
			 * */
			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				/*
				 * BeanDefinitionRegistryPostProcessor
				 * BeanFactoryPostProcessor
				 * 完成对这两个接口的调用
				 * */
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				/*
				 * 把实现了BeanPostProcessor接口的类实例化，并且加入到BeanFactory中
				 * */
				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				/*
				 * 国际化,重要程度2
				 * */
				// Initialize message source for this context.
				initMessageSource();

				//初始化事件管理类
				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				//这个方法着重理解模板设计模式，因为在springboot中，这个方法是用来做内嵌tomcat启动的
				// Initialize other special beans in specific context subclasses.
				onRefresh();

				/*
				 * 往事件管理类中注册事件类
				 * */
				// Check for listener beans and register them.
				registerListeners();

				/*
				 * 这个方法是spring中最重要的方法，没有之一
				 * 所以这个方法一定要理解要具体看
				 * 1、bean实例化过程
				 * 2、ioc
				 * 3、注解支持
				 * 4、BeanPostProcessor的执行
				 * 5、Aop的入口
				 * */
				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}

	
```

- ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
该方法主要进行xml文件解析
```java 
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	//核心方法，必须读，重要程度：5   调用到该类AbstractRefreshableApplicationContext的refreshBeanFactory方法
	refreshBeanFactory();
	return getBeanFactory();
}


@Override
protected final void refreshBeanFactory() throws BeansException {
	//如果BeanFactory不为空，则清除BeanFactory和里面的实例
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
		//BeanFactory 实例工厂
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		//设置是否可以循环依赖 allowCircularReferences
		//是否允许使用相同名称重新注册不同的bean实现.
		customizeBeanFactory(beanFactory);
		//解析xml，并把xml中的标签封装成BeanDefinition对象
		loadBeanDefinitions(beanFactory);
		this.beanFactory = beanFactory;
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}

@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// Create a new XmlBeanDefinitionReader for the given BeanFactory.
	//创建xml的解析器，这里是一个委托模式
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// Configure the bean definition reader with this context's
	// resource loading environment.
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	//这里传一个this进去，因为ApplicationContext是实现了ResourceLoader接口的
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	// Allow a subclass to provide custom initialization of the reader,
	// then proceed with actually loading the bean definitions.
	initBeanDefinitionReader(beanDefinitionReader);
	//主要看这个方法  重要程度 5
	loadBeanDefinitions(beanDefinitionReader);
	}
	
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
		throws BeanDefinitionStoreException {

	try {
	//把inputSource对象封装成Document对象
		Document doc = doLoadDocument(inputSource, resource);
		解析Document中的元素封装成BeaDefinition对象
		int count = registerBeanDefinitions(doc, resource);
		if (logger.isDebugEnabled()) {
			logger.debug("Loaded " + count + " bean definitions from " + resource);
		}
		return count;
	}
	catch (BeanDefinitionStoreException ex) {
		throw ex;
	}
	catch (SAXParseException ex) {
		throw new XmlBeanDefinitionStoreException(resource.getDescription(),
				"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
	}
	catch (SAXException ex) {
		throw new XmlBeanDefinitionStoreException(resource.getDescription(),
				"XML document from " + resource + " is invalid", ex);
	}
	catch (ParserConfigurationException ex) {
		throw new BeanDefinitionStoreException(resource.getDescription(),
				"Parser configuration exception parsing XML from " + resource, ex);
	}
	catch (IOException ex) {
		throw new BeanDefinitionStoreException(resource.getDescription(),
				"IOException parsing XML document from " + resource, ex);
	}
	catch (Throwable ex) {
		throw new BeanDefinitionStoreException(resource.getDescription(),
				"Unexpected exception parsing XML document from " + resource, ex);
	}
}

public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	int countBefore = getRegistry().getBeanDefinitionCount();
	//主要看这个方法
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	return getRegistry().getBeanDefinitionCount() - countBefore;
}

protected void doRegisterBeanDefinitions(Element root) {
	// Any nested <beans> elements will cause recursion in this method. In
	// order to propagate and preserve <beans> default-* attributes correctly,
	// keep track of the current (parent) delegate, which may be null. Create
	// the new (child) delegate with a reference to the parent for fallback purposes,
	// then ultimately reset this.delegate back to its original (parent) reference.
	// this behavior emulates a stack of delegates without actually necessitating one.
	BeanDefinitionParserDelegate parent = this.delegate;
	this.delegate = createDelegate(getReaderContext(), root, parent);

	if (this.delegate.isDefaultNamespace(root)) {
		String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
		if (StringUtils.hasText(profileSpec)) {
			String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
					profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			// We cannot use Profiles.of(...) since profile expressions are not supported
			// in XML config. See SPR-12458 for details.
			if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
							"] not matching: " + getReaderContext().getResource());
				}
				return;
			}
		}
	}

	preProcessXml(root);
	//标签解析 
	parseBeanDefinitions(root, this.delegate);
	postProcessXml(root);

	this.delegate = parent;
}

/**
 * Parse the elements at the root level in the document:
 * "import", "alias", "bean".
 * @param root the DOM root element of the document
 */
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	if (delegate.isDefaultNamespace(root)) {
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element) {
				Element ele = (Element) node;
				if (delegate.isDefaultNamespace(ele)) {
				    //默认标签解析
					parseDefaultElement(ele, delegate);
				}
				else {
				    //自定义标签解析
					delegate.parseCustomElement(ele);
				}
			}
		}
	}
	else {
		delegate.parseCustomElement(root);
	}
}

@Nullable
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
	//获取到命名空间
	String namespaceUri = getNamespaceURI(ele);
	if (namespaceUri == null) {
		return null;
	}
	//根据命名空间找到对应的命名空间处理器
	NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
	if (handler == null) {
		error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
		return null;
	}
	//根据用户自定义的处理器进行解析
	return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}

public class DefaultNamespaceHandlerResolver implements NamespaceHandlerResolver {
    /**
     * Locate the {@link NamespaceHandler} for the supplied namespace URI
     * from the configured mappings.
     * @param namespaceUri the relevant namespace URI
     * @return the located {@link NamespaceHandler}, or {@code null} if none found
     */
    @Override
    public NamespaceHandler resolve(String namespaceUri) {
        // 获取所有已经配置的handler映射
        Map<String, Object> handlerMappings = getHandlerMappings();
        // 根据命名空间获取对应的信息
        Object handlerOrClassName = handlerMappings.get(namespaceUri);
        if (handlerOrClassName == null) {
            return null;
        }
        else if (handlerOrClassName instanceof NamespaceHandler) {
            // 已经做过解析的直接从缓存中获取
            return (NamespaceHandler) handlerOrClassName;
        }
        else {
            // 没有做过解析的，则返回的是类路径
            String className = (String) handlerOrClassName;
            try {
                // 使用反射将类路径转化为类
                Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
                if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
                    throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
                            "] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
                }
                // 初始化处理器
                NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
                // 调用自定义的NameSpaceHandler的init方法
                namespaceHandler.init();
                // 存入缓存
                handlerMappings.put(namespaceUri, namespaceHandler);
                return namespaceHandler;
            }
            catch (ClassNotFoundException ex) {
                throw new FatalBeanException("NamespaceHandler class [" + className + "] for namespace [" +
                        namespaceUri + "] not found", ex);
            }
            catch (LinkageError err) {
                throw new FatalBeanException("Invalid NamespaceHandler class [" + className + "] for namespace [" +
                        namespaceUri + "]: problem with handler class file or dependent class", err);
            }
        }
    }

        /**
     * Load the specified NamespaceHandler mappings lazily.
     */
    private Map<String, Object> getHandlerMappings() {
        // 如果没有缓存则开始进行缓存
        if (this.handlerMappings == null) {
            synchronized (this) {
                if (this.handlerMappings == null) {
                    try {
                        // this.handlerMappingsLocation在构造函数中初始化为META-INF/spring.handlers
                        Properties mappings =
                                PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
                        if (logger.isDebugEnabled()) {
                            logger.debug("Loaded NamespaceHandler mappings: " + mappings);
                        }
                        Map<String, Object> handlerMappings = new ConcurrentHashMap<String, Object>(mappings.size());
                        CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
                        this.handlerMappings = handlerMappings;
                    }
                    catch (IOException ex) {
                        throw new IllegalStateException(
                                "Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
                    }
                }
            }
        }
        return this.handlerMappings;
    }
}
```
现在我们知道了为什么需要在META-INF/spring.handlers文件中那样配置了。因为在加载所有的处理器的配置的默认目录就是META-INF/spring.handlers;
当获取到自定义的NameSpaceHandler之后就可以进行处理器初始化并解析了。实际上就是调用init方法。下面我们在分析dubbo中如何进行xml文件解析的，就很简单了。




1.dubbo中如何解析xml文件：
(1)找到spring.handlers文件  （org\apache\dubbo\dubbo\3.0.2.1\dubbo-3.0.2.1.jar!\META-INF\spring.handlers）
(2)spring.handlers该文件中对应着着uri和该类之间的映射关系，就是通过该类的init方法进行xml解析的 
那么该类的init方法是什么时候执行的呢？ 其实是通过spring 加载init方法是会进行调用。
dubbo进行xml文件解析的时候使用的就是spring的扩展机制：

对比spring 中的xml 文件解析 dubbo 中是DubboNamespaceHandler类  spring 中的入口是 AbstractApplicationContext 中的refresh方法 
中的ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory(); 进行的，
也就是在启动spring 的时候调用该方法可以进行dubbo 配置文件的解析

dubbo 中每个标签都对应这一个映射类
- init方法
```java 
    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class));
        registerBeanDefinitionParser("config-center", new DubboBeanDefinitionParser(ConfigCenterBean.class));
        registerBeanDefinitionParser("metadata-report", new DubboBeanDefinitionParser(MetadataReportConfig.class));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class));
        registerBeanDefinitionParser("metrics", new DubboBeanDefinitionParser(MetricsConfig.class));
        registerBeanDefinitionParser("ssl", new DubboBeanDefinitionParser(SslConfig.class));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
```
讲白了就是将标签对应的解析类关联起来，这样在解析到标签的时候就知道委托给对应的解析类解析，本质就是为了生成 Spring 的 BeanDefinition，然后利用 Spring 最终创建对应的对象。

- parse方法
对于xml文件的解析：parse方法进行每个标签的解析工作
```java 
public BeanDefinition parse(Element element, ParserContext parserContext) {
	BeanDefinitionRegistry registry = parserContext.getRegistry();
	registerAnnotationConfigProcessors(registry);
	/**
	 * @since 2.7.8
	 * issue : https://github.com/apache/dubbo/issues/6275
	 */
	
	//核心方法，注册bean ：让dubbo中的核心类转化为BeaDifition对象，加载到spring容器中进行实例化
	registerCommonBeans(registry);
	
	调用父类的parse方法
	BeanDefinition beanDefinition = super.parse(element, parserContext);
	setSource(beanDefinition);
	return beanDefinition;
}

static void registerCommonBeans(BeanDefinitionRegistry registry) {

	registerInfrastructureBean(registry, ServicePackagesHolder.BEAN_NAME, ServicePackagesHolder.class);

	registerInfrastructureBean(registry, ReferenceBeanManager.BEAN_NAME, ReferenceBeanManager.class);

	// Since 2.5.7 Register @Reference Annotation Bean Processor as an infrastructure Bean
	registerInfrastructureBean(registry, ReferenceAnnotationBeanPostProcessor.BEAN_NAME,
			ReferenceAnnotationBeanPostProcessor.class);

	//DubboConfigAliasPostProcessor：标签解析类，进行标签的解析
	// TODO Whether DubboConfigAliasPostProcessor can be removed ?
	// Since 2.7.4 [Feature] https://github.com/apache/dubbo/issues/5093
	registerInfrastructureBean(registry, DubboConfigAliasPostProcessor.BEAN_NAME,
			DubboConfigAliasPostProcessor.class);

	//dubbo 监听器，在spring启动完成后会调用该类的方法进行服务的发布
	// Since 2.7.4 Register DubboBootstrapApplicationListener as an infrastructure Bean
	registerInfrastructureBean(registry, DubboBootstrapApplicationListener.BEAN_NAME,
			DubboBootstrapApplicationListener.class);

	// Since 2.7.6 Register DubboConfigDefaultPropertyValueBeanPostProcessor as an infrastructure Bean
	registerInfrastructureBean(registry, DubboConfigDefaultPropertyValueBeanPostProcessor.BEAN_NAME,
			DubboConfigDefaultPropertyValueBeanPostProcessor.class);

	// Dubbo config initializer
	registerInfrastructureBean(registry, DubboConfigBeanInitializer.BEAN_NAME, DubboConfigBeanInitializer.class);

	// register infra bean if not exists later
	registerInfrastructureBean(registry, DubboInfraBeanRegisterPostProcessor.BEAN_NAME, DubboInfraBeanRegisterPostProcessor.class);
}
				
```

- 调用父类NamespaceHandlerSupport的parse方法
```java 

public BeanDefinition parse(Element element, ParserContext parserContext) {
    //找到标签对应的解析类
	BeanDefinitionParser parser = findParserForElement(element, parserContext);
	//调用parser方法进行标签的解析
	return (parser != null ? parser.parse(element, parserContext) : null);
}
```

- findParserForElement
```java 
	@Nullable
private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
	找到localName也就是上面init方法中put到parser对象中的key 其实就是标签中的 <dubbo:service interface="cn.enjoy.service.UserService" ref="userServiceImpl" timeout="2000"> dubbo:service 中的sercice，
	通过映射关系获取对应的解析类，想当于返回了 DubboBeanDefinitionParser 对象，并调用了该方法的parse方法
	String localName = parserContext.getDelegate().getLocalName(element);
	BeanDefinitionParser parser = this.parsers.get(localName);
	if (parser == null) {
		parserContext.getReaderContext().fatal(
				"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
	}
	return parser;
}

- DubboBeanDefinitionParser中的parse方法
DubboBeanDefinitionParser中的parse方法，可以看到该方法返回的是一个RootBeanDefinition对象，也就是说明该方法会把解析xml对象的类封装成为beanDefinition对象，用于spring进行实例化操作
即，解析标签的目的是为了把对应标签的配置类进行实例化
@Override
public BeanDefinition parse(Element element, ParserContext parserContext) {
    //该方法就是对标签中的属性进行解析，转化为RootBeanDefinition对象，为了加入到spring容器中进行实例化
	return parse(element, parserContext, beanClass, true);
}
```
	
				
2. 服务发布  
服务发布和spring 有什么关系（service这里以service标签解析进行分析）
registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class));

其中ServiceBean中实现了spring 中的一系列接口，其中InitializingBean接口我们都知道：
InitializingBean接口为bean提供了初始化方法的方式，它只包括afterPropertiesSet方法，凡是继承该接口的类，在初始化bean的时候都会执行该方法。

- afterPropertiesSet
其实在该方法中的核心逻辑就是往ConfigManager中添加ServiceBean的实例，这样当spring容器完成启动后，
dubbo的事件监听类DubboBootstrapApplicationListener就会根据ConfigManager中的ServiceBean实例来完成
服务的发布
```java
public void afterPropertiesSet() throws Exception {
	if (StringUtils.isEmpty(getPath())) {
		if (StringUtils.isNotEmpty(getInterface())) {
			setPath(getInterface());
		}
	}
	//register service bean and set bootstrap
	//穿件DubboBootstrap对象，并把该对象设置到ServiceBean中，并且把各种标签解析类加入到ConfigManager对象中
	DubboBootstrap.getInstance().service(this);
}

    public DubboBootstrap service(ServiceConfig<?> serviceConfig) {
        serviceConfig.setBootstrap(this);
        configManager.addService(serviceConfig);
        return this;
    }
```
	
以上都是为服务发布做准备，还没有进行服务的发布，服务的发布是通过一个时间监听机制实现的，在spring 容器加载完成后才进行的服务发布，
在 AbstractApplicationContext 中的refresh方法 会执行finishRefresh();方法
- finishRefresh
```java 
// Last step: publish corresponding event.
				finishRefresh();

protected void finishRefresh() {
	// Clear context-level resource caches (such as ASM metadata from scanning).
	clearResourceCaches();

	// Initialize lifecycle processor for this context.
	initLifecycleProcessor();

	// Propagate refresh to lifecycle processor first.
	getLifecycleProcessor().onRefresh();

	// Publish the final event. 
	//spring容器加载好后发布监听事件
	publishEvent(new ContextRefreshedEvent(this));

	// Participate in LiveBeansView MBean, if active.
	LiveBeansView.registerApplicationContext(this);


}

方法的链路调用：
publishEvent(new ContextRefreshedEvent(this)); -> multicastEvent(applicationEvent, eventType);  
-> invokeListener(listener, event)  - > doInvokeListener(listener, event); -> onApplicationEvent(event); 
此时就会调用到bubbo中的监听器DubboBootstrapApplicationListener中的onApplicationEvent方法
```java 
@Override
public void onApplicationEvent(ApplicationEvent event) {
	if (isOriginalEventSource(event)) {
		if (event instanceof DubboAnnotationInitedEvent) {
			// This event will be notified at AbstractApplicationContext.registerListeners(),
			// init dubbo config beans before spring singleton beans
			initDubboConfigBeans();
		} else if (event instanceof ApplicationContextEvent) {
			this.onApplicationContextEvent((ApplicationContextEvent) event);
		}
	}
}
	
private void onApplicationContextEvent(ApplicationContextEvent event) {
	if (DubboBootstrapStartStopListenerSpringAdapter.applicationContext == null) {
		DubboBootstrapStartStopListenerSpringAdapter.applicationContext = event.getApplicationContext();
	}

	if (event instanceof ContextRefreshedEvent) {
		onContextRefreshedEvent((ContextRefreshedEvent) event);
	} else if (event instanceof ContextClosedEvent) {
		onContextClosedEvent((ContextClosedEvent) event);
	}
}


```

```java 
private void onContextRefreshedEvent(ContextRefreshedEvent event) {
	if (dubboBootstrap.getTakeoverMode() == BootstrapTakeoverMode.SPRING) {
		//服务的发布
		dubboBootstrap.start();
	}
} 
```

- start()
该方法进行的服务发布
```java 
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
			//完成注册中心和元数据中心的初始化
			initialize();

			if (logger.isInfoEnabled()) {
				logger.info(NAME + " is starting...");
			}
			//服务的发布，根据ConfigManager中加载的配置类进行服务的发布
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
```

- doStart()
```java 
private void doStart() {
	// 1. export Dubbo Services  标签的发布
	exportServices();

	// If register consumer instance or has exported services
	if (isRegisterConsumerInstance() || hasExportedServices()) {
		// 2. export MetadataService
		exportMetadataService();
		// 3. Register the local ServiceInstance if required
		registerServiceInstance();
	}

	//标签的引用
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
```

- exportServices
这里就和上面的configManager对上了，因为configManager中保存了所有的serviceBean实例（xml中的配置信息），可以拿到xml中配置的服务实例，就可以进行发布操作
```java 
private void exportServices() {
	for (ServiceConfigBase sc : configManager.getServices()) {
		// TODO, compatible with ServiceConfig.export()
		ServiceConfig<?> serviceConfig = (ServiceConfig<?>) sc;
		serviceConfig.setBootstrap(this);
		if (!serviceConfig.isRefreshed()) {
		//属性的注入
			serviceConfig.refresh();
		}
		if (sc.isExported()) {
			continue;
		}
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
			if (!sc.isExported()) {
				//最终这里会调用ServerBean的export方法进行服务的发布
				sc.export();
				exportedServices.add(sc);
			}
		}
	}
}

```	

- export
```java 
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

		if (shouldDelay()) {
			DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
		} else {
			doExport();
		}

		if (this.bootstrap.getTakeoverMode() == BootstrapTakeoverMode.AUTO) {
			this.bootstrap.start();
		}
	}
}
```
至此，dubbo的xml方式跟spring整合就分析完成	


### dubbo注解的方式源码分析
注解的方式只需要在需要暴露的服务上面加上一个@DubboService注解就可以完成服务的暴露了，配置量非常少，
但是这种方式也有一定的侵入性。然后通过@EnableDubbo注解，扫描类上面的@DubboService注解以及@DubboReference注解

@EnableDubbo 为什么会被spring识别

1. @EnableDubbo中有一个@DubboComponentScan注解 在改注解中有一个import注解(@Import(DubboComponentScanRegistrar.class))
而import注解会被spring扫描到,会在spring的ConfigurationClassPostProcessor类中进行注解的解析，会解析@Import注解
然后会把import中的 DubboComponentScanRegistrar.class类转化为beanDefinition

- ConfigurationClassPostProcessor
```java
public class AnnotationProvider
{
    public static void main( String[] args ) throws InterruptedException {
	    //以注解的方式启动
        new AnnotationConfigApplicationContext(ProviderConfiguration.class);
        System.out.println("dubbo service started.");
        new CountDownLatch(1).await();
    }
}

public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
	this();
	register(componentClasses);
	refresh();
}

public AnnotationConfigApplicationContext() {
	//通过spring上下文对象实例化把ConfigurationClassPostProcessor变成BeanDefinition对象
	this.reader = new AnnotatedBeanDefinitionReader(this);
	this.scanner = new ClassPathBeanDefinitionScanner(this);
}

public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
	this(registry, getOrCreateEnvironment(registry));
}

public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	Assert.notNull(environment, "Environment must not be null");
	this.registry = registry;
	this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
	//进入这个方法
	AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}

public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
	registerAnnotationConfigProcessors(registry, null);
}

public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
		BeanDefinitionRegistry registry, @Nullable Object source) {

	DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
	if (beanFactory != null) {
		if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
			beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
		}
		if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
			beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
		}
	}

	Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

	//ConfigurationClassPostProcessor 转化为RootBeanDefinition对象，为了加入到spring容器中进行实例化
	if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
	}
```

2. ConfigurationClassPostProcessor对@Import注解的扫描
由于ConfigurationClassPostProcessor类是一个BeanDefinitionRegistryPostProcessor类型的，所以在spring容器
中它会优先被实例化，实例化的地方在refresh核心方法的：

```java
/*
 * BeanDefinitionRegistryPostProcessor
 * BeanFactoryPostProcessor
 * 完成对这两个接口的调用
 * */
// Invoke factory processors registered as beans in the context.
invokeBeanFactoryPostProcessors(beanFactory);
```

所以当ConfigurationClassPostProcessor实例化的时候就调用到postProcessBeanDefinitionRegistry方法，方法逻
辑如下：

spring中会执行如下代码对import注解进行解析
```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
	List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
	//获取所有beanNames
	String[] candidateNames = registry.getBeanDefinitionNames();

	for (String beanName : candidateNames) {
		BeanDefinition beanDef = registry.getBeanDefinition(beanName);
		if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
			if (logger.isDebugEnabled()) {
				logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
			}
		}
		else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
			configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
		}
	}

	// Return immediately if no @Configuration classes were found
	if (configCandidates.isEmpty()) {
		return;
	}

	// Sort by previously determined @Order value, if applicable
	configCandidates.sort((bd1, bd2) -> {
		int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
		int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
		return Integer.compare(i1, i2);
	});

	// Detect any custom bean name generation strategy supplied through the enclosing application context
	SingletonBeanRegistry sbr = null;
	if (registry instanceof SingletonBeanRegistry) {
		sbr = (SingletonBeanRegistry) registry;
		if (!this.localBeanNameGeneratorSet) {
			BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
					AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
			if (generator != null) {
				this.componentScanBeanNameGenerator = generator;
				this.importBeanNameGenerator = generator;
			}
		}
	}

	if (this.environment == null) {
		this.environment = new StandardEnvironment();
	}

	// Parse each @Configuration class
	ConfigurationClassParser parser = new ConfigurationClassParser(
			this.metadataReaderFactory, this.problemReporter, this.environment,
			this.resourceLoader, this.componentScanBeanNameGenerator, registry);

	Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
	Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
	do {
		parser.parse(candidates);
		parser.validate();

		Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
		configClasses.removeAll(alreadyParsed);

		// Read the model and create bean definitions based on its content
		if (this.reader == null) {
			this.reader = new ConfigurationClassBeanDefinitionReader(
					registry, this.sourceExtractor, this.resourceLoader, this.environment,
					this.importBeanNameGenerator, parser.getImportRegistry());
		}
		//@Bean @Import 内部类 @ImportedResource ImportBeanDefinitionRegistrar具体处理逻辑
		this.reader.loadBeanDefinitions(configClasses);
		alreadyParsed.addAll(configClasses);

		candidates.clear();
		if (registry.getBeanDefinitionCount() > candidateNames.length) {
			String[] newCandidateNames = registry.getBeanDefinitionNames();
			Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
			Set<String> alreadyParsedClasses = new HashSet<>();
			for (ConfigurationClass configurationClass : alreadyParsed) {
				alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
			}
			for (String candidateName : newCandidateNames) {
				if (!oldCandidateNames.contains(candidateName)) {
					BeanDefinition bd = registry.getBeanDefinition(candidateName);
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
							!alreadyParsedClasses.contains(bd.getBeanClassName())) {
						candidates.add(new BeanDefinitionHolder(bd, candidateName));
					}
				}
			}
			candidateNames = newCandidateNames;
		}
	}
	while (!candidates.isEmpty());

	// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
	if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
		sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
	}

	if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
		// Clear cache in externally provided MetadataReaderFactory; this is a no-op
		// for a shared cache since it'll be cleared by the ApplicationContext.
		((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
	}
}
```
就是在上面方面里面的this.reader.loadBeanDefinitions(configClasses);这行代码对所有BeanDefinition中对应类上
如果有@Import注解进行了解析处理，这样spring就能够扫描的@EnableDubbo注解了	
	
- DubboComponentScanRegistrar
DubboComponentScanRegistrar类是通过@EnableDubbo上面的@DubboComponentScan注解import进来的，
它是一个ImportBeanDefinitionRegistrar类型的，所有当被引入进来以后就会被spring调用到它的
registerBeanDefinitions方法，代码如下：
	
```java
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

 
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

		// @since 2.7.6 Register the common beans 实现一些核心类的实例化:ReferenceAnnotationBeanPostProcessor  DubboBootstrapApplicationListener
		registerCommonBeans(registry);

		// AnnotationMetadata 该对象会拿到@EnableDubbo(scanBasePackages = "cn.enjoy") 注解里面的value值即包的路径，
		Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

		//ServiceAnnotationPostProcessor
		registerServiceAnnotationPostProcessor(packagesToScan, registry);
	}

	private void registerServiceAnnotationPostProcessor(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
		//把该类转化为beanfinition 对象 ServiceAnnotationPostProcessor
		BeanDefinitionBuilder builder = rootBeanDefinition(ServiceAnnotationPostProcessor.class);
		builder.addConstructorArgValue(packagesToScan);
		builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();

		//注册到该注册器中registry ，进行实例化操作
		BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry);
    }

}
```

该方法其实核心就是注册了三个比较重要的类给了spring容器：
1. ReferenceAnnotationBeanPostProcessor
2. DubboBootstrapApplicationListener
3. ServiceAnnotationPostProcessor

DubboBootstrapApplicationListener这个类前面我们分析过，它就是在spring容器启动完成后通过spring发布一个
spring容器启动完成的事件，然后该类捕获到事件，通过捕获事件来完成服务的发布和引用的，这里就不再赘述了。
现在主要分析ServiceAnnotationPostProcessor和ReferenceAnnotationBeanPostProcessor
	
- ServiceAnnotationPostProcessor类分析

**这个类的作用就是用来扫描类上面的@DubboService注解的，就只有这个功能。**
	
该类 public class ServiceAnnotationPostProcessor implements BeanDefinitionRegistryPostProcessor 实现了一个spring提供的接口，
该接口提供了void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;方法，会在类实例化的时候执行,
那么我们就看下 ServiceAnnotationPostProcessor 类中的postProcessBeanDefinitionRegistry 方法
	
```java 
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
	this.registry = registry;

	Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

	if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
		//进行扫描操作
		scanServiceBeans(resolvedPackagesToScan, registry);
	} else {
		if (logger.isWarnEnabled()) {
			logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
		}
	}

}
```

- scanServiceBeans方法分析
```java 
/**
 * Scan and registers service beans whose classes was annotated {@link Service}
 *
 * @param packagesToScan The base packages to scan
 * @param registry       {@link BeanDefinitionRegistry}
 */
private void scanServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

	//定义一个扫描器，这个dubbo扫描器其实就是继承了spring的ClassPathBeanDefinitionScanner扫描器
	DubboClassPathBeanDefinitionScanner scanner =
			new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

	BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);
	scanner.setBeanNameGenerator(beanNameGenerator);
	//对扫描器添加需要扫描的注解类型，这里的注解类型有三种，apache的@Service，@DubboService，阿里巴巴的@Service
	for (Class<? extends Annotation> annotationType : serviceAnnotationTypes) {
		scanner.addIncludeFilter(new AnnotationTypeFilter(annotationType));
	}

	ScanExcludeFilter scanExcludeFilter = new ScanExcludeFilter();
	scanner.addExcludeFilter(scanExcludeFilter);

	for (String packageToScan : packagesToScan) {

		// avoid duplicated scans
		if (servicePackagesHolder.isPackageScanned(packageToScan)) {
			if (logger.isInfoEnabled()) {
				logger.info("Ignore package who has already bean scanned: " + packageToScan);
			}
			continue;
		}

		//这里就是核心的扫描代码
		//扫描的核心流程，其实就是递归找文件的过程，如果找的是文件夹则递归找，如果是文件则根据完整限定名反射
		//出反射对象，根据反射对象判断类上是否有DubboService注解，如果有则把该类变成BeanDefinition
		// Registers @Service Bean first
		scanner.scan(packageToScan);

		// Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not.
		//根据前面的扫描找到的BeanDefinition集合，获取这个集合
		Set<BeanDefinitionHolder> beanDefinitionHolders =
				findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

		if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {
			if (logger.isInfoEnabled()) {
				List<String> serviceClasses = new ArrayList<>(beanDefinitionHolders.size());
				for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
					serviceClasses.add(beanDefinitionHolder.getBeanDefinition().getBeanClassName());
				}
				logger.info("Found " + beanDefinitionHolders.size() + " classes annotated by Dubbo @Service under package [" + packageToScan + "]: " + serviceClasses);
			}
			//这行代码很重要，因为要完成服务暴露那么就必须调用到ServiceBean中的export方法
			//这行代码就是根据有注解的类的BeanDefinition来创建根该类对应的ServiceBean的
			//BeanDefinition对象
			for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
				processScannedBeanDefinition(beanDefinitionHolder, registry, scanner);
				servicePackagesHolder.addScannedClass(beanDefinitionHolder.getBeanDefinition().getBeanClassName());
			}
		} else {
			if (logger.isWarnEnabled()) {
				logger.warn("No class annotated by Dubbo @Service was found under package ["
						+ packageToScan + "], ignore re-scanned classes: " + scanExcludeFilter.getExcludedCount());
			}
		}

		servicePackagesHolder.addScannedPackage(packageToScan);
	}
}
```

从上面的分析，我们知道，扫描到有@DubboService注解的类以后，实际上会创建跟该类对应的ServiceBean
的实例，也就是有一个@DubboService注解的类就会对应一个ServiceBean的实例在spring容器中。那么又回到了
上面xml分析的流程了，ServiceBean实例化走到afterPropertiesSet方法，然后spring容器启动完成走到dubbo
的监听器完成服务发布了	   
	   
	  
- ReferenceAnnotationBeanPostProcessor 类分析
ReferenceAnnotationBeanPostProcessor的作用就是完成@DubboReference属性或者方法的依赖注入，其依赖注
入的流程也是完全依赖spring的，所以我们必须要掌握spring的依赖注入流程，其实spring的依赖注入核心就两点；
1. 注解的收集
2. 实例的注入

1. spring中的依赖注入
以@Autowired注解的属性来分析spring的依赖注入，例如：
```java 
@Component
public class SpringTest {
	@Autowired
	private UserService userService;

	private UserService userService1;

	private UserService userService2;

	public UserService getUserService() {
	return userService;
	} 
	
	@PostConstruct
	public void init() {
	System.out.println(userService);
	}
}
```

1. @Autowired注解收集  
注解收集的核心流程
```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					//对类中注解的装配过程
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
	
	
```

其实dubbo的依赖注入流程跟spring是一模一样的，也是借助BeanPostProcessor类型的接口来实现，只是这个类是

```java 
public class AnnotationProvider
{
    public static void main( String[] args ) throws InterruptedException {
		//通过注解的方式进入
        new AnnotationConfigApplicationContext(ProviderConfiguration.class);
        System.out.println("dubbo service started.");
        new CountDownLatch(1).await();
    }
}

	public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
		this();
		register(componentClasses);
		refresh();
	}
```

对于refreash()方法中：
```java 	
	
/*
 * 这个方法是spring中最重要的方法，没有之一
 * 所以这个方法一定要理解要具体看
 * 1、bean实例化过程
 * 2、ioc
 * 3、注解支持
 * 4、BeanPostProcessor的执行
 * 5、Aop的入口
 * */
// Instantiate all remaining (non-lazy-init) singletons.
finishBeanFactoryInitialization(beanFactory);


protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		//设置类型转换器
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		//暂时不要看
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		//暂时不看
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		//重点看这个方法，重要程度：5 单例实例化
		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}

	
	
@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged(
									(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
	
	
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
	
	@SuppressWarnings("unchecked")
	protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {

		String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.  创建单例实例化 
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					if (!StringUtils.hasLength(scopeName)) {
						throw new IllegalStateException("No scope name defined for bean ′" + beanName + "'");
					}
					Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
	
	
@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
	
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			//这里会通过三种方式进行实例化 ，但是此时并没有进行ioc的操作
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		//AutowiredAnnotationBeanPostProcessor该对象是完成依赖注入的核心类
		//下面是进行ioc依赖注入的核心流程
		//ioc 依赖注入有两个过程， 1.注解收集 2. 依赖注入
		Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					//注解收集  
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			//添加三级缓存。解决循环依赖的问题
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			//ioc  di 的核心逻辑
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}
		
		
		
		
		protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						return;
					}
				}
			}
		}

		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		int resolvedAutowireMode = mbd.getResolvedAutowireMode();
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			// Add property values based on autowire by name if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}
			// Add property values based on autowire by type if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

		PropertyDescriptor[] filteredPds = null;
		if (hasInstAwareBpps) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					@Autowired注解的收集  依赖注入的过程
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						//老版本使用的这个方法完成的依赖注入 @Autowired的支持，以及dubbo中@DubboReference注解  这里会调用到 ReferenceAnnotationBeanPostProcessor类中的postProcessPropertyValues方法 
						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvsToUse == null) {
							return;
						}
					}
					pvs = pvsToUse;
				}
			}
		}
		if (needsDepCheck) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}

		if (pvs != null) {
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}

		

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```
	
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
		//遍历所有的beanPostProcessor实例
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof MergedBeanDefinitionPostProcessor) {
				MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
				//进入到AutowiredAnnotationBeanPostProcessor 进行 @Autowired注解的解析 
				//dubbo中通过ReferenceAnnotationBeanPostProcessor 进行@DubboReference注解的收集 
				bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
			}
		}
	}
	
		@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
		metadata.checkConfigMembers(beanDefinition);
	}
	
	private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
		// Fall back to class name as cache key, for backwards compatibility with custom callers.
		//cacheKey 对应的就是类名称，先从缓存中拿，缓存中拿不到就去build创建，查看类中的属性或者方法是否有@Autowired注解
		String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
		// Quick check on the concurrent map first, with minimal locking.
		InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
		if (InjectionMetadata.needsRefresh(metadata, clazz)) {
			synchronized (this.injectionMetadataCache) {
				metadata = this.injectionMetadataCache.get(cacheKey);
				if (InjectionMetadata.needsRefresh(metadata, clazz)) {
					if (metadata != null) {
						metadata.clear(pvs);
					}
					metadata = buildAutowiringMetadata(clazz);
					//放入到缓存中 
					this.injectionMetadataCache.put(cacheKey, metadata);
				}
			}
		}
		return metadata;
	}
	
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
		if (!AnnotationUtils.isCandidateClass(clazz, this.autowiredAnnotationTypes)) {
			return InjectionMetadata.EMPTY;
		}

		List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
		Class<?> targetClass = clazz;

		do {
			final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

			//寻找所有字段中有@Autowired注解
			ReflectionUtils.doWithLocalFields(targetClass, field -> {
				MergedAnnotation<?> ann = findAutowiredAnnotation(field);
				if (ann != null) {
					if (Modifier.isStatic(field.getModifiers())) {
						if (logger.isInfoEnabled()) {
							logger.info("Autowired annotation is not supported on static fields: " + field);
						}
						return;
					}
					boolean required = determineRequiredStatus(ann);
					currElements.add(new AutowiredFieldElement(field, required));
				}
			});
			//寻找所有方法中有@Autowired注解
			ReflectionUtils.doWithLocalMethods(targetClass, method -> {
				Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
				if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
					return;
				}
				MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
				if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
					if (Modifier.isStatic(method.getModifiers())) {
						if (logger.isInfoEnabled()) {
							logger.info("Autowired annotation is not supported on static methods: " + method);
						}
						return;
					}
					if (method.getParameterCount() == 0) {
						if (logger.isInfoEnabled()) {
							logger.info("Autowired annotation should only be used on methods with parameters: " +
									method);
						}
					}
					boolean required = determineRequiredStatus(ann);
					PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
					currElements.add(new AutowiredMethodElement(method, required, pd));
				}
			});

			elements.addAll(0, currElements);
			targetClass = targetClass.getSuperclass();
		}
		while (targetClass != null && targetClass != Object.class);

		return InjectionMetadata.forElements(elements, clazz);
	}
```




- ReferenceAnnotationBeanPostProcessor
	  	  
```java 
public class Ioc {
	@DubboReference
	private UserService userService;
	@PostConstruct
	public void init() {
	System.out.println(userService);
	}
}
``
也是两个步骤，只是这两个步骤是由ReferenceAnnotationBeanPostProcessor它来完成的。
1、@DubboReference注解收集
```java 
    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        if (beanType != null) {
            if (isReferenceBean(beanDefinition)) {
                //mark property value as optional
                List<PropertyValue> propertyValues = beanDefinition.getPropertyValues().getPropertyValueList();
                for (PropertyValue propertyValue : propertyValues) {
                    propertyValue.setOptional(true);
                }
            } else if (isAnnotatedReferenceBean(beanDefinition)) {
                // extract beanClass from java-config bean method generic return type: ReferenceBean<DemoService>
                //Class beanClass = getBeanFactory().getType(beanName);
            } else {
				//收集有@DubboReference注解的属性或者方法并封装成对象
                AnnotatedInjectionMetadata metadata = findInjectionMetadata(beanName, beanType, null);
                metadata.checkConfigMembers(beanDefinition);
                try {
					//这里是对每一个有@DubboReference注解的属性或者方法创建一个ReferenceBean的BeanDefinition对象，因为要完成服务的引用，必须要有@ReferenceBean的实例
                    prepareInjection(metadata);
                } catch (Exception e) {
                    throw new IllegalStateException("Prepare dubbo reference injection element failed", e);
                }
            }
        }
    }
	
	protected AnnotatedInjectionMetadata findInjectionMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {
        // Fall back to class name as cache key, for backwards compatibility with custom callers.
        String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
        // Quick check on the concurrent map first, with minimal locking.
        AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
        if (needsRefreshInjectionMetadata(metadata, clazz)) {
            synchronized (this.injectionMetadataCache) {
                metadata = this.injectionMetadataCache.get(cacheKey);

                if (needsRefreshInjectionMetadata(metadata, clazz)) {
                    if (metadata != null) {
                        metadata.clear(pvs);
                    }
                    try {
                        metadata = buildAnnotatedMetadata(clazz);
                        this.injectionMetadataCache.put(cacheKey, metadata);
                    } catch (NoClassDefFoundError err) {
                        throw new IllegalStateException("Failed to introspect object class [" + clazz.getName() +
                                "] for annotation metadata: could not find class that it depends on", err);
                    }
                }
            }
        }
        return metadata;
    }
	
	private AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata buildAnnotatedMetadata(final Class<?> beanClass) {
        Collection<AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement> fieldElements = findFieldAnnotationMetadata(beanClass);
        Collection<AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement> methodElements = findAnnotatedMethodMetadata(beanClass);
        return new AnnotatedInjectionMetadata(beanClass, fieldElements, methodElements);
    }
```
	
	
```java 	
    protected void prepareInjection(AnnotatedInjectionMetadata metadata) throws BeansException {
        try {
            //find and registry bean definition for @DubboReference/@Reference
            for (AnnotatedFieldElement fieldElement : metadata.getFieldElements()) {
                if (fieldElement.injectedObject != null) {
                    continue;
                }
                Class<?> injectedType = fieldElement.field.getType();
                AnnotationAttributes attributes = fieldElement.attributes;
				//这里就会创建ReferenceBean对象
                String referenceBeanName = registerReferenceBean(fieldElement.getPropertyName(), injectedType, attributes, fieldElement.field);

                //associate fieldElement and reference bean
                fieldElement.injectedObject = referenceBeanName;
                injectedFieldReferenceBeanCache.put(fieldElement, referenceBeanName);

            }

            for (AnnotatedMethodElement methodElement : metadata.getMethodElements()) {
                if (methodElement.injectedObject != null) {
                    continue;
                }
                Class<?> injectedType = methodElement.getInjectedType();
                AnnotationAttributes attributes = methodElement.attributes;
                String referenceBeanName = registerReferenceBean(methodElement.getPropertyName(), injectedType, attributes, methodElement.method);

                //associate fieldElement and reference bean
                methodElement.injectedObject = referenceBeanName;
                injectedMethodReferenceBeanCache.put(methodElement, referenceBeanName);
            }
        } catch (ClassNotFoundException e) {
            throw new BeanCreationException("prepare reference annotation failed", e);
        }
    }
```

- postProcessPropertyValues
```java 
 @Override
    public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

        try {
            AnnotatedInjectionMetadata metadata = findInjectionMetadata(beanName, bean.getClass(), pvs);
            prepareInjection(metadata);
			//依赖注入 
            metadata.inject(bean, beanName, pvs);
        } catch (BeansException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Injection of @" + getAnnotationType().getSimpleName()
                    + " dependencies is failed", ex);
        }
        return pvs;
    }
	
	
	protected AnnotatedInjectionMetadata findInjectionMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {
        // Fall back to class name as cache key, for backwards compatibility with custom callers.
        String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
        // Quick check on the concurrent map first, with minimal locking.
        AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
        if (needsRefreshInjectionMetadata(metadata, clazz)) {
            synchronized (this.injectionMetadataCache) {
                metadata = this.injectionMetadataCache.get(cacheKey);

                if (needsRefreshInjectionMetadata(metadata, clazz)) {
                    if (metadata != null) {
                        metadata.clear(pvs);
                    }
                    try {
                        metadata = buildAnnotatedMetadata(clazz);
                        this.injectionMetadataCache.put(cacheKey, metadata);
                    } catch (NoClassDefFoundError err) {
                        throw new IllegalStateException("Failed to introspect object class [" + clazz.getName() +
                                "] for annotation metadata: could not find class that it depends on", err);
                    }
                }
            }
        }
        return metadata;
    }
	
	public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
		Collection<InjectedElement> checkedElements = this.checkedElements;
		//遍历所有需要依赖注入的集合，这里会调用到bubbo中的源码，因为dubbo中实现了InjectedElement的接口
		Collection<InjectedElement> elementsToIterate =
				(checkedElements != null ? checkedElements : this.injectedElements);
		if (!elementsToIterate.isEmpty()) {
			for (InjectedElement element : elementsToIterate) {
				if (logger.isTraceEnabled()) {
					logger.trace("Processing injected element of bean '" + beanName + "': " + element);
				}
				element.inject(target, beanName, pvs);
			}
		}
	}
	
	
```

总结一下，收集注解会收集@DubboReference注解的属性或方法，然后还会创建跟@DubboReference对应的
ReferenceBean对象，因为要完成代理的生成和服务的引用。

- xml方式整合
dubbo和spring的整合用到了很多spring中提供的扩展点，通过spring中的refresh方法进行触发
1. xml形式首先会进行自定义标签的解析调用refresh方法中的obtainFreshBeanFactory();方法
	a、创建BeanFactory对象，然后解析xml文件  
	a、根据当前解析标签的头信息找到对应的namespaceUri  
	b、加载spring所以jar中的spring.handlers文件。并建立映射关系  
	c、根据namespaceUri从映射关系中找到对应的实现了NamespaceHandler接口的类  
	d、调用类的init方法，init方法是注册了各种自定义标签的解析类  
	e、根据namespaceUri找到对应的解析类，然后调用paser方法完成标签解析  
2. 找到spring.handlers文件  （org\apache\dubbo\dubbo\3.0.2.1\dubbo-3.0.2.1.jar!\META-INF\spring.handlers）
3. spring.handlers该文件中对应着着uri和该类之间的映射关系
4. spring.handlers该文件中有一个DubboNamespaceHandler，它继承了NamespaceHandlerSupport 并重写了init方法和parse方法
5. init方法中会为每个标签生成一个对应的解析类
6. 然后会通过parse方法进行每个标签的解析工作，把标签中的属性进行解析，转化为RootBeanDefinition对象，为了加入到spring容器中进行实例化


- 注解的方式
注解的方式只需要在需要暴露的服务上面加上一个@DubboService注解就可以完成服务的暴露了，配置量非常少，
但是这种方式也有一定的侵入性。然后通过@EnableDubbo注解配置scanBasePackages需要扫描的路径，扫描类上面的@DubboService注解以及@DubboReference注解  

@EnableDubbo 为什么会被spring识别?  

@EnableDubbo中有一个@DubboComponentScan注解 在改注解中有一个import注解(@Import(DubboComponentScanRegistrar.class))
而import注解会被spring扫描到,会在spring的ConfigurationClassPostProcessor类中进行注解的解析，会解析@Import注解
然后会把import中的 DubboComponentScanRegistrar.class类转化为beanDefinition

1. ConfigurationClassPostProcessor类是如何加载到spring容器中的呢
我们以注解的方式启动的时候AnnotationConfigApplicationContext通过该类进行实现，可以看到在该类的构造方法中通过spring上下文对象实例化把ConfigurationClassPostProcessor变成BeanDefinition对象然后加载到ioc容器中。
2. ConfigurationClassPostProcessor类会扫描inport注解，这里还是通过refresh方法进行触发，会调用refresh方法中的invokeBeanFactoryPostProcessors(beanFactory);然后在ConfigurationClassPostProcessor实例化的时候会调用到postProcessBeanDefinitionRegistry方法，会对所有的BeanFefinition类上有@Import注解，进行解析，
所以此时会调用DubboComponentScanRegistrar类，该类实现了ImportBeanDefinitionRegistrar接口，会调用registerBeanDefinitions方法，
该方法其实核心就是注册了三个比较重要的类给了spring容器：
- ReferenceAnnotationBeanPostProcessor (@DubborReference注解的解析)
- DubboBootstrapApplicationListener(负责服务发布)
- ServiceAnnotationPostProcessor（@DubboService注解的解析）
会拿到@EnableDubbo注解，并拿到scanBasePackages属性，进行文件路径的递归扫描，把所有扫描到的类转化为BeanDefinition加载到spring容器中。

2. 实例的依赖注入
 待完成。。。

