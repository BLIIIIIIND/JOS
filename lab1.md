# Lab 1

### Part 1: PC Bootstrap

##### Exercise 1

主要是要了解 GCC 所用的内联汇编跟硬件厂商公司的标准不一样，值得注意的就是内联汇编左边是源寄存器（内存），右边是目标寄存器（内存）。此外，还应了解内联汇编中调用外部变量的写法。详见 [Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html)

##### Exercise 2

主要是初始化一些硬件（操作IO端口），以及填写基本的中断向量表（ IDT ）。

### Part 2: The Boot Loader

##### Exercise 3

跟着指引走即可，可以对照 obj/boot/boot.asm 文件。

##### Exercise 4

此处推荐阅读 *The C Programming Language* 这本书的 5.1 到 5.5 这部分的内容。然后分析 [pointers.c](https://pdos.csail.mit.edu/6.828/2018/labs/lab1/pointers.c) 中的内容。其中比较有意思的是第五行输出。

 ```5: a[0] = 200, a[1] = 128144, a[2] = 256, a[3] = 302```

在第四行输出之后，变量的状态为：

 ``` a[0] == 200, a[1] == 400, a[2] == 301, a[3] == 300, c == a[1]```

然后， ```c = (int *) ((char *) c + 1);``` 让 c 首先从 int* 类型转换成 char* 类型，再 +1 ，此时 c 所指向的内存地址是 a[1] 后一个字节的位置，因此 ```*c = 500;``` 所写入的 int 值 500 的低三个字节覆盖了 a[1] 的高三个字节，高一个字节覆盖了 a[2] 的低一个字节。

x86 CPU 采用小端字节序，也就是说高内存地址存储高位字节。

使用 16 进制表示数字，每两位都相当于一个字节，以上操作的分解如下：

1. a[1] 的 16 进制形式： 0x00000190
2. a[2] 的 16 进制形式： 0x0000012D
3. *c 写入数字的 16 进制形式： 0x000001F4
4. 以上 16 进制数字从左到右即为从高到低，将 0x000001F4 的低三个字节 0x0001F4 替换到 a[1] 高三个字节，得到 a[1] = 0x0001F490 = 128144 ，将 0x000001F4 的高一个字节 0x00 替换到 a[2] 低一个字节，得到 a[2] = 0x00000100 = 256 。

##### Exercise 5

第一条出错的指令是 boot/boot.S 中的

 ```ljmp   $PROT_MODE_CSEG, $protcseg``` 

因为此处使用了常量 ```PROT_MODE_CSEG = 0x08``` 来选择全局描述符表（ GDT ）中的第一个描述符。

 ```lgdt   gdtdesc``` 语句按照计算出来的 gdtdesc 链接地址找到对应的内存区域加载 GDT ，而修改 boot/Makefrag 文件的链接地址不能改变编译后的二进制文件加载到以 0x7c00 开始，却使得 gdtdesc 对应的内存地址并不是界限值+ GDT 地址这6个字节。

综上，修改链接地址后会导致 CPU 加载错误的 GDT ，那么从 GDT 中取出描述符进行跳转时，会取得非法的描述符，因此运行会出错。

##### Exercise 6

此处要求观察内存地址 0x00100000 处的数据

一开始内存里没有数据，那自然是全 0 。到达 0x7c00 时，ROM BIOS 已经将主引导扇区（ MBR ）中的 boot loader 加载到对应位置，但此时内核尚未加载，因此 0x00100000 处的数据仍为全0。再次设置断点为 0x10000c ，即为内核的 entry ，到达此处时 0x00100000 已有数据，因为内核已经被加载好了，运行过程如下。

```
(gdb) x/8x 0x00100000
0x100000:       0x00000000      0x00000000      0x00000000      0x00000000
0x100010:       0x00000000      0x00000000      0x00000000      0x00000000
(gdb) b *0x7c00
Breakpoint 1 at 0x7c00
(gdb) c
Continuing.
[   0:7c00] => 0x7c00:  cli    

Breakpoint 1, 0x00007c00 in ?? ()
(gdb) x/8x 0x00100000
0x100000:       0x00000000      0x00000000      0x00000000      0x00000000
0x100010:       0x00000000      0x00000000      0x00000000      0x00000000
(gdb) b *0x10000c
Breakpoint 2 at 0x10000c
(gdb) c
Continuing.
The target architecture is set to "i386".
=> 0x10000c:    movw   $0x1234,0x472

Breakpoint 2, 0x0010000c in ?? ()
(gdb) x/8x 0x00100000
0x100000:       0x1badb002      0x00000000      0xe4524ffe      0x7205c766
0x100010:       0x34000004      0x2000b812      0x220f0011      0xc0200fd8
```

### Part 3: The Kernel

##### Exercise 7

执行 ```movl %eax, %cr0``` 前 0x00100000 上是有数据的，而 0xf0100000 不可以访问，这里是因为 qemu 虚拟机未持有这么大的物理内存，如果计算机的物理内存大于等于 4 GB ，访问此处就会得到全 0 的数据。执行后两处数据相等，这是因为这条语句将 eax 中设置好的比特位存入 cr0 ，详见 [CR0](https://en.wikipedia.org/wiki/Control_register#CR0) 。那么 CPU 此时就会开启分页机制，开启分页机制后 CPU 访问内存就会通过 cr3 储存的多级页表访问，这个页表将底端 256 MB 的物理内存即 0x00000000 到 0x0fffffff 映射到了线性地址空间的顶端即 0xf0000000 到 0xffffffff 。

```
(gdb) b *0x100025
Breakpoint 1 at 0x100025
(gdb) c
Continuing.
The target architecture is set to "i386".
=> 0x100025:    mov    %eax,%cr0

Breakpoint 1, 0x00100025 in ?? ()
(gdb) x/4x 0x00100000
0x100000:       0x1badb002      0x00000000      0xe4524ffe      0x7205c766
(gdb) x/4x 0xf0100000
0xf0100000 <_start-268435468>:  Cannot access memory at address 0xf0100000
(gdb) si
=> 0x100028:    mov    $0xf010002f,%eax
0x00100028 in ?? ()
(gdb) x/4x 0x00100000
0x100000:       0x1badb002      0x00000000      0xe4524ffe      0x7205c766
(gdb) x/4x 0xf0100000
0xf0100000 <_start-268435468>:  0x1badb002      0x00000000      0xe4524ffe      0x7205c766
```

##### Exercise 8

这段代码在 lib/printfmt.c 的 vprintfmt 函数中的 reswitch 这段中， reswitch 用来判断 % 符号后的字符以决定输出的格式，其中漏了字符为 ‘o’ 的 case。补全很简单，需要的功能都封装好了，代码如下：

```c
	  ...
	  // (unsigned) octal
      case 'o':
        num = getuint(&ap, lflag);
        base = 8;
        goto number;
      ...
```

后面有个 challenge ，要求实现字符颜色打印。可以给出现 % 打印符时多设计一种情况，通过参数传入，将颜色属性填到字符的高一个字节即可相应位置即可。我这里使用了 %a 组合来输入颜色：

```C
void vprintfmt(void (*putch)(int, void *), void *putdat, const char *fmt,
               va_list ap) {
  ...
  while (1) {
    while ((ch = *(unsigned char *)fmt++) != '%') {
      if (ch == '\0') return;
      if (color) {
        ch |= color << 8;
        color = 0;
      }
      putch(ch, putdat);
    }
    ...
  reswitch:
    switch (ch = *(unsigned char *)fmt++) {
      ...
      // color
      case 'a':
        color = va_arg(ap, int);
        break;
  ...
}     
```

**注意：如果要打印颜色， 启动 qemu 时应使用 make qemu 而不是 make qemu-nox ，否则没有虚拟显示器。**

将 test_backtrace 函数里的打印语句稍作修改如下：

```C
void
test_backtrace(int x)
{
	...
	cprintf("%al%ae%aa%av%ai%an%ag test_backtrace %d\n",
			0x02, 0x02, 0x02, 0x02, 0x02, 0x02, 0x02, x);
}
```

效果如下：

![image-20220924192457196](F:\study\6.828\lab1.assets\image-20220924192457196.png)

##### Exercise 9

查看 kern/entry.S 的代码，可以看到在 relocated 标识后的代码有：

```assembly
relocated:
	...
	# Set the stack pointer
	movl	$(bootstacktop),%esp
	...
```

此处修改了栈指针寄存器（ esp ），平坦模型下内存寻址中的段地址统一为 0 ，因此修改 esp 即是修改栈顶位置，是内核初始化栈的行为。 x86 CPU 栈增长的方向为从高地址向低地址，因此 esp 指向的是栈的顶端。

##### Exercise 10

obj/kern/kernal.asm 中 test_backtrace 函数如下：

```assembly
void
test_backtrace(int x)
{
f0100040:	55                   	push   %ebp
f0100041:	89 e5                	mov    %esp,%ebp
f0100043:	56                   	push   %esi
f0100044:	53                   	push   %ebx
f0100045:	e8 72 01 00 00       	call   f01001bc <__x86.get_pc_thunk.bx>
f010004a:	81 c3 be 12 01 00    	add    $0x112be,%ebx
f0100050:	8b 75 08             	mov    0x8(%ebp),%esi
	cprintf("entering test_backtrace %d\n", x);
f0100053:	83 ec 08             	sub    $0x8,%esp
f0100056:	56                   	push   %esi
f0100057:	8d 83 18 08 ff ff    	lea    -0xf7e8(%ebx),%eax
f010005d:	50                   	push   %eax
f010005e:	e8 87 0a 00 00       	call   f0100aea <cprintf>
	if (x > 0)
f0100063:	83 c4 10             	add    $0x10,%esp
f0100066:	85 f6                	test   %esi,%esi
f0100068:	7e 29                	jle    f0100093 <test_backtrace+0x53>
		test_backtrace(x-1);
f010006a:	83 ec 0c             	sub    $0xc,%esp
f010006d:	8d 46 ff             	lea    -0x1(%esi),%eax
f0100070:	50                   	push   %eax
f0100071:	e8 ca ff ff ff       	call   f0100040 <test_backtrace>
f0100076:	83 c4 10             	add    $0x10,%esp
	else
		mon_backtrace(0, 0, 0);
	cprintf("leaving test_backtrace %d\n", x);
f0100079:	83 ec 08             	sub    $0x8,%esp
f010007c:	56                   	push   %esi
f010007d:	8d 83 34 08 ff ff    	lea    -0xf7cc(%ebx),%eax
f0100083:	50                   	push   %eax
f0100084:	e8 61 0a 00 00       	call   f0100aea <cprintf>
}
f0100089:	83 c4 10             	add    $0x10,%esp
f010008c:	8d 65 f8             	lea    -0x8(%ebp),%esp
f010008f:	5b                   	pop    %ebx
f0100090:	5e                   	pop    %esi
f0100091:	5d                   	pop    %ebp
f0100092:	c3                   	ret    
```

先看每一次 test_backtrace 调用前发生了什么， f010006d 地址处的 ```lea    -0x1(%esi),%eax``` 通过 lea 指令给 esi 减 1 并加载到 eax 中，然后压入栈中。然后 call 调用函数， call 指令执行时会将 eip 压入栈中。

进入 test_backtrace 函数后，从 f0100040 地址处开始看，会分别压入 ebp、 esi 、 ebx 。

##### Exercise 11

由上一个实验可以得知， C 函数调用时，栈上的数据结构应该是这样的：

```
high  |            |
  ^   |____________|
  |   |    arg0    |
  |   |     ..     |
  |   |    argN    |
  |   |  old %eip  |
  |   |  old %ebp  |  <--  %ebp  
  |   | local vars |
  |   | saved regs |  <--- %esp
  v   |------------|
 low  |            |
```

那么要实现 mon_backtrace 函数，可以先通过 read_ebp 函数获取当前 ebp ， 然后 ebp + 4 字节的位置就是 eip 。同理也可以获取参数的位置，实现如下：

```c
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	uint32_t ebp = read_ebp();
	uint32_t eip = *((uint32_t*)ebp+1);
	
	cprintf("Stack backtrace:\n");
	while (eip >= 0xf010003e) {
		cprintf("  ebp %08x  eip %08x  args %08x %08x %08x %08x %08x\n",
			ebp, eip, *((uint32_t*)ebp+2), *((uint32_t*)ebp+3),
			 *((uint32_t*)ebp+4), *((uint32_t*)ebp+5), *((uint32_t*)ebp+6));

		ebp = *(uint32_t*)ebp;
		eip = *((uint32_t*)ebp+1);
	}

	return 0;
}
```

##### Exercise 12

这里要求进一步完善 mon_backtrace ，主要是通过查符号表来根据 eip 的值获取当前所处文件和函数。首先要完善 debuginfo_eip 函数，其中缺少了对 eip_line 数据的获取：

```C
int
debuginfo_eip(uintptr_t addr, struct Eipdebuginfo *info)
{
    ...
	// Search within [lline, rline] for the line number stab.
	// If found, set info->eip_line to the right line number.
	// If not found, return -1.
	//
	// Hint:
	//	There's a particular stabs type used for line numbers.
	//	Look at the STABS documentation and <inc/stab.h> to find
	//	which one.
	// Your code here.
	stab_binsearch(stabs, &lline, &rline, N_SLINE, addr);
	if (lline <= rline)
		info->eip_line = stabs[lline].n_desc;
    ...
}
```

最后， mon_backtrace 实现如下：

```C
int
mon_backtrace(int argc, char **argv, struct Trapframe *tf)
{
	uint32_t ebp = read_ebp();
	uint32_t eip = *((uint32_t*)ebp+1);
	struct Eipdebuginfo info;

	cprintf("Stack backtrace:\n");
	while (eip >= 0xf010003e) {
		cprintf("  ebp %08x  eip %08x  args %08x %08x %08x %08x %08x\n",
			ebp, eip, *((uint32_t*)ebp+2), *((uint32_t*)ebp+3),
			 *((uint32_t*)ebp+4), *((uint32_t*)ebp+5), *((uint32_t*)ebp+6));
		if (debuginfo_eip(eip, &info) == 0) 
			cprintf("         %s:%d: %.*s+%d\n", info.eip_file, info.eip_line, 
				info.eip_fn_namelen, info.eip_fn_name, eip-info.eip_fn_addr);
		else 
			cprintf("         Can't find debug info.\n");

		ebp = *(uint32_t*)ebp;
		eip = *((uint32_t*)ebp+1);
	}

	return 0;
}
```