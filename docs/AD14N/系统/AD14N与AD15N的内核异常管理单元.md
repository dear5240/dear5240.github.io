# 6. AD14N与AD15N的内核异常管理单元[](https://doc.zh-jieli.com/AD14/zh-cn/master/emu/emu_ad14n_ad15n.html#ad14nad15n)

异常管理单元开启后，会在MCU异常时触发中断，并复位或者打印出异常信息并断言;

1.调用函数 `emu_init` 后异常开始工作。

2.异常断言

```
//内核异常打印,为1时会断言，开发调试时可改为1，量产时必须为0
const u8 config_asser = 0;
```

> 备注
> 
> **config_asser = 1** 时，异常触发后断言，并在DEBUG打印控制开启的情况下会打印出异常信息；
> 
> **config_asser = 0** 时，异常触发后芯片复位。
> 
> **量产时，请将config_asser = 0，否则死机后不能复位。**

3.异常断言打印

 ```
 const char log_tag_const_i_DEBUG AT(.LOG_TAG_CONST) = CONFIG_DEBUG_LIBS(1);
 const char log_tag_const_d_DEBUG AT(.LOG_TAG_CONST) = CONFIG_DEBUG_LIBS(1);
 const char log_tag_const_e_DEBUG AT(.LOG_TAG_CONST) = CONFIG_DEBUG_LIBS(1);
 const char log_tag_const_c_DEBUG AT(.LOG_TAG_CONST) = CONFIG_DEBUG_LIBS(1);上面四行代码为异常断言打印信息的开关，默认时开启状态。
 ```

## 6.1. 内核异常示例（非对齐访问）：

在 `config_asser = 1` 并开启了断言打印的情况下，触发异常有会有打印信息，以下以非对齐访问为例解释。

非对齐访问是指CPU尝试从非其数据类型自然边界（如4字节数据不从4的倍数地址开始）的内存地址读取或写入数据，这通常会导致硬件异常或性能损失。

```
 1  Error> [DEBUG]
 2  -----Err: function:exception_analyze() @line:87
 3
 4  <Error> [DEBUG]stack_magic[0]:0x5a5a5a5a
 5  <Error> [DEBUG]stack_magic0[0]:0x5a5a5a5a
 6  <Error> [DEBUG]reti 0x00112434
 7  <Error> [DEBUG]rete 0x00000000
 8  <Error> [DEBUG]rets 0x00112402
 9  <Error> [DEBUG]psr  0x00000000
10  <Error> [DEBUG]ssp  0x00002520
11  <Error> [DEBUG]icfg 0x07010a80
12  <Error> [DEBUG]usp  0x00000800
13  <Error> [DEBUG] sp  0x00001608
14
15  <Error> [DEBUG]---reti---
16  40 AC 11 EA FF FF 40 D4 F8 71 08 80 00 ED 71 AC
17  08 78 20 E8 D1 F8 20 E4 02 92 60 E7 21 00 48 D0
18  60 E8 D0 F8 20 E8 D0 F8 00 80 60 E8 D0 F8 FE E1
19  3F A2 FF 0F 18 2F 21 C6 D8 E1 6F 5C 3F 0F 24 D1
20
21  <Error> [DEBUG]---rets---
22  CF 5B 38 C6 64 E8 06 32 31 80 C1 77 60 E8 C4 1E
23  28 C6 81 C6 03 E1 B0 37 20 E4 00 9D D8 E1 DF 5E
24  20 E8 B0 03 20 E8 B1 01 0F ED 12 FF 52 AC BA 75
25  40 D4 40 AC 11 EA FF FF 40 D4 F8 71 08 80 00 ED
26
27  <Error> [DEBUG]EMU_CON:0x00008007
28  <Error> [DEBUG]EMU_MSG:0x00000001
29  <Error> [DEBUG]------EMUerr msg0:misalign_err-----
30
31  <Error> [DEBUG]r0:0x000022c6
32  <Error> [DEBUG]r1:0x000022c5
33  <Error> [DEBUG]r2:0x00001d10
34  <Error> [DEBUG]r3:0x00000001
35  <Error> [DEBUG]r4:0x000020e4
36  <Error> [DEBUG]r5:0x003fa000
37  <Error> [DEBUG]r6:0x000022c4
38  <Error> [DEBUG]r7:0x00002020
39  <Error> [DEBUG]r8:0x00006000
40  <Error> [DEBUG]r9:0x000021e0
41  <Error> [DEBUG]r10:0x00001678
42  <Error> [DEBUG]r11:0x00007e8c
43  <Error> [DEBUG]r12:0x00002210
44  <Error> [DEBUG]r13:0x00091420
45  <Error> [DEBUG]r14:0x00000001
```

### 6.1.1. 第1，2行

 ```
 1  Error> [DEBUG]
 2  -----Err: function:exception_analyze() @line:87
 ```

 这两行打印表示进入了异常断言。



### 6.1.2. 第4，5行 堆栈边界

 ```
 4  <Error> [DEBUG]stack_magic[0]:0x5a5a5a5a
 5  <Error> [DEBUG]stack_magic0[0]:0x5a5a5a5a
 ```

 这两行打印堆栈边界信息，边界信息不等于 `0x5a5a5a5a` 时堆栈已溢出。

> 备注
> 
> stack_magic != 0x5a5a5a5a时，爆栈(Stack Overflow)
> 
> 这是最常见的情况。当程序调用层次过深、局部变量过大或存在无限递归时，栈顶（Stack Pointer, SP） 会不断向低地址方向增长（对于向下生长的栈），最终超越为栈分配的栈底（Stack Base/Limit） 边界，侵入到其他内存区域（如全局数据区或堆区）。
> 
> stack_magic0 != 0x5a5a5a5a时，爆栈(Stack Underflow)
> 
> 这种情况相对少见，但更危险。通常发生在函数返回（POP操作）或调整栈指针时出现了严重错误（如错误的汇编指令、缓冲区下溢写入破坏了返回地址和栈帧后，又执行了多次返回）。

### 6.1.3. 第6~13行 CPU瞬态

```
06  <Error> [DEBUG]reti 0x00112434
07  <Error> [DEBUG]rete 0x00000000
08  <Error> [DEBUG]rets 0x00112402
09  <Error> [DEBUG]psr  0x00000000
10  <Error> [DEBUG]ssp  0x00002520
11  <Error> [DEBUG]icfg 0x07010a80
12  <Error> [DEBUG]usp  0x00000800
13  <Error> [DEBUG] sp  0x00001608
```

这几行显示的是一个处理器在异常或中断发生时的关键上下文寄存器。它们共同记录了CPU被打断那一瞬间的状态，是调试异常、崩溃和中断的核心信息。

> 备注
> 
> 在分析函数调用关系时，需从汇编层面入手， **因为C语言的函数在编译链接后可能被内联优化** ，导致源码层面的调用链不完整。
> 
> 从汇编视角解析寄存器记录的逻辑：
> 
> \1. 当函数 A 调用函数 B 时，进入 B 函数后， **rets** 寄存器中保存的是 A 函数中“调用指令”的下一条指令地址。此地址是 B 函数执行完毕后需要返回的位置。
> 
> \2. 若 B 函数在运行过程中触发异常（如非对齐访问），CPU 在跳转至异常中断处理程序后， **reti** 寄存器中保存的是 B 函数中触发异常的那条指令的下一条指令地址。此地址是异常处理结束后理论上应返回的位置（但通常因错误无法直接返回）。

1.reti (0x00112434) - 中断/异常返回地址

 作用：记录了被中断的程序下一条即将执行的指令地址（PC值）。

分析：当CPU正在执行 0x00112434 这条指令时（或刚执行完上一条），被异常/中断打断。处理完异常后，CPU会跳回这个地址继续执行。

调试价值：这是最重要的寄存器之一。通过反汇编这个地址附近的代码，可以精确知道是哪条指令（或前一条指令）触发了异常。

2.rete (0x00000000) - 异常返回地址（可能）

 作用：在某些架构中用于记录嵌套异常或特定类型异常（如精确的指令异常）的返回地址。值为0可能表示当前是最外层异常，或该寄存器未使用。

分析：如果 reti 和 rets 都存在，rete 有时指向导致异常（如 misalign_err）的那条故障指令本身的地址。

3.rets (0x00112402) - 子程序/状态返回地址

作用：可能记录了发生异常时，当前函数的调用者（父函数）的返回地址（即 LR 链接寄存器的值），或者是CPU在特定模式下的备用返回地址。

分析：0x00112402 这个地址很接近 reti (0x00112434)，相差仅 0x32字节。这表明异常很可能发生在某个刚被调用不久的函数内部。结合这两个地址可以画出函数调用链。

4.psr (0x00000000) - 程序状态寄存器

作用：包含ALU状态标志（如零标志Z、进位C、负数N、溢出V），以及当前处理器模式和中断开关状态。

分析：值为0非常可疑。通常至少会有某些状态位被置位（例如Thumb状态位，在Cortex-M中必须为1）。全0可能意味着：

 寄存器被意外清零（内存破坏？）。

异常发生在CPU复位后的最初状态。

这是某个特定模式下的备份PSR，其值在进入异常时被保存到了其他寄存器。

5.sp (0x00001608) - 当前堆栈指针

作用：最关键的寄存器之一。这是异常发生瞬间的栈指针值。所有局部变量、函数调用帧都保存在这个地址指向的内存区域。

 分析：

 它是异常现场的“快照”。

其值（0x1608）位于 ssp(0x2520) 和 usp(0x800) 之间。

### 6.1.4. 第15~25行

```
15  <Error> [DEBUG]---reti---
16  40 AC 11 EA FF FF 40 D4 F8 71 08 80 00 ED 71 AC
17  08 78 20 E8 D1 F8 20 E4 02 92 60 E7 21 00 48 D0
18  60 E8 D0 F8 20 E8 D0 F8 00 80 60 E8 D0 F8 FE E1
19  3F A2 FF 0F 18 2F 21 C6 D8 E1 6F 5C 3F 0F 24 D1
20
21  <Error> [DEBUG]---rets---
22  CF 5B 38 C6 64 E8 06 32 31 80 C1 77 60 E8 C4 1E
23  28 C6 81 C6 03 E1 B0 37 20 E4 00 9D D8 E1 DF 5E
24  20 E8 B0 03 20 E8 B1 01 0F ED 12 FF 52 AC BA 75
25  40 D4 40 AC 11 EA FF FF 40 D4 F8 71 08 80 00 ED
```

这几行信息是reti与rets所在位置附件的机器码。

### 6.1.5. 第27~28行

EMU_CON：异常控制寄存器，bit2为1时，使能除0异常。

```
27  <Error> [DEBUG]EMU_CON:0x00008007
28  <Error> [DEBUG]EMU_MSG:0x00000001
```

EMU_MSG 内核异常信息:

```
bit0 = 1时，非对齐访问异常
bit1 = 1时，非法指令异常（cpu取指出现非法指令）
bit2 = 1时，除0异常
```

### 6.1.6. 第29行 异常信息种类

```
29  <Error> [DEBUG]------EMUerr msg0:misalign_err-----
```

信息显示错误类型， `misalign_err` 为非对齐访问。

> 备注
>
> 异常信息种类
>
> misalign_err ： 为非对齐访问
>
> illeg_err ： 非法指令异常
>
> div0_err ： 除0异常

### 6.1.7. 第31~45行 通用寄存器信息

```
31  <Error> [DEBUG]r0:0x000022c6
32  <Error> [DEBUG]r1:0x000022c5
33  <Error> [DEBUG]r2:0x00001d10
34  <Error> [DEBUG]r3:0x00000001
35  <Error> [DEBUG]r4:0x000020e4
36  <Error> [DEBUG]r5:0x003fa000
37  <Error> [DEBUG]r6:0x000022c4
38  <Error> [DEBUG]r7:0x00002020
39  <Error> [DEBUG]r8:0x00006000
40  <Error> [DEBUG]r9:0x000021e0
41  <Error> [DEBUG]r10:0x00001678
42  <Error> [DEBUG]r11:0x00007e8c
43  <Error> [DEBUG]r12:0x00002210
44  <Error> [DEBUG]r13:0x00091420
45  <Error> [DEBUG]r14:0x00000001
```

这些是发生异常时，CPU通用寄存器在那一刻的“快照”。它们是定位软件错误的最关键线索，相当于程序的“犯罪现场指纹”。













