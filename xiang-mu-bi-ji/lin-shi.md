# 指令序列的重排序
## 为什么要对指令进行重排序？提高性能
为了提高执行效率，提高性能。



```
举个不恰当的例子：
1. a=1
2. b=2
3. a = a + 2
```



如果按照1，2，3的顺序执行，那就是把 a 读入寄存器-赋值-写回，把 b 读入寄存器-赋值-写回，把 a 读入寄存器-加法-写回。`《a 读入寄存器发生了2次》`
可以讲执行顺序调整为1，3，2，这样的话，把 a 读入寄存器-赋值-加法-写回，把 b 读入寄存器-赋值-写回。`《a 读入寄存器发生了1次》`

## 在什么地方发生了指令重排？编译器和处理器
1. 编译器重排，重排的是语句（Java 中是.class字节码）
2. 处理器（重排的是机器指令）
    1. 指令级并行技术：将多条指令重叠执行。如果程序语句间无数据依赖，处理器可以改变对应的机器指令的执行顺序，进一步可能通过指令级并行技术来重叠执行多条指令。
    2. 内存读写重排序。处理器使用缓存和读写缓冲区，这使得加载和存储操作看上去可能是乱序执行。

> 对处理器两种重排的理解。
处理器要执行的指令放在高速缓存中，它可以分析这些指令，通过重排来重叠执行（2.1）。

### CPU 写缓冲
#### CPU 为什么要进行写缓冲？减少等待写产生的停顿，提高执行效率
有了写缓冲区，可避免 CPU 停顿下来等待写内存。
CPU 将数据写入写缓冲区。
批处理方式刷新写缓冲区。
合并写缓冲区对同一地址的多次写。
减少对内存总线的占用。
> 写缓冲区只对自己的 CPU 可见
产生的问题：CPU 对内存的读写操作顺序，与实际上发生的内存读写顺序不一致。
比如：CPU 先写缓冲后读内存，但内存实际上先执行了读然后才发生了写，因为写缓冲的写是滞后的。

### 再议内存读写重排
本质上，就是因为 CPU 采用了写缓冲，导致实际在内存的读写顺序发生了变化。
在并发环境中导致的问题：当前线程的更新，对其他线程不可见（因为写缓冲滞后，且写缓冲只对当前线程可见）。

## 重排序可能导致并发时内存可见性问题，因此需要控制
1. 控制编译器重排。JMM 的编译器重排规则，会禁止部分编译器重排
2. 控制处理器重排。Java 编译器生成指令序列时，插入内存屏障指令，来禁止部分处理器重排。

### 内存屏障
JMM 4种内存屏障。LoadLoad，StoreStore，LoadStore，StoreLoad
StoreLoad
Store1 ；StoreStore；Load2
此屏障要求：确保 Store1数据对其他处理器变得可见（可见：指刷新到内存），先于 Load2及所有后续装载指令的装载。


```
StoreLoad 屏障会使该屏障之前的所有内存访问指令（存储和装载）完成之后，才执行该屏障之后的内存访问指令
该屏障开销也大，因为必须要刷新写缓存全部数据到内存
```



## 从源代码到指令的整个流程
源代码 --> 编译器语句重排 --> 指令级并行重排 --> 内存读写重排 --> 最终执行的指令序列