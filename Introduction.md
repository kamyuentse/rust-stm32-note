# 简介

本手册将对以下的问题进行讨论：

1. 编写、编译、烧写以及调试 Rust 嵌入式程序；
2. 访问通用外设：GPIO、ADC、PWM、I2C、SPI等；
3. 实时系统的进程调度；

即使你没有 STM32 开发经验因为无须担心，涉及的相关内容将会在需要使用时提及。

在阅读本手册时，推荐你准备下面的东西：

1. STM32F4 系列开发板
2. 对应芯片的参考手册

## Cortex-M 微处理器基础

### 启动过程分析

系统从上电启动称作上电复位。在复位以后以及处理器开始执行程序前，会从存储器中取出头两个字，然后将这两个值分别设置为主栈指针以及程序计数器的初始值，这个流程如下图所示：![](/assets/Fig1-复位流程.png)地址空间表示的内容取决于启动模式，Cortex-M 系列处理器可以通过配置 BOOT\[1:0\] ，选择下面 3 中启动模式的一种

| BOOT1 | BOOT0 | 启动区域 | 说明 |
| :--- | :--- | :--- | :--- |
| X | 0 | 主闪存存储器 | 主闪存存储器为启动区域 |
| 0 | 1 | 系统存储器 | 系统存储器为启动区域 |
| 1 | 1 | 内置SRAM | 内置SRAM为启动区域 |

在不同的启动模式下，处理器将会进行地址空间的映射。TODO：以 STM32-F446RE 为例，不同启动模式下的地址空间映射图如下：

作为例子，接下来对一个使用 mbed 库编写的简单程序进行分析：

##### ELF 文件头信息

```
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x8001f49
  Start of program headers:          52 (bytes into file)
  Start of section headers:          166868 (bytes into file)
  Flags:                             0x5000200, Version5 EABI, soft-float ABI
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         3
  Size of section headers:           40 (bytes)
  Number of section headers:         12
  Section header string table index: 11
```

##### 部分反汇编内容

```
BUILD/mbed-os-example-blinky.elf:     file format elf32-littlearm


Disassembly of section .text:

08000000 <g_pfnVectors>:
 8000000:    20020000     andcs    r0, r2, r0
 8000004:    08001f49     stmdaeq    r0, {r0, r3, r6, r8, r9, sl, fp, ip}

 ...

 08001f48 <Reset_Handler>:
 8001f48:    f8df d020     ldr.w    sp, [pc, #32]    ; 8001f6c <LoopCopyDataInit+0x14>
 8001f4c:    2100          movs    r1, #0
 8001f4e:    e003          b.n    8001f58 <LoopCopyDataInit>

 ...
```

从上面可以看到，中断向量表存放在存储器开始的位置，并且向量表中第一个地址为主栈指针（一般设置为RAM的顶部地址），第二个位复位中断处理函数的入口地址，注意到这个位置的值为 0x08001f49，而 Reset\_Handler 的地址为 0x08001f48，这是由于在 Cortex-M 处理器中，向量表中的向量地址最低位应该为1，以表示它们为 Thumb 指令代码。

下一章节中，将会讨论如在

### 系统异常及中断

中断几乎是所有微处理器的一种特性，在 Cortex-M 系列处理器中，都提供了一个用于中断处理的嵌套向量中断控制器（NVIC）。除了中断请求，还有其他服务事件，称为“异常”。

##### 系统异常列表

| 位置 | 优先级 | 优先级类型 | 缩写 | 说明 | 地址 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| \ | \ | \ | \ | 保留 | 0x0000 0000 |
|  | -3 | 固定 | Reset | 复位 | 0x0000 0004 |
|  | -2 | 固定 | NMI | 不可屏蔽中断（外部NMI输入） | 0x0000 0008 |
|  | -1 | 固定 | HardFault | 所有的错误都可能引发该中断（相应的错误处理未使能） | 0x0000 000C |
|  | 0 | 可设置 | MemManage | 存储器管理错误，存储器管理单元冲突或访问非法位置 | 0x0000 0010 |
|  | 1 | 可设置 | BusFault | 总线错误。当高级性能总线接口收到从总线的错误响应时产生（若为取指也被称作取指z终止，数据访问则为数据终止） | 0x0000 0014 |
|  | 2 | 可设置 | UsageFault | 程序错误或试图访问协处理器导致的错误（Cortex-M3不支持协处理器） | 0x0000 0018 |
| \ | \ | \ | \ | 保留 | 0x0000 001C |
| \ | \ | \ | \ | 保留 | 0x0000 0020 |
| \ | \ | \ | \ | 保留 | 0x0000 0024 |
| \ | \ | \ | \ | 保留 | 0x0000 0028 |
| \ | 3 | 可设置 | SVCall | 请求管理调用，一般用于 OS 环境且允许应用任务访问系统服务。 | 0x0000 002C |
|  | 4 | 可设置 | Debug Monitor | 调试监控器。在使用基于软件的调试方案时，断点和监视点等调试事件的异常 | 0x0000 0030 |
| \ | \ | \ | \ | 保留 | 0x0000 0034 |
|  | 5 | 可设置 | PendSV | 可挂起的服务调用，一般使用该异常进行上下文切换。 | 0x0000 0038 |
|  | 6 | 可设置 | SysTick | 系统节拍定时器。 | 0x000 0003C |

备注：

1. 上面向量表的地址使用了默认地址了，从 0x00000004 开始。并且包含了地址 0x00000000，该地址是MSP（主栈指针）的初始值；
2. 起始地址 0x00000000 处应为启动储存器，根据启动模式的不同，处理器会将相应存储器（Flash、ROM等）的地址空间映射到该位置；
3. 向量表中存放的是相应中断处理函数的入口地址（4 Byte）；

### 链接器脚本

## 环境配置

#### 安装 Rust

```
curl https://sh.rustup.rs -sSf | sh
```

#### 安装 ARM-GCC

```
brew cask install gcc-arm-embedded openocd qemu
```

## 第一个程序

在编写第一个程序之前，需要对下面的内容进行讲解。

1. 程序从源代码编译为可运行代码经过的过程，编译——链接
2. Rust 如何指定链接器
3. 链接器脚本的编写；

在 Rust 中，可以通过 cargo 配置来指定链接时使用的链接器，当前我们仍然需要使用 gcc-ld 来进行链接工作。

链接器所进行的工作是将

