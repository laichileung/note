1、CAS不需要同步锁，在多线程安全的情况下改变内存数据，本质上是乐观锁(0,0,1)，只有0=0时，将b写入。
2、ABA问题，操作系统test and set。JVM使用加版本号处理，基础类型不需要版本号。解决思路就是使用版本号。在变量前面追加版本号，每次变量更新的时候把版本号+1，那么A->B->A 就会变成1A->2B->3A。从JDK1.5 开始，JDK的Atomic包里提供了一个类Atomic包里提供了一个AtomicStampedReference来解决ABA问题，

3、自旋compareAndSwap,汇编lock cmpxchg 指令，lock代表其他cup线程无法读取该块内存。MESI

4、CAS可能出现死循环但几率非常小
5、JOL, java object layout, java对象布局
new Object() 占用16字节，前8字节是markWord，8-12字节属于类型指针class pointer，最后4个字节用于对齐padding，任何Object对象必须被8整除。

6、java markword实现表，最后3位代表对象的锁状态
7、锁消除与锁粗化


# valatile

1、保证线程可见性

2、防止指令重排序


3、l1、l2缓存在cup核内部，l3为多核共享缓存大多位于主板，再下一层为主存。
4、超线程：一个ALU对应多个PC,所谓的四核八线程，节省线程切换时间
5、总线将一块内存读进l3 cache， 形成cache line 64字节

6、MESI cache一致性协议， cache line 
已修改Modified (M)
缓存行是脏的（dirty），与主存的值不同。如果别的CPU内核要读主存这块数据，该缓存行必须回写到主存，状态变为共享(S).
独占Exclusive (E)
缓存行只在当前缓存中，但是干净的（clean）--缓存数据同于主存数据。当别的缓存读取它时，状态变为共享；当前写数据时，变为已修改状态。

7、cache line padding

8、DCL单例是否需要加volatile， double check lock 需要加volatile，否则会因为对象半初始化并发生指令重排时，将不安全的对象发布出去。

9、对象创建时候，半初始化后调用构造方法给默认值赋值，最后与栈空间链接。

10、jvm happens-before原则，jvm指令重排原则，通过内存屏障实现，屏障两边的指令无法重排，一共4种。storestorebarrbarrier，