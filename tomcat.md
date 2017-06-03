### Tomcat8 优化

#### 1. 优化JVM
> 
    1.1 内存优化
    调整catalina.sh中JAVA_OPTS变量的设置，如下：
    具体设置如下： 
    JAVA_OPTS="$JAVA_OPTS -Xmx3550m -Xms3550m -Xss128k -XX:NewRatio=4 -XX:SurvivorRatio=4" 
> 
    其各项参数如下： 
    -Xmx3550m：设置JVM最大可用内存为3550M。 
    -Xms3550m：设置JVM促使内存为3550m。此值可以设置与-Xmx相同，以避免每次垃圾回收完成后JVM重新分配内存。 
    -Xmn2g：设置年轻代大小为2G。整个堆大小=年轻代大小 + 年老代大小 + 持久代大小。持久代一般固定大小为64m，所以增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。 
    -Xss128k：设置每个线程的堆栈大小。JDK5.0以后每个线程堆栈大小为1M，以前每个线程堆栈大小为256K。更具应用的线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。 
> 
    -XX:NewRatio=4:设置年轻代（包括Eden和两个Survivor区）与年老代的比值（除去持久代）。设置为4，则年轻代与年老代所占比值为1：4，年轻代占整个堆栈的1/5 
    -XX:SurvivorRatio=4：设置年轻代中Eden区与Survivor区的大小比值。设置为4，则两个Survivor区与一个Eden区的比值为2:4，一个Survivor区占整个年轻代的1/6 
    -XX:MaxPermSize=16m:设置持久代大小为16m。 
    -XX:MaxTenuringThreshold=0：设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概论。 
> 
    1.2 GC优化
> 
    垃圾回收的设置也是在catalina.sh中，调整JAVA_OPTS变量。 
    具体设置如下： 
    JAVA_OPTS="$JAVA_OPTS -Xmx3550m -Xms3550m -Xss128k -XX:+UseParallelGC  -XX:MaxGCPauseMillis=100" 
    具体的垃圾回收策略及相应策略的各项参数如下： 
> 
    串行收集器（JDK1.5以前主要的回收方式） 
    -XX:+UseSerialGC:设置串行收集器 
>     
    并行收集器（吞吐量优先） 
    示例： 
    java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:+UseParallelGC  -XX:MaxGCPauseMillis=100 
>     
    -XX:+UseParallelGC：选择垃圾收集器为并行收集器。此配置仅对年轻代有效。即上述配置下，年轻代使用并发收集，而年老代仍旧使用串行收集。 
    -XX:ParallelGCThreads=20：配置并行收集器的线程数，即：同时多少个线程一起进行垃圾回收。此值最好配置与处理器数目相等。 
    -XX:+UseParallelOldGC：配置年老代垃圾收集方式为并行收集。JDK6.0支持对年老代并行收集 
    -XX:MaxGCPauseMillis=100:设置每次年轻代垃圾回收的最长时间，如果无法满足此时间，JVM会自动调整年轻代大小，以满足此值。 
    -XX:+UseAdaptiveSizePolicy：设置此选项后，并行收集器会自动选择年轻代区大小和相应的Survivor区比例，以达到目标系统规定的最低相应时间或者收集频率等，此值建议使用并行收集器时，一直打开。 
>     
    并发收集器（响应时间优先） 
    示例：java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:+UseConcMarkSweepGC 
    -XX:+UseConcMarkSweepGC：设置年老代为并发收集。测试中配置这个以后，-XX:NewRatio=4的配置失效了，原因不明。所以，此时年轻代大小最好用-Xmn设置。 
    -XX:+UseParNewGC: 设置年轻代为并行收集。可与CMS收集同时使用。JDK5.0以上，JVM会根据系统配置自行设置，所以无需再设置此值。 
    -XX:CMSFullGCsBeforeCompaction：由于并发收集器不对内存空间进行压缩、整理，所以运行一段时间以后会产生“碎片”，使得运行效率降低。此值设置运行多少次GC以后对内存空间进行压缩、整理。 
    -XX:+UseCMSCompactAtFullCollection：打开对年老代的压缩。可能会影响性能，但是可以消除碎片 
    
    
#### 2. 优化server.xml
> 
    2.1 禁用Tomcat管理界面
>    
    生产环境一般不适用Tomcat默认的管理界面，这些页面存放在Tomcat        的webapps安装目录下，把该目录下的所有文件删除即可：
    rm -rf  /usr/local/tomcat8/webapps/*
>    
    另外删除相关的配置文件 host-manager.xml 和 manager.xml，在Tomcat     安装目录 conf/Catalina/localhost目录下。
    注释或删除tomcat_user.xml 中的所有用户权限。
>    
    2.2 禁用Tomcat热部署
>   
    tomcat默认 开启了对war热部署。为了防止被植入木马恶意攻击，我们要 关闭war包自动部署。关闭自动加载最新代码（设置reloadable）
>   
    >> 
    ```
    <Host name="localhost"  appBase="webapps"  
unpackWARs="false" autoDeploy="false"   
reloadable="false">
    ```
>
    2.3 更改Tomcat关闭指令
>   
    server.xml中定义了可以直接关闭 Tomcat 实例的管理端口。我们通过 telnet 连接上该端口之后，输入 SHUTDOWN （此为默认关闭指令）即可关闭 Tomcat 实例（注意，此时虽然实例关闭了，但是进程还是存在的）。由于默认关闭 Tomcat 的端口和指令都很简单。默认端口为8005，指令为SHUTDOWN 。因此我们需要将关闭指令修改复杂一点。
>    
    当然，在新版的Tomcat中该端口仅监听在127.0.0.1上，因此大家也不必担心。除非黑客登陆到tomcat本机去执行关闭操作。
修改实例：
   `<Server port="8005" shutdown="9SDKJ29jksjf23sjf0LSDF92JKS9DKkjsd">`
或者禁用8005端口
    `<Server port="-1" shutdown="SHUTDOWN">`
>    
    2.4 连接池配置
>
    使用线程池，用较少的线程处理较多的访问，可以提高tomcat处理请求的能力。编辑配置文件 server.xml : vi server.xml
>
    打开被注释的默认连接池配置
    默认配置：                                              
    <!--
    <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
        maxThreads="150" minSpareThreads="4"/>
     -->
>
    修改实例：           
    <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
     maxThreads="150" minSpareThreads="100" 
    prestartminSpareThreads="true" maxQueueSize="100"/> 
>
    参数说明:
    name: 线程名称
    namePrefix: 线程前缀
    maxThreads : 最大并发连接数，不配置时默认200，一般建议设置500~      800 ，要根据自己的硬件设施条件和实际业务需求而定。
    minSpareThreads：Tomcat启动初始化的线程数，默认值25   
    prestartminSpareThreads：在tomcat初始化的时候就初始化minSpareThr     eads的值， 不设置true时minSpareThreads   
    maxQueueSize: 最大的等待队列数，超过则拒绝请求
>    
    2.5 Connector配置
>   
    默认配置：
    <Connector port="8080" protocol="HTTP/1.1"
    connectionTimeout="20000"
    redirectPort="8443" />
>
    修改配置：
    <Connector port="8080"
    protocol="org.apache.coyote.http11.Http11Nio2Protocol"
    connectionTimeout="20000"
    redirectPort="8443" 
    executor="tomcatThreadPool"
    enableLookups="false" 
    acceptCount="100" 
    maxPostSize="10485760" 
    compression="on" 
    disableUploadTimeout="true" 
    compressionMinSize="2048" 
    noCompressionUserAgents="gozilla, traviata" 
    acceptorThreadCount="2" 
    compressableMimeType="text/html,text/xml,text/plain,text/css,text/javascript,application/javascript" 
    URIEncoding="utf-8"/>
>
    参数讲解：
    - port：连接端口。  
    - protocol：连接器使用的传输方式。Tomcat 8 设置 nio2 更好：org.apache.coyote.http11.Http11Nio2Protocol    
    protocol， Tomcat 6、7 设置 nio     更好：org.apache.coyote.http11.Http11NioProtocol      
    注：每个web客户端请求对于服务器端来说就一个单独的线程，客户端的请求数量增多将会导致线程数就上去了，CPU就忙着跟线程切换。而NIO则是使用单线程(单个CPU)或者只使用少量的多线程(多CPU)来接受Socket，而由线程池来处理堵塞在pipe或者队列里的请求.这样的话，只要OS可以接受TCP的连接，web服务器就可以处理该请求。大大提高了web服务器的可伸缩性。   
    - executor： 连接器使用的线程池名称
    - enableLookups：禁用DNS  查询 
    - acceptCount：指定当所有可以使用的处理请求的线程数都被使用时，可以放到处理队列中的请求数，超过这个数的请求将不予处理，默认设置 100 。
    - maxPostSize：限制 以FORM URL参数方式的POST请求的内容大小，单位字节，默认是 2097152(2兆)，10485760 为 10M。如果要禁用限制，则可以设置为 -1。
    - acceptorThreadCount： 用于接收连接的线程的数量，默认值是1。一般这个指需要改动的时候是因为该服务器是一个多核CPU，如果是多核 CPU 一般配置为 2。
    - compression：传输时是压缩。
    - compressionMinSize：压缩的大小
    - noCompressionUserAgents：不启用压缩的浏览器
    提示：压缩会增加Tomcat负担，最好采用Nginx + Tomcat 或者 Apache + Tomcat 方式，压缩交由Nginx/Apache 去做。 
    Tomcat 的压缩是在客户端请求服务器对应资源后，从服务器端将资源文件压缩，再输出到客户端，由客户端的浏览器负责解压缩并浏览。相对于普通的 浏览过程 HTML、CSS、Javascript和Text，它可以节省40% 左右的流量。更为重要的是，它可以对动态生成的，包括CGI、PHP、JSP、ASP、Servlet,SHTML等输出的网页也能进行压缩，压缩效率也很高。        
>
    2.6 应用程序部署
>    
    默认tomcat是root身份运行的，这样不安全。不要使用root用户启动tomcat。Java程序与C程序不同。nginx,httpd使用root用户启动守护80端口，子进程/线程会通过setuid(),setgid()两个函数切换到普通用户。即父进程所有者是root用户，子进程与多线程所有者是一个非root用户，这个用户没有shell，无法通过ssh与控制台登陆系统，Java的JVM是与系统无关的，是建立在OS之上的，你使用什么用户启动Tomcat，那麽Tomcat就会继承该所有者的权限。为了防止 Tomcat 被植入 web shell程序后，可以修改项目文件。因此我们要将 Tomcat 和项目的属主做分离，这样子，即便被搞，他也无法创建和编辑项目文件。 
    
#### 附：    
- [参考文档一](http://transcoder.tradaquan.com/from=844b/bd_page_type=1/ssid=0/uid=0/pu=usm%401%2Csz%401320_2001%2Cta%40iphone_1_10.3_3_603/baiduid=1720759AD4690EA321619F3ACD0122C0/w=0_10_/t=iphone/l=3/tc?ref=www_iphone&lid=14214402601153136240&order=9&fm=alop&h5ad=1&srd=1&dict=32&tj=www_normal_9_0_10_title&url_mf_score=5&vit=osres&m=8&cltj=cloud_title&asres=1&title=Tomcat%E8%B0%83%E4%BC%98-%E5%A4%AA%E6%B8%85-%E5%8D%9A%E5%AE%A2%E5%9B%AD&w_qd=IlPT2AEptyoA_yilI5qeGjVkf910miV3s_&sec=21468&di=9fb4eb1e19ca05ce&bdenc=1&tch=124.81.316.1802.0.0&nsrc=IlPT2AEptyoA_yixCFOxXnANedT62v3IEQGG_ytK1DK6mlrte4viZQRAWSHqLzrIBVWwdoTKtRwJuHSdAT-il17&eqid=c543b3dac81f08001000000659315538&wd=&clk_info=%7B%22srcid%22%3A%221599%22%2C%22tplname%22%3A%22www_normal%22%2C%22t%22%3A1496478439702%2C%22sig%22%3A%2254801%22%2C%22xpath%22%3A%22div-div-div-a-p%22%7D)
- [参考文档二](http://m.blog.csdn.net/article/details?id=51362676)
- [参考文档三](http://nolinux.blog.51cto.com/4824967/1608940)
- [参考文档四](http://www.jianshu.com/p/c8613d17e5fe)
