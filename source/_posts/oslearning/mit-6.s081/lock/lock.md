## 为什么使用锁

app wants to multiple cores. Kernel must handle parallel syscalls and access shared data structures in parallel.

- locks for correct sharing
- locks can limit performance

为什么使用锁？避免race conditions



加锁的目的就是保证共享资源在任意时间内，只有一个线程可以访问，以此避免数据共享导致错乱的问题（避免race condition）。最底层就是两种锁：「互斥锁」和「自旋锁」，其他高级锁，如读写锁、悲观锁、乐观锁等都是基于它们实现的。

- **互斥锁**加锁失败后，线程释放CPU，给其他线程；
- **自旋锁**加锁失败后，线程会忙等待，直到它拿到锁；

xv6里的有两种锁，spinlock和sleeplock，sleeplock是基于spinlock和sleep实现的



## 互斥锁

互斥锁是一种「独占锁」，比如当线程 A 加锁成功后，此时互斥锁已经被线程 A 独占了，只要线程 A 没有释放手中的锁，线程 B 加锁就会失败，失败的线程B于是就会释放 CPU 让给其他线程，**既然线程 B 释放掉了 CPU，自然线程 B 加锁的代码就会被阻塞**。

**对于互斥锁加锁失败而阻塞的现象，是由操作系统内核实现的**。当加锁失败时，内核会将线程置为「睡眠」状态，等到锁被释放后，内核会在合适的时机唤醒线程，当这个线程成功获取到锁后，于是就可以继续执行

互斥锁加锁失败，就会从用户态陷入内核态，内核帮我们切换线程，这简化了互斥锁使用的难度，但也存在性能开销。

那这个开销成本是什么呢？会有**两次线程上下文切换的成本**：

- 当线程加锁失败时，内核会把线程的状态从「运行」状态设置为「睡眠」状态，然后把 CPU 切换给其他线程运行；
- 接着，当锁被释放时，之前「睡眠」状态的线程会变为「就绪」状态，然后内核会在合适的时间，把 CPU 切换给该线程运行。

## 自旋锁

自旋锁是最比较简单的一种锁，一直自旋，利用 CPU 周期，直到锁可用。**需要注意，在单核 CPU 上，需要抢占式的调度器（即通过时钟中断一个线程，运行其他线程）。否则，自旋锁在单 CPU 上无法使用，因为一个自旋的线程永远不会放弃 CPU。**

自旋锁开销少，在多核系统下一般不会主动产生线程切换，适合异步、协程等在用户态切换请求的编程方式，但如果被锁住的代码执行时间过长，自旋的线程会长时间占用 CPU 资源，所以自旋的时间和被锁住的代码执行的时间是成「正比」的关系，我们需要清楚的知道这一点。

## 锁抽象

```
struct lock{
}
acuire(&lock) => 只有一个进程可以获得锁
/////////这一部分是临界区代码，是原子性的
release(&lock)

struct spinlock {
	uint locked;
	char *name;
	struct cpu *cpu;
}
```



## 什么时候使用锁

两个以上进程共享一个数据结构的时候

lock perspectives

1. locks help avoid lost updates
2. locks make most step atomic
3. locks help maintain invariant



## Dead Lock

```
case 1:
acquire(&l)
acquire(&l)

case 2:
acquire(&d1)
acquire(&d2)
acquire(&d2)
acquire(&d1)
```

解决方法：锁排序





m1.g() <=>m2.f()

m2中的锁必须对m1可见（无锁编程里m2不需要知道m1的任何信息）



性能：

need to split up data structures



## 锁原理

先看一个基本的模拟锁的代码：

```
Broken acquire(struct lock *l) {
	while (1) {
		if (l -> locked == 0) {
			l -> locked = 1;
			return;
		}
	}
}

这个实现有问题，假设有两个cpu核心，都同时运行到了if语句，这个时候也会同时执行到if内部的语句，这就会导致问题，这是没办法的，因为仅仅从软件没有办法控制CPU的执行顺序；所以必须得依赖硬件指令才能解决这个问题。
```

锁的实现依赖于硬件支持

```
硬件的test andset support(测试和设置保证)
amoswap	addr1, reg1, reg2，这个指令做了以下内容：
lock addr(锁定地址)
	tmp <- *addr
	*addr <- r1
	r2 <- tmp
unlock

为什么叫测试和设置
测试：判断被锁定地址的值为0还是1？
设置：设置锁定地址的值

acquire(spinlock *lk) {
	push_off() // 禁用中断
	while(__sync_lock_test_and_set(&lk-<locked, 1) != 0);
	__sync_synchronize();
	lk->cpu = mucpu();
}

```

`Q: 为什么acquire里要禁用中断？`
因为不禁用中断的话，这个时候有中断进来，会调用uartintr, 而uartintr会获取同一把锁，如果是仅有一个cpu的架构，这个时候就没有cpu可以处理这个新来的中断

### 内存屏障和内存排序

某些编译器或者处理器对编译后的代码做优化，导致锁临界区间的代码被重调了，xv6有个内存屏障(synchronize)防止内存排序，_sync_synchronize()，在这个点之前的保存或存储不允许移动到这个点之后。



## 锁

首先，我们来看一下为什么我们需要锁？故事要从应用程序想要使用多个CPU核开始。使用多个CPU核可以带来性能的提升，如果一个应用程序运行在多个CPU核上，并且执行了系统调用，那么内核需要能够处理并行的系统调用。如果系统调用并行的运行在多个CPU核上，那么它们可能会并行的访问内核中共享的数据结构。到目前为止，你们也看到了XV6有很多共享的数据结构，例如**proc**、**ticks**和我们之后会看到的**buffer cache**等等。当并行的访问数据结构时，例如一个核在读取数据，另一个核在写入数据，我们需要使用锁来协调对于共享数据的更新，以确保数据的一致性。所以，我们需要锁来控制并确保共享的数据是正确的。

但是实际的情况有些令人失望，因为我们想要通过并行来获得高性能，我们想要并行的在不同的CPU核上执行系统调用，但是如果这些系统调用使用了共享的数据，**我们又需要使用锁，而锁又会使得这些系统调用串行执行，所以最后锁反过来又限制了性能。**

所以现在我们处于一个矛盾的处境，出于正确性，我们需要使用锁，但是考虑到性能，锁又是极不好的。这就是现实，我们接下来会看看如何改善这个处境。

那为什么要使用锁呢？前面我们已经提到了，是为了确保正确性。当一份共享数据同时被读写时，如果没有锁的话，可能会出现race condition，进而导致程序出错。

首先你们在脑海里应该有多个CPU核在运行，比如说CPU0在运行指令，CPU1也在运行指令，这两个CPU核都连接到同一个内存上。在前面的代码中，数据freelist位于内存中，它里面记录了2个内存page。假设两个CPU核在相同的时间调用kfree。

### 锁如何避免race condition

首先你们在脑海里应该有多个CPU核在运行，比如说CPU0在运行指令，CPU1也在运行指令，这两个CPU核都连接到同一个内存上。在前面的代码中，数据freelist位于内存中，它里面记录了2个内存page。假设两个CPU核在相同的时间调用kfree。

kfree函数接收一个物理地址pa作为参数，freelist是个单链表，kfree中将pa作为单链表的新的head节点，并更新freelist指向pa（注，也就是将空闲的内存page加在单链表的头部）。当两个CPU都调用kfree时，CPU0想要释放一个page，CPU1也想要释放一个page，现在这两个page都需要加到freelist中。

kfree中首先将对应内存page的变量r指向了当前的freelist（也就是单链表当前的head节点）。我们假设CPU0先运行，那么CPU0会将它的变量r的next指向当前的freelist。如果CPU1在同一时间运行，它可能在CPU0运行第二条指令（kmem.freelist = r）之前运行代码。所以它也会完成相同的事情，它会将自己的变量r的next指向当前的freelist。现在两个物理page对应的变量r都指向了同一个freelist（注，也就是原来单链表的head节点）。

接下来，剩下的代码也会并行的执行（kmem.freelist = r），这行代码会更新freelist为r。因为我们这里只有一个内存，所以总是有一个CPU会先执行，另一个后执行。我们假设CPU0先执行，那么freelist会等于CPU0的变量r。之后CPU1再执行，它又会将freelist更新为CPU1的变量r。这样的结果是，我们丢失了CPU0对应的page。CPU0想要释放的内存page最终没有出现在freelist数据中。

这是一种具体的坏的结果，当然可能会有更多坏的结果，因为可能会有更多的CPU。例如第三个CPU可能会短暂的发现freelist等于CPU0对应的变量r，并且使用这个page，但是之后很快freelist又被CPU1更新了。所以，拥有越多的CPU，我们就可能看到比丢失page更奇怪的现象。

### 锁的作用

通常锁有三种作用，理解它们可以帮助你更好的理解锁。

- 锁可以避免丢失更新。如果你回想我们之前在kalloc.c中的例子，丢失更新是指我们丢失了对于某个内存page在kfree函数中的更新。如果没有锁，在出现race condition的时候，内存page不会被加到freelist中。但是加上锁之后，我们就不会丢失这里的更新。
- 锁可以打包多个操作，使它们具有原子性。我们之前介绍了加锁解锁之间的区域是critical section，在critical section的所有操作会都会作为一个原子操作执行。
- 锁可以维护共享数据结构的不变性。共享数据结构如果不被任何进程修改的话是会保持不变的。如果某个进程acquire了锁并且做了一些更新操作，共享数据的不变性暂时会被破坏，但是在release锁之后，数据的不变性又恢复了。你们可以回想一下之前在kfree函数中的freelist数据，所有的free page都在一个单链表上。但是在kfree函数中，这个单链表的head节点会更新。freelist并不太复杂，对于一些更复杂的数据结构可能会更好的帮助你理解锁的作用

接下来我们再来看一下锁可能带来的一些缺点。我们已经知道了锁可以被用来解决一些正确性问题，例如避免race condition。但是不恰当的使用锁，可能会带来一些锁特有的问题。最明显的一个例子就是死锁（Deadlock）。

一个死锁的最简单的场景就是：首先acquire一个锁，然后进入到critical section；在critical section中，再acquire同一个锁；第二个acquire必须要等到第一个acquire状态被release了才能继续执行，但是不继续执行的话又走不到第一个release，所以程序就一直卡在这了。这就是一个死锁。XV6会探测这样的死锁，如果XV6看到了同一个进程多次acquire同一个锁，就会触发一个panic。

当有多个锁的时候，场景会更加有趣。假设现在我们有两个CPU，一个是CPU1，另一个是CPU2。CPU1执行rename将文件d1/x移到d2/y，CPU2执行rename将文件d2/a移到d1/b。这里CPU1将文件从d1移到d2，CPU2正好相反将文件从d2移到d1。我们假设我们按照参数的顺序来acquire锁，那么CPU1会先获取d1的锁，如果程序是真正的并行运行，CPU2同时也会获取d2的锁。之后CPU1需要获取d2的锁，这里不能成功，因为CPU2现在持有锁，所以CPU1会停在这个位置等待d2的锁释放。而另一个CPU2，接下来会获取d1的锁，它也不能成功，因为CPU1现在持有锁。这也是死锁的一个例子，有时候这种场景也被称为deadly embrace。这里的死锁就没那么容易探测了。

这里的解决方案是，如果你有多个锁，你需要对锁进行排序，所有的操作都必须以相同的顺序获取锁。

所以对于一个系统设计者，你需要确定对于所有的锁对象的全局的顺序。例如在这里的例子中我们让d1一直在d2之前，这样我们在rename的时候，总是先获取排序靠前的目录的锁，再获取排序靠后的目录的锁。如果对于所有的锁有了一个全局的排序，这里的死锁就不会出现了。

不过在设计一个操作系统的时候，定义一个全局的锁的顺序会有些问题。如果一个模块m1中方法g调用了另一个模块m2中的方法f，那么m1中的方法g需要知道m2的方法f使用了哪些锁。因为如果m2使用了一些锁，那么m1的方法g必须集合f和g中的锁，并形成一个全局的锁的排序。这意味着在m2中的锁必须对m1可见，这样m1才能以恰当的方法调用m2。

但是这样又违背了代码抽象的原则。在完美的情况下，代码抽象要求m1完全不知道m2是如何实现的。但是不幸的是，具体实现中，m2内部的锁需要泄露给m1，这样m1才能完成全局锁排序。所以当你设计一些更大的系统时，锁使得代码的模块化更加的复杂了。

### Uart

可以认为对于UART模块来说，现在是一个coarse-grained lock的设计。这个锁保护了UART的的传输缓存；写指针；读指针。当我们传输数据时，写指针会指向传输缓存的下一个空闲槽位，而读指针指向的是下一个需要被传输的槽位。这是我们对于并行运算的一个标准设计，它叫做消费者-生产者模式。

所以现在有了一个缓存，一个写指针和一个读指针。读指针的内容需要被显示，写指针接收来自例如printf的数据。我们前面已经了解到了锁有多个角色。第一个是保护数据结构的特性不变，数据结构有一些不变的特性，例如读指针需要追赶写指针；从读指针到写指针之间的数据是需要被发送到显示端；从写指针到读指针之间的是空闲槽位，锁帮助我们维护了这些特性不变。

### Spin lock实现

我们看一下spinlock.c文件，先来看一下acquire函数，在函数中有一个while循环，这就是我刚刚提到的test-and-set循环。实际上C的标准库已经定义了这些原子操作，所以C标准库中已经有一个函数__sync_lock_test_and_set，它里面的具体行为与我刚刚描述的是一样的。因为大部分处理器都有的test-and-set硬件指令，所以这个函数的实现比较直观。我们可以通过查看kernel.asm来了解RISC-V具体是如何实现的。下图就是atomic swap操作。

这里比较复杂，总的来说，一种情况下我们跳出循环，另一种情况我们继续执行循环。C代码就要简单的多。如果锁没有被持有，那么锁对象的locked字段会是0，如果locked字段等于0，我们调用test-and-set将1写入locked字段，并且返回locked字段之前的数值0。如果返回0，那么意味着没有人持有锁，循环结束。如果locked字段之前是1，那么这里的流程是，先将之前的1读出，然后写入一个新的1，但是这不会改变任何数据，因为locked之前已经是1了。之后__sync_lock_test_and_set会返回1，表明锁之前已经被人持有了，这样的话，判断语句不成立，程序会持续循环（spin），直到锁的locked字段被设置回0。

> 有很多同学提问说为什么release函数中不直接使用一个store指令将锁的locked字段写为0？有人想回答一下为什么吗？
>
> 是的，可能有两个处理器或者两个CPU同时在向locked字段写入数据。这里的问题是，对于很多人包括我自己来说，经常会认为一个store指令是一个原子操作，但实际并不总是这样，这取决于具体的实现。例如，对于CPU内的缓存，每一个cache line的大小可能大于一个整数，那么store指令实际的过程将会是：首先会加载cache line，之后再更新cache line。所以对于store指令来说，里面包含了两个微指令。这样的话就有可能得到错误的结果。所以为了避免理解硬件实现的所有细节，例如整数操作不是原子的，或者向一个64bit的内存值写数据是不是原子的，我们直接使用一个RISC-V提供的确保原子性的指令来将locked字段写为0。

第二个细节是，在acquire函数的最开始，会先关闭中断。为什么会是这样呢？让我们回到uart.c中。我们先来假设acquire在一开始并没有关闭中断。在uartputc函数中，首先会acquire锁，如果不关闭中断会发生什么呢？

uartputc函数会acquire锁，UART本质上就是传输字符，当UART完成了字符传输它会做什么？是的，它会产生一个中断之后会运行uartintr函数，在uartintr函数中，会获取同一把锁，但是这把锁正在被uartputc持有。如果这里只有一个CPU的话，那这里就是死锁。中断处理程序uartintr函数会一直等待锁释放，但是CPU不出让给uartputc执行的话锁又不会释放。在XV6中，这样的场景会触发panic，因为同一个CPU会再次尝试acquire同一个锁。

所以spinlock需要处理两类并发，一类是不同CPU之间的并发，一类是相同CPU上中断和普通程序之间的并发。针对后一种情况，我们需要在acquire中关闭中断。中断会在release的结束位置再次打开，因为在这个位置才能再次安全的接收中断