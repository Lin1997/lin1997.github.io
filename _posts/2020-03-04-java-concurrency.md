---
title: Java 并发底层源码分析
tags:
    - Java
    - Concurrent
    - 并发
    - 源码
---

**本文转载至：[Java并发之基石篇](https://createchance.github.io/post/java-并发之基石篇)**（有改动）

> 作者：createchance
邮箱：createchance@163.com
日期：2019.07.28
版本：v1.0
[![知识共享许可协议](https://i.creativecommons.org/l/by-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-sa/4.0/)
本作品采用[知识共享署名-相同方式共享 4.0 国际许可协议](http://creativecommons.org/licenses/by-sa/4.0/)进行许可。

## 导读

声明：本文所有的分析内容基于 OpenDK 的 Java 11 版本的 HotSpot JVM 源代码。

在阅读本文之前，你需要：

1. 了解 Java 中的基本的线程使用方式以及注意点
2. 了解 Java 中的基本线程间通讯的方式
3. 了解 Java 中的 volatile 的基本语义
4. 了解 C/C++ 编程
5. 了解 JNI 的相关开发知识
6. 了解一些 x86 的汇编（仅仅是很简单的内容，要求能读懂）

本文重点分析内容：

1. 共享内存多核系统基本架构与设计
2. Java 内存模型设计
3. Java Thread 的创建与停止
4. Java synchonized 的实现机制
5. Java Object 的 wait 和 notify/notifyAll 实现机制
6. Java volatile 关键字的实现方式

阅读建议：

1. 下载一份 HotSpot JVM 11 的代码，了解基本代码架构，并且本地能够编译和调试，如何编译或者调试可以参考 OpenJDK wiki 或者 周志明的《深入理解 Java 虚拟机：第二版》一书
2. 使用你熟悉的 IDE 导入 JVM 的源码，建议使用 eclipse CDT，其他的 IDE 笔者都觉得不是很方便
3. 在阅读的过程中，不能仅仅看本文的分析，需要结合起来，自己下断点或者打印日志调试，并且不断反复尝试，反复理解核心代码片段
4. 最后一点建议，现代 JVM 的实现非常复杂，会涉及到很多的操作系统、算法、硬件、统设计以及性能调优等等方便的知识，所以在阅读源码的时候千万不要深入无关代码的细节部分，否则你会陷于无边无际的代码海洋而无法自拔～

本文非常长，笔者写了一个多星期，纯属手打，文中的观点和分析均是本人经过反复的试验和分析得出的，同时借鉴了很多网络上的文章，这些借鉴的内容已经放到文末的参考资料中，在此非常感谢这些作者的分享。真心希望你可以认真读完，我敢保证，只要你认真读完，肯定收获很多，加油～

## 概要

并行是这个时代的主旋律，也是很多现代操作系统需要提供的必备功能。在过去摩尔定律催生下，单个 CPU 核心计算的速度越来越快。但是随着产业的发展，单个CPU核心的计算上限已经难以突破，传统的加强单核的思维模式已经不能满足需求。在古代，人们需要强大的战马驱动战车，为了能够使得战斗力越来越强，人们驯化出了越来越强劲的战马，但是单匹马的力量始终是有限的，因此人们发明了多马并驾的战车结构，大量出现的多乘战车产催生了强大的万乘之国。同样地，在现代计算机领域，人们在单个 CPU 核心能力有限的情况下，使用多个核心的 CPU 进行并行计算，以驱动强大的算力。

但是，多 CPU 和多战马是远远不同的，在现实世界中的计算任务大多需要互相协调，其根本原因是人类的思维方式就是线性串行的，设计一个完全并行的计算逻辑体系还是有相当大难度的。

如何设计一个高并发的程序，不仅仅是工程界的难题，在计算机学术界也是一个需要不断突破的研究领域。从学术理论提出，到算法设计，再到工程实施，再到产业验证调优，整个流程都需要比较长的时间来进行迭代，究其根本，并行计算本身就是非常复杂、不确定的、不可预测的逻辑系统。

## 多核系统中的一致性

号称一次编写，到处运行的 Java，其本身也是构建在不同的系统之上的，以其运行时 JVM 来屏蔽系统底层的差异。因此，在介绍 Java 并发体系之间，有必要简要介绍下计算机系统层面上的并发，以及面对的问题。

我们的目的其实很简单，就是让计算机在同一时刻，能够运行更多的任务。而并行计算，提供了非常不错的解决方案。虽然这看起来很自然，但实际上面临着众多的问题，其中一个重大的问题就是绝大多数的计算不仅仅是 CPU 自己的事，而是需要很多计算机系统部件共同参与。但是我们知道，计算机系统中运行速度最快就是 CPU，其他部件例如：内存、磁盘、网络等等都是及其缓慢的，同时这些操作在目前的计算机体系中是很难消除的，因为我们不可能仅仅靠寄存器就完成所有的计算任务。面对高速 CPU 和低速存储之间的鸿沟，如果想要实现高效数据通讯，一个良好的解决方案就是在它们之间添加一个 cache 层，这个 cache 层的速度和整体的速度关系如下：

`CPU --> cache --> 存储`

通过 cache 这个缓冲地带，实现 CPU 和存储之间的高效「对话」。这是计算机和软件领域通用的一个问题解决方案：增加中间层。没有什么问题是一个中间层解决不了的，如果有，那就两层。在运算的时候，CPU 将需要使用到的数据复制到 Cache 中，以后每次获取数据都从较为快速的 cache 中获取，加快访问速度。

所谓理想很丰满，现实很骨感。这种计算体系有一个重要的问题需要解决，那就是：缓存一致性（cache coherence）问题。在现代的计算机系统中，主要都是多核系统为主。在这些计算机系统中，每一个 CPU 都拥有自己独立的高速缓存，但是因为主存只有一个，因此它们之间只能共享，这种系统也称为：共享内存多核系统（Shared-Memory multiprocessors System），如下图所示：

![java_concurrency_share_memory_system](/assets/posts/java_concurrency_share_memory_system.png)

因此，当多个处理器同时需要访问同一个内存区域的数据时，首先回去访问 CPU 的 cache 区域中的数据，但是 cache 中的数据也是从共享内存中获取的，此时如果别的 CPU 修改了 cache 中的数据，那么就造成了数据不一致的问题了。因此，如果发生了这种「数据竞态」的问题，到底该以哪个数据为准呢？此时，我们需要一个一致性协议来保证。各个 CPU 在操作的时候都需要遵守缓存一致性协议来进行操作，这类型的协议有很多，例如：MSI、MESI、MOSI、Synapse、Firefly 以及 Dragon Protocol 等等。所以，通常情况下，共享内存多核系统的架构如下所示：

![共享内存多核系统架构](/assets/posts/shared_memory_multi-core_system_architecture.png)

除了使用高速 cache 来缓和 CPU 和存储设备之间的速度鸿沟，为了能够充分利用多核 CPU 的处理性能，处理在实际执行机器指令时并不一定会按照程序设定的指令顺序执行，可能存在代码乱序执行（Out-Of-Order Execution）优化。注意，这里虽然乱序执行了，但是系统会保证执行的结果逻辑上的正确的，从宏观上看就好像是顺序执行一样。举个例子，比如我们有如下代码：

```java
int a = value1;
int b = value2;
```

这两句话实际的执行顺序可能是先赋值 a 然后赋值 b，但是也可能反过来，反正这两句话执行完毕之后 a 和 b 的值都被赋值上了就可以，这里对外表现为顺序串行执行，这其实就是 as-serial 协议保证的。为什么需要这样？一方面这两句话本身并没有什么逻辑上的依赖性，完全可以并行执行；另一方面，如果我们傻傻地按照顺序执行的话，在执行第一句话的时候，我们可能需要从主存中读取 value1 的值，这种操作对于 CPU 来讲是及其缓慢的操作，如果我们顺序执行的话，那么就只能等待 value1 值读取成功之后才能继续执行下面的指令，这样就造成了 CPU 的空等待，白白浪费了资源。

## Java 内存模型

上面我们探讨了共享内存多核系统的内存模型，我们提到了高速缓存以及缓存一致性问题，同时还介绍了指令乱序执行的问题。其实，这些概念在 Java 中也是存在的。因为 Java 的目标是：一次编写，到处运行。每一个计算机系统或者操作系统都会有自己特殊的内存模型，如果 Java 想要实现一次编写到处运行的目标，就必须在 JVM 层面上将系统之间的差异屏蔽掉。面对如此多的系统，最好的方式就是定义一套 Java 自己的内存访问模型，然后在不同的硬件平台和操作系统上分别利用本地接口来实现。这里的思想其实和增加 cache 是一样的，通过增加中间层来解决系统差异带来的协作问题。

Java 在 1.5 版本中引入了 [JSR 133 标准](https://jcp.org/en/jsr/detail?id=133)，这个标准提出了 Java 中的并发内存模型和线程规范，这个标准的发布标志着 Java 拥有独立于系统平台的并发内存模型。和 C/C++不同的是，Java 并没有直接操作系统平台中的内存模型，而是自己定义了一套机制，这套机制包含了并发访问机制、缓存一致性协议以及指令重排序解决方案等内容。在 JSR 133 标准中，定义了如下的 Java 并发内存模型：

![java_concurrency_jar](https://raw.githubusercontent.com/CreateChance/MyBlog/master/images/javaconcurrency/base/java_concurrency_jar.jpg)

可以看到，这里的内存模型和上面讲到的计算机系统中的内存模型是十分类似的。在 JVM 中，并发的最小单位是 Thread，在不考虑 JVM 线程实现细节上，可以简单认为一个 Thread 对应一个内核线程，这样就可以进而认为 Thread 对应一个 CPU 核心。这里需要注意的是，工作内存和 Java 内存区域中的堆、栈或者方法区（java 和 native）等并不是一个层面上的东西，它们之间也没有直接的对应关系。同时，很多人会误以为这里的工作内存其实就是 TLAB （thread local allocation buffers），字面上看起来很像，但是没有任何关系的。

从上面的图中，可以看出每个线程的工作内存和主存之间的一致性保证是通过 save 和 load 等等一系列的操作完成的。JSR 133 早期版本中定义了 8 种操作（早期版本的描述可以参考：[Thread and Locks](https://docs.oracle.com/javase/specs/jvms/se6/html/Threads.doc.html)），但是后来处于描述简化以及方便不同 JVM 实现修为了 4 种操作，但是只是描述上的修改，内存模型基本设计并没有改变（周志明的书中有描述，感谢这本书的指点）。这里我们采用最新版本的 4 种操作来描述，这种方式比较清晰易懂。这 4 种操作和 Java 内存模型的对应关系如下图：

![java_concurrency_read_write](/assets/posts/java_concurrency_read_write.jpg)

下面分别介绍下上面图中涉及的 4 种操作：

1. read：Java 执行引擎访问本地工作内存中的变量副本，如果变量副本无效（变量副本不存在也是无效的一种），那就去主存中获取，同时在本地工作内存中缓存一份
2. write：Java 执行引擎将最新的变量值赋值给工作内存中的变量副本，同时需要判断是否需要将这个新的值立即同步给主内存，如果需要同步的话，还需要配合 lock 操作
3. lock：Java 执行引擎将主内存中的变量锁定，锁定的含义有：其他的线程在此之后不能访问这个变量直到本线程 unlock；一旦锁定，其他线程针对这个变量的操作必须等待
4. unlock：Java 执行引擎将主内存中的变量解锁，解锁之后才能：各个线程并发访问这个变量；某个线程再次锁定

## Java Thread 创建

在 Java 中，我们都知道，一个线程直接对应了一个 Thread 类对象。创建和启动一个线程是比较容易的，我们只需要创建一个 Thread 对象，然后调用对象的 start 方法即可。但是在创建一个 Thread 对象和启动线程 JVM 中究竟发生了什么？本节我们就来看下。

如果你仔细看过 Thread 类的源码就知道，在创建一个 Thread 对象的时候，除了一些初始化设置之外就没有什么实质性的操作，真正的工作其实是在 start 方法调用中产生的。也就是说，只是创建了一个 Thread 对象和创建一个普通的 Java 对象没什么实质性的差异。因此我们需要看下在 HotSpot 11 中的 Thread start 实现，那么怎么找实现的代码呢？打开 Thread 类的代码，在这个类开始的地方我们看到了如下的代码：

```java
/* Make sure registerNatives is the first thing <clinit> does. */
private static native void registerNatives();
static {
    registerNatives();
}
```

如果你熟悉 JNI 的话，就知道这里的 registerNatives 方法就是将 Thread 类中的 java 方法和一个本地的 C/C++ 函数进行对应，同时由于这个方法是类加载的时候调用的，因此在类首次加载的时候（Bootstrap 类加载）就会注册这些 native 方法，那么 Thread 中都有哪些 native 方法呢？看下 Thread 类的结尾处（JDK 源码中一般都是将 native 方法声明在类的结尾处，方便查找）：

```java
public static native Thread currentThread();
public static native void yield();
public static native void sleep(long millis) throws InterruptedException;
private native void start0();
private native boolean isInterrupted(boolean ClearInterrupted);
public final native boolean isAlive();
public native int countStackFrames();
public static native boolean holdsLock(Object obj);
private static native StackTraceElement[][] dumpThreads(Thread[] threads);
private static native Thread[] getThreads();
private native void setPriority0(int newPriority);
private native void stop0(Object o);
private native void suspend0();
private native void resume0();
private native void interrupt0();
private native void setNativeName(String name);
```

这些方法，不用说你肯定非常熟悉，这里就不赘述了。

在进入代码量及其巨大且复杂的 OpenJDK 之前有几点想说明下：

1. 所有用的源代码都是从 OpenJDK 官方下载的 HotSpot JVM 代码，版本 11
2. 分析的时候我会将代码中的注释一起放上来，方便大家阅读
3. 分析的时候，我们只关注重点的代码，也就是核心功能代码，细节暂时不会关心

好的，现在我们打开 OpenJDK 11 的代码（至于下载源码，请参考 [OpenJDK wiki](https://wiki.openjdk.java.net/)；使用什么 IDE 打开全看你的心情，我是使用 eclipse cdt），全局搜索如下内容：
`java_lang_Thread_registerNatives`
什么？你问我为啥搜这个？如果你有这个疑问的话，可以先看下 [JNI 的内容](https://blog.csdn.net/createchance/article/details/53783490)～这里简单地说下，这种内容是 JNI 默认的java 方法和 native 方法对应的方式，JVM 运行的时候会通过这种方式查找本地符号表中的符号的符号，然后直接跳转过去～

我们搜索之后可以看到在`src/java.base/share/native/libjava/Thread.c`中定了这个函数：

```c
static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    {"stop0",            "(" OBJ ")V", (void *)&JVM_StopThread},
    {"isAlive",          "()Z",        (void *)&JVM_IsThreadAlive},
    {"suspend0",         "()V",        (void *)&JVM_SuspendThread},
    {"resume0",          "()V",        (void *)&JVM_ResumeThread},
    {"setPriority0",     "(I)V",       (void *)&JVM_SetThreadPriority},
    {"yield",            "()V",        (void *)&JVM_Yield},
    {"sleep",            "(J)V",       (void *)&JVM_Sleep},
    {"currentThread",    "()" THD,     (void *)&JVM_CurrentThread},
    {"countStackFrames", "()I",        (void *)&JVM_CountStackFrames},
    {"interrupt0",       "()V",        (void *)&JVM_Interrupt},
    {"isInterrupted",    "(Z)Z",       (void *)&JVM_IsInterrupted},
    {"holdsLock",        "(" OBJ ")Z", (void *)&JVM_HoldsLock},
    {"getThreads",        "()[" THD,   (void *)&JVM_GetAllThreads},
    {"dumpThreads",      "([" THD ")[[" STE, (void *)&JVM_DumpThreads},
    {"setNativeName",    "(" STR ")V", (void *)&JVM_SetNativeThreadName},
};
JNIEXPORT void JNICALL
Java_java_lang_Thread_registerNatives(JNIEnv *env, jclass cls)
{
    (*env)->RegisterNatives(env, cls, methods, ARRAY_LENGTH(methods));
}
```

可以看到，在 registerNatives 函数中，向虚拟机注册了很多的本地方法，基本就是上面我们提到的 Thread 中的所有 native 方法。JNINativeMethod 这是个函数指针，定义在 JNI 中，内容如下：

```c
/*
 * used in RegisterNatives to describe native method name, signature,
 * and function pointer.
 */
typedef struct {
    char *name;
    char *signature;
    void *fnPtr;
} JNINativeMethod;
```

现在，我们知道了，第一列是 Java 中定义的 native 方法名称，第二列是 Java 方法签名，第三列是本地方法对应函数。因此，Java 中的 start 方法就是对应 native 的 JVM_StartThread 函数：

```cpp
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  JavaThread *native_thread = NULL;
  // We cannot hold the Threads_lock when we throw an exception,
  // due to rank ordering issues. Example:  we might need to grab the
  // Heap_lock while we construct the exception.
  bool throw_illegal_thread_state = false;
  // We must release the Threads_lock before we can post a jvmti event
  // in Thread::start.
  {
    // Ensure that the C++ Thread and OSThread structures aren't freed before
    // we operate.
    MutexLocker mu(Threads_lock);
    // Since JDK 5 the java.lang.Thread threadStatus is used to prevent
    // re-starting an already started thread, so we should usually find
    // that the JavaThread is null. However for a JNI attached thread
    // there is a small window between the Thread object being created
    // (with its JavaThread set) and the update to its threadStatus, so we
    // have to check for this
    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
    } else {
      // We could also check the stillborn flag to see if this thread was already stopped, but
      // for historical reasons we let the thread detect that itself when it starts running
      jlong size =
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));
      // Allocate the C++ Thread structure and create the native thread.  The
      // stack size retrieved from java is 64-bit signed, but the constructor takes
      // size_t (an unsigned type), which may be 32 or 64-bit depending on the platform.
      //  - Avoid truncating on 32-bit platforms if size is greater than UINT_MAX.
      //  - Avoid passing negative values which would result in really large stacks.
      NOT_LP64(if (size > SIZE_MAX) size = SIZE_MAX;)
      size_t sz = size > 0 ? (size_t) size : 0;
      // 重点看这里！！！
      native_thread = new JavaThread(&thread_entry, sz);
      // At this point it may be possible that no osthread was created for the
      // JavaThread due to lack of memory. Check for this situation and throw
      // an exception if necessary. Eventually we may want to change this so
      // that we only grab the lock if the thread was created successfully -
      // then we can also do this check and throw the exception in the
      // JavaThread constructor.
      if (native_thread->osthread() != NULL) {
        // Note: the current thread is not being used within "prepare".
        native_thread->prepare(jthread);
      }
    }
  }
  if (throw_illegal_thread_state) {
    THROW(vmSymbols::java_lang_IllegalThreadStateException());
  }
  assert(native_thread != NULL, "Starting null thread?");
  if (native_thread->osthread() == NULL) {
    // No one should hold a reference to the 'native_thread'.
    native_thread->smr_delete();
    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_THREADS,
        os::native_thread_creation_failed_msg());
    }
    THROW_MSG(vmSymbols::java_lang_OutOfMemoryError(),
              os::native_thread_creation_failed_msg());
  }
  Thread::start(native_thread);
JVM_END
```

上面代码本身并不多，并且有很多的注释，这样我们就能很好地理解这段代码了。我们要关注的重点是这一行：

```cpp
native_thread = new JavaThread(&thread_entry, sz);
```

这里创建了一个 JavaThread 对象，并且给了两个参数，第一个暂且不管，后面我们会重点说明，第二个是 stack size，也就是每一个线程的栈大小，这个参数可以在创建 Thread 对象的时候指定，也可以添加 JVM 启动参数：`Xss`(设定每个线程的堆栈大小)。进去看下：

```cpp
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
                       Thread() {
  initialize();
  _jni_attach_state = _not_attaching_via_jni;
  set_entry_point(entry_point);
  // Create the native thread itself.
  // %note runtime_23
  os::ThreadType thr_type = os::java_thread;
  thr_type = entry_point == &compiler_thread_entry ? os::compiler_thread :
                                                     os::java_thread;
  // 通过 os 类的 create_thread 函数来创建一个线程
  os::create_thread(this, thr_type, stack_sz);
  // The _osthread may be NULL here because we ran out of memory (too many threads active).
  // We need to throw and OutOfMemoryError - however we cannot do this here because the caller
  // may hold a lock and all locks must be unlocked before throwing the exception (throwing
  // the exception consists of creating the exception object & initializing it, initialization
  // will leave the VM via a JavaCall and then all locks must be unlocked).
  //
  // The thread is still suspended when we reach here. Thread must be explicit started
  // by creator! Furthermore, the thread must also explicitly be added to the Threads list
  // by calling Threads:add. The reason why this is not done here, is because the thread
  // object must be fully initialized (take a look at JVM_Start)
}
```

可以看到，重点是通过 os 类的 create_thread 函数来创建一个线程，因为 JVM 是跨平台的，并且不同操作系统上的线程实现机制可能是不一样的，因此这里的 create_thread 肯定会有多个针对不同平台的实现，我们查看这个函数的实现就知道了：
![java_concurrency_create_thread](/assets/posts/java_concurrency_create_thread.png)
可以看到，HotSpot 提供了主要的操作系统上的实现，因为在服务器上，linux 的占比是很高的，因此我们这里就看下 linux 上的实现即可：

```cpp
bool os::create_thread(Thread* thread, ThreadType thr_type,
                       size_t req_stack_size) {
  ...
  // init thread attributes
  pthread_attr_t attr;
  pthread_attr_init(&attr);
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
  // Calculate stack size if it's not specified by caller.
  size_t stack_size = os::Posix::get_initial_stack_size(thr_type, req_stack_size);
  // In the Linux NPTL pthread implementation the guard size mechanism
  // is not implemented properly. The posix standard requires adding
  // the size of the guard pages to the stack size, instead Linux
  // takes the space out of 'stacksize'. Thus we adapt the requested
  // stack_size by the size of the guard pages to mimick proper
  // behaviour. However, be careful not to end up with a size
  // of zero due to overflow. Don't add the guard page in that case.
  size_t guard_size = os::Linux::default_guard_size(thr_type);
  if (stack_size <= SIZE_MAX - guard_size) {
    stack_size += guard_size;
  }
  assert(is_aligned(stack_size, os::vm_page_size()), "stack_size not aligned");
  int status = pthread_attr_setstacksize(&attr, stack_size);
  assert_status(status == 0, status, "pthread_attr_setstacksize");
  // Configure glibc guard page.
  pthread_attr_setguardsize(&attr, os::Linux::default_guard_size(thr_type));
  ...
  pthread_t tid;
  // 创建并启动线程
  int ret = pthread_create(&tid, &attr, (void* (*)(void*)) thread_native_entry, thread);
  ...
}
```

这个函数比较长，这里就省略部分，只保留和线程创建启动相关的部分。可以看到，在 linux 平台上，JVM 的线程是通过大名鼎鼎的 pthread 库来创建启动线程的，这里需要注意下的是，在指定线程栈大小的时候，并不是程序员指定多少就实际是多少的，而是要根据系统平台的限制来综合决定的。

到这里，我们大致上明白了，Java Thread 在底层是对应到一个 pthread 线程。这里有一个问题，就是底层是如果执行我们指定的 run 方法的呢？我们先看下创建并且启动线程的这一行：

```cpp
pthread_t tid;
// 创建并启动线程
int ret = pthread_create(&tid, &attr, (void* (*)(void*)) thread_native_entry, thread);
```

这里通过 pthread_create 来创建并且启动线程，我们看下这个接口的定义：

```cpp
int
pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void *), void *arg);
```

第一个是 pthread_t 结构体数据指针，存放线程信息的，第二个是线程的属性，第三个是线程体，也就是线程实际执行的函数，第四个是线程体的参数列表。

上面调用这个接口的地方，我们指定了线程体函数是 thread_native_entry，参数是 thread 指针。我们先看下 thread_native_entry 这个函数的定义：

```cpp
// Thread start routine for all newly created threads
static void *thread_native_entry(Thread *thread) {
  ...
  // call one more level start routine
  thread->run();
  ...
}
```

同样地，这里省略了很多代码，只保留的重点代码。通过注释我们可以知道，thread-&gt;run() 这一行是最可能执行我们 run 方法的地方。所以问题的重点是，thread 指针指向了谁？并且 run 函数的实现是怎样的？我们知道 thread 指针是我们传递进来的，因此通过往回寻找代码我们发现在创建线程的时候我们执行了如下调用（如果你还记得的话）：

```cpp
os::create_thread(this, thr_type, stack_sz);
```

这里我们将 thread 指针赋值为 this，this 就是 JavaThread 对象，还记得吗？我们前面就是通过创建 JavaThread 对象来创建和启动线程的。所以，我们现在看下 JavaThread 类的 run 函数：

```cpp
// The first routine called by a new Java thread
void JavaThread::run() {
  ...
  // We call another function to do the rest so we are sure that the stack addresses used
  // from there will be lower than the stack base just computed
  thread_main_inner();
}
```

这里重点是调用了 thread_main_inner 函数：

```cpp
void JavaThread::thread_main_inner() {
  assert(JavaThread::current() == this, "sanity check");
  assert(this->threadObj() != NULL, "just checking");
  // Execute thread entry point unless this thread has a pending exception
  // or has been stopped before starting.
  // Note: Due to JVM_StopThread we can have pending exceptions already!
  if (!this->has_pending_exception() &&
      !java_lang_Thread::is_stillborn(this->threadObj())) {
    {
      ResourceMark rm(this);
      this->set_native_thread_name(this->get_thread_name());
    }
    HandleMark hm(this);
    // 这里开始调用 java thread 的 run 方法啦～～～
    this->entry_point()(this, this);
  }
  DTRACE_THREAD_PROBE(stop, this);
  // java 中的 run 方法执行完毕了，这里需要退出线程并清理资源
  this->exit(false);
  // delete cpp 的对象
  this->smr_delete();
}
```

这里就是我们的函数执行体了！！Java Thread 中的 run 方法是在 this-&gt;entry_point()(this, this); 这里调用的。看这里的调用方式就知道，entry_point() 返回的是一个函数指针，然后直接执行了调用。entry_point 函数实现如下：

```cpp
ThreadFunction entry_point() const             { return _entry_point; }
```

这里直接放回了 _entry_point 的 ThreadFunction 类型指针，ThreadFunction 类型其实就是函数指针：

```cpp
typedef void (*ThreadFunction)(JavaThread*, TRAPS);
```

因此，重点就是这里的 _entry_point 是哪里赋值的？这里就需要提到前面埋下的一个坑，在创建 JavaThread 对象的时候，我们传递了一个函数指针 thread_entry：

```cpp
native_thread = new JavaThread(&thread_entry, sz);
```

现在我们在来看 JavaThread 中对 thread_entry 的处理：

```cpp
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
                       Thread() {
   ...
   set_entry_point(entry_point);
   ...
}
```

ok，看到这里，我们明白了上面我们需要寻找的 _entry_point 其实就是 thread_entry 指针！

现在看下 thread_entry 指针指向的函数：

```cpp
static void thread_entry(JavaThread* thread, TRAPS) {
  HandleMark hm(THREAD);
  Handle obj(THREAD, thread->threadObj());
  JavaValue result(T_VOID);
  JavaCalls::call_virtual(&result,
                          obj,
                          SystemDictionary::Thread_klass(),
                          vmSymbols::run_method_name(),
                          vmSymbols::void_method_signature(),
                          THREAD);
}
```

这里就是调用我们 java 中 run 方法的地方，但是好像不能很直观的看出来。我们需要解释一下 call_virtual 函数调用的几个参数，第一个是存放函数执行结果的，因为java Thread 中的 run 是 void 型的，所以不必关心；第二个是 java thread object 对象；第三个是 java Thread class 对象；第四个就是我们 run 方法的名称，其实就是字符串“run”，第五个是方法的签名，最后一个是当前执行线程的宏，通过这个宏可以获得到当前执行线程的指针，指向 JavaThread 对象。现在我们首先看下 run_method_name 定义：

```cpp
  #define VM_SYMBOL_DECLARE(name, ignore)                 \
    static Symbol* name() {                               \
      return _symbols[VM_SYMBOL_ENUM_NAME(name)];         \
    }
  VM_SYMBOLS_DO(VM_SYMBOL_DECLARE, VM_SYMBOL_DECLARE)
  #undef VM_SYMBOL_DECLARE
```

这里是通过宏定义的方式指定的，我们直接搜索这个宏扩展的地方：

```cpp
template(run_method_name,                           "run")  \

```

这里通过 cpp 模版方式定义，现在我们知道了 run_method_name 这个宏定义展开之后会变成 “run” 的 Symbol 指针。同时 void_method_signature 可以看到定义：

```cpp
template(void_method_signature,                     "()V")  \

```

这里就是 public void run() 方法的签名啦～

好的，现在我们知道了参数的信息，至于 call_virtual 函数是怎样调用到 java 方法的，这里我只能说是通过 JVM call_stub 函数指针指向的指针函数跳转到 vtable 实现的。关于 call_stub 实现机制，是 JVM 中非常复杂的一个独立模块，这个模块涉及到 Java 中众多 invoke* 类的字节码执行细节逻辑。这些内容绝对不是一篇文章能够讲完的，因此这里挖一个坑后面专门写文介绍这块的实现（剧透一下，内容涉及汇编，不过只是简单的几条汇编），如果你急于了解这块的内容，可以参考 HotSpot 技术专家的[文章](https://shipilev.net/blog/2015/black-magic-method-dispatch/)。好吧，又挖了一个坑～

到这里，我们梳理清楚了 java 中 thread 的创建和启动过程，以及 run 方法执行的过程。下面总结下：

Java 线程创建和启动过程如下：

```cpp
java_lang_Thread->JNI:start0
JNI->JavaThread:new
JavaThread->os:create_thread
os->os_linux:pthread_create
os_linux->JavaThread:thread_entry
JavaThread->java_lang_Thread:run
```

## Synchronized 实现机制

synchronized 是 Java 并发同步开发的基本技术，是 java 语言层面提供的线程间同步手段。我们编写如下一段代码：

```java
public class SyncTest {
    private static final Object lock = new Object();
    public static void main(String[] args) {
        int a = 0;
        synchronized (lock) {
            a++;
        }
        System.out.println("Result: " + a);
    }
}
```

使用```javap```命令对*.class文件进行反汇编，针对其中同步的部分我们会看到如下字节码：

```bash
monitorenter
iinc 1 by 1
aload_2
monitorexit
```

这其实是 javac 和我们玩的一个小把戏，在编译时将 synchronized 同步块的前后插入 montor 进入和退出的字节码指令，相信这一点大多数 java 程序员都了如指掌。

所以我们想要探索 synchronized 的实现机制，就需要探索 monitorenter 和monitorexit 指令的执行过程了～

我们知道，在 HotSpot JVM 早期的时候，使用了字节码解释器(bytecodeInterpreter)，用C++实现了每条JVM指令（如monitorenter、invokevirtual等），但是这种方式效率低下。这里插个内容，你知道为啥这种方式效率低下呢？JVM 也是通过 C/C++ 来编写的，为啥效率就那么慢呢？其实所谓效率低下，在 CPU 层面其实就是一个本来很简单的功能却需要大量的 CPU 机器指令来完成，从而导致效率低下。但是，我们还要问，为啥一个简单的功能会产生很多的机器指令呢？原因是这样，Java 程序编译之后，会产生很多字节码指令，每一个字节码指令在 JVM 底层执行的时候又会变成一堆 C 代码的执行，这一堆 C 代码在编译之后又会变成很多的机器指令，这样一来，我们的 java 代码最终到机器指令一层，所产生的机器指令将是指数级的，因此就导致了 Java 执行效率非常低下，话说这个帽子貌似现在还在～

如果我们仔细思考下，怎么优化这个问题呢？字节码是肯定不能动的，因为 JVM 的一处编写，到处运行的梦想就是靠它完成的。其实，我们会发现，问题的根本就在于 Java 和机器指令之间隔了一层 C/C++，而例如 GCC 之类的编译器又不能做到绝对的智能编译，所产生的机器码效率仍然不是非常高。因此，我们会想，能不能跳过 C/C++ 这个层次能，直接将 java 字节码和本地机器码进行一个对应呢？是的！可以的！HotSpot 工程师们早就想到了，因此早期的字节码解释器很快就被废弃了，转而采用模板解释器(templateInterpreter)。什么是模板解释器，顾名思义，模版就是将一个 java 字节码通过「人工手动」的方式编写为固定模式的机器指令，这部分不在需要 GCC 的帮助，这样就可以大大减少最终需要执行的机器指令，所以才能提高效率。 所谓模版，就是定义了字节码到机器码转化的统一方式，就像生活中的模板一样，执行字节码的时候套用模版就能得到对应的「人工」调优的机器码～说到这里，真是要感叹，JVM 的字节码执行调优过程真是「人工」智能的结果啊！

上面我们向大家解释了，现代 HotSpot JVM 中采用模板解释器，而不是字节码解释器的原因以及基本的技术方案。现在我们就要从 OpenJDK 中的模版解释器中探索 monitorenter 和 monitorexit 指令执行的细节过程。

在 OpenJDK 11 的源码中，所有 JVM 的解释器（包括最古老的原始解释器）都在：src/hotspot/share/interpreter 目录下，通过这个目录下的代码文件名称我们就很容易地找到模版解释器的代码位置：templateInterpreter.cpp。通过分析这里的实现，我们知道，其实字节码对应到机器码的模版是在 templateTable.cpp 中定义的，这个文件中的代码量不多，全是字节码对应到本地机器码的实现逻辑，这里我们只是放上 monitor 相关的内容：

```cpp
def(Bytecodes::_monitorenter        , ____|disp|clvm|____, atos, vtos, monitorenter        ,  _           );
def(Bytecodes::_monitorexit         , ____|____|clvm|____, atos, vtos, monitorexit         ,  _           );
```

这里的每一列的含义，在代码中都有说明，这里我们只要知道 monitorenter 函数和 monitorexit 函数就是对应字节码的机器码模版的位置，首先我们看下 monitorenter 的实现：

![monitorenter impl](/assets/posts/monitorenter_impl.png)

因为，实际的机器码是和 CPU 相关的，因此 JVM 提供给了几乎所有主流 CPU 的对应版本。这里我们依然看最主流的 x86 的实现（下面要看汇编了，是不是有点激动，但是不要慌，JVM 针对汇编的调用已经做了非常完备的封装，以至于下面的代码看起来和普通的 C 代码没啥区别）：

```cpp
void TemplateTable::monitorenter() {
    ...
    // store object
    __ movptr(Address(rmon, BasicObjectLock::obj_offset_in_bytes()), rax);
    // 跳转执行 lock_object 函数
    __ lock_object(rmon);
    ...
}
```

这里我们仍然只给出重点代码部分，代码比较长，前面有很多指令是初始化执行环境的，最后重点会跳转执行 lock_object 函数，同样这个函数也是有不同 CPU 平台实现的，我们还是看 X86 平台的：

```cpp
void InterpreterMacroAssembler::lock_object(Register lock_reg) {
  ......
  // 如果使用重量级锁，直接进入InterpreterRuntime::monitorenter()执行
  if (UseHeavyMonitors) {
    call_VM(noreg,
            CAST_FROM_FN_PTR(address, InterpreterRuntime::monitorenter),
            lock_reg);
  } else {
    ......
    if (UseBiasedLocking) {
      biased_locking_enter(lock_reg, obj_reg, swap_reg, tmp_reg, false, done, &slow_case);
    }
    ......
    // Call the runtime routine for slow case
    call_VM(noreg,
            CAST_FROM_FN_PTR(address, InterpreterRuntime::monitorenter),
            lock_reg);
    ......
    // 直接跳到这表明获取锁成功，接下来就会返回到entry_point例程进行字节码的执行了。
  }
}
```

上面的代码只保留了其逻辑结构。其实这里的逻辑很简单，就是根据`UseHeavyMonitors`以及`UseBiasedLocking`这两个bool变量决定执行的锁的逻辑，若`UseHeavyMonitors`为true，就执行`InterpreterRuntime::monitorenter`逻辑；否则，若`UseBiasedLocking`为true,就执行`biased_locking_enter`逻辑。

其实从这个变量的名称就能看出来，UseHeavyMonitors指的是是否使用重量级锁而不进行偏向锁优化，可以通过JVM启动参数`-XX:+UseHeavyMonitors`来开启；而UseBiasedLocking指的是 JVM 1.6 之后默认使能的偏向锁优化，可以通过JVM启动参数 ```-XX:+/-UseBiasedLocking``` 来控制开关。什么？你问我什么是偏向锁？ok，别急，下面我们就要开讲了。

### 同步锁优化处理

为了方便接下来的代码分析，下面我要放出 [OpenJDK 官方 wiki](https://wiki.openjdk.java.net/display/HotSpot/Synchronization) 中针对锁优化的原理图：
![锁优化的原理图](/assets/posts/header_word_layout_and_object_states.gif)
这张图咋一看，很复杂，你可能看不懂。但是，相信我，如果你仔细看完下面的代码分析并且自己结合 JVM 源码尝试理解，你肯定会完全吃透这张图。

预警：接下来的内容会比较烧脑，内容比较复杂，建议你休息一下再来看～

在解释上面那张图之前，需要介绍一下 Java 对象的内存布局，因为上面图中的实现原理就是充分利用 java 对象的头完成的。Java 对象在内存的结构基本上分为：对象头和对象体，其中对象头存储对象特征信息，对象体存放对象数据部分。那么我们除了研究 JVM 源代码获取 java 对象内存布局之外，还有什么办法得知对象的内存布局呢？有的，在 OpenJDK 工程中，有一个子工程叫做：[jol](https://openjdk.java.net/projects/code-tools/jol/)，全名是：java object layout，是的就是 java 对象布局的意思。这是一个工具库，通过这个库可以获取 JVM 中对象布局信息，下面我们展示一个简单的例子（这也是官方给的例子）：

```java
public class JOLTest {
    public static void main(String[] args) {
        System.out.println(VM.current().details());
        System.out.println(ClassLayout.parseClass(A.class).toPrintable());
    }
    public static class A {
        boolean f;
    }
}
```

这里通过 JOL 的接口来获取类 A 的对象内存布局，执行之后输出如下内容：

```bash
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# WARNING | Compressed references base/shifts are guessed by the experiment!
# WARNING | Therefore, computed addresses are just guesses, and ARE NOT RELIABLE.
# WARNING | Make sure to attach Serviceability Agent to get the reliable addresses.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
JOLTest$A object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                               VALUE
      0    12           (object header)                           N/A
     12     1   boolean A.f                                       N/A
     13     3           (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 3 bytes external = 3 bytes total
```

这里我们看到输出了很多信息，上面我们类 A 的对象布局如下：12 byte 的对象头 + 1 byte 的对象体 + 3 byte 的填充部分。下面我们分别简要说明下：

对象头：从 JVM 的代码中我们可以看出一个对象的头部定义：

```cpp
  volatile markOop _mark;
  union _metadata {
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;
```

可以看到分为两部分：第一部分就是 mark 部分，官方称之为 mark word，第二个是 klass 的类型指针，指向这个对象的类对象。这里的 mark word 长度是一个系统字宽，在 64 bit 系统上就是 8 个字节，从上面的日志中我们可以看到虚拟机默认使能了 compressed klass，因此第二部分的 union 其实就是 narrowKlass 类型的，如果我们继续看下 narrowKlass 的定义就知道这是个 32 bit 的 unsigned int 类型，因此将占用 4 个字节，所以对象的头部长度整体为 12 字节。

对象体：因为 A 类只定义了一个字段，是 boolean 类型的，在 JVM 底层占用一个字节的长度

填充部分：上面的对象头和对象体长度的总和为 13 字节，因为 JVM 的内存是以 8 字节长度对齐的，因此这里需要填充 3 个字节的长度是的整体的长度等于 16 字节

上面我们说明了下一个 java 对象的内存布局，上面我们展示的是 JVM 启用了 oop 压缩技术的结果，你可以通过 ```-XX:-UseCompressedOops``` 来关闭它，关闭之后的布局如下图（这是标准的内存布局）：

![object_layout](/assets/posts/object_layout.png)

下面回到我们的话题，我们重点需要关注的是对象的头部定义。上面我们看到，对象的头部总共可以分为两个部分：第一个是 mark word ，第二个是这个类的对象指针信息。其中 Mark word 用于存储对象自身运行时的数据，如 hash code、GC 分代年龄等等信息，他是实现偏向锁的关键。

对象头部信息是与对象自身定义的数据无关的额外信息，考虑到虚拟机的空间效率，mark word 被设计成一个非固定数据结构以便在极小的空间内存储尽量多的信息，它会根据对象的状态复用自己的存储空间。在 JVM 中，mark word 内存布局定义在 /src/hotspot/share/oops/markOop.hpp 中，在这个文件的注释中清晰地说明了在 32bit 和 64bit 系统中对象不同状态下的 mark word 布局：

```cpp
//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
...
//    [JavaThread* | epoch | age | 1 | 01]       lock is biased toward given thread
//    [0           | epoch | age | 1 | 01]       lock is anonymously biased
//
//  - the two lock bits are used to describe three states: locked/unlocked and monitor.
//
//    [ptr             | 00]  locked             ptr points to real header on stack
//    [header      | 0 | 01]  unlocked           regular object header
//    [ptr             | 10]  monitor            inflated lock (header is wapped out)
//    [ptr             | 11]  marked             used by markSweep to mark an object
//                                               not valid at any other time
```

在所有状态的前两个状态是我们需要重点关注的：normal object 和 biased object。仔细看这两部分的头定义，最后三个 bit 用来区分偏向锁和普通锁的部分，这里就和前面 OpenJDK wiki 中的图对上了，总结如下：

biased_lock | lock | 状态
------------|------|-----
1           | 01   | 可偏置、但未锁且未偏置
1           | 01   | 已偏置、锁定或未锁定
0           | 01   | 已解锁、不可偏置
-           | 00   | 轻量级锁定
-           | 10   | 重量级锁定

具体的内容参考 [OpenJDK wiki](https://wiki.openjdk.java.net/display/HotSpot/Synchronization) 中的图，为了方便描述，这里再贴一次：

![锁优化的原理图](/assets/posts/header_word_layout_and_object_states.gif)

下面我们针对上面的那张图解释一下，首先解释一下什么是偏向锁。所谓偏置，就是偏向某一个线程的意思，也就是说这个锁首先假设自己被偏向的线程持有。在单个线程连续持有锁的时候，偏向锁就起作用了。如果一个线程连续不停滴获取锁，那么获取的过程中如果没有发生竞态，那么可以跳过繁重的同步过程，直接就获得锁执行，这样可以大大提高性能。偏向锁是 JDK 1.6 中引入的一项锁优化手段，它的目的就是消除数据在无争用的情况下的同步操作，进一步提高运行性能。这里还涉及到了轻量级锁，轻量级锁也是 JDK 1.6 引入的一个锁优化机制，所谓轻量级是相对于使用操作系统互斥机制来实现传统锁而言的，在这个角度上，传统的方式就是重量级锁，所谓重量级的原因是同步的方式是一种悲观锁，会导致线程的状态切换，而线程状态的切换是一个相当重量级的操作。在 JVM 6 以上版本中，默认使能偏置优化技术，因此上面的图中，只要分配了新的对象，都会指定左图中的逻辑。首先一个新的对象处于未锁定、未偏置但是可以偏置的状态，也就是上面表格的第一行，这个时候如果有个线程来获取这个对象锁，那么就直接进入已偏置状态，在64位的JVM中，这个状态和未偏置状态的差别就是原来开头的 25 bit 的 unused 与 31bit 的 hash code 变成了 54 bit 的 thread 指针和 2 bit 的分代信息，看起来hashCode与偏向锁冲突了？其实，如果一个对象请求计算identity hashcode，那么将导致它不能被偏向锁锁定，后面我们会在代码中看到这一点。经过这一步的操作，我们就在对象头部存储上了线程指针信息，标记这个对象的锁已经被这个线程持有了，相当于表明：此花已有主。下次当这个线程在此获取这个锁的时候，只要状态没有发生变化，所需要的开销就是一次指针的比较运算，而这个运算是非常轻量的。但是在某个线程持有这个对象锁的时候，如果有另外一个线程来竞争了，锁的偏置状态结束，会触发撤销偏置的逻辑，这个时候可以分为如下两个情况（持有锁的线程称为 线程 A，竞争的称为 线程 B）：

1. 线程 B 到达的时候，线程 A 已经放开对象锁，此时对象锁处于的状态是：偏置对象、未锁定
2. 线程 B 到达的时候，线程 A 正持有这个锁，此时对象处于的状态是：偏置对象，已锁定

以上两种情况的操作是不同的，下面分别讲述。

第一种情况，由于对象的状态是偏置对象并且未锁定，因此首先将对象状态置为不可偏置对象并且未锁定。然后在线程的栈空间新建一个 lock record 的空间，用于存储对象目前的 mark word 的拷贝，然后虚拟机将使用 CAS 操作尝试将对象的 mark word 更新指向 lock record 空间。如果这个更新成功了，那么这个线程就拥有了该对象的锁，并且 mark word 的锁标志位变成 00，表示当前对象锁处于轻量级锁定状态，这个过程如下图所示（上图是锁定前，下图是锁定后）：

![before_lock](/assets/posts/before_lock.png)

---

![after_lock](/assets/posts/after_lock.png)

第二种情况，就是当前对象正在处于锁定状态，这个时候仍然会升级为轻量级锁定对象，但是此时线程 B 获取锁会失败，因此这个对象锁会进行「膨胀」操作，彻底成为一个重量级的锁。

另外，需要说明的是，在一个线程轻量级锁定某个对象的时候，另外一个线程过来竞争也会导致锁的膨胀，进入到重量级的锁。总结起来的话，就是没有竞争就是偏向锁，少量竞争就是轻量级锁，大量竞争就是重量级锁。

需要说明的是，偏向锁和轻量级锁的关系并不是互相取代或者竞争关系，而是属于在不同情况下的不同的锁优化手段。这里就需要提到在概念上的锁分类，通常可以分为：悲观锁和乐观锁。所谓悲观锁，就是认为如果我不做充分的同步的手段（包括执行重量级的操作）就肯定会出现问题；所谓乐观锁，就是会乐观地预估系统当前的状态，认为状态是符合预期的，因此不用重量级的同步也可以完成同步，如果不巧发生了竞态，就退避，然后再按照一定的策略重试。在 java 中，很多传统的同步方式（包括 synchronized，重入锁）都是悲观锁，在 java 9 中在 Thread 类中新加入了一个接口 onSpinWait 就是一种乐观锁，另外在 JUC 中很多的原子工具类都使用到了 CAS（Compare And Swap）操作，这种操作本质上是利用 CPU 提供的特定原子操作指令，基于冲突检测的方式来实现的一种乐观锁定的同步技术。

上面我们详细介绍了 JVM 中的各种锁优化的技术细节，现在我们看下在 JVM 的代码实现上是如何操作的。我们回到之前对代码的分析。

```cpp
void InterpreterMacroAssembler::lock_object(Register lock_reg) {
  ......
  // 如果使用重量级锁，直接进入InterpreterRuntime::monitorenter()执行
  if (UseHeavyMonitors) {
    call_VM(noreg,
            CAST_FROM_FN_PTR(address, InterpreterRuntime::monitorenter),
            lock_reg);
  } else {
    Label done;
    // cmpxchg其实就是CAS操作，必须使用rax寄存器作为老数据的存储。
    const Register swap_reg = rax; // Must use rax for cmpxchg instruction
    const Register tmp_reg = rbx; // Will be passed to biased_locking_enter to avoid a problematic case where tmp_reg = no_reg.
    const Register obj_reg = LP64_ONLY(c_rarg3) NOT_LP64(rcx); // Will contain the oop

    ......
    Label slow_case;

    // Load object pointer into obj_reg
    movptr(obj_reg, Address(lock_reg, obj_offset));
    //如果虚拟机参数允许使用偏向锁，那么进入biased_locking_enter()中
    if (UseBiasedLocking) {
      // lock_reg :存储的是分配的BasicObjectLock的指针
      // obj_reg :存储的是锁对象的指针
      // slow_case :即InterpreterRuntime::monitorenter()；
      // done :标志着获取锁成功。

      // slow_case 和 done 也被传入，这样在biased_locking_enter()中，就可以根据情况跳到这两处了。
      biased_locking_enter(lock_reg, obj_reg, swap_reg, tmp_reg, false, done, &slow_case);
    }
    ......
    ......

    // 如果使用重量级锁，直接跳到这，需要进入InterpreterRuntime::monitorenter()中去获取锁。
    bind(slow_case);
    // Call the runtime routine for slow case
    call_VM(noreg,
            CAST_FROM_FN_PTR(address, InterpreterRuntime::monitorenter),
            lock_reg);
    // 直接跳到这表明获取锁成功，接下来就会返回到entry_point例程进行字节码的执行了。
    bind(done);
  }
}
```

上面的代码，在原始的代码基础上有删减，保留了核心关键逻辑。前文提到了，这里根据`UseHeavyMonitors`以及`UseBiasedLocking`这两个bool变量决定执行的锁的逻辑，若`UseHeavyMonitors`为true，就执行`InterpreterRuntime::monitorenter()`逻辑；否则，若`UseBiasedLocking`为true,就执行`biased_locking_enter()`逻辑。由于JDK1.6后默认开启偏向锁优化，因此线程会执行`biased_locking_enter()`获取偏向锁。我们先了解一下函数中出现的bind(slow_case)和bind(done)，有何作用？看看它的声明：

```cpp
  // Label functions
  void bind(Label& L); // binds an unbound label L to the current code position
```

其实是用来将当前代码的位置绑定到一个Label中的，这样程序就能通过这个label跳转到这段代码执行（相当于汇编中的LABEL伪指令）。在调用`biased_locking_enter`时，slow_case 和 done 标签也被传入，这样在`biased_locking_enter`中，就可以根据情况回跳到这两处了：比如说获取偏向锁失败，需要跳回时，slow_case进行锁升级；或者成功获取偏向锁，返回done处，继续执行方法的字节码。

`biased_locking_enter(...)`用于获取偏向锁，代码比较长，下面将一步步分析。

#### 1. 判断锁对象是否为偏向锁状态

```cpp
// mark_addr:锁对象头中的markOop指针。
Address mark_addr      (obj_reg, oopDesc::mark_offset_in_bytes());
NOT_LP64( Address saved_mark_addr(lock_reg, 0); )

if (PrintBiasedLockingStatistics && counters == NULL) {
  counters = BiasedLocking::counters();
}

Label cas_label;
int null_check_offset = -1;
// 如果swap_reg中没存mark_addr，那么就先将mark_addr存入swap_reg中。
if (!swap_reg_contains_mark) {
  null_check_offset = offset();
  movptr(swap_reg, mark_addr);
}
// 将对象的mark_addr，即markOop指针移入tmp_reg中
movptr(tmp_reg, swap_reg);
// 将tmp_reg和biased_lock_mask_in_place进行与操作，biased_lock_mask_in_place为111，和它进行与就可以取出markOop中后三位，即（是否偏向锁+锁标志位）
andptr(tmp_reg, markOopDesc::biased_lock_mask_in_place);
// 将上面结果，即（是否偏向锁+锁标志位）和biased_lock_pattern再次比较（biased_lock_pattern为5，即101），如果不相等，则表明不为偏向锁状态，需要进行CAS操作，跳往cas_label；否则即为偏向锁状态，接着往下走。
cmpptr(tmp_reg, markOopDesc::biased_lock_pattern);
jcc(Assembler::notEqual, cas_label);
```

对于上文的A、B线程例子的的情况一：线程A将继续执行下面步骤获取偏向锁；由于假设B在A释放锁后进入同步代码块，此时被锁对象处于可偏向状态（见后文，其实偏向锁的释放就是什么也不做），故B也继续执行下面步骤。

#### 2. 判断锁对象之前是否已经偏向当前线程

走到这，表明锁对象已经为偏向锁态，需要判断锁对象之前是否已经偏向当前线程。

```cpp
// 将锁对象所属类的prototype_header移动至tmp_reg中，prototype_header中存储的也是markOop。
// prototype_header是专门为偏向锁打造的，初始时类的prototype_header为偏向锁态，即后三位为101，一旦发生了bulk_revoke,那么就会设为无锁态，即001。
// bulk_revoke为批量撤销，每次类发生bulk_rebais时（类的所有对象重设偏向锁），类prototype_header中的epoch就会+1，当epoch达到一个阈值时，就会发生bulk_revoke，撤销该类每个对象的偏向锁，这样该类的所有对象以后都不能使用偏向锁了，其实也就是虚拟机认为该对象不适合偏向锁。
load_prototype_header(tmp_reg, obj_reg);

// 将当前线程id和类的prototype_header相或，这样得到的markOop为（当前线程id + prototype_header中的（epoch + 分代年龄 + 偏向锁标志 + 锁标志位）），注意prototype_header的分代年龄那4个字节为0
orptr(tmp_reg, r15_thread);
// 将上面计算得到的结果与锁对象的markOop进行异或，tmp_reg中相等的位全部被置为0，只剩下不相等的位。
xorptr(tmp_reg, swap_reg);
Register header_reg = tmp_reg;
// 对((int) markOopDesc::age_mask_in_place)进行按位取反，age_mask_in_place为...0001111000,取反后，变成了...1110000111,除了分代年龄那4位，其他位全为1；
// 将取反后的结果再与header_reg相与，这样就把header_reg中除了分代年龄之外的其他位取了出来，即将上面异或得到的结果中分代年龄给忽略掉。
andptr(header_reg, ~((int) markOopDesc::age_mask_in_place));
// 如果除了分代年龄，对象的markOop和（当前线程id+其他位）相等，那么上面与操作的结果应该为0，表明对象之前已经偏向当前线程，即markOop中存放有当前线程id，那么跳到done处，直接进入方法执行即可；否则表明当前线程还不是偏向锁的持有者，会接着往下走。
jcc(Assembler::equal, done);
```

对于A、B线程例子的的情况一，A线程进入时由于还未偏向，接着执行；线程B进入时，由于对象并不是偏向当前B线程自己（见后文，其实偏向锁的释放就是什么也不做，故markOop的线程ID还是A），因此也是继续执行。

#### 3. 判断是否需要撤销锁对象的偏向

走到这，表明锁对象并没有偏向当前线程，接下来判断是否需要撤销锁对象的偏向。

```cpp
// (将header_reg和111相与，如果结果不为0，则表明header_reg后三位存在不为0的位，证明之前进行异或时，类的prototype_header后面三位与对象markOop的后三位不相等，但是)? 能走到这，表明对象markword后三位为101，即偏向模式。现在类的prototype_header和对象markOop后三位不相等，即对象所属类不再支持偏向，发生了bulk_revoke，所以需要对当前对象进行偏向锁的撤销；否则表明目前该类还支持偏向锁，接着往下走。
testptr(header_reg, markOopDesc::biased_lock_mask_in_place);
jccb(Assembler::notZero, try_revoke_bias);
```

对于A、B线程例子的的情况一，这里假设锁的对象所属于的类依旧支持偏向锁，因此线程A与B都无需撤销重偏向，继续执行。

#### 4. 断当前对象的epoch是否合法

走到这，表明锁对象还支持偏向锁，需要判断当前对象的epoch是否合法，如果不合法，需要取进行重偏向。合法的话接着往下走。

```cpp
// 测试对象所属类的prototype_header中epoch是否为0，不为0的话则表明之前异或时，类的prototype_header中epoch和对象markOop的epoch不相等，表明类在对象分配后发生过bulk_rebais()（前面提到过，每次发生bulk_rebaise,类的prototype header中的epoch都会+1），所以之前对象的偏向就无效了，需要进行重偏向。否则接着往下走。
testptr(header_reg, markOopDesc::epoch_mask_in_place);
jccb(Assembler::notZero, try_rebias);
```

对于A、B线程例子的的情况一，仍然假设没有发生bulk_rebais()，故线程A和B都继续执行。

#### 5. 尝试获取偏向锁

走到这，表明锁对象的偏向态合法，可以尝试去获取锁，使对象偏向当前线程。

```cpp
// swap_reg存放着mark_addr
// 故此处是取出对象markOop中除线程id之外的其他位
andptr(swap_reg,
        markOopDesc::biased_lock_mask_in_place | markOopDesc::age_mask_in_place | markOopDesc::epoch_mask_in_place);
// 将其他位移动至 tmp_reg。
movptr(tmp_reg, swap_reg);
// 将其他位和当前线程id进行或，构造成一个新的完整的一个字(32bit/64bit)的markOop,存入tmp_reg中。新的markOop因为保存了当前线程id，所以会偏向当前线程。
orptr(tmp_reg, r15_thread);
// 尝试利用CAS操作将新构成的markOop存入对象头的mark_addr处，如果设置成功，则获取偏向锁成功。
// 这里说明下，cmpxchgptr操作会强制将rax寄存器（swap_reg）中内容作为老数据，与第二个参数，在这里即mark_addr处的内容进行比较，如果相等，则将第一个参数的内容，即tmp_reg中的新数据，存入mark_addr。
cmpxchgptr(tmp_reg, mark_addr); // compare tmp_reg and swap_reg
// 上面CAS操作失败的情况下，表明对象头中的markOop数据已经被篡改，即有其他线程已经获取到偏向锁，因为偏向锁不容许多个线程访问同一个锁对象，所以需要跳到slow_case处，去撤销该对象的偏向锁，并进行锁升级。
if (slow_case != NULL) {
  jcc(Assembler::notZero, *slow_case);
}
// 上面CAS成功的情况下，直接就跳往done处，回去执行方法的字节码了。
jmp(done);
```

对于A、B线程例子的的情况一：对于线程A，此处的cmpxchgptr成功(新构建的markOop与原markOop相同)，成功获得偏向锁；而对于后来的线程B，cmpxchgptr失败（见后文，其实偏向锁的释放就是什么也不做，故原markOop的线程ID还是A，因而新构建的markOop与原markOop不同），跳转到slow_case，也就是执行`InterpreterRuntime::monitorenter`，进行撤销偏向锁。

#### 6. try_rebais和try_revoke

其实到这里，biased_locking_enter()已经结束了，不过上面多处提到了try_rebais和try_revoke，两个Label，它们对应的代码也定义在biased_locking_enter中。

```cpp
bind(try_rebias);
// 将锁对象所属类的prototype_header送入tmp_reg。
load_prototype_header(tmp_reg, obj_reg);
// 尝试用CAS操作，使对象的markOop重置为无线程id的偏向锁态，即不偏向任何线程。
cmpxchgptr(tmp_reg, mark_addr);
// 和第5步一样，如果CAS失败，则表明对象头的markOop数据已经被其他线程更改，需要跳往slow_case进行撤销偏向锁，否则跳往done处，执行字节码。
if (slow_case != NULL) {
  jcc(Assembler::notZero, *slow_case);
}
jmp(done);
```

```cpp
bind(try_revoke_bias);
// 走到这，表明这个类的prototype_header中已经没有偏向锁的位了，即这个类的所有对象都不再支持偏向锁了，但是当前对象仍为偏向锁状态，所以我们需要重置下当前对象的markOop为无锁态。
// 将锁对象所属类的prototype_header送入tmp_reg。
load_prototype_header(tmp_reg, obj_reg);
// 尝试用CAS操作，使对象的markOop重置为无锁态。这里是否失败无所谓，即使失败了，也表明其他线程已经移除了对象的偏向锁标志。
cmpxchgptr(tmp_reg, mark_addr);
//接下来会回到lock_object()方法中，继续轻量级锁的获取。
```

### 退出偏向锁

与monitorenter一样，我们看下monitorexit的实现：

```cpp
void TemplateTable::monitorexit() {
  ...
  __ unlock_object(c_rarg1);
  ...
}
```

```cpp
void InterpreterMacroAssembler::unlock_object(Register lock_reg) {
  assert(lock_reg == c_rarg1, "The argument is only for looks. It must be rarg1");

  if (UseHeavyMonitors) {
    call_VM(noreg,
            CAST_FROM_FN_PTR(address, InterpreterRuntime::monitorexit),
            lock_reg);
  } else {
    ...
    if (UseBiasedLocking) {
      biased_locking_exit(obj_reg, header_reg, done);
    }
    ...
    // Call the runtime routine for slow case.
    movptr(Address(lock_reg, BasicObjectLock::obj_offset_in_bytes()),
         obj_reg); // restore obj
    call_VM(noreg,
            CAST_FROM_FN_PTR(address, InterpreterRuntime::monitorexit),
            lock_reg);

    bind(done);

    restore_bcp();
  }
}
```

```cpp
void MacroAssembler::biased_locking_exit(Register obj_reg, Register temp_reg, Label& done) {
  assert(UseBiasedLocking, "why call this otherwise?");

  // Check for biased locking unlock case, which is a no-op
  // Note: we do not have to check the thread ID for two reasons.
  // First, the interpreter checks for IllegalMonitorStateException at
  // a higher level. Second, if the bias was revoked while we held the
  // lock, the object could not be rebiased toward another thread, so
  // the bias bit would be clear.
  movptr(temp_reg, Address(obj_reg, oopDesc::mark_offset_in_bytes()));
  andptr(temp_reg, markOopDesc::biased_lock_mask_in_place);
  cmpptr(temp_reg, markOopDesc::biased_lock_pattern);
  jcc(Assembler::equal, done);
}
```

如注释所言，这是一个“无操作”的过程(no-op)，仅仅检查了对象头是否还是可偏向状态，然后跳转到done标签。

### 升级为轻量级锁

从OpenJDK wiki的那张图我们知道，对于A、B线程例子的的情况一，当后来的B线程获取锁时会撤销对象的偏向锁升级为轻量级锁，下面我们从源码角度来分析。上文说了，当获取偏向锁失败，线程就会转跳到slow_case处执行 InterpreterRuntime::monitorenter 函数。这个函数不仅仅是模版解释器会调用，解释执行器也会执行这个，所以定义在 InterpreterRuntime 类下：

```cpp
// Synchronization
//
// The interpreter's synchronization code is factored out so that it can
// be shared by method invocation and synchronized blocks.
//%note synchronization_3
//%note monitor_1
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
  Handle h_obj(thread, elem->obj());
  if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
  } else {
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
  }
IRT_END
```

因为我们是在 JDK 11 上分析，因此上面的代码，肯定是执行 fast_enter 啦～下面是 fast_enter 函数的定义：

```cpp
//  Fast Monitor Enter/Exit
// This the fast monitor enter. The interpreter and compiler use
// some assembly copies of this code. Make sure update those code
// if the following function is changed. The implementation is
// extremely sensitive to race condition. Be careful.
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock,
                                    bool attempt_rebias, TRAPS) {
  //虚拟机参数判断是否开启了偏向锁
  if (UseBiasedLocking) {
    //如果不处于全局安全点，即正常的Java线程
    if (!SafepointSynchronize::is_at_safepoint()) {
      //通过`revoke_and_rebias`这个函数尝试获取偏向锁
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {//如果在安全点，即VM线程，撤销偏向锁
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }
  slow_enter(obj, lock, THREAD);
}
```

这里开始还是要判断 UseBiasedLocking，如果是 true 的话，就开始执行优化逻辑，否则会 fall back 到 slow_enter 的。是不是感觉判断 UseBiasedLocking 有点啰嗦？其实不是的，因为这个函数在很多地方都会调用的，因此判断是必要的！

在 fast_enter 函数中，有判断 safepoint 的地方，这里大家先不用关心，这个是和 JVM 的 GC 有关的内容，不是我们这里关心的内容。对于正常的Java线程执行，会执行到 _revoke_and_rebias_ 这个函数中，这个函数比较长，主要是在执行 OpenJDK wiki 中图的 revoke 和 rebias 操作。

```cpp
BiasedLocking::Condition BiasedLocking::revoke_and_rebias(Handle obj, bool attempt_rebias, TRAPS) {
  //1：必须在安全点
  assert(!SafepointSynchronize::is_at_safepoint(), "must not be called while at safepoint");
  //2：读取对象头
  markOop mark = obj->mark();
  //is_biased_anonymously()判断mark是否为可偏向状态，即mark的偏向锁标志位为1，锁标志位为 01，线程id为null
  //当计算hashCode时，进入此分支，会在一个非全局安全点进行偏向锁撤销(BIAS_REVOKED)
  if (mark->is_biased_anonymously() && !attempt_rebias) {
    markOop biased_value       = mark;
    //创建一个非偏向的markword
    markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
    //通过cas重新设置偏向锁状态
    markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
    //如果CAS成功，返回偏向锁撤销状态
    if (res_mark == biased_value) {
      return BIAS_REVOKED;
    }
  } else if (mark->has_bias_pattern()) {
    //如果锁对象为可偏向状态（biased_lock:1, lock:01，不管线程id是否为空）,尝试重新偏向
    Klass* k = obj->klass();
    markOop prototype_header = k->prototype_header();
    //如果已经有线程对锁对象进行了全局锁定，则取消偏向锁操作
    if (!prototype_header->has_bias_pattern()) {
      // This object has a stale bias from before the bulk revocation
      // for this data type occurred. It's pointless to update the
      // heuristics at this point so simply update the header with a
      // CAS. If we fail this race, the object's bias has been revoked
      // by another thread so we simply return and let the caller deal
      // with it.
      markOop biased_value       = mark;
      //CAS 更新对象头markword为非偏向锁
      markOop res_mark = (markOop) Atomic::cmpxchg_ptr(prototype_header, obj->mark_addr(), mark);
      assert(!(*(obj->mark_addr()))->has_bias_pattern(), "even if we raced, should still be revoked");
      return BIAS_REVOKED;
    } else if (prototype_header->bias_epoch() != mark->bias_epoch()) {//如果偏向锁epoch过期，则进入当前分支
      //如果允许尝试获取偏向锁
      if (attempt_rebias) {
        //本例中，fast_enter会进入此分支！！！
        assert(THREAD->is_Java_thread(), "");
        markOop biased_value       = mark;
        //构建新的偏向本线程的markOop，设置好本线程的 ThreadID 、分代年龄、epoch值
        markOop rebiased_prototype = markOopDesc::encode((JavaThread*) THREAD, mark->age(), prototype_header->bias_epoch());
        //通过CAS 操作， 将尝试新markOop写入对象头中
        markOop res_mark = (markOop) Atomic::cmpxchg_ptr(rebiased_prototype, obj->mark_addr(), mark);
        //CAS成功，则返回撤销和重新偏向状态
        if (res_mark == biased_value) {
          return BIAS_REVOKED_AND_REBIASED;
        }
      } else {
        //不尝试获取偏向锁，则取消偏向锁
        markOop biased_value       = mark;
        markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
        markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
        //如果CAS操作成功，返回偏向锁撤销状态
        if (res_mark == biased_value) {
          return BIAS_REVOKED;
        }
      }
    }
  }
  ...
}
```

(是不是发现这个函数的偏向锁实现与前面的`biased_locking_enter(...)`又重复了？其实这是因为这个函数不仅仅是模版解释器会调用，字节码解释器也会执行这个函数。)?

`fast_enter`函数中调用了上述函数，最终进入```if (attempt_rebias)```分支，尝试获取偏向锁，先构建新的偏向本线程的markOop，设置好本线程 ThreadID 、分代年龄、epoch值，然后通过CAS 操作， 尝试将其写入对象头中。对于情况一的线程B来说，由于对象头的ThreadID为之前的线程A ID，这里会失败并继续执行后面的重偏向与撤销的逻辑。

```cpp
  //...接上一段代码

  //重偏向与撤销的逻辑
  //通过启发式的方式决定到底是执行撤销还是执行重偏向
  HeuristicsResult heuristics = update_heuristics(obj(), attempt_rebias);
  if (heuristics == HR_NOT_BIASED) {//决定不偏向
    return NOT_BIASED;
  } else if (heuristics == HR_SINGLE_REVOKE) {//决定撤销单个线程
    ...
    if (mark->biased_locker() == THREAD &&
        prototype_header->bias_epoch() == mark->bias_epoch()) {//需要撤销的是偏向当前线程的锁，当调用Object#hashcode方法时会走到这一步
      // 撤销逻辑1...
      ...
      return cond;
    } else {
      // 撤销逻辑2...
      ...
      return revoke.status_code();
    }
  }
  ...
  //批量撤销、批量重偏向的逻辑...
  ...
  return bulk_revoke.status_code();
```

代码逻辑很简单，就是通过通过启发式（Heuristics）的方式决定到底是执行撤销还是执行重偏向。我们先来看看这个启发式策略`update_heuristics()`：

```cpp
//启发式的方式决定要做哪种操作
static HeuristicsResult update_heuristics(oop o, bool allow_rebias) {
  markOop mark = o->mark();
  if (!mark->has_bias_pattern()) {
    //不可偏向直接返回
    return HR_NOT_BIASED;
  }
  //控制撤销的次数
  // Heuristics to attempt to throttle the number of revocations.
  // Stages:
  // 1. Revoke the biases of all objects in the heap of this type,
  //    but allow rebiasing of those objects if unlocked.
  // 2. Revoke the biases of all objects in the heap of this type
  //    and don't allow rebiasing of these objects. Disable
  //    allocation of objects of that type with the bias bit set.
  Klass* k = o->klass();
  jlong cur_time = os::javaTimeMillis();
  //获取上次执行bulk revication的时间
  jlong last_bulk_revocation_time = k->last_biased_lock_bulk_revocation_time();
  //获取执行bulk revocation的次数
  int revocation_count = k->biased_lock_revocation_count();
  //定义在globs.hpp,BiasedLockingBulkRebiasThreshold取值为20；BiasedLockingBulkRevokeThreshold取值为40,BiasedLockingDecayTime为25000毫秒，可通过JVM启动参数改变
  if ((revocation_count >= BiasedLockingBulkRebiasThreshold) &&
      (revocation_count <  BiasedLockingBulkRevokeThreshold) &&
      (last_bulk_revocation_time != 0) &&
      (cur_time - last_bulk_revocation_time >= BiasedLockingDecayTime)) {
    // This is the first revocation we've seen in a while of an
    // object of this type since the last time we performed a bulk
    // rebiasing operation. The application is allocating objects in
    // bulk which are biased toward a thread and then handing them
    // off to another thread. We can cope with this allocation
    // pattern via the bulk rebiasing mechanism so we reset the
    // klass's revocation count rather than allow it to increase
    // monotonically. If we see the need to perform another bulk
    // rebias operation later, we will, and if subsequently we see
    // many more revocation operations in a short period of time we
    // will completely disable biasing for this type.
    //在执行了一定时间之内，执行的撤销次数没有超过阈值，那么认为可以优先执行bulk rebias,因此将计数回归原始值
    k->set_biased_lock_revocation_count(0);
    revocation_count = 0;
  }

  // Make revocation count saturate just beyond BiasedLockingBulkRevokeThreshold
  if (revocation_count <= BiasedLockingBulkRevokeThreshold) {
    //对执行撤销的次数进行计数
    revocation_count = k->atomic_incr_biased_lock_revocation_count();
  }

  if (revocation_count == BiasedLockingBulkRevokeThreshold) {
    //达到执行bulk revoke的阈值，执行bulk revoke
    return HR_BULK_REVOKE;
  }

  if (revocation_count == BiasedLockingBulkRebiasThreshold) {
    //达到 bulk rebias的阈值，执行bulk rebias
    return HR_BULK_REBIAS;
  }
  //默认执行单次的撤销
  return HR_SINGLE_REVOKE;
}
```

通过代码中的注释可以了解到启发式策略的执行逻辑。//TODO...

再回到`revoke_and_rebias()`中重偏向与撤销的逻辑的分析：

```cpp
  //...接上一段代码

  //重偏向与撤销的逻辑
  //通过启发式的方式决定到底是执行撤销还是执行重偏向
  HeuristicsResult heuristics = update_heuristics(obj(), attempt_rebias);
  if (heuristics == HR_NOT_BIASED) {//决定不偏向
    return NOT_BIASED;
  } else if (heuristics == HR_SINGLE_REVOKE) {//决定撤销单个线程
    Klass *k = obj->klass();
    markOop prototype_header = k->prototype_header();
    if (mark->biased_locker() == THREAD &&
        prototype_header->bias_epoch() == mark->bias_epoch()) {// 走到这里说明需要撤销的是偏向当前线程的锁，当调用Object#hashcode方法时会走到这一步
      // 因为只要遍历当前线程的栈就好了，所以不需要等到safepoint再撤销。
      ResourceMark rm;
      if (TraceBiasedLocking) {
        tty->print_cr("Revoking bias by walking my own stack:");
      }
      BiasedLocking::Condition cond = revoke_bias(obj(), false, false, (JavaThread*) THREAD);
      ((JavaThread*) THREAD)->set_cached_monitor_info(NULL);
      assert(cond == BIAS_REVOKED, "why not?");
      return cond;
    } else {// 下面代码最终会在VM线程中的safepoint调用revoke_bias方法
      VM_RevokeBias revoke(&obj, (JavaThread*) THREAD);
      VMThread::execute(&revoke);
      return revoke.status_code();
    }
  }

  assert((heuristics == HR_BULK_REVOKE) ||
         (heuristics == HR_BULK_REBIAS), "?");
  //批量撤销、批量重偏向的逻辑
  VM_BulkRevokeBias bulk_revoke(&obj, (JavaThread*) THREAD,
                                (heuristics == HR_BULK_REBIAS),
                                attempt_rebias);
  VMThread::execute(&bulk_revoke);
  return bulk_revoke.status_code();
```

对于情况一，我们假设这是被锁对象头一次发生偏向撤销，因此B线程调用`update_heuristics(...)`时会返回`HR_SINGLE_REVOKE`，从而进入```else if (heuristics == HR_SINGLE_REVOKE)```分支里的`else`分支，通过调用```VMThread::execute(&revoke)```最终在VM线程中的safepoint调用`revoke_bias`方法。

```cpp
void VMThread::execute(VM_Operation* op) {
  Thread* t = Thread::current();
  //如果不是虚拟机线程，即普通Java线程
  if (!t->is_VM_thread()) {
    ...
    // 进行前置处理，根据返回值确定是否继续
    if (!op->doit_prologue()) {
      return;   // op was cancelled
    }
    ...
    // Add VM operation to list of waiting threads. We are guaranteed not to block while holding the
    // VMOperationQueue_lock, so we can block without a safepoint check. This allows vm operation requests
    // to be queued up during a safepoint synchronization.
    //添加到VM operation等待执行
    {
      VMOperationQueue_lock->lock_without_safepoint_check();
      bool ok = _vm_queue->add(op);
    op->set_timestamp(os::javaTimeMillis());
      VMOperationQueue_lock->notify();
      VMOperationQueue_lock->unlock();
      // VM_Operation got skipped
      if (!ok) {
        assert(concurrent, "can only skip concurrent tasks");
        if (op->is_cheap_allocated()) delete op;
        return;
      }
    }
    if (execute_epilog) {
      //  进行后置处理
      op->doit_epilogue();
    }
  } else {//如果已经是虚拟机线程
    // invoked by VM thread; usually nested VM operation
    ...
    op->evaluate();
    ...
  }
}
```

evaluate()会回调VM_Operation类的doit()虚函数：

```cpp
//  回调doit()
void VM_Operation::evaluate() {
  ...
  doit();
  ...
}
```

VM_RevokeBias继承了VM_Operation并重写了doit_prologue()和doit()两个虚函数，doit()中完成了撤销偏向锁：

```cpp
//  VM_RevokeBias继承了VM_Operation
//  并重写了doit_prologue()和doit()两个虚函数
class VM_RevokeBias : public VM_Operation {
...
public:
...
  virtual VMOp_Type type() const { return VMOp_RevokeBias; }

  // 如vm_operations.hpp注释描述的，若返回false则取消该VM操作
  virtual bool doit_prologue() {
    // Verify that there is actual work to do since the callers just
    // give us locked object(s). If we don't find any biased objects
    // there is nothing to do and we avoid a safepoint.
    //  若被锁对象为空，则返回true继续后续逻辑即通过evaluate()回调doit()
    if (_obj != NULL) {
      markOop mark = (*_obj)()->mark();
      if (mark->has_bias_pattern()) {
        return true;
      }
    } else {
      for ( int i = 0 ; i < _objs->length(); i++ ) {
        //  遍历被锁对象，若存在可偏向状态，返回true继续后续逻辑即通过evaluate()回调doit()
        markOop mark = (_objs->at(i))()->mark();
        if (mark->has_bias_pattern()) {
          return true;
        }
      }
    }
    // 没有可偏向状态，返回true继续后续逻辑即通过evaluate()回调doit()
    return false;
  }

  virtual void doit() {
    //  被锁对象不为空
    if (_obj != NULL) {
      _status_code = revoke_bias((*_obj)(), false, false, _requesting_thread);
      ...
      return;
    } else {
      ...
    }
  }

  BiasedLocking::Condition status_code() const {
    return _status_code;
  }
};
```

doit()又调用了revoke_bias：

```cpp
static BiasedLocking::Condition revoke_bias(oop obj, bool allow_rebias, bool is_bulk, JavaThread* requesting_thread) {
  //  获取对象头
  markOop mark = obj->mark();
  if (!mark->has_bias_pattern()) {
    ...
    //  不是可偏向状态，返回
    return BiasedLocking::NOT_BIASED;
  }

  uint age = mark->age();
  //新建可偏向锁的头：分代年龄为age、可偏向(101)状态
  markOop   biased_prototype = markOopDesc::biased_locking_prototype()->set_age(age);
  //新建非偏向锁的头:无hash、分代年龄为age、无锁(001)状态
  markOop unbiased_prototype = markOopDesc::prototype()->set_age(age);
  ....
  //获取偏向的线程
  JavaThread* biased_thread = mark->biased_locker();
  if (biased_thread == NULL) {
    // Object is anonymously biased. We can get here if, for
    // example, we revoke the bias due to an identity hash code
    // being computed for an object.
    //  没有偏向线程，identity hash code的计算会导致进入此分支
    if (!allow_rebias) {
      //  设置对象头为前面构建的非偏向头:无hash、分代年龄为age、无锁(001)状态
      obj->set_mark(unbiased_prototype);
    }
    //  偏向锁撤销完成
    return BiasedLocking::BIAS_REVOKED;
  }

  // Handle case where the thread toward which the object was biased has exited
  //对象头偏向线程是否活着
  bool thread_is_alive = false;
  if (requesting_thread == biased_thread) {
    //当前请求线程==对象头偏向线程，说明偏向线程活着
    thread_is_alive = true;
  } else {
    //  遍历存活线程队列，看看能不能找到对象头偏向线程
    for (JavaThread* cur_thread = Threads::first(); cur_thread != NULL; cur_thread = cur_thread->next()) {
      if (cur_thread == biased_thread) {
        //  能找到说明对象头的偏向线程活着
        thread_is_alive = true;
        break;
      }
    }
  }
  //  如果对象头偏向线程已退出，则撤销偏向锁
  if (!thread_is_alive) {
    if (allow_rebias) {
      //  设置可偏向锁头:分代年龄为age、可偏向(101)状态
      obj->set_mark(biased_prototype);
    } else {//由于传入该函数的allow_rebias==false，故走此分支
      //  设置非偏向锁头:无hash、分代年龄为age、无锁(001)状态
      obj->set_mark(unbiased_prototype);
    }
    //  偏向锁撤销完成
    return BiasedLocking::BIAS_REVOKED;
  }

  // 注释解释了逻辑：
  // Thread owning bias is alive.
  // Check to see whether it currently owns the lock and, if so,
  // write down the needed displaced headers to the thread's stack.
  // Otherwise, restore the object's header either to the unlocked
  // or unbiased state.

  //get_or_compute_monitor_info返回对象头偏向线程的所有的被锁住的对象(轻量级锁)的监视器信息(Lock Record)
  GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(biased_thread);
  BasicLock* highest_lock = NULL;
  //在偏向线程的线程栈中寻找当前被(轻量级)锁对象(obj)对应的displaced_header(一种markOop)，
  //则需要重新设置其对象头markOop，这里先将其设为NULL
  for (int i = 0; i < cached_monitor_info->length(); i++) {
    MonitorInfo* mon_info = cached_monitor_info->at(i);
    if (mon_info->owner() == obj) {
      // Assume recursive case and fix up highest lock later
      markOop mark = markOopDesc::encode((BasicLock*) NULL);
      highest_lock = mon_info->lock();
      //设置栈中的displaced_header为NULL  
      highest_lock->set_displaced_header(mark);
    }
    ...
  }
  //  如果上面的过程找到了displaced_header，则highest_lock不为空
  //  说明当前撤销偏向的对象正在被偏向头里的线程锁定，进入下面分支，
  //  对正在被(轻量级)锁定的对象进行偏向锁撤销
  if (highest_lock != NULL) {
    // Fix up highest lock to contain displaced header and point
    // object at it
    // 如上面注释所言，重新设置线程栈上的displaced header，用unbiased_prototype
    highest_lock->set_displaced_header(unbiased_prototype);
    // Reset object header to point to displaced mark.
    // 使对象头(mark)指向这个displaced header
    obj->release_set_mark(markOopDesc::encode(highest_lock));
    ...
    //对正在被(轻量级)锁定的对象进行偏向锁撤销完成
  } else {//  如果上面的过程没找displaced_header，说明当前撤销偏向锁
          //  的对象目前没有被(轻量级)锁定，进入这个分支，对未锁定对象进行偏向撤销
    if (allow_rebias) {
      obj->set_mark(biased_prototype);
    } else {//由于传入该函数的allow_rebias==false，故走此分支
      // Store the unlocked value into the object's header.
      obj->set_mark(unbiased_prototype);
      //对未锁定对象进行偏向锁撤销完成，也表明对象成为了:无hash、分代年龄为age、无锁(001)状态
    }
  }
  //  偏向撤销完成
  return BiasedLocking::BIAS_REVOKED;
}
```

这里解释一下，轻量级锁在虚拟机内部，使用一个成为BasicObjectLock的对象实现的，这个对象内部由一个BasicLock对象和一个持有该锁的Java对象指针组成。BasicObjectLock对象放置在Java栈帧中。在BasicLock对象内部还维护着displaced_header字段，用于备份对象头部的Mark Word。还记得吗，当一个线程持有一个对象的轻量级锁的时候，对象头部Mark Word信息如下：

```cpp
//    [ptr             | 00]  locked  
```

末尾的两位比特为00，整个Mark Word为指向BasicLock对象的指针。由于BasicObjectLock对象在线程栈中，因此该指针必然指向持有该锁的线程栈空间。当需要判断一个线程是否持有该对象时，只需要简单地判断对象头的指针是否在当前线程的栈地址范围即可。同时，BasicLock对象的displaced_header，备份了原对象的Mark word内容，BasicObjectLock对象的obj字段则指向持有锁的对象头部。

前面说到B线程最总调用了revoke_bias进行偏向锁撤销（以及升级为轻量级锁）。从上面的代码我们看出，由于被锁对象还不是轻量级锁，线程栈中没有它对应的Lock Record，又由于传入该函数的allow_rebias==false，所以在这里会进入下面的如下分支：

```cpp
...
else {//由于传入该函数的allow_rebias==false，故走此分支
      // Store the unlocked value into the object's header.
      obj->set_mark(unbiased_prototype);
      //对未锁定对象进行偏向锁撤销完成，也表明对象成为了:从而成为了无hash、分代年龄为age、无锁(001)状态。
    }
  }
  //  偏向撤销完成
  return BiasedLocking::BIAS_REVOKED;
```

从而成为了无hash、分代年龄为age、无锁(001)状态。至此情况一的分析就完成了。对于情况二，即B进入时A还未释放锁的情况，请读者对照上面的源码自行分析。
(//TODO
~~与情况一类似，线程B使用VMThread::execute()执行VM_RevokeBias中的doit()方法->revoke_bias()，最终返回的status_code为BIAS_REVOKED，于是在fast_enter中没有进入```if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED)```分支返回，fallback到了slow_enter中。~~)

> 前文提到，若对象计算并保存了hashCode到对象的Mark Word中，会导致该对象无法被偏向锁锁定，[Oracle这篇文章](https://blogs.oracle.com/dave/biased-locking-in-hotspot)就有说到。而且，如果该对象已经偏向，如果如果我们没有重写默认的hashCode方法，默认的hashCode要通过`ObjectSynchronizer::FastHashCode`函数计算identity hashcode，该函数会调用revoke_and_rebias，这将导致偏向锁的撤销。
>
> ```cpp
> intptr_t ObjectSynchronizer::FastHashCode (Thread * Self, oop obj) {
>  if (UseBiasedLocking) {
>   //  如果是可偏向状态
>   if (obj->mark()->has_bias_pattern()) {
>     ...
>     BiasedLocking::revoke_and_rebias(hobj, false, JavaThread::current());
>     ...
> ```
>
> 通过`FastHashCode`实现我们发现，调用`revoke_and_rebias`时会进入```if (mark->is_biased_anonymously() && !attempt_rebias)```分支，从而撤销偏向锁。读者可以自行[编写代码验证](https://blog.csdn.net/P19777/article/details/103125545)。

### inflate 成为重锁

从 fast_enter 函数中可以看到，我们可能会执行到 slow_enter 函数中：

```cpp
// Interpreter/Compiler Slow Case
// This routine is used to handle interpreter/compiler slow case
// We don't need to use fast path here, because it must have been
// failed in the interpreter/compiler code.
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");
  if (mark->is_neutral()) {//如果当前是无锁状态, markword的biase_lock:0，lock:01
    // Anticipate successful CAS -- the ST of the displaced mark must
    // be visible <= the ST performed by the CAS.
    //直接把mark保存到BasicLock对象的_displaced_header字段
    lock->set_displaced_header(mark);
    if (mark == obj()->cas_set_mark((markOop) lock, mark)) {
      TEVENT(slow_enter: release stacklock);
      return;
    }
    // Fall through to inflate() ...
  } else if (mark->has_locker() &&
             THREAD->is_lock_owned((address)mark->locker())) {//如果markword处于加锁状态、且markword中的ptr指针指向当前线程的栈帧，表示为重入操作，不需要争抢锁
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    lock->set_displaced_header(NULL);
    return;
  }
  // The object header will never be displaced to this lock,
  // so it does not matter what the value is, except that it
  // must be non-zero to avoid looking like a re-entrant lock,
  // and must not look locked either.
  lock->set_displaced_header(markOopDesc::unused_mark());
  ObjectSynchronizer::inflate(THREAD,
                              obj(),
                              inflate_cause_monitor_enter)->enter(THREAD);
}
```

这里的执行逻辑比较简洁，主要执行上面 OpenJDK wiki 中的锁优化逻辑。首先会判断对象锁是否为中立的（neutral）：

```cpp
bool is_neutral()  const {
  // 这里的 biased_lock_mask_in_place 是 7, 即二进制111
  // unlocked_value 值是 1
  return (mask_bits(value(), biased_lock_mask_in_place) == unlocked_value);
}
```

这里的判断也是比较简单的，就是将 mark word 中的最后 3 个 bit 进行掩码运算，将得到的值和 1(unlocked_value) 进行比较，如果等于 1 就表示对象是中立的，也就是没有被任何线程锁定，否则就失败。这里需要问一个问题，那就是为什么我们要对 mark word 的最后的 3 个 bit 进行掩码运算？这里我们就需要再次看下在 biase 模式下的对象 mark word 的布局（这里以 64 bit 为例，仍然是上面的 oopDesc 注释描述）：

```cpp
unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
```

可以看到，无论是普通对象或者是可偏置的对象，最后 3 个 bit 的格式是固定的，我们再次看下上面 OpenJDK wiki 中的锁优化图，会发现在普通对象的时候，也就是 biase revoke 时 unlock 状态下的 header 最后三个 bit 就是 001，也就是十进制的 1！所以这里通过简单高效的二进制运算就获得了对象的锁定状态。

再次回到上面的 slow_enter 函数，如果判断为中立的，也就是没有锁定的话，会执行：

```cpp
if (mark->is_neutral()) {
    // Anticipate successful CAS -- the ST of the displaced mark must
    // be visible <= the ST performed by the CAS.
    lock->set_displaced_header(mark);
    if (mark == obj()->cas_set_mark((markOop) lock, mark)) {
      TEVENT(slow_enter: release stacklock);
      return;
    }
    // Fall through to inflate() ...
}
```

首先将当前的 mark word，存储到 lock 指针指向的对象中，这里的 lock 指针指向的就是上面提到的 lock record。然后进行一个非常重要的操作，就是通过原子 cas 操作将这个 lock 指针安装到对象 mark word 中，如果安装成功就表示当前线程获得了这个对象锁，可以直接返回执行同步代码块了，否则就会 fall back 到膨胀锁中，正如注释说的那样。

上面是判断对象是否为中立的逻辑，如果当线程进来的发现当前的对象锁已经被另外一个线程锁定了。这个时候就会执行到 else 逻辑中：

```cpp
if (mark->has_locker() &&
             THREAD->is_lock_owned((address)mark->locker())) {
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    lock->set_displaced_header(NULL);
    return;
}
```

如果发现当前对象已经锁定，需要判断下是不是当前线程自己锁定了，因为在 synchronized 中可能再一次 synchronized，这种情况下就直接返回即可。

如果上面的两个判断都失败了，也就是对象被锁定，并且锁定线程不是当前线程，这个时候需要执行上面 OpenJDK wiki 中的 inflate 膨胀逻辑。所谓膨胀，就是根据当前锁对象，生成一个 ObjectMonitor 对象，这个对象中保存了 sychronized 阻塞的队列，以及实现了不同的队列调度策略。
ObjectMonitor 的构造函数如下：

```cpp
class ObjectMonitor {
  ...
 public:
  // initialize the monitor, exception the semaphore, all other fields
  // are simple integers or pointers
  ObjectMonitor() {
    _header       = NULL;   // displaced object header word - mark
    _count        = 0;      // reference count to prevent reclaimation/deflation
                            // at stop-the-world time.  See deflate_idle_monitors().
                            // _count is approximately |_WaitSet| + |_EntryList|
    _waiters      = 0,      // number of waiting threads
    _recursions   = 0;      // recursion count, 0 for first entry
    _object       = NULL;   // backward object pointer - strong root
    _owner        = NULL;   // pointer to owning thread OR BasicLock
    _WaitSet      = NULL;   // LL of threads wait()ing on the monitor
    _WaitSetLock  = 0 ;     // protects Wait Queue - simple spinlock
    _Responsible  = NULL ;
    _succ         = NULL ;  // Heir presumptive thread - used for futile wakeup throttling
    _cxq          = NULL ;  // LL of recently-arrived threads blocked on entry.
                            // The list is actually composed of WaitNodes, acting
                            // as proxies for Threads.
    FreeNext      = NULL ;
    _EntryList    = NULL ;  // Threads blocked on entry or reentry.
    _SpinFreq     = 0 ;     // Spin 1-out-of-N attempts: success rate
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;     // _owner is (Thread *) vs SP/BasicLock
    _previous_owner_tid = 0;  // thread id of the previous owner of the monitor
  }
  ...
}
  ```

下面我们重点看下 ObjectMonitor 中的 enter 逻辑（inflate 逻辑不是十分复杂，但是代码量较大，主要是在判断一堆 obj 的状态和检查，这里就不再分析了）。

### ObjectMonitor enter

在 enter 函数中，有很多判断和优化执行的逻辑，但是核心是通过 **EnterI** 函数实际进入队列将当前线程阻塞：

```cpp
void ObjectMonitor::EnterI(TRAPS) {
  ...
  // Try the lock - TATAS
  if (TryLock (Self) > 0) {
    assert(_succ != Self, "invariant");
    assert(_owner == Self, "invariant");
    assert(_Responsible != Self, "invariant");
    return;
  }
  ...
  // We try one round of spinning *before* enqueueing Self.
  //
  // If the _owner is ready but OFFPROC we could use a YieldTo()
  // operation to donate the remainder of this thread's quantum
  // to the owner.  This has subtle but beneficial affinity
  // effects.
  if (TrySpin (Self) > 0) {
    assert(_owner == Self, "invariant");
    assert(_succ != Self, "invariant");
    assert(_Responsible != Self, "invariant");
    return;
  }
  ...
  ObjectWaiter node(Self);
  // Push "Self" onto the front of the _cxq.
  // Once on cxq/EntryList, Self stays on-queue until it acquires the lock.
  // Note that spinning tends to reduce the rate at which threads
  // enqueue and dequeue on EntryList|cxq.
  ObjectWaiter * nxt;
  for (;;) {
    node._next = nxt = _cxq;
    if (Atomic::cmpxchg(&node, &_cxq, nxt) == nxt) break;
    // Interference - the CAS failed because _cxq changed.  Just retry.
    // As an optional optimization we retry the lock.
    if (TryLock (Self) > 0) {
      assert(_succ != Self, "invariant");
      assert(_owner == Self, "invariant");
      assert(_Responsible != Self, "invariant");
      return;
    }
  }
  ...
  for (;;) {
    if (TryLock(Self) > 0) break;
    ...
    if ((SyncFlags & 2) && _Responsible == NULL) {
      Atomic::replace_if_null(Self, &_Responsible);
    }
    // park self
    if (_Responsible == Self || (SyncFlags & 1)) {
      TEVENT(Inflated enter - park TIMED);
      Self->_ParkEvent->park((jlong) recheckInterval);
      // Increase the recheckInterval, but clamp the value.
      recheckInterval *= 8;
      if (recheckInterval > MAX_RECHECK_INTERVAL) {
        recheckInterval = MAX_RECHECK_INTERVAL;
      }
    } else {
      TEVENT(Inflated enter - park UNTIMED);
      Self->_ParkEvent->park();
    }
    if (TryLock(Self) > 0) break;
    ...
  }
  ...
  if (_Responsible == Self) {
    _Responsible = NULL;
  }
  // 善后处理，比如将当前线程从等待队列 CXQ 中移除
  ...
}
```

这个函数比较复杂，代码很长，上面将重点核心逻辑列出来了。上面的逻辑上来先执行了 TryLock：

```cpp
int ObjectMonitor::TryLock(Thread * Self) {
  void * own = _owner;
  if (own != NULL) return 0;
  if (Atomic::replace_if_null(Self, &_owner)) {
    // Either guarantee _recursions == 0 or set _recursions = 0.
    assert(_recursions == 0, "invariant");
    assert(_owner == Self, "invariant");
    return 1;
  }
  // The lock had been free momentarily, but we lost the race to the lock.
  // Interference -- the CAS failed.
  // We can either return -1 or retry.
  // Retry doesn't make as much sense because the lock was just acquired.
  return -1;
}
```

这里的逻辑很简单，主要是尝试通过 cas 操作将 _owner 字段设置为 Self，其中 _owner 表示当前 ObjectMonitor 对象锁持有的线程指针，Self 指向当前执行的线程。如果设置上了，表示当前线程获得了锁，否则没有获得。

再回到上面的 EnterI 函数中，我们看到 TryLock 前后连续执行了两次，而且代码判断逻辑一样，为什么要这样？这其实是为了在入队阻塞线程之前的最后检查，防止线程无谓地进行状态切换。但是为什么执行两次？其实第二次执行的注释已经说明了，这么做有一些微妙的亲和力影响？什么是亲和力？这是很多操作系统中都有的一种 CPU 执行调度逻辑，说的是，如果在过去一段时间内，某个线程尝试获取某种资源一直失败，那么系统在后面会倾向于将该资源分配给这个线程。这里我们前后两次执行，就是告诉系统当前线程「迫切」想要获得这个 cas 资源，如果可以用的话尽量分配给它。当然这种亲和力不是一种得到保证的协议，因此这种操作只能是一种积极的、并且人畜无害的操作。

如果上面进行了两次「微妙」的 try lock 之后仍然失败，那么就只能乖乖入队阻塞了。在入队之前需要创建一个 ObjectWaiter 对象，这个对象将当前线程的对象（注意是 JavaThread 对象）包裹起来，我们看下 ObjectWaiter 的定义：

```cpp
class ObjectWaiter : public StackObj {
 public:
  enum TStates { TS_UNDEF, TS_READY, TS_RUN, TS_WAIT, TS_ENTER, TS_CXQ };
  enum Sorted  { PREPEND, APPEND, SORTED };
  ObjectWaiter * volatile _next;
  ObjectWaiter * volatile _prev;
  Thread*       _thread;
  jlong         _notifier_tid;
  ParkEvent *   _event;
  volatile int  _notified;
  volatile TStates TState;
  Sorted        _Sorted;           // List placement disposition
  bool          _active;           // Contention monitoring is enabled
 public:
  ObjectWaiter(Thread* thread);
  void wait_reenter_begin(ObjectMonitor *mon);
  void wait_reenter_end(ObjectMonitor *mon);
};
```

如果你看到 _next 和 _prev 你就会立即明白，这是需要使用双向队列实现等待队列的节奏（但是实际上，下面入队操作并没有形成双向链表，真正形成双向链表时在 exit 的时候，下面分析 exit 的时候会看到这个逻辑）。node 节点创建完毕之后会执行如下入队操作（为了方便大家阅读，我把上面的入队逻辑 copy 过来了）：

```cpp
// Push "Self" onto the front of the _cxq.
// Once on cxq/EntryList, Self stays on-queue until it acquires the lock.
// Note that spinning tends to reduce the rate at which threads
// enqueue and dequeue on EntryList|cxq.
ObjectWaiter * nxt;
for (;;) {
  node._next = nxt = _cxq;
  if (Atomic::cmpxchg(&node, &_cxq, nxt) == nxt) break;
  // Interference - the CAS failed because _cxq changed.  Just retry.
  // As an optional optimization we retry the lock.
  if (TryLock (Self) > 0) {
    assert(_succ != Self, "invariant");
    assert(_owner == Self, "invariant");
    assert(_Responsible != Self, "invariant");
    return;
  }
}
```

正如注释中说的那样，我们是要将当前节点放到 CXQ 队列的头部，将节点的 next 指针通过 cas 操作指向 _cxq 指针就完成了入队操作。如果入队成功，则退出当前循环，否则再次尝试 lock，因为可能这个时候会成功。这里使用循环操作 cas 的逻辑，就是处理在高并发的状态下 cas 锁定失败问题，这一点和 JUC 中的 atomic 类的很多 update 操作是一致的。

如果上面的循环退出了，就表示当前线程的 node 节点已经顺利进入 CXQ 队列了，那么接下来需要进入另外一个循环：

```cpp
for (;;) {
    if (TryLock(Self) > 0) break;
    ...
    if ((SyncFlags & 2) && _Responsible == NULL) {
      Atomic::replace_if_null(Self, &_Responsible);
    }
    // park self
    if (_Responsible == Self || (SyncFlags & 1)) {
      TEVENT(Inflated enter - park TIMED);
      Self->_ParkEvent->park((jlong) recheckInterval);
      // Increase the recheckInterval, but clamp the value.
      recheckInterval *= 8;
      if (recheckInterval > MAX_RECHECK_INTERVAL) {
        recheckInterval = MAX_RECHECK_INTERVAL;
      }
    } else {
      TEVENT(Inflated enter - park UNTIMED);
      Self->_ParkEvent->park();
    }
    if (TryLock(Self) > 0) break;
    ...
}
```

这个循环中的逻辑比较简单，一眼就能看明白。主要执行三件事：

1. 尝试获取锁
2. park 当前线程
3. 再次尝试获取锁

重点在于第 2 步，我们知道 synchronzed 如果获取对象锁失败的话，会导致当前线程被阻塞，那么这个阻塞操作就是在这里完成的。这里需要注意的是，这里需要判断一下 _Responsible 指针，如果这个指针为 NULL，表示之前对象锁还没有等待线程，也就是说当前线程是第一个等待线程，这个时候通过 cas 操作将 _Responsible 指向 Self，表示当前线程是这个对象锁的等待线程。接下来，如果发现当前线程就是等待线程或者不是等待线程，那么执行的的逻辑是不一样的。如果是当前线程是等待线程，那么会执行一个简单的「退避算法」，进行一个短时间的阻塞等待。这个算法很简单，第一次等待 1 ms，第二次等待 8 ms，第三次等待 64 ms，以此类推，直到达到等待时长的上线：MAX_RECHECK_INTERVAL，这个 MAX_RECHECK_INTERVAL 的值默认是 1000 ms，也就是说在 synchronize 在一个对象锁上的线程，如果他是第一个等待线程的话，那么他会不停滴休眠、检查锁，休眠的时间由刚才的退避算法指定。如果当前线程不是第一个等待线程，那么只能执行无限期的休眠，一直等待对象锁的 exit 函数执行唤醒才行，这一点在下面分析 exit 函数的时候会说明。

这里有一个问题，就是 park 函数是怎么实现线程的阻塞的？我们看下 park 的实现：

![park_impl](/assets/posts/park_impl.png)

一共有三个实现，从名字可以看出来实现的方式。通常情况下，我们关心在 linux 上的 posix 实现方式：

```cpp
void os::PlatformEvent::park() {
  ...
  int status = pthread_mutex_lock(_mutex);
  ...
  status = pthread_cond_wait(_cond, _mutex);
  ...
  status = pthread_mutex_unlock(_mutex);
}
```

从这里，我们一下子就明白这里的实现方法，其实就是通过 pthread 的 condition wait 函数实现 pthread 线程的等待。

当线程获得了锁可以进入时，也就是上面的循环退出来的时候，这个时候需要执行一个重要的操作，就是将 _Responsible 置为 NULL：

```cpp
if (_Responsible == Self) {
    _Responsible = NULL;
}
```

这样以来，当下个线程唤醒的时候，发现 _Responsible 为 NULL 就会尝试上面的 cas 方式将自己标注为第一个等待线程，这样就可以重复上面的操作，完成 monitor 锁的交接。

### ObjectMonitor exit

上面我们理清了 ObjectMonitor enter 的逻辑，我们知道了如下几件事情：

1. ObjectMonitor 内部通过一个 CXQ 队列保存所有的等待线程
2. 在实际进入队列之前，会反复尝试 lock，在某些系统上会存在 CPU 亲和力的优化
3. 入队的时候，通过 ObjectWaiter 对象将当前线程包裹起来，并且入到 CXQ 队列的头部
4. 入队成功以后，会根据当前线程是否为第一个等待线程做不同的处理
5. 如果是第一个等待线程，会根据一个简单的「退避算法」来有条件的 wait
6. 如果不是第一个等待线程，那么会执行无限期等待
7. 线程的 park 在 posix 系统上是通过 pthread 的 condition wait 实现的

当一个线程获得对象锁成功之后，就可以执行自定义的同步代码块了。执行完成之后会执行到 ObjectMonitor 的 exit 函数中，释放当前对象锁，方便下一个线程来获取这个锁，下面我们逐步分析下 exit 的实现过程。

exit 函数的实现比较长，但是整体上的结构比较清晰：

```cpp
void ObjectMonitor::exit(bool not_suspended, TRAPS) {
  for (;;) {
    ...
        ObjectWaiter * w = NULL;
    int QMode = Knob_QMode;

    if (QMode == 2 && _cxq != NULL) {
        ...
    }

    if (QMode == 3 && _cxq != NULL) {
        ...
    }

    if (QMode == 4 && _cxq != NULL) {
        ...
    }
    ...
    ExitEpilog(Self, w);
    return;
  }
}
```

上面的 exit 函数整体上分为如下几个部分：

1. 根据 Knob_QMode 的值和 _cxq 是否为空执行不同策略
2. 根据一定策略唤醒等待队列中的下一个线程

下面我们分两部分分析 exit 逻辑。

#### 出队策略 0——默认策略

在 exit 函数中首先是根据 Knob_QMode 值的不同执行不同逻辑，首先我们看下这个值的默认值：

```cpp
// EntryList-cxq policy - queue discipline
static int Knob_QMode = 0;
```

这个值默认是 0，但是这个值本身表示什么含义呢？从这个值的注释中可以看出，这个值是用来指定在退出时的 EntryList 和 CXQ 队列出队策略的，CXQ 我们知道是 enter 的时候因为锁已经被别的线程阻塞而进不来的线程，但是 EntryList 是什么呢？如果你了解 Java Object 的 wait 和 notify，你就会知道这是 notify 唤醒 wait 线程准备执行时，被唤醒线程加入的队列就是 EntryList。不过没关系，下面我们会单独详细分析 Object 的 wait 和 notify 的实现～那么回到我们的问题，Knob_QMode 这个变量主要用来指定在 exit 的时候 EntryList 和 CXQ 队列之间的唤醒关系，也就是说，当 EntryList 和 CXQ 中都有等待的线程时，因为 exit 之后只能有一个线程得到锁，这个时候选择唤醒哪个队列中的线程是一个值得考虑的事情。因此这里就有很多中策略可以执行，这里的默认策略就是 0。下面我们分别分析一下不同策略下的不同行为。

首先我们看下 JVM 默认的策略，也就是策略 0，这也是大家手上 JVM 的行为表现。在实际分析 JVM 的实现之前，我们先看下一段 java 代码：

```java
Thread t3 = new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Thread 3 start!!!!!!");
        synchronized (lock) {
            try {
                System.in.read();
            } catch (Exception e) {
            }
            System.out.println("Thread 3 end!!!!!!");
        }
    }
});

Thread t4 = new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("Thread 4 start!!!!!!");
        synchronized (lock) {
            System.out.println("Thread 4 end!!!!!!");
        }
    }
});

Thread t5 = new Thread(new Runnable() {
    @Override
    public void run() {
        int a = 0;
        System.out.println("Thread 5 start!!!!!!");
        synchronized (lock) {
            a++;
            System.out.println("Thread 5 end!!!!!!");
        }
    }
});

Thread t6 = new Thread(new Runnable() {
    @Override
    public void run() {
        int a = 0;
        System.out.println("Thread 6 start!!!!!!");
        synchronized (lock) {
            a++;
            System.out.println("Thread 6 end!!!!!!");
        }
    }
});

t3.start();
try {
    TimeUnit.SECONDS.sleep(1);
} catch (Exception e) {
}
t4.start();
try {
    TimeUnit.SECONDS.sleep(1);
} catch (Exception e) {
}
t5.start();
try {
    TimeUnit.SECONDS.sleep(1);
} catch (Exception e) {
}
t6.start();
```

上面的代码很简单，t3 线程首先启动，首先抢占 lock 锁，然后在同步块中调用 system.in 的 read 方法，这个方法从键盘获取一个输入，在没有得到输入之前会一直等待，注意此时没有放弃 monitor 锁，这一点和 wait 不同，当我按下键盘回车的时候，会唤醒 t3 线程并结束执行释放 lock 锁。t3 启动之后，t4、t5、t6相继启动。那么当我按下键盘会输出如下内容：

```bash
Thread 3 start!!!!!!
Thread 4 start!!!!!!
Thread 5 start!!!!!!
Thread 6 start!!!!!!

Thread 3 end!!!!!!
Thread 6 end!!!!!!
Thread 5 end!!!!!!
Thread 4 end!!!!!!
```

这个结果详细如果你了解 synchronized 的机制都不会陌生，看起来后来启动的线程优先获得 lock，产生了一种「后来者居上」的效果。但事实上为什么会这样呢？下面我们从 JVM 底层代码中寻找答案。我们前面在分析 enter 函数的时候，说到了，如果当前对象锁被另外一个线程锁锁定了，那么就需要将当前线程包装进 ObjectWaiter 对象中，然后插入到 CXQ 队列的头部（希望你没有晕掉，还记得这些内容～）。上面的代码中，在我按下回车键之前的 CXQ 队列如下：

![java_concurrency_cxq_1](/assets/posts/java_concurrency_cxq_1.jpg)

当 t6 线程启动之后，会进入 cxq 队列，此时 cxq 队列会指向 t6 的 wait 节点，现在大家记住这张图，下面我们开始看下 exit 中针对默认状态下，也就是 mode 等于 0 的时候的逻辑：

```cpp
void ObjectMonitor::exit(bool not_suspended, TRAPS) {
   for (;;) {
     ...
     ObjectWaiter * w = NULL;
     int QMode = Knob_QMode;
     ...
     w = _cxq;
     ...
     _EntryList = w;
     ObjectWaiter * q = NULL;
     ObjectWaiter * p;
     for (p = w; p != NULL; p = p->_next) {
        guarantee(p->TState == ObjectWaiter::TS_CXQ, "Invariant");
        p->TState = ObjectWaiter::TS_ENTER;
        p->_prev = q;
        q = p;
     }
     ...
     w = _EntryList;
     if (w != NULL) {
       guarantee(w->TState == ObjectWaiter::TS_ENTER, "invariant");
       ExitEpilog(Self, w);
       return;
     }
   }
}
```

因为我们只关注 mode == 0 的情况，因此 exit 函数的行为就变得非常简单。首先将 cxq 指针赋值给 _EntryList，然后通过一个循环将原本单向链表的 cxq 链表变成双向链表，这么做的目的就是为了方便后面针对 cxq 链表进行查询。还记得前面我们分析 enter 的时候，说到了虽然 ObjectWaiter 时一个双向链表的节点，但是我们在 enter 的时候并没有形成双向链表，当时我们提到了说双向链表的形成会在 exit 时形成，其实说的就是这里。

在上面的链表操作完毕之后，会将 _EntryList 中（注意 _EntryList 其实就是 cxq，因为前面赋值了）队头的节点传递给 ExitEpilog 函数：

```cpp
void ObjectMonitor::ExitEpilog(Thread * Self, ObjectWaiter * Wakee) {
  assert(_owner == Self, "invariant");
  // Exit protocol:
  // 1. ST _succ = wakee
  // 2. membar #loadstore|#storestore;
  // 2. ST _owner = NULL
  // 3. unpark(wakee)
  _succ = Knob_SuccEnabled ? Wakee->_thread : NULL;
  ParkEvent * Trigger = Wakee->_event;
  // Hygiene -- once we've set _owner = NULL we can't safely dereference Wakee again.
  // The thread associated with Wakee may have grabbed the lock and "Wakee" may be
  // out-of-scope (non-extant).
  Wakee  = NULL;
  // Drop the lock
  OrderAccess::release_store(&_owner, (void*)NULL);
  OrderAccess::fence();                               // ST _owner vs LD in unpark()
  if (SafepointMechanism::poll(Self)) {
    TEVENT(unpark before SAFEPOINT);
  }
  DTRACE_MONITOR_PROBE(contended__exit, this, object(), Self);
  Trigger->unpark();
  // Maintain stats and report events to JVMTI
  OM_PERFDATA_OP(Parks, inc());
}
```

这里的实现非常简单，其实就是通过 park event 将等待的那个线程唤醒，通过执行 unpark 函数（前面入队之后的休眠操作是 park），同样这个方法有很多平台的实现：

![unpark_impl](/assets/posts/unpark_impl.png)

我们依然是关心 posix 平台上的实现：

```cpp
void os::PlatformEvent::unpark() {
  if (Atomic::xchg(1, &_event) >= 0) return;
  int status = pthread_mutex_lock(_mutex);
  int anyWaiters = _nParked;
  status = pthread_mutex_unlock(_mutex);
  if (anyWaiters != 0) {
    status = pthread_cond_signal(_cond);
    assert_status(status == 0, status, "cond_signal");
  }
}
```

这是依然是通过 pthread 的 condition signal 唤醒线程，前面线程休眠是通过 condition wait 实现的。

我们继续回到 exit 函数，我们现在可以解释上面 JVM 平台默认执行「后来者居上」的逻辑了：根本原因就是在 enter 的时候插入 cxq 队列是将线程节点插入到队头，这样我们后面又从队头获取节点唤醒，因此总是唤醒后面加进来的节点线程。

#### 出队策略 1

现在我们看下，如果我们将 Knob_QMode 的默认值修改为 1，然后重新编译 JVM，再次运行会出现如下结果：

```bash
Thread 3 start!!!!!!
Thread 4 start!!!!!!
Thread 5 start!!!!!!
Thread 6 start!!!!!!

Thread 3 end!!!!!!
Thread 4 end!!!!!!
Thread 5 end!!!!!!
Thread 6 end!!!!!!
```

Wow，终于结束了「后来者居上」的模式了！这种模式下看起来先入队先执行，这种行为才是 FIFO 队列嘛～上面默认的形式其实是 LIFO 栈实现～

那么 mode == 1和 mode == 0 的差别在哪里，差别很小，就在于如下：

```cpp
    if (QMode == 1) {
      // QMode == 1 : drain cxq to EntryList, reversing order
      // We also reverse the order of the list.
      ObjectWaiter * s = NULL;
      ObjectWaiter * t = w;
      ObjectWaiter * u = NULL;
      while (t != NULL) {
        guarantee(t->TState == ObjectWaiter::TS_CXQ, "invariant");
        t->TState = ObjectWaiter::TS_ENTER;
        u = t->_next;
        t->_prev = u;
        t->_next = s;
        s = t;
        t = u;
      }
      _EntryList  = s;
      assert(s != NULL, "invariant");
    } else {
      // QMode == 0 or QMode == 2
      _EntryList = w;
      ObjectWaiter * q = NULL;
      ObjectWaiter * p;
      for (p = w; p != NULL; p = p->_next) {
        guarantee(p->TState == ObjectWaiter::TS_CXQ, "Invariant");
        p->TState = ObjectWaiter::TS_ENTER;
        p->_prev = q;
        q = p;
      }
    }
```

可以看到上面我们分析默认行为是 else 分支中的，如果 mode == 1 的话，会走上面的分支。正如注释中说的，下面会通过一个循环将 cxq 队列 reverse 一下，然后将新的队头也就是原来的队尾赋值给 _EntryList，接下来的唤醒还是通过 ExitEpilog 函数实现。

#### 出队策略 2

上面我们分析了默认下的 mode == 0 和 mode ==1 的行为，在 JVM 中还支持 2、3、4另外三种模式，下面我们分别看下。mode == 2 并且 cxq 队列不等于 NULL 的话会进行如下逻辑：

```cpp
if (QMode == 2 && _cxq != NULL) {
      // QMode == 2 : cxq has precedence over EntryList.
      // Try to directly wake a successor from the cxq.
      // If successful, the successor will need to unlink itself from cxq.
      w = _cxq;
      assert(w != NULL, "invariant");
      assert(w->TState == ObjectWaiter::TS_CXQ, "Invariant");
      ExitEpilog(Self, w);
      return;
}
```

这里将 w 指针指向了 cxq 队头，然后直接执行将队头唤醒的逻辑。这也就是说，如果 mode == 2 的话，cxq 的优先级是比 EntryList 高的。如果 cxq == NULL，那么会执行后面的逻辑，也就是先看 EntryList 中是否为空，如果 EntryList 不为空，就唤醒 EntryList 中的线程，否则将会继续 exit 中的那个大循环，在每一次循环的开头会判断一下 EntryList 和 cxq 队列是否同时为空，如果是的话，就以为这此时没有线程同步在这个锁上，并且也没有 notify 待唤醒的线程，因此就直接退出了 exit 函数。关于 wait 和 notify 的逻辑下面我们会分析，这里大家只要知道，notify 的时候会将 wait 的线程放到 EntryList 队列中。

这里我们可以看到，如果 mode == 2，并且 cxq 不为空的话，那么对于 cxq 队列执行的逻辑其实和 mode == 2 是一样的。唯一区别就是，当 EntryList 中有被 notify 唤醒的线程时，mode == 0 会优先执行 EntryList 中的线程唤醒，而 mode == 2 会优先执行 cxq 中，等到 cxq 中的线程全部唤醒的之后才会唤醒 EntryList 中的线程，只是一个次序的差别。下面我们通过如下代码验证我们的结论：

```java
    Thread t0 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 0 start!!!!!!");
            synchronized (lock) {
                try {
                    lock.wait();
                } catch (Exception e) {
                }
                System.out.println("Thread 0 end!!!!!!");
            }
        }
    });

    Thread t3 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 3 start!!!!!!");
            synchronized (lock) {
                try {
                    System.in.read();
                } catch (Exception e) {
                }
                lock.notify();
                System.out.println("Thread 3 end!!!!!!");
            }
        }
    });

    Thread t4 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 4 start!!!!!!");
            synchronized (lock) {
                System.out.println("Thread 4 end!!!!!!");
            }
        }
    });

    Thread t5 = new Thread(new Runnable() {
        @Override
        public void run() {
            int a = 0;
            System.out.println("Thread 5 start!!!!!!");
            synchronized (lock) {
                a++;
                System.out.println("Thread 5 end!!!!!!");
            }
        }
    });

    Thread t6 = new Thread(new Runnable() {
        @Override
        public void run() {
            int a = 0;
            System.out.println("Thread 6 start!!!!!!");
            synchronized (lock) {
                a++;
                System.out.println("Thread 6 end!!!!!!");
            }
        }
    });

    t0.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t3.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t4.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t5.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t6.start();
```

这段代码执行之后，在我按下回车之后并且没有 exit 出 monitor 时的 EntryList 队列和 cxq 队列状态如下：

![java_concurrency_cxq_2](/assets/posts/java_concurrency_cxq_2.jpg)

在 mode == 0 也就是默认状态下，执行结果如下：

```bash
Thread 0 start!!!!!!
Thread 3 start!!!!!!
Thread 4 start!!!!!!
Thread 5 start!!!!!!
Thread 6 start!!!!!!

Thread 3 end!!!!!!
Thread 0 end!!!!!!
Thread 6 end!!!!!!
Thread 5 end!!!!!!
Thread 4 end!!!!!!
```

在 mode == 2 的时候，执行结果如下：

```bash
Thread 0 start!!!!!!
Thread 3 start!!!!!!
Thread 4 start!!!!!!
Thread 5 start!!!!!!
Thread 6 start!!!!!!

Thread 3 end!!!!!!
Thread 6 end!!!!!!
Thread 5 end!!!!!!
Thread 4 end!!!!!!
Thread 0 end!!!!!!
```

我们从 t0 的唤醒时机执行完毕的时机就可以验证我们上面的结论了。

#### 出队策略 3 和出队策略 4

策略 3 和策略 4 的逻辑比较相似：

```cpp
    if (QMode == 3 && _cxq != NULL) {
      // Aggressively drain cxq into EntryList at the first opportunity.
      // This policy ensure that recently-run threads live at the head of EntryList.
      // Drain _cxq into EntryList - bulk transfer.
      // First, detach _cxq.
      // The following loop is tantamount to: w = swap(&cxq, NULL)
      w = _cxq;
      for (;;) {
        assert(w != NULL, "Invariant");
        ObjectWaiter * u = Atomic::cmpxchg((ObjectWaiter*)NULL, &_cxq, w);
        if (u == w) break;
        w = u;
      }
      assert(w != NULL, "invariant");
      ObjectWaiter * q = NULL;
      ObjectWaiter * p;
      for (p = w; p != NULL; p = p->_next) {
        guarantee(p->TState == ObjectWaiter::TS_CXQ, "Invariant");
        p->TState = ObjectWaiter::TS_ENTER;
        p->_prev = q;
        q = p;
      }
      // Append the RATs to the EntryList
      // TODO: organize EntryList as a CDLL so we can locate the tail in constant-time.
      ObjectWaiter * Tail;
      for (Tail = _EntryList; Tail != NULL && Tail->_next != NULL;
           Tail = Tail->_next)
        /* empty */;
      if (Tail == NULL) {
        _EntryList = w;
      } else {
        Tail->_next = w;
        w->_prev = Tail;
      }
      // Fall thru into code that tries to wake a successor from EntryList
    }
    if (QMode == 4 && _cxq != NULL) {
      // Aggressively drain cxq into EntryList at the first opportunity.
      // This policy ensure that recently-run threads live at the head of EntryList.
      // Drain _cxq into EntryList - bulk transfer.
      // First, detach _cxq.
      // The following loop is tantamount to: w = swap(&cxq, NULL)
      w = _cxq;
      for (;;) {
        assert(w != NULL, "Invariant");
        ObjectWaiter * u = Atomic::cmpxchg((ObjectWaiter*)NULL, &_cxq, w);
        if (u == w) break;
        w = u;
      }
      assert(w != NULL, "invariant");
      ObjectWaiter * q = NULL;
      ObjectWaiter * p;
      for (p = w; p != NULL; p = p->_next) {
        guarantee(p->TState == ObjectWaiter::TS_CXQ, "Invariant");
        p->TState = ObjectWaiter::TS_ENTER;
        p->_prev = q;
        q = p;
      }
      // Prepend the RATs to the EntryList
      if (_EntryList != NULL) {
        q->_next = _EntryList;
        _EntryList->_prev = q;
      }
      _EntryList = w;
      // Fall thru into code that tries to wake a successor from EntryList
    }
```

从上面两个策略的注释中可以看出，这两个策略的主要目的是将 EntryList 和 cxq 队列进行一个链接操作。既然是两个队列的链接，就会涉及到谁链接到谁前面的问题。还是拿上面策略 2 的例子来说，如果我的当前 EntryList 和 cxq 队列状态如下：

![java_concurrency_cxq_2 (1)](/assets/posts/java_concurrency_cxq_2.jpg)

那么策略 3 的链接之后会成下面这样：

![java_concurrency_cxq_3](/assets/posts/java_concurrency_cxq_3.jpg)

经过策略 4 的链接之后会变成下面这样：

![java_concurrency_cxq_4](/assets/posts/java_concurrency_cxq_4.jpg)

很明显，策略 3 是将 cxq 放在 EntryList 之后，而策略 3 是放在 EntryList 之前。

下面我们还是以策略 2 中代码分别测试，验证我们的结论。

在策略 3 下的运行结果：

```bash
Thread 0 start!!!!!!
Thread 3 start!!!!!!
Thread 4 start!!!!!!
Thread 5 start!!!!!!
Thread 6 start!!!!!!

Thread 3 end!!!!!!
Thread 0 end!!!!!!
Thread 6 end!!!!!!
Thread 5 end!!!!!!
Thread 4 end!!!!!!
```

在策略 4 下的运行结果：

```bash
Thread 0 start!!!!!!
Thread 3 start!!!!!!
Thread 4 start!!!!!!
Thread 5 start!!!!!!
Thread 6 start!!!!!!

Thread 3 end!!!!!!
Thread 6 end!!!!!!
Thread 5 end!!!!!!
Thread 4 end!!!!!!
Thread 0 end!!!!!!
```

这个运行结果直接就是印证了上面的分析，线程结束运行的时机就是上面图中的合并之后的链表的顺序。

到这里，我们全部分析完了 exit 的执行逻辑，exit 中的重点就是 EntryList 和 cxq 队列的出队策略。下面我们总结下，出队策略总体上可以分为两组：

1. EntryList 优先于 cxq  
这种模式对应模式 0 、模式 1和模式 3。其中模式 0 对 cxq 队列会保持「后来者居上」的队列顺序，而模式 1 会 reverse 这个顺序，模式 3 不会变更任何顺序，只是简单地将 cxq 链表放在 EntryList 的后面
2. cxq 优先于 EntryList  
这种模式对应模式 2 、模式 4。其中模式2中若cxq不为空则优先取cxq，而模式4只是将 EntryList 放在 cxq 的后面，然后按照新的 EntryList 队列开始唤醒线程

## Object wait 和 notify 的实现机制

Java Object 类提供了一个基于 native 实现的 wait 和 notify 线程间通讯的方式，这是除了 synchronized 之外的另外一块独立的并发基础部分，有关 wait 和 notify 的部分内容，我们在上面分析 monitor 的 exit 的时候已经有一些涉及，但是并没有过多的深入，导致留下了不少的疑问，下面本小节会详细分析一下在 HotSpot JVM 中的 wait 和 notify 的实现逻辑。

如果你打开 JDK 中的 Object 类的代码，你会看到 wait 和 notify/notifyAll 的实现全部是采用采用 native 实现的，并且在 Object 的开头有如下代码：

```java
private static native void registerNatives();
static {
    registerNatives();
}
```

这里是不是和前面的 Thread 类如出一辙，因此查找 JVM 中的本地实现函数也是一样的手段。所以这里就省略这个查找的部分，相信聪明的你已经知道怎么在 JVM 代码中找实现的函数了。如果一路查找的话，你会发现 wait 和 notify/notifyAll 还是在 src/hotspot/share/runtime/objectMonitor.cpp 中实现的。因此我们还是会在这个文件中进行分析。

### wait 实现

ObjectMonitor 类中的 wait 函数实现如下：

```cpp
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
  ...
  if (interruptible && Thread::is_interrupted(Self, true) && !HAS_PENDING_EXCEPTION) {
    ...
    // 抛出异常，不会直接进入等待
    THROW(vmSymbols::java_lang_InterruptedException());
    ...
  }
  ...
  ObjectWaiter node(Self);
  node.TState = ObjectWaiter::TS_WAIT;
  Self->_ParkEvent->reset();
  OrderAccess::fence();
  
  Thread::SpinAcquire(&_WaitSetLock, "WaitSet - add");
  AddWaiter(&node);
  Thread::SpinRelease(&_WaitSetLock);
  
  if ((SyncFlags & 4) == 0) {
    _Responsible = NULL;
  }
  
  ...
  // exit the monitor
  exit(true, Self);
  ...
  if (interruptible && (Thread::is_interrupted(THREAD, false) || HAS_PENDING_EXCEPTION)) {
        // Intentionally empty
      } else if (node._notified == 0) {
        if (millis <= 0) {
          Self->_ParkEvent->park();
        } else {
          ret = Self->_ParkEvent->park(millis);
        }
  }
  // 被 notify 唤醒之后的善后逻辑
  ...
}
```

wait 函数的实现也比较长，但是我们关心的核心功能部分就是上面列出的。首先会判断一下当前线程是否为可中断并且是否已经被中断，如果是的话会直接抛出 InterruptedException 异常，而不会进入 wait 等待，否则的话，就需要执行下面的等待的过程。首先会根据 Self 当前线程新建一个 ObjectWaiter 对象节点，这个对象我们在前面分析 monitor 的 enter 的时候就已经见到过了。生成一个新的节点之后就是需要将这个节点放到等待队列中，通过调用 AddWaiter 函数实现 node 的入队操作，不过在入队操作之前需要获得互斥锁以保证并发安全：

```cpp
void Thread::SpinAcquire(volatile int * adr, const char * LockName) {
  if (Atomic::cmpxchg (1, adr, 0) == 0) {
    return;   // normal fast-path return
  }
  // Slow-path : We've encountered contention -- Spin/Yield/Block strategy.
  TEVENT(SpinAcquire - ctx);
  int ctr = 0;
  int Yields = 0;
  for (;;) {
    while (*adr != 0) {
      ++ctr;
      if ((ctr & 0xFFF) == 0 || !os::is_MP()) {
        if (Yields > 5) {
          os::naked_short_sleep(1);
        } else {
          os::naked_yield();
          ++Yields;
        }
      } else {
        SpinPause();
      }
    }
    if (Atomic::cmpxchg(1, adr, 0) == 0) return;
  }
}
```

从函数的名称就能看出这是一个自旋锁的实现，并不会「立即」使得线程陷入等待状态，从实现上看，这里是通过一个死循环不断通过 cas 检查判断是否获得锁。这里开始会通过一个 cas 检查看下是否能够成功，如果成功的话就不用进行下面比较重量级的 spin 过程。如果获取失败，就需要进入下面的 spin 过程，这里的 spin 逻辑是一个比较有意思的算法。这里定义了一个 ctr 变量，其实就是 counter 计数器的意思，```(ctr &amp; 0xFFF) == 0 || !os::is_MP()``` 这个条件比较有意思，意思是说如果我尝试的次数大于 0xfff，或者当前系统是一个单核处理器系统，那么就执行下面的逻辑。可以看到这里的 spin 是有一定的限度的。首先开始的时候，如果是多核系统，会直接执行 SpinPause ，我们看下 SpinPause 函数的实现，这个函数是实现 CPU 的忙等待，因此会有不同系统和 CPU 架构的对应实现：

![SpinPause_impl](/assets/posts/SpinPause_impl.png)

我们这里依然只关心 linux 平台上的 64 bit 架构的实现：

```cpp
  int SpinPause() {
    return 0;
  }
```

是的，你没有看错，这里的实现就是没有实现，只是返回一个 0 就完事！其实你想想，通过调用一个立即返回的空函数不就实现了 CPU 的忙等待了么？只不过这种实现方式比较不太优雅罢了～我们再来看 **SpinAcquire** 函数的实现，如果我们尝试的次数已经到了 0xFFF 次的话，那就表示我们需要使用另外一种机制来实现忙等了，因为这里尝试获取锁不能预测多久可以获得，因此不可能无限期地执行上面调用空函数，这是对资源的一种极大的浪费。如果尝试了 0xFFF 次还没有成功的话，就通过如下方式实现等待：

```cpp
if (Yields > 5) {
   os::naked_short_sleep(1);
} else {
   os::naked_yield();
   ++Yields;
}
```

首先会尝试通过 yield 函数来将当前线程的 CPU 执行时间让出来，如果让了 5 次还是没有获得锁，那就只能通过 naked_short_sleep 来实现等待了，这里的 naked_short_sleep 函数从名字就能看出来是短暂休眠等待，通过每次休眠等待 1ms 实现。我们现在看下 naked_yield 的实现方式，同样这个函数也有很多系统平台的实现，我们老规矩只看 linux：

```cpp
void os::naked_yield() {
  sched_yield();
}
```

可以看到这里的实现是比较简单的，直接通过 pthread 的 sched_yield 函数实现线程的时间片让出。下面在看下 naked_short_sleep 的实现（依旧是 linux 平台）：

```cpp
void os::naked_short_sleep(jlong ms) {
  struct timespec req;
  assert(ms < 1000, "Un-interruptable sleep, short time use only");
  req.tv_sec = 0;
  if (ms > 0) {
    req.tv_nsec = (ms % 1000) * 1000000;
  } else {
    req.tv_nsec = 1;
  }
  nanosleep(&req, NULL);
  return;
}
```

这里我们通过 nanosleep 系统调用实现线程的 timed waiting。

到这里我们大致分析了 **SpinAcquire** 函数的实现，现在我们需要说明下这个函数中为啥需要判断 os::_is_MP_()，逻辑是这样的：如果是单核处理器就通过 yield 或者 sleep 实现等待，如果是多核处理器的话就通过调用空实现函数来忙等待。为啥需要这样呢？因为如果是单核 CPU 的话，你通过调用空实现函数实现忙等待是不科学的，因为只有一个核，你却通过这个核来实现忙等待，那么原本需要释放锁的线程得不到执行，那就可能造成饥饿等待，我们的 CPU 一直在转动，但是没有解决任何问题。所以如果是单核 CPU 系统的话，我们不能通过调用空函数来实现等待。相反，如果是多核的话，那就可以在另外一个空闲的 CPU 上实现忙等待一增加系统吞吐量，可以看出在 jVM 中为了增加系统的系统和保证系统的兼容性，做了多少的努力和实现啊！

上面的 SpinAcquire 函数返回之后，就表示我们获得了锁，现在可以将我们的 node 放到等待队列中了：

```cpp
inline void ObjectMonitor::AddWaiter(ObjectWaiter* node) {
  assert(node != NULL, "should not add NULL node");
  assert(node->_prev == NULL, "node already in list");
  assert(node->_next == NULL, "node already in list");
  // put node at end of queue (circular doubly linked list)
  if (_WaitSet == NULL) {
    _WaitSet = node;
    node->_prev = node;
    node->_next = node;
  } else {
    ObjectWaiter* head = _WaitSet;
    ObjectWaiter* tail = head->_prev;
    assert(tail->_next == head, "invariant check");
    tail->_next = node;
    head->_prev = node;
    node->_next = head;
    node->_prev = tail;
  }
}
```

这里的实现其实非常简单，就是将 node 插入双向链表 _WaitSet 的尾部。插入链表完毕之后，需要通过 _SpinRelease_ 将锁释放。

现在我们已经将新建的 node 节点加入到 WaitSet 队列中了，我们继续看 wait 函数接下来的逻辑，现在我们就要执行如下内容：

```cpp
// exit the monitor
exit(true, Self);
```

是的，你肯定知道 Java  Object 的 wait 操作会释放 monitor 锁，释放操作就是这里实现的！

释放了 monitor 锁之后，我们就需要将当前线程进行 park 等待了：

```cpp
if (interruptible && (Thread::is_interrupted(THREAD, false) || HAS_PENDING_EXCEPTION)) {
    // Intentionally empty
} else if (node._notified == 0) {
    if (millis <= 0) {
        Self->_ParkEvent->park();
    } else {
        ret = Self->_ParkEvent->park(millis);
    }
}
```

在正式 park 之前，还会再一次看下是否有 interrupted，如果有的话就会跳过 park 操作，否则就会进行 park 阻塞。因为 wait 操作可以带时间，表示阻塞的时间，这里会根据需要阻塞的时间给 park 函数不同的参数。park 函数我们前面在分析 monitor 的 enter 的时候已经分析过了，因此这里就不再赘述了。

在 wait 接下来的函数，就是 park 阻塞唤醒之后的善后逻辑，对于我们的分析不是十分重要，这里就跳过。接下来，我们重点分析一下 notify 唤醒的逻辑。

### notify 实现

notify 函数的实现如下：

```cpp
void ObjectMonitor::notify(TRAPS) {
  CHECK_OWNER();
  if (_WaitSet == NULL) {
    TEVENT(Empty-Notify);
    return;
  }
  DTRACE_MONITOR_PROBE(notify, this, object(), THREAD);
  INotify(THREAD);
  OM_PERFDATA_OP(Notifications, inc(1));
}
```

可以看到，首先会检查 WaitSet 队列，如果队列为空的话，表示没有线程执行了 wait，也就没有必要执行接下来的操作了，直接返回即可。

如果 WaitSet 队列不为空，表示有线程在这个 monitor 上 wait 了，因此就需要唤醒某个线程，这里是通过调用 INotify 函数实现：

```cpp
void ObjectMonitor::INotify(Thread * Self) {
  const int policy = Knob_MoveNotifyee;
  Thread::SpinAcquire(&_WaitSetLock, "WaitSet - notify");
  ObjectWaiter * iterator = DequeueWaiter();
  if (iterator != NULL) {
    ObjectWaiter * list = _EntryList;
    if (policy == 0) {
      // prepend to EntryList
      if (list == NULL) {
        ...
      } else {
        ...
      }
    } else if (policy == 1) {
      // append to EntryList
      if (list == NULL) {
        ...
      } else {
        ...
      }
    } else if (policy == 2) {
      // prepend to cxq
      if (list == NULL) {
        ...
      } else {
        ...
      }
    } else if (policy == 3) {
      // append to cxq
      ...
    } else {
      ...
    }
    ...
  }
  Thread::SpinRelease(&_WaitSetLock);
}
```

我们先不深入理解代码的细节，先来把握一下 INotify 函数的框架。可以看到这里的操作都是在  _WaitSetLock 保护下的，首先会从 WaitSet 队列中出队一个节点，然后针对这个节点根据 Knob_MoveNotifyee 来决定执行不同策略逻辑，并且策略中的逻辑框架就是一样的，根据 _EntryList 是否为空执行不同操作（策略 3 除外，下面会单独分析）。

那么，Knob_MoveNotifyee 是什么呢？其实从定义的地方可以看出：

```cpp
// notify() - disposition of notifyee
static int Knob_MoveNotifyee        = 2;
```

从注释中可以看出，这个就是 notify 唤醒的策略定义。从上面的 INotify 函数的注释中可以看出总共有如下几种模式：

1. 策略 0：将需要唤醒的 node 放到 EntryList 的头部
2. 策略 1：将需要唤醒的 node 放到 EntryList 的尾部
3. 策略 2：将需要唤醒的 node 放到 CXQ 的头部
4. 策略 3：将需要唤醒的 node 放到 CXQ 的尾部

> PS：当然，对于策略2 （默认），当EntryList为空时，该node会放到CXQ中，后文会解释。

在分析不同策略的逻辑之前，我们先看下 WaitSet 的出队逻辑的实现，这是 INotify 函数开始会执行的事情：

```cpp
inline ObjectWaiter* ObjectMonitor::DequeueWaiter() {
  // dequeue the very first waiter
  ObjectWaiter* waiter = _WaitSet;
  if (waiter) {
    DequeueSpecificWaiter(waiter);
  }
  return waiter;
}
```

从注释中可以看出，这里将 WaitSet 队列中的第一个 node 出队，下面直接返回 WaitSet 队列的指针也就是队头，然后删除出队节点：

```cpp
inline void ObjectMonitor::DequeueSpecificWaiter(ObjectWaiter* node) {
  assert(node != NULL, "should not dequeue NULL node");
  assert(node->_prev != NULL, "node already removed from list");
  assert(node->_next != NULL, "node already removed from list");
  // when the waiter has woken up because of interrupt,
  // timeout or other spurious wake-up, dequeue the
  // waiter from waiting list
  ObjectWaiter* next = node->_next;
  if (next == node) {
    assert(node->_prev == node, "invariant check");
    _WaitSet = NULL;
  } else {
    ObjectWaiter* prev = node->_prev;
    assert(prev->_next == node, "invariant check");
    assert(next->_prev == node, "invariant check");
    next->_prev = prev;
    prev->_next = next;
    if (_WaitSet == node) {
      _WaitSet = next;
    }
  }
  node->_next = NULL;
  node->_prev = NULL;
}
```

这样我们就完成了从 WaitSet 双向链表队列中的队头出队逻辑。

如果我们自己看下 **INotify** 函数的实现，你会发现这里全是队列的操作，并没有唤醒线程。是的，唤醒线程不在这里，这里只是将需要唤醒的线程放到 EntryList 队列中，然后在 exit 函数中唤醒。而 exit 函数我们已经很详细地分析过了，相信这时的你已经有一个深入的理解了吧～那为啥 notify 不直接唤醒呢？因为 wait 等待的线程是 synchronized 同步块中的呀！它需要拿到 monitor 才能继续执行啊，什么时候才能拿到 monitor 呢？也就是别人 exit 的时候你才能啊～这就解释了为毛 notify 不直接唤醒而是在 exit 的时候唤醒。

上面，我们看到了 JVM 的默认策略是 2，下面我们分别分析一下不同的策略逻辑。

#### 唤醒策略 0

策略 0 的执行逻辑如下：

```cpp
if (list == NULL) {
    iterator->_next = iterator->_prev = NULL;
    _EntryList = iterator;
} else {
    list->_prev = iterator;
    iterator->_next = list;
    iterator->_prev = NULL;
    _EntryList = iterator;
}
```

如果 EntryList 为空的话，表示之前没有线程被 notify 唤醒，已经直接将当前节点放到 EntryList 中即可。否则的话，就将当前节点放到 EntryList 的头部。

下面我们通过一个实验来验证我们的结论。

实验的 java 代码：

```java
    Thread t0 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 0 start!!!!!!");
            synchronized (lock) {
                try {
                    lock.wait();
                } catch (Exception e) {
                }
                System.out.println("Thread 0 end!!!!!!");
            }
        }
    });
    Thread t1 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 1 start!!!!!!");
            synchronized (lock) {
                try {
                    lock.wait();
                } catch (Exception e) {
                }
                System.out.println("Thread 1 end!!!!!!");
            }
        }
    });
    Thread t2 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 2 start!!!!!!");
            synchronized (lock) {
                try {
                    lock.wait();
                } catch (Exception e) {
                }
                System.out.println("Thread 2 end!!!!!!");
            }
        }
    });
    Thread t3 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 3 start!!!!!!");
            synchronized (lock) {
                for (int i = 0; i < 3; i++) {
                    try {
                        System.in.read();
                    } catch (Exception e) {
                    }
                    lock.notify();
                }
                System.out.println("Thread 3 end!!!!!!");
            }
        }
    });
    t0.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t1.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t2.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t3.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
```

在我按下三次回车键之前的 WaitSet 状态如下：

![java_concurrency_notify_0_0](/assets/posts/java_concurrency_notify_0_0.jpg)

当我依次按下三次回车键的之后，WaitSet 链表就为空，此时 EntryList 如下：

![java_concurrency_notify_0_1](/assets/posts/java_concurrency_notify_0_1.jpg)

我们将 Knob_MoveNotifyee 的默认值修改 0，然后重新编译 JVM，执行上面的 java 代码结果如下：

```bash
Thread 0 start!!!!!!
Thread 1 start!!!!!!
Thread 2 start!!!!!!
Thread 3 start!!!!!!



Thread 3 end!!!!!!
Thread 2 end!!!!!!
Thread 1 end!!!!!!
Thread 0 end!!!!!!
```

可以看到线程结束运行的顺序和我们分析的一样，就是 2 -&gt; 1 -&gt; 0。

#### 唤醒策略 1

策略 1 和策略 0 逻辑是相似，只是这里将节点放到尾部：

```cpp
if (list == NULL) {
        iterator->_next = iterator->_prev = NULL;
        _EntryList = iterator;
} else {
        // CONSIDER:  finding the tail currently requires a linear-time walk of
        // the EntryList.  We can make tail access constant-time by converting to
        // a CDLL instead of using our current DLL.
        ObjectWaiter * tail;
        for (tail = list; tail->_next != NULL; tail = tail->_next) {}
        assert(tail != NULL && tail->_next == NULL, "invariant");
        tail->_next = iterator;
        iterator->_prev = tail;
        iterator->_next = NULL;
}
```

这里的注释很有意思，说是可以将 EntryList 做成循环双向队列（CDLL）可以优化操作，因为 CDLL 查找 tail 节点的时间是常量时间的，大家有兴趣可以修改下这里的实现，兴许你可以给 JVM 提交一个 patch，然后你也是 JVM 源码贡献者之一呢。。。

下面我们看下上面策略 0 的代码执行的结果：

```bash
Thread 0 start!!!!!!
Thread 1 start!!!!!!
Thread 2 start!!!!!!
Thread 3 start!!!!!!



Thread 3 end!!!!!!
Thread 0 end!!!!!!
Thread 1 end!!!!!!
Thread 2 end!!!!!!
```

可以看到，这里的结束的顺序和策略 0 是相反的。

#### 唤醒策略 2——默认策略

策略 0 和策略 1 是将需要唤醒的节点放到 EntryLIst 中，而策略 2 和策略 3 是将节点放到 cxq 队列中，只不过策略 2 放到 cxq 的头部，策略 3 放到 cxq 的尾部。

策略 2 是默认策略，也就是说大家手上的 JVM 行为是将唤醒的节点放到 cxq 队列的头部。你还记得 cxq 队列吧？就是 synchronized 的等待队列啊，希望你还没有忘记。

为了验证我们的结论，我们需要使用一个不一样的 java 代码，我们需要结合 synchronized 阻塞队列才能看出效果：

```java
    Thread t0 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 0 start!!!!!!");
            synchronized (lock) {
                try {
                    lock.wait();
                } catch (Exception e) {
                }
                System.out.println("Thread 0 end!!!!!!");
            }
        }
    });
    Thread t1 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 1 start!!!!!!");
            synchronized (lock) {
                try {
                    lock.wait();
                } catch (Exception e) {
                }
                System.out.println("Thread 1 end!!!!!!");
            }
        }
    });
    Thread t2 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 2 start!!!!!!");
            synchronized (lock) {
                try {
                    lock.wait();
                } catch (Exception e) {
                }
                System.out.println("Thread 2 end!!!!!!");
            }
        }
    });
    Thread t3 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 3 start!!!!!!");
            synchronized (lock) {
                try {
                    System.in.read();
                } catch (Exception e) {
                }
                lock.notify();
                lock.notify();
                lock.notify();
                System.out.println("Thread 3 end!!!!!!");
            }
        }
    });
    Thread t4 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 4 start!!!!!!");
            synchronized (lock) {
                System.out.println("Thread 4 end!!!!!!");
            }
        }
    });
    Thread t5 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 5 start!!!!!!");
            synchronized (lock) {
                System.out.println("Thread 5 end!!!!!!");
            }
        }
    });
    Thread t6 = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("Thread 6 start!!!!!!");
            synchronized (lock) {
                System.out.println("Thread 6 end!!!!!!");
            }
        }
    });
    t0.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t1.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t2.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t3.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t4.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t5.start();
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (Exception e) {
    }
    t6.start();
```

我们启动了 6 个线程，其中 0 ～ 2 是会 wait 到 WaitSet 中的，4 ～ 6 是等待在 cxq 中的。

好的，下面我们看下策略 2，也就是默认策略的执行逻辑：

```cpp
if (list == NULL) {
        iterator->_next = iterator->_prev = NULL;
        _EntryList = iterator;
} else {
        iterator->TState = ObjectWaiter::TS_CXQ;
        for (;;) {
          ObjectWaiter * front = _cxq;
          iterator->_next = front;
          if (Atomic::cmpxchg(iterator, &_cxq, front) == front) {
            break;
          }
        }
}
```

首先如果发现 EntryList 为空的话，也就是第一个被 notify 唤醒的线程会进入到 EntryList，而 WaitSet 中剩下的节点会依次插入到 cxq 的头部，然后更新 cxq 指针指向新的头节点。

以上面的 java 代码为例，在我按下回车键之前的状态如下：

![java_concurrency_2_0](/assets/posts/java_concurrency_2_0.jpg)

当我按下回车之后状态如下：

![java_concurrency_2_1](/assets/posts/java_concurrency_2_1.jpg)

因此，在 exit 的时候，在默认状态下会实现唤醒 EntryList 中线程，然后在唤醒 cxq 中的，所以唤醒的顺序是：0 -&gt; 2 -&gt; 1 -&gt; 6 -&gt; 5 -&gt; 4。

下面我们执行代码，验证我们的猜想：

```bash
Thread 0 start!!!!!!
Thread 1 start!!!!!!
Thread 2 start!!!!!!
Thread 3 start!!!!!!
Thread 4 start!!!!!!
Thread 5 start!!!!!!
Thread 6 start!!!!!!

Thread 3 end!!!!!!
Thread 0 end!!!!!!
Thread 2 end!!!!!!
Thread 1 end!!!!!!
Thread 6 end!!!!!!
Thread 5 end!!!!!!
Thread 4 end!!!!!!
```

可以看到，和我们的猜想完全一样。

#### 唤醒策略 3

策略 3 的逻辑和策略 2 比较相似，只是策略 3 会将节点放到 cxq 尾部:

```cpp
iterator->TState = ObjectWaiter::TS_CXQ;
      for (;;) {
        ObjectWaiter * tail = _cxq;
        if (tail == NULL) {
          iterator->_next = NULL;
          if (Atomic::replace_if_null(iterator, &_cxq)) {
            break;
          }
        } else {
          while (tail->_next != NULL) tail = tail->_next;
          tail->_next = iterator;
          iterator->_prev = tail;
          iterator->_next = NULL;
          break;
        }
}
```

这里不会判断 EntryList 是否为空，而是直接将节点放到 cxq 的尾部，这一点和前面几个策略不一样，需要注意下。

所以我们可以预测，上面策略 3 验证代码中的唤醒顺序是：6 -&gt; 5 -&gt; 4 -&gt; 0 -&gt;1 -&gt; 2，下面执行下代码看看结果：

```bash
Thread 0 start!!!!!!
Thread 1 start!!!!!!
Thread 2 start!!!!!!
Thread 3 start!!!!!!
Thread 4 start!!!!!!
Thread 5 start!!!!!!
Thread 6 start!!!!!!

Thread 3 end!!!!!!
Thread 6 end!!!!!!
Thread 5 end!!!!!!
Thread 4 end!!!!!!
Thread 0 end!!!!!!
Thread 1 end!!!!!!
Thread 2 end!!!!!!
```

可以看到，和我们的预测结果依然是一样的。

到这里我们就完整分析完毕了 notify 的实现逻辑，整体上的实现还是比较简单的，只是根据不同的策略执行不同的唤醒出队逻辑，同时这里的逻辑会和 exit 中的出队逻辑协调起来，上面我们已经通过实际的例子验证了这一点。

### notifyAll 实现

notifyAll 的实现其实和 notify 实现大同小异：

```cpp
void ObjectMonitor::notifyAll(TRAPS) {
  CHECK_OWNER();
  if (_WaitSet == NULL) {
    TEVENT(Empty-NotifyAll);
    return;
  }
  DTRACE_MONITOR_PROBE(notifyAll, this, object(), THREAD);
  int tally = 0;
  while (_WaitSet != NULL) {
    tally++;
    INotify(THREAD);
  }
  OM_PERFDATA_OP(Notifications, inc(tally));
}
```

可以看到，其实就是根据 WaitSet 长度，反复调用 INotify 函数，相当于多次调用 notify，因此这里就不在赘述了。

## Volatile 语义

Volatile 这个话题是并行计算领域一个非常有意思的话题，涉及到非常多的细节。在 C/C++ 和 Java 中都有 volatile 这个关键字，在实际探讨这个关键字的语义之前，我们先看下这个词的字面含义，下面是柯林斯高阶词典中的含义：

> A situation that is volatile is likely to change suddenly and unexpectedly.

这里的解释有三个重要的含义：

1. likely：可能的，这意味着被 volatile 形容的对象「可能」发生改变，因此我们不应该针对这个变量的值作出任何假设
2. suddenly：突然地，这意味着这个变量有可能会在瞬间很快的发生变化
3. unexpectedly：不可预期地，这其实与 likely 的含义一致，意味着这个变量可能随时随地发生变化，我们不能依赖于它的状态

因此，在编程语言中使用这个关键字修饰我们的变量，就意味着：这个变量可能会在任何时候改变为任何值，任何使用方必须实时关注这个值的变化，并且不能作出任何假设。

在实际探讨 Java 中的 volatile 关键字的语义之前，我们首先看下 C/C++ 中的关键字的语义，理解了 C/C++ 的 volatile 语义对于理解 volatile 非常有帮助。

### C/C++ volatile 语义

查看 C  volatile 的语义最简单的方式就是看 [CPP Reference](https://en.cppreference.com/w/c/language/volatile) 中的介绍：

> Every access (both read and write) made through an lvalue expression of volatile-qualified type is considered an observable side effect for the purpose of optimization and is evaluated strictly according to the rules of the abstract machine (that is, all writes are completed at some time before the next sequence point). This means that within a single thread of execution, a volatile access cannot be optimized out or reordered relative to another visible side effect that is separated by a [sequence point](https://en.cppreference.com/w/c/language/eval_order) from the volatile access.

从这里的描述我们可以看出在 C 中对 volatile 的访问规则如下：

1. 不允许被优化消失（optimized out）
2. 不被编译器优化乱序（reorder）

下面我们通过一个简单的例子来了解下 C 中的 volatile 关键字。

下面是一段简单的 c 程序：

```cpp
static int a = 12345;
static int t = 9090;
int main(void)
{
  int b = a;
  int c = a;
  int e = t;
  return b + c + e;
}
```

我们通过如下命令将上面的 c 代码编译成 .s 汇编代码：

```bash
gcc -S -O3 main.c -o main.s
```

得到如下汇编代码：

```nasm
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 13
	.globl	_main                   ## -- Begin function main
	.p2align	4, 0x90
_main:                                  ## @main
	.cfi_startproc
## %bb.0:
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register %rbp
	movl	$33780, %eax            ## imm = 0x83F4
	popq	%rbp
	retq
	.cfi_endproc
                                        ## -- End function
.subsections_via_symbols
```

可以看到，这里编译器已经帮我们将需要 return 的结果计算出来了，无需再进行取值然后计算了。也就是说，gcc 编译器已经针对我们的变量作出内存上的假设。现在我们将变量 a 和 t 使用 volatile 来修饰：

```cpp
static volatile int a = 12345;
static volatile int t = 9090;
int main(void)
{
  int b = a;
  int c = a;
  int e = t;
  return b + c + e;
}
```

可以得到如下汇编：

```nasm
	.section	__TEXT,__text,regular,pure_instructions
	.macosx_version_min 10, 13
	.globl	_main                   ## -- Begin function main
	.p2align	4, 0x90
_main:                                  ## @main
	.cfi_startproc
## %bb.0:
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register %rbp
	movl	_a(%rip), %eax
	addl	_a(%rip), %eax
	addl	_t(%rip), %eax
	popq	%rbp
	retq
	.cfi_endproc
                                        ## -- End function
	.section	__DATA,__data
	.p2align	2               ## @a
_a:
	.long	12345                   ## 0x3039
	.p2align	2               ## @t
_t:
	.long	9090                    ## 0x2382
.subsections_via_symbols
```

可以很清晰地看到，这里加上 volatile 之后 gcc 就不会对变量的值作出任何假设，只能老老实实地按部就班读取然后计算。

以上的一个小例子就验证了上面我们讲到的 c 中的 volatile 第一个特性：防止被优化消失。至于第二个特点：不被编译器优化乱序，则是保证了被 volatile 修饰的变量在编译之后生成的机器码顺序肯定和原始代码中的顺序是一样的。

但是这里仅仅是保证了编译之后的机器指令码是有序的，但是我们都知道现代的 CPU 会乱序执行指令，只要保证最终的结果正确即可。因此，这里的 volatile 只能保证编译阶段不被乱序优化，不能保证执行阶段的乱序优化。如果想要保证执行阶段的乱序优化必须使用系统提供的内存屏障技术来实现。

更多关于 C++ 语义的分析，可以参考这篇博文（文末参考资料中有）：[C/C++ 中的 volatile 语义](https://liam.page/2018/01/18/volatile-in-C-and-Cpp/)

### Java volatile 语义

上面我们简单滴了解了在 c 中的 volatile 的语义，我们对 volatile 有了一个大致上的认识。那么在 java 中的 volatile 的语义又是什么呢？基本上如下：

1. 保证在修改之后，其他的线程立即可见，这一点和 C 中的不对变量内存做任何假设是一致的
2. 保证指令不会乱序执行，也就是说，执行到 volatile 变量的时候，在此之前的指令必须全部完成。这一点比 c 中的 volatile 更加彻底，因为 java 中的 volatile 实现底层采用了内存屏障技术保证了这一点

上面的第一个语义相信大家都了解，也很容易通过例子来验证。下面我们重点看下第二个特性是怎么保证的。我们知道 JVM 为了加快执行的效率，通常会采用 JIT 来优化代码执行的速度，也就是说，经过 JIT compile 之后的代码，就不会再执行字节码了，而是直接执行对应的底层机器码。如果我们能够有办法拿到 JIT 执行的对应机器码，我们不就能窥探到 JVM 底层在执行 volatile 变量的存取时的执行逻辑了吗？是的，我们真的可以获取到 JVM 底层的 JIT 机器代码，在 OpenJDK 的源码中提供了一个非常强大的工具：hsdis，关于这个工具的介绍可以参考源码中的 README：/src/utils/hsdis/README。我们需要编译下这个工具，编译的时候需要 binutils 这个 GNU 工具集合的源码，我们只要将下载好的 binutils 代码放到同一个目录下，然后执行如下命令即可编译 hsdis：

```bash
make BINUTILS=binutils-2.17 ARCH=amd64 CFLAGS="-Wno-error"
```

因为我的机器是 64 bit 的，所以这里执行 ARCH 为 amd64，又因为我是在 mac 平台上编译的，默认使用 clang（apple xcode 工具链），这个工具比 gcc 检查要严格地多，因此需要加上 CFLAGS=&ldquo;-Wno-error&rdquo;，不然很多 binutils 中的 warning 全成了错误而无法编译。另需要说明的是，这里最好使用 2.17 版本的 binutils，别的版本我都编译失败了，因为接口不兼容了，这一点其实在 README 中也有说到。

如果我们顺利编译，可以在当前目录下的 build/macosx-amd64 目录下看到如下内容：

```bash
-rw-r--r--   1 gaochao  staff   289K  7 23 13:39 Makefile
drwxr-xr-x  90 gaochao  staff   2.8K  7 23 15:50 bfd
-rw-r--r--   1 gaochao  staff   2.6K  7 23 13:39 config.cache
-rw-r--r--   1 gaochao  staff   5.1K  7 23 13:39 config.log
-rwxr-xr-x   1 gaochao  staff   9.5K  7 23 15:50 config.status
-rwxr-xr-x   1 gaochao  staff   587K  7 23 13:40 hsdis-amd64.dylib
drwxr-xr-x  10 gaochao  staff   320B  7 23 15:50 intl
drwxr-xr-x  64 gaochao  staff   2.0K  7 23 15:50 libiberty
drwxr-xr-x  26 gaochao  staff   832B  7 23 15:50 opcodes
-rw-r--r--   1 gaochao  staff    13B  7 23 13:39 serdep.tmp
```

这里的  hsdis-amd64.dylib 就是我们要的结果，我们将这个动态库拷贝到如下目录：

build/macosx-x86_64-normal-server-fastdebug/jdk/lib

这是 JVM 源码编译输出的目录，因为我编译的是 fastdebug 版本的 JVM，因此放到这里。这样，我们就可以通过 java 命令使用这个插件了。

下面我们定义一个如下的测试 java 代码：

```java
public class Count {
    private volatile int count = 0;
    private volatile int test = 0;
    public void testMethod() {
      count++;
      test++;
    }
}
```

定义个 Counter，其中计数通过 volatile 实现。然后我们在测试代码中调用这个 Counter：

```java
public class Hello {
  public static void main(String[] args) {
    Count count = new Count();
    for (int i = 0; i < 100000; i++) {
      count.testMethod();
    }
  }
}
```

注意，为了能够触发 JIT 优化，这里强制将 testMethod 方法执行 100000，这样才能将 JVM 中的代码跑热以生成 JIT 代码。
> 当然，也可以使用参数`-Xcomp`让虚拟机以编译模式执行代码，这样代码可以偷懒，不需要执行足够次数来预热都能触发JIT编译。

然后编译如上代码，然后在命令行执行如下命令来执行代码：

```bash
java -XX:+PrintAssembly -XX:CompileCommand=dontinline,Count.testMethod -XX:CompileCommand=compileonly,Count.testMethod Hello > out
```

这里解释一下上面的命令，首先我们使用 ```-XX:+PrintAssembly``` 参数使用 hsdis 插件获取 JIT 机器码，然后通过 XX:CompileCommand 参数指定不要将我们的代码内联优化，并且只编译 Count.testMethod 方法，其他方法我们不关心。

如果上面的命令顺利执行，我们会得到 out 输出文件，这个文件非常长，这里我们只看 testMethod 方法执行的机器码：

```bash
  0x00000001116b1234: mov    0xc(%rsi),%edi     ;*getfield count {reexecute=0 rethrow=0 return_oop=0}
                                                ; - Count::testMethod@2 (line 6)
  0x00000001116b1237: inc    %edi
  0x00000001116b1239: mov    %edi,0xc(%rsi)
  0x00000001116b123c: lock addl $0x0,0xffffffffffffffc0(%rsp)
                                                ;*putfield count {reexecute=0 rethrow=0 return_oop=0}
                                                ; - Count::testMethod@7 (line 6)
  0x00000001116b1242: mov    0x10(%rsi),%edi    ;*getfield test {reexecute=0 rethrow=0 return_oop=0}
                                                ; - Count::testMethod@12 (line 7)
  0x00000001116b1245: inc    %edi
  0x00000001116b1247: mov    %edi,0x10(%rsi)
  0x00000001116b124a: lock addl $0x0,0xffffffffffffffc0(%rsp)
                                                ;*putfield test {reexecute=0 rethrow=0 return_oop=0}
                                                ; - Count::testMethod@17 (line 7)
  0x00000001116b1250: add    $0x30,%rsp
  0x00000001116b1254: pop    %rbp
  0x00000001116b1255: mov    0x120(%r15),%r10
  0x00000001116b125c: test   %eax,(%r10)        ;   {poll_return}
```

我们在代码中针对变量进行 ++ 自增操作，因此可以看到首先通过 mov 读取原始值，然后通过 inc 指令将值增加 1，然后再通过 mov 指令将新的值推送到栈中，然后通过一个 lock addl 指令将 rsp 栈中的数据加 0，然后 ++ 自增操作就完毕了。前面的三个步骤我们都能明白，只是最后一个 lock addl 将 rsp 加 0 不太好理解，这里将一个值加 0 不相当于什么都没做吗？这句话是废话吗？其实不是的，我们需要了解下 lock 这个指令前缀是在做什么，我们需要查阅下 Intel 的 IA32 指令开发者手册 看下这个指令前缀的定义（[“LOCK—Assert LOCK# Signal Prefix” in Chapter 3, “Instruction Set Reference, A-L”](https://software.intel.com/sites/default/files/managed/39/c5/325462-sdm-vol-1-2abcd-3abcd.pdf#page=1158)）：

> Causes the processor’s LOCK# signal to be asserted during execution of the accompanying instruction **(turns the instruction into an atomic instruction)**. In a multiprocessor environment, the LOCK# signal ensures that the processor **has exclusive use of any shared memory** while the signal is asserted.  
The LOCK prefix can be prepended only to the following instructions and only to those forms of the instructions where the destination operand is a memory operand: ADD, ADC, AND, BTC, BTR, BTS, CMPXCHG, CMPXCH8B, CMPXCHG16B, DEC, INC, NEG, NOT, OR, SBB, SUB, XOR, XADD, and XCHG. If the LOCK prefix is used with one of these instructions and the source operand is a memory operand, an undefined opcode exception (#UD) may be generated. An undefined opcode exception will also be generated if the LOCK prefix is used with any instruction not in the above list. The XCHG instruction always asserts the LOCK# signal regardless of the presence or absence of the LOCK prefix.  
The LOCK prefix is typically used with the BTS instruction to perform a read-modify-write operation on a memory location in shared memory environment.  
The integrity of the LOCK prefix is not affected by the alignment of the memory field. Memory locking is observed for arbitrarily misaligned fields.

8.1.4小节中说明了该指令对Caches的效果，"[Effects of a LOCK Operation on Internal Processor Caches](https://software.intel.com/sites/default/files/managed/39/c5/325462-sdm-vol-1-2abcd-3abcd.pdf#page=3055)"：
> For the Intel486 and Pentium processors, the LOCK# signal is always asserted on the bus during a LOCK operation, even if the area of memory being locked is cached in the processor.  
For the P6 and more recent processor families, if the area of memory being locked during a LOCK operation is cached in the processor that is performing the LOCK operation as write-back memory and is completely contained in a cache line, the processor may not assert the LOCK# signal on the bus. Instead, it will modify the memory location internally and allow it’s cache coherency mechanism to ensure that the operation is carried out atomically. This operation is called “cache locking.” **The cache coherency mechanism automatically prevents two or more processors that have cached the same area of memory from simultaneously modifying data in that area**.

可以看到这里描述，是说通过 lock 可以在共享内存的系统上使得被修饰的指令成为一个排他性指令，并且防止其它处理器缓存被修改的共享内存，也就是说只要这个指令在执行了可以保证如下两件事情：

1. 修改完成的内容值，其他 CPU 核心可以立即看到
2. 修改的时候，其他 CPU 不能操作这个值，并且在 lock 之前的指令不能重排到这句话的后面

这其实就是 java 中的 volatile 的语义啊～

还有一个疑问，那就是为啥使用 lock 修饰 addl，而不是 nop 指令呢？其实上面的开发者手册说的很清楚，lock 后面不能跟 nop 的，只能跟 add 之类的指令。因此，JVM 就采用了 lock addl 将一个栈中的值加 0 这种人畜无害的操作实现 volatile 的语义。

通过上面的分析，我们也就知道了，在 java 中的 volatile 关键字修饰的变量的访问在 intel x86 CPU 上是通过 lock 修饰的 addl 指令实现的。

## 后记

到这里我们就全部介绍完毕了关于 Java 并发基石部分的内容了，我们从共享内存多核系统设计讲起，然后介绍了 Java 的内存模型，接着介绍了 JVM 创建一个线程的过程。然后重点介绍了 synchonized 和 wait/notify 的实现机制，最后我们介绍了下 volatile 关键字的语义以及 java 的实现方案。

探索底层技术是复杂的，也是枯燥的，因为这部分的内容有如下困难：

1. 原理复杂
2. 网上的资料及其稀少甚至没有，只能自己实验，不断尝试，不断失败，不断努力才能获得知识

但是，一旦你学会了研究底层技术的基本套路，那么你的技术发展道路将会出现不一样的风景，你不会对新技术感到迷茫，因为你知道这些东西知识换汤不换药，基本的机制和理论已经稳定了若干年了，在理论没有突破的情况下，技术上的突破是很难的。

底层技术好比内功心法，虽然不能立即增加你的功力，但是经过日积月累，你会犹如九阳神功护体，天下武功过眼即会，并且百毒不侵。与君共勉，祝你成功。

## 参考资料

### 基础理论

1. [多处理器编程艺术](https://book.douban.com/subject/3901836/)
2. [深入理解并行编程](https://book.douban.com/subject/27078711/)
3. [Introduction to Parallel Computing](https://computing.llnl.gov/tutorials/parallel_comp/)

### Java 实践

1. [深入理解Java虚拟机（第2版）](https://book.douban.com/subject/24722612/)
2. [Java并发编程实战](https://book.douban.com/subject/10484692/)
3. [Java并发编程之美](https://book.douban.com/subject/30351286/)
4. [concurrent programming in java design principles and pattern](https://book.douban.com/subject/1440218/)
5. [The Java Virtual Machine Instruction Set](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.getfield)
6. [JSR 133](https://download.oracle.com/otndocs/jcp/memory_model-1.0-pfd-spec-oth-JSpec/)
7. [Thread and Locks](https://docs.oracle.com/javase/specs/jvms/se6/html/Threads.doc.html)
8. [JSR 133 中文版](http://ifeve.com/wp-content/uploads/2014/03/JSR133中文版.pdf)
9. [The JSR-133 Cookbook for Compiler Writers by Doug Lea](http://gee.cs.oswego.edu/dl/jmm/cookbook.html)
10. [Synchronization and Object Locking](https://wiki.openjdk.java.net/display/HotSpot/Synchronization)
11. [Biased Locking in HotSpot](https://blogs.oracle.com/dave/biased-locking-in-hotspot)
12. [Memory Barriers and JVM Concurrency](https://www.infoq.com/articles/memory_barriers_jvm_concurrency/)
13. [Memory barrier](https://en.wikipedia.org/wiki/Memory_barrier)
14. [Performance Improvements of Contended Java Monitor from JDK 6 to 9](https://vmlens.com/articles/performance-improvements-of-java-monitor/)
15. [Java 内存访问重排序的研究](https://tech.meituan.com/2014/09/23/java-memory-reordering.html)
16. [JVM PrintAssembly](https://wiki.openjdk.java.net/display/HotSpot/PrintAssembly)
17. [Intel IA32 Dev](https://software.intel.com/sites/default/files/managed/39/c5/325462-sdm-vol-1-2abcd-3abcd.pdf)
18. [Java 对象内存布局](https://cgiirw.github.io/2018/04/16/JVM_Oop_Desc/)
19. [快速编译OpenJDK的Dockers镜像](https://hub.docker.com/r/bolingcavalry/openjdksrc11)

### C/C++ 相关的

1. [C/C++ 中的 volatile 语义](https://liam.page/2018/01/18/volatile-in-C-and-Cpp/)

### Pthread Library

1. [POSIX多线程程序设计中文版](https://book.douban.com/subject/1236825/)
2. [POSIX Threads Programming](https://computing.llnl.gov/tutorials/pthreads/)

### Papers

1. [Synchronization- Related Atomic Operations with Biased Locking and Bulk Rebiasing](https://pdfs.semanticscholar.org/bc8f/7a35b87b452924e180ed15b58f049bcac9db.pdf)
2. [Thin Lock](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.90.664&amp;rep=rep1&amp;type=pdf)
3. [JUC Framework by Doug Lea](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
4. [CLH Lock](ftp://ftp.cs.washington.edu/tr/1993/02/UW-CSE-93-02-02.pdf)
5. [MCS Lock](https://www.cs.rochester.edu/u/scott/papers/1991_TOCS_synch.pdf)
