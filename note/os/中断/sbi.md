## 概念

SBI 调用是运行在 S 模式（监管模式，通常指操作系统内核）的代码，请求运行在 M 模式（机器模式，通常指底层固件）的代码提供服务的一种标准化接口和机制。


## 执行

sbi执行在操作系统中是用 `sbi_ecall`函数中
``` c

struct sbiret sbi_ecall(int ext, int fid, unsigned long arg0,
			unsigned long arg1, unsigned long arg2,
			unsigned long arg3, unsigned long arg4,
			unsigned long arg5){
    struct sbiret ret;
    // 绑定寄存器和c语言变量
	register uintptr_t a0 asm ("a0") = (uintptr_t)(arg0);
	register uintptr_t a1 asm ("a1") = (uintptr_t)(arg1);
	register uintptr_t a2 asm ("a2") = (uintptr_t)(arg2);
	register uintptr_t a3 asm ("a3") = (uintptr_t)(arg3);
	register uintptr_t a4 asm ("a4") = (uintptr_t)(arg4);
	register uintptr_t a5 asm ("a5") = (uintptr_t)(arg5);
	register uintptr_t a6 asm ("a6") = (uintptr_t)(fid);
	register uintptr_t a7 asm ("a7") = (uintptr_t)(ext);
	asm volatile ("ecall"
		      : "+r" (a0), "+r" (a1) 
		      : "r" (a2), "r" (a3), "r" (a4), "r" (a5), "r" (a6), "r" (a7)
		      : "memory");
	ret.error = a0;
	ret.value = a1;
            }
```
在`sbi_ecall`之中，会执行`ecall`的汇编代码。

最后会将`错误码`和`返回指`放入寄存器`a0`和`a1`两个寄存器中

tis:`+r`表示即是输出也是输入。

## 返回结构体
`sbiret`是`sbi_ecall`的返回结构体
``` c
struct sbiret {
	long error;
	long value;
    };
```
