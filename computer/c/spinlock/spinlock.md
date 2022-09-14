# spinlock

## glibc-2.31 实现源码

``` c
int pthread_spin_init (pthread_spinlock_t *lock, int pshared)
{
  /* Relaxed MO is fine because this is an initializing store.  */
  atomic_store_relaxed (lock, 0);  // __atomic_store_n (lock, 0, __ATOMIC_RELAXED);  
  return 0;
}

int pthread_spin_lock (pthread_spinlock_t *lock)
{                       
  int val = 0;

#if ! ATOMIC_EXCHANGE_USES_CAS
  if (__glibc_likely (atomic_exchange_acquire (lock, 1) == 0)) 
    return 0;
#else
  if (__glibc_likely (atomic_compare_exchange_weak_acquire (lock, &val, 1)))
    return 0;
#endif

  do {
    do {   
      atomic_spin_nop ();   // PAUSE
      val = atomic_load_relaxed (lock);
    } while (val != 0); 
  } while (!atomic_compare_exchange_weak_acquire (lock, &val, 1));

  return 0;
}


int pthread_spin_unlock (pthread_spinlock_t *lock)
{
  atomic_store_release (lock, 0);  // __atomic_store_n (lock, 0, __ATOMIC_RELEASE);
  return 0;
}

```

----

## 自旋锁无竞争的情况

假设线程一的步骤如下，没有其他线程参与竞争
``` c 
pthread_spinlock_t lock;   // volatile int lock;

pthread_spinlock_init(&lock)  // __atomic_store_n(&lock, 0, __ATOMIC_RELAXED);  //置零

// thread 1
pthread_spin_lock(&lock);
// do something ...
pthread_spin_unlock(&lock);  // __atomic_store_n (lock, 0, __ATOMIC_RELEASE);

```

`pthread_spin_lock`函数会先直接尝试调用`XCHG`指令（认为这比`CMPXCHG`快），并得到lock此前的值，发现是0，即成功加锁，返回。

* 处理器执行`XCHG`时自动发出`LOCK#`信号，使得该`load-modify-store`指令成为原子操作


## 自旋锁有竞争的情况

`pthread_spin_lock`第一次执行`XCHG`指令时，发现lock的值已经是1，因此继续执行下面的循环。

第二层循环由一个`PAUSE`指令，和一个`__atomic_load_n(&lock, 0, __ATOMIC_RELAXED)`构成。

此时默认锁可能处于长时间被占用或者激烈竞争的状态，因此可以用`PAUSE`来减少内存序冲突，而且获得锁的线程可能频繁修改lock所在的cache line，降低load的频率也能减少cache line在cpu间的同步次数，提高CPU总线的利用率。

当load发现lock的值为0时，说明某个线程释放了该锁，本线程通过CAS指令(`LOCK CMPXCHG`)尝试加锁。

为什么第一次尝试是直接用`XCHG`指令加锁，而后面的循环则是通过`LOCK CMPXCHG`加锁呢？我的理解是`XCHG`一定会有store操作，使得其他cache line被标记无效，占用CPU总线。而`CMPXCHG`则会先load一次，先判断值是否与AX寄存器的值相同再决定要不要store，相对而言`XCHG`的失败成本更高点。

### 缓存一致性

由于每个核都有自己的`L1-cache`，假设CPU_0和CPU_1已预先将`&lock`对应的cache line载入自己的`L1-cache`，接下来两个核同时给`&lock`加锁。如果此时CPU_0先成功给自己`L1-cache`中的`&lock`上锁，那么CPU_1如何才能得知`&lock`已经加锁，避免两个核都成功加锁呢？毕竟CPU_0并没有去修改CPU_1中的`L1-cache`。

这就涉及到缓存一致性的问题。一般来说缓存一致性有两种保证方法：
* Directory 协议，会有一个集中式的控制器
* Snoopy 协议，CPU的Cache状态更新后会发送广播告知其他Cache 

具体细节参考：
[MESI协议](/computer/cpu/intel/MESI/mesi-intel.md)

---

## libpthread.so 的实现 (glibc-2.2.5)

```ARM Assembly
000000000000c4a0 <pthread_spin_lock>:
    c4a0:   f0 ff 0f                lock decl (%rdi)
    c4a3:   75 0b                   jne    c4b0 <pthread_spin_lock+0x10>
    c4a5:   31 c0                   xor    %eax,%eax
    c4a7:   c3                      retq   
    c4a8:   0f 1f 84 00 00 00 00    nopl   0x0(%rax,%rax,1)
    c4af:   00    
    c4b0:   f3 90                   pause  
    c4b2:   83 3f 00                cmpl   $0x0,(%rdi)
    c4b5:   7f e9                   jg     c4a0 <pthread_spin_lock>
    c4b7:   eb f7                   jmp    c4b0 <pthread_spin_lock+0x10>
    c4b9:   0f 1f 80 00 00 00 00    nopl   0x0(%rax)

000000000000c4e0 <pthread_spin_init>:
    c4e0:   c7 07 01 00 00 00       movl   $0x1,(%rdi)
    c4e6:   31 c0                   xor    %eax,%eax
    c4e8:   c3                      retq
    c4e9:   0f 1f 80 00 00 00 00    nopl   0x0(%rax)
```

*pthread_spin_unlock的实现似乎不在libpthread.so，而是在sysdeps/x86_64/pthread_spin_unlock.S*

linux中实际使用的办法更加简洁高效。初始值1表示unlock状态，其他值表示lock状态，pthread_spin_lock时通过加锁自减指令(`lock decl`)保证原子性。

在unlock状态时，`decl`让`(%rdi)`从1自减为0，刚好触发设置zero flag，于是在第二行的`jne`不会跳转，直接返回0。

在lock状态时，`decl`让`(%rdi)`从0自减为-1，不会设置zero flag，于是跳转至`c4b0`，休眠几个周期后，重新把`(%rdi)`和0比较，如果大于0（说明此时是unlock状态），则跳到开头重新尝试`lock decl`，否则跳回`c4b0`。

当有n个线程同时pthread_spin_lock时，`(%rdi)`不会小于-n。

---

tags: c, linux, glibc
