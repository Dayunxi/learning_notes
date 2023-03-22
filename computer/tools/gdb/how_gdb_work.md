# 工作原理

通过ptrace调用，将目标函数的首地址对应的第一个字节替换为`int 3`中断指令。进程运行至目标位置并且发生中断后，被gdb捕获，随后gdb通过`ptrace`读取tracee的寄存器等信息，通过查找自己维护的链表，确认当前是哪个断点。
如果想要tracee继续执行，则先恢复断点的首字节，然后通过`ptrace`回退一个指令，随后单步执行，接着再重新向断点注入`int 3`指令，后面便可正常运行。

```c
#include <sys/ptrace.h>
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);
```

PTRACE_GETREGS - 获取寄存器信息
PTRACE_SETREGS - 修改寄存器信息
PTRACE_PEEKTEXT - 从目标地址读取一字节 
PTRACE_POKETEXT - 向目标地址写入一字节
PTRACE_SINGLESTEP - 单步执行

# 参考

1. [一窥GDB原理][gdb_ref]

[gdb_ref]: https://mp.weixin.qq.com/s/teERWh9IRMuO6tieOZNNng