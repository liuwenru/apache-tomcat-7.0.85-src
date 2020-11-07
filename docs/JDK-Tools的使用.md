# JDK-Tools的使用介绍

本篇主要是讲解一下关于`JDK`中自带的一些诊断工具的使用，这些工具的功能说白了就是帮我们去观测`JVM`是否运行正常的，这个不是唯一的方法，现在市面上当然还有其他的工具一样可以帮我们完成这些活而且做的更好比如`JProfile`，但是因为这个需要收费，而且需要在线上去重启`JVM`，万一要是很难复现的间歇问题已重启就没有了。所以笔者还是倾向于使用土办法自己撸起袖子干。

-----------



## 一、 环境配置


因为线上环境是没有`JDK`的只有`JRE`环境，所以我们在使用前需要先`export`好`JDK`的环境变量(熟悉`Linux`的同学应该不陌生如何在当前的`Shell`环境变量中`export`变量，`Windows`系统中也可以只是个人实践幺蛾子比较多)，笔者建议请使用对应版本的`JDK`。以`Linux`环境为例，在临时在当前的环境变量中添加`JDK`

```bash
Shell> export JAVA_HOME=/opt/jdk
Shell> export PATH=$PATH:$JAVA_HOME/bin
```

如上步骤配置完成后即可实现在当前环境`Shell`中使用`JDK 提供的工具`


**为了线上系统的安全，生产环境中调试完毕后请及时的删除JDK**


在使用了`Dcoker`容器后，有时使用`JDK`工具时会出现如下报错

```bash
/opt # jmap -heap  127
Attaching to process ID 127, please wait...
Error attaching to process: sun.jvm.hotspot.debugger.DebuggerException: Can't attach to the process
sun.jvm.hotspot.debugger.DebuggerException: sun.jvm.hotspot.debugger.DebuggerException: Can't attach to the process
        at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$LinuxDebuggerLocalWorkerThread.execute(LinuxDebuggerLocal.java:163)
        at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.attach(LinuxDebuggerLocal.java:278)
        at sun.jvm.hotspot.HotSpotAgent.attachDebugger(HotSpotAgent.java:671)
        at sun.jvm.hotspot.HotSpotAgent.setupDebuggerLinux(HotSpotAgent.java:611)
        at sun.jvm.hotspot.HotSpotAgent.setupDebugger(HotSpotAgent.java:337)
        at sun.jvm.hotspot.HotSpotAgent.go(HotSpotAgent.java:304)
        at sun.jvm.hotspot.HotSpotAgent.attach(HotSpotAgent.java:140)
        at sun.jvm.hotspot.tools.Tool.start(Tool.java:185)
        at sun.jvm.hotspot.tools.Tool.execute(Tool.java:118)
        at sun.jvm.hotspot.tools.HeapSummary.main(HeapSummary.java:49)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:497)
        at sun.tools.jmap.JMap.runTool(JMap.java:201)
        at sun.tools.jmap.JMap.main(JMap.java:130)
Caused by: sun.jvm.hotspot.debugger.DebuggerException: Can't attach to the process
        at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.attach0(Native Method)
        at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.access$100(LinuxDebuggerLocal.java:62)
        at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$1AttachTask.doit(LinuxDebuggerLocal.java:269)
        at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal$LinuxDebuggerLocalWorkerThread.run(LinuxDebuggerLocal.java:138)
```

出现该问题的时候主要是因为`Dcoker`的权限控制机制导致的，解决方法很简单说只要在启动容器的过程中加入参数`--cap-add=SYS_PTRACE`即可。


在使用过程中有时我们还会得到如下所示的错误，

```bash
Exception in thread "main" java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:497)
        at sun.tools.jmap.JMap.runTool(JMap.java:201)
        at sun.tools.jmap.JMap.main(JMap.java:130)
Caused by: sun.jvm.hotspot.debugger.UnmappedAddressException: 3080318
        at sun.jvm.hotspot.debugger.PageCache.checkPage(PageCache.java:208)
        at sun.jvm.hotspot.debugger.PageCache.getData(PageCache.java:63)
        at sun.jvm.hotspot.debugger.DebuggerBase.readBytes(DebuggerBase.java:225)
        at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.readCInteger(LinuxDebuggerLocal.java:498)
        at sun.jvm.hotspot.debugger.DebuggerBase.readAddressValue(DebuggerBase.java:462)
        at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.readAddress(LinuxDebuggerLocal.java:433)
        at sun.jvm.hotspot.debugger.linux.LinuxAddress.getAddressAt(LinuxAddress.java:74)
        at sun.jvm.hotspot.types.basic.BasicTypeDataBase.findDynamicTypeForAddress(BasicTypeDataBase.java:296)
        at sun.jvm.hotspot.runtime.VirtualBaseConstructor.instantiateWrapperFor(VirtualBaseConstructor.java:102)
        at sun.jvm.hotspot.oops.Metadata.instantiateWrapperFor(Metadata.java:68)
        at sun.jvm.hotspot.oops.Oop.getKlassForOopHandle(Oop.java:211)
        at sun.jvm.hotspot.oops.ObjectHeap.newOop(ObjectHeap.java:251)
        at sun.jvm.hotspot.memory.StringTable.stringsDo(StringTable.java:72)
        at sun.jvm.hotspot.tools.HeapSummary.printInternStringStatistics(HeapSummary.java:299)
        at sun.jvm.hotspot.tools.HeapSummary.run(HeapSummary.java:148)
        at sun.jvm.hotspot.tools.Tool.startInternal(Tool.java:260)
        at sun.jvm.hotspot.tools.Tool.start(Tool.java:223)
        at sun.jvm.hotspot.tools.Tool.execute(Tool.java:118)
        at sun.jvm.hotspot.tools.HeapSummary.main(HeapSummary.java:49)
```

出现该问题的原因是对于`64`位的虚拟机当内存使用量超过`2G`的量内存计算工具已经不支持了，解决的版本也很简单在执行命令时加上`-J-d64`即可，例如`jmap  -J64 -heap 2768`





下面我们就按照[官方的分类方法](https://docs.oracle.com/javase/8/docs/technotes/tools/)介绍各个工具




## 二、 监控工具类(Monitoring Tools)

### 2.1、JPS本地已经启动的JVM实例

使用该命令可以查看到当前环境下所有正在运行的`JVM`实例，使用方法如下：

```bash
Shell> jps -lv
502 sun.tools.jps.Jps -Dapplication.home=/opt/jdk -Xms8m
127 org.tanukisoftware.wrapper.WrapperStartStopApp -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djdk.tls.ephemeralDHKeySize=2048 -Dcatalina.base=../ -Dcatalina.home=../ -XX:MaxPermSize=512m -Duser.timezone=Asia/Shanghai -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=../logs -Dcom.sun.management.jmxremote.port=8888 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=192.168.149.150 -Xms3m -Xmx2048m -Djava.library.path=../lib -Dwrapper.key=9hgdS-D-T-c9hf8O -Dwrapper.port=32000 -Dwrapper.jvm.port.min=31000 -Dwrapper.jvm.port.max=31999 -Dwrapper.pid=125 -Dwrapper.version=3.5.30 -Dwrapper.native_library=wrapper -Dwrapper.arch=x86 -Dwrapper.cpu.timeout=10 -Dwrapper.jvmid=1

```

从上面的信息中可以看出，当前换进中有两个`JVM`实例，一个是`502号进程`，一个是`127号进程`，其中`502`是我们刚刚运行的`JPS`，`127号进程就是一般我们要调试的Tomcat进程`,从其中我们可以找到启动`JVM`时传入的参数信息。比如`-Duser.timezone=Asia/Shanghai`，如果没有这个参数代码中所有的`new Date()`出来的对象都在我们所在东八区时区内，算出来的时间都是少了8小时的。



### 2.2、jstat监控本地JVM运行时堆栈内存使用情况

该命令可以查看到正在运行状态下，JVM中`年轻代`、`幸存区`、`持久区`的使用状态，该命令笔者在实际的生产环境中也是用的不多，一般都是实在想不到原因点了去使用`-gc和-gcutil`两个`options`查看是否JVM在活动，尤其是当堆栈中出现大量的`mem lock时才回去看一下`说白了就算他不GC了，我也没有办法，以自己的能力并不能给出一个有始有终的结论出来，一般都会调转方向往其他的点上想如何优化，如下所示，命令行中的`1000`表示每隔`1秒`输出一次

```bash

Shell> jstat -gc 1224 1000
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
47104.0 48128.0 16440.7  0.0   261632.0 45708.0   98816.0    80185.0   87424.0 86301.9 11136.0 10933.4     88    1.243  10      1.015    2.258
47104.0 48128.0 16440.7  0.0   261632.0 45708.0   98816.0    80185.0   87424.0 86301.9 11136.0 10933.4     88    1.243  10      1.015    2.258
47104.0 48128.0 16440.7  0.0   261632.0 45708.0   98816.0    80185.0   87424.0 86301.9 11136.0 10933.4     88    1.243  10      1.015    2.258



Shell> jstat -gcutil 1224 1000
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
 34.90   0.00  13.86  81.15  98.72  98.18     88    1.243    10    1.015    2.258
 34.90   0.00  13.86  81.15  98.72  98.18     88    1.243    10    1.015    2.258
 34.90   0.00  13.86  81.15  98.72  98.18     88    1.243    10    1.015    2.258
 34.90   0.00  13.86  81.15  98.72  98.18     88    1.243    10    1.015    2.258
 34.90   0.00  13.86  81.15  98.72  98.18     88    1.243    10    1.015    2.258
 34.90   0.00  13.86  81.15  98.72  98.18     88    1.243    10    1.015    2.258
 34.90   0.00  13.86  81.15  98.72  98.18     88    1.243    10    1.015    2.258
 34.90   0.00  13.86  81.15  98.72  98.18     88    1.243    10    1.015    2.258
 34.90   0.00  13.86  81.15  98.72  98.18     88    1.243    10    1.015    2.258
 34.90   0.00  13.86  81.15  98.72  98.18     88    1.243    10    1.015    2.258

```

如上所示我们可以看见在内存中GC整理的`S0和S1`区域的内存变化，正常的情况下你应该可以看到`S0`和`S1`在不断的切换变换就说明JVM在不断的整理内存，上表格中的数值单位为(KB)，我们可以使用`-gc`查看到每一个区域的大小情况，使用`-gcutil`查看到每一个区域的占比(可能该视图更加的直观一点)

[官方文档地址-jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)

### 2.3、jstatd允许远程监控的一个补偿

该工具其实实现的功能和`jstat`类似，我个人觉得他其实是一个监控方法的补偿，因为他能做的事情命令行都能做。他实现的方法是在本地启动一个程序`attach`至本地的`JVM`中，然后在本地的端口上外部工具实现了`RMI`这一套接口的工具即可获取JVM运行时状态信息。使用方法如下：
首先在客户端的`${JDK_HOME}/bin`目录下放置一个安全策略文件，内容如下所示
```bash
grant codebase "file:${java.home}/../lib/tools.jar" {   
    permission java.security.AllPermission;
};

```
由上可以我们将所有的权限都授予出去了，所以杀伤力比较大，笔者曾经尝试过使用`JMX`的功能直接修改JVM的参数，并且成功了，所以该功能只有在紧急的时刻在去使用，但是其实出现问题的时候纳管得了那么多。配置完成后在`$JAVA_HOME/bin`目录下运行如下命令
```bash
Shell> jstatd -J-Djava.security.policy=jstatd.all.policy -p 7777
# 程序会夯住，在本地监听7777端口

```

成功后我们就可以在远程使用`jvisualvm.exe`工具连接进来。因为没有很好的实践经验，所以暂时对其不做深入的介绍。



## 三、 排错工具类(Troubleshooting Tools)

## 3.1、JINFO查看与设置本地JVM配置信息

该命令的主要作用是对线上的一些环境信息做在线调整，比如给正在运行中的`JVM`添加一些参数信息。例如我们可以通过`jinfo`命令查看指定进程编号的JVM的环境变量配置

```bash
Shell> jinfo 80
Attaching to process ID 80, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.65-b01
Java System Properties:

java.vendor = Oracle Corporation
sun.java.launcher = SUN_STANDARD
common.loader = ${catalina.base}/lib,${catalina.base}/lib/*.jar,${catalina.home}/lib,${catalina.home}/lib/*.jar
# ..........省略部分输出.............
wrapper.jvmid = 1
java.runtime.name = Java(TM) SE Runtime Environment
sun.java.command = org.tanukisoftware.wrapper.WrapperStartStopApp org.apache.catalina.startup.Bootstrap 1 start org.apache.catalina.startup.Bootstrap TRUE 1 stop
java.class.path = ../lib/wrapper.jar:../bin/bootstrap.jar:../bin/tomcat-juli.jar
java.vm.specification.name = Java Virtual Machine Specification
java.vm.specification.version = 1.8
catalina.home = /opt/apache-tomcat-7.0.82
sun.cpu.endian = little
sun.os.patch.level = unknown
wrapper.cpu.timeout = 10
java.io.tmpdir = /tmp
java.rmi.server.hostname = 192.168.149.150

VM Flags:
Non-default VM flags: -XX:CICompilerCount=3 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=null -XX:InitialHeapSize=4194304 -XX:+ManagementServer -XX:MaxHeapSize=2147483648 -XX:MaxNewSize=715653120 -XX:MinHeapDeltaBytes=524288 -XX:OldSize=2621440 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC 
Command line:  -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djdk.tls.ephemeralDHKeySize=2048 -Dcatalina.base=../ -Dcatalina.home=../ -XX:MaxPermSize=512m -Duser.timezone=Asia/Shanghai -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=../logs -Dcom.sun.management.jmxremote.port=8888 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=192.168.149.150 -Xms3m -Xmx2048m -Djava.library.path=../lib -Dwrapper.key=4fCqz4tnRqY4ajiY -Dwrapper.port=32000 -Dwrapper.jvm.port.min=31000 -Dwrapper.jvm.port.max=31999 -Dwrapper.pid=78 -Dwrapper.version=3.5.30 -Dwrapper.native_library=wrapper -Dwrapper.arch=x86 -Dwrapper.cpu.timeout=10 -Dwrapper.jvmid=1

```

如上所示就是启动当前JVM时所有的环境变量信息。当然我们也可以使用`System.getProperties()`在代码中获取到。当然按照官方文档中的说法，这个命令行工具还可以实现动态的设置`JVM`里面的参数，但是实际笔者在测试时发现该方法并不一定奏效，且官方也指明了在`JDK1.8`开始后续的版本逐渐的也开始将废弃该工具了，但是对于我们当前查看一些环境信息的还是很有帮助的。官方的说明如下:

> The jinfo command prints Java configuration information for a specified Java process or core file or a remote debug server. The configuration information includes Java system properties and Java Virtual Machine (JVM) command-line flags. If the specified process is running on a 64-bit JVM, then you might need to specify the -J-d64 option, for example: jinfo -J-d64 -sysprops pid.

> This utility is unsupported and might not be available in future releases of the JDK. In Windows Systems where dbgeng.dll is not present, Debugging Tools For Windows must be installed to have these tools working. The PATH environment variable should contain the location of the jvm.dll that is used by the target process or the location from which the crash dump file was produced. For example, set PATH=%JDK_HOME%\jre\bin\client;%PATH% . 

[官方文档地址-jinfo](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jinfo.html)

## 3.2、 jmap 对JVM内存分析的工具

该工具大家应该都有所见闻，该工具可以对JVM中所有的对象进行查看与分析。该工具支持工具方法如下，`jmap [option] <pid>`，其中`option`支持如下选项

- `-dump:` DUMP出当前JVM堆存储文件，一般我们在这里面找到内存占用比较大的对象，常用的完整的命令格式为`jmap -dump:format=b,file=./要保存的DUMP文件名     pid`
- `-heap:` 显示堆栈详细信息，使用的垃圾回收器的类型参数，内存分代使用情况
- `-clstats:` 以`classLoadder`的视角查看内存使用状态，没有实际的实践经验，所以不做深入介绍
- `finalizerinfo:` 查看正在等待执行`finalization`操作的对象信息，没有实际的实践经验，所以不做深入介绍

下面着重介绍一下笔者有实践经验的几个用法：

1. `DUMP出当前的内存:` 查看究竟是哪一个对象占用内存比较大，这种情况下一般是程序并未发生`OOM`报错，但是服务器资源紧张，JAVA进程占用了大量的内存。

```bash
Shell> jmap  -dump:format=b,file=./tomcat.hprof 1224
Dumping heap to /root/tomcat.hprof ...
Heap dump file created

```

运行如上命令即可在当前目录下创建生成一个`tomcat.hprof`文件，完成后我们会将`tomcat.hprof`文件取回来使用`MemoryAnalyzer`工具进行分析


2. `查看当前的堆使用状态:` 该方法一般线上紧急的时候查看一下内存，以快速定位的时候使用，在这个里面可以看到每一个内存块的使用状况以及堆区的配置信息等

```bash
Shell> jmap  -heap 1224
Attaching to process ID 1224, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.65-b01

using thread-local object allocation.
Parallel GC with 4 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 2147483648 (2048.0MB)
   NewSize                  = 1572864 (1.5MB)
   MaxNewSize               = 715653120 (682.5MB)
   OldSize                  = 2621440 (2.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 267911168 (255.5MB)
   used     = 97250856 (92.74564361572266MB)
   free     = 170660312 (162.75435638427734MB)
   36.2996648202437% used
From Space:
   capacity = 48234496 (46.0MB)
   used     = 16835248 (16.055343627929688MB)
   free     = 31399248 (29.944656372070312MB)
   34.90292093028193% used
To Space:
   capacity = 49283072 (47.0MB)
   used     = 0 (0.0MB)
   free     = 49283072 (47.0MB)
   0.0% used
PS Old Generation
   capacity = 101187584 (96.5MB)
   used     = 82109392 (78.30561828613281MB)
   free     = 19078192 (18.194381713867188MB)
   81.14571843122572% used

34261 interned Strings occupying 3735448 bytes.
```
[官方文档-jmap](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jmap.html)



### 3.3、jstack 显示当前JVM运行的堆栈信息

该工具可以让我们查询到当前`JVM`中正在运行的线程堆栈信息，是一个瞬时值，也就是说你每次去执行结果都应该是不一样的，当然如果你的线程状态都是`Blocked`状态的那么每次都是一样的，我们很多时候`Tomcat`请求访问卡主了等等这类问题其实都是需要使用到这个工具去查看究竟是为何卡主了。大量的线程卡在哪里了，使用方法很简单

```bash

## .....省略部分输出........

Shell> jstack 1224 
"http-nio-8080-Acceptor-0" #121 daemon prio=5 os_prio=0 tid=0x00007f9f94258000 nid=0x544 runnable [0x00007f9f5d489000]
   java.lang.Thread.State: RUNNABLE
	at sun.nio.ch.ServerSocketChannelImpl.accept0(Native Method)
	at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:422)
	at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:250)
	- locked <0x0000000080357cf8> (a java.lang.Object)
	at org.apache.tomcat.util.net.NioEndpoint$Acceptor.run(NioEndpoint.java:827)
	at java.lang.Thread.run(Thread.java:745)

"http-nio-8080-ClientPoller-1" #120 daemon prio=5 os_prio=0 tid=0x00007f9f94255800 nid=0x543 runnable [0x00007f9f5d58a000]
   java.lang.Thread.State: RUNNABLE
	at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
	at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
	at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:79)
	at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
	- locked <0x00000000e8533b20> (a sun.nio.ch.Util$2)
	- locked <0x00000000e8533b10> (a java.util.Collections$UnmodifiableSet)
	- locked <0x00000000e85339d8> (a sun.nio.ch.EPollSelectorImpl)
	at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
	at org.apache.tomcat.util.net.NioEndpoint$Poller.run(NioEndpoint.java:1213)
	at java.lang.Thread.run(Thread.java:745)

"http-nio-8080-ClientPoller-0" #119 daemon prio=5 os_prio=0 tid=0x00007f9f94255000 nid=0x542 runnable [0x00007f9f5d68b000]
   java.lang.Thread.State: RUNNABLE
	at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
	at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
	at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:79)
	at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
	- locked <0x00000000e8533ee8> (a sun.nio.ch.Util$2)
	- locked <0x00000000e8533ed8> (a java.util.Collections$UnmodifiableSet)
	- locked <0x00000000e8533da0> (a sun.nio.ch.EPollSelectorImpl)
	at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
	at org.apache.tomcat.util.net.NioEndpoint$Poller.run(NioEndpoint.java:1213)
	at java.lang.Thread.run(Thread.java:745)

"http-nio-8080-exec-64" #118 daemon prio=5 os_prio=0 tid=0x00007f9f94239000 nid=0x541 waiting on condition [0x00007f9f5d78c000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000e850f538> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:104)
	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:32)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:745)


## .....省略部分输出........

```

如上所示，我们可以看到`Tomcat`当前状态下`线程状态`，我们需要着重注意的是`java.lang.Thread.State`值，如果是`RUNNABLE`的则表示当前的线程正在运行,如果是`WAITING`状态的则说明是此线程此时处于挂起状态，我们就需要结合堆栈中的代码去分析里面的问题。线程所有可能所在的状态其实在[`JDK文档中包含定义`](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.State.html)

总结笔者现有的遇见过的情况，`java.lang.Thread.State`一般会有如下几种情况以及产生的原因，由于涉及到公司的代码安全，所以只能给出部分堆栈

- `NEW:` 该状态很快就是转变掉，笔者重来就没有在快照中见过该状态的线程

- `BLOCKED:` 阻塞状态，当线程处于该状态时通常是由于两个或者多个线程之间相互挣用一个所资源，A先得到了就可以运行，B没有得到就会进入`BLOCKED`状态，在堆栈中可以看到线程`locked at 某一个地址`我们在堆栈快照中查找到该值，当然目前以笔者的时间来看大多数情形下依靠经验去排查的比较多一点

- `WAITING`  等待状态，此时我们需要结合堆栈查看是否是线程自己调用的`wait`方法导致的，还是别的线程调用了它的`wait`方法

- `TIMED_WAITING` 和上面的差不多，但是这种等待状态是有时间限制的。时间到了就继续获得CPU的执行时间执行了。

- `RUNNABLE` 运行状态


[官方文档-jstack](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstack.html)

[其他参考资料1 ](https://fangjian0423.github.io/2016/06/04/java-thread-state/)


