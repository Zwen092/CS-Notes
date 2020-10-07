## 1. 实验目的

+ 逆向工程`bomb.c`主函数程序，获取每个phase的密钥
+ [课程目标与工具](http://csapp.cs.cmu.edu/3e/bomblab.pdf)
+ 大致完成以下步骤：
  + 阅读`bomb.c`的注释与代码 & 阅读`bomb`的反汇编代码；
  + 分析程序运行的思路，并推测defuse bomb中当前的phase key是什么；
  + 通过gdb等方式，验证并测试自己的猜测；
  + 回到2，直到解决所有问题

+ GDB(GNU SYMBOLIC DEBUGGER)

  + `print $[register name]`: 显示寄存器的值

  + `x/NFU`

    + x: examine

    + N: 要显示的内存单元的个数

    + U：格式，b、h、w、g



## 2. 实验分析

实验给出了一个二进制文件`bomb`，反汇编它:

`objdump -d bomb > bomb.asm`

得到`bomb.asm`，这里用sublime3打开，正式进入实验



### **Phase_1**

+ ```asm
  	0000000000400ee0 <phase_1>:
    400ee0:	48 83 ec 08          	sub    $0x8,%rsp
    400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
    400ee9:	e8 4a 04 00 00       	callq  401338 <strings_not_equal>
    400eee:	85 c0                	test   %eax,%eax
    400ef0:	74 05                	je     400ef7 <phase_1+0x17>
    400ef2:	e8 43 05 00 00       	callq  40143a <explode_bomb>
    400ef7:	48 83 c4 08          	add    $0x8,%rsp
    400efb:	c3                   	retq   
  ```

  代码很短，400ee9处调用函数`strings_not_equal`。此时参数1为rdi，**值为输入的字符串的地址**；参数2为rdi，**恰巧看起来也是个地址**，则在gdb里将其输出一下:

  ```
  (gdb) x/s 0x402400
  0x402400: "Border relations with Canada have never been better."
  ```

  结合函数名`strings_not_equal`可推断其Phase_1的答案。为了求证，进入`strings_not_equal`:

  ```asm
  0000000000401338 <strings_not_equal>:
    401338:	41 54                	push   %r12
    40133a:	55                   	push   %rbp
    40133b:	53                   	push   %rbx
    40133c:	48 89 fb             	mov    %rdi,%rbx	//用rbp保存rdi
    40133f:	48 89 f5             	mov    %rsi,%rbp	//用rbp保存rsi
    401342:	e8 d4 ff ff ff       	callq  40131b <string_length> 
    401347:	41 89 c4             	mov    %eax,%r12d	//返回值1存入r12d
    40134a:	48 89 ef             	mov    %rbp,%rdi	//rsi作为参数
    40134d:	e8 c9 ff ff ff       	callq  40131b <string_length> //获得返回值2
    401352:	ba 01 00 00 00       	mov    $0x1,%edx //edx = 1
    401357:	41 39 c4             	cmp    %eax,%r12d	//比较两个返回值
    40135a:	75 3f                	jne    40139b <strings_not_equal+0x63> //不等，调转后eax = 1，爆炸
    40135c:	0f b6 03             	movzbl (%rbx),%eax	//输入的第一个字符的编码给到eax
    40135f:	84 c0                	test   %al,%al	//如果为0，即字符为null，通过
    //这里逻辑很迷，能到这里说明输入的字符串长度一致，首字符如何为null？
    401361:	74 25                	je     401388 <strings_not_equal+0x50>
    401363:	3a 45 00             	cmp    0x0(%rbp),%al //al与已有字符串的第一个字符开始比较，相等则跳转
    401366:	74 0a                	je     401372 <strings_not_equal+0x3a>
    401368:	eb 25                	jmp    40138f <strings_not_equal+0x57> //不等则爆炸
    40136a:	3a 45 00             	cmp    0x0(%rbp),%al //不太懂为啥还要对比一次
    40136d:	0f 1f 00             	nopl   (%rax)	//无操作
    401370:	75 24                	jne    401396 <strings_not_equal+0x5e> //不等则爆炸
    401372:	48 83 c3 01          	add    $0x1,%rbx //char是1个字节，加1得到下个char的地址
    401376:	48 83 c5 01          	add    $0x1,%rbp	//同上
    40137a:	0f b6 03             	movzbl (%rbx),%eax	//rbx值给到eax
    40137d:	84 c0                	test   %al,%al //相等为0表示null，即字符串结尾，跳转
    40137f:	75 e9                	jne    40136a <strings_not_equal+0x32>//不等则循环对比
    401381:	ba 00 00 00 00       	mov    $0x0,%edx
    401386:	eb 13                	jmp    40139b <strings_not_equal+0x63>
    401388:	ba 00 00 00 00       	mov    $0x0,%edx
    40138d:	eb 0c                	jmp    40139b <strings_not_equal+0x63>
    40138f:	ba 01 00 00 00       	mov    $0x1,%edx
    401394:	eb 05                	jmp    40139b <strings_not_equal+0x63>
    401396:	ba 01 00 00 00       	mov    $0x1,%edx
    40139b:	89 d0                	mov    %edx,%eax
    40139d:	5b                   	pop    %rbx
    40139e:	5d                   	pop    %rbp
    40139f:	41 5c                	pop    %r12
    4013a1:	c3                   	retq   
  ```

  如上图，可以看到`strings_not_equal`的判断逻辑为先对比长度，然后逐个字符对比，直到字符串末尾，如果完全相同，则返回值eax = 0，Phase_1通过。

  

### Phase_2

```asm
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp //开辟栈空间
  400f02:	48 89 e6             	mov    %rsp,%rsi //给第二个参数赋值
  400f05:	e8 52 05 00 00       	callq  40145c <read_six_numbers>
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp) //比较1和rsp地址的值
  400f0e:	74 20                	je     400f30 <phase_2+0x34> //相等跳转
  400f10:	e8 25 05 00 00       	callq  40143a <explode_bomb> //不等爆炸
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34> //跳转
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax //eax = M(rbx - 4)
  400f1a:	01 c0                	add    %eax,%eax //eax = 2eax
  400f1c:	39 03                	cmp    %eax,(%rbx)  //M(rbx) - eax
  400f1e:	74 05                	je     400f25 <phase_2+0x29> //相等跳转
  400f20:	e8 15 05 00 00       	callq  40143a <explode_bomb> //不等爆炸
  400f25:	48 83 c3 04          	add    $0x4,%rbx //rbx = rbx + 4
  400f29:	48 39 eb             	cmp    %rbp,%rbx //rbx - rbp
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b> //不等跳转
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40> //跳转
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx //rbx = rsp + 4
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp //rbp = rsp + 18
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	retq
  ...
  ...
  000000000040145c <read_six_numbers>:
  40145c:	48 83 ec 18          	sub    $0x18,%rsp
  401460:	48 89 f2             	mov    %rsi,%rdx # rdx = rsi
  401463:	48 8d 4e 04          	lea    0x4(%rsi),%rcx  # rcx = rsi+4
  401467:	48 8d 46 14          	lea    0x14(%rsi),%rax  # rax = rsi+0x14
  40146b:	48 89 44 24 08       	mov    %rax,0x8(%rsp)  # m( rsp+0x8) = rax
  401470:	48 8d 46 10          	lea    0x10(%rsi),%rax # rax = rsi + 0x10
  401474:	48 89 04 24          	mov    %rax,(%rsp) # R[rsp] = rax
  401478:	4c 8d 4e 0c          	lea    0xc(%rsi),%r9 # r9 = rsi + 0xc
  40147c:	4c 8d 46 08          	lea    0x8(%rsi),%r8 # r8 = rsi + 0x8
  401480:	be c3 25 40 00       	mov    $0x4025c3,%esi # second para
  401485:	b8 00 00 00 00       	mov    $0x0,%eax
  40148a:	e8 61 f7 ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  40148f:	83 f8 05             	cmp    $0x5,%eax
  401492:	7f 05                	jg     401499 <read_six_numbers+0x3d>
  401494:	e8 a1 ff ff ff       	callq  40143a <explode_bomb>
  401499:	48 83 c4 18          	add    $0x18,%rsp
  40149d:	c3                   	retq   
```



**明确一下寄存器传递参数的顺序为：rdi, rsi, rdx, rcx, r8, r9**，超过的部分需要通过内存读取

Phase_2一开始 **将rsp给到rsi** 并调用`read_six_numbers`函数，进入此函数后发现此函数在分配好参数后调用了`<isoc99_sscanf@plt`>函数，其声明为：

`int sscanf(const char *str, const char *format, ...)`

1. 附加参数： 这个函数接受一系列的指针作为附加参数，每一个指针都指向一个对象，对象类型由 format 字符串中相应的 % 标签指定，参数与 % 标签的顺序相同。

​		针对检索数据的 format 字符串中的每个 format 说明符，应指定一个附加参数。如果您想要把 sscanf 操作的		结果存储在一个普通的变量中，您应该在标识符前放置引用运算符（&），例如：

```
int n;
sscanf (str,"%d",&amp;n);
```

2. 返回值：成功返回匹配和赋值的个数，如达到末尾或发生度错误，则返回EOF



上述分析可得：`read_six_numbers`函数作用是将输入的字符串中的整数存到 `rsi, rsi + 0x4, rsi + 0x8, rsi + 0xc, rsi + 0x10, rsi + 0x14 ` 这六个位置，即原函数`rsp`到`rsp + 0x14` 对应位置。

到这里Phase_2的逻辑就非常简单了，R[rsp] = 1，然后比较看后一个数是否为前一个数的2倍，因此答案为: 

**1 2 3 4 5 6**



### Phase_3

```asm
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp        //开栈空间
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx	 //rcx = m(rsp + c)
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx    //rdx = m(rsp + 8)
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi    //
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	callq  400bf0 <__isoc99_sscanf@plt> //调用sscanf，从字符串读取格式化输入
  400f60:	83 f8 01             	cmp    $0x1,%eax
  400f63:	7f 05                	jg     400f6a <phase_3+0x27> //eax > 1 jump
  400f65:	e8 d0 04 00 00       	callq  40143a <explode_bomb>
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)        
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a > //m(rsp + 8) > 7, boom!
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax //eax = m(rsp + 8)
  400f75:	ff 24 c5 70 24 40 00 	jmpq   *0x402470(,%rax,8)
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	callq  40143a <explode_bomb> //boom!
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax        //eax == m(rsp + c)
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	callq  40143a <explode_bomb>
  400fc9:	48 83 c4 18          	add    $0x18,%rsp
  400fcd:	c3                   	retq   
```



有了Phase_2的基础，可以很快明确输入的两个数位于`rsp + 0x8` 与`rsp + 0xc`处，输入的第一个数**小于**7且为无符号数。根据输入的第一个数确定跳转位置，查看一下：

```asm
(gdb) x/8ag 0x402470
0x402470:       0x400f7c <phase_3+57>   0x400fb9 <phase_3+118>
0x402480:       0x400f83 <phase_3+64>   0x400f8a <phase_3+71>
0x402490:       0x400f91 <phase_3+78>   0x400f98 <phase_3+85>
0x4024a0:       0x400f9f <phase_3+92>   0x400fa6 <phase_3+99>
```

即根据第一个数对第二个数进行赋值，答案为以下任何一组：

(0, 0xcf), (1, 0x137), (2, 0x2c3), (3, 0x100), (4, 0x185), (5, 0xce), (6, 0x2aa), (7, 0x147)



### Phase_4

```asm
000000000040100c <phase_4>:
  40100c:	48 83 ec 18          	sub    $0x18,%rsp
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  401029:	83 f8 02             	cmp    $0x2,%eax
  40102c:	75 07                	jne    401035 <phase_4+0x29>
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp) //
  401033:	76 05                	jbe    40103a <phase_4+0x2e> //输入的第一个数《=0xe否则爆炸
  401035:	e8 00 04 00 00       	callq  40143a <explode_bomb>
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx
  40103f:	be 00 00 00 00       	mov    $0x0,%esi
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi
  401048:	e8 81 ff ff ff       	callq  400fce <func4>
  40104d:	85 c0                	test   %eax,%eax
  40104f:	75 07                	jne    401058 < phase_4+0x4c> //返回值！=0 爆炸
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp) //第二个参数与0 不等爆炸
  401056:	74 05                	je     40105d <phase_4+0x51>
  401058:	e8 dd 03 00 00       	callq  40143a <explode_bomb>
  40105d:	48 83 c4 18          	add    $0x18,%rsp
  401061:	c3                   	retq   
  ..
  ..
  0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp
  400fd2:	89 d0                	mov    %edx,%eax //eax = 0xe
  400fd4:	29 f0                	sub    %esi,%eax //eax = 0xe - 0 = 0xe
  400fd6:	89 c1                	mov    %eax,%ecx //ecx = 0xe
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx //ecx = 0
  400fdb:	01 c8                	add    %ecx,%eax //eax = 0xe
  400fdd:	d1 f8                	sar    %eax //eax = 7
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx //ecx = 7
  400fe2:	39 f9                	cmp    %edi,%ecx  //7-p
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24> //<= 跳转
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx //edx = rcx - 0x1 = 6
  400fe9:	e8 e0 ff ff ff       	callq  400fce <func4>
  400fee:	01 c0                	add    %eax,%eax
  400ff0:	eb 15                	jmp    401007 <func4+0x39>
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax
  400ff7:	39 f9                	cmp    %edi,%ecx//
  400ff9:	7d 0c                	jge    401007 <func4+0x39> //ecx > 
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi //esi = 
  400ffe:	e8 cb ff ff ff       	callq  400fce <func4>
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	retq   
```

分析Phase_4可知，第二个参数必为0。当eax = 0时，通过；分析func4可得，当edi中的值为7时，eax = 0，满足条件，答案为(7,0)。

逆向工程func4可得如下c代码:

```c
// a:rdi b:rsi c:rdx
int fun4(int a, int b, int c)
{
  int return_value = c - b; //rax
  int t = ((unsigned)return_value) >> 31; //rcx
  return_value = (t + return_value) >> 1;
  t = return_value + b;
  if(t - a <= 0){
    return_value = 0;
    if(t - a >= 0){
      return return_value;
    }else{
      b = t + 1;
      int r = func4(a,b,c);
      return 2*r + 1;
    }
  }else{
    c = t - 1;
    int r = func4(a, b, c);
    return 2*r;
  }
}

//代码修整后可得
//x = input, y = 0; z = 14
int func4(int x, int y, int z)
{
  int k = z - y;
  k = ((int)((unsigned)k>>31) + k) >>1) + y;
  if(k < x)
    return 2*func4(x, k + 1, z) + 1;
  else if(k > x)
    return 2*func4(x, y, k  - 1);
  else
    return 0;
}

//由于x的取值范围为[0,14]，故可以穷举
int main(int argc, char const *argv[])
{
	for (int i = 0; i <= 14; ++i)
	{
		if(!func4(i, 0, 14))
			printf("%d\n", i);
	}
	return 0;
}
因此 i = 0, 1, 3 ,7
```



### Phase_5

```asm
0000000000401062 <phase_5>:
  401062:	53                   	push   %rbx
  401063:	48 83 ec 20          	sub    $0x20,%rsp # 
  401067:	48 89 fb             	mov    %rdi,%rbx
  40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax # canary value
  401071:	00 00 
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp) # 
  401078:	31 c0                	xor    %eax,%eax # put to zero
  40107a:	e8 9c 02 00 00       	callq  40131b <string_length>
  40107f:	83 f8 06             	cmp    $0x6,%eax 
  401082:	74 4e                	je     4010d2 <phase_5+0x70> # string length = 6
  401084:	e8 b1 03 00 00       	callq  40143a <explode_bomb> 
  401089:	eb 47                	jmp    4010d2 <phase_5+0x70>
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx # ecx = M(rbx)
  40108f:	88 0c 24             	mov    %cl,(%rsp)   # rsp store 1nd char
  401092:	48 8b 14 24          	mov    (%rsp),%rdx  # 
  401096:	83 e2 0f             	and    $0xf,%edx    # edx = 0xf
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx   # (edx) = l
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1) # (rps+10) = l
  4010a4:	48 83 c0 01          	add    $0x1,%rax # rax = 1
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
  4010bd:	e8 76 02 00 00       	callq  401338 <strings_not_equal>
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  4010c6:	e8 6f 03 00 00       	callq  40143a <explode_bomb>
  4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
  4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
  4010d2:	b8 00 00 00 00       	mov    $0x0,%eax     
  4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
  4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
  4010de:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
  4010e5:	00 00 
  4010e7:	74 05                	je     4010ee <phase_5+0x8c>
  4010e9:	e8 42 fa ff ff       	callq  400b30 <__stack_chk_fail@plt>
  4010ee:	48 83 c4 20          	add    $0x20,%rsp
  4010f2:	5b                   	pop    %rbx
  4010f3:	c3                   	retq   

```

分析40108b到4010ac可知，这段逻辑为将输入的字符串逐个与0xf做与运算，将得到的值加上0x4024b0后的地址的值存到rsp+0x10到rsp+0x15，然后取出这段字符串，与给定的字符串做对比，相同则通过。

打印两段关键地址:

```
(gdb) x/s 0x40245e
0x40245e:       "flyers"

(gdb) x/s 0x4024b0
0x4024b0 <array.3449>:  "maduiersnfotvbylSo you think you can stop the bomb with
 ctrl-c, do you?"
```

分析到这里可知，需找到特定的字符，使得其与0xf做与运算后，得到的值加上0x4024b0后分别取得"f l y e r s"，基于此分析可得答案为:

![img](https://pic3.zhimg.com/v2-e124b1056f2b52623a293e58bdf18fe4_b.jpg)

取上表中任意“flyers”对应的字符即可



### Phase_6

代码太长，这里只放重要代码片段：

首先是`read_six_numbers`函数，然后是一段循环判断逻辑，保证所有的数都位于[1, 6]之间且互不相等

```asm
	40110b:	49 89 e6             	mov    %rsp,%r14
  40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d
  401114:	4c 89 ed             	mov    %r13,%rbp
  401117:	41 8b 45 00          	mov    0x0(%r13),%eax
  40111b:	83 e8 01             	sub    $0x1,%eax
  40111e:	83 f8 05             	cmp    $0x5,%eax
  401121:	76 05                	jbe    401128 <phase_6+0x34>
  401123:	e8 12 03 00 00       	callq  40143a <explode_bomb>
  401128:	41 83 c4 01          	add    $0x1,%r12d
  40112c:	41 83 fc 06          	cmp    $0x6,%r12d
  401130:	74 21                	je     401153 <phase_6+0x5f>
  401132:	44 89 e3             	mov    %r12d,%ebx
  401135:	48 63 c3             	movslq %ebx,%rax
  401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax
  40113b:	39 45 00             	cmp    %eax,0x0(%rbp)
  40113e:	75 05                	jne    401145 <phase_6+0x51>
  401140:	e8 f5 02 00 00       	callq  40143a <explode_bomb>
  401145:	83 c3 01             	add    $0x1,%ebx
  401148:	83 fb 05             	cmp    $0x5,%ebx
  40114b:	7e e8                	jle    401135 <phase_6+0x41>
  40114d:	49 83 c5 04          	add    $0x4,%r13
  401151:	eb c1                	jmp    401114 <phase_6+0x20>
```

这一段将所有的数做7-x的操作

```asm
 401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi
  401158:	4c 89 f0             	mov    %r14,%rax
  40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
  401160:	89 ca                	mov    %ecx,%edx
  401162:	2b 10                	sub    (%rax),%edx
  401164:	89 10                	mov    %edx,(%rax)
  401166:	48 83 c0 04          	add    $0x4,%rax
  40116a:	48 39 f0             	cmp    %rsi,%rax
  40116d:	75 f1                	jne    401160 <phase_6+0x6c>
```



这一段为解题关键

```asm
 40116f:	be 00 00 00 00       	mov    $0x0,%esi
  401174:	eb 21                	jmp    401197 <phase_6+0xa3>
  401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx
  40117a:	83 c0 01             	add    $0x1,%eax
  40117d:	39 c8                	cmp    %ecx,%eax
  40117f:	75 f5                	jne    401176 <phase_6+0x82>
  401181:	eb 05                	jmp    401188 <phase_6+0x94>
  401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)
  40118d:	48 83 c6 04          	add    $0x4,%rsi
  401191:	48 83 fe 18          	cmp    $0x18,%rsi
  401195:	74 14                	je     4011ab <phase_6+0xb7>
  401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx
  40119a:	83 f9 01             	cmp    $0x1,%ecx
  40119d:	7e e4                	jle    401183 <phase_6+0x8f>
  40119f:	b8 01 00 00 00       	mov    $0x1,%eax
  4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
  4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
```

为了方便理解，不如先打印出关键信息看看：

```
(gdb) x/24xw 0x6032d0
0x6032d0 <node1>:       0x0000014c      0x00000001      0x006032e0      0x00000000
0x6032e0 <node2>:       0x000000a8      0x00000002      0x006032f0      0x00000000
0x6032f0 <node3>:       0x0000039c      0x00000003      0x00603300      0x00000000
0x603300 <node4>:       0x000002b3      0x00000004      0x00603310      0x00000000
0x603310 <node5>:       0x000001dd      0x00000005      0x00603320      0x00000000
0x603320 <node6>:       0x000001bb      0x00000006      0x00000000      0x00000000
```

从这里可以看到，0x6032d0处的结构是可能是一个结构体，组成为:

```c
struct s{
  int sth;
  int input;
  node *next;
}
```

回过头来看，这一段的逻辑就是根据7-x的值，取0x6032d0到0x603320的地址，依次存放在rsp+20到rsp+48内

下面的代码对上面存放好的值进行了修改:

```asm
 	4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
  4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax
  4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi
  4011ba:	48 89 d9             	mov    %rbx,%rcx # rcx = head
  4011bd:	48 8b 10             	mov    (%rax),%rdx # rdx = head.next.value
  4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx) 
  4011c4:	48 83 c0 08          	add    $0x8,%rax
  4011c8:	48 39 f0             	cmp    %rsi,%rax
  4011cb:	74 05                	je     4011d2 <phase_6+0xde>
  4011cd:	48 89 d1             	mov    %rdx,%rcx
  4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
  4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)
  4011d9:	00 
```



分析发现是在修改链表的指向，举个例子：输入(3, 5 ,4 ,6, 2, 1) 得到(4,2,3,1,5,6)，则rsp+0x20到rsp+0x48中的存储的地址顺序为:

0x00603300, 0x006032e0, 0x006032f0, 0x006032d0, 0x00603310, 0x00603320

则本段代码将这一段地址以0x00603300为头结点依次相连，并以0x00000000结尾，为了验证，可在此段代码结束后打断点并打印执行后的相关信息:

```
(gdb) x/24xw 0x6032d0
0x6032d0 <node1>:       0x0000014c      0x00000001      0x00603310      0x00000000
0x6032e0 <node2>:       0x000000a8      0x00000002      0x006032f0      0x00000000
0x6032f0 <node3>:       0x0000039c      0x00000003      0x006032d0      0x00000000
0x603300 <node4>:       0x000002b3      0x00000004      0x006032e0      0x00000000
0x603310 <node5>:       0x000001dd      0x00000005      0x00603320      0x00000000
0x603320 <node6>:       0x000001bb      0x00000006      0x00000000      0x00000000
```

与我们的推断相符

进入最后的判定阶段：

```asm
  4011da:	bd 05 00 00 00       	mov    $0x5,%ebp
  4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax
  4011e3:	8b 00                	mov    (%rax),%eax
  4011e5:	39 03                	cmp    %eax,(%rbx)
  4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
  4011e9:	e8 4c 02 00 00       	callq  40143a <explode_bomb>
  4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx
  4011f2:	83 ed 01             	sub    $0x1,%ebp
  4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
  4011f7:	48 83 c4 50          	add    $0x50,%rsp
  4011fb:	5b                   	pop    %rbx
  4011fc:	5d                   	pop    %rbp
  4011fd:	41 5c                	pop    %r12
  4011ff:	41 5d                	pop    %r13
  401201:	41 5e                	pop    %r14
  401203:	c3                   	retq 
```

通过上述分析可知rbx为头结点，本段逻辑判断该列表是否为降序排列，因此需按照指定的顺序修改链表。将原始链表降序后得到：

```asm
0x6032f0 <node3>:       0x0000039c
0x603300 <node4>:       0x000002b3
0x603310 <node5>:       0x000001dd
0x603320 <node6>:       0x000001bb 
0x6032d0 <node1>:       0x0000014c 
0x6032e0 <node2>:       0x000000a8  
```

因此需要将链表修改为3-4-5-6-1-2，则应输入4-3-2-1-6-5，得解



## 3.实验感悟

+ 做了蛮久的，也参考了很多大佬的操作，已完全弄清所有操作的目的，收获良多
+ 以后的实验应该边做边记录，而不是像这一篇一样做完之后才写。