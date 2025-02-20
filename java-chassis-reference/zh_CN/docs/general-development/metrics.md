# 应用性能监控（metrics）

应用性能监控通过周期性的统计数据，帮助开发者分析业务性能瓶颈，在性能优化、故障定位的时候非常有帮助。应用性能监控的指标包括
请求各个环节的时延、线程池使用情况、连接池使用情况、CPU和网络使用情况等。

统计数据是一个周期性时效数据，只在统计周期内具有意义。可以选择通过 `REST` 接口查询实时的统计数据，也可以通过日志开关，将
统计数据输出到日志文件里面，帮助分析性能问题。

## 使用方法

使用应用性能监控非常简单， 只需要在应用中包含相关的依赖包，通过配置项开启和关闭相关功能。 

1. 开启应用性能监控功能， 需要引入下面的依赖：

        ```
        <dependency>
          <groupId>org.apache.servicecomb</groupId>
          <artifactId>metrics-core</artifactId>
        </dependency>
        ```

  如果与prometheus集成，则还需要加入依赖
  
        ```
        <dependency>
          <groupId>org.apache.servicecomb</groupId>
          <artifactId>metrics-prometheus</artifactId>
        </dependency>
        ```

  通过在 `microservice.yaml` 中增加下面的配置项，就可以将性能统计数据输出到日志文件中。
  
        ```
        servicecomb:
          metrics:
            window_time: 60000
            invocation:
              latencyDistribution: 0,1,10,100,1000
            Consumer.invocation.slow:
              enabled: true
              msTime: 1000
            Provider.invocation.slow:
              enabled: true
              msTime: 1000
            publisher.defaultLog:
              enabled: true
              endpoints.client.detail.enabled: true
        ```
  
  上述配置开启了慢调用检测，如果存在慢调用，则会立即输出相应日志：

        ```
        2019-04-02 23:01:09,103\[WARN]\[pool-7-thread-74]\[5ca37935c00ff2c7-350076] - slow(40 ms) invocation, CONSUMER highway perf1.impl.syncQuery
          http method: GET
          url        : /v1/syncQuery/{id}/
          server     : highway://192.168.0.152:7070?login=true
          status code: 200
          total      : 50.760 ms
            prepare                : 0.0 ms
            handlers request       : 0.0 ms
            client filters request : 0.0 ms
            send request           : 0.5 ms
            get connection         : 0.0 ms
            write to buf           : 0.5 ms
            wait response          : 50.727 ms
            wake consumer          : 0.23 ms
            client filters response: 0.2 ms
            handlers response      : 0.0 ms (SlowInvocationLogger.java:121)
        ```

  其中 5ca37935c00ff2c7-350076 是 `${traceId}-${invocationId}` 的结构，在log4j2 或 logback 的输出格式中通过 %marker 引用。
  
  也可以通过 `REST` 接口查询性能统数据。使用浏览器访问http://ip:port/metrics 即可，将会得到类似下面格式的json数据：
                         
        ```
        {
        "servicecomb.vertx.endpoints(address=192.168.0.124:7070,statistic=connectCount,type=client)": 0.0,
        "servicecomb.vertx.endpoints(address=192.168.0.124:7070,statistic=disconnectCount,type=client)": 0.0,
        "servicecomb.vertx.endpoints(address=192.168.0.124:7070,statistic=connections,type=client)": 1.0,
        "servicecomb.vertx.endpoints(address=192.168.0.124:7070,statistic=bytesRead,type=client)": 508011.0,
        "servicecomb.vertx.endpoints(address=192.168.0.124:7070,statistic=bytesWritten,type=client)": 542163.0,
        "servicecomb.vertx.endpoints(address=192.168.0.124:7070,statistic=queueCount,type=client)": 0.0,
        
        "servicecomb.vertx.endpoints(address=0.0.0.0:7070,statistic=connectCount,type=server)": 0.0,
        "servicecomb.vertx.endpoints(address=0.0.0.0:7070,statistic=disconnectCount,type=server)": 0.0,
        "servicecomb.vertx.endpoints(address=0.0.0.0:7070,statistic=connections,type=server)": 1.0,
        "servicecomb.vertx.endpoints(address=0.0.0.0:7070,statistic=bytesRead,type=server)": 542163.0
        ... ...
        }
        ```

## 配置说明

<div class="metrics-cfg"></div>

| 配置项 | 默认值 | 含义 |
| :--- | :--- | :--- |
| servicecomb.metrics.window_time                | 60000 | 统计周期，单位为毫秒<br>TPS、时延等等周期性的数据，每周期更新一次，在周期内获取到的值，实际是上一周期的值 |
| servicecomb.metrics<br>.invocation.latencyDistribution          |       | 时延分布时间段定义，单位为毫秒<br>例如：0,1,10,100,1000<br>表示定义了下列时延段[0, 1),[1, 10),[10, 100),[100, 1000),[1000, ) |
| servicecomb.metrics<br>.Consumer.invocation.slow.enabled | false | 是否开启Consumer端的慢调用检测<br>通过增加后缀.${service}.${schema}.${operation}，可以支持4级优先级定义 |
| servicecomb.metrics<br>.Consumer.invocation.slow.msTime | 1000 | 时延超过配置值，则会立即输出日志，记录本次调用的stage耗时信息<br>通过增加后缀.${service}.${schema}.${operation}，可以支持4级优先级定义 |
| servicecomb.metrics<br>.Provider.invocation.slow.enabled | false | 是否开启Provide端的慢调用检测<br>通过增加后缀.${service}.${schema}.${operation}，可以支持4级优先级定义 |
| servicecomb.metrics<br>.Provider.invocation.slow.msTime | 1000 | 时延超过配置值，则会立即输出日志，记录本次调用的stage耗时信息<br>通过增加后缀.${service}.${schema}.${operation}，可以支持4级优先级定义 |
| servicecomb.metrics<br>.prometheus.address | 0.0.0.0:9696 | prometheus监听地址 |
| servicecomb.metrics.publisher.defaultLog<br>.enabled | false | 是否输出默认的统计日志 |
| servicecomb.metrics.publisher.defaultLog<br>.endpoints.client.detail.enabled | false | 是否输出每一条client endpoint统计日志，因为跟目标的ip:port数有关，可能会有很多数据，所以默认不输出 |

## 统计指标含义说明

* CPU 统计信息

<table class="metrics-table">
  <tr>
    <th>Name</th>
    <th>Tag keys</th>
    <th>Tag values</th>
    <th>含义</th>
  </tr>
  <tr>
    <td rowspan="2">os</td>
    <td rowspan="2">type</td>
    <td>cpu</td>
    <td>当前周期内系统CPU使用率，Solaris模式</td>
  </tr>
  <tr>
    <td>processCpu</td>
    <td>当前周期内微服务进程CPU使用率，IRIX模式<br>
        processCpu除以cpu近似等于系统CPU数</td>
  </tr>
</table>

* 网络统计信息

<table class="metrics-table">
  <tr>
    <th>Name</th>
    <th>Tag keys</th>
    <th>Tag values</th>
    <th>含义</th>
  </tr>
  <tr>
    <td rowspan="6">os</td>
    <td>type</td>
    <td>net</td>
    <td></td>
  </tr>
  <tr>
    <td rowspan="4">statistic</td>
    <td>send</td>
    <td>当前周期内平均每秒发送的字节数(Bps)</td>
  </tr>
  <tr>
    <td>receive</td>
    <td>当前周期内平均每秒接收的字节数(Bps)</td>
  </tr>
  <tr>
    <td>sendPackets</td>
    <td>当前周期内平均每秒发送的包数(pps)</td>
  </tr>
  <tr>
    <td>receivePackets</td>
    <td>当前周期内平均每秒接收的包数(pps)</td>
  </tr>
  <tr>
    <td>interface</td>
    <td></td>
    <td>网卡设备名</td>
  </tr>
</table>

* vertx client endpoints 统计信息 
  
<table class="metrics-table">
  <tr>
    <th>Name</th>
    <th>Tag keys</th>
    <th>Tag values</th>
    <th>含义</th>
  </tr>
  <tr>
    <td rowspan="8">servicecomb<br>.vertx<br>.endpoints</td>
    <td>type</td>
    <td>client</td>
    <td></td>
  </tr>
  <tr>
    <td>address</td>
    <td>${ip}:${port}</td>
    <td>服务端的ip:port</td>
  </tr>
  <tr>
    <td rowspan="6">statistic</td>
    <td>connectCount</td>
    <td>当前周期内共发起多少次连接</td>
  </tr>
  <tr>
    <td>disconnectCount</td>
    <td>当前周期内断连的次数</td>
  </tr>
  <tr>
    <td>queueCount</td>
    <td>http连接池中正在等待获取连接的请求数</td>
  </tr>
  <tr>
    <td>connections</td>
    <td>当前时刻的连接数</td>
  </tr>
  <tr>
    <td>bytesRead</td>
    <td>当前周期内平均每秒接收的字节数(Bps)<br>
        业务层的统计，相对从网卡获取的数据，这里的数据不包括包头的大小<br>
        对于http消息，不包括http header大小</td>
  </tr>
  <tr>
    <td>bytesWritten</td>
    <td>当前周期内平均每秒发送的字节数(Bps)<br>
        业务层的统计，相对从网卡获取的数据，这里的数据不包括包头的大小<br>
        对于http消息，不包括http header大小</td>
  </tr>
</table>

* vertx server endpoints 统计信息
  
<table class="metrics-table">
  <tr>
    <th>Name</th>
    <th>Tag keys</th>
    <th>Tag values</th>
    <th>含义</th>
  </tr>
  <tr>
    <td rowspan="8">servicecomb<br>.vertx<br>.endpoints</td>
    <td>type</td>
    <td>server</td>
    <td></td>
  </tr>
  <tr>
    <td>address</td>
    <td>${ip}:${port}</td>
    <td>监听的ip:port</td>
  </tr>
  <tr>
    <td rowspan="6">statistic</td>
    <td>connectCount</td>
    <td>当前周期内共接入多少次连接</td>
  </tr>
  <tr>
    <td>disconnectCount</td>
    <td>当前周期内断连的次数</td>
  </tr>
  <tr>
    <td>rejectByConnectionLimit</td>
    <td>当前周期内因超出连接数限制而主动断连的次数</td>
  </tr>
  <tr>
    <td>connections</td>
    <td>当前时刻的连接数</td>
  </tr>
  <tr>
    <td>bytesRead</td>
    <td>当前周期内平均每秒发送的字节数(Bps)<br>
        业务层的统计，相对从网卡获取的数据，这里的数据不包括包头的大小<br>
        对于http消息，不包括http header大小</td>
  </tr>
  <tr>
    <td>bytesWritten</td>
    <td>当前周期内平均每秒接收的字节数(Bps)<br>
        业务层的统计，相对从网卡获取的数据，这里的数据不包括包头的大小<br>
        对于http消息，不包括http header大小</td>
  </tr>
</table>

* 时延分布概览
 
<table class="metrics-table">
  <tr>
    <th>Name</th>
    <th>Tag keys</th>
    <th>Tag values</th>
    <th>含义</th>
  </tr>
  <tr>
    <td rowspan="11">servicecomb<br>.invocation</td>
    <td>role</td>
    <td>CONSUMER、PRODUCER、EDGE</td>
    <td>是CONSUMER、PRODUCER还是EDGE端的统计</td>
  </tr>
  <tr>
    <td>operation</td>
    <td>${microserviceName}<br>.${schemaId}<br>.${operationName}</td>
    <td>调用的方法名</td>
  </tr>
  <tr>
    <td>transport</td>
    <td>highway或rest</td>
    <td>调用是在哪个传输通道上发生的</td>
  </tr>
  <tr>
    <td>status</td>
    <td>http status code</td>
    <td></td>
  </tr>
  <tr>
    <td>type</td>
    <td>latencyDistribution</td>
    <td>调用时延分布</td>
  </tr>
  <tr>
    <td>scope</td>
    <td>[${min}, ${max})</td>
    <td>当前周期内调用时延大于等于min，小于max的次数<br>
        [${min},)表示max为无限大</td>
  </tr>
</table>

* consumer 详细时延分布

<table class="metrics-table">
  <tr>
    <th>Name</th>
    <th>Tag keys</th>
    <th>Tag values</th>
    <th>含义</th>
  </tr>
  <tr>
    <td rowspan="19">servicecomb<br>.invocation</td>
    <td>role</td>
    <td>CONSUMER</td>
    <td>CONSUMER端的统计</td>
  </tr>
  <tr>
    <td>operation</td>
    <td>${microserviceName}<br>.${schemaId}<br>.${operationName}</td>
    <td>调用的方法名</td>
  </tr>
  <tr>
    <td>transport</td>
    <td>highway或rest</td>
    <td>调用是在哪个传输通道上发生的</td>
  </tr>
  <tr>
    <td>status</td>
    <td>http status code</td>
    <td></td>
  </tr>
  <tr>
    <td>type</td>
    <td>stage</td>
    <td>stage时延</td>
  </tr>
  <tr>
    <td rowspan="11">stage</td>
    <td>total</td>
    <td>全流程</td>
  </tr>
  <tr>
    <td>prepare</td>
    <td>初始化</td>
  </tr>
  <tr>
    <td>handlers_request</td>
    <td>handler链请求流程</td>
  </tr>
  <tr>
    <td>client_filters_request</td>
    <td>http client filter链请求流程<br>
        只有走rest transport才有本阶段</td>
  </tr>
  <tr>
    <td>consumer_send_request</td>
    <td>发送请求阶段，包括consumer_get_connection和consumer_write_to_buf</td>
  </tr>
  <tr>
    <td>consumer_get_connection</td>
    <td>从连接池获取连接</td>
  </tr>
  <tr>
    <td>consumer_write_to_buf</td>
    <td>向网络缓冲区写数据</td>
  </tr>
  <tr>
    <td>consumer_wait_response</td>
    <td>等待服务端应答</td>
  </tr>
  <tr>
    <td>consumer_wake_consumer</td>
    <td>同步流程中，收到应答后，从唤醒等待线程，到等待线程开始处理应答的耗时</td>
  </tr>
  <tr>
    <td>client_filters_response</td>
    <td>http client filter链应答流程</td>
  </tr>
  <tr>
    <td>handlers_response</td>
    <td>handler链应答流程</td>
  </tr>
  <tr>
    <td rowspan="3">statistic</td>
    <td>count</td>
    <td>平均每秒调用次数，即TPS<br>
        count=统计周期内的调用次数/周期（秒）</td>
  </tr>
  <tr>
    <td>totalTime</td>
    <td>单位为秒<br>
        totalTime=当前周期内的调用耗时总时长/周期（秒）<br>
        totalTime除以count即可得到平均时延</td>
  </tr>
  <tr>
    <td>max</td>
    <td>单位为秒<br>
        当前周期内最大耗时</td>
  </tr>
</table>

* provider 详细时延分布

<table class="metrics-table">
  <tr>
    <th>Name</th>
    <th>Tag keys</th>
    <th>Tag values</th>
    <th>含义</th>
  </tr>
  <tr>
    <td rowspan="17">servicecomb<br>.invocation</td>
    <td>role</td>
    <td>PRODUCER</td>
    <td>PRODUCER端的统计</td>
  </tr>
  <tr>
    <td>operation</td>
    <td>${microserviceName}<br>.${schemaId}<br>.${operationName}</td>
    <td>调用的方法名</td>
  </tr>
  <tr>
    <td>transport</td>
    <td>highway或rest</td>
    <td>调用是在哪个传输通道上发生的</td>
  </tr>
  <tr>
    <td>status</td>
    <td>http status code</td>
    <td></td>
  </tr>
  <tr>
    <td>type</td>
    <td>stage</td>
    <td>stage时延</td>
  </tr>
  <tr>
    <td rowspan="9">stage</td>
    <td>total</td>
    <td>全流程</td>
  </tr>
  <tr>
    <td>prepare</td>
    <td>初始化</td>
  </tr>
  <tr>
    <td>queue</td>
    <td>仅在使用线程池时有意义<br>
        表示调用在线程池中排队的时长</td>
  </tr>
  <tr>
    <td>server_filters_request</td>
    <td>http server filter链请求流程<br>
        只有走rest transport才有本阶段</td>
  </tr>
  <tr>
    <td>handlers_request</td>
    <td>handler链请求流程</td>
  </tr>
  <tr>
    <td>execution</td>
    <td>业务方法</td>
  </tr>
  <tr>
    <td>handlers_response</td>
    <td>handler链应答流程</td>
  </tr>
  <tr>
    <td>server_filters_response</td>
    <td>http server filter链应答流程</td>
  </tr>
  <tr>
    <td>producer_send_response</td>
    <td>发送应答</td>
  </tr>
  <tr>
    <td rowspan="3">statistic</td>
    <td>count</td>
    <td>平均每秒调用次数，即TPS<br>
        count=统计周期内的调用次数/周期（秒）</td>
  </tr>
  <tr>
    <td>totalTime</td>
    <td>单位为秒<br>
        totalTime=当前周期内的调用耗时总时长/周期（秒）<br>
        totalTime除以count即可得到平均时延</td>
  </tr>
  <tr>
    <td>max</td>
    <td>单位为秒<br>
        当前周期内最大耗时</td>
  </tr>
</table>

* edge service 详细时延分布

<table class="metrics-table">
  <tr>
    <th>Name</th>
    <th>Tag keys</th>
    <th>Tag values</th>
    <th>含义</th>
  </tr>
  <tr>
    <td rowspan="23">servicecomb<br>.invocation</td>
    <td>role</td>
    <td>EDGE</td>
    <td>EDGE的统计</td>
  </tr>
  <tr>
    <td>operation</td>
    <td>${microserviceName}<br>.${schemaId}<br>.${operationName}</td>
    <td>调用的方法名</td>
  </tr>
  <tr>
    <td>transport</td>
    <td>highway或rest</td>
    <td>调用是在哪个传输通道上发生的</td>
  </tr>
  <tr>
    <td>status</td>
    <td>http status code</td>
    <td></td>
  </tr>
  <tr>
    <td>type</td>
    <td>stage</td>
    <td>stage时延</td>
  </tr>
  <tr>
    <td rowspan="15">stage</td>
    <td>total</td>
    <td>全流程</td>
  </tr>
  <tr>
    <td>prepare</td>
    <td>初始化</td>
  </tr>
  <tr>
    <td>queue</td>
    <td>仅在使用线程池时有意义<br>
        表示调用在线程池中排队的时长</td>
  </tr>
  <tr>
    <td>server_filters_request</td>
    <td>http server filter链请求流程</td>
  </tr>
  <tr>
    <td>handlers_request</td>
    <td>handler链请求流程</td>
  </tr>
  <tr>
    <td>client_filters_request</td>
    <td>http client filter链请求流程</td>
  </tr>
  <tr>
    <td>consumer_send_request</td>
    <td>发送请求阶段，包括consumer_get_connection和consumer_write_to_buf</td>
  </tr>
  <tr>
    <td>consumer_get_connection</td>
    <td>从连接池获取连接</td>
  </tr>
  <tr>
    <td>consumer_write_to_buf</td>
    <td>向网络缓冲区写数据</td>
  </tr>
  <tr>
    <td>consumer_wait_response</td>
    <td>等待服务端应答</td>
  </tr>
  <tr>
    <td>consumer_wake_consumer</td>
    <td>同步流程中，收到应答后，从唤醒等待线程，到等待线程开始处理应答的耗时</td>
  </tr>
  <tr>
    <td>client_filters_response</td>
    <td>http client filter链应答流程</td>
  </tr>
  <tr>
    <td>handlers_response</td>
    <td>handler链应答流程</td>
  </tr>
  <tr>
    <td>server_filters_response</td>
    <td>http server filter链应答流程</td>
  </tr>
  <tr>
    <td>producer_send_response</td>
    <td>发送应答</td>
  </tr>
  <tr>
    <td rowspan="3">statistic</td>
    <td>count</td>
    <td>平均每秒调用次数，即TPS<br>
        count=统计周期内的调用次数/周期（秒）</td>
  </tr>
  <tr>
    <td>totalTime</td>
    <td>单位为秒<br>
        totalTime=当前周期内的调用耗时总时长/周期（秒）<br>
        totalTime除以count即可得到平均时延</td>
  </tr>
  <tr>
    <td>max</td>
    <td>单位为秒<br>
        当前周期内最大耗时</td>
  </tr>
</table>

* 线程池信息

<table class="metrics-table">
  <tr>
    <th>Name</th>
    <th>Tag keys</th>
    <th>Tag values</th>
    <th>含义</th>
  </tr>
  <tr>
    <td>threadpool.corePoolSize  </td>
    <td rowspan="8">id</td>
    <td rowspan="8">${threadPoolName}</td>
    <td>最小线程数</td>
  </tr>
  <tr>
    <td>threadpool.maxThreads </td>
    <td>最大允许的线程数</td>
  </tr>
  <tr>
    <td>threadpool.poolSize </td>
    <td>当前实际线程数</td>
  </tr>
  <tr>
    <td>threadpool.currentThreadsBusy </td>
    <td>当前的活动线程数，即当前正在执行的任务数</td>
  </tr>
  <tr>
    <td>threadpool.queueSize </td>
    <td>当前正在排队的任务数</td>
  </tr>
  <tr>
    <td>threadpool.rejectedCount </td>
    <td>当前周期内平均每秒被拒绝的任务数</td>
  </tr>
  <tr>
    <td>threadpool.taskCount</td>
    <td>统计周期内平均每秒提交的任务数<br>taskCount=（completed + queue + active）/周期（秒）</td>
  </tr>
  <tr>
    <td>threadpool.completedTaskCount </td>
    <td>统计周期内平均每秒完成的任务数<br>completedTaskCount=completed/周期（秒）</td>
  </tr>
</table>


## 开发者信息和高级课题

* 实现原理

![](../assets/metrics/logicDiagram.png)

    1. 基于[netflix spectator](https://github.com/Netflix/spectator)
    2. Foundation-metrics通过SPI机制加载所有MetricsInitializer实现，实现者可以通过MetricsInitializer中的getOrder规划执行顺序，order数字越小，越先执行。
    3. Metrics-core实现3类MetricsInitializer：
      1. DefaultRegistryInitializer: 实例化并注册spectator-reg-servo，设置较小的order，保证比下面2类MetricsInitializer先执行
      2. Meters Initializer: 实现TPS、时延、线程池、jvm资源等等数据的统计
      3. Publisher: 输出统计结果，内置了日志输出，以及通过RESTful接口输出
    4. Metrics-prometheus提供与prometheus对接的能力


* 业务定制

因为ServiceComb已经初始化了servo的registry，所以业务不必再创建registry。实现MetricsInitializer接口，定
  义业务级的Meters，或实现定制的Publisher，再通过SPI机制声明自己的实现即可。

  1.Meters
  
    创建Meters能力均由spectator提供，可查阅[netflix spectator](https://github.com/Netflix/spectator)文档

  2.Publisher
  
    周期性输出的场景，比如日志场景，通过eventBus订阅org.apache.servicecomb.foundation.metrics.PolledEvent，PolledEvent.getMeters()即是本周期的统计结果
     非周期性输出的场景，比如通过RESTful接口访问，通过globalRegistry.iterator()即可得到本周期的统计结果
