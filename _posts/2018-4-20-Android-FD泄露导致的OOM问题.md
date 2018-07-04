---
layout: post
title: Android-FD泄露导致的OOM问题
categories: [Android]
description: 
tags: OOM
---

最近遇到一个线上问题，因为业务需求需要文件下载功能，因此接入了Github上一个比较流行的下载库FileDownloader。

https://github.com/lingochamp/FileDownloader

但是在线上却出现了oom的问题

```

[error:java.io.IOException: Cannot run program "logcat": error=24, Too many open files]

FileDownloader-Network44(702)

java.lang.OutOfMemoryError

Could not allocate JNI Env
java.lang.Thread.nativeCreate(Native Method)
java.lang.Thread.start(Thread.java:1063)
java.util.concurrent.ThreadPoolExecutor.addWorker(ThreadPoolExecutor.java:921)
4 java.util.concurrent.ThreadPoolExecutor.processWorkerExit(ThreadPoolExecutor.java:989)
5 java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1131)
6 java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
7 java.lang.Thread.run(Thread.java:818)
```

一般创建线程的时候出现OOM的问题，可以有以下的情况：

```
1）内存堆栈满了，没有多余的内存空间

2）进程创建的线程数量超过了进程的最大限制数量

3）进程的文件操作符数量超过了进程的最大限制数量
```

从崩溃的日志上看，这个问题是由于进程文件操作符超过了进程的最大限制造成的。


## 文件描述符

FD(File Descriptor)文件描述符作为一个索引值，用于指向进程内的打开文件。当我们在进程中，打开文件，打开网络流（socket），管道或者其他资源，都会生成文件描述符。然后每个进程中这个值都是有限制的，一般情况下为1024。

我们可以通过命令行来查看指定进程的文件描述符数量限制

>* 首先切换到adb的shell，然后再获取指定的PID

```
$ adb shell  //切换到shell环境

$ su         //切换到超级管理员身份

# ps | grep "xxx.xxx.xxx"
//查找我们想要查看的进程的PID
u0_a211   15429 100   902   1270960 62088 SyS_epoll_ 00f24fff30 S xxx.xxx.xxx
```

>* cat /proc/15429/limits

```
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             13419                13419                processes 
Max open files            1024                 4096                 files     
Max locked memory         67108864             67108864             bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       13419                13419                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         40                   40                   
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us
```

我们看到进程内的文件描述符数量是有限制的，如果超过了这个数量，就会出现OOM问题。


## 验证问题


我们在应用的基础上，调用文件下载库，进行串行下载实验。

切换到FileDownloader进程的FD目录下：

```
cd /proc/15429/fd
```

然后在查看当前FD目录下文件描述符数量：

```
ls -l | wc -l

94
```

可以看到目前该进程下是有94个文件描述符.

现在我们开始发起下载任务去下载一个MP3音频文件，然后在下载完毕之后再次查看FD目录.

```
ls -l | wc -l

99
```
可以看到，下载完之后，文件描述符并没有被释放，说明这时候出现了FD泄露。

那么，是什么操作导致文件描述符泄露，我们可以先查看FD目录下，哪些文件描述符没有被释放。

```
ls -l

lrwx------ 1 u0_a211 u0_a211 64 2018-04-20 17:36 0 -> /dev/null
lrwx------ 1 u0_a211 u0_a211 64 2018-04-20 17:36 1 -> /dev/null
lr-x------ 1 u0_a211 u0_a211 64 2018-04-20 17:36 10 -> /system/framework/core-oj
.jar
...//此处省略若干个文件描述符
lrwx------ 1 u0_a211 u0_a211 64 2018-04-20 17:36 100 -> /storage/emulated/0/GGMusic/Myudio/15241341_AAdAFrYcQGID2I0AABOcDTGpbMAAFvOAP_sUwAAE60322.mp3

```

可以看到音频文件相关的文件描述符没有被释放，可以推测，是在写入文件的时候，没有对相关的文件流进行关闭回收操作。


我们看到FileDownloader的源码：


FileDownloadRandomAccessFile.java

```
    FileDownloadRandomAccessFile(File file) throws IOException {
        randomAccess = new RandomAccessFile(file, "rw");
        fd = randomAccess.getFD();
        out = new BufferedOutputStream(new FileOutputStream(randomAccess.getFD()));
    }
```

FileDownloadRandomAccessFile封装了FDw文件类以及文件写入流，在构造方法体内对这些变量进行初始化。


```
    @Override
    public void close() throws IOException {
        out.close();
    }
```

但是在FileDownloadAccessFile#close()方法内，却只对了BufferedOutputStream调用close回收操作，而没有对RandomAccessFile调用close方法对回收操作。这里造成了Fd文件描述符的泄露。


## 为什么出现的概率低

这个问题我已经给作者提交了PR，普通测试的情况下，这个case也是很难重现的。

1）一般场景不会触发那么多的下载需求，因为文件描述符的限制是1024，而每次进程重启，FD目录都会清空。

2）另一个原因是当一段时间不使用之后，FD资源在java的gc系统中会被释放掉。

我们可以看到java在IO流中的代码：

```
 /**
     * 该段代码摘自FileInputStream源码，jdk版本1.8
     */
    protected void finalize() throws IOException {
        if ((fd != null) &&  (fd != FileDescriptor.in)) {
            /* if fd is shared, the references in FileDescriptor
             * will ensure that finalizer is only called when
             * safe to do so. All references using the fd have
             * become unreachable. We can call close()
             */
            close();
        }
    }
```

因此，综上，出现开篇提到的oom问题，是用户在短时间内大量触发有FD泄露的线程导致的。

