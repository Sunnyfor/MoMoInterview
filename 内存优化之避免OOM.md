# 内存优化之避免 OOM

Android的内存优化是性能优化中很重要的一部分，而避免OOM又是内存优化中比较核心的一点，这是一篇关于内存优化中如何避免OOM的总结性概要文章，内容大多都是和OOM有关的实践总结概要。

## 一、Android的内存管理机制

Google 在 Android 的官网上有这样一篇文章，初步介绍了 Android 是如何管理应用的进程与内存分配：[http://developer.android.com/training/articles/memory.html。](http://developer.android.com/training/articles/memory.html%E3%80%82) Android 系统的 Dalvik 虚拟机扮演了常规的内存垃圾自动回收的角色，**Android 系统没有为内存提供交换区，它使用 paging 与 memory-mapping(mmapping) 的机制来管理内存**，下面简要概述一些 Android 系统中重要的内存管理基础概念

### 1. 共享内存

Android 系统通过下面几种方式来实现共享内存：

- Android 应用的进程都是从一个叫做 Zygote 的进程fork出来的。Zygote 进程在系统启动并且载入通用的 framework 的代码与资源之后开始启动。为了启动一个新的程序进程，系统会 fork Zygote 进程生成一个新的进程，然后在新的进程中加载并运行应用程序的代码。这使得大多数的 RAM pages 被用来分配给 framework 的代码，同时使得 RAM 资源能够在应用的所有进程之间进行共享。
- 大多数 **static** 的数据被 mmapped 到一个进程中。这不仅仅使得同样的数据能够在进程间进行共享，而且使得它能够在需要的时候被 paged out。常见的static 数据包括 Dalvik Code，app resources，so 文件等。
- 大多数情况下，Android 通过**显式的分配共享内存区域**(例如 ashmem 或者 gralloc)**来实现动态 RAM 区域能够在不同进程之间进行共享的机制**。例如，Window Surface 在 App 与 Screen Compositor 之间使用共享的内存，Cursor Buffers 在 Content Provider 与 Clients 之间共享内存。

### 2. 分配与回收内存

- 每一个进程的 Dalvik heap 都反映了使用内存的占用范围。这就是通常逻辑意义上提到的 Dalvik Heap Size，它可以随着需要进行增长，但是增长行为会有一个系统为它设定的上限。
- 逻辑上讲的 Heap Size 和实际物理意义上使用的内存大小是不对等的，Proportional Set Size(PSS) 记录了应用程序自身占用以及和其他进程进行共享的内存。
- Android 系统并不会对 Heap 中空闲内存区域做碎片整理。系统仅仅会在新的内存分配之前判断 Heap 的尾端剩余空间是否足够，如果空间不够会触发 gc操作，从而腾出更多空闲的内存空间。在 Android 的高级系统版本里面针对 Heap 空间有一个 Generational Heap Memory 的模型，最近分配的对象会存放在 Young Generation 区域，当这个对象在这个区域停留的时间达到一定程度，它会被移动到 Old Generation，最后累积一定时间再移动到 Permanent Generation 区域。系统会根据内存中不同的内存数据类型分别执行不同的 gc操作。例如，刚分配到 Young Generation 区域的对象通常更容易被销毁回收，同时在 Young Generation 区域的 gc 操作速度会比 Old Generation 区域的gc操作速度更快。如下图所示：

    ![](http://hukai.me/images/memory_mode_generation.png)

    ![](http://hukai.me/images/android_memory_gc_mode.png)

    每一个 Generation 的内存区域都有固定的大小，随着新的对象陆续被分配到此区域，当这些对象总的大小快达到这一级别内存区域的阀值时，会触发 GC 的操作，以便腾出空间来存放其他新的对象。如下图所示：

    ![](http://hukai.me/images/gc_threshold.png)

    通**常情况下，GC 发生的时候，所有的线程都是会被暂停的**。执行 GC 所占用的时间和它发生在哪一个 Generation 也有关系，Young Generation 中的每次 GC 操作时间是最短的，Old Generation 其次，Permanent Generation 最长。执行时间的长短也和当前 Generation 中的对象数量有关，遍历树结构查找 20000 个对象比起遍历 50 个对象自然是要慢很多的。

### 3. 限制应用的内存

为了整个 Android 系统的内存控制需要，**Android 系统为每一个应用程序都设置了一个硬性的 Dalvik Heap Size 最大限制阈值**，这个阈值在不同的设备上会因为 RAM 大小不同而各有差异。如果你的应用占用内存空间已经接近这个阈值，此时再尝试分配内存的话，很容易引起`OutOfMemoryError` 的错误。

`ActivityManager.getMemoryClass()` 可以用来查询当前应用的 Heap Size 阈值，这个方法会返回一个整数，表明你的应用的 Heap Size 阈值是多少Mb(megabates)。

### 4. 应用切换操作

Android 系统并不会在用户切换应用的时候做交换内存的操作。Android 会把那些不包含 Foreground 组件的应用进程放到 **LRU Cache** 中。例如，当用户开始启动了一个应用，系统会为它创建了一个进程，但是当用户离开这个应用，此进程并不会立即被销毁，而是会被放到系统的 Cache 当中，如果用户后来再切换回到这个应用，此进程就能够被马上完整的恢复，从而实现应用的快速切换。

如果你的应用中有一个被缓存的进程，这个进程会占用一定的内存空间，它会对系统的整体性能有影响。因此当系统开始进入Low Memory 的状态时，它会由系统根据 LRU 的规则与应用的优先级，内存占用情况以及其他因素的影响综合评估之后决定是否被杀掉。

对于那些非 foreground 的进程，Android 系统是如何判断 Kill 掉哪些进程的问题，请参考[Processes and Threads](http://developer.android.com/guide/components/processes-and-threads.html)。

## 二、OOM

前面我们提到过使用 `getMemoryClass()` 的方法可以得到 Dalvik Heap 的阈值。简要的获取某个应用的内存占用情况可以参考下面的示例（ 关于更多内存查看的知识，可以参考这篇官方教程：Investigating Your RAM Usage ）

### 1. 查看内存使用情况

- 通过命令行查看内存详细占用情况：

    ![](http://hukai.me/images/android_perf_oom_dumpsys_meminfo.png)

- 通过Android Studio的Memory Monitor查看内存中Dalvik Heap的实时变化

    ![](http://hukai.me/images/android_perf_oom_studio_mem_monitor.png)

    ![](http://hukai.me/images/memory_monitor_free_allocation.png)

    ![](http://hukai.me/images/memory_monitor_gc_event.png)

### 2. **发生OOM的条件**

关于Native Heap，Dalvik Heap，Pss 等内存管理机制比较复杂，这里不展开描述。简单的说，通过不同的内存分配方式（malloc/mmap/JNIEnv/etc）对不同的对象（bitmap，etc）进行操作会因为 Android 系统版本的差异而产生不同的行为，对 Native Heap 与Dalvik Heap 以及 OOM 的判断条件都会有所影响。在 2.x的系统上，我们常常可以看到 Heap Size 的 total 值明显超过了通过getMemoryClass() 获取到的阈值而不会发生 OOM 的情况，那么针对2.x与4.x的Android系统，到底是如何判断会发生OOM呢？

- Android 2.x 系统 GC LOG 中的 **dalvik allocated + external allocated + 新分配的大小 >= `getMemoryClass()`** 值的时候就会发生OOM。 例如，假设有这么一段 Dalvik 输出的 `GC LOG：GC_FOR_MALLOC free 2K, 13% free 32586K/37455K, external 8989K/10356K, paused 20ms`，那么 32586 + 8989 + (新分配23975) = 65550 > 64M 时，就会发生OOM。
- Android 4.x 系统 Android 4.x 的系统废除了 external 的计数器，类似 bitmap 的分配改到 dalvik 的 java heap 中申请，只要 **allocated + 新分配的内存 >= `getMemoryClass()`** 的时候就会发生 OOM，如下图所示（虽然图示演示的是 art 运行环境，但是统计规则还是和 dalvik 保持一致）

    ![](http://hukai.me/images/android_perf_oom_gc_log.png)

## 如何避免OOM

前面介绍了一些基础的内存管理机制以及 OOM 的基础知识，那么在实践操作当中，有哪些指导性的规则可以参考呢？归纳下来，可以从四个方面着手，首先是减小对象的内存占用，其次是内存对象的重复利用，然后是避免对象的内存泄露，最后是内存使用策略优化。

### 1. **减小对象的内存占用**

避免 OOM 的第一步就是要尽量减少新分配出来的对象占用内存的大小，尽量使用更加轻量的对象。

**1）使用更加轻量的数据结构**

例如，我们可以考虑使用 ArrayMap/SparseArray 而不是 HashMap 等传统数据结构，下图演示了 HashMap 的简要工作原理，相比起 Android 系统专门为移动操作系统编写的 ArrayMap 容器，在大多数情况下，都显示效率低下，更占内存。通常的 HashMap 的实现方式更加消耗内存，因为它需要一个额外的实例对象来记录 Mapping 操作。另外，**SparseArray 更加高效在于他们避免了对 key 与 value 的 autobox 自动装箱，并且避免了装箱后的解箱**。

![](http://hukai.me/images/android_perf_3_arraymap_key_value.png)

关于更多ArrayMap/SparseArray的讨论，请参考[http://hukai.me/android-performance-patterns-season-3/](http://hukai.me/android-performance-patterns-season-3/)的前三个段落

**2）避免在 Android 里面使用 Enum**

Android 官方培训课程提到过**“**Enums often require more than twice as much memory as static constants. You should strictly avoid using enums on Android.**”**，具体原理请参考 [http://hukai.me/android-performance-patterns-season-3/](http://hukai.me/android-performance-patterns-season-3/)，所以请避免在 Android 里面使用到枚举。

**3）减小 Bitmap 对象的内存占用**

Bitmap 是一个极容易消耗内存的大胖子，减小创建出来的 Bitmap 的内存占用是很重要的，通常来说有下面2个措施：

- inSampleSize：缩放比例，在把图片载入内存之前，我们需要先计算出一个合适的缩放比例，避免不必要的大图载入。
- decode format：解码格式，选择ARGB_8888/RBG_565/ARGB_4444/ALPHA_8，存在很大差异。

**4）使用更小的图片**

在设计给到资源图片的时候，我们需要特别留意这张图片是否存在可以压缩的空间，是否可以使用一张更小的图片。尽量使用更小的图片不仅仅可以减少内存的使用，还可以避免出现大量的InflationException。假设有一张很大的图片被XML文件直接引用，很有可能在初始化视图的时候就会因为内存不足而发生InflationException，这个问题的根本原因其实是发生了OOM。

### 2. **内存对象的重复利用**

大多数对象的复用，最终实施的方案都是利用**对象池**技术，要么是在编写代码的时候显式的在程序里面去创建对象池，然后处理好复用的实现逻辑，要么就是利用系统框架既有的某些复用特性达到减少对象的重复创建，从而减少内存的分配与回收。

![](http://hukai.me/images/android_perf_2_object_pool.png)

在 Android 上面最常用的一个缓存算法是 **LRU(Least Recently Use)**，简要操作原理如下图所示：

![](http://hukai.me/images/android_perf_2_lru_mode.png)

**1）复用系统自带的资源**

Android系统本身内置了很多的资源，例如字符串/颜色/图片/动画/样式以及简单布局等等，这些资源都可以在应用程序中直接引用。这样做不仅仅可以减少应用程序的自身负重，减小APK的大小，另外还可以一定程度上减少内存的开销，复用性更好。但是也有必要留意Android系统的版本差异性，对那些不同系统版本上表现存在很大差异，不符合需求的情况，还是需要应用程序自身内置进去。

**2）注意在 ListView/GridView 等出现大量重复子组件的视图里面对ConvertView 的复用**

![](http://hukai.me/images/android_perf_oom_listview_recycle.png)

**3）Bitmap 对象的复用** [Bitmap 复用](https://www.notion.so/a2272f79-d41b-4c66-abcf-1342d51dfaf5) 

**4）避免在onDraw方法里面执行对象的创建**

类似onDraw等频繁调用的方法，一定需要注意避免在这里做创建对象的操作，因为他会迅速增加内存的使用，而且很容易引起频繁的 gc，甚至是内存抖动。

[在 onDraw() 中避免内存分配](https://www.notion.so/bb2db01d-4ab4-4652-a03f-d7ccbcf10e1f) 

**5）StringBuilder**

在有些时候，代码中会需要使用到大量的字符串拼接的操作，这种时候有必要考虑使用 StringBuilder 来替代频繁的“+”。

### 3. **避免对象的内存泄露**

内存对象的泄漏，会导致一些不再使用的对象无法及时释放，这样一方面占用了宝贵的内存空间，很容易导致后续需要分配内存的时候，空闲空间不足而出现OOM。显然，这还使得每级 Generation 的内存区域可用空间变小，**gc 就会更容易被触发，容易出现内存抖动，从而引起性能问题。**

![](http://hukai.me/images/android_perf_3_leak.png)

最新的 LeakCanary 开源控件，可以很好的帮助我们发现内存泄露的情况，更多关于 LeakCanary 的介绍，请看这里[https://github.com/square/leakcanary](https://github.com/square/leakcanary)(中文使用说明[http://www.liaohuqiu.net/cn/posts/leak-canary-read-me/](http://www.liaohuqiu.net/cn/posts/leak-canary-read-me/))。另外也可以使用传统的MAT工具查找内存泄露，请参考这里[http://android-developers.blogspot.pt/2011/03/memory-analysis-for-android.html](http://android-developers.blogspot.pt/2011/03/memory-analysis-for-android.html)（便捷的中文资料[http://androidperformance.com/2015/04/11/AndroidMemory-Usage-Of-MAT/](http://androidperformance.com/2015/04/11/AndroidMemory-Usage-Of-MAT/)）

**1）注意Activity的泄漏**

通常来说，Activity 的泄漏是内存泄漏里面最严重的问题，它占用的内存多，影响面广，我们需要特别注意以下两种情况导致的 Activity泄漏：

- 内部类引用导致 Activity 的泄漏
最典型的场景是 Handler 导致的 Activity 泄漏，如果 **Handler 中有延迟的任务或者是等待执行的任务队列过长**，都有可能因为 Handler 继续执行而导致Activity 发生泄漏。此时的引用关系链是 Looper -> MessageQueue -> Message -> Handler -> Activity。
为了解决这个问题，**可以在 UI 退出之前，执行 remove Handler 消息队列中的消息与runnable对象。或者是使用 Static + WeakReference 的方式来达到断开 Handler 与 Activity 之间存在引用关系的目的。**
- **Activity Context 被传递到其他实例中**，这可能导致自身被引用而发生泄漏
内部类引起的泄漏不仅仅会发生在 Activity 上，其他任何内部类出现的地方，都需要特别留意！我们可以考虑尽量使用 static 类型的内部类，同时使用 WeakReference 的机制来避免因为互相引用而出现的泄露。

**2）考虑使用 Application Context 而不是Activity Context**

对于大部分非必须使用 Activity Context 的情况（Dialog的Context就必须是Activity Context），我们都可以考虑使用 Application Context 而不是 Activity 的 Context，这样可以避免不经意的 Activity 泄露。

**3）注意临时 Bitmap 对象的及时回收**

虽然在大多数情况下，我们会对 Bitmap 增加缓存机制，但是在某些时候，部分Bitmap 是需要及时回收的。例如临时创建的某个相对比较大的 bitmap 对象，在经过变换得到新的 bitmap 对象之后，应该尽快回收原始的 bitmap，这样能够更快释放原始 bitmap 所占用的空间。

需要特别留意的是 Bitmap 类里面提供的 createBitmap() 方法：

![](http://hukai.me/images/android_perf_oom_create_bitmap.png)

这个函数返回的 bitmap 有可能和 source bitmap 是同一个，在回收的时候，需要特别检查 source bitmap 与 return bitmap 的引用是否相同，只有在不等的情况下，才能够执行 source bitmap 的 recycle 方法。

**4）注意监听器的注销**

在 Android 程序里面存在很多需要 register 与 unregister 的监听器，我们需要确保在合适的时候及时 unregister 那些监听器。自己手动 add 的 listener，需要记得及时 remove 这个 listener。

**5）注意缓存容器中的对象泄漏**

有时候，我们为了提高对象的复用性把某些对象放到缓存容器中，可是如果这些对象没有及时从容器中清除，也是有可能导致内存泄漏的。例如，针对 2.3 的系统，如果把 drawable 添加到缓存容器，因为 drawable 与 View 的强应用，很容易导致 activity 发生泄漏。而从 4.0 开始，就不存在这个问题。解决这个问题，需要对 2.3 系统上的缓存 drawable 做特殊封装，处理引用解绑的问题，避免泄漏的情况。

**6）注意WebView的泄漏**

Android 中的 WebView 存在很大的兼容性问题，不仅仅是 Android 系统版本的不同对 WebView 产生很大的差异，另外不同的厂商出货的 ROM 里面 WebView也存在着很大的差异。更严重的是标准的 WebView 存在内存泄露的问题，看这里[WebView causes memory leak - leaks the parent Activity](https://code.google.com/p/android/issues/detail?id=5067)。所以通常根治这个问题的办法是为 WebView 开启另外一个进程，通过 AIDL 与主进程进行通信，WebView 所在的进程可以根据业务的需要选择合适的时机进行销毁，从而达到内存的完整释放。

**7）注意Cursor对象是否及时关闭**

在程序中我们经常会进行查询数据库的操作，但时常会存在不小心使用Cursor之后没有及时关闭的情况。这些Cursor的泄露，反复多次出现的话会对内存管理产生很大的负面影响，我们需要谨记对Cursor对象的及时关闭。

### 4. **内存使用策略优化**

**1）谨慎使用 large heap**

正如前面提到的，Android 设备根据硬件与软件的设置差异而存在不同大小的内存空间，他们为应用程序设置了不同大小的 Heap 限制阈值。你可以通过调用`getMemoryClass()`来获取应用的可用Heap大小。在一些特殊的情景下，你可以通过在`manifest`的`application`标签下添加`largeHeap=true`的属性来为应用声明一个更大的 heap 空间。然后，你可以通过`getLargeMemoryClass()`来获取到这个更大的 heap size 阈值。然而，声明得到更大 Heap 阈值的本意是为了一小部分会消耗大量 RAM 的应用(例如一个大图片的编辑应用)。不要轻易的因为你需要使用更多的内存而去请求一个大的 Heap Size。只有当你清楚的知道哪里会使用大量的内存并且知道为什么这些内存必须被保留时才去使用 large heap。因此请谨慎使用 large heap 属性。使用额外的内存空间会影响系统整体的用户体验，并且会使得每次 gc 的运行时间更长。在任务切换时，系统的性能会大打折扣。另外, large heap 并不一定能够获取到更大的 heap。在某些有严格限制的机器上，large heap 的大小和通常的 heap size 是一样的。因此即使你申请了l arge heap，你还是应该通过执行 `getMemoryClass()` 来检查实际获取到的 heap 大小。

**2）综合考虑设备内存阈值与其他因素设计合适的缓存大小**

例如，在设计 ListView 或者 GridView 的 Bitmap LRU 缓存的时候，需要考虑的点有：

- 应用程序剩下了多少可用的内存空间?
- 有多少图片会被一次呈现到屏幕上？有多少图片需要事先缓存好以便快速滑动时能够立即显示到屏幕？
- 设备的屏幕大小与密度是多少? 一个 xhdpi 的设备会比 hdpi 需要一个更大的 Cache 来 hold 住同样数量的图片。
- 不同的页面针对 Bitmap 的设计的尺寸与配置是什么，大概会花费多少内存？
- 页面图片被访问的频率？是否存在其中的一部分比其他的图片具有更高的访问频繁？如果是，也许你想要保存那些最常访问的到内存中，或者为不同组别的位图(按访问频率分组)设置多个 LruCache 容器。

**3）onLowMemory() 与 onTrimMemory()**

Android 用户可以随意在不同的应用之间进行快速切换。为了让 background 的应用能够迅速的切换到 foreground，每一个 background 的应用都会占用一定的内存。Android 系统会根据当前的系统的内存使用情况，决定回收部分background 的应用内存。如果 background 的应用从暂停状态直接被恢复到forground，能够获得较快的恢复体验，如果 background 应用是从 Kill 的状态进行恢复，相比之下就显得稍微有点慢。

![](http://hukai.me/images/android_perf_3_memory_bg_2_for.png)

- **onLowMemory()**：Android 系统提供了一些回调来通知当前应用的内存使用情况，通常来说，当所有的 background 应用都被 kill 掉的时候，forground 应用会收到 onLowMemory() 的回调。在这种情况下，需要尽快释放当前应用的非必须的内存资源，从而确保系统能够继续稳定运行。
- **onTrimMemory(int)**：Android 系统从 4.0 开始还提供了 onTrimMemory() 的回调，当系统内存达到某些条件的时候，所有正在运行的应用都会收到这个回调，同时在这个回调里面会传递以下的参数，代表不同的内存使用情况，收到 onTrimMemory() 回调的时候，需要根据传递的参数类型进行判断，合理的选择释放自身的一些内存占用，一方面可以提高系统的整体运行流畅度，另外也可以避免自己被系统判断为优先需要杀掉的应用。
    - 下面介绍了各种不同的回调参数：
        - **TRIM_MEMORY_UI_HIDDEN**：你的应用程序的所有 UI 界面被隐藏了，即用户点击了Home键或者Back键退出应用，导致应用的UI界面完全不可见。这个时候应该释放一些不可见的时候非必须的资源
        当程序正在前台运行的时候，可能会接收到从 onTrimMemory() 中返回的下面的值之一：
        - **TRIM_MEMORY_RUNNING_MODERATE**：你的应用正在运行并且不会被列为可杀死的。但是设备此时正运行于低内存状态下，系统开始触发杀死LRU Cache中的Process的机制。
        - **TRIM_MEMORY_RUNNING_LOW**：你的应用正在运行且没有被列为可杀死的。但是设备正运行于更低内存的状态下，你应该释放不用的资源用来提升系统性能。
        - **TRIM_MEMORY_RUNNING_CRITICAL**：你的应用仍在运行，但是系统已经把LRU Cache中的大多数进程都已经杀死，因此你应该立即释放所有非必须的资源。如果系统不能回收到足够的RAM数量，系统将会清除所有的LRU缓存中的进程，并且开始杀死那些之前被认为不应该杀死的进程，例如那个包含了一个运行态Service的进程。
        当应用进程退到后台正在被 Cached 的时候，可能会接收到从onTrimMemory()中返回的下面的值之一：
        - **TRIM_MEMORY_BACKGROUND**: 系统正运行于低内存状态并且你的进程正处于LRU缓存名单中最不容易杀掉的位置。尽管你的应用进程并不是处于被杀掉的高危险状态，系统可能已经开始杀掉LRU缓存中的其他进程了。你应该释放那些容易恢复的资源，以便于你的进程可以保留下来，这样当用户回退到你的应用的时候才能够迅速恢复。
        - **TRIM_MEMORY_MODERATE**: 系统正运行于低内存状态并且你的进程已经已经接近LRU名单的中部位置。如果系统开始变得更加内存紧张，你的进程是有可能被杀死的。
        - **TRIM_MEMORY_COMPLETE**: 系统正运行于低内存的状态并且你的进程正处于LRU名单中最容易被杀掉的位置。你应该释放任何不影响你的应用恢复状态的资源。

            ![](http://hukai.me/images/android_perf_3_memory_ontrimmemory.png)

- 因为 onTrimMemory() 的回调是在API 14才被加进来的，对于老的版本，你可以使用 onLowMemory() 回调来进行兼容。onLowMemory 相当与TRIM_MEMORY_COMPLETE。
- 请注意：**当系统开始清除 LRU 缓存中的进程时，虽然它首先按照 LRU 的顺序来执行操作，但是它同样会考虑进程的内存使用量以及其他因素。占用越少的进程越容易被留下来。**

**4）资源文件需要选择合适的文件夹进行存放**

我们知道 `hdpi/xhdpi/xxhdpi` 等等不同 dpi 的文件夹下的图片在不同的设备上会经过 scale 的处理。例如我们只在 hdpi 的目录下放置了一张 100*100 的图片，那么根据换算关系，`xxhdpi` 的手机去引用那张图片就会被拉伸到 200*200。需要注意到在这种情况下，内存占用是会显著提高的。对于不希望被拉伸的图片，需要放到 assets 或者 nodpi 的目录下。

**5）Try catch 某些大内存分配的操作**

在某些情况下，我们需要事先评估那些可能发生 OOM 的代码，对于这些可能发生 OOM 的代码，加入 catch 机制，可以考虑在 catch 里面尝试一次降级的内存分配操作。例如 decode bitmap 的时候，catch 到 OOM，可以尝试把采样比例再增加一倍之后，再次尝试 decode。

**6）谨慎使用static对象**

因为 static 的生命周期过长，和应用的进程保持一致，使用不当很可能导致对象泄漏，在 Android 中应该谨慎使用 static 对象。

![](http://hukai.me/images/android_perf_3_leak_static.png)

**7）特别留意单例对象中不合理的持有**

虽然单例模式简单实用，提供了很多便利性，但是因为单例的生命周期和应用保持一致，使用不合理很容易出现持有对象的泄漏。

**8）珍惜 Services 资源**

如果你的应用需要在后台使用 service，除非它被触发并执行一个任务，否则其他时候 Service 都应该是停止状态。另外需要注意当这个 service 完成任务之后因为停止 service 失败而引起的内存泄漏。 当你启动一个 Service，系统会倾向为了保留这个 Service 而一直保留 Service 所在的进程。这使得进程的运行代价很高，因为系统没有办法把 Service 所占用的RAM空间腾出来让给其他组件，另外 Service 还不能被Paged out。这减少了系统能够存放到 LRU 缓存当中的进程数量，它会影响应用之间的切换效率，甚至会导致系统内存使用不稳定，从而无法继续保持住所有目前正在运行的 service。 *建议使用 [IntentService](http://developer.android.com/reference/android/app/IntentService.html)，它会在处理完交代给它的任务之后尽快结束自己*。更多信息，请阅读[Running in a Background Service](http://developer.android.com/training/run-background-service/index.html)。

**9）优化布局层次，减少内存消耗**

越扁平化的视图布局，占用的内存就越少，效率越高。我们需要尽量保证布局足够扁平化，**当使用系统提供的 View 无法实现足够扁平的时候考虑使用自定义View 来达到目的**。

**10）谨慎使用“抽象”编程**

很多时候，开发者会使用抽象类作为”好的编程实践”，因为抽象能够提升代码的灵活性与可维护性。然而，抽象会导致一个显著的额外内存开销：他们需要同等量的代码用于可执行，那些代码会被 mapping 到内存中，因此如果你的抽象没有显著的提升效率，应该尽量避免他们。

**11）使用 nano protobufs 序列化数据**

Protocol buffers 是由 Google 为序列化结构数据而设计的，一种语言无关，平台无关，具有良好的扩展性。类似 XML，却比 XML 更加轻量，快速，简单。如果你需要为你的数据实现序列化与协议化，建议使用 nano protobufs。关于更多细节，请参考[protobuf readme](https://android.googlesource.com/platform/external/protobuf/+/master/java/README.txt)的 ”Nano version” 章节。

**12）谨慎使用依赖注入框架**

使用类似 Guice 或者 RoboGuice 等框架注入代码，在某种程度上可以简化你的代码。下面是使用 RoboGuice 前后的对比图：

![](http://hukai.me/images/android_perf_oom_roboguice_1.png)

![](http://hukai.me/images/android_perf_oom_roboguice_2.png)

使用 RoboGuice 之后，代码是简化了不少。**然而，那些注入框架会通过扫描你的代码执行许多初始化的操作，这会导致你的代码需要大量的内存空间来mapping 代码**，而且 mapped pages 会长时间的被保留在内存中。除非真的很有必要，建议谨慎使用这种技术。

**13）谨慎使用多进程**

使用多进程可以把应用中的部分组件运行在单独的进程当中，这样可以扩大应用的内存占用范围，但是这个技术必须谨慎使用，绝大多数应用都不应该贸然使用多进程，一方面是因为使用多进程会使得代码逻辑更加复杂，另外如果使用不当，它可能反而会导致显著增加内存。当你的应用需要运行一个常驻后台的任务，而且这个任务并不轻量，可以考虑使用这个技术。

一个典型的例子是创建一个可以长时间后台播放的 Music Player。如果整个应用都运行在一个进程中，当后台播放的时候，前台的那些UI资源也没有办法得到释放。类似这样的应用可以切分成 2 个进程：一个用来操作 UI，另外一个给后台的 Service。

**14）使用 ProGuard 来剔除不需要的代码**

[ProGuard](http://developer.android.com/tools/help/proguard.html) 能够通过移除不需要的代码，重命名类，域与方法等等对代码进行压缩，优化与混淆。使用 ProGuard 可以使得你的代码更加紧凑，这样能够减少 mapping代码所需要的内存空间。

**15）谨慎使用第三方 libraries**

很多开源的 library 代码都不是为移动网络环境而编写的，如果运用在移动设备上，并不一定适合。即使是针对 Android 而设计的 library，也需要特别谨慎，特别是在你不知道引入的 library 具体做了什么事情的时候。例如，其中一个 library 使用的是 nano protobufs, 而另外一个使用的是 micro protobufs。这样一来，在你的应用里面就有 2 种 protobuf 的实现方式。这样类似的冲突还可能发生在输出日志，加载图片，缓存等等模块里面。另外不要为了 1 个或者 2 个功能而导入整个 library，**如果没有一个合适的库与你的需求相吻合，你应该考虑自己去实现，而不是导入一个大而全的解决方案。**

**16）考虑不同的实现方式来优化内存占用**

在某些情况下，设计的某个方案能够快速实现需求，但是这个方案却可能在内存占用上表现的效率不够好。例如：

![](http://hukai.me/images/android_perf_2_waer_animation.png)

对于上面这样一个时钟表盘的实现，最简单的就是使用很多张包含指针的表盘图片，使用帧动画实现指针的旋转。但是如果把指针扣出来，单独进行旋转绘制，显然比载入 N 多张图片占用的内存要少很多。当然这样做，代码复杂度上会有所增加，这里就需要在优化内存占用与实现简易度之间进行权衡了。

---

写在最后：

- 设计风格很大程度上会影响到程序的内存与性能，相对来说，如果大量使用类似 Material Design 的风格，不仅安装包可以变小，还可以减少内存的占用，渲染性能与加载性能都会有一定的提升。
- 内存优化并不就是说程序占用的内存越少就越好，如果因为想要保持更低的内存占用，而频繁触发执行 gc 操作，在某种程度上反而会导致应用性能整体有所下降，这里需要综合考虑做一定的权衡。
- Android 的内存优化涉及的知识面还有很多：内存管理的细节，垃圾回收的工作原理，如何查找内存泄漏等等都可以展开讲很多。OOM 是内存优化当中比较突出的一点，尽量减少 OOM 的概率对内存优化有着很大的意义。