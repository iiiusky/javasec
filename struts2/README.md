## Apache Struts2 RCE via Log4j2 CVE-2021-44228

### 环境搭建
首先搭建环境
- 下载源码：https://dlcdn.apache.org/struts/2.5.27/struts-2.5.27-all.zip，解压
- 复制struts-2.5.27\src\apps\showcase文件夹到自己的测试目录，右键用IDEA打开
- 配置tomcat，启动即可

![image](vulnerability-research.assets/145716989-360e998a-0014-44d2-b37c-cce6fd7e310e.png)

注：确认是否使用log4j2组件

![image](vulnerability-research.assets/145717003-47737614-74c3-45e8-89d4-8cd971fdee39.png)


### 漏洞分析

根据流传的payload中携带的HTTP头字段`If-Modified-Since`找到对应的源码位置，下断点

![image](vulnerability-research.assets/145717032-722780ec-d87b-4dca-af86-0354e33491fc.png)

- org.apache.struts2.dispatcher.DefaultStaticContentLoader#process

![image](vulnerability-research.assets/145717042-0b40e957-e827-40b1-a258-d89769cb1ad5.png)


![image](vulnerability-research.assets/145717306-5a735d51-7867-40b8-85d7-ed3533875387.png)

整个调用栈
> ```java
> warn:2774, AbstractLogger (org.apache.logging.log4j.spi)
> process:241, DefaultStaticContentLoader (org.apache.struts2.dispatcher)
> findStaticResource:215, DefaultStaticContentLoader (org.apache.struts2.dispatcher)
> executeStaticResourceRequest:59, ExecuteOperations (org.apache.struts2.dispatcher)
> doFilter:81, StrutsExecuteFilter (org.apache.struts2.dispatcher.filter)
> internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
> doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
> doFilter:65, SiteMeshFilter (com.opensymphony.sitemesh.webapp)
> internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
> doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
> doFilter:92, StrutsPrepareFilter (org.apache.struts2.dispatcher.filter)
> internalDoFilter:193, ApplicationFilterChain (org.apache.catalina.core)
> doFilter:166, ApplicationFilterChain (org.apache.catalina.core)
> invoke:196, StandardWrapperValve (org.apache.catalina.core)
> invoke:97, StandardContextValve (org.apache.catalina.core)
> invoke:544, AuthenticatorBase (org.apache.catalina.authenticator)
> invoke:135, StandardHostValve (org.apache.catalina.core)
> invoke:81, ErrorReportValve (org.apache.catalina.valves)
> invoke:698, AbstractAccessLogValve (org.apache.catalina.valves)
> invoke:78, StandardEngineValve (org.apache.catalina.core)
> service:364, CoyoteAdapter (org.apache.catalina.connector)
> service:624, Http11Processor (org.apache.coyote.http11)
> process:65, AbstractProcessorLight (org.apache.coyote)
> process:831, AbstractProtocol$ConnectionHandler (org.apache.coyote)
> doRun:1650, NioEndpoint$SocketProcessor (org.apache.tomcat.util.net)
> run:49, SocketProcessorBase (org.apache.tomcat.util.net)
> runWorker:1191, ThreadPoolExecutor (org.apache.tomcat.util.threads)
> run:659, ThreadPoolExecutor$Worker (org.apache.tomcat.util.threads)
> run:61, TaskThread$WrappingRunnable (org.apache.tomcat.util.threads)
> run:745, Thread (java.lang)
> ```

根据调用栈中的调用情况，分析为什么请求会执行到这里
- ALT + F7

![image](vulnerability-research.assets/145717438-6546ca05-c3c1-4d3c-ae6b-042906149b29.png)

![image](vulnerability-research.assets/145717545-86ceb682-0867-49d6-b538-d0a50f73930f.png)

![image](vulnerability-research.assets/145717566-8eb50b1a-b190-4c2a-8d9e-d556917f2851.png)

![image](vulnerability-research.assets/145717713-e8f50df4-3490-43d1-8c76-d01e4a1f7196.png)

捋一捋其请求的执行流程
- 1、请求A 首先被StrutsExecuteFilter进行处理
  - execute.executeStaticResourceRequest  `处理对静态资源的请求`
    ![image](vulnerability-research.assets/145718094-0007d715-0105-4d0f-8587-af4162f8e077.png)

- 2、需要满足条件（请求的静态资源路径以“/struts”或“/static”开头）
  - ![image](vulnerability-research.assets/145718186-123dd677-bb4d-438a-a77b-1b8bdd564841.png)

- 3、条件满足后，执行到org.apache.struts2.dispatcher.DefaultStaticContentLoader#findStaticResource
  - 需要满足条件：请求的静态资源需要存在，否则直接返回404
  - 可以通过右键查看源码查找需要的路径
    ![image](vulnerability-research.assets/145718469-f53027a1-6403-4b3a-b0cb-cb481ea24a53.png)

- 4、条件满足后，执行到org.apache.struts2.dispatcher.DefaultStaticContentLoader#process
  - 构造恶意的If-Modified-Since即可触发log4j2的RCE
  ![image](vulnerability-research.assets/145718511-bb6c8844-472f-4238-9781-dd35a4751fbf.png)
  

至此，分析完毕。

### 漏洞复现


![image](vulnerability-research.assets/145717219-5339230e-b62d-464d-ab50-4aaa995dcc12.png)


参考：

https://twitter.com/payloadartist/status/1469987703429103622
