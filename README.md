# ARM Keil IDE view memory in debug mode
ARM Keil集成开发环境中调试模式下查看存储器数据内容

<br />

ARM Keil开发环境用于开发C8051系MCU程序来说非常友好。其自带的Cx51编译器能充分利用C8051的硬件特性方便我们编写性能高且代码紧凑的程序出来。而Keil IDE也是制作得比较人性化，只要用过Visual Studio的开发者应该基本都能很快上手。因此这里本人将简单介绍一下Keil开发环境下如何在调试模式中查看局部变量、全局变量以及指定的存储器空间中的数据。这里笔者所使用的Keil版本比较老，是2.x版本，不过新版本在UI布局上其实也差不多，完全可以参考🤭。

<br />

### 第一步，先编写一段简单的C程序

在贴出C代码之前笔者先说明一下：笔者这里所使用的板子基于C8051F380，其硬件规格跟C8051F340差不多。笔者手头上的芯片具有256字节的内部SRAM，64KB大小的外部RAM以及64KB的Flash Memory。

C8051有三种存储空间，一种是内部SRAM，在Cx51中用`data`限定符修饰。内部SRAM速度很快，而且R0到R7这8个数据寄存器也是直接被映射到此空间的，但是它很小，有些MCU才128个字节，而有些则有256个字节。这里顺便提一下，特殊功能寄存器（SFR）尽管被映射在0x80开始的存储空间，但它与内部SRAM是相互独立的，在汇编表示中通过直接寻址与寄存器间接寻址加以区分。直接寻址获访问的是SFR的内容；而间接寻址则访问到的是内部SRAM中的数据内容。

第二种是外部RAM（XRAM），在Cx51中用`xdata`限定符修饰。外部RAM对于有些低端MCU而言是可选的，但大部分都多多少少有一些。在Ax51汇编中，专门有`MOVX`指令来读写XRAM中的数据。

第三种则是ROM，在Cx51中用`code`限定符修饰。我们可以在Cx51中将指定的全局数据放入到ROM存储空间。Ax51汇编中，专门有`MOVC`指令将数据从ROM中读出。像C8051F340、C8051F380还搭载了可编程的Flash Memory，使得软件可以在程序运行时对这块Flash Memory进行擦除写入。当我们将PSCTL（程序存储R/W控制寄存器）的PSWE(程序存储写允许)位设置为1之后，使用`MOVX`指令对指定存储地址的写将会被写入到Flash存储器中，而不是外部RAM。

下面将贴出我们的测试代码：

```c
#include <c8051F340.h>

#include <stddef.h>
#include <stdbool.h>
#include <stdint.h>
#include <intrins.h>

/// Declare sBuffer located in External RAM
/// Maximum XRAM size is 64KB
static unsigned char xdata sBuffer[1024];

/// Declare sROMBuffer located in Flash ROM
static unsigned char const code sROMBuffer[4] = { 0x20, 0x30, 0x40, 0x50 };

/// Declare sDataBuffer located in Internal SRAM
static unsigned char data sDataBuffer[4] = { 0x60, 0x70, 0x80, 0x90 };

int main(void)
{
    unsigned char i;

    // The first thing in C8051 program is to disable watchdog timer by clearing WDTE bit in PCA0MD
    PCA0MD &= ~0x40;

    for(i = 0; i < 250; i++)
        sBuffer[i] = i + 1;

    sBuffer[100] = sROMBuffer[0] + sDataBuffer[0];
    sBuffer[101] = sROMBuffer[3] + sDataBuffer[3];
    
    _nop_();
    
    while(1)
    {
        _nop_();
    }
}
    
```

完成之后，我们可以在第一次出现的“_nop_();”这条语句处设置断点。

<br />

### 查看局部变量与全局变量

我们看下面这张图。图中，右上方用蓝色方框框出来的图标是调试会话按钮。点击此按钮之后我们就开始调试了，当然在你第一次调试之前需要做一些设置，各位可以参考其他文章做Keil对C8051芯片的硬件调试设置。左上方蓝色圈圈圈出来的图标是运行按钮，当程序执行到某一断点之后，若要让它继续运行，则点击此按钮。

图中用红色圆圈圈出来的图标就是Watch and Call Stack Window按钮。点击此按钮之后，编辑窗口的下方将会显示出用于查看局部变量以及全局变量的调试窗口。在截图中，我们可以看到这里显示的是局部变量“i”，它当前的值为0。所以如果我们要查看当前函数的局部变量的话，使用此窗口的“Locals”选项卡进行查看。

![debug1.png](https://github.com/zenny-chen/ARM-Keil-IDE-view-memory-in-debug-mode/blob/master/debug1.PNG)

如果我们想要查看全局变量，那么需要点击下方“Watch and Call Stack Window”窗口中的“Watch #”选项卡。watch选项卡提供了两个，如果我们要观察的全局变量较多的话可以分两组，分别在这两个watch选项卡中查看。我们选中“Type F2 to edit”文本框，然后点击F2功能键即可编辑了。输入想要观察的全局变量名即能查看其详细的数据内容了，如下图所示。点击变量名左边的加号“+”按钮，即可展开该变量的数据内容。

![debug2.png](https://github.com/zenny-chen/ARM-Keil-IDE-view-memory-in-debug-mode/blob/master/debug2.PNG)

这里需要再说明一下的是，如果当前所观察的变量未展开，那么在右边“Value”一栏中所显示的是该变量所在的地址。其中，“X”前缀表示该变量位于外部RAM中（对应于`xdata`）；“D”前缀表示该变量位于内部SRAM（对应于`data`）；“C”前缀表示该变量位于ROM存储空间（对应于`code`）。这个规则将用于下面所提到的对指定位置的存储器查看数据内容的情况。

<br />

### 查看指定存储器位置的数据内容

