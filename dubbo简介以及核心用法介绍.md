### 1.dubbo���

Apache Dubbo��һ��΢���񿪷���ܣ����ṩ�� RPCͨ�� �� ΢�������� ����ؼ�������
����ζ�ţ�ʹ�� Dubbo ������΢���񣬽��߱��໥֮���Զ�̷�����ͨ��������ͬʱ����
Dubbo �ṩ�ķḻ������������������ʵ����������֡����ؾ��⡢�������ȵȷ�����������
ͬʱ Dubbo �Ǹ߶ȿ���չ�ģ��û��������������⹦�ܵ�ȥ�����Լ���ʵ�֣��Ըı��ܵ�Ĭ����Ϊ�������Լ���ҵ������

�翪ƪ������Dubbo �ṩ�˹�����ԭ��΢����ҵ���һվʽ�������������ʹ�� Dubbo ���ٶ��岢����΢���������
ͬʱ���� Dubbo ���伴�õķḻ���Լ���ǿ����չ������������ά����΢������ϵ����ĸ������������������ Tracing��Transaction �ȣ�Dubbo �ṩ�Ļ�������������
������
��ʽͨ��
���ؾ���
��������

- ���伴��
	�����Ըߣ��� Java �汾������ӿڴ���������ʵ�ֱ���͸������
	���ܷḻ������ԭ�����������չ����ʵ�־��������΢������������
- �����ģ΢����Ⱥʵ��
	�����ܵĿ����ͨ��Э��
	��ַ���֡�����������棬����֧�ְ����ģ��Ⱥʵ��
- ��ҵ��΢������������
	�������
	����Mock

1.�汾ѡ�� dubbo3 vs dubbo2
	Dubbo3 ���� Dubbo2 �ݽ��������ڱ���ԭ�к��Ĺ������Ե�ͬʱ�� Dubbo3 �������ԡ������ģ΢����ʵ������ԭ��������ʩ���䡢��ȫ��Ƶȼ������Ͻ�����ȫ��������
	�ο�������https://dubbo.apache.org/zh/docs/performance/benchmarking/
	(1)��Դ������ gc����
	(2)dubboЭ�飺TripleЭ�� vs DubboЭ��
	
	
	
2.dubbo vs spring cloud
3.Ϊʲôʹ��dubbo
	�ܹ����ݱ�  ����ܹ� -> ��ֱ��չ + nginx/lvs/f5��һ�������𵽶�̨�����������Ǵ�ʱ��ÿ������������ͬһ�����񣬿��ܰ����ܶ�ģ�飬��û�жԷ�����в�ֺ�ϸ����  
							һ̨���ڱ��3̨��ȷʵ������������������ĳ������ֲ����ˣ�����Ҫ���ӻ�������ʱ��Ҫ�޸�һ�����ã����ң�ÿ��ģ��ķ��ʲ����ܶ��Ǻܸߵ�qps
						
					    -> �ֲ�ʽӦ�üܹ� �� һ��Ӧ�ò�ֳɶ����ͬ��Ӧ�ã�Ӧ�õĺ�����չ��  �ŵ�: ְ�𻮷ָ��ӵ�һ  ȱ�㣺���������������ҪԶ��ͨ�ţ�����һ������ֱ�ӵ�service֮���໥���� ��Ӧ��ֱ�ӵķ��ʱ�ø���
						-> ���������� : soa������������ʱ��Ԥ
						
### 2.dubbo��ʹ������

**�汾������**
```java
<dependency>
<groupId>org.apache.dubbo</groupId>
<artifactId>dubbo</artifactId>
<version>3.0.2.1</version>
</dependency>
```

**����Zookeeper��Ϊ����ע������**
���������ע�����ľ���Ҫָ��url

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

### 3.����ܹ���ע������ �������� Ԫ�������ģ�

��Ϊһ��΢�����ܣ�Dubbo sdk ������΢��������������ڷֲ�ʽ��Ⱥ����λ�ã�Ϊ���ڷֲ�ʽ������ʵ�ָ���΢����������Э���� Dubbo ������һЩ���Ļ������������
1. ע������
- Э�� Consumer �� Provider ֮��ĵ�ַע���뷢��
2. ��������
- �洢 Dubbo �����׶ε�ȫ�����ã���֤���õĿ绷��������ȫ��һ����
- ��������������·�ɹ��򡢶�̬���õȣ��Ĵ洢�����͡�
- ����˵�����ǰ� dubbo.properties �е����Խ��м���ʽ�洢���洢�������ķ�������
- Ŀǰ Dubbo ��֧�ֵ����������У�apollo��nacos��zookeeper��etcd��
3. Ԫ��������
- ���� Provider �ϱ��ķ���ӿ�Ԫ���ݣ�Ϊ Admin �ȿ���̨�ṩ��ά�������������ԡ��ӿ��ĵ��ȣ�
- ��Ϊ�����ֻ��ƵĲ��䣬�ṩ����Ľӿ�/��������������Ϣ��ͬ���������൱��ע�����ĵĶ�����չ

### 4.dubbo�ĸ߼��÷�

##### Provider��
```java

/**
* provider�����ã�����zookeeper���з���ע���ڷ��� dubbo-provider.properties
*/

dubbo.application.name=dubbo_provider
dubbo.registry.address=zookeeper://${zookeeper.address:127.0.0.1}:2181
dubbo.protocol.name=dubbo
dubbo.protocol.port=20880

/**
* �������� dubbo provider�˷���
*/
public class AnnotationProvider {
    public static void main(String[] args) throws InterruptedException {
        AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(ProviderConfiguration.class);
        System.out.println("dubbo service stated");
        new CountDownLatch(1).await();
    }
}

/**
* ͨ��EnableDubboע�� ɨ��������DubboService����  PropertySourceָ�������ļ�
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
*provider �˶���userSevice�ľ���ʵ��
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

##### Consumer��
```java 
/**
* consumer ������ dubbo-consumer.properties
*/
dubbo.application.name=dubbo_consumer
dubbo.registry.address=zookeeper://${zookeeper.address:127.0.0.1}:2181
dubbo.protocol.port=29015


/**
* ͨ��EnableDubboע��ɨ������DubboReference ע�� 
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

1. ����ʱ���
`@DubboReference(check = false)` :�ر�ĳ�����������ʱ��� (false������飬��ʹû�м�⵽����Ҳ���ᱨ��)

2. ��Ⱥ�ݴ�
��Ⱥ����ʧ��ʱ��Dubbo �ṩ���ݴ���
�ڼ�Ⱥ����ʧ��ʱ��Dubbo �ṩ�˶����ݴ�����Ĭ��Ϊ failover ���ԡ�

- Failover Cluster
ʧ���Զ��л���������ʧ�ܣ�����������������ͨ�����ڶ������������Ի���������ӳ١���ͨ�� retries="2" ��
�������Դ���(������һ��)��������Ϊȱʡ����
- Failfast Cluster
����ʧ�ܣ�ֻ����һ�ε��ã�ʧ����������ͨ�����ڷ��ݵ��Ե�д����������������¼��
- Failsafe Cluster
ʧ�ܰ�ȫ�������쳣ʱ��ֱ�Ӻ��ԡ�ͨ������д�������־�Ȳ�����
- Failback Cluster
ʧ���Զ��ָ�����̨��¼ʧ�����󣬶�ʱ�ط���ͨ��������Ϣ֪ͨ������
- Forking Cluster
���е��ö����������ֻҪһ���ɹ������ء�ͨ������ʵʱ��Ҫ��ϸߵĶ�����������Ҫ�˷Ѹ��������Դ����ͨ��
forks="2" �������������

- Broadcast Cluster
�㲥���������ṩ�ߣ�������ã�����һ̨�����򱨴�ͨ������֪ͨ�����ṩ�߸��»������־�ȱ�����Դ��Ϣ��
- Available Cluster
����Ŀǰ���õ�ʵ����ֻ����һ�����������ǰû�п��õ�ʵ�������׳��쳣��ͨ�����ڲ���Ҫ���ؾ���ĳ�����

���ã�
`@reference(cluster = "broadcast", parameters = {"broadcast.fail.percent", "20"})`

3. ���ؾ���
Dubbo �ṩ�ļ�Ⱥ���ؾ������
�ڼ�Ⱥ���ؾ���ʱ��Dubbo �ṩ�˶��־�����ԣ�Ĭ��Ϊ random ������á�
����ʵ���ϣ�Dubbo �ṩ���ǿͻ��˸��ؾ��⣬���� Consumer ͨ�����ؾ����㷨�ó���Ҫ�������ύ���ĸ�Provider ʵ��

Ŀǰ Dubbo ���������¸��ؾ����㷨���û���ֱ������ʹ�ã�
�㷨                          ����                            ��ע
RandomLoadBalance            ��Ȩ���                  Ĭ���㷨��Ĭ��Ȩ����ͬ
RoundRobinLoadBalance        ��Ȩ��ѯ                  ����� Nginx ��ƽ����Ȩ��ѯ�㷨��Ĭ��Ȩ����ͬ
LeastActiveLoadBalance       ���ٻ�Ծ���� + ��Ȩ        ��� ���������߶��͵�˼��
ShortestResponseLoadBalance  �����Ӧ���� + ��Ȩ        ��� ���ӹ�ע��Ӧ�ٶ�
ConsistentHashLoadBalance    һ���� Hash               ȷ������Σ�ȷ�����ṩ�ߣ���������״̬����

���ã�
<dubbo:service interface="..." loadbalance="roundrobin" />

4. �������
ʹ�÷���������ַ���ӿڵĲ�ͬʵ��
��һ���ӿ��ж���ʵ��ʱ�������� group ����
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
/**group = "*" ������ �ϲ�����ʵ�֣����Ǵ�ʱ��Ҫ�ṩһ��ʵ���࣬ʵ��Merger�ӿڣ�ʵ�����б�дҵ���߼���
*������Ҫ��reosurceĿ¼�´��� META-INF Ŀ¼���ڸ�Ŀ¼�´���dobboĿ¼����dubboĿ¼�´��� resources\META-INF\dubbo\org.apache.dubbo.rpc.cluster.Merger �ļ� 
* ���ļ�����key=value ����ʽ�������� �� string=cn.zhangyu.merger.StringMerger 
*���group������ָ��ĳһ��ʵ���࣬��ô�ͻ����ָ����ʵ���� group = groupImpl1
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

5. ��汾
�� Dubbo ��Ϊͬһ���������ö���汾
��һ���ӿ�ʵ�֣����ֲ���������ʱ�������ð汾�Ź��ɣ��汾�Ų�ͬ�ķ����໥�䲻���á�
���԰������µĲ�����а汾Ǩ�ƣ�
1. �ڵ�ѹ��ʱ��Σ�������һ���ṩ��Ϊ�°汾
2. �ٽ���������������Ϊ�°汾
3. Ȼ��ʣ�µ�һ���ṩ������Ϊ�°汾
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

6. ������֤

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

7. �����ص�

�������ͻ��˵��÷����֧���ӿڣ�Ȼ��ص��ͻ��˽��ж���״̬���޸ġ�

ͨ�������ص��ӷ������˵��ÿͻ����߼�
�����ص���ʽ����ñ��� callback �� listener ��ͬ��ֻ��Ҫ�� Spring �������ļ��������ĸ������� callback ���ͼ�
�ɡ�Dubbo �����ڳ��������ɷ�����������Ϳ��Դӷ������˵��ÿͻ����߼���
```java 
public interface CallbackListener {
    void changed(String msg);
}

@DubboService(methods = {@Method(name = "addListener", arguments = {@Argument(index = 1, callback = true)})})
public class CallBackServiceImpl implements CallbackService {
    @Override
    public void addListener(String s, CallbackListener callbackListener) {
        //1 ֧������
        System.out.println("֧�������ɹ�");

        //�ص��ͻ���
        callbackListener.changed(getChanged("�޸Ķ���״̬"));
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
        callbackService.addListener("�޸Ķ������", s -> {
            //�����������������ͻ��˵Ļص��߼� �޸Ķ���״̬
            System.out.println("�ص��ͻ����߼����޸Ķ���״̬==" + s);
        });
    }
```

8. ���ش��
�� Dubbo �����ñ��ش���ڿͻ���ִ�в����߼�
Զ�̷���󣬿ͻ���ͨ��ֻʣ�½ӿڣ���ʵ��ȫ�ڷ������ˣ����ṩ����Щʱ�����ڿͻ���Ҳִ�в����߼������磺
�� ThreadLocal ���棬��ǰ��֤����������ʧ�ܺ�α���ݴ����ݵȵȣ���ʱ����Ҫ�� API �д��� Stub���ͻ�������
Proxy ʵ������� Proxy ͨ�����캯������ Stub 1��Ȼ��� Stub ��¶���û���Stub ���Ծ���Ҫ��Ҫȥ�� Proxy��

```java 
@DubboService
public class StubServiceImpl implements StubService {
    @Override
    public String stub(String s) {
        System.out.println("==========���ش��ҵ���߼�=========");
        return "���ش��ҵ";
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
 * �����
 * 1������ʵ�ֽӿ�
 * 2��Ҫ���幹�캯�������ݿͻ������õ�ʵ���ģ�
 */
public class LocalStubProxy implements StubService {

    private StubService stubService;

    public LocalStubProxy(StubService stubService) {
        this.stubService = stubService;
    }

    /**
     * ������ͻ��Զ�̵��ý��а�װ,���ʾ��Ǿ�̬����
     * @param s
     * @return
     */
    @Override
    public String stub(String s) {
        try {
            //1��һϵ��У��
            System.out.println("У��");
            //���У��ͨ�����ú�˽ӿ�
            return stubService.stub(s);
        } catch (Exception e) {
            e.printStackTrace();
            System.out.println("�쳣��װ�����񽵼�");
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

9. ����αװ 
����� Dubbo �����ñ���αװʵ�ַ��񽵼�
����αװͨ�����ڷ��񽵼�������ĳ��Ȩ���񣬵������ṩ��ȫ���ҵ��󣬿ͻ��˲��׳��쳣������ͨ�� Mock ����
������Ȩʧ��

```java 
@DubboService
public class MockServiceImpl implements MockService {
    @Override
    public String mock(String s) {
        System.out.println("=======mockservice��ҵ����=======");
        try {
            Thread.sleep(1000000);
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("=======mockservice��ҵ�������=======");
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
 * 1���ӿ����� + Mock
 *
 * MockService + Mock
 *
 * 2��mock�߼����붨���ڽӿڵİ�����
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
 * 1���ӿ����� + Mock
 *
 * MockService + Mock
 *
 * 2��mock�߼����붨���ڽӿڵİ�����
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

// ��� mock����ֵΪtrue �Ǹ���Ҫ����ӿں�ʵ����Ҫ����ͬ����·���� �����������ǽӿ����� + Mock��������Ҫ�������ȫ·�����ơ� ͬʱҲ����mock = "force:return gpwy ֱ�ӽ��з��񽵼����ã������Ͳ�����÷���˽ӿ�ֱ�ӷ������õ�ֵ
//    @DubboReference(check = false,mock = "true")
//    @DubboReference(check = false,mock = "cn.enjoy.mock.LocalMockService")
    @DubboReference(check = false,mock = "force:return gpwy")
    private MockService mockService;
	
	    @Test
    public void mockTest(){
        System.out.println(mockService.mock("mock"));
    }
```

10. �첽����
�� Dubbo �з����첽����
�� 2.7.0 ��ʼ��Dubbo �������첽��̽ӿڿ�ʼ�� CompletableFuture Ϊ����
���� NIO �ķ�����ʵ�ֲ��е��ã��ͻ��˲���Ҫ�������̼߳�����ɲ��е��ö��Զ�̷�����Զ��߳̿�����С��


```java 
@DubboService
public class AsyncServiceImpl implements AsyncService {
    //�첽����
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
            System.out.println("=======doOne ��ҵ��ִ��=========");
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
        System.out.println("���е��������ӿ�====");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //��Ҫ�õ��첽���õķ��ؽ��
        CompletableFuture<Object> resultFuture = RpcContext.getContext().getCompletableFuture();
        resultFuture.whenComplete((retValue,exception)->{
            if(exception == null) {
                System.out.println("��������==" + retValue);
            } else {
                exception.printStackTrace();
            }
        });
        Thread.currentThread().join();
    }

```

