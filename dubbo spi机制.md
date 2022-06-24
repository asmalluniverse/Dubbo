### dubbo spi 原理分析

1. 什么是spi   
SPI (Service Provider Interface)，主要是用来在框架中使用的，最常见和莫过于我们在访问数据库时候用到的java.sql.Driver接口了。你想一下首先市面上的数据库五花八门，不同的数据库底层协议的大不相同，所以首先需要定制一个接口，来约束一下这些数据库，使得 Java 语言的使用者在调用数据库的时候可以方便、统一的面向接口编程。

2. SPI的运用场景   
当你在写核心代码的时候，如果某个点有涉及到会根据参数的不同走不同的逻辑，如果没有SPI，你可能会在代码里面写大量的if else代码，这样代码就非常不灵活，假设有一天又新增了一种逻辑，代码里面也要跟着改，这个就违背了开闭原则，SPI的出现就是解决这种扩展问题的，你可以把实现类全部都配置到配置文件中，然后在核心代码里面就只要加载配置文件，然后根据入参跟加载的这些类进行匹配，如果匹配的就走该逻辑，这样如果有一天新增了逻辑，核心代码是不用变的，唯一变的就是自己工程里面的配置文件和新增类，符合了开闭原则。很多框架中也运用了spi的技术实现扩展

3. java spi  
Java SPI 就是这样做的，约定在 Classpath 下的 META-INF/services/ 目录里创建一个以服务接口命名的文件，然后文件里面记录的是此 jar 包提供的具体实现类的全限定名。这样当我们引用了某个 jar 包的时候就可以去找这个 jar 包的 META-INF/services/ 目录，再根据接口名找到文件，然后读取文件里面的内容去进行实现类的加载与实例化。


- 代码示例 
我们先定义一个Log接口，用于不同类型的日志打印，然后实现了LogBack和SL4J两种类型的日志打印实现类
```java  

package cn.train.javaspi;

public interface Log {

    String type();

    void debug();

    void info();
}

package cn.train.javaspi;


public class LogBackLog implements Log {
    @Override
    public String type() {
        return "LogBack";
    }

    @Override
    public void debug() {
        System.out.println("LogBack debug log");
    }

    @Override
    public void info() {
        System.out.println("LogBack info log");
    }
}

package cn.train.javaspi;

public class Sl4jLog implements  Log {
    @Override
    public String type() {
        return "sl4j";
    }

    @Override
    public void debug() {
        System.out.println("sl4f debug log");
    }

    @Override
    public void info() {
        System.out.println("sl4f info log");
    }
}

package cn.train.javaspi;

import java.io.IOException;
import java.util.Iterator;
import java.util.Scanner;
import java.util.ServiceLoader;


public class LogTest {

    public static void main(String[] args) throws IOException {
        ServiceLoader<Log> serviceLoader = ServiceLoader.load(Log.class);

        Scanner sc = new Scanner(System.in);
        System.out.println("type：");
        String type = sc.nextLine();
        Iterator<Log> iterator = serviceLoader.iterator();
        while (iterator.hasNext()){
            Log log = iterator.next();
            if (log.type().toLowerCase().trim().equals(type)){
                log.debug();
                log.info();
            }
        }

    }
}
```
```shell
然后我在 META-INF/services/ 目录下建了个以接口全限定名命名的文件，路径和内容如下：
路径：src/main/resources/META-INF/services/cn.train.javaspi.Log
内容：
cn.train.javaspi.LogBackLog
cn.train.javaspi.Sl4jLog
```
当我们运行LogTest类的时候根据输入的类型可以走不同的实现类

- Java SPI 源码分析  
从上面我的示例中可以看到ServiceLoader.load()其实就是 Java SPI 入口，我们来看看到底做了什么操作。  

```java 
public static <S> ServiceLoader<S> load(Class<S> service) {
	//获取当前线程的ClossLoader
	ClassLoader cl = Thread.currentThread().getContextClassLoader();
	//加载实现类 入参是接口的类名称和ClassLoader
	return ServiceLoader.load(service, cl);
}

public static <S> ServiceLoader<S> load(Class<S> service,
										ClassLoader loader)
{
	return new ServiceLoader<>(service, loader);
}

private ServiceLoader(Class<S> svc, ClassLoader cl) {
	service = Objects.requireNonNull(svc, "Service interface cannot be null");
	//判断当前线程的ClassLoader 是否为空，如果为空就使用系统的ClassLoader
	loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
	acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
	reload();
}

public void reload() {
	//provider 就是一个 LinkedHashMap，相当于清理缓存
	providers.clear();
	//创建一个LazyIterator，把当前的接口类名称和ClassLoader设置到该类中
	lookupIterator = new LazyIterator(service, loader);
}
```

那现在重点就是 LazyIterator了，从上面代码可以看到我们调用了 hasNext() 来做实例循环，通过 next() 得到一个实例。而 LazyIterator 其实就是 Iterator 的实现类。我们来看看它到底干了啥。

- hasNext()方法解析  

```java 
public boolean hasNext() {
	if (acc == null) {
		return hasNextService();
	} else {
		PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
			public Boolean run() { return hasNextService(); }
		};
		return AccessController.doPrivileged(action, acc);
	}
}

private static final String PREFIX = "META-INF/services/";

private boolean hasNextService() {
	if (nextName != null) {
		return true;
	}
	if (configs == null) {
		try {
			//获取文件的全路径 ：META-INF/services/cn.train.javaspi.Log
			String fullName = PREFIX + service.getName();
			if (loader == null)
				configs = ClassLoader.getSystemResources(fullName);
			else 
			    //加载配置文件
				configs = loader.getResources(fullName);
		} catch (IOException x) {
			fail(service, "Error locating configuration files", x);
		}
	}
	//遍历文件内容
	while ((pending == null) || !pending.hasNext()) {
		if (!configs.hasMoreElements()) {
			return false;
		}
		//解析文件内容
		pending = parse(service, configs.nextElement());
	}
	//赋值实现类的全路径限定名
	nextName = pending.next();
	return true;
}
```
可以看到这个方法其实就是在约定好的地方找到接口对应的文件，然后加载文件并且解析文件里面的内容。

- next方法解析  
```java 
public S next() {
	if (acc == null) {
		return nextService();
	} else {
		PrivilegedAction<S> action = new PrivilegedAction<S>() {
			public S run() { return nextService(); }
		};
		return AccessController.doPrivileged(action, acc);
	}
}

private S nextService() {
	if (!hasNextService())
		throw new NoSuchElementException();
	////赋值实现类的全路径限定名
	String cn = nextName;
	nextName = null;
	Class<?> c = null;
	try {
		//反射创建类
		c = Class.forName(cn, false, loader);
	} catch (ClassNotFoundException x) {
		fail(service,
			 "Provider " + cn + " not found");
	}
	if (!service.isAssignableFrom(c)) {
		fail(service,
			 "Provider " + cn  + " not a subtype");
	}
	try {
		S p = service.cast(c.newInstance());
		//放入到hashmap中 进行缓存
		providers.put(cn, p);
		//返回对应的实现类
		return p;
	} catch (Throwable x) {
		fail(service,
			 "Provider " + cn + " could not be instantiated",
			 x);
	}
	throw new Error();          // This cannot happen
}
```
就是通过文件里填写的全限定名加载类，并且创建其实例放入缓存之后返回实例。  

**整体的 Java SPI 的源码解析已经完毕，是不是很简单？就是约定一个目录，根据接口名去那个目录找到文件，文件解析得到实现类的全限定名，然后循环加载实现类和创建其实例。**  

**这里我们可以思考下Java SPI 哪里不好?**  

相信大家一眼就能看出来，Java SPI 在查找扩展实现类的时候遍历 SPI 的配置文件并且将实现类全部实例化，假设一个实现类初始化过程比较消耗资源且耗时，但是你的代码里面又用不上它，这就产生了资源的浪费。所以说 Java SPI 无法按需加载实现类。那么dubbo spi机制就可以实现，西面我们来介绍dubbo中的spi机制。

4. dubbo spi机制

Dubbo SPI 除了可以按需加载实现类之外，增加了 IOC 和 AOP 的特性，还有个自适应扩展机制。  

我们先来看一下 Dubbo 对配置文件目录的约定，不同于 Java SPI ，Dubbo 分为了三类目录。  
1. META-INF/services/ 目录：该目录下的 SPI 配置文件是为了用来兼容 Java SPI 。
2. META-INF/dubbo/ 目录：该目录存放用户自定义的 SPI 配置文件。
3. META-INF/dubbo/internal/ 目录：该目录存放 Dubbo 内部使用的 SPI 配置文件。


##### 1.代码示例
- 顶层接口
```java 
package cn.train.dubbospi;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.Adaptive;
import org.apache.dubbo.common.extension.SPI;

/**
 * 
 * @date 2020/12/21 15:44
 * @descript 接口中必须要有@SPI注解
 */
@SPI("dubbo")
public interface Activate {

    void test1();

    @Adaptive
    String test2(URL url);
}
```

- 实现类 

```java 
package cn.train.dubbospi;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.Adaptive;

/**
 * 
 * @date 2020/12/21 15:44
 * @descript
 */
public class DubboActive implements Activate{
    @Override
    public void test1() {
        System.out.println("dubbo test1");
    }

    @Override
    public String test2(URL url) {
        System.out.println("dubbo test2");
        return "dubbo test2";
    }
}

package cn.train.dubbospi;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.Adaptive;

/**
 * 
 * @date 2020/12/21 15:44
 * @descript
 */
@Adaptive
public class MqActive implements Activate{


    @Override
    public void test1() {
        System.out.println("mq test1");
    }

    @Override
    public String test2(URL url) {
        System.out.println("mq test2");
        return "mq test2";
    }
}

public class DubboTest {

    public static void main(String[] args) {
        Activate activate = ExtensionLoader.getExtensionLoader(Activate.class).getAdaptiveExtension();
        System.out.println(activate);
        activate.test1();
    }
}

```

- 配置文件 

在resources/META-INF/dubbo下面配置接口文件，文件名称必须跟接口完整限定名相同，
```xml
META-INF/dubbo/cn.train.dubbospi.Activate
	dubbo=cn.train.dubbospi.DubboActive
	mq=cn.train.dubbospi.MqActive
```

接口文件中配置的内容就是该接口的实现类的完整限定名，跟jdk中配置不一样的地方就是，dubbo中可以带key，这个key就是该实现类的映射key，可以根据这个key获取到该key对应的类实例

总结：
(1):接口中必须要有@SPI注解   
(2):实现类中，如果没有@Adaptive注解，必须在接口中有@Adaptive注解，那么通过该api`ExtensionLoader.getExtensionLoader(Activate.class).getAdaptiveExtension();` 获取的是dubbo自己生成的代理类  
(3):如果有一个实现类中有@Adaptive注解，那么通过该api`ExtensionLoader.getExtensionLoader(Activate.class).getAdaptiveExtension();` 获取的是标有注解的实现类  
(4):如果有一个实现类中有@Adaptive注解，那么通过该api`ExtensionLoader.getExtensionLoader(Activate.class).getAdaptiveExtension();` 获取的是标有注解的最后一个实现类(按照字符顺序排序)  


##### 常用api分析
1. getAdaptiveExtension()  
该方法是获取到一个接口的实现类，获取的方式有：  
- 获取类上有@Adaptive注解的类的实例
- 如果接口的所有实现类都没有@Adaptive注解则dubbo动态生成一个

```java 
//需要获取什么接口的实现类，getExtensionLoader中就传该接口的类型
ActivateApi adaptiveExtension =
ExtensionLoader.getExtensionLoader(ActivateApi.class).getAdaptiveExtension();
```

2. 源码分析

- ExtensionLoader类中的getExtensionLoader 
```java 
    @SuppressWarnings("unchecked")
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
		//type 为空抛异常
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
		//type 不是接口抛异常
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
        }
		//如果接口上没有@SPI注解，则报错
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
        }

		//private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>(64);  EXTENSION_LOADERS对象就是一个缓存,保存接口类对应的 ExtensionLoader对象
		// 根据 接口的class对象即type 获取map中的value值
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
		    //每一个@SPI接口类型都会对应一个ExtensionLoader对象，如果map中没有对应的key，创建并添加一到map中
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

- getAdaptiveExtension
核心思想就是先从缓存中拿实例，如果没有才调用createAdaptiveExtension();方法创建实例。创建完成后放入到缓存中。缓存就是在ExtensionLoader中的一个Holder对象，如：Holder cachedAdaptiveInstance = new Holder<>();

```java 
    @SuppressWarnings("unchecked")
    public T getAdaptiveExtension() {
		//先从缓存中拿实例
        Object instance = cachedAdaptiveInstance.get();
		//DCL 思想 双重检锁
        if (instance == null) {
            if (createAdaptiveInstanceError != null) {
                throw new IllegalStateException("Failed to create adaptive instance: " +
                    createAdaptiveInstanceError.toString(),
                    createAdaptiveInstanceError);
            }

            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
						//创建接口实例的核心方法
                        instance = createAdaptiveExtension();
						//把创建的实例放入缓存中
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        }

        return (T) instance;
    }
	
	//该方法就是创建实例的方法，创建实例并对实例进行IOC属性的依赖注入
	private T createAdaptiveExtension() {
		try {
			//创建实例，以及ioc逻辑
			return injectExtension((T) getAdaptiveExtensionClass().newInstance());
		} catch (Exception e) {
			throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
		}
    }
	
	//获取到要实例化的类的Class对象并返回
	private Class<?> getAdaptiveExtensionClass() {
		//扫描三个文件，解析文件中的内容，建立名称(key)和类(value)的映射关系
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
		//如果没有找到Adapter注解的实现类
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
	//该方法是一个非常核心的方法，dubbo spi中的很多API都需要先调用这个方法来建立key和class的映射关系。获取映射关系的流程也是先从缓存里面拿，如果缓存没有才读配置文件建立映射关系，并把映射关系缓存起来。
	private Map<String, Class<?>> getExtensionClasses() {
		//先从缓存中拿 
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
					//如果缓存中没有，从本地文件中加载key和类的关系
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
	
	/**
     * synchronized in getExtensionClasses  该方法就是读取resources/META-INF/dubbo、META-INF/dubbo/internal/ META-INF/services/下面的配置文件，然后解析配置文件，建
	 * 立key和class的映射关系，给ExtensionLoader里面的全局变量赋值等功能。
     */
    private Map<String, Class<?>> loadExtensionClasses() {
		//设置cachedDefaultName 
        cacheDefaultExtensionName();

        Map<String, Class<?>> extensionClasses = new HashMap<>();

		//strategies 利用的是java spi机制进行解析 
        for (LoadingStrategy strategy : strategies) {
            loadDirectory(extensionClasses, strategy.directory(), type.getName(), strategy.preferExtensionClassLoader(), strategy.overridden(), strategy.excludedPackages());
            loadDirectory(extensionClasses, strategy.directory(), type.getName().replace("org.apache", "com.alibaba"), strategy.preferExtensionClassLoader(), strategy.overridden(), strategy.excludedPackages());
        }

        return extensionClasses;
    }
	
	 /**
     * extract and cache default extension name if exists 
	 * 设置ExtensionLoader中的全局变量cachedDefaultName的值，全局变量值的来源就是接口中@SPI("spring")注解的
	 * value值。
     */
    private void cacheDefaultExtensionName() {
		//获取类上的@SPI注解
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation == null) {
            return;
        }

        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                throw new IllegalStateException("More than 1 default extension name on extension " + type.getName()
                    + ": " + Arrays.toString(names));
            }
			//把value值设置到 cachedDefaultName，，这个就是默认的实现类
            if (names.length == 1) {
                cachedDefaultName = names[0];
            }
        }
    }
	
	private static volatile LoadingStrategy[] strategies = loadLoadingStrategies();
	
    /**
	 *利用java spi机制加载LoadingStrategy接口的实现类，查看META-INF目录下的services目录： 
	 *org.apache.dubbo.common.extension.DubboInternalLoadingStrategy
	 *org.apache.dubbo.common.extension.DubboLoadingStrategy
	 *org.apache.dubbo.common.extension.ServicesLoadingStrategy
	 * 为了扫描 META-INF/dubbo/internal/   META-INF/dubbo/ META-INF/services/ 这三个目录下的实现类，后续用于保存到 extensionClasses 缓存中
	/
	private static LoadingStrategy[] loadLoadingStrategies() {
	return stream(load(LoadingStrategy.class).spliterator(), false)
		.sorted()
		.toArray(LoadingStrategy[]::new);
    }
	
	private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type,
                               boolean extensionLoaderClassLoaderFirst, boolean overridden, String... excludedPackages) {
        String fileName = dir + type;
        try {
            Enumeration<java.net.URL> urls = null;
            ClassLoader classLoader = findClassLoader();

            // try to load from ExtensionLoader's ClassLoader first
            if (extensionLoaderClassLoaderFirst) {
                ClassLoader extensionLoaderClassLoader = ExtensionLoader.class.getClassLoader();
                if (ClassLoader.getSystemClassLoader() != extensionLoaderClassLoader) {
                    urls = extensionLoaderClassLoader.getResources(fileName);
                }
            }

            if (urls == null || !urls.hasMoreElements()) {
                if (classLoader != null) {
                    urls = classLoader.getResources(fileName);
                } else {
                    urls = ClassLoader.getSystemResources(fileName);
                }
            }

            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL resourceURL = urls.nextElement();
					//加载接口对应的文件的核心方法
                    loadResource(extensionClasses, classLoader, resourceURL, overridden, excludedPackages);
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                type + ", description file: " + fileName + ").", t);
        }
    }
	
	
	/**
	*该方法就是对每一个文件进行处理的，建立名称和类的映射关系 ，设置全局变量的值。对读到的文件中的每一行数
	*据进行处理
	*/
	private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader,
                              java.net.URL resourceURL, boolean overridden, String... excludedPackages) {
        try {
            try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), StandardCharsets.UTF_8))) {
                String line;
                String clazz = null;
				//读一行数据
                while ((line = reader.readLine()) != null) {
				    //查看是否有注释 ，注释都是#开头
                    final int ci = line.indexOf('#');
                    if (ci >= 0) {
                        line = line.substring(0, ci);
                    }
                    line = line.trim();
                    if (line.length() > 0) {
                        try {
                            String name = null;
                            int i = line.indexOf('=');
                            if (i > 0) {
							    //获取key value 使用=进行分隔
                                name = line.substring(0, i).trim();
                                clazz = line.substring(i + 1).trim();
                            } else {
                                clazz = line;
                            }
                            if (StringUtils.isNotEmpty(clazz) && !isExcluded(clazz, excludedPackages)) {
								//类加载的核心逻辑
                                loadClass(extensionClasses, resourceURL, Class.forName(clazz, true, classLoader), name, overridden);
                            }
                        } catch (Throwable t) {
                            IllegalStateException e = new IllegalStateException("Failed to load extension class (interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
                            exceptions.put(line, e);
                        }
                    }
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                type + ", class file: " + resourceURL + ") in " + resourceURL, t);
        }
    }
	
	private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name,
                           boolean overridden) throws NoSuchMethodException {
		////如果类类型和接口类型不一致，报错
        if (!type.isAssignableFrom(clazz)) {
            throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + " is not subtype of interface.");
        }
		//判断类上是否有Adaptive注解
        if (clazz.isAnnotationPresent(Adaptive.class)) {
			//如果类上有Adaptive注解，对 cachedAdaptiveClass变量赋值，把当前类赋值给ExtensionLoader类中的cachedAdaptiveClass变量
            cacheAdaptiveClass(clazz, overridden);
			
		//如果是包装类，包装类必然是持有目标接口的引用的，有目标接口对应的构造函数
        } else if (isWrapperClass(clazz)) {
			/对 cachedWrapperClasses 变量赋值 （Set<Class<?>> cachedWrapperClasses）
            cacheWrapperClass(clazz);
			
			//如果上面两个条件都不满住，获取类的无参构造函数
        } else {
            clazz.getConstructor();
            if (StringUtils.isEmpty(name)) {
				//如果类有Extension注解，则是注解的value，如果没注解则是类名称的小写做为name
                name = findAnnotationName(clazz);
                if (name.length() == 0) {
                    throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
                }
            }

            String[] names = NAME_SEPARATOR.split(name);
            if (ArrayUtils.isNotEmpty(names)) {
				//如果类上面有@Activate注解，则建立名称和注解的映射
				//对 cachedActivates 全局变量赋值
                cacheActivateClass(clazz, names[0]);
                for (String n : names) {
                    cacheName(clazz, n);
					//这里 extensionClasses 建立了 key和class的映射
                    saveInExtensionClass(extensionClasses, clazz, n, overridden);
                }
            }
        }
    }
```
getAdaptiveExtension方法总结：
前面分析类建立映射关系的过程，如果cachedAdaptiveClass属性不为空，也就是有@Adaptive注解的类，则直接返回该类，如果该属性为空，则会走到createAdaptiveExtensionClass方法，该方法的核心作用：  
1. 自动根据接口中的注解和参数配置生成类的字符串
2. 根据类的字符串动态编译生成字节码文件加载到jvm  

接口定义的规则  
1. 接口中的方法必须要有一个方法有@Adaptive注解，要不然会报错  
2. 如果想让dubbo中给我们生成一个代理类，方法的参数中必须要有URL参数，要不然会报错  

**为什么要这样设计**
因为这种方式dubbo会自动生成一个类，这个类其实就是一个代理类，但是dubbo并不知道应该都哪个逻辑，也就是这个代理类并不知道走该接口的哪个实现类，所以我们必须用代理对象掉方法的时候用方法的入参告诉dubbo应该走哪个实现类，而选择掉哪个实现类的参数就在URL参数中。所以参数中必须要有URL参数。


### @Spi + @Adaptive注解源码解析

1. ExtensionLoader.getExtensionLoader(ActivateApi.class).getAdaptiveExtension();
getExtensionLoader方法分析

```java
   @SuppressWarnings("unchecked")
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
        }
		先去map中去取，如果有值直接返回
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
		如果Map中没有值，则会新建一个ExtensionLoader对象，并放入map中 然后返回
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

### 核心方法总结

##### getExtensionLoader(Class<T> type) api分析
1. 首先分析ExtensionLoader.getExtensionLoader(Protocal.class)方法，参数是一个带有@SPI注解的接口类
2. getExtensionLoader(Class<T> type)方法中首先会判断，type为空或者，type不是接口以及没有@SPI注解都会抛异常
3. 通过ExtensionLoader中的全局变量EXTENSION_LOADERS，它是一个ConcurrentHashMap，key是接口类，value是ExtensionLoader对象；会根据key获取ExtensionLoader对象，如果没有就新建一个塞入缓存。最后返回接口类对应的 ExtensionLoader。（一个接口类型对应这一个ExtensionLoader实例对象）

##### getAdaptiveExtension() api分析
1. 接着分析getAdaptiveExtension()方法，最终会返回一个接口对应的实例类。
首先会从ExtensionLoader类中的cachedAdaptiveInstance变量中去获取一个实例，其中cachedAdaptiveInstance 变量是一个Holder<Object>对象，也就是每一个ExtensionLoader对象中有一个Holder对象 （Interface.class->ExtensionLoader.class->Holder.class）,
如果cachedAdaptiveInstance变量中获取不到接口的实例，那么就会通过DCL双重检锁进行实例的创建，并把创建的实例设置到cachedAdaptiveInstance 变量中
2. 接下来就是创建实例的分析，通过该方法实现createAdaptiveExtension()，该方法创建实例的同时会进行IOC属性的依赖注入
- 实例的创建 getAdaptiveExtensionClass().newInstance()
    	(1)建立名称和类的映射关系；  
			1.首先从ExtensionLoader的内部变量cachedClasses中获取  
			2.如果缓存中没有，从本地文件中加载key和类的映射关系  
				1.获取类型的spi注解，把spi上的value值赋值给cachedDefaultName（即获取到默认的实现类）  
				2.扫描META-INF/dubbo/这个目录，加载接口对应的实现类，如果类上有Adaptive注解，把该类赋值给cachedAdaptiveClass变量，如果是包装类加到cachedWrapperClasses成员变量中  
		(2)如果缓存中有，即cachedAdaptiveClass不为空，就只返回  
		(3)缓存中没有，调用该方法createAdaptiveExtensionClass()创建；  
- IOC属性的注入 injectExtension((T) getAdaptiveExtensionClass().newInstance())
	
		
