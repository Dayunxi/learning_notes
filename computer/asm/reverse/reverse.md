# 逆向基础知识

## 基础汇编指令

### INTEL语法

### AT&T语法

- mov
- lea
- test
- cmp
- jmp
- jg/jl/jne

## gdb

- `finsh` 跳出函数
- `display /i $pc` 显示当前汇编指令
- `disassemble` 显示当前函数的汇编指令
- `disable [no]` 禁用某个断点

### 断点

1. 普通断点(`break`)
2. 观察断点(`watch`)
3. 捕捉断点(`catch`)

## 函数传参方法


## 栈/局部变量


## 查看虚函数表的顺序

`readelf --relocs --wide`