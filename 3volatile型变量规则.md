# volatile 型变量的特殊规则

关键字 `volatile` 可以说是 Java 虚拟机提供的最轻量级的同步机制。

当一个变量定义为 `volatile` 之后，它将具备两种特性，

1. 保证此变量对所有线程的**可见性（也就是一致性）**，这里的「可见性」是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。而普通变量的值在线程之间传递均需要通过主内存来完成。
    - 原理
        1. Lock 前缀的指令会引起处理器缓存（工作内存）写回内存（共享内存）；
        2. 一个处理器的缓存回写到内存会导致其他处理器的缓存失效；
        3. 当处理器发现本地缓存失效后，就会从内存中重读该变量数据，即可以获取当前最新值。
    - 但是 Java 里面的运算并非原子操作，导致 **volatile 变量的运算在并发下一样是不安全的，所以在不符合以下两条规则的运算场景中，**我们仍然要通过**加锁**（synchronized 或 java.util.concurrent 的原子类）来保证**原子性。**
        1. 运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。

                volatile boolean shutdownRequested；
                
                public void shutdown(){
                	shutdownRequested = true；
                }
                
                public void doWork(){
                	while（!shutdownRequested）{
                		//dostuff
                	}
                }

        2. 变量不需要与其他的状态变量共同参与不变约束。
2. **禁止指令重排序优化**
    - 什么是指令重排序？

        从硬件架构上讲，指令重排序是指 CPU 采用了允许将多条指令不按程序规定的顺序分开发送给各相应电路单元处理。但并不是说指令任意重排，CPU 需要能正确处理指令依赖情况以保障程序能得出正确的执行结果。譬如指令 1 把地址 A 中的值加 10，指令 2 把地址 A 中的值乘以 2，指令 3 把地址 B 中的值减去 3，这时指令 1 和指令 2 是有依赖的，它们之间的顺序不能重排

    - 有 volatile 修饰的变量，赋值后多执行了一个**lock前缀的指令操作**，这个操作相当于一个**内存屏障**，指重排序时不能把后面的指令重排序到内存屏障之前的位置。只有一个 CPU 访问内存时，并不需要内存屏障；但如果有两个或更多CPU 访问同一块内存，且其中有一个在观测另一个，就需要内存屏障来保证一致性了。

            // 在JDK1.5 后使用 DCL（双锁校验）
            public class Singleton {
            	private volatile static Singleton instance;
            
            	private Singleton(){}
            
            	public static getInstance(){
            		if(instance == null){
            			synchronized(Singleton.class){
            				if(instance == null) {
            						instance = new Singleton();
            				}
            			}
            		}
            		return instance;
            	}
            }