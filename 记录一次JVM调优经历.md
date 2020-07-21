今天把公司代码拉下来，导入idea,gradle build之后就报错

````java
java.lang.OutOfMemoryError: GC Overhead Limit Exceeded
````

首先看下gradle.properties配置文件

 1，jstat -gc -进程号看下内存情况（实际上这种大概率是-Xms与-Xmx配置小了），发现堆内存配置512m，对于三十多个微服务来说，堆内存配置比较小，于是增加-Xmx与-Xms

````java
org.gradle.jvmargs=-Xmx1024m -Xms1024m 
````

2，再次运行，这次不再报错GC错误，但是报了OOM，为了便于排查，于是增加配置已打印报错信息

> 此过程也可以用jmap命令实现，jhat 命令是与jmap搭配使用，用来分析jmap生成的dump，jhat内置了一个微型的HTTP/HTML服务器，生成dump的分析结果后，可以在浏览器中查看 如：jhat -J-Xmx512m dump.hprof

```java
org.gradle.jvmargs=-Xmx1024m -Xms1024m  -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8
```

3，分析log文件，可以发现看到young区使用率100%，触发YGC，eden区清除，存活的送去Survivor （s0,s1交换清除，未使用的被清除，可以看到使用率不停的从99%调换），大于Survivor 区内存，直接送去老年代；oldgc达到67%，导致了OOM,于是配置MaxMetaspaceSize大小为256m（jdk8之前区别于永久代Perm， jdk8之后叫元空间Metaspace ，在本地内存中分配。在 JDK8 里， Perm 区中的所有内容 中字符串常量移至堆内存，其他内容包括类元信息、字段、静态属性、方法、常量等 都移动至无空间内）

```java
Event: 129.515 GC heap before
{Heap before GC invocations=57 (full 4):
 PSYoungGen      total 229888K, used 223591K [0x00000000eab00000, 0x0000000100000000, 0x0000000100000000)
  eden space 116736K, 100% used [0x00000000eab00000,0x00000000f1d00000,0x00000000f1d00000)
  from space 113152K, 94% used [0x00000000f9180000,0x00000000ff9d9dd0,0x0000000100000000)
  to   space 116224K, 0% used [0x00000000f1d00000,0x00000000f1d00000,0x00000000f8e80000)
 ParOldGen       total 699392K, used 450027K [0x00000000c0000000, 0x00000000eab00000, 0x00000000eab00000)
  object space 699392K, 64% used [0x00000000c0000000,0x00000000db77af40,0x00000000eab00000)
 Metaspace       used 71931K, capacity 102552K, committed 102720K, reserved 1130496K
  class space    used 11666K, capacity 22049K, committed 22080K, reserved 1048576K
Event: 129.630 GC heap after
Heap after GC invocations=57 (full 4):
 PSYoungGen      total 232960K, used 116199K [0x00000000eab00000, 0x0000000100000000, 0x0000000100000000)
  eden space 116736K, 0% used [0x00000000eab00000,0x00000000eab00000,0x00000000f1d00000)
  from space 116224K, 99% used [0x00000000f1d00000,0x00000000f8e79dc0,0x00000000f8e80000)
  to   space 116224K, 0% used [0x00000000f8e80000,0x00000000f8e80000,0x0000000100000000)
 ParOldGen       total 699392K, used 472516K [0x00000000c0000000, 0x00000000eab00000, 0x00000000eab00000)
  object space 699392K, 67% used [0x00000000c0000000,0x00000000dcd711c0,0x00000000eab00000)
 Metaspace       used 71931K, capacity 102552K, committed 102720K, reserved 1130496K
  class space    used 11666K, capacity 22049K, committed 22080K, reserved 1048576K
}
```

关于元空间：

> jdk8之前区别于永久代Perm， jdk8之后叫元空间Metaspace ，在本地内存中分配。在 JDK8 里， Perm 区中的所有内容 中字符串常量移至堆内存，其他内容包括类元信息、字段、静态属性、方法、常量等 都移动至无空间内
>
> 1. 如果没有配置`-XX:MetaspaceSize`，那么触发FGC的阈值是21807104（约20.8m），可以通过jinfo -flag MetaspaceSize pid得到这个值；
>
> 2. 如果配置了`-XX:MetaspaceSize`，那么触发FGC的阈值就是配置的值；
>
> 3. Metaspace由于使用不断扩容到`-XX:MetaspaceSize`参数指定的量，就会发生FGC；且之后每次Metaspace扩容都会发生FGC；
>
> 4. 如果Old区配置CMS垃圾回收，那么扩容引起的FGC也会使用CMS算法进行回收；
>
> 5. 如果MaxMetaspaceSize设置太小，可能会导致频繁FGC，甚至OOM
>
> 6. jstat -gcutil pid 可以看到Meta空间的使用率
>
>    建议：
>
>    1. `MetaspaceSize`和`MaxMetaspaceSize`设置一样大；
>    2. 具体设置多大，建议稳定运行一段时间后通过`jstat -gc pid`确认且这个值大一些，对于大部分项目256m即可

4，所以最终配置为

````java
org.gradle.jvmargs=-Xmx1024m -Xms1024m -XX:MaxMetaspaceSize=256m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8

````

顺便记录下其他几个简单的命令：

jps jstat jmap jhat jstack jinfo

* jps JVM Process Status Tool,显示指定系统内所有的HotSpot虚拟机进程。 如： jps -l -m
* jstat 是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。 如: jstat -gc 11589 --可以查看11589进程的内存使用情况
* jmap 命令用于生成heap dump文件，如果不使用这个命令，还阔以使用-XX:+HeapDumpOnOutOfMemoryError参数来让虚拟机出现OOM的时候·自动生成dump文件 如： jmap -heap 28920
* jhat 命令是与jmap搭配使用，用来分析jmap生成的dump，jhat内置了一个微型的HTTP/HTML服务器，生成dump的分析结果后，可以在浏览器中查看 如：jhat -J-Xmx512m dump.hprof
* jstack jstack用于生成java虚拟机当前时刻的线程快照 如：jstack -l 11494|more
* jinfo jinfo(JVM Configuration info)这个命令作用是实时查看和调整虚拟机运行参数