# Chapter 5

# 中断和设备驱动

一个**驱动程序**是操作系统中管理特定设备的代码：它配置设备硬件，告诉设备执行操作，处理结果中的中断，并与可能正在等待来自设备的I/O的进程进行交互。驱动程序的编写难度很大，因为它需与它所管理的设备并发执行。此外，驱动程序必须理解设备的硬件接口，这可能非常复杂且文档不良好。

通常，需要操作系统关注的设备可以配置为生成中断信号，这是一种trap。内核trap处理代码在设备触发中断时调用驱动程序的中断处理程序。

许多设备驱动程序执行两种代码：在进程的内核线程中运行的`top half`和在中断时间执行的`bottom half`。top half是通过系统调用如`read`和`write`调用的，这些调用需要设备执行I/O。该代码可能会请求硬件启动操作（如，请求磁盘读取一个块）；然后代码等待操作完成。最终设备完成操作并触发中断。驱动程序的中断处理程序作为bottom half，找出哪个操作已经完成，适当时唤醒等待的进程，并告诉硬件开始处理任何等待的下一个操作。

## 代码:控制台输入
控制台驱动程序(kernel/console.c)是驱动程序结构的一个简单示例。控制台驱动程序接受由连接在RISC-V上的串口硬件`UART`接收到人类键入的字符，逐行处理输入中的特殊字符，例如回退和ctrl+u。用户进程（例如shell进程）使用`read`系统调用来从控制台获取输入行。当您在QEMU中输入xv6时，您的按键通过QEMU的模拟UART硬件传递给xv6。

驱动程序通信的UART硬件是QEMU模拟的16550芯片。在真实的计算机上，16550将管理连接到终端或其他计算机的RS232串行连接。在运行QEMU时，它连接到您的键盘和显示器。

UART硬件对软件来说是一组*内存映射*的控制寄存器。也就是说，有一些物理地址RISC-V硬件连接到UART设备，使得加载和存储与设备硬件而非RAM交互。UART的内存映射地址从0x10000000开始，或称为`UART0`。有几个UART控制寄存器，每个都是一个字节宽。它们从`UART0`的偏移量定义在`LSR`寄存器中包含位，指示输入字符是否等待软件读取。如果有字符正在等待，则可以从`RHR`寄存器中读取这些字符。每次读取一个字符时，UART硬件会将其从正在等待的字符的内部FIFO中删除，并在FIFO为空时清除LSR中的“就绪”位。UART发送硬件大部分独立于接收硬件；如果软件向`THR`写入一个字节，则UART会将该字节发送出去。

Xv6的`main`调用`consoleinit`来初始化控制台驱动程序。该代码配置UART以在接收每个输入字节时生成接收中断，并且在每个字节发送完成时生成一个**传输完成**中断。

xv6 shell通过文件描述符从控制台中读取，`read`系统调用通过内核使它们到达`consoleread`等待输入到达（通过中断）并缓冲到`cons.buf`中，将输入复制到用户空间（等待整行到达后），然后返回给用户进程。如果用户尚未输入完整行，则任何读取进程都将在`sleep`调用中等待（第7章解释了`sleep`的详细信息）。

当用户键入字符时，UART硬件请RISC-V引发中断，激活了xv6的陷阱处理程序。陷阱处理程序调用`devintr`，后者查看RISC-V的`scause`寄存器，以发现中断是来自外部设备。然后它询问一个称为PLIC（）的硬件单元，了解哪个设备触发了中断。如果是UART，则`devintr`调用`uartintr`。

`uartintr`从UART硬件读取任何等待的输入字符并将它们传递给`consoleintr`。`uartintr`不必等待字符，因为后续输入会引发新的中断。`consoleintr`的工作是积累输入字符到`cons.buf`，直到整行到达。`consoleintr`对回退和一些其他字符进行特殊处理。当换行符到达时，`consoleintr`唤醒等待的`consoleread`（如果有的话）。

一旦`consoleread`被唤醒，它将在`cons.buf`中检索完整行，并将其复制到用户空间，然后通过系统调用机制返回给用户空间。
## 代码:控制台输出
对于连接到控制台的文件描述符的 `write` 系统调用最终会到达 `uartputc` 函数。设备驱动程序维护着一个输出缓存区 (`uart_tx_buf`)，这样写进程不必等待 UART 完成发送。相反，`uartputc` 函数将每个字符附加到缓存区中，调用 `uartstart` 函数启动设备进行传输（如果设备还没有开始发送），然后返回。唯一需要 `uartputc` 函数等待的情况是当缓存区已经满时。

每次 UART 完成发送一个字节时，它都会生成一个中断。`uartintr` 函数会调用 `uartstart` 函数，它会检查设备是否真的已经完成了发送，并将下一个缓存的输出字符传递给设备。因此，如果一个进程向控制台写入多个字节，则通常情况下，第一个字节将由 `uartputc` 函数调用 `uartstart` 函数发送，其余缓存的字节将由 `uartintr` 函数调用 `uartstart` 函数在传输完成的中断到达时发送。

需要注意的一个常见模式是，通过缓存和中断的方法，将设备活动与进程活动分离开。控制台驱动程序甚至可以在没有进程等待读取时处理输入；随后的读取操作便可看到输入。同样地，进程可以发送输出，而无需等待设备。这种分离方式可以通过允许进程与设备 I/O 并发执行来提高性能，特别是当设备速度较慢（如 UART）或需要立即处理时（如回显键入的字符）。这种思路有时被称为 **I/O并发**。

## 驱动程序中的并发

你可能已经注意到了在 `consoleread` 和 `consoleintr` 函数中的 `acquire` 函数调用。这些调用会获取一个锁，用于保护控制台驱动程序的数据结构，防止并发访问。在这里会存在三个并发问题：不同 CPU 上的两个进程可能同时调用 `consoleread` 函数；硬件可能在某个 CPU 正在执行 `consoleread` 函数时请求该 CPU 进行控制台（实际上是 UART）中断；硬件可能在不同的 CPU 上上面通知 `consoleread` 函数输入已到达。这些问题可能导致竞争或死锁。第6 章会深入探讨这些问题以及锁如何解决它们。

驱动程序中另一个需要考虑并发的问题是，一个进程可能在等待设备输入，但是在该输入到达的中断信号到达时，可能有不同的进程（或没有进程）正在运行。因此，中断处理程序不允许考虑其打断的进程或代码。例如，中断处理程序不能使用当前进程的页表安全地调用 `copyout` 函数。中断处理程序通常只做一些相对简单的工作（例如将输入数据复制到缓冲区），并唤醒顶部代码来处理其余的工作。

## 定时器中断

Xv6 使用定时器中断来维护其系统时间，并使其能够在计算密集型进程之间切换；在 `usertrap` 和 `kerneltrap` 中的 `yield` 调用就是通过定时器中断来实现进程切换的。定时器中断来自连接到每个 RISC-V CPU 的时钟硬件设备。Xv6 将这个时钟硬件设备程序化，使其周期性地中断每个 CPU。

RISC-V 要求定时器中断在机器模式下处理，而不是特权模式下。RISC-V 机器模式在没有分页的情况下执行，并具有单独的一组控制寄存器，因此在机器模式下运行普通的 xv6 内核代码不太现实。因此，xv6 完全将定时器中断与上述陷阱机制分开处理。

在 `main` 函数执行之前，在 `start.c` 中以机器模式执行的代码会进行一些设置，以便接收计时器中断。其一部分工作是程序 CLINT 硬件（core-local interruptor），使其在一定延迟后生成一个中断。另一部分工作则是设置一个类似于陷阱帧的 scratch 区域，用来帮助计时器中断处理程序保存寄存器和 CLINT 寄存器的地址。最后，`start` 函数将 `mtvec` 设置为 `timervec`，并启用计时器中断。

计时器中断可以在用户或内核代码执行任何时刻发生；内核无法在关键操作期间禁用计时器中断。因此，计时器中断处理程序必须以一种不会干扰中断的内核代码的方式执行其任务。基本的策略是让处理程序请求 RISC-V 引发“软件中断”，然后立即返回。RISC-V 会使用普通的陷阱机制将软件中断传递给内核，并允许内核禁用它们。处理由计时器中断产生的软件中断的代码如下：

机器模式计时器中断处理程序是 `timervec`。它会在由 `start` 准备的 scratch 区域中保存一些寄存器，告诉 CLINT 何时生成下一个计时器中断，请求 RISC-V 引发一个软件中断，然后恢复寄存器并返回。计时器中断处理程序中没有 C 代码。

## 实际情况

Xv6在内核执行时以及执行用户程序时都允许设备和定时器中断。即使在内核中执行时，定时器中断处理程序也会强制进行线程切换（调用yield）。按时间分片公平地在内核线程之间分配CPU是有用的，如果内核线程有时需花费大量时间计算，而不返回用户空间。然而，内核代码需要注意它可能会被挂起（由于定时器中断），并稍后在不同的CPU上恢复，这也是Xv6中一些复杂性的来源（见第6.6节）。如果设备和定时器中断只在执行用户代码时发生，则可以使内核更简单。

支持典型计算机上的所有设备并发挥它们的完整功能是很费力的，因为设备很多，设备拥有许多功能，并且设备驱动程序与驱动程序之间的协议可能很复杂且缺乏文档化。在许多操作系统中，驱动程序占用的代码比核心内核还多。

串行通讯UART驱动程序通过读取UART控制寄存器每次获取一个字节的数据，这种模式被称为编程I/O（PIO），因为软件在驱动数据的移动。编程I/O很简单，但在高速数据率下使用它太慢了。需要在高速移动大量数据的设备通常使用直接内存访问（DMA）。DMA设备硬件会直接将输入数据写入RAM，将输出数据从RAM读出。现代磁盘和网络设备使用DMA。适用于DMA设备的驱动程序将数据准备在RAM中，并使用一次写入控制寄存器的操作通知设备处理准备好的数据。

中断的合理使用在设备需要随机的关注时是有意义的，而且这种情况不太频繁。但是，中断会占用大量CPU时间。因此，高速设备（如网络和磁盘控制器）使用技巧来减少中断需求。其中一个技巧是为一整批输入和输出请求引发单个中断。另一个技巧是让驱动程序完全禁用中断，并定期检查设备是否需要关注。这种技术被称为轮询。如果设备执行操作非常快，轮询是有意义的，但如果设备大部分时间处于闲置状态，轮询会浪费CPU时间。一些驱动程序会根据当前设备负载动态地在轮询和中断之间切换。

UART驱动程序首先将传入的数据复制到内核缓冲区，然后复制到用户空间。在低数据率下这是合理的，但这种双重复制方式对于产生或消费数据非常快的设备会显著降低性能。一些操作系统可以直接在用户空间缓冲区和设备硬件之间移动数据，通常使用DMA技术。

正如第1章所述，控制台对于应用程序而言是一个常规文件，应用程序使用read和write系统调用读取输入和写入输出。应用程序可能希望控制无法通过标准文件系统调用表达的设备方面（例如在控制台驱动程序中启用/禁用行缓冲）。 Unix操作系统支持ioctl系统调用以满足这些情况。

某些计算机使用需要系统在有限的时间内作出响应的情况。例如，在安全关键系统中，未能按期限交付可能导致灾难。 Xv6不适用于硬实时环境。适用于硬实时的操作系统通常是链接到应用程序的库，以允许进行分析以确定最坏情况下的响应时间。由于xv6的调度程序过于简单并且具有在禁用很长时间的中断时执行的内核代码路径，因此它也不适合于软实时应用程序。

## 练习

1. 修改`uart.c`以完全不使用中断。您可能需要同时修改`console.c`。
2. 添加一个以太网卡的驱动程序。