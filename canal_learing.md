[toc]



* [ ] admin
* [ ] client
* [ ] client-adapter
* [ ] common
* [x] connector
* [ ] dbsync
* [x] deployer
* [ ] docker
* [ ] example
* [ ] filter
* [ ] images
* [ ] instance
* [ ] meta
* [ ] parse
* [ ] prometheus
* [ ] protocol
* [x] server
* [ ] sink
* [ ] store

## deployer

> 部署者

### CanalLauncher

* [x] 定时同步远程配置，有更新则重启应用
* [x] 使用CountDownLatch 挂起，应用down了再解除挂起

```mermaid
sequenceDiagram
    alt 配置canal.admin.manager
    rect rgb(230, 230, 250)
    loop 每5s同步重启
    CanalLauncher->>PlainCanalConfigClient: findServer
    PlainCanalConfigClient ->> CanalLauncher: PlainCanal 有更新
    CanalLauncher->>CanalStarter: stop 
    CanalLauncher->>CanalStarter: start
    end
    end
   	CanalLauncher->>PlainCanalConfigClient: findServer
    PlainCanalConfigClient ->> CanalLauncher: PlainCanal 不为空, 将本地覆盖远程配置
    CanalLauncher->>CanalStarter: setProperties
    else 没有配置canal.admin.manager
    CanalLauncher->>CanalStarter: setProperties
    end 
    CanalLauncher->>CanalStarter: start
```

### CanalStarter

* [x] canalAdmin、controller、canalMQProducer、canalMQStarter

#### 启动流程图

```mermaid
sequenceDiagram
    alt canal.serverMode 是tcp模式
   	CanalStarter->>CanalController: start
    else canal.serverMode 非tcp模式,即mq模式
    rect rgb(230, 230, 250)
    CanalStarter->>+ExtensionLoader: getExtension
    ExtensionLoader->>-CanalStarter: canal.serverMode 配置的CanalMQProducer
    CanalStarter->>CanalController: start
    CanalStarter->>CanalMQStarter: start
    CanalStarter->>CanalController: setCanalMQStarter
    end
    end 
    alt canal.admin.port 有值
    CanalStarter->>CanalAdminWithNetty: start
    end
```

## server

### CanalMQStarter

```mermaid
sequenceDiagram
    alt !running
    rect rgb(230, 230, 250)
    CanalMQStarter->>+CanalServerWithEmbedded: instance
    CanalServerWithEmbedded->>-CanalMQStarter: 设置全局 canalServer
    CanalMQStarter->>+CanalMQRunnable: new Runnable
    CanalMQRunnable->>-CanalMQStarter: 设置destinations Map<String, CanalMQRunnable>
    CanalMQStarter->>ExecutorService: execute CanalMQRunnable
    end
    end 
```

#### CanalMQRunable

```mermaid
sequenceDiagram
    alt running 与 destinationRunning.get()
    rect rgb(230, 230, 250)
    CanalMQStarter->>+CanalServerWithEmbedded: getCanalInstances.get(destination)
    CanalServerWithEmbedded->>-CanalMQStarter: 返回destination对应的canal实例
    CanalMQStarter->>CanalServerWithEmbedded: subscribe
    CanalMQStarter->>+CanalServerWithEmbedded: getWithoutAck
    CanalServerWithEmbedded->>-CanalMQStarter: Message
    CanalMQStarter->>CanalMQProducer: send(MQDestination canalDestination, Message message, Callback callback)
    end
    end 
```

##### CanalServerWithEmbedded

> subscribe

```mermaid
sequenceDiagram
    CanalServerWithEmbedded->>CanalServerWithEmbedded: checkStart
    CanalServerWithEmbedded->>+CanalMetaManager: isStart
    CanalMetaManager->>-CanalServerWithEmbedded: 返回是否启动
    alt !未启动
    	 CanalServerWithEmbedded->>CanalMetaManager: start
    end
    CanalServerWithEmbedded->>CanalMetaManager: subsribe
    CanalServerWithEmbedded->>+CanalMetaManager: getCursor
    CanalMetaManager->>-CanalServerWithEmbedded: 返回position
   	alt position 为空
   		CanalServerWithEmbedded->>+CanalEventStore: getFirstPosition
   		CanalEventStore->>-CanalServerWithEmbedded: 返回第一条数据的position
   		alt position 不为空
   			CanalServerWithEmbedded->>+CanalMetaManager: updateCursor
   		end
   	end
   	CanalServerWithEmbedded->>CanalInstance: subscribeChange
```



## connector

### CanalRocketMQProducer



## client-adapter

### 代码片段

> com.alibaba.otter.canal.client.adapter.support.Util#sqlRS(javax.sql.DataSource, java.lang.String, java.util.List<java.lang.Object>, java.util.function.Function<java.sql.ResultSet,java.lang.Object>)
>
> 匿名函数的运用

🌰

```java
public class Test {

    public static void main(String[] args) {
        System.out.println(func(rs -> rs.getA() + rs.getB()));
        System.out.println(func(rs -> rs.getA() * rs.getB()));
    }

    public static Object func(Function<Args, Object> func) {
        Args args = new Args();
        args.setA(1);
        args.setB(2);
        return func.apply(args);
    }

    static class Args {
        private int a;
        private int b;

        public int getA() {
            return a;
        }

        public void setA(int a) {
            this.a = a;
        }

        public int getB() {
            return b;
        }

        public void setB(int b) {
            this.b = b;
        }
    }
}
```

