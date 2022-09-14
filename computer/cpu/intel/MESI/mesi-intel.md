# 缓存一致性

## MESI协议

*不确定intel的cpu使用的MESI协议是否完全和wiki上的一致，但目前只需要知道确实存在一种控制协议能保障缓存一致性即可，因此举例时使用wiki上的解释。*

MESI协议将各个核中的各个Cache Line都分为了4个状态(Modified, Exclusive, Shared, Invalid)。

* Modified表示当前的Cache Line已被修改，而实际内存还未更新，且其他核的此Cache Line都处于Invalid状态
* Exclusive表示当前的Cache Line与实际内存相同，且其他核的此Cache Line都处于Invalid状态
* Shared表示当前的Cache Line与实际内存相同，可能在其他核存在此Cache Line
* Invalid表示当前Cache Line已经失效，不论读写都需要先重新获取

举两个例子

### 例子1
假如有CPU0与CPU1，地址addr所在的Cache Line在两个CPU中都处于Invalid状态，此时CPU0 `store(addr, $0x0)`，CPU1 `store(addr, $0x1)`，两个操作同时发生。

根据[wiki](https://en.wikipedia.org/wiki/MESI_protocol)的状态响应图，两个CPU会先发送`BusRdX`信号，

1. 不妨假设CPU0先发送`BusRdX`信号，占用总线，同时更新Cache Line状态为`Modified`。
2. 此时CPU1的信号发不出去，反而监听到了其他核的`BusRdx`信号，但由于此时的状态为Invalid，忽略该信号。
3. 内存收到addr地址的读请求(BusRdX包含向内存请求数据的信号)，返回addr的实际数据
4. CPU0通过数据总线收到数据，更新自己的Cache Line
5. CPU1此时可以通过地址总线重新发送`BusRdx`信号，并更新状态为`Modified`。
6. 同时CPU0向addr写入0x0。
7. CPU0收到`BusRdx`信号，更新状态为`Invalid`，并将Cache Line写入数据总线发给内存和CPU1
8. CPU1收到数据，更新cache并向addr写入0x1


### 例子2
```c
//thread 0
a = 1;
b = 1;

//thread 1
while (b == 0);
assert(a == 1);
```
假设a, b在分别在不同的Cache Line A, B，且在两个CPU均处于Invalid状态。

1. CPU0执行`a = 1`先从内存获取实际的A，然后更新A状态为`Modified`并向&a写入1
2. 同时CPU1执行`b == 0`也会获取实际的B，并更新B的状态为`Exclusive`
3. CPU0开始执行`b = 1`，先发送`BusRdX B`信号，然后更新B状态为`Modified`
4. CPU1侦测到信号，更新B状态为`Invalid`，并将B发送给CPU0，接下来继续执行`b == 0`
5. CPU0收到B，然后向&b写入1
6. CPU1发送`BusRd B`信号
7. CPU0收到信号，将B的状态改为`Shared`，并将B发给CPU1
8. CPU1收到B，也将B状态改为`Shared`，然后读取&b的值，判断后结束while循环
9. CPU1开始读取A，先发送`BusRdX A`信号
10. CPU0将A的状态改为`Shared`，发送A给CPU1
11. CPU1收到A，状态改为`Shared`，并读取&a的值，assert通过

## 注意

假如有两个CPU同时执行`lock add $0x1, addr`。(`add`是一个`load-modify-store`操作)。

带有lock前缀的指令会发送LOCK#信号(一般情况仅锁住cache line)，占用总线直至该指令结束，从而保证两个add按顺序执行并保证原子性。

> Effects of a LOCK Operation on Internal Processor Caches (vol3-8.1.4)
> 
> For the Intel486 and Pentium processors, the LOCK# signal is always asserted on the bus during a LOCK operation,
even if the area of memory being locked is cached in the processor.
>
> For the P6 and more recent processor families, if the area of memory being locked during a LOCK operation is cached in the processor that is performing the LOCK operation as write-back memory and is completely contained in a cache line, the processor may not assert the LOCK# signal on the bus. Instead, it will modify the memory location internally and allow it’s cache coherency mechanism to ensure that the operation is carried out atomically. This operation is called “cache locking.” The cache coherency mechanism automatically prevents two or more processors that have cached the same area of memory from simultaneously modifying data in that area.


> Controlling the Processor (vol3-2.8.5)
> 
> The HLT (halt processor) instruction stops the processor until an enabled interrupt (such as NMI or SMI, which are normally enabled), a debug exception, the BINIT# signal, the INIT# signal, or the RESET# signal is received. The processor generates a special bus cycle to indicate that the halt mode has been entered. 
> 
> Hardware may respond to this signal in a number of ways. An indicator light on the front panel may be turned on. An NMI interrupt for recording diagnostic information may be generated. Reset initialization may be invoked (note that the BINIT# pin was introduced with the Pentium Pro processor). If any non-wake events are pending during shutdown, they will be handled after the wake event from shutdown is processed (for example, A20M# interrupts). 
> 
> The LOCK prefix invokes a locked (atomic) read-modify-write operation when modifying a memory operand. This mechanism is used to allow reliable communications between processors in multiprocessor systems, as described
below:
> * In the Pentium processor and earlier IA-32 processors, the LOCK prefix causes the processor to assert the LOCK# signal during the instruction. This always causes an explicit bus lock to occur.
> * In the Pentium 4, Intel Xeon, and P6 family processors, the locking operation is handled with either a cache lock or bus lock. If a memory access is cacheable and affects only a single cache line, a cache lock is invoked and the system bus and the actual memory location in system memory are not locked during the operation. Here, other Pentium 4, Intel Xeon, or P6 family processors on the bus write-back any modified data and invalidate their caches as necessary to maintain system memory coherency. If the memory access is not cacheable and/or it crosses a cache line boundary, the processor’s LOCK# signal is asserted and the processor does not respond to requests for bus control during the locked operation.

---

tags: cpu, intel, MESI, cache