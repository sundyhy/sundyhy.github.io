---
layout:     post
title:      句柄泄露调查
subtitle:   Android端
date:       2018-02-08
author:     DEH000
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Android
    - 句柄
    - 泄露
    - 内存泄露
---

>Android端句柄泄露调查


## 为终端添加一个快捷键打开方式

## 句柄

句柄(file descriptor)即文件描述符，具体解释详见[File descriptor](https://en.wikipedia.org/wiki/File_descriptor)解释，以下简称fd。在android系统中，每个进程最多可以使用1024个fd, 任何一个IO操作都会使用一个fd，比如socket, open file , pipe等等。


## 句柄泄露

句柄泄露是指程序中已分配的fd由于某种原因未释放或者无法释放，造成fd句柄占用越来越多，当达到上限1024后，程序将发生崩溃。常见的句柄泄露崩溃可能有如下错误日志：

- Could not read input channel file descriptors from parcel 	
- Too many open files 	
- file descriptor >= FD_SETSIZE

## 句柄泄漏调查

在我们android YY中，近期发现句柄泄露的问题较严重，故专门针对句柄泄露做了个调查，下面分享一下调查过程。

### 1 准备设备

调查中部分操作需要使用到root权限，故需要准备一台root过的手机，如果没有root的手机，推荐使用多玩模拟器代替。

### 2 确定问题

针对怀疑的操作过程，反复操作，确定句柄数是否上涨，在android系统中，/proc/pid/为当前进程的一些状态描述信息，/proc/pid/fd下面为句柄信息，一般可通过两种方式查看：

- 在adb shell中，可通过ls -all /proc/{pid}/fd查看句柄信息，执行该操作需要root后的手机
- 对于未root的手机，可在程序中读取/proc/self/fd来查看句柄信息，下面是一段在程序中读取并保存为文件的代码

		FileOutputStream os = new FileOutputStream(saveFile);
        File fddir = new File("/proc/self/fd");
        int count = 0;
        for (File ff : fddir.listFiles()) {
            Date lastModified = new Date(ff.lastModified());
            String line = String.format("%s -> %s -> %s\n", ff.getName(), lastModified.toString(), ff.getCanonicalPath());
            os.write(line.getBytes());
            count += 1;
        }
        os.close();
       

首先保存开始测试时当前的句柄信息到文件，然后开始测试，等测试结束时，再保存对应的句柄信息到文件，对比两个文件的差别，查看增加是否明显，有明显增加，表明存在句柄泄漏。

### 3 调查泄漏的句柄类型

通过步骤2，可以得到增加的句柄信息，常见的内容如下：

    		28 -> /system/framework/smartfaceservice.jar
			280 -> pipe:[572698]
			283 -> anon_inode:dm
			67 -> /storage/emulated/0/yymobile/logs/sdklog/pushsvc_log.txt
			84 -> /data/data/com.duowan.mobile/databases/accs.db
			96 -> socket:[566584]
			98 -> pipe:[563797]
			99 -> anon_inode:[eventpoll]

针对上面的主要类型做个简单说明：

-  28、67、84为打开还未关闭的文件句柄，创建方式为下层统一调用libc的open、create及opendir。
-  280 98 pipe为管道句柄，创建方式为下层统一调用libc的pipe函数
-  99 anon_inode:[eventpoll]通常为epoll或select创建，创建方式为下层统一调用libc的pipe函数
-  96 socket为网络连接使用，创建方式下层统一调用libc的socket和socketpair

对比测试发现，我们APP在使用中这三类句柄（socket, pipe , anon_inode）同步增加较多。


### 4 查找泄漏的代码位置

我们知道，存在泄漏是因为打开了未关闭导致一直增加，那我们要查找具体的代码位置，可以监控每次句柄的创建跟关闭，最后查看剩下的是什么地方创建的即可。在这里可以一类一类追踪，比如针对socket句柄，可以监控socket及socketpair函数来调查、针对pipe可以监控pipe函数来调查，所有句柄关闭均调用libc的close函数，统一监控。

我们调查中统一使用hook 对应函数的方式进行监控，[这里](http://code.yy.com/dw_zhangdeheng/elfhook)为我自己实现的一个小的hook库，无依赖，直接加载so即可，主要函数为：

	elfHook(const char* soName, const char* symbol, void* newFunc, void** oldFunc);	
	


这里我们最先调查的是pipe的增加，通过上面的方式，在对应的函数里面打印调用堆栈，通过测试完成后，发现增加的pipe均是由下面的堆栈创建：

	12-28 17:24:28.005 16150 16813 W System.err: java.lang.Throwable: YY_IN_LIKEVIEW
	12-28 17:24:28.005 16150 16813 W System.err: 	at android.os.MessageQueue.nativeInit(Native Method)
	12-28 17:24:28.015 16150 16813 W System.err: 	at android.os.MessageQueue.<init>(MessageQueue.java)
	12-28 17:24:28.015 16150 16813 W System.err: 	at android.os.Looper.<init>(Looper.java)
	12-28 17:24:28.015 16150 16813 W System.err: 	at android.os.Looper.prepare(Looper.java)
	12-28 17:24:28.015 16150 16813 W System.err: 	at android.os.Looper.prepare(Looper.java)
	12-28 17:24:28.015 16150 16813 W System.err: 	at android.os.HandlerThread.run(HandlerThread.java)
	12-28 17:24:28.015 16150 16813 W DH_ANDROID_UTIL: YYAPPHOOK user call pipe[300, 302]

可以看出，泄漏的pipe是由Looper使用，它是在Looper.prepare的时候创建，这里Throwable里面的字符串为我们自己输出的线程名，可以看出线程为YY_IN_LIKEVIEW，由线程名可以定位到具体代码为小视频LinkView的DrawThread。

### 5 分析泄漏原因

查看DrawThread代码， DrawThread为一个HandlerThread， 初步调查并无异常，并且每次退出有调用quit，进一步分析DrawThread，会发现存在一个可疑对象ValueAnimator， ValueAnimator一般为主线程使用，而这里在非主线程使用是否会存在问题，于是测试去掉ValueAnimator，发现句柄不再泄漏，那可以确定就是ValueAnimator的问题。

接下来查看ValueAnimator源码：
			
	public class ValueAnimator extends Animator {
	
		// The static sAnimationHandler processes the internal timing loop on which all animations
    	// are based
		/**
		* @hide
		*/
		protected static ThreadLocal<AnimationHandler> sAnimationHandler = new ThreadLocal<AnimationHandler>();

再看AnimationHandler如下：
			
	protected static class AnimationHandler implements Runnable {

		private final Choreographer mChoreographer;

		private AnimationHandler() {
			mChoreographer = Choreographer.getInstance();
		}


Choreographer如下：
			
	public final class Choreographer {

		private final Looper mLooper;
			
		// Thread local storage for the choreographer.
		private static final ThreadLocal<Choreographer> sThreadInstance =
			new ThreadLocal<Choreographer>() {
				@Override
				protected Choreographer initialValue() {
					Looper looper = Looper.myLooper();
					if (looper == null) {
						throw new IllegalStateException("The current thread must have a looper!");
					}
					return new Choreographer(looper);
				}
		};


		/**
		* Gets the choreographer for the calling thread.  Must be called from
		* a thread that already has a {@link android.os.Looper} associated with it.
		*
		* @return The choreographer for this thread.
		* @throws IllegalStateException if the thread does not have a looper.
		*/
		public static Choreographer getInstance() {
			return sThreadInstance.get();
		}

		
到这里就清晰了，我们知道，ThreadLocal维护变量时，为每个使用该变量的线程提供独立的变量副本，因此，当前调用线程的looper都会被一个静态变量引用，并且一直不会释放，导致了looper不会被回收，进而导致了相关资源的泄漏。 

	注：这里解释一下Looper跟上面泄漏的三类句柄（socket, pipe , anon_inode）的关系：Looper在创建消息循环时，会使用pipe, socketpair创建流管道，anon_inode为做消息监听，因此会创建对应句柄。

	
### 6 修复问题：

从五可以知道ValueAnimator都会导致对应的线程looper被保存，而只有主线程的looper是不会退出的，因此，ValueAnimator只能在主线程调用，相关操作全部改为主线程执行即可。



  
-------------------
最后说一点，大家在使用系统api时，都应该先看一遍他整个类及函数的说明，均应该严格按照说明。在ValueAnimator的介绍中，多处写到了在ui thread执行。自行调用，虽然可能看到功能正确，但android系统繁多，难保所有系统都能正常使用；并且在一些复杂场景下，也难保不出问题，这些测试也是没办法全部覆盖到的。
