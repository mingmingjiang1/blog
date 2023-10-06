## 中断(Intertupt)

背景知识：

通常一个简易的RISC-V处理器由三大块构成，一是执行指令流水的core，二是平台中断控制的PLIC，三是负责调试的DM（debug module）

PLIC(platform-level interrupt controller)，平台级中断控制器。用来将外部的全局中断请求处理后转至中断目标。PLIC理论上支持1023个外部中断源和15872个上下文，但真实设计实现时一般不需要这么多。

通俗一点理解，PLIC就像公司的秘书，中断目标就像公司的老板，外部中断请求就像要拜访老板的访客。因为访客很多，所以如果访客不经任何处理全挤在老板的办公室里，会很嘈杂。因此，老板就招了一个秘书，并规定，所有的访客在秘书那里登记，然后秘书根据访客的重要性，对访客进行排序，每次只将优先级最高的访客通知给老板，以此来提高办公效率。

通常一个系统（平台）中会有很多外设，而每个外设都有一个或多个中断请求，在系统正常工作中处理器内核负责处理（服务）所有外设中断，但通常这些外设中断并不直接接入内核，而是经平台中断控制器（PLIC）处理后再转至内核处理。

PLIC将所有外部中断源的中断请求汇总起来，并根据用户配置的中断优先级，中断使能，中断目标阈值等信息进行处理，最终将仲裁”胜利“的（最高优先级且大于中断目标阈值）外部中断源的中断请求转至给中断目标处理。因这些配置信息由用户配置，所以PLIC也可以理解为可编程的中断控制器。

https://zhuanlan.zhihu.com/p/628296038

### 什么是中断？

**中断对应的场景很简单，就是硬件想要得到操作系统的关注。**例如网卡收到了一个packet，网卡会生成一个中断；用户通过键盘按下了一个按键，键盘会产生一个中断。操作系统需要做的是，保存当前的工作，处理中断，处理完成之后再恢复之前的工作。这里的保存和恢复工作，与我们之前看到的系统调用过程非常相似。**所以系统调用，page fault，中断，都使用相同的机制**。

但是中断又有一些不一样的地方，这就是为什么我们要花一节课的时间来讲它。中断与系统调用主要有3个小的差别：

1. asynchronous。当硬件生成中断时，Interrupt handler与当前运行的进程在CPU上没有任何关联。但如果是系统调用的话，系统调用发生在运行进程的context下。
2. concurrency。我们这节课会稍微介绍并发，在下一节课，我们会介绍更多并发相关的内容。对于中断来说，CPU和生成中断的设备是并行的在运行。网卡自己独立的处理来自网络的packet，然后在某个时间点产生中断，但是同时，CPU也在运行。所以我们在CPU和设备之间是真正的并行的，我们必须管理这里的并行。
3. program device。我们这节课主要关注外部设备，例如网卡，UART，而这些设备需要被编程。每个设备都有一个编程手册，就像RISC-V有一个包含了指令和寄存器的手册一样。设备的编程手册包含了它有什么样的寄存器，它能执行什么样的操作，在读写控制寄存器的时候，设备会如何响应。不过通常来说，设备的手册不如RISC-V的手册清晰，这会使得对于设备的编程会更加复杂。

### 中断来自于哪里？

外中断：各种设备

内中断：时钟中断，exception

### 驱动程序

通常来说，**管理设备的代码称为驱动**，所有的驱动都在内核中。我们今天要看的是UART设备的驱动，代码在uart.c文件中。如果我们查看代码的结构，我们可以发现大部分驱动都分为两个部分，bottom/top。

**bottom部分通常是Interrupt handler。当一个中断送到了CPU，并且CPU设置接收这个中断，CPU会调用相应的Interrupt handler。Interrupt handler并不运行在任何特定进程的context中，它只是处理中断。**

top部分，是用户进程，或者内核的其他部分**调用的接口**。对于UART来说，这里有read/write接口，这些接口可以被更高层级的代码调用。

通常情况下，驱动中会有一些队列（或者说buffer），top部分的代码会从队列中读写数据，而Interrupt handler（bottom部分）同时也会向队列中读写数据。这里的队列可以将并行运行的设备和CPU解耦开来。

通常对于Interrupt handler来说存在一些限制，因为它并没有运行在任何进程的context中，所以进程的page table并不知道该从哪个地址读写数据，也就无法直接从Interrupt handler读写数据。驱动的top部分通常与用户的进程交互，并进行数据的读写。我们后面会看更多的细节，这里是一个驱动的典型架构。

### 设备编程基础

SIE: 管理程序中的中断寄存器，one bit for E(外部中断), S(软件中断), T(时钟中断)

SSTATUS: bit for enable/disable

STP: interrupt pending(中断挂起寄存器)

SSCAUSE：造成中断的原因

STVEC：它会保存当trap，page fault或者中断发生时，CPU运行的用户程序的程序计数器，这样才能在稍后恢复程序的运行。

### Top

init.c[main]: 首先这个进程的main函数创建了一个代表Console的设备。这里通过mknod操作创建了console设备。因为这是第一个打开的文件，所以这里的文件描述符0。之后通过dup创建stdout和stderr。这里实际上通过复制文件描述符0，得到了另外两个文件描述符1，2。最终文件描述符0，1，2都用来代表Console。

Getcmd: fprintf(2, '$ );

=> write

=> sys_write

=> filewrite：在filewrite函数中首先会判断文件描述符的类型。mknod生成的文件描述符属于设备（FD_DEVICE），而对于设备类型的文件描述符，我们会为这个特定的设备执行设备相应的write函数。因为我们现在的设备是Console，所以我们知道这里会调用console.c中的consolewrite函数。

Consolewrite: 这里先通过either_copyin将字符拷入，之后调用uartputc函数。uartputc函数将字符写入给UART设备，所以你可以认为consolewrite是一个UART驱动的top部分。uart.c文件中的uartputc函数会实际的打印字符。

Uartputc: uartputc函数会稍微有趣一些。在UART的内部会有一个buffer用来发送数据，buffer的大小是32个字符。同时还有一个为consumer提供的读指针和为producer提供的写指针，来构建一个环形的buffer（注，或者可以认为是环形队列）,在我们的例子中，Shell是producer，所以需要调用uartputc函数。在函数中第一件事情是判断环形buffer是否已经满了。如果读写指针相同，那么buffer是空的，如果写指针加1等于读指针，那么buffer满了。当buffer是满的时候，向其写入数据是没有意义的，所以这里会sleep一段时间，将CPU出让给其他进程。当然，对于我们来说，buffer必然不是满的，因为提示符“$”是我们送出的第一个字符。所以代码会走到else，字符会被送到buffer中，更新写指针，之后再调用uartstart函数。

Uart: uartstart就是通知设备执行操作。首先是检查当前设备是否空闲，如果空闲的话，我们会从buffer中读出数据，然后将数据写入到THR（Transmission Holding Register）发送寄存器。这里相当于告诉设备，我这里有一个字节需要你来发送。一旦数据送到了设备，系统调用会返回，用户应用程序Shell就可以继续执行。这里从内核返回到用户空间的机制与lec06的trap机制是一样的。

### Bottom

在我们向Console输出字符时，如果发生了中断，RISC-V会做什么操作？我们之前已经在SSTATUS寄存器中打开了中断，所以处理器会被中断。假设键盘生成了一个中断并且发向了PLIC，PLIC会将中断路由给一个特定的CPU核，并且如果这个CPU核设置了SIE寄存器的E bit（注，针对外部中断的bit位），那么会发生以下事情：

首先，会清除SIE寄存器相应的bit，这样可以阻止CPU核被其他中断打扰，该CPU核可以专心处理当前中断。处理完成之后，可以再次恢复SIE寄存器相应的bit。

之后，会设置SEPC寄存器为当前的程序计数器。我们假设Shell正在用户空间运行，突然来了一个中断，那么当前Shell的程序计数器会被保存。

之后，要保存当前的mode。在我们的例子里面，因为当前运行的是Shell程序，所以会记录user mode。

再将mode设置为Supervisor mode。

最后将程序计数器的值设置成STVEC的值。（注，STVEC用来保存trap处理程序的地址，详见lec06）在XV6中，STVEC保存的要么是uservec或者kernelvec函数的地址，具体取决于发生中断时程序运行是在用户空间还是内核空间。在我们的例子中，Shell运行在用户空间，所以STVEC保存的是uservec函数的地址。而从之前的课程我们可以知道uservec函数会调用usertrap函数。所以最终，我们在usertrap函数中。我们这节课不会介绍trap过程中的拷贝，恢复过程，因为在之前的课程中已经详细的介绍过了。

在trap.c的devintr函数中，首先会通过SCAUSE寄存器判断当前中断是否是来自于外设的中断。如果是的话，再调用plic_claim函数来获取中断。

plic_claim函数位于plic.c文件中。在这个函数中，当前CPU核会告知PLIC，自己要处理中断，PLIC_SCLAIM会将中断号返回，对于UART来说，返回的中断号是10

所以代码会直接运行到uartstart函数，这个函数会将Shell存储在buffer中的任意字符送出。实际上在提示符“$”之后，Shell还会输出一个空格字符，write系统调用可以在UART发送提示符“$”的同时，并发的将空格字符写入到buffer中。所以UART的发送中断触发时，可以发现在buffer中还有一个空格字符，之后会将这个空格字符送出。

### Case：shell提示符的代码跟踪

当XV6启动时，Shell会输出提示符“$ ”，如果我们在键盘上输入ls，最终可以看到“$ ls”。我们接下来通过研究Console是如何显示出“$ ls”，来看一下设备中断是如何工作的。

实际上“$ ”和“ls”还不太一样，“$ ”是Shell程序的输出，而“ls”是用户通过键盘输入之后再显示出来的。

对于“$ ”来说，实际上就是设备会将字符传输给UART的寄存器，UART之后会在发送完字符之后产生一个中断。在QEMU中，模拟的线路的另一端会有另一个UART芯片（模拟的），这个UART芯片连接到了虚拟的Console，它会进一步将“$ ”显示在console上.

另一方面，对于“ls”，这是用户输入的字符。键盘连接到了UART的输入线路，当你在键盘上按下一个按键，UART芯片会将按键字符通过串口线发送到另一端的UART芯片。另一端的UART芯片先将数据bit合并成一个Byte，之后再产生一个中断，并告诉处理器说这里有一个来自于键盘的字符。之后Interrupt handler会处理来自于UART的字符。我们接下来会深入通过这两部分来弄清楚这里是如何工作的。

### 中断与并发

1. device + cpu run in parallel (produce/consumer parallelism)：设备与CPU是并行运行的。例如当UART向Console发送字符的时候，CPU会返回执行Shell，而Shell可能会再执行一次系统调用，向buffer中写入另一个字符，这些都是在并行的执行。这里的并行称为producer-consumer并行。，
2. interrput stops the current program：中断可以发生在任何时候，这意味可能在内核空间发生中断，所以即使是内核也不是一定会同步执行每条命令的，所以必须在某段时间内禁用中断，从而使得部分操作是原子的，
3. 驱动的并行性（驱动的上部分和下部分是并行的），在打印完$之后，shell会再次调用write系统调用，同时驱动顶部也在处理$ 之后的空格，但是在另一个核心，可能在处理来自UART的中断（其实就是一份代码可能同时运行在不同的核心上，而同一个设备是共享一个buffer的，为了让同一时间内只允许一个cpu处理buffer，使用锁）管理这种方式就是使用锁，

当用户在终端上输入的时候，一方面触发系统调用read => fileread => 如果这个时候缓冲区没有字符则sleep，否则从从缓冲区读取；另一方面用户输入字符这将导致字符发生到电路板上的UART芯片，通过PLIC到达某个核心，核心将接受中断，同时uart驱动自己会把设备传输到的字符放入缓冲区，唤醒等待区的进程，在这里即用户触发的系统调用

如果两个指针相等，那么buffer是空的。当Shell调用uartputc函数时，会将字符，例如提示符“$”，写入到写指针的位置，并将写指针加1。这就是producer对于buffer的操作。这是驱动中的非常常见的典型现象。如你们所见的，在驱动中会有一个buffer，在我们之前的例子中，buffer是32字节大小。并且有两个指针，分别是读指针和写指针。

![img](https://tnrhmtc0g9.feishu.cn/space/api/box/stream/download/asynccode/?code=OTRjNmJkMDJmMGRhYmIxMzZjNTc5NTVkMjY5Mjc2YzRfSWM1WUJEQ2t6UjMySkFaeXFnQ2dXbHc1dnJMVm40WDhfVG9rZW46Q0NNc2JEZXgzb0hjVnJ4YThoVmM3YnkxbmJkXzE2OTY1NzQ3MjQ6MTY5NjU3ODMyNF9WNA)

producer可以一直写入数据，直到写指针 + 1等于读指针，因为这时，buffer已经满了。当buffer满了的时候，producer必须停止运行。我们之前在uartputc函数中看过，如果buffer满了，代码会sleep，暂时搁置Shell并运行其他的进程。

Interrupt handler，也就是uartintr函数，在这个场景下是consumer，每当有一个中断，并且读指针落后于写指针，uartintr函数就会从读指针中读取一个字符再通过UART设备发送，并且将读指针加1。当读指针追上写指针，也就是两个指针相等的时候，buffer为空，这时就不用做任何操作

> 学生提问：这里的buffer对于所有的CPU核都是共享的吗？
>
> Frans教授：这里的buffer存在于内存中，并且只有一份，所以，所有的CPU核都并行的与这一份数据交互。所以我们才需要lock。
>
> 学生提问：对于uartputc中的sleep，它怎么知道应该让Shell去sleep？
>
> Frans教授： sleep会将当前在运行的进程存放于sleep数据中。它传入的参数是需要等待的信号，在这个例子中传入的是uart_tx_r的地址。在uartstart函数中，一旦buffer中有了空间，会调用与sleep对应的函数wakeup，传入的也是uart_tx_r的地址。任何等待在这个地址的进程都会被唤醒。有时候这种机制被称为conditional synchronization。

### **中断的发展历史**

### 中断发展历史

Unix: 在以前比较简单的系统中，中断很快，有什么重要的工作要做，直接中断即可

Now：很慢，因为中断需要保持寄存器，而且现代设备越来越复杂

对于高速设置，设备来不及中断，解决方案：pulling，比如千兆以太网网卡设备，那么网卡美妙产生大约150w个包。这意味着对每个包产生一个中断，平均1us一次中断。

轮询的方式，有个缺点，就是如果设备太慢的话，cpu自旋，浪费了cpu周期；太快的话，节省了内核切换成本

CPU spins until device has data，but wastes cpu cycles，

现代高级cpu会动态地在轮询和中断状态直接切换