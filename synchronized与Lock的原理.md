# synchronized 与 Lock 的原理

`synchronized` 是 Java 中最基本的**互斥同步**手段

**synchronized 的原理**

1. `synchronized` 关键字经过编译之后，会在同步块的前后分别形成 `monitorenter` 和 `monitorexit` 这两个字节码。这两个字节码都需要一个 reference 类型的参数来指明要锁定和解锁的对象。
    - 如果代码中指定了对象参数，那就是这个对象的 reference
    `synchronized(monitor1) { ... };`
    `synchronized(this) { ... };`
    - 如果未明确指定
        - 修饰的是实例方法：对象实例为锁对象
        也就是给方法头加修饰
        - 修饰的是类方法：Class 对象为锁对象
        也就是给 `static` 方法头加修饰
2. 根据 Java 虚拟机的规范，在执行 `monitorenter` 指令的时，**首先要尝试获取对象的锁**（每个 Java 对象就有一把看不见的锁，称之为内部锁，也就是 Monitor 锁）。
    - 如果这个对象没被锁定，或者当前线程已经拥有这个对象的锁，则把锁的**计数器 +1**
    - 相应的，执行 `monitorexit` 指令时，会将锁的**计数器 -1**
    - 当**计数器为 0** 时，锁就被释放。如果获取锁对象失败，那么当前线程就要**阻塞**等待，直到对象锁被另一个线程释放为止
    - [Monitor](https://www.notion.so/fcd7a62d-7397-4697-86d2-fd3e479b7b5a)

---

**为什么 synchronized 会影响代码执行效率？**

Java 的线程是映射到操作系统的原生线程之上的，如果要阻塞或唤醒一个线程，都需要操作系统来帮忙完成，这就需要从用户态转换到核心态中，因此**状态转换需要耗费很多的处理器时间**。对于代码简单的同步块（如被 synchronized 修饰的 `getter()` 或 `setter()` 方法），**状态转换消耗的时间有可能比用户代码执行的时间还要长。**

所以 synchronized 是 Java 语言中一个**重量级**（Heavyweight）的操作，有经验的程序员都会在确实必要的情况下才使用这种操作。

而虚拟机本身也会进行一些优化，譬如在通知操作系统阻塞线程之前加入一段自旋等待过程，避免频繁地切入到核心态之中。

---

**Lock**

我们还可以使用 `java.util.concurrent` 包中的可重入锁 `ReentrantLock` 来实现同步，在基本用法上和 `synchronized` 很相似，他们都具备一样的线程重入特征，知识在代码写法上有点区别

- ReentrantLock 是 **API 层面**的互斥锁，`lock()` 和 `unlock()` 配合 `try/finally` 来完成
- synchronized 是**原生语法层面**的互斥锁

ReentrantLock 还增加了 3 项高级功能：

1. 等待可中断

    是指当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情，可中断特性对处理执行时间非常长的同步块很有帮助。

2. 可实现公平锁

    公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁；而非公平锁则不保证这一点，在锁被释放时，任何一个等待锁的线程都有机会获得锁。**synchronized 中的锁是非公平的**，**ReentrantLock 默认情况下也是非公平的，但可以通过带布尔值的构造函数要求使用公平锁**。

3. 锁可以绑定多个条件

    是指一个 **ReentrantLock 对象可以同时绑定多个 Condition 对象**，而在synchronized 中，锁对象的 wait() 和 notify() 或 notifyAll() 方法可以实现一个隐含的条件，如果要和多于一个的条件关联的时候，就不得不额外地添加一个锁，而ReentrantLock则无须这样做，只需要多次调用 `newCondition()`方法即可。

---

**性能对比**

多线程环境下 synchronized 的吞吐量下降得非常严重，而 ReentrantLock 则能基本保持在同一个比较稳定的水平上。与其说 ReentrantLock 性能好，还不如说synchronized 还有非常大的优化余地。

后续的技术发展也证明了这一点，JDK1.6 中加入了很多针对锁的优化措施，synchronized 与 ReentrantLock的性能基本上是完全持平了。因此，如果程序是使用 JDK1.6 或以上部署的话，性能因素就不再是选择 ReentrantLock 的理由了，**虚拟机在未来的性能改进中肯定也会更加偏向于原生的 synchronized**，所以还是提倡在 synchronized 能实现需求的情况下，优先考虑使用 synchronized 来进行同步。

---

**区别**

1. 同步代码块其实是具有自动上锁、自动解锁功能的；`Lock` 锁机制是手动上锁、手动解锁（所以需要在`finally` 中显示的 `unlock()`以防死锁）
2. 用 `synchronized` 修饰的同步代码块还有同步方法是有同步锁对象的；`Lock` 锁机制是没有同步对象的
3. 因为 `synchronized` 修饰的同步代码块还有同步方法是具有锁对象的，因此可以调用 `notify()`、`wait()`、`notifyAll()` 的方法；`Lock` 锁机制没有同步锁，所以不可以调用这些方法，会报错
4. `synchronized` 是在 JVM 层面实现的，而 `Lock` 是通过代码实现的