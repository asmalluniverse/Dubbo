### 1.dubbo简介

Apache Dubbo是一款微服务开发框架，它提供了 RPC通信 与 微服务治理 两大关键能力。
这意味着，使用 Dubbo 开发的微服务，将具备相互之间的远程发现与通信能力，同时利用
Dubbo 提供的丰富服务治理能力，可以实现诸如服务发现、负载均衡、流量调度等服务治理诉求。
同时 Dubbo 是高度可扩展的，用户几乎可以在任意功能点去定制自己的实现，以改变框架的默认行为来满足自己的业务需求。

如开篇所述，Dubbo 提供了构建云原生微服务业务的一站式解决方案，可以使用 Dubbo 快速定义并发布微服务组件，
同时基于 Dubbo 开箱即用的丰富特性及超强的扩展能力，构建运维整个微服务体系所需的各项服务治理能力，如 Tracing、Transaction 等，Dubbo 提供的基础能力包括：
服务发现
流式通信
负载均衡
流量治理

- 开箱即用
	易用性高，如 Java 版本的面向接口代理特性能实现本地透明调用
	功能丰富，基于原生库或轻量扩展即可实现绝大多数的微服务治理能力
- 超大规模微服务集群实践
	高性能的跨进程通信协议
	地址发现、流量治理层面，轻松支持百万规模集群实例
- 企业级微服务治理能力
	服务测试
	服务Mock

1.版本选择 dubbo3 vs dubbo2
	Dubbo3 基于 Dubbo2 演进而来，在保持原有核心功能特性的同时， Dubbo3 在易用性、超大规模微服务实践、云原生基础设施适配、安全设计等几大方向上进行了全面升级。
	参考官网：https://dubbo.apache.org/zh/docs/performance/benchmarking/
	(1)资源利用率 gc次数
	(2)dubbo协议：Triple协议 vs Dubbo协议
	
	
	
2.dubbo vs spring cloud
3.为什么使用dubbo
	架构的演变  单体架构 -> 垂直扩展 + nginx/lvs/f5（一个服务部署到多台服务器，但是此时，每个服务器都是同一个服务，可能包含很多模块，并没有对服务进行拆分和细化）  
							一台现在变成3台，确实增加了吞吐量，但是某天可能又不够了，又需要增加机器，此时需要修改一堆配置，并且，每个模块的访问不可能都是很高的qps
						
					    -> 分布式应用架构 ： 一个应用拆分成多个不同的应用（应用的横向扩展）  优点: 职责划分更加单一  缺点：单次请求变慢，需要远程通信，不是一个服务直接的service之间相互调用 ；应用直接的访问变得复杂
						-> 流动计算框架 : soa服务治理：运行时干预
						
### 2.dubbo的使用配置

**版本依赖：**
```java
<dependency>
<groupId>org.apache.dubbo</groupId>
<artifactId>dubbo</artifactId>
<version>3.0.2.1</version>
</dependency>
```

**集成Zookeeper作为服务注册中心**
如果不集成注册中心就需要指定url

```java
@DubboReference(check = false,protocol = "dubbo",retries = 3,timeout = 1000000000,cluster =
"failover",loadbalance = "random",sticky = true,methods = {@Method(name = "doKill",isReturn
= false)},url = "dubbo://localhost:20880")
```

```java
<dependency>
<groupId>org.apache.dubbo</groupId>
<artifactId>dubbo-dependencies-zookeeper</artifactId>
<version>${dubbo.version}</version>
<type>pom</type>
</dependency>
```

### 3.部署架构（注册中心 配置中心 元数据中心）

作为一个微服务框架，Dubbo sdk 跟随着微服务组件被部署在分布式集群各个位置，为了在分布式环境下实现各个微服务组件间的协作， Dubbo 定义了一些中心化组件，包括：
1. 注册中心
- 协调 Consumer 与 Provider 之间的地址注册与发现
2. 配置中心
- 存储 Dubbo 启动阶段的全局配置，保证配置的跨环境共享与全局一致性
- 负责服务治理规则（路由规则、动态配置等）的存储与推送。
- 简单来说，就是把 dubbo.properties 中的属性进行集中式存储，存储在其他的服务器上
- 目前 Dubbo 能支持的配置中心有：apollo、nacos、zookeeper、etcd等
3. 元数据中心
- 接收 Provider 上报的服务接口元数据，为 Admin 等控制台提供运维能力（如服务测试、接口文档等）
- 作为服务发现机制的补充，提供额外的接口/方法级别配置信息的同步能力，相当于注册中心的额外扩展

### 4.dubbo的高级用法

##### Provider端
```java

/**
* provider端配置，基于zookeeper进行服务注册于发现 dubbo-provider.properties
*/

dubbo.application.name=dubbo_provider
dubbo.registry.address=zookeeper://${zookeeper.address:127.0.0.1}:2181
dubbo.protocol.name=dubbo
dubbo.protocol.port=20880

/**
* 用于启动 dubbo provider端服务
*/
public class AnnotationProvider {
    public static void main(String[] args) throws InterruptedException {
        AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(ProviderConfiguration.class);
        System.out.println("dubbo service stated");
        new CountDownLatch(1).await();
    }
}

/**
* 通过EnableDubbo注解 扫描配置了DubboService的类  PropertySource指定配置文件
*/
@EnableDubbo(scanBasePackages = "cn.zhangyu")
@Configuration
@PropertySource("classpath:/dubbo-provider.properties")
public class ProviderConfiguration {
}


public interface UserService {
    String queryUser(String var1);

    void doKill(String var1);
}

/**
*provider 端对于userSevice的具体实现
*/
@DubboService(register = false)
public class UserServiceImpl implements UserService {
    @Override
    public String queryUser(String s) {
        System.out.println("==========provider===========" + s);
        return "OK--" + s;
    }

    @Override
    public void doKill(String s) {
        System.out.println("==========provider===========" + s);
    }
}
```

##### Consumer端
```java 
/**
* consumer 端配置 dubbo-consumer.properties
*/
dubbo.application.name=dubbo_consumer
dubbo.registry.address=zookeeper://${zookeeper.address:127.0.0.1}:2181
dubbo.protocol.port=29015


/**
* 通过EnableDubbo注解扫描服务端DubboReference 注解 
*/
@Configuration
@PropertySource("classpath:/dubbo-consumer.properties")
@EnableDubbo(scanBasePackages = "cn.zhangyu")
public class ConsumerConfiguration {
}

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = ConsumerConfiguration.class)
public class AnnotationTest {

    @DubboReference(check = false)
    private UserService userService;
	
	   @Test
    public void test1(){
        System.out.println(userService.queryUser("hello world"));
    }
}
```

1. 启动时检查
`@DubboReference(check = false)` :关闭某个服务的启动时检查 (false代表不检查，即使没有检测到服务也不会报错)

2. 集群容错
集群调用失败时，Dubbo 提供的容错方案
在集群调用失败时，Dubbo 提供了多种容错方案，默认为 failover 重试。

- Failover Cluster
失败自动切换，当出现失败，重试其它服务器。通常用于读操作，但重试会带来更长延迟。可通过 retries="2" 来
设置重试次数(不含第一次)。该配置为缺省配置
- Failfast Cluster
快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。
- Failsafe Cluster
失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。
- Failback Cluster
失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。
- Forking Cluster
并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过
forks="2" 来设置最大并行数

- Broadcast Cluster
广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有提供者更新缓存或日志等本地资源信息。
- Available Cluster
调用目前可用的实例（只调用一个），如果当前没有可用的实例，则抛出异常。通常用于不需要负载均衡的场景。

配置：
`@reference(cluster = "broadcast", parameters = {"broadcast.fail.percent", "20"})`

3. 负载均衡
Dubbo 提供的集群负载均衡策略
在集群负载均衡时，Dubbo 提供了多种均衡策略，默认为 random 随机调用。
具体实现上，Dubbo 提供的是客户端负载均衡，即由 Consumer 通过负载均衡算法得出需要将请求提交到哪个Provider 实例

目前 Dubbo 内置了如下负载均衡算法，用户可直接配置使用：
算法                          特性                            备注
RandomLoadBalance            加权随机                  默认算法，默认权重相同
RoundRobinLoadBalance        加权轮询                  借鉴于 Nginx 的平滑加权轮询算法，默认权重相同
LeastActiveLoadBalance       最少活跃优先 + 加权        随机 背后是能者多劳的思想
ShortestResponseLoadBalance  最短响应优先 + 加权        随机 更加关注响应速度
ConsistentHashLoadBalance    一致性 Hash               确定的入参，确定的提供者，适用于有状态请求

配置：
<dubbo:service interface="..." loadbalance="roundrobin" />

4. 服务分组
使用服务分组区分服务接口的不同实现
当一个接口有多种实现时，可以用 group 区分
- provider
```java 
@DubboService(group = "groupImpl1")
public class GroupImpl1 implements Group {
    @Override
    public String doSomething(String s) {
        return "GroupImpl1";
    }
}

@DubboService(group = "groupImpl2")
public class GroupImpl2 implements Group {
    @Override
    public String doSomething(String s) {
        return "GroupImpl2";
    }
}
```

- consumer
```java
/**group = "*" ：代表 合并所有实现，但是此时需要提供一个实现类，实现Merger接口，实现类中编写业务逻辑，
*并且需要在reosurce目录下创建 META-INF 目录，在该目录下创建dobbo目录，在dubbo目录下创建 resources\META-INF\dubbo\org.apache.dubbo.rpc.cluster.Merger 文件 
* 该文件中以key=value 的形式进行配置 ： string=cn.zhangyu.merger.StringMerger 
*如果group属性中指定某一个实现类，那么就会调用指定的实现类 group = groupImpl1
*/
@DubboReference(check = false, group = "*", parameters = {"merger","true"})
    private Group group;
	
    @Test
    public void groupTest(){
        String doSomething = group.doSomething("group test");
        System.out.println(doSomething);
    }
	
public class StringMerger implements Merger<String> {
    @Override
    public String merge(String... items) {
        if(items.length == 0) {
            return null;
        }
        StringBuilder builder = new StringBuilder();
        for (String item : items) {
            if(item != null) {
                builder.append(item).append("-");
            }
        }
        return builder.toString();
    }
}
```

5. 多版本
在 Dubbo 中为同一个服务配置多个版本
当一个接口实现，出现不兼容升级时，可以用版本号过渡，版本号不同的服务相互间不引用。
可以按照以下的步骤进行版本迁移：
1. 在低压力时间段，先升级一半提供者为新版本
2. 再将所有消费者升级为新版本
3. 然后将剩下的一半提供者升级为新版本
- provider
```java 
@DubboService(version = "versionServiceImpl1")
public class VersionServiceImpl1 implements VersionService {

    @Override
    public String version(String s) {
        return "VersionServiceImpl1";
    }
}

@DubboService(version = "versionServiceImpl2")
public class VersionServiceImpl2 implements VersionService {
    @Override
    public String version(String s) {
        return "VersionServiceImpl2";
    }
}
```
- consumer
```java 
    @DubboReference(check = false, version = "versionServiceImpl2")
    private VersionService versionService;
	
	    @Test
    public void  versionTest(){
        System.out.println(versionService.version("version test"));
    }
```

6. 参数验证

```java
public interface ValidationService {
    void save(ValidationParamter var1);

    void update(ValidationParamter var1);

    void delete(@Min(1L) long var1, @NotNull @Size(min = 2,max = 16) String var3);
}

@DubboService
public class ValidationServiceImpl implements ValidationService {
    @Override
    public void save(ValidationParamter validationParamter) {
        System.out.println("ValidationServiceImpl.save");
    }

    @Override
    public void update(ValidationParamter validationParamter) {
        System.out.println("ValidationServiceImpl.update");
    }

    @Override
    public void delete(long l, String s) {
        System.out.println("ValidationServiceImpl.delete");
    }
}

   
```


```java 
@DubboReference(check = false,validation = "true")
private ValidationService validationService;

	@Test
public void  validationTest(){
	ValidationParamter params = new ValidationParamter();
	params.setName("test");
	params.setAge(101);
	params.setLoginDate(new Date(System.currentTimeMillis() - 10000000));
	params.setExpiryDate(new Date(System.currentTimeMillis() + 10000000));
	validationService.save(params);
}
```

7. 参数回调

场景：客户端调用服务端支付接口，然后回调客户端进行订单状态的修改。

通过参数回调从服务器端调用客户端逻辑
参数回调方式与调用本地 callback 或 listener 相同，只需要在 Spring 的配置文件中声明哪个参数是 callback 类型即
可。Dubbo 将基于长连接生成反向代理，这样就可以从服务器端调用客户端逻辑。
```java 
public interface CallbackListener {
    void changed(String msg);
}

@DubboService(methods = {@Method(name = "addListener", arguments = {@Argument(index = 1, callback = true)})})
public class CallBackServiceImpl implements CallbackService {
    @Override
    public void addListener(String s, CallbackListener callbackListener) {
        //1 支付操作
        System.out.println("支付操作成功");

        //回调客户端
        callbackListener.changed(getChanged("修改订单状态"));
    }

    private String getChanged(String key) {
        return "Changed  " + key + ":" + new SimpleDateFormat("yyyy-MM-dd:mm:ss").format(new Date());
    }
}
```

```java
    @DubboReference(check = false)
    private CallbackService callbackService;
	
	    @Test
    public void callBackTest(){
        callbackService.addListener("修改订单编号", s -> {
            //这个方法就是来处理客户端的回调逻辑 修改订单状态
            System.out.println("回调客户端逻辑，修改订单状态==" + s);
        });
    }
```

8. 本地存根
在 Dubbo 中利用本地存根在客户端执行部分逻辑
远程服务后，客户端通常只剩下接口，而实现全在服务器端，但提供方有些时候想在客户端也执行部分逻辑，比如：
做 ThreadLocal 缓存，提前验证参数，调用失败后伪造容错数据等等，此时就需要在 API 中带上 Stub，客户端生成
Proxy 实例，会把 Proxy 通过构造函数传给 Stub 1，然后把 Stub 暴露给用户，Stub 可以决定要不要去调 Proxy。

```java 
@DubboService
public class StubServiceImpl implements StubService {
    @Override
    public String stub(String s) {
        System.out.println("==========本地存根业务逻辑=========");
        return "本地存根业";
    }
}


```

```java 
import cn.enjoy.stub.StubService;

/**
 * @author zhangyu49
 * @date 2020/12/21 15:44
 * @email zhangyu49@hikvision.com.cn
 * @descript
 * 代理层
 * 1、必须实现接口
 * 2、要定义构造函数（传递客户端引用的实例的）
 */
public class LocalStubProxy implements StubService {

    private StubService stubService;

    public LocalStubProxy(StubService stubService) {
        this.stubService = stubService;
    }

    /**
     * 在这里就会对远程调用进行包装,本质就是静态代理
     * @param s
     * @return
     */
    @Override
    public String stub(String s) {
        try {
            //1、一系列校验
            System.out.println("校验");
            //如果校验通过调用后端接口
            return stubService.stub(s);
        } catch (Exception e) {
            e.printStackTrace();
            System.out.println("异常包装，服务降级");
            return "ok";
        }
    }
}

    @DubboReference(check = false, stub = "cn.zhangyu.stub.LocalStubProxy")
    private StubService stubService;
	
	
	@Test
    public void stubTest(){
        System.out.println(stubService.stub("stub test"));
    }

```

9. 本地伪装 
如何在 Dubbo 中利用本地伪装实现服务降级
本地伪装通常用于服务降级，比如某验权服务，当服务提供方全部挂掉后，客户端不抛出异常，而是通过 Mock 数据
返回授权失败

```java 
@DubboService
public class MockServiceImpl implements MockService {
    @Override
    public String mock(String s) {
        System.out.println("=======mockservice的业务处理=======");
        try {
            Thread.sleep(1000000);
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("=======mockservice的业务处理完成=======");
        return "MockServiceImpl.mock";
    }

    @Override
    public String queryArea(String s) {
        return s;
    }

    @Override
    public String queryUser(String s) {
        return s;
    }
}
```

```java
package cn.enjoy.mock;

/**
 * 1、接口名称 + Mock
 *
 * MockService + Mock
 *
 * 2、mock逻辑必须定义在接口的包下面
 *
 */
public class MockServiceMock implements MockService {
    @Override
    public String mock(String s) {
        System.out.println(this.getClass().getName() + "--mock");
        return s;
    }

    @Override
    public String queryArea(String s) {
        System.out.println(this.getClass().getName() + "--queryArea");
        return s;
    }

    @Override
    public String queryUser(String s) {
        System.out.println(this.getClass().getName() + "--queryUser");
        return s;
    }
}

/**
 *
 * 1、接口名称 + Mock
 *
 * MockService + Mock
 *
 * 2、mock逻辑必须定义在接口的包下面
 *
 */
public class LocalMockService implements MockService {
    @Override
    public String mock(String s) {
        System.out.println(this.getClass().getName() + "--mock");
        return s;
    }

    @Override
    public String queryArea(String s) {
        System.out.println(this.getClass().getName() + "--queryArea");
        return s;
    }

    @Override
    public String queryUser(String s) {
        System.out.println(this.getClass().getName() + "--queryUser");
        return s;
    }
}

// 如果 mock属性值为true 那个就要保存接口和实现类要在相同的类路径下 并且类名称是接口名称 + Mock，否则需要配置类的全路径名称。 同时也可以mock = "force:return gpwy 直接进行服务降级配置，这样就不会调用服务端接口直接返回配置的值
//    @DubboReference(check = false,mock = "true")
//    @DubboReference(check = false,mock = "cn.enjoy.mock.LocalMockService")
    @DubboReference(check = false,mock = "force:return gpwy")
    private MockService mockService;
	
	    @Test
    public void mockTest(){
        System.out.println(mockService.mock("mock"));
    }
```

10. 异步调用
在 Dubbo 中发起异步调用
从 2.7.0 开始，Dubbo 的所有异步编程接口开始以 CompletableFuture 为基础
基于 NIO 的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销较小。


```java 
@DubboService
public class AsyncServiceImpl implements AsyncService {
    //异步调用
    @Override
    public String asynctoDo(String s) {
        for (int i = 0; i < 10; i++) {
            System.out.println("=========AsyncServiceImpl.asynctoDo");
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        return "OK," + s;
    }

    @Override
    public CompletableFuture<String> doOne(String s) {
        return CompletableFuture.supplyAsync(()->{
            System.out.println("=======doOne 的业务执行=========");
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "doOne--OK";
        });
    }

    @Override
    public String doTwo(String s) {
        return null;
    }
}

```

```java 
    @DubboReference(check = false,timeout = 1000000000,methods = {@Method(name = "asynctoDo",async = true),@Method(name = "doOne",async = false)})
    private AsyncService asyncService;
	
	@Test
    public void asyncTest() throws InterruptedException {
        String aa = asyncService.asynctoDo("aa");
        System.out.println("main==" + aa);
        System.out.println("并行调用其他接口====");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //需要拿到异步调用的返回结果
        CompletableFuture<Object> resultFuture = RpcContext.getContext().getCompletableFuture();
        resultFuture.whenComplete((retValue,exception)->{
            if(exception == null) {
                System.out.println("正常返回==" + retValue);
            } else {
                exception.printStackTrace();
            }
        });
        Thread.currentThread().join();
    }

```

