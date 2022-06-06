---
title: 单核心处理器，总线架构及外围设备
summary: MIPS - 本篇文章我们将学习计算机物理，寻址空间及软件组成原理，探究处理器，内存以及外围设备间的信息沟通。
date: 2019-11-17
tags: ['System']
---

本篇文章我们将学习计算机物理，寻址空间及软件组成原理，探究处理器，内存以及外围设备间的信息沟通。如下图所示，我们所研究的内容包含：

- 一枚MIPS32架构的处理器，包含一级缓存L1用来缓存数据和指令
- 二级缓存L2，用来与外部内存RAM链接
- 只读存储器ROM，用来存放机器启动时的代码
- 用来控制终端读写的外围设备（即TTY屏幕和键盘）
- 一根总线，连接一个主设备和三个目标设备

![archi.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856529298_8845.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

![almo.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856529951_2209.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

系统在处理指令时，使用小型的开发系统GIET来处理异常（法语：Gestionnaire d’Interruptions, Exceptions et Trappes）（翻译：GIET : Interruption / Exception / Trap Handler for MIPS32 processor）。

# TTY

从历史上看，终端刚开始就是终端机，配有打印机，键盘，带有一个串口，通过串口传送数据到主机端，然后主机处理完交给终端打印出来。电传打字机(teletype)可以被看作是这类设备的统称，因此终端也被简称为 TTY(teletype 的缩写)。  
后来的终端慢慢演变成了键盘 + 显示器。如果我们要把内容输出到显示器，只要把这些内容写入到显示器对应的 TTY 设备就可以了，然后由 TTY 层负责匹配合适的驱动完成输出。  
每个外围设备所对应的寄存器的地址会映射在内存中。
每个终端与四个寄存器相连

- `TTY_WRITE`：将一个ascii字符显示在终端的屏幕上，处理器需要使用`sb`指令将一个byte写入`TTY_WRITE`寄存器的地址
- `TTY_READ`：从键盘读取一个ascii字符，处理器需要使用`lb`指令将一个byte写入`TTY_READ`寄存器的地址
- `TTY_STATUS`：判断`TTY_READ`中有无字符，处理器需要从`TTY_STATUS`读取一个长度为32bits的词
  - 若这个词第0位不为0，则寄存器`TTY_READ`不为空
  - 若这个词第1位不为0，则寄存器`TTY_WRITE`不为空
- `TTY_CONFIG`：软件可以将外围设备的信息写入在这个寄存器内

若`TTY_WRITE`收到读取信息的指令，或者`TTY_READ`/`TTY_STATUS`收到写入信息的指令，则TTY控制器会返回错误代码，并在处理器端触发一个异常（exception）：错误的程序将停止运行（interruption），处理器将会跳转至异常处理系统以触发异常*Data BUS Error*。  
TTY控制器能够作用于多个终端，而每个终端都具有4种TTY的控制权。每个TTY地址长度为4字节，因此每个终端也就占据4*4=16字节的地址长度。

![périphériques cibles.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856530949_9006.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

下图记录了两个终端持有8个TTY寄存器的32bits地址段，其中

- 第2，3位记录终端
- 第4，5位记录4个寄存器中的哪一个

![tty_base.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856531716_1611.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

当MIPS做出load或store指令时，所有的组件都会受到目标地址。每个组件在虚拟地址映射所属的地址段（segment de l'espace d'adressage）：若映射到的地址属于此组件管理，则此组件处理此请求；否则则不做任何事。

# MIPS虚拟地址内存映射

![espace adressage.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856532602_164.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

用户模式地址区：

- `seg_code`：用户程序（在`main.c`里定义）以及一些用户模式下
所使用的外围设备的系统服务函数（在`stdio.c`中）。对应的地址为`seg_code_base = 0x00400000`
- `seg_data`：用户程序使用的全局数据，对应的地址是`seg_data_base = 0x10000000`
- `seg_stack`：程序所使用的栈，对应的地址是
`seg_stack_base = 0x20000000`  

内核模式地址区：

- `seg_reset`：在ROM中。机器启动代码，对应的地址是`seg_reset_base = 0xBFC00000`，代码定义在文件`reset.s`
- `seg_kcode`：受操作系统保护的代码区。对应的地址是`seg_kcode_base = 0x80000000`，主要包含了**GIET**代码
- `seg_kdata`：受操作系统保护的数据，对应的地址`seg_kdata_base = 0x81000000`
- `seg_kunc`：操作系统中**不可缓存**的数据，对应的地址是`seg_kunc_base = 0x82000000`
- `seg_tty`：包含了四个TTY，对应的地址为`seg_tty_base = 0x90000000`  

The base address of the segment containing this code MUST be `0x80000000` , in order to have the entry point at address `0x80000180` !  
GIET系统地址起始点为`0x80000000`，其中进入点为`0x80000180`。多出来的`0x180`为`.space 0x180`所定义的384字节的空间，这一段空间留空。（详见[giet.s](https://www-soc.lip6.fr/trac/sesi-almo/chrome/site/docs/ALMO-giet-src-code.pdf)源代码）  

GIET代码启动时存储在RAM中因为其代码属于操作系统一部分，在boot阶段运行。GIET数据存储在`seg_kdata`和`seg_kunc`中。

# 操作系统

1. 管理硬件及应用资源
    - 对硬件的调用：终端，硬盘，网络等
    - 内存分配
    - 处理器资源分配
2. 管理外围设备事件
    - 结束对硬件的调用
    - 处理周期性操作
3. 处理程序异常
    - 以0做除数
    - 非法指令
    - 分页错误，溢出等

总结：操作系统应保证：

- 服务质量（Quality of Service）：公平，优先权，期限
- 应用安全：机密性，整体性
- 硬件使用可靠性

## 软件架构

![archi logicielle.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856533548_6599.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

- 上部：应用层，可以直接通向某些地址段，但若要接触硬件层（下部matériel）则需要通过系统代码库（biblithèques système）
- 系统代码库（biblithèques système）：定义了需要使用的API。这些功能可在用户模式下运行，通过调用系统内核服务来进行（`syscall`）。服务通过`eret`（异常返回）返回给用户信息。
- 中部：系统内核，他接收来自应用层的服务请求（`syscall`）
- 下部：硬件，通过各种load/store指令写入数据

## 具体应用

一个程序应该至少有一个`main()`函数用来启动，他的代码，数据以及堆栈都存储在内存中只要这个程序是可以在用户模式下运行的（也就是地址小于`0x80000000`）。  
当应用需要访问外围设备时，他需要向系统内核通过`syscall`发出请求，通过系统代码库中的某个API来实现。  

```
$4 - $7 ⇐ 系统服务参数
$2 ⇐ 系统服务代号
syscall ≃ jal kernel
$2 ⇒ 返回错误代码，或0表示成功
```

## 实例

```c
#include <stdio.h>
__attribute__((constructor)) void main(void)
{
    char byte;
    char str[] = "\nHello World!\n";
    while (1) {
        tty_puts(str);
        tty_getc(&byte);
        if (byte == 'q') {
            exit(0);
        }
    }
    exit(0);
}
```

这段程序为一个死循环，在每次循环中在屏幕上显示Hello World（`tty_puts(str);`），并且从键盘读入一个字符（`tty_getc(&byte);`），当读取到的字符为q时，退出循环，程序结束。

输出信息的步骤：

1. 启动时，处理器跳转到`0xBFC00000`并运行存储在`seg_reset`的启动代码
2. 当处理器收到eret返回值时（即机器启动结束），开始运行`main()`函数的第一行代码（存储在`seg_code`）中
3. 当处理器运行至`tty_puts(str);`时（存储在`seg_code`），跳转到其所在的地址段
4. 处理器运行在`tty_puts()`中的`syscall`跳转至`0x80000180(seg_kcode)`
5. **GIET**开始处理第一次跳转，调用他本身的系统函数`__tty_write()`，以在屏幕上输出一行长度已知的字符串。这个函数从`seg_data`中依次读取字符，并将其写入`seg_tty`中。（即在寄存器`TTY_WRITE`中）
6. `__tty_write()`函数返回至`tty_puts()`再返回至`main()`

# SR (Status Register $12)

![SR.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856534244_6830.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

1. 第15 - 第8位：IM[7:0] Interrupt Mask即IRQ。高六位为Hardware，低两位为Software
2. UM：User Mode 若为1，则为USER模式
3. ERL：Error Level
4. EXL：Exception Level
5. IE：Interrupt Level 若为1，则存在中断

若ERL+EXL=1，则在KERNEL模式，IRQ非激活态

典型值：

- 0xFF11：用户模式
- 0xFF13：在内核
- 0xFF01：syscall
- 0xFF00：临界区段

# CR (Cause Register $13)

![CR.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856535142_1169.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

仅需要注意XCODE数段，4位16个可能至代表系统进入内核模式的原因

# 进入内核的步骤

## syscall

1. `EPC <- PC` : 保存syscall的地址
2. `SR[EXL] = 1`
3. `CR[XCODE] = 8` 即1000 SYS，为进入的原因
4. `PC = 0x80000180`

## Exception异常

1. `EPC <-PC`: 保存syscall的地址
2. `SR[EXL] = 1`
3. `CR[XCODE] = 异常原因`
4. `PC = 0x80000180`

# 从内核返回用户应用

1. `SR[EXL] = 0`
2. `PC = EPC`

# GIET构成

> GIET = "Gestionnaire d'Interruptions, Exceptions et Trappes" or Exception Handling

## E - Exception异常

由于程序出错而导致的系统异常，例如除数为0，地址未对齐。异常通常导致程序错误地终止。异常管理器应当诊断出异常的类型，并显示异常信息以帮助程序员修改程序。异常管理器在文件`exc_handler.c`中

## I - Interruption中断

来自于硬件，在软件层面上，中断是不可预知的，并随时有可能发生。

   > Les interruptions provenant de périphériques sont imprévisibles du point de vue du logiciel en cours d'exécution sur le processeur, et peuvent donc survenir à tout instant.

   若一个中断没有被隐藏，则会触发中断处理程序(ISR : [Interrupt Service Routine](https://en.wikipedia.org/wiki/Interrupt_handler) )用于处理中断，硬件可从处理器“偷窃”数个时钟周期(cycle)以做处理。在处理中断结束后，被中断的应用则会恢复运行。终端管理器会通过访问中断集中器(ICU : Interrupt Concentrator Unity)应诊断中断的来源，以便调用对应的ISR。`MIPS32`处理器拥有6个中断接口，但仅有处理器接口 0° 被使用，因此应当系统性地访问ICU。中断处理程序在文件`irq_handler.c`中。

   > L'ICU est un multi concentrateur de signaux d'IRQ. Chaque IRQ peut être masquée. Dans le cas d'une architecture multi cores, il permet de router l'IRQ vers n'importe quel core. C'est un périphérique cible contrôlé par des accès en lecture/écriture en registres.

## T - Trappe风道(系统调用)

由用户模式的应用软件向系统内核发送的服务调用请求，以`syscall`的方式发出，例如在TTY终端读写或是读写硬盘。通过`syscall`处理的方式为

- 从`$2`读取服务号

- 从`$4,$5,$6或$7`读取服务参数

- 调用系统服务

GIET可在需要时，使用栈来存储不同应用程序的所需调用的系统服务。这一程序在文件`sys_handler.c`中

# 硬件构成

## TTY

## TIMER

![TIMER.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856535911_4382.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

TIMER包含了一个计时器，可定时触发一个中断，通过读写寄存器的方式设置。

- TIMER_VALUE : 读/写，每个cycle值+1

- TIMER_PERIOD : 只写。两次IRQ之间的周期

- TIMER_MODE : 运行模式，两位二进制数

  - 第0位：若为1则TIMER正在运行，为0则TIMER已停止
  - 第1位：若为1，则当TIMER计时器自减至0时，无IRQ中断

- TIMER_RESTIRQ：只写，在此地址写入以处理IRQ

  在`stido.c`中有TIMER源代码，其中部分函数用法如下

  - `timer_set_period()`:

  This function defines the period value of a timer to enable a periodic interrupt.

  Returns 0 if success , > 0 if error .

  - `timer_set_mode (int mode)`:

    This function defines the operation mode of a timer . The possible values forthis mode are:

    - 0x0 : Timer not activated

    - 0x1 : Timer activated , but no interrupt is generated

  - 0x3 : Timer activarted and periodic interrupts generated.

  Returns 0 if success , > 0 if error.

  ```c
  timer_set_period(5000000);
  timer_set_mode(3);
  ```

  若将这两个函数放入代码中，则每隔5,000,000cycles，屏幕会输出TIMER中断信息" Interrupt timer received at cycle : "+当前处理器经过的cycles数

## ICU

ICU : Interrupt Concentrator Unity中断集中器。每个IRQ都可能会被隐藏，在多核心架构中，它将不同的IRQ导向不同的核心。

![QUUueU.md.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856536753_8619.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

ICU_INT: IRQ的连接

![QUak7D.jpg](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856537702_890.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

可以看到，ICU可管理32个不同来源的IRQ

在`seg_id`中：

```
seg_icu_base = 0x9F000000; /* ICU device */
```

在`reset.s`中，初始化`_interrupt_vector`，并将TIMER和TTY连接到ICU上

(已知TIMER连接至`IN_IRQ[1]`, TTY连接至`IN_IRQ[3]`)

```asm
/* initializes interrupt vector */
la $26, _interrupt_vector
/* $26 <= interrupt_vector address */
la $27, _isr_timer
/* $27 <= isr_timer address */
sw $27, 4($26)
/* interrupt_vector[1] <= _isr_timer */
la $27, _isr_tty_get_task0
/* $27 <= isr_tty_get_task0 address */
sw $27, 3*4($26)
/* interrupt_vector[3] <= _isr_tty_get_task0 */
```

初始化ICU

```asm
la $26, seg_icu_base
/* initializes ICU */
li $27, 0xA
/* IRQ_IN[1] & IRQ_IN[3] enabled */
sw $27, 2*4($26)
/* ICU_MASK_SET = 0xA */
```
