---
title: MIPS汇编语言入门
summary: MIPS：无互锁流水级的微处理器(Microprocessor without Interlocked Piped Stages) 
date: 2020-05-08
tags: ['System']
---
 MIPS架构（英语：MIPS architecture，为Microprocessor without Interlocked Pipeline Stages的缩写，亦为Millions of Instructions Per Second的头字语），是一种采取精简指令集（RISC）的处理器架构，1981年出现，由MIPS科技公司开发并授权，广泛被使用在许多电子产品、网络设备、个人娱乐设备与商业设备上。最早的MIPS架构是32位，最新的版本已经变成64位。  
在1981年，斯坦福大学教授约翰·轩尼诗领导他的团队，实现出第一个MIPS架构的处理器。他们原始的概令是透过指令管线化来增加CPU运算的速度。
1984年，约翰·轩尼诗教授离开斯坦福大学，创立MIPS科技公司。于1985年，设计出R2000芯片，1988年，将其改进为R3000芯片。

# 机器周期

大多数计算机处理器都会不断地重复三个基本步骤。每个机器周期内会执行一条机器指令。一个现代的计算机处理器每秒钟运行数百万次机器周期。  
一条机器指令是由一串对应着处理器基本操作的二进制码组成的，在不同的处理器架构中，机器周期的组成也不相同，但他们的基本行为都包含下面三个主要步骤： 

- 从内存中读取指令：指令存放在内存中，PC *(Program Counter)* 存放了指令在内存中的地址

- PC=PC+4：让PC指向下一条指令所在的地址

- 执行所得到的指令

在一台32位的处理器上，内存地址为32位宽，因此PC也拥有长度为32位的地址

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856574725_460.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

# 汇编语言

机器指令是由0,1的二进制码构成的因而人类无法阅读。相对应地，汇编语言允许我们使用相应的代码来编写指令。下面是对应的机器码和汇编语言：

machine instruction  
`0000 0001 0010 1011 1000 0000 0010 0000`

assembly language statement  
```mipsasm
add $t0,$t1,$t2
```

这条指令的意思是：寄存器$t0,$t1,$t2间
```
$t0 = $t1 + $t2
```

![conceputual view.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856575689_8605.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0 "Conceptual view")

# MIPS

## 寄存器种类

- MIPS下一共有32个通用寄存器
- 在汇编中，寄存器标志由$符开头
- 寄存器表示可以有两种方式
  - 直接使用该寄存器对应的编号，例如：从$0到$31
  - 使用对应的寄存器名称，例如：$t1, $sp
- 对于乘法和除法分别有对应的两个寄存器$lo, $hi
  - 对于以上二者，不存在直接寻址；必须要通过mfhi("move from hi")以及mflo("move from lo")分别来进行访问对应的内容
  - 乘法：`HI` 存储32位高位， `LO`存储32位低位
  - 除法：`LO`存储结果，`HI`存储余数
- 栈的走向是从高地址到低地址



| REGISTER | NAME      | USAGE                                                        |
| -------- | --------- | ------------------------------------------------------------ |
| $0       | $zero     | 常量0(constant value 0)                                      |
| $1       | $at       | 保留给汇编器(Reserved for assembler)                         |
| $2-$3    | $v0 - $v1 | 函数调用返回值(values for results and expression evaluation) |
| $4-$7    | $a0-$a3   | 函数调用参数(arguments)                                      |
| $8-$15   | $t0-$t7   | 暂时的(或随便用的)                                           |
| $16-$23  | $s0-$s7   | 保存的(或如果用，需要SAVE/RESTORE的)(saved)                  |
| $24-$25  | $t8-$t9   | 暂时的(或随便用的)                                           |
| $28      | $gp       | 全局指针(Global Pointer)                                     |
| $29      | $sp       | 堆栈指针(Stack Pointer)                                      |
| $30      | $fp       | 帧指针(Frame Pointer)                                        |



说明

1. $0:即$zero,该寄存器总是返回零，为0这个有用常数提供了一个简洁的编码形式。  
```mipsasm
move $t0,$t1
#实际为  
add $t0,$0,$t1
```
使用伪指令可以简化任务，汇编程序提供了比硬件更丰富的指令集。
2. $1:即 $at，该寄存器为汇编保留，由于I型指令的立即数字段只有16位，在加载大常数时，编译器或汇编程序需要把大常数拆开，然后重新组合到寄存器里。比如加载一个32位立即数需要 `lui`（装入高位立即数）和`addi`两条指令。像MIPS程序拆散和重装大常数由汇编程序来完成，汇编程序必需一个临时寄存器来重组大常数，这也是为汇编保留$at的原因之一。
3. $2..$3:($v0-$v1)用于子程序的非浮点结果或返回值，对于子程序如何传递参数及如何返回，MIPS范围有一套约定，堆栈中少数几个位置处的内容装入CPU寄存器，其相应内存位置保留未做定义，当这两个寄存器不够存放返回值时，编译器通过内存来完成。
4. $4..$7:($a0-$a3)用来传递前四个参数给子程序，不够的用堆栈。a0-a3和v0-v1以及ra一起来支持子程序／过程调用，分别用以传递参数，返回结果和存放返回地址。当需要使用更多的寄存器时，就需要堆栈（stack)了,MIPS编译器总是为参数在堆栈中留有空间以防有参数需要存储。
5. $8..$15:($t0-$t7)临时寄存器，子程序可以使用它们而不用保留。
6. $16..$23:($s0-$s7)保存寄存器，在过程调用过程中需要保留（被调用者保存和恢复，还包括$fp和$ra），MIPS提供了临时寄存器和保存寄存器，这样就减少了寄存器溢出（spilling,即将不常用的变量放到存储器的过程),编译器在编译一个叶（leaf)过程（不调用其它过程的过程）的时候，总是在临时寄存器分配完了才使用需要保存的寄存器。
7. $24..$25:($t8-$t9)同($t0-$t7)
$26..$27:($k0,$k1)为操作系统／异常处理保留，至少要预留一个。 异常（或中断）是一种不需要在程序中显示调用的过程。MIPS有个叫异常程序计数器（exception program counter,EPC)的寄存器，属于CP0寄存器，用于保存造成异常的那条指令的地址。查看控制寄存器的唯一方法是把它复制到通用寄存器里，指令`mfc0`(move from system control)可以将EPC中的地址复制到某个通用寄存器中，通过跳转语句（`jr`)，程序可以返回到造成异常的那条指令处继续执行。MIPS程序员都必须保留两个寄存器$k0和$k1，供操作系统使用。  
发生异常时，这两个寄存器的值不会被恢复，编译器也不使用k0和k1,异常处理函数可以将返回地址放到这两个中的任何一个，然后使用jr跳转到造成异常的指令处继续执行。
8. $28:($gp)为了简化静态数据的访问，MIPS软件保留了一个寄存器：全局指针gp(global pointer,$gp)，全局指针指向静态数据区中的运行时决定的地址，在存取位于gp值上下32KB范围内的数据时，只需要一条以gp为基指针的指令即可。在编译时，数据须在以gp为基指针的64KB范围内。
9. $29:($sp)MIPS硬件并不直接支持堆栈，你可以把它用于别的目的，但为了使用别人的程序或让别人使用你的程序，还是要遵守这个约定的，但这和硬件没有关系。
10. $30:($fp)GNU MIPS C编译器使用了帧指针(frame pointer),而SGI的C编译器没有使用，而把这个寄存器当作保存寄存器使用（$s8),这节省了调用和返回开销，但增加了代码生成的复杂性。
11. $31:($ra)存放返回地址，MIPS有个`jal`(jump-and-link,跳转并 链接)指令，在跳转到某个地址时，把下一条指令的地址放到$ra中。用于支持子程序，例如调用程序把参数放到$a0~$a3,然后`jal X`跳到X过程，被调过程完成后把结果放到$v0,$v1,然后使用`jr $ra`返回。



## 指令类型

### R型指令：操作寄存器

常见指令：`add`,  `sub`, `and`, `or`, `nor`, `slt`, `sll`, `srl`, `jr` 

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856576638_6558.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

- opcod: Operation Code
- rs: number of source register
- rt: number of source register
- rd: number of destination register
- sham: Shift amount (number of bits the operation is shifted)
- Func: function (opcod extension)

Sham仅用于操作偏移量的指令（例如`stl`）

### I型指令：操作常量

常见指令：`addi`, `lw`, `sw`, `lh`, `sh`, `lb`, `lbu`, `sb`, `ll`, `sc`, `lui`, `andi`, `ori`, `beq`, `bne`, `slti`, `sltiu`

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856577691_8905.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

- opcod: Operation Code
- rs: number of source register
- rd: number of source or destination register
- imd: Immediate value 即时量

### J型指令：跳跃寻址

处理器直接跳跃到给定的地址，执行所在地址的指令

常见指令：`j`, `jal`

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856578572_889.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

- Opcod: Operation Code
- Imd: immediate value


## 程序结构

### 代码

- 代码段以 .text为开始标志
- 包含了各项指令操作
- 程序入口为main：标志
- 程序结束标志
- 注释以#开始

### 数据声明

- 数据段以 .data为开始标志
- 声明变量后，即在主存中分配空间。

格式：  
`name: storage_type value(s)`  

- name:变量名
- storage_type:数据类型，可以为.word, .ascii, .data, .byte等等
  - 对于.space 需要指明需要多大空间(bytes)
  - .ascii 与 .asciiz唯一区别就是 后者会在字符串最后自动加上一个终止符
- value:初始值

例子：

```asm
.data
var1: .word 3 # 声明一个 word 类型的变量 var1, 同时给其赋值为 3
array1: .byte 'a','b' # 声明一个存储2个字符的数组 array1，并赋值 'a', 'b'
array2: .space 40 # 为变量 array2 分配 40字节（bytes)未使用的连续空间，最好事先注释数据类型
```

### 常用指令

#### 算术与逻辑指令 Arithmetic and logic instructions

- 最多3个操作数
- 在这里，操作数只能是寄存器，绝对不允许出现地址
- 所有指令统一是32位 = 4 * 8 bit = 4bytes = 1 word  

##### 算数指令

`add`用于实现将两个来源（source）寄存器内的值相加，并存储到目标（destination）寄存器中。然而，有时候两个寄存器内的值相加会超过32位。由于我们使用补码（Two's Complement）， 两个32位的值相加之和会达到33位。每个寄存器内的值在 $ [-2^{31} , 2^{31}-1]$区间，因此他们的和也就在$ [-2^{32} , 2^{32}-2]$区间。  

如果和超出了寄存器所存储值得区间范围，那么也就会发生**溢出（overflow）**。  

若发生位数溢出，处理器会拒绝执行`add`指令，并触发程序错误异常。  

而`addu`同为加法，但却不会判断溢出情况。  

对于减法`sub`和`subu`同理。

`addi rd rs imd`将即时值`imd`与寄存器`rs`中的值相加，并检查溢出情况。然而有时，rs的值是32位编码，但imd的值是16位编码，为了将这两个不同位数的值相加，需要将16位转换为32位。有两种方法：

- 如果我们认为这个imd是自然数，那么在其高位添加上16个0（也就是零位拓展）
- 如果是整数，那么将这个数的最高位（也就是第15位）复制16次添加到高位（因为是补码也就是符号位拓展）

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856579223_6318.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

在Mips架构中，我们认为即时数是整数，因此我们先将它做符号位拓展再与寄存器内的数相加。  

进一步通俗的来说，在算数运算中（arithmetic）我们认为immediate是整数；在逻辑运算（logic）中，我们认为immediate是自然数。  

`addiu`和`addi`相同，只不过它不判断溢出。



#####  逻辑运算

- `or rd rs rt` 或
- `and rd rs rt` 且
- `xor rd rs rt` 异或
- `nor rd rs rt` 非或

下同理

- `ori rd rs imd`
- `and rd rs imd`
- `xori rd rs imd`

要注意的是，此时immediate被看做是自然数，因而他执行的是零位拓展。

##### 移位运算

1. 逻辑移位
   ![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856579956_4580.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)
   `sll rd rt sham`: rt >> sham => rd, shift left logical 向左移位，将最高位丢弃，每位向左移动，最低位补0。
   ![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856580789_1625.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)
   `srl rd rt sham`: `rt << sham => rd` 向右移位
   `sllv rd rt rs`使用寄存器rs内的值做sham让rt向左逻辑移位  
   **注意：用rs寄存器内值做sham，仅有最低五位（也就是靠右的）是有意义的. **  
2. 算数移位
   ![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856581671_6406.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)
   `sra rd rt sham`: rt >> sham => rd shift right arithmetic  向右算数移位，即保持高位数不变，因移动而缺失的位置用原数字补上。常用除法：`srl $t1, $t0, 1` 将t0的值除于2；若除于4，则移动2位。  
   `srav rd rt rs`同上，用寄存器rs的值做`sra`操作。

##### 比较并设置

1. R型指令
   `slt rd rs rt`: set if less than: `if rs < rt` then `1 => rd`, else `0 => rd`; **operands are signed** （整数）  
   `sltu rd rs rt`: set if less than **unsigned**  （自然数）
2. I型指令
   `slti rd rs imd` set if less than immediate （sign extension注意是符号位拓展因为是算数操作）  
   `slitu rd rs imd` set if less than immediate unsigned 注意当需要16位转32位时，immediate被看作整数，即使用符号位拓展，因为是算数操作。在比较时，两个比较的值都将被视作自然数！  
   `lui rd imd`: load upper immediate 将imd的值载入到寄存器rd的高16位


##### 其他

```asm
add $t0,$t1,$t2 # $t0 = $t1 + $t2; add as signed (2's complement) integers

sub $t2,$t3,$t4     #$t2 = $t3 Ð $t4
addi $t2,$t3, 5     #$t2 = $t3 + 5;   "add immediate" (no sub immediate)
addu $t1,$t6,$t7    #$t1 = $t6 + $t7;   add as unsigned integers
subu $t1,$t6,$t7    #$t1 = $t6 + $t7;   subtract as unsigned integers

mult $t3,$t4        
#multiply 32-bit quantities in $t3 and $t4, and store 64-bit result in special registers Lo #and Hi:  (Hi,Lo) = $t3 * $t4　　　　　　　　　 
#运算结果存储在hi,lo（hi高位数据， lo地位数据）

div $t5,$t6		    
#  Lo = $t5 / $t6   (integer quotient)				    
#  Hi = $t5 mod $t6   (remainder)
#  商数存放在 lo, 余数存放在 hi

mfhi $t0		    
#move quantity in special register Hi to $t0:   $t0 = Hi
#不能直接获取 hi 或 lo中的值， 需要mfhi, mflo指令传值给寄存器

mflo $t1		
#move quantity in special register Lo to $t1:   $t1 = Lo
#used to get at result of product or quotient

move $t2,$t3	#$t2 = $t3复制粘贴
```

#### 内存访问（存储与读取）Memory access instructions

- 两种操作：存储，读取
- 数据大小：一字节（Byte），半个词（Half Word），一个词或以上（Word）

对于所有访问内存的指令，其计算地址的方式都是一样的：将寄存器register source的值与即时值immediate想加（采用算数指令的加法`Rs + Imd`）。然而，如果得到的地址处在系统权限内存区间（超出了 `0x7fff ffff`）而与此同时系统在用户模式（User）下，那么处理器就会触发异常，报告程序错误，这行指令就不会得到执行。  

同样的，如果地址没有对齐（读取Word一个词地址不是4的倍数，读取Half Word半个词不是2的倍数），那么处理器也会拒绝执行指令并触发异常。

1. `lw rd imd(rs) `: load word, 4 bytes from memory at `rs + imd` saved to `rd`
2. `sw rt imd(rs)`: save word, 4 bytes from memory `rt` written into memory at `rs + imd`. The conversion of word respect Little Endian
3. `lh rd (imd)rs`: load half word, 2 bytes from memory at `rs + imd` saved to `rd`, respect Little Endian. The half word read from memory is considered as a relative entire number so the MSB 16 bits will be filled with the 15 th bit of this half word. （signed sign extended 符号位拓展）![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856582329_8.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)
4. `lhu rd imd(rs)`: same with `lh` but the half word is considered as a natural entire number and the MSB 16 bits will be filled with fifteen 0. （unsigned sign extended零位拓展）
5. `sh rt imd(rs)`: store a half word, as `sw`
6. `lb`: load a byte, with signed sign extended
7. `sb`: save a byte
8. `lhu`: load a half word unsigned, with unsigned sign extended 


```assembly
#load word
lw register_destination, RAM_source
#copy word (4 bytes) at source RAM location to destination register.
#从内存中 复制 RAM_source 的内容到 对应的寄存器中（lw中的'w'意为'word',即该数据大小为4个字节）

lb register_destination, RAM_source
#copy byte at source RAM location to low-order byte of destination register,
# and sign-e.g.tend to higher-order bytes
#从内存RAM location中复制低端位的byte至目标寄存器；同上， lb 意为 load byte

#store word
sw register_source, RAM_destination
#store word in source register into RAM destination

sb register_source, RAM_destination
#store byte (low-order) in source register into RAM destination
#将指定寄存器中的数据 写入 到指定的内存中

#直接寻址:  
la $t0, var1
#copy RAM address of var1 (presumably a label defined in the program) into register $t0
#将var1在内存中的地址放入$t0，可以理解为指针

#间接寻址
lw $t2, ($t0)
#load word at RAM address contained in $t0 into $t2
#将 内存中地址指向为$t0的词 保存在 寄存器$t2

sw $t2, ($t0)
#store word in register $t2 into RAM at address contained in $t0
#将 寄存器$t2中的词 保存在 内存地址指向为$t0的内存中


#+偏移量
lw $t2, 4($t0)
#load word at RAM address ($t0+4) into register $t2
"4" gives offset from address in register $t0

sw $t2, -12($t0)
#store word in register $t2 into RAM at address ($t0 - 12)
#negative offsets are fine
#括号前的数字为地址偏移量，每四字节为一空间，+4即为下一个空间

#不必多说，要用到偏移量的寻址，基本上使用最多的场景无非两种：数组，栈。
#Note: based addressing is especially useful for:
#arrays; access elements as offset from base address
#stacks; easy to access elements at offset from stack pointer or frame pointer
```

例子：

```asm
.data
var1: .word 23
# declare storage for var1; initial value is 23
# 先声明一个 word 型的变量 var1 = 3;

.text
__start:
lw $t0, var1
# load contents of RAM location into register $t0: $t0 = var1
# 令寄存器 $t0 = var1 = 3;

li $t1, 5
# $t1 = 5   ("load immediate")
# 令寄存器 $t1 = 5;

sw $t1, var1
# store contents of register $t1 into RAM: var1 = $t1
# 将var1的值修改为$t1中的值： var1 = $t1 = 5;
```

#### 控制流 Control instructions

##### J指令是如何运作的

When a program is executing, its instructions are located in main memory. Each instruction has an address.

Each machine cycle executes one machine instruction. At the top of the machine cycle, the **PC** (program counter) contains the address of an instruction to fetch from memory. The instruction is fetched into the processor and is prepared for execution.

In the middle of the machine cycle the **PC** is incremented by four so that it points to the instruction that follows the one just fetched. Then the fetched instruction is executed and the cycle repeats. The machine cycle automatically executes instructions in sequence.

When a **jump** instruction executes (in the last step of the machine cycle), it puts a new address into the PC. Now the fetch at the top of the next machine cycle would fetch the instruction at that new address. Instead of executing the instruction that follows the jump instruction in memory, the processor would "jump" to an instruction somewhere else in memory.

However, it takes an extra machine cycle before the change in the PC takes effect. Before the PC changes, the instruction that follows the jump instruction in memory is fetched and executed. After that instruction executes, the next instruction to execute is the one that was jumped to. The instruction that follows a jump instruction in memory is said to be in the **branch delay** slot.

The reason for this delay is that MIPS is **pipelined.** Normally, instructions are executed one after another in sequence. In order to gain speed, the processor cleverly fetches several sequential instructions and starts working on them all. When the machine cycle calls for one of these instructions to be executed, much of the work has already been done. These instructions are in an instruction **pipe**.

This means that the instruction in the branch delay slot has mostly been completed when the jump is executed. Rather than waste this effort, the instruction in the branch delay slot is allowed to finish. Only then is the PC changed by the jump instruction.

The instruction that follows a jump instruction in memory (in the branch delay slot) is always executed. Often this is a no-op (A no-op instruction is an instruction that has no effect. A common no-op instruction is `sll $0,$0,0`.) instruction. After it executes, the next instruction to execute is the one that was the target of the jump instruction.

##### 地址是如何计算的

汇编语言通过为指令分配一个“标签”Label，使用户在不知道命令所在地址的情况下，使用标签跳跃到想要的指令处。在机器语言中，处理器需要知道所要跳跃到的指令的地址。J型指令包含了26位immediate值，用于计算label所表示的指令的地址。那么如何从26位的immediate值，计算出32位所需要的目标地址值呢？

Instructions always start on an address that is a multiple of four (they are word-aligned). So the low order two bits of a 32-bit instruction address are always "00". Shifting the 26-bit target left two places results in a 28-bit word-aligned address (the low-order two bits become "00".)

After the shift, we need to fill in the high-order four bits of the address. These four bits come from the high-order four bits in the PC. These are concatenated to the high-order end of the 28-bit address to form a 32-bit address.

For example, here is the machine language for the instruction that jumps to location `0x5B145188`. Say that the instruction is located at address `0x56767250`.

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856583375_9113.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

##### Most Jumps (and Branches) are Local

Most jumps and branches are to nearby addresses. The target of the jump instruction and the instruction following the jump instruction are likely to be close together in memory. The high-order four bits of their addresses will be identical. So the high-order four bits of the PC are the same as needed for the target address.

Of course, an assembly language programmer must be careful to make sure that this is so. When a compiler translates a source program into machine language it also must pay attention to addresses. For the tiny programs you will write for this course the high four bits of the PC will always be the high four bits of the jump address.

A jump instruction can't jump to any arbitrary location in the full 32-bit address space. It must jump to an address within the following range:

```
wxyz 0000 0000 0000 0000 0000 0000 0000
                   .
                   .
                   .
wxyz 1111 1111 1111 1111 1111 1111 1100
```

Here, **wxyz** represents the high-order four bits of the PC. Almost always the jump instruction and the jump address are both within this range.

内存被分为16个区块，这16个区块由**wxyz**四位MSB所能产生的2^4=16个值所表示。每个区块的空间大小是256Mb。一段程序被加载在内存中，而一个jump指令，是不被允许跳跃出当前程序所在的区块的，因此在*大多数情况下*，`j`指令都是可以成功跳转到label所代表的程序。

##### 如果目的地和我们不在同一区块呢

上一段提到，内存16个区块，我们当前所在指令的区块，和`j label`中的label所在区块，*通常是同一区块*。那么如果我们想jump到别的区块，那么`j`指令中的26位immediate值就不够用了，需要一个新的指令：`jr rs`：jump to register。rs是存储地址的寄存器，程序直接跳转到rs中所存储的值所代表的地址。rs中所存储的值，可能会：

- 不是4的倍数，也就是这个地址没有对齐
- 在系统权限内存区间内，超出了用户权限区间

这两种情况都会触发异常。

```asm
j	target	　　　　 
#  unconditional jump to program label target
#  不考虑任何条件，直接跳转至label指向的子程序/过程
jr	$t3		        
#  jump to address contained in $t3 ("jump register")
#  类似相对寻址，跳到该寄存器给出的地址处
```

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856584098_5175.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

分支if else  
comparison for conditional branches is built into instruction  

```asm
b	target		    #  unconditional branch to program label target
beq	$t0,$t1,target	#  branch to target if $t0 = $t1
blt	$t0,$t1,target	#  branch to target if $t0 < $t1
ble	$t0,$t1,target	#  branch to target if $t0 <= $t1
bgt	$t0,$t1,target	#  branch to target if $t0 > $t1
bge	$t0,$t1,target	#  branch to target if $t0 >= $t1
bne	$t0,$t1,target	#  branch to target if $t0 <> $t1
bgez $t0,target     #  branch to target if $t0 >= 0
bgtz $t0,target     #  branch to target if $t0 > 0
blez $t0,target     #  branch to target if $t0 <= 0
bltz $t0,target     #  branch to target if $t0 < 0  
```

##### 子程序调用 Subroutine Calls

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856584992_9422.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

At top is a sketch of what you can do with the `j` instruction. (The same could be done with the `b` instruction.) If the main routine needs to start up ("to call") a subroutine `sub` it can jump to it with a `j` instruction. At the end of the subroutine, control can be returned with another `j` instruction.

The subroutine returns to a statement in `main` labeled `ret`. The subroutine is called at just one point in `main` and it returns to an address a few instructions after that point.

The subroutine is only used once in the `main` program because it always returns to the same location. (You could write some tricky code to overcome this limitation. But it is much better to follow a subroutine linkage convention such as is about to be discussed.)

A subroutine **call** is when a main routine (or other routine) passes control to a subroutine. The main routine is said to be the CALLER and the subroutine is said to be the CALLEE. A **return** from a subroutine is when a subroutine passes control back to its CALLER. When a CALLEE finishes execution it nearly always returns control to its CALLER.

与`j`指令同样，`jal`和`jalr`都是跳跃到地址，但这两条指令会将执行子程序结束后需要返回的地址保存到R31（也就是$ra）以便与执行结束后再返回。

##### jal

The register that is used for linkage is register `$31`, which is called **`$ra`** by the extended assembler. It holds the ***return address\*** for a subroutine. The instruction that puts the return address into `$ra` is (usually) the `jal` instruction.

Register `$31` is one of the two "general purpose registers" that behave differently from the others. (The other one is register `$0`.) The `jal` instruction and register `$31` provide the hardware support necessary to elegantly implement subroutines.

To understand how `jal` works, review the machine cycle. The MIPS endlessly cycles through three basic steps. Each cycle executes one machine instruction. (This is a somewhat simplified view, but sufficient for now).

The `jal` instruction does the following in the execute phase of the machine cycle:

```
jal sub    # $ra <― PC+4  (the address 8 bytes away from the jal) 
           # PC  <― sub   load the PC with the subroutine entry point
           # a branch delay slot follows this instruction
```

**Very Tricky:** the middle step of the machine cycle has already incremented the PC by four. At this point the PC holds the address of the instruction just after the `jal` instruction. Now the execute phase of the `jal` instruction adds four to that address and puts the result in `$ra`. So now `$ra` holds the address of the second instruction after the `jal` instruction.

The correct return address is "address of the `jal` plus eight". This is because: (i) returning from the subroutine to the `jal` instruction would be a disaster (since it would execute again, sending control back to the subroutine), and (ii) the instruction following the `jal` is a branch delay slot.

![](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856585866_5260.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

Here is how the `jal` instruction works in general:

```
jal sub    # $ra <― PC+4  (the address 8 bytes away from the jal) 
           # PC  <― sub   load the PC with the subroutine entry point
           # a branch delay slot follows this instruction
```

Here is how it works in this example. The entry point of `sub` is `0x00400100`.

```
Fetch:      When the jal is fetched the PC has 0x00400014.

Increment:  The PC is incremented to 0x00400018.
            
Execute:    $ra <― 0x004001C = 0x0040018+4 
            PC  <― 0x00400100
```

The `nop` instruction in the branch delay slot is executed. Then execution continues with the first instruction of the subroutine at `0x00400100`. Control has been passed to the subroutine and the return address in `$ra`.

```asm
#subroutine call: "jump and link" instruction
jal	sub_label	
#"jump and link"
#copy program counter (return address) to register $ra (return address register)
#将当前的程序计数器保存到 $ra 中
#jump to program statement at sub_label
#并跳转至子程序sub_label
#subroutine return: "jump register" instruction
jalr $rs
#"jump and link register"
#jump to the address contained in Rs
jr	$ra	
#"jump to register"
#jump to return address in $ra (stored by jal instruction)
#通过上面保存在 $ra 中的计数器返回调用前
```

Note: return address stored in register `$ra`; if subroutine will call other subroutines, or is recursive, return address should be copied from `$ra` onto stack to preserve it, since jal always places return address in this register and hence will overwrite previous value  
如果说调用的子程序中有调用了其他子程序，如此往复， 则返回地址的标记就用 栈（stack） 来存储, 毕竟 $ra 只有一个


#### 系统指令

- 打印寄存器中的值
必须将值放在寄存器$4中，通过对$2赋值打印不同类型的数据  
例：
```mipsasm
li  $4, 12345678    #将整数值12345678放在寄存器$4中
li  $2, 1           #对$2赋值1
syscall             #将值打印在终端
```
对于不同的数据类型，赋给$2的值也不同
  Value for $2 to print|type of data
  :-:|:-:
  1|integer
  2|float
  3|double
  4|string
  11|character

- 从终端读取值，并存放在寄存器$2中
```mipsasm
li  $2, 5           #对$2赋值5
syscall             #读取终端的值，并放入$2中
```
对于不同的数据类型，赋给$2的值也不同
  Value for $2 to read|type of data
  :-:|:-:
  5|integer
  6|float
  7|double
  8|string
  12|character

  注意：对于读取字符串，需规定读取的字符串最大长度
  ```mipsasm
  read:   .space 256  #声明一个256长度的内存中buffer
          la $4,read  #将$4指向read地址
          li $5,255   #读取的字符串最大长度为255
          li $2,$8    #读取
          syscall     
  ```

- 程序退出
```mipsasm
li  $2,10
syscall
```

----
**参考资料:**  
[Central Connecticut State University QtSpim Edition, August 2015](https://chortle.ccsu.edu/AssemblyTutorial/index.html)    
[巴黎六大-索邦大学计算机软件及硬件组成原理](https://www-soc.lip6.fr/trac/sesi-almo/)  
[MIPS编程入门](https://www.cnblogs.com/thoupin/p/4018455.html)  
其他指令详询[常用mips指令](https://e-mailky.github.io/2017-09-07-mips_instruction)