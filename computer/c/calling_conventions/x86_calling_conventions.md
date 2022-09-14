# 1. 起因

网上随便搜一篇关于printf，变参函数以及va\_start/va\_arg的原理的帖子,基本都是以参数都保存在栈里作为前提。那么问题来了:

```c
int a = 0x777, b = 0x666;
double c = 0x888888;
printf("%d %lf %d\n", a, b, c);
```
使用64位编译器编译上述代码，查看调用printf函数的那部分汇编时，发现根本没有压栈操作，只有给寄存器赋值的操作。要知道printf早就提前编译好了放在libc.so里，机器码都已经固定了，那它怎么判断我们放了几个参数，以及分别放到了哪里呢？printf是如何正确拿到我们传入的参数的？如果我往里面填了十几二十个参数它又该怎么办？


## 1.1 64位程序的汇编

```ARM Assembly
0000000000400797 <simple_printf>:

void simple_printf(void)
{
  400797:   55                      push   %rbp
  400798:   48 89 e5                mov    %rsp,%rbp
  40079b:   48 83 ec 20             sub    $0x20,%rsp
    int a = 0x777, c = 0x666;
  40079f:   c7 45 fc 77 07 00 00    movl   $0x777,-0x4(%rbp)
  4007a6:   c7 45 f8 66 06 00 00    movl   $0x666,-0x8(%rbp)
    double b = 0x888888;
  4007ad:   48 b8 00 00 00 00 11    movabs $0x4161111100000000,%rax
  4007b4:   11 61 41
  4007b7:   48 89 45 f0             mov    %rax,-0x10(%rbp)
    printf("%d %lf %d\n", a, b, c);
  4007bb:   8b 55 f8                mov    -0x8(%rbp),%edx
  4007be:   48 8b 45 f0             mov    -0x10(%rbp),%rax
  4007c2:   8b 4d fc                mov    -0x4(%rbp),%ecx
  4007c5:   48 89 45 e8             mov    %rax,-0x18(%rbp)
  4007c9:   f2 0f 10 45 e8          movsd  -0x18(%rbp),%xmm0
  4007ce:   89 ce                   mov    %ecx,%esi
  4007d0:   bf 07 09 40 00          mov    $0x400907,%edi
  4007d5:   b8 01 00 00 00          mov    $0x1,%eax
  4007da:   e8 31 fc ff ff          callq  400410 <printf@plt>
}
  4007df:   c9                      leaveq
  4007e0:   c3                      retq
```
可以看到在64位程序中，调用printf前将参数分别放进了几个寄存器中（fmt指针在`EDI`, a在`ESI`, b在`XMM0`, c在`EDX`），还将`EAX`置为了1，没有任何压栈操作。这时候libc.so中的printf在**提前编译好的情况下**是怎么知道我们写了几个参数？怎么知道我们这边编译后把参数放到了哪些寄存器里？如何正确拿到我们调用时的入参？

## 1.2 32位程序的汇编

```ARM Assembly
08048443 <simple_printf_m32>:

void simple_printf_m32(void)
{
 8048443:       55                      push   %ebp
 8048444:       89 e5                   mov    %esp,%ebp
 8048446:       83 ec 38                sub    $0x38,%esp
        int a = 0x777, c = 0x666;
 8048449:       c7 45 f4 77 07 00 00    movl   $0x777,-0xc(%ebp)
 8048450:       c7 45 f0 66 06 00 00    movl   $0x666,-0x10(%ebp)
        double b = 0x888888;
 8048457:       dd 05 90 85 04 08       fldl   0x8048590
 804845d:       dd 5d e8                fstpl  -0x18(%ebp)
        printf("%d %lf %d\n", a, b, c);
 8048460:       8b 45 f0                mov    -0x10(%ebp),%eax
 8048463:       89 44 24 10             mov    %eax,0x10(%esp)
 8048467:       dd 45 e8                fldl   -0x18(%ebp)
 804846a:       dd 5c 24 08             fstpl  0x8(%esp)
 804846e:       8b 45 f4                mov    -0xc(%ebp),%eax
 8048471:       89 44 24 04             mov    %eax,0x4(%esp)
 8048475:       c7 04 24 80 85 04 08    movl   $0x8048580,(%esp)
 804847c:       e8 4f fe ff ff          call   80482d0 <printf@plt>
}
 8048481:       c9                      leave
 8048482:       c3                      ret
```
再看看这段在GCC 4.8.5编译出来的32位程序。可以看到变量`c`, `b`, `a`以及字符串常量指针）依次入栈。
`leave`命令会先将`EBP`的值赋给`ESP`, 再pop出旧的栈底指针给`EBP`(即第一行push进栈的ebp)，释放函数占用的栈空间 。
printf的函数原型: `int printf(const char *format, ...);`
可以看到libc.so中的printf可以简单地通过fmt指针的地址得到最后一个入栈参数的地址，然后通过fmt字符串的内容得到各变量的类型及大小，并取得各变量的值。这时候一切都很好理解。

---

想要知道64位程序的printf到底是怎么入参的，有必要先了解一下函数的各种入参方式。


# 2. [函数调用传参方式](https://en.wikipedia.org/wiki/X86_calling_conventions)

几个主要方式

## 2.1 cdecl (C declaration)

```C
int callee(int, int, int);

int caller(void)
{
	return callee(1, 2, 3) + 5;
}
```

```ARM Assembly
caller:
    push    %ebp        ; 保存旧的栈底指针
    mov     %esp, %ebp  ; 将旧的栈顶指针置为新的栈底指针
    ; 从右至左将参数入栈，可以用push，也可以直接移动栈顶指针
    ; 地址越往下越靠近栈顶
    ; sub $12, %esp
    ; mov $3, -4(%ebp)
    ; mov $2, -8(%ebp)
    ; mov $1, -12(%ebp)
    push    $3
    push    $2
    push    $1
    call    callee     ; 调用函数callee
    add     $12, %esp  ; 恢复传参占用的栈
    add     $5, %eax   ; 将结果+5 (callee的返回值在eax)
    ; 恢复栈,有些编译器(如gcc-4.8.5)会通过leave指令代替下面两条指令
    mov     %ebp, %esp ; 恢复旧的栈顶指针 （绝大多数调用约定中的ebp都由callee自己维护，对caller透明）
    pop     %ebp       ; 恢复旧的栈底指针
    ret                ; 返回(eip指针出栈)
```

 - 整型返回值存至EAX寄存器，浮点型存至ST0
 - EAX, ECX, EDX由caller维护（callee可任意修改）
 - 参数通过栈传递，从右至左入参
 - 由caller清理栈
 - IA-32架构下的GCC编译时，如果返回的是结构体/类，则由caller将预先分配好的地址指针入栈，callee会将返回值写入该地址


## 2.2 stdcall (Microsoft)

 - 返回值存至EAX寄存器
 - EAX, ECX, EDX由caller维护（callee可任意修改）
 - 参数通过栈传递，从右至左入参
 - 由callee清理栈
 - GCC也支持

## 2.3 thiscall

 - GCC compiler
   - this指针最后一个入栈(等效于默认的第一个参数)
   - 其余同cdecl(由caller清理栈)

 - Microsoft Visual C++ compiler
   - this指针通过`ECX`传递
   - 其余同stdcall(但如果是变参函数则由caller来清理栈)

------

## 2.4 syscall (linux)

 - i386
	 - 汇编指令为`int $0x80`
	 - 系统调用类型通过`EAX`传递，返回值存至`EAX`
	 - 最多六个参数依次通过`EBX`, `ECX`, `EDX`, `ESI`, `EDI`, `EBP`传递

 - x86_64
	 - 汇编指令为`syscall`
	 - 系统调用类型通过`RAX`传递，返回值存至`RAX`
	 - 最多六个参数依次通过`RDI`, `RSI`, `RDX`, `R10`, `R8`, `R9`传递

libc.so.6中`mmap`的汇编(glibc-2.17)
```ARM Assembly
00000000000f8330 <mmap>:
   f8330:   41 57                   push   %r15   
   f8332:   48 85 ff                test   %rdi,%rdi
   f8335:   41 56                   push   %r14   
   f8337:   41 55                   push   %r13   
   f8339:   4d 89 cd                mov    %r9,%r13
   f833c:   41 54                   push   %r12   
   f833e:   41 89 cc                mov    %ecx,%r12d
   f8341:   55                      push   %rbp   
   f8342:   48 89 f5                mov    %rsi,%rbp
   f8345:   53                      push   %rbx   
   f8346:   48 89 fb                mov    %rdi,%rbx
   f8349:   74 35                   je     f8380 <mmap+0x50>
   f834b:   4d 63 f0                movslq %r8d,%r14
   f834e:   4c 63 fa                movslq %edx,%r15
   f8351:   4d 89 e9                mov    %r13,%r9
   f8354:   4d 89 f0                mov    %r14,%r8
   f8357:   4d 63 d4                movslq %r12d,%r10
   f835a:   4c 89 fa                mov    %r15,%rdx
   f835d:   48 89 ee                mov    %rbp,%rsi
   f8360:   48 89 df                mov    %rbx,%rdi
   f8363:   b8 09 00 00 00          mov    $0x9,%eax
   f8368:   0f 05                   syscall 
   ......
```

## 2.5 x86-64 calling conventions

 - Microsoft x64
   - 前四个整型参数分别放至`RCX`, `RDX`, `R8`, `R9`
   - 前四个浮点参数放至`XMM0`, `XMM1`, `XMM2`, `XMM3`
   - 剩下的参数从右至左入栈
   - 整型返回值放至`RAX`，浮点型返回值放至`XMM0`
   - 结构体参数如果超过整型大小，会将指针作为参数传入
   - 如果返回类型过大，则需要caller将返回左值的地址作为隐含的第一个参数传入
   - `RAX`, `RCX`, `RDX`, `R8`, `R9`, `R10`, `R11`由caller维护

 - System V AMD64 ABI
   - 前六个整型分别放至`RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9`
   - 前八个浮点型分别放至`XMM0`至`XMM7`
   - 剩下的参数从右至左入栈
   - 整型返回值放至`RAX`及`RDX`，浮点型返回值放至`XMM0`及`XMM1`
   - 参数为较大结构体(超过四个八字节)时，直接入栈
   - 如果返回类型过大(>32)，则需要caller将返回左值的地址作为隐含的第一个参数传至`RDI`
   - **除了**`RBX`, `RBP`, `R12-R15`，其他都由caller维护

------

## 2.6 Variable Argument Lists (变参函数)

仅介绍x86-64下的linux/gcc的做法，以下面的函数作为例子
```c
void sample_args_func(const char *fmt, ...)
{
    va_list ap;
    va_start(ap, fmt);
    for (; *fmt; fmt++) {
        if (*fmt == 'i') {
            int val = va_arg(ap, int);
            // ...
        } else if (*fmt == 'd') {
            double val = va_arg(ap, double);
            // ...
        }
    }
    va_end(ap);		// 对GCC而言，该宏不做任何事，仅为了可移植性而写
}
```

### 2.6.1 System V AMD64 ABI

1. 调用变参函数时，`RAX`必须被置为浮点型参数的个数（最大值为8）

2. 类型大小超过8字节，且没有作为显式的形参被命名，直接入栈

3. 函数栈内会预留部分空间分别保存各传参寄存器的值(Register Save Area)。若`RAX`为0则不会保存`XMM`寄存器组

4. `va_list`结构体定义：

```C
// __builtin_va_list
typedef struct {
	unsigned int gp_offset;
	unsigned int fp_offset;
	void *overflow_arg_area;
	void *reg_save_area;
} va_list[1];
```

5. `va_start`宏：
	- 将`reg_save_area`指向Register Save Area对应的内存地址
	- 将`overflow_arg_area`指向通过栈传递的参数的起始地址（每次更新均指向下一个参数）
	- `gp_offset`指`reg_save_area`中整型参数的地址偏移量，初始值为8(跳过第一个参数)，最大值为48(6*8)
	- `fp_offset`指`reg_save_area`中浮点型参数的地址偏移量，gcc-4.8.5中的最大值为`6*8 + 8*16`

6. `va_arg`宏：
	以`va_arg(ap, int)`为例，其逻辑等价于以下代码：
	```
	int va_arg(va_list ap) {
		int ret;
		// 如果已经取完了寄存器中的参数，就从栈里取
		if (ap->gp_offset >= 48) {
			ret = *(int *)ap->overflow_arg_area;
			ap->overflow_arg_area += 8;
		// 否则从寄存器里取
		} else {
			ret = *(int *)(ap->reg_save_area + ap->gp_offset);  // 浮点型则加数为ap->fp_offset
			ap->gp_offset += 8;
		}
		return ret;	
	}
	```
	`va_arg(ap, double)`等其他调用的逻辑也类似。`va_arg`一般由各个编译器自己实现(__builtin_va_start)，并且根据参数的具体类型展开为相应的形式(比如类型较大的参数就直接从栈里取)

---

到现在我们就知道printf这类函数到底是怎么获取参数的了。

举个例子:
```c
void sample_args_func(const char *fmt, ...)
{
    va_list ap;
    va_start(ap, fmt);
    for (; *fmt; fmt++) {
        if (*fmt == 'i') {
            int val = va_arg(ap, int);
            // ...
            //printf("i=%d\n", val);
        } else if (*fmt == 'd') {
            double val = va_arg(ap, double);
            // ...
            //printf("d=%lf\n", val);
        }
    }
    va_end(ap);
}
```
```ARM Assembly
000000000040079b <sample_args_func>:

void sample_args_func(const char *fmt, ...)
{
  40079b:	55                   	push   %rbp
  40079c:	48 89 e5             	mov    %rsp,%rbp
  40079f:	48 83 ec 70          	sub    $0x70,%rsp
  4007a3:	48 89 b5 58 ff ff ff 	mov    %rsi,-0xa8(%rbp)
  4007aa:	48 89 95 60 ff ff ff 	mov    %rdx,-0xa0(%rbp)
  4007b1:	48 89 8d 68 ff ff ff 	mov    %rcx,-0x98(%rbp)
  4007b8:	4c 89 85 70 ff ff ff 	mov    %r8,-0x90(%rbp)
  4007bf:	4c 89 8d 78 ff ff ff 	mov    %r9,-0x88(%rbp)
  4007c6:	84 c0                	test   %al,%al
  4007c8:	74 20                	je     4007ea <sample_args_func+0x4f>
  4007ca:	0f 29 45 80          	movaps %xmm0,-0x80(%rbp)
  4007ce:	0f 29 4d 90          	movaps %xmm1,-0x70(%rbp)
  4007d2:	0f 29 55 a0          	movaps %xmm2,-0x60(%rbp)
  4007d6:	0f 29 5d b0          	movaps %xmm3,-0x50(%rbp)
  4007da:	0f 29 65 c0          	movaps %xmm4,-0x40(%rbp)
  4007de:	0f 29 6d d0          	movaps %xmm5,-0x30(%rbp)
  4007e2:	0f 29 75 e0          	movaps %xmm6,-0x20(%rbp)
  4007e6:	0f 29 7d f0          	movaps %xmm7,-0x10(%rbp)
  4007ea:	48 89 bd 18 ff ff ff 	mov    %rdi,-0xe8(%rbp)
	va_list ap;
	va_start(ap, fmt);
  4007f1:	c7 85 28 ff ff ff 08 	movl   $0x8,-0xd8(%rbp)
  4007f8:	00 00 00 
  4007fb:	c7 85 2c ff ff ff 30 	movl   $0x30,-0xd4(%rbp)
  400802:	00 00 00 
  400805:	48 8d 45 10          	lea    0x10(%rbp),%rax
  400809:	48 89 85 30 ff ff ff 	mov    %rax,-0xd0(%rbp)
  400810:	48 8d 85 50 ff ff ff 	lea    -0xb0(%rbp),%rax
  400817:	48 89 85 38 ff ff ff 	mov    %rax,-0xc8(%rbp)
	for (; *fmt; fmt++) {
  40081e:	e9 c0 00 00 00       	jmpq   4008e3 <sample_args_func+0x148>
		if (*fmt == 'i') {
  400823:	48 8b 85 18 ff ff ff 	mov    -0xe8(%rbp),%rax
  40082a:	0f b6 00             	movzbl (%rax),%eax
  40082d:	3c 69                	cmp    $0x69,%al
  40082f:	75 4d                	jne    40087e <sample_args_func+0xe3>
			int val = va_arg(ap, int);
  400831:	8b 85 28 ff ff ff    	mov    -0xd8(%rbp),%eax
  400837:	83 f8 30             	cmp    $0x30,%eax
  40083a:	73 23                	jae    40085f <sample_args_func+0xc4>
  40083c:	48 8b 95 38 ff ff ff 	mov    -0xc8(%rbp),%rdx
  400843:	8b 85 28 ff ff ff    	mov    -0xd8(%rbp),%eax
  400849:	89 c0                	mov    %eax,%eax
  40084b:	48 01 d0             	add    %rdx,%rax
  40084e:	8b 95 28 ff ff ff    	mov    -0xd8(%rbp),%edx
  400854:	83 c2 08             	add    $0x8,%edx
  400857:	89 95 28 ff ff ff    	mov    %edx,-0xd8(%rbp)
  40085d:	eb 15                	jmp    400874 <sample_args_func+0xd9>
  40085f:	48 8b 95 30 ff ff ff 	mov    -0xd0(%rbp),%rdx
  400866:	48 89 d0             	mov    %rdx,%rax
  400869:	48 83 c2 08          	add    $0x8,%rdx
  40086d:	48 89 95 30 ff ff ff 	mov    %rdx,-0xd0(%rbp)
  400874:	8b 00                	mov    (%rax),%eax
  400876:	89 85 4c ff ff ff    	mov    %eax,-0xb4(%rbp)
  40087c:	eb 5d                	jmp    4008db <sample_args_func+0x140>
			// ...
			//printf("i=%d\n", val);
		} else if (*fmt == 'd') {
  40087e:	48 8b 85 18 ff ff ff 	mov    -0xe8(%rbp),%rax
  400885:	0f b6 00             	movzbl (%rax),%eax
  400888:	3c 64                	cmp    $0x64,%al
  40088a:	75 4f                	jne    4008db <sample_args_func+0x140>
			double val = va_arg(ap, double);
  40088c:	8b 85 2c ff ff ff    	mov    -0xd4(%rbp),%eax
  400892:	3d b0 00 00 00       	cmp    $0xb0,%eax
  400897:	73 23                	jae    4008bc <sample_args_func+0x121>
  400899:	48 8b 95 38 ff ff ff 	mov    -0xc8(%rbp),%rdx
  4008a0:	8b 85 2c ff ff ff    	mov    -0xd4(%rbp),%eax
  4008a6:	89 c0                	mov    %eax,%eax
  4008a8:	48 01 d0             	add    %rdx,%rax
  4008ab:	8b 95 2c ff ff ff    	mov    -0xd4(%rbp),%edx
  4008b1:	83 c2 10             	add    $0x10,%edx
  4008b4:	89 95 2c ff ff ff    	mov    %edx,-0xd4(%rbp)
  4008ba:	eb 15                	jmp    4008d1 <sample_args_func+0x136>
  4008bc:	48 8b 95 30 ff ff ff 	mov    -0xd0(%rbp),%rdx
  4008c3:	48 89 d0             	mov    %rdx,%rax
  4008c6:	48 83 c2 08          	add    $0x8,%rdx
  4008ca:	48 89 95 30 ff ff ff 	mov    %rdx,-0xd0(%rbp)
  4008d1:	48 8b 00             	mov    (%rax),%rax
  4008d4:	48 89 85 40 ff ff ff 	mov    %rax,-0xc0(%rbp)
	for (; *fmt; fmt++) {
  4008db:	48 83 85 18 ff ff ff 	addq   $0x1,-0xe8(%rbp)
  4008e2:	01 
  4008e3:	48 8b 85 18 ff ff ff 	mov    -0xe8(%rbp),%rax
  4008ea:	0f b6 00             	movzbl (%rax),%eax
  4008ed:	84 c0                	test   %al,%al
  4008ef:	0f 85 2e ff ff ff    	jne    400823 <sample_args_func+0x88>
			// ...
			//printf("d=%lf\n", val);
		}
	}
	va_end(ap);
}
  4008f5:	c9                   	leaveq 
  4008f6:	c3                   	retq  
```


# 3. 其他补充

## linux kernel

linux的系统调用也最多支持6个参数(*include/linux/syscalls.h*)。

---

tags: gcc, c, linux, calling conventions, abi

 