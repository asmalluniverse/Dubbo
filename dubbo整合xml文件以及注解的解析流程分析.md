

Դ�����

### dubbo ��spring ����Դ�����

1.����xml�ķ�ʽ��spring�����ϣ��������Ǳ���Ҫ֪��spring��xml�������̣�ֻ��֪�����������dubbo���Զ����ǩ����ν����ģ�
�������Ƿ���spring������ν���xml�ļ������ġ�
```java
public class XmlProvider {

    public static void main(String[] args) throws InterruptedException {
        ClassPathXmlApplicationContext cxa = new ClassPathXmlApplicationContext("applicationProvider.xml");
        System.out.println("dubbo service started.");
        new CountDownLatch(1).await();
    }
}

�Ӹ���Ĺ��췽�����룬����Դ����refresh()�������棺
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
	 *	�÷�����spring������ʼ���ĺ��ķ�������spring������ʼ���ĺ������̣���һ�����͵ĸ���ģ�����ģʽ������
	 *	���ݲ�ͬ�������Ķ��󣬻������ͬ�������Ķ������෽����
	 *
	 *	���������������У�
	 *	ClassPathXmlApplicationContext
	 *	FileSystemXmlApplicationContext
	 *	AnnotationConfigApplicationContext
	 *	EmbeddedWebApplicationContext(springboot)
	 *
	 * ������Ҫ�̶ȣ�
	 *  0������Ҫ�����Բ���
	 *  1��һ����Ҫ���ɿ��ɲ���
	 *  5���ǳ���Ҫ��һ��Ҫ��
	 *  ������� ����Ҫ�̶� 5
	 * */
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			//Ϊ������ʼ����׼������Ҫ�̶ȣ�0
			// Prepare this context for refreshing.
			prepareRefresh();

			/*
			   ��Ҫ�̶ȣ�5
			  1������BeanFactory����
			* 2��xml����
			* 	��ͳ��ǩ������bean��import��
			* 	�Զ����ǩ���� �磺<context:component-scan base-package="com.xiangxue.jack"/>
			* 	�Զ����ǩ�������̣�
			* 		a�����ݵ�ǰ������ǩ��ͷ��Ϣ�ҵ���Ӧ��namespaceUri
			* 		b������spring����jar�е�spring.handlers�ļ���������ӳ���ϵ
			* 		c������namespaceUri��ӳ���ϵ���ҵ���Ӧ��ʵ����NamespaceHandler�ӿڵ���
			* 		d���������init������init������ע���˸����Զ����ǩ�Ľ�����
			* 		e������namespaceUri�ҵ���Ӧ�Ľ����࣬Ȼ�����paser������ɱ�ǩ����
			*
			* 3���ѽ���������xml��ǩ��װ��BeanDefinition����
			* */
			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			/*
			 * ��beanFactory����һЩ����ֵ�����Բ���
			 * */
			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				/*
				 * BeanDefinitionRegistryPostProcessor
				 * BeanFactoryPostProcessor
				 * ��ɶ��������ӿڵĵ���
				 * */
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				/*
				 * ��ʵ����BeanPostProcessor�ӿڵ���ʵ���������Ҽ��뵽BeanFactory��
				 * */
				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				/*
				 * ���ʻ�,��Ҫ�̶�2
				 * */
				// Initialize message source for this context.
				initMessageSource();

				//��ʼ���¼�������
				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				//��������������ģ�����ģʽ����Ϊ��springboot�У������������������Ƕtomcat������
				// Initialize other special beans in specific context subclasses.
				onRefresh();

				/*
				 * ���¼���������ע���¼���
				 * */
				// Check for listener beans and register them.
				registerListeners();

				/*
				 * ���������spring������Ҫ�ķ�����û��֮һ
				 * �����������һ��Ҫ���Ҫ���忴
				 * 1��beanʵ��������
				 * 2��ioc
				 * 3��ע��֧��
				 * 4��BeanPostProcessor��ִ��
				 * 5��Aop�����
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
�÷�����Ҫ����xml�ļ�����
```java 
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	//���ķ��������������Ҫ�̶ȣ�5   ���õ�����AbstractRefreshableApplicationContext��refreshBeanFactory����
	refreshBeanFactory();
	return getBeanFactory();
}


@Override
protected final void refreshBeanFactory() throws BeansException {
	//���BeanFactory��Ϊ�գ������BeanFactory�������ʵ��
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
		//BeanFactory ʵ������
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		//�����Ƿ����ѭ������ allowCircularReferences
		//�Ƿ�����ʹ����ͬ��������ע�᲻ͬ��beanʵ��.
		customizeBeanFactory(beanFactory);
		//����xml������xml�еı�ǩ��װ��BeanDefinition����
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
	//����xml�Ľ�������������һ��ί��ģʽ
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// Configure the bean definition reader with this context's
	// resource loading environment.
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	//���ﴫһ��this��ȥ����ΪApplicationContext��ʵ����ResourceLoader�ӿڵ�
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	// Allow a subclass to provide custom initialization of the reader,
	// then proceed with actually loading the bean definitions.
	initBeanDefinitionReader(beanDefinitionReader);
	//��Ҫ���������  ��Ҫ�̶� 5
	loadBeanDefinitions(beanDefinitionReader);
	}
	
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
		throws BeanDefinitionStoreException {

	try {
	//��inputSource�����װ��Document����
		Document doc = doLoadDocument(inputSource, resource);
		����Document�е�Ԫ�ط�װ��BeaDefinition����
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
	//��Ҫ���������
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
	//��ǩ���� 
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
				    //Ĭ�ϱ�ǩ����
					parseDefaultElement(ele, delegate);
				}
				else {
				    //�Զ����ǩ����
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
	//��ȡ�������ռ�
	String namespaceUri = getNamespaceURI(ele);
	if (namespaceUri == null) {
		return null;
	}
	//���������ռ��ҵ���Ӧ�������ռ䴦����
	NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
	if (handler == null) {
		error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
		return null;
	}
	//�����û��Զ���Ĵ��������н���
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
        // ��ȡ�����Ѿ����õ�handlerӳ��
        Map<String, Object> handlerMappings = getHandlerMappings();
        // ���������ռ��ȡ��Ӧ����Ϣ
        Object handlerOrClassName = handlerMappings.get(namespaceUri);
        if (handlerOrClassName == null) {
            return null;
        }
        else if (handlerOrClassName instanceof NamespaceHandler) {
            // �Ѿ�����������ֱ�Ӵӻ����л�ȡ
            return (NamespaceHandler) handlerOrClassName;
        }
        else {
            // û�����������ģ��򷵻ص�����·��
            String className = (String) handlerOrClassName;
            try {
                // ʹ�÷��佫��·��ת��Ϊ��
                Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
                if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
                    throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
                            "] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
                }
                // ��ʼ��������
                NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
                // �����Զ����NameSpaceHandler��init����
                namespaceHandler.init();
                // ���뻺��
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
        // ���û�л�����ʼ���л���
        if (this.handlerMappings == null) {
            synchronized (this) {
                if (this.handlerMappings == null) {
                    try {
                        // this.handlerMappingsLocation�ڹ��캯���г�ʼ��ΪMETA-INF/spring.handlers
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
��������֪����Ϊʲô��Ҫ��META-INF/spring.handlers�ļ������������ˡ���Ϊ�ڼ������еĴ����������õ�Ĭ��Ŀ¼����META-INF/spring.handlers;
����ȡ���Զ����NameSpaceHandler֮��Ϳ��Խ��д�������ʼ���������ˡ�ʵ���Ͼ��ǵ���init���������������ڷ���dubbo����ν���xml�ļ������ģ��ͺܼ��ˡ�




1.dubbo����ν���xml�ļ���
(1)�ҵ�spring.handlers�ļ�  ��org\apache\dubbo\dubbo\3.0.2.1\dubbo-3.0.2.1.jar!\META-INF\spring.handlers��
(2)spring.handlers���ļ��ж�Ӧ����uri�͸���֮���ӳ���ϵ������ͨ�������init��������xml������ 
��ô�����init������ʲôʱ��ִ�е��أ� ��ʵ��ͨ��spring ����init�����ǻ���е��á�
dubbo����xml�ļ�������ʱ��ʹ�õľ���spring����չ���ƣ�

�Ա�spring �е�xml �ļ����� dubbo ����DubboNamespaceHandler��  spring �е������ AbstractApplicationContext �е�refresh���� 
�е�ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory(); ���еģ�
Ҳ����������spring ��ʱ����ø÷������Խ���dubbo �����ļ��Ľ���

dubbo ��ÿ����ǩ����Ӧ��һ��ӳ����
- init����
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
�����˾��ǽ���ǩ��Ӧ�Ľ�������������������ڽ�������ǩ��ʱ���֪��ί�и���Ӧ�Ľ�������������ʾ���Ϊ������ Spring �� BeanDefinition��Ȼ������ Spring ���մ�����Ӧ�Ķ���

- parse����
����xml�ļ��Ľ�����parse��������ÿ����ǩ�Ľ�������
```java 
public BeanDefinition parse(Element element, ParserContext parserContext) {
	BeanDefinitionRegistry registry = parserContext.getRegistry();
	registerAnnotationConfigProcessors(registry);
	/**
	 * @since 2.7.8
	 * issue : https://github.com/apache/dubbo/issues/6275
	 */
	
	//���ķ�����ע��bean ����dubbo�еĺ�����ת��ΪBeaDifition���󣬼��ص�spring�����н���ʵ����
	registerCommonBeans(registry);
	
	���ø����parse����
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

	//DubboConfigAliasPostProcessor����ǩ�����࣬���б�ǩ�Ľ���
	// TODO Whether DubboConfigAliasPostProcessor can be removed ?
	// Since 2.7.4 [Feature] https://github.com/apache/dubbo/issues/5093
	registerInfrastructureBean(registry, DubboConfigAliasPostProcessor.BEAN_NAME,
			DubboConfigAliasPostProcessor.class);

	//dubbo ����������spring������ɺ����ø���ķ������з���ķ���
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

- ���ø���NamespaceHandlerSupport��parse����
```java 

public BeanDefinition parse(Element element, ParserContext parserContext) {
    //�ҵ���ǩ��Ӧ�Ľ�����
	BeanDefinitionParser parser = findParserForElement(element, parserContext);
	//����parser�������б�ǩ�Ľ���
	return (parser != null ? parser.parse(element, parserContext) : null);
}
```

- findParserForElement
```java 
	@Nullable
private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
	�ҵ�localNameҲ��������init������put��parser�����е�key ��ʵ���Ǳ�ǩ�е� <dubbo:service interface="cn.enjoy.service.UserService" ref="userServiceImpl" timeout="2000"> dubbo:service �е�sercice��
	ͨ��ӳ���ϵ��ȡ��Ӧ�Ľ����࣬�뵱�ڷ����� DubboBeanDefinitionParser ���󣬲������˸÷�����parse����
	String localName = parserContext.getDelegate().getLocalName(element);
	BeanDefinitionParser parser = this.parsers.get(localName);
	if (parser == null) {
		parserContext.getReaderContext().fatal(
				"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
	}
	return parser;
}

- DubboBeanDefinitionParser�е�parse����
DubboBeanDefinitionParser�е�parse���������Կ����÷������ص���һ��RootBeanDefinition����Ҳ����˵���÷�����ѽ���xml��������װ��ΪbeanDefinition��������spring����ʵ��������
����������ǩ��Ŀ����Ϊ�˰Ѷ�Ӧ��ǩ�����������ʵ����
@Override
public BeanDefinition parse(Element element, ParserContext parserContext) {
    //�÷������ǶԱ�ǩ�е����Խ��н�����ת��ΪRootBeanDefinition����Ϊ�˼��뵽spring�����н���ʵ����
	return parse(element, parserContext, beanClass, true);
}
```
	
				
2. ���񷢲�  
���񷢲���spring ��ʲô��ϵ��service������service��ǩ�������з�����
registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class));

����ServiceBean��ʵ����spring �е�һϵ�нӿڣ�����InitializingBean�ӿ����Ƕ�֪����
InitializingBean�ӿ�Ϊbean�ṩ�˳�ʼ�������ķ�ʽ����ֻ����afterPropertiesSet���������Ǽ̳иýӿڵ��࣬�ڳ�ʼ��bean��ʱ�򶼻�ִ�и÷�����

- afterPropertiesSet
��ʵ�ڸ÷����еĺ����߼�������ConfigManager�����ServiceBean��ʵ����������spring�������������
dubbo���¼�������DubboBootstrapApplicationListener�ͻ����ConfigManager�е�ServiceBeanʵ�������
����ķ���
```java
public void afterPropertiesSet() throws Exception {
	if (StringUtils.isEmpty(getPath())) {
		if (StringUtils.isNotEmpty(getInterface())) {
			setPath(getInterface());
		}
	}
	//register service bean and set bootstrap
	//����DubboBootstrap���󣬲��Ѹö������õ�ServiceBean�У����ҰѸ��ֱ�ǩ��������뵽ConfigManager������
	DubboBootstrap.getInstance().service(this);
}

    public DubboBootstrap service(ServiceConfig<?> serviceConfig) {
        serviceConfig.setBootstrap(this);
        configManager.addService(serviceConfig);
        return this;
    }
```
	
���϶���Ϊ���񷢲���׼������û�н��з���ķ���������ķ�����ͨ��һ��ʱ���������ʵ�ֵģ���spring ����������ɺ�Ž��еķ��񷢲���
�� AbstractApplicationContext �е�refresh���� ��ִ��finishRefresh();����
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
	//spring�������غú󷢲������¼�
	publishEvent(new ContextRefreshedEvent(this));

	// Participate in LiveBeansView MBean, if active.
	LiveBeansView.registerApplicationContext(this);


}

��������·���ã�
publishEvent(new ContextRefreshedEvent(this)); -> multicastEvent(applicationEvent, eventType);  
-> invokeListener(listener, event)  - > doInvokeListener(listener, event); -> onApplicationEvent(event); 
��ʱ�ͻ���õ�bubbo�еļ�����DubboBootstrapApplicationListener�е�onApplicationEvent����
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
		//����ķ���
		dubboBootstrap.start();
	}
} 
```

- start()
�÷������еķ��񷢲�
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
			//���ע�����ĺ�Ԫ�������ĵĳ�ʼ��
			initialize();

			if (logger.isInfoEnabled()) {
				logger.info(NAME + " is starting...");
			}
			//����ķ���������ConfigManager�м��ص���������з���ķ���
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
	// 1. export Dubbo Services  ��ǩ�ķ���
	exportServices();

	// If register consumer instance or has exported services
	if (isRegisterConsumerInstance() || hasExportedServices()) {
		// 2. export MetadataService
		exportMetadataService();
		// 3. Register the local ServiceInstance if required
		registerServiceInstance();
	}

	//��ǩ������
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
����ͺ������configManager�����ˣ���ΪconfigManager�б��������е�serviceBeanʵ����xml�е�������Ϣ���������õ�xml�����õķ���ʵ�����Ϳ��Խ��з�������
```java 
private void exportServices() {
	for (ServiceConfigBase sc : configManager.getServices()) {
		// TODO, compatible with ServiceConfig.export()
		ServiceConfig<?> serviceConfig = (ServiceConfig<?>) sc;
		serviceConfig.setBootstrap(this);
		if (!serviceConfig.isRefreshed()) {
		//���Ե�ע��
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
				//������������ServerBean��export�������з���ķ���
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
���ˣ�dubbo��xml��ʽ��spring���Ͼͷ������	


### dubboע��ķ�ʽԴ�����
ע��ķ�ʽֻ��Ҫ����Ҫ��¶�ķ����������һ��@DubboServiceע��Ϳ�����ɷ���ı�¶�ˣ��������ǳ��٣�
�������ַ�ʽҲ��һ���������ԡ�Ȼ��ͨ��@EnableDubboע�⣬ɨ���������@DubboServiceע���Լ�@DubboReferenceע��

@EnableDubbo Ϊʲô�ᱻspringʶ��

1. @EnableDubbo����һ��@DubboComponentScanע�� �ڸ�ע������һ��importע��(@Import(DubboComponentScanRegistrar.class))
��importע��ᱻspringɨ�赽,����spring��ConfigurationClassPostProcessor���н���ע��Ľ����������@Importע��
Ȼ����import�е� DubboComponentScanRegistrar.class��ת��ΪbeanDefinition

- ConfigurationClassPostProcessor
```java
public class AnnotationProvider
{
    public static void main( String[] args ) throws InterruptedException {
	    //��ע��ķ�ʽ����
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
	//ͨ��spring�����Ķ���ʵ������ConfigurationClassPostProcessor���BeanDefinition����
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
	//�����������
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

	//ConfigurationClassPostProcessor ת��ΪRootBeanDefinition����Ϊ�˼��뵽spring�����н���ʵ����
	if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
	}
```

2. ConfigurationClassPostProcessor��@Importע���ɨ��
����ConfigurationClassPostProcessor����һ��BeanDefinitionRegistryPostProcessor���͵ģ�������spring����
���������ȱ�ʵ������ʵ�����ĵط���refresh���ķ����ģ�

```java
/*
 * BeanDefinitionRegistryPostProcessor
 * BeanFactoryPostProcessor
 * ��ɶ��������ӿڵĵ���
 * */
// Invoke factory processors registered as beans in the context.
invokeBeanFactoryPostProcessors(beanFactory);
```

���Ե�ConfigurationClassPostProcessorʵ������ʱ��͵��õ�postProcessBeanDefinitionRegistry������������
�����£�

spring�л�ִ�����´����importע����н���
```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
	List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
	//��ȡ����beanNames
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
		//@Bean @Import �ڲ��� @ImportedResource ImportBeanDefinitionRegistrar���崦���߼�
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
���������淽�������this.reader.loadBeanDefinitions(configClasses);���д��������BeanDefinition�ж�Ӧ����
�����@Importע������˽�����������spring���ܹ�ɨ���@EnableDubboע����	
	
- DubboComponentScanRegistrar
DubboComponentScanRegistrar����ͨ��@EnableDubbo�����@DubboComponentScanע��import�����ģ�
����һ��ImportBeanDefinitionRegistrar���͵ģ����е�����������Ժ�ͻᱻspring���õ�����
registerBeanDefinitions�������������£�
	
```java
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

 
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

		// @since 2.7.6 Register the common beans ʵ��һЩ�������ʵ����:ReferenceAnnotationBeanPostProcessor  DubboBootstrapApplicationListener
		registerCommonBeans(registry);

		// AnnotationMetadata �ö�����õ�@EnableDubbo(scanBasePackages = "cn.enjoy") ע�������valueֵ������·����
		Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

		//ServiceAnnotationPostProcessor
		registerServiceAnnotationPostProcessor(packagesToScan, registry);
	}

	private void registerServiceAnnotationPostProcessor(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
		//�Ѹ���ת��Ϊbeanfinition ���� ServiceAnnotationPostProcessor
		BeanDefinitionBuilder builder = rootBeanDefinition(ServiceAnnotationPostProcessor.class);
		builder.addConstructorArgValue(packagesToScan);
		builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();

		//ע�ᵽ��ע������registry ������ʵ��������
		BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry);
    }

}
```

�÷�����ʵ���ľ���ע���������Ƚ���Ҫ�������spring������
1. ReferenceAnnotationBeanPostProcessor
2. DubboBootstrapApplicationListener
3. ServiceAnnotationPostProcessor

DubboBootstrapApplicationListener�����ǰ�����Ƿ���������������spring����������ɺ�ͨ��spring����һ��
spring����������ɵ��¼���Ȼ����ಶ���¼���ͨ�������¼�����ɷ���ķ��������õģ�����Ͳ���׸���ˡ�
������Ҫ����ServiceAnnotationPostProcessor��ReferenceAnnotationBeanPostProcessor
	
- ServiceAnnotationPostProcessor�����

**���������þ�������ɨ���������@DubboServiceע��ģ���ֻ��������ܡ�**
	
���� public class ServiceAnnotationPostProcessor implements BeanDefinitionRegistryPostProcessor ʵ����һ��spring�ṩ�Ľӿڣ�
�ýӿ��ṩ��void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;������������ʵ������ʱ��ִ��,
��ô���ǾͿ��� ServiceAnnotationPostProcessor ���е�postProcessBeanDefinitionRegistry ����
	
```java 
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
	this.registry = registry;

	Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

	if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
		//����ɨ�����
		scanServiceBeans(resolvedPackagesToScan, registry);
	} else {
		if (logger.isWarnEnabled()) {
			logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
		}
	}

}
```

- scanServiceBeans��������
```java 
/**
 * Scan and registers service beans whose classes was annotated {@link Service}
 *
 * @param packagesToScan The base packages to scan
 * @param registry       {@link BeanDefinitionRegistry}
 */
private void scanServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

	//����һ��ɨ���������dubboɨ������ʵ���Ǽ̳���spring��ClassPathBeanDefinitionScannerɨ����
	DubboClassPathBeanDefinitionScanner scanner =
			new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

	BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);
	scanner.setBeanNameGenerator(beanNameGenerator);
	//��ɨ���������Ҫɨ���ע�����ͣ������ע�����������֣�apache��@Service��@DubboService������Ͱ͵�@Service
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

		//������Ǻ��ĵ�ɨ�����
		//ɨ��ĺ������̣���ʵ���ǵݹ����ļ��Ĺ��̣�����ҵ����ļ�����ݹ��ң�������ļ�����������޶�������
		//��������󣬸��ݷ�������ж������Ƿ���DubboServiceע�⣬�������Ѹ�����BeanDefinition
		// Registers @Service Bean first
		scanner.scan(packageToScan);

		// Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not.
		//����ǰ���ɨ���ҵ���BeanDefinition���ϣ���ȡ�������
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
			//���д������Ҫ����ΪҪ��ɷ���¶��ô�ͱ�����õ�ServiceBean�е�export����
			//���д�����Ǹ�����ע������BeanDefinition�������������Ӧ��ServiceBean��
			//BeanDefinition����
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

������ķ���������֪����ɨ�赽��@DubboServiceע������Ժ�ʵ���ϻᴴ���������Ӧ��ServiceBean
��ʵ����Ҳ������һ��@DubboServiceע�����ͻ��Ӧһ��ServiceBean��ʵ����spring�����С���ô�ֻص���
����xml�����������ˣ�ServiceBeanʵ�����ߵ�afterPropertiesSet������Ȼ��spring������������ߵ�dubbo
�ļ�������ɷ��񷢲���	   
	   
	  
- ReferenceAnnotationBeanPostProcessor �����
ReferenceAnnotationBeanPostProcessor�����þ������@DubboReference���Ի��߷���������ע�룬������ע
�������Ҳ����ȫ����spring�ģ��������Ǳ���Ҫ����spring������ע�����̣���ʵspring������ע����ľ����㣻
1. ע����ռ�
2. ʵ����ע��

1. spring�е�����ע��
��@Autowiredע�������������spring������ע�룬���磺
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

1. @Autowiredע���ռ�  
ע���ռ��ĺ�������
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
					//������ע���װ�����
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

��ʵdubbo������ע�����̸�spring��һģһ���ģ�Ҳ�ǽ���BeanPostProcessor���͵Ľӿ���ʵ�֣�ֻ���������

```java 
public class AnnotationProvider
{
    public static void main( String[] args ) throws InterruptedException {
		//ͨ��ע��ķ�ʽ����
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

����refreash()�����У�
```java 	
	
/*
 * ���������spring������Ҫ�ķ�����û��֮һ
 * �����������һ��Ҫ���Ҫ���忴
 * 1��beanʵ��������
 * 2��ioc
 * 3��ע��֧��
 * 4��BeanPostProcessor��ִ��
 * 5��Aop�����
 * */
// Instantiate all remaining (non-lazy-init) singletons.
finishBeanFactoryInitialization(beanFactory);


protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		//��������ת����
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		//��ʱ��Ҫ��
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		//��ʱ����
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		//�ص㿴�����������Ҫ�̶ȣ�5 ����ʵ����
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

				// Create bean instance.  ��������ʵ���� 
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
						throw new IllegalStateException("No scope name defined for bean ��" + beanName + "'");
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
			//�����ͨ�����ַ�ʽ����ʵ���� �����Ǵ�ʱ��û�н���ioc�Ĳ���
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		//AutowiredAnnotationBeanPostProcessor�ö������������ע��ĺ�����
		//�����ǽ���ioc����ע��ĺ�������
		//ioc ����ע�����������̣� 1.ע���ռ� 2. ����ע��
		Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					//ע���ռ�  
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
			//����������档���ѭ������������
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			//ioc  di �ĺ����߼�
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
					@Autowiredע����ռ�  ����ע��Ĺ���
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						//�ϰ汾ʹ�õ����������ɵ�����ע�� @Autowired��֧�֣��Լ�dubbo��@DubboReferenceע��  �������õ� ReferenceAnnotationBeanPostProcessor���е�postProcessPropertyValues���� 
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
		//�������е�beanPostProcessorʵ��
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof MergedBeanDefinitionPostProcessor) {
				MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
				//���뵽AutowiredAnnotationBeanPostProcessor ���� @Autowiredע��Ľ��� 
				//dubbo��ͨ��ReferenceAnnotationBeanPostProcessor ����@DubboReferenceע����ռ� 
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
		//cacheKey ��Ӧ�ľ��������ƣ��ȴӻ������ã��������ò�����ȥbuild�������鿴���е����Ի��߷����Ƿ���@Autowiredע��
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
					//���뵽������ 
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

			//Ѱ�������ֶ�����@Autowiredע��
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
			//Ѱ�����з�������@Autowiredע��
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
Ҳ���������裬ֻ����������������ReferenceAnnotationBeanPostProcessor������ɵġ�
1��@DubboReferenceע���ռ�
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
				//�ռ���@DubboReferenceע������Ի��߷�������װ�ɶ���
                AnnotatedInjectionMetadata metadata = findInjectionMetadata(beanName, beanType, null);
                metadata.checkConfigMembers(beanDefinition);
                try {
					//�����Ƕ�ÿһ����@DubboReferenceע������Ի��߷�������һ��ReferenceBean��BeanDefinition������ΪҪ��ɷ�������ã�����Ҫ��@ReferenceBean��ʵ��
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
				//����ͻᴴ��ReferenceBean����
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
			//����ע�� 
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
		//����������Ҫ����ע��ļ��ϣ��������õ�bubbo�е�Դ�룬��Ϊdubbo��ʵ����InjectedElement�Ľӿ�
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

�ܽ�һ�£��ռ�ע����ռ�@DubboReferenceע������Ի򷽷���Ȼ�󻹻ᴴ����@DubboReference��Ӧ��
ReferenceBean������ΪҪ��ɴ�������ɺͷ�������á�

- xml��ʽ����
dubbo��spring�������õ��˺ܶ�spring���ṩ����չ�㣬ͨ��spring�е�refresh�������д���
1. xml��ʽ���Ȼ�����Զ����ǩ�Ľ�������refresh�����е�obtainFreshBeanFactory();����
	a������BeanFactory����Ȼ�����xml�ļ�  
	a�����ݵ�ǰ������ǩ��ͷ��Ϣ�ҵ���Ӧ��namespaceUri  
	b������spring����jar�е�spring.handlers�ļ���������ӳ���ϵ  
	c������namespaceUri��ӳ���ϵ���ҵ���Ӧ��ʵ����NamespaceHandler�ӿڵ���  
	d���������init������init������ע���˸����Զ����ǩ�Ľ�����  
	e������namespaceUri�ҵ���Ӧ�Ľ����࣬Ȼ�����paser������ɱ�ǩ����  
2. �ҵ�spring.handlers�ļ�  ��org\apache\dubbo\dubbo\3.0.2.1\dubbo-3.0.2.1.jar!\META-INF\spring.handlers��
3. spring.handlers���ļ��ж�Ӧ����uri�͸���֮���ӳ���ϵ
4. spring.handlers���ļ�����һ��DubboNamespaceHandler�����̳���NamespaceHandlerSupport ����д��init������parse����
5. init�����л�Ϊÿ����ǩ����һ����Ӧ�Ľ�����
6. Ȼ���ͨ��parse��������ÿ����ǩ�Ľ����������ѱ�ǩ�е����Խ��н�����ת��ΪRootBeanDefinition����Ϊ�˼��뵽spring�����н���ʵ����


- ע��ķ�ʽ
ע��ķ�ʽֻ��Ҫ����Ҫ��¶�ķ����������һ��@DubboServiceע��Ϳ�����ɷ���ı�¶�ˣ��������ǳ��٣�
�������ַ�ʽҲ��һ���������ԡ�Ȼ��ͨ��@EnableDubboע������scanBasePackages��Ҫɨ���·����ɨ���������@DubboServiceע���Լ�@DubboReferenceע��  

@EnableDubbo Ϊʲô�ᱻspringʶ��?  

@EnableDubbo����һ��@DubboComponentScanע�� �ڸ�ע������һ��importע��(@Import(DubboComponentScanRegistrar.class))
��importע��ᱻspringɨ�赽,����spring��ConfigurationClassPostProcessor���н���ע��Ľ����������@Importע��
Ȼ����import�е� DubboComponentScanRegistrar.class��ת��ΪbeanDefinition

1. ConfigurationClassPostProcessor������μ��ص�spring�����е���
������ע��ķ�ʽ������ʱ��AnnotationConfigApplicationContextͨ���������ʵ�֣����Կ����ڸ���Ĺ��췽����ͨ��spring�����Ķ���ʵ������ConfigurationClassPostProcessor���BeanDefinition����Ȼ����ص�ioc�����С�
2. ConfigurationClassPostProcessor���ɨ��inportע�⣬���ﻹ��ͨ��refresh�������д����������refresh�����е�invokeBeanFactoryPostProcessors(beanFactory);Ȼ����ConfigurationClassPostProcessorʵ������ʱ�����õ�postProcessBeanDefinitionRegistry������������е�BeanFefinition������@Importע�⣬���н�����
���Դ�ʱ�����DubboComponentScanRegistrar�࣬����ʵ����ImportBeanDefinitionRegistrar�ӿڣ������registerBeanDefinitions������
�÷�����ʵ���ľ���ע���������Ƚ���Ҫ�������spring������
- ReferenceAnnotationBeanPostProcessor (@DubborReferenceע��Ľ���)
- DubboBootstrapApplicationListener(������񷢲�)
- ServiceAnnotationPostProcessor��@DubboServiceע��Ľ�����
���õ�@EnableDubboע�⣬���õ�scanBasePackages���ԣ������ļ�·���ĵݹ�ɨ�裬������ɨ�赽����ת��ΪBeanDefinition���ص�spring�����С�

2. ʵ��������ע��
 ����ɡ�����

