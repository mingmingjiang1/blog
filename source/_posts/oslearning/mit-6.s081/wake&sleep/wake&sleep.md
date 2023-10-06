## sleep &wakeup

像锁一样，sleep是另一种帮助程序线性化的工具

场景：

1. 等待管道有数据
2. 等待磁盘读取完成
3. wait系统调用，等待子进程完成某个事件

以软件的形式模拟：

```
Busy wait:
	while pipe buf is empty {} // 这样cpu空转，浪费cpu资源，最好的做饭是类似swtch一样学会放弃cpu，直到某个事件发生，才重新拥有cpu控制权，sleep和wakeup就是做这件事情的
	return the data
```

教授重写了UART的驱动代码：

```
uartwrite (char buf[], int n) // 驱动——处理输入，写入控制台的函数，写入硬件，每次一个字符one by one，由于uart每次只能写入一个字符，但实际情况一般是有很多字符，所以需要等待uart硬件处理完了，才继续发送下一个字符
while (i < n) {  // buf size是16，每16个字符就有一次中断
	while (tx_done == 0) { // 表明硬件还没准备好
		sleep(&tx_chan, &uart_tx_lock); // 硬件没准备好，就去睡觉，不要占着茅坑不拉屎
	}
	// 否则硬件已经准备好了
	WriteReg(THR, buf[i]) // buf是缓冲区的字符集，THR是硬件的传输寄存器
	i += 1;
	tx+ done = 0 // 重置状态
}

uartintr() // 驱动——处理中断
acquire(&uart_tx_lock)
if (ReadReg(LSR) & LSR_TX_IDLE) { // 硬件读取完成读取当前字符
	tx_done = 1; // 重置状态为1
	wakeup(&tx_chan); // 唤醒uartwrite，uartwrite重新掌握cou控制权
}
```



lost &wakeup原理：

```
broken_sleep(channel) {
	p -> state = SLEEPING;
	p->channel = channel
	swtch()
}

wakeup(channel) {
	for each p in procs[] {
		if (p ->state == SLEEPING && p -> channel == channel) {
			p->state = RUNNABLE;
		}
	}
}

为什么要有channel呢？没有行不行？
int done
uartwrite(buf) {
	for each c in buf {
		Lock
		while not done {
			sleep(&tx_chan)
		}
		send c
		done
		unLock
	}
}

intr() {
	Lock
	done = 1
	wakeup(&tx_chan)
	unLock
}

两个例程必须含锁，因为有一个共享数据结构buf；另外它们还共同访问同一个硬件，为了避免竞态条件，必须使用锁。锁加在哪里呢？
上面代码会导致死锁，因为中断触发的时候又加了一次锁，导致never wakeu up

所以一种解决办法是在sleep之前unlock，这样就给了中断例程去执行代码，然后wakeup，再在sleep之后acquire，放在在执行write的时候其他进程影响，
uartwrite(buf) {
	for each c in buf {
		Lock
		while not done {
			unLock
			broken_sleep(&tx_chan)
			Lock
		}
		send c
		done
		unLock
	}
}

但是这样还会有问题

事实上教授的演示里，是多个核心的，其他核心上确实发生了中断，但是中断发生的时候还没有执行到broken_sleep（差一行），这就会导致wakeup的时候，并没有唤醒任何东西；然后后面才sleep，这就是所谓的wakeup lost问题

但是有时候wakeup lost会意外地修复自己？
因为这里uart只有一个中断例程，无论是信号输入还是信号完成输出，所以当教授重新键入的时候，触发信号输入，触发中断，会重新wakeup，可能帮助之前的sleep wakeup了

那sleep到底如何实现呢？
出现上面case的本质原因是release之后，sleep的操作不是原子性的，导致了会被中断，有没有一直方法让sleep成为原子性的？
加另一把锁：
if (lk != &p->lock) {
	acquire(&p->lock); // 另一把锁，这里直接用了进程锁
	unLock; // 同时释放之前的条件锁，这个时候中断进程可以执行wakeup了，但是由于wakeup的时候，又被进程锁拦截了，导致没办法立即恢复运行，所以就SLEEP了
}
// Go to sleep
p-chan = chan
p->state = SLEEPING

sched() => 切换到调度器线程schedule的时候，会释放最近一次的运行的进程锁

p->chan = 0;

if (lk != &p->lock) {
	release(&p->lock); // 另一把锁，这里直接用了进程锁
	Lock;
}
```

> 一旦加了锁之后，进程就会在cpu上自旋，一直运行，除非release，也不允许中断



## Sleep & wakeup

在XV6中，任何时候调用switch函数都会从一个线程切换到另一个线程，通常是在用户进程的内核线程和调度器线程之间切换。在调用switch函数之前，总是会先获取线程对应的用户进程的锁。所以过程是这样，一个进程先获取自己的锁，然后调用switch函数切换到调度器线程，调度器线程再释放进程锁。

实际上的代码顺序更像这样：

1. 一个进程出于某种原因想要进入休眠状态，比如说出让CPU或者等待数据，它会先获取自己的锁；
2. 之后进程将自己的状态从RUNNING设置为RUNNABLE；
3. 之后进程调用switch函数，其实是调用sched函数在sched函数中再调用的switch函数；
4. switch函数将当前的线程切换到调度器线程；
5. 调度器线程之前也调用了switch函数，现在恢复执行会从自己的switch函数返回；
6. 返回之后，调度器线程会释放刚刚出让了CPU的进程的锁

在第1步中获取进程的锁的原因是，这样可以阻止其他CPU核的调度器线程在当前进程完成切换前，发现进程是RUNNABLE的状态并尝试运行它。为什么要阻止呢？因为其他每一个CPU核都有一个调度器线程在遍历进程表单，如果没有在进程切换的最开始就获取进程的锁的话，其他CPU核就有可能在当前进程还在运行时，认为该进程是RUNNABLE并运行它。而两个CPU核使用同一个栈运行同一个线程会使得系统立即崩溃。

所以，在进程切换的最开始，进程先获取自己的锁，并且直到调用switch函数时也不释放锁。而另一个线程，也就是调度器线程会在进程的线程完全停止使用自己的栈之后，再释放进程的锁。释放锁之后，就可以由其他的CPU核再来运行进程的线程，因为这些线程现在已经不在运行了。

> 学生提问：当我们有多个CPU核时，它们能看到同样的锁对象的唯一原因只可能是它们有一个共享的物理内存系统，对吧？
>
> Robert教授：是的。如果你有两个电脑，那么它们不会共享内存，并且我们就不会有这些问题。现在的处理器上，总是有多个CPU核，它们共享了相同的内存系统。

在线程切换的过程中，还有一点我之前没有提过。XV6中，不允许进程在执行switch函数的过程中，持有任何其他的锁。所以，进程在调用switch函数的过程中，必须要持有p->lock（注，也就是进程对应的proc结构体中的锁），但是同时又不能持有任何其他的锁。这也是包含了Sleep在内的很多设计的限制条件之一。如果你是一个XV6的程序员，你需要遵循这条规则。接下来让我解释一下背后的原因，首先构建一个不满足这个限制条件的场景：

我们有进程P1，P1的内核线程持有了p->lock以外的其他锁，这些锁可能是在使用磁盘，UART，console过程中持有的。之后内核线程在持有锁的时候，通过调用switch/yield/sched函数出让CPU，这会导致进程P1持有了锁，但是进程P1又不在运行。

假设我们在一个只有一个CPU核的机器上，进程P1调用了switch函数将CPU控制转给了调度器线程，调度器线程发现还有一个进程P2的内核线程正在等待被运行，所以调度器线程会切换到运行进程P2。假设P2也想使用磁盘，UART或者console，它会对P1持有的锁调用acquire，这是对于同一个锁的第二个acquire调用。当然这个锁现在已经被P1持有了，所以这里的acquire并不能获取锁。假设这里是spinlock，那么进程P2会在一个循环里不停的“旋转”并等待锁被释放。但是很明显进程P2的acquire不会返回，所以即使进程P2稍后愿意出让CPU，P2也没机会这么做。之所以没机会是因为P2对于锁的acquire调用在直到锁释放之前都不会返回，而唯一锁能被释放的方式就是进程P1恢复执行并在稍后release锁，但是这一步又还没有发生，因为进程P1通过调用switch函数切换到了P2，而P2又在不停的“旋转”并等待锁被释放。这是一种死锁，它会导致系统停止运行。

虽然刚刚的描述是基于机器上只有一个CPU核，但是你可以通过多个锁在多个CPU核的机器上构建类似的死锁场景。所以，我们在XV6中禁止在调用switch时持有除进程自身锁（注，也就是p->lock）以外的其他锁。

> 学生提问：难道定时器中断不会将CPU控制切换回进程P1从而解决死锁的问题吗？
>
> Robert教授：首先，所有的进程切换过程都发生在内核中，所有的acquire，switch，release都发生在内核代码而不是用户代码。实际上XV6允许在执行内核代码时触发中断，如果你查看trap.c中的代码你可以发现，如果XV6正在执行内核代码时发生了定时器中断，中断处理程序会调用yield函数并出让CPU。
>
> 但是在之前的课程中我们讲过acquire函数在等待锁之前会关闭中断，否则的话可能会引起死锁（注，详见10.8），所以我们不能在等待锁的时候处理中断。所以如果你查看XV6中的acquire函数，你可以发现函数中第一件事情就是关闭中断，之后再“自旋”等待锁释放。你或许会想，为什么不能先“自旋”等待锁释放，再关闭中断？因为这样会有一个短暂的时间段锁被持有了但是中断没有关闭，在这个时间段内的设备的中断处理程序可能会引起死锁。
>
> 所以不幸的是，当我们在自旋等待锁释放时会关闭中断，进而阻止了定时器中断并且阻止了进程P2将CPU出让回给进程P1。嗯，这是个好问题。
>
> 学生提问：能重复一下死锁是如何避免的吗？
>
> Robert教授：哦，在XV6中，死锁是通过禁止在线程切换的时候加锁来避免的。
>
> XV6禁止在调用switch函数时，获取除了p->lock以外的其他锁。如果你查看sched函数的代码（注，详见11.6），里面包含了一些检查代码来确保除了p->lock以外线程不持有其他锁。所以上面会产生死锁的代码在XV6中是不合法的并被禁止的。

前面我们介绍了在UART的驱动中，如何使用sleep和wakeup才能避免lost wakeup。前面这个特定的场景中，sleep等待的condition是发生了中断并且硬件准备好了传输下一个字符。在一些其他场景，内核代码会调用sleep函数并等待其他的线程完成某些事情。这些场景从概念上来说与我们介绍之前的场景没有什么区别，但是感觉上还是有些差异。例如，在读写pipe的代码中，如果你查看pipe.c中的piperead函数

![img](https://tnrhmtc0g9.feishu.cn/space/api/box/stream/download/asynccode/?code=MGNiNmViZjViNmVkOTBjM2IyNGY1NWQ0MTVlM2M2YmVfTlNLWGN1RnFURTdMRFhuUkFSU2hObUNSZ3Fub1U4ZnhfVG9rZW46UUZrbWJEcVVib1BUWEV4dHBwSGN6aDJCbkRnXzE2OTY1NzQ2OTE6MTY5NjU3ODI5MV9WNA)

所以会出现lost wakeup，是因为在一个不同的CPU核上可能有另一个线程刚刚调用了pipewrite:

![img](https://tnrhmtc0g9.feishu.cn/space/api/box/stream/download/asynccode/?code=NzUyOTJiMmU3MjZkOTE0YzNhODBiMTJlMmVhZDE0N2RfcTY2YWJZeEZnWHBxcTZlb2w5NlJIWUJrZ24wNkdGQzFfVG9rZW46R0UwVmJSQWFQb3dES2V4T2dyemNBS280bnFiXzE2OTY1NzQ2OTE6MTY5NjU3ODI5MV9WNA)

pipewrite会向pipe的缓存写数据，并最后在piperead所等待的sleep channel上调用wakeup。而我们想要避免这样的风险：在piperead函数检查发现没有字节可以读取，到piperead函数调用sleep函数之间，另一个CPU调用了pipewrite函数。因为这样的话，另一个CPU会向pipe写入数据并在piperead进程进入SLEEPING之前调用wakeup，进而产生一次lost wakeup。

在pipe的代码中，pipewrite和piperead都将sleep包装在一个while循环中。piperead中的循环等待pipe的缓存为非空（pipewrite中的循环等待的是pipe的缓存不为full）。之所以要将sleep包装在一个循环中，是因为可能有多个进程在读取同一个pipe。如果一个进程向pipe中写入了一个字节，这个进程会调用wakeup进而同时唤醒所有在读取同一个pipe的进程。但是因为pipe中只有一个字节并且总是有一个进程能够先被唤醒，哦，这正好提醒了我有关sleep我忘记了一些非常关键的事情。sleep函数中最后一件事情就是重新获取condition lock。所以调用sleep函数的时候，需要对condition lock上锁（注，在sleep函数内部会对condition lock解锁），在sleep函数返回时会重新对condition lock上锁。这样第一个被唤醒的线程会持有condition lock，而其他的线程在重新对condition lock上锁的时候会在锁的acquire函数中等待。

那个幸运的进程（注，这里线程和进程描述的有些乱，但是基本意思是一样的，当说到线程时是指进程唯一的内核线程）会从sleep函数中返回，之后通过检查可以发现pi->nwrite比pi->nread大1，所以进程可以从piperead的循环中退出，并读取一个字节，之后pipe缓存中就没有数据了。之后piperead函数释放锁并返回。接下来，第二个被唤醒的线程，它的sleep函数可以获取condition lock并返回，但是通过检查发现pi->nwrite等于pi->nread（注，因为唯一的字节已经被前一个进程读走了），所以这个线程以及其他所有的等待线程都会重新进入sleep函数。所以这里也可以看出，几乎所有对于sleep的调用都需要包装在一个循环中，这样从sleep中返回的时候才能够重新检查condition是否还符合。

sleep和wakeup的规则稍微有点复杂。因为你需要向sleep展示你正在等待什么数据，你需要传入锁并遵循一些规则，某些时候这些规则还挺烦人的。另一方面sleep和wakeup又足够灵活，因为它们并不需要理解对应的condition，只是需要有个condition和保护这个condition的锁。

除了sleep&wakeup之外，还有一些其他的更高级的Coordination实现方式。例如今天课程的阅读材料中的semaphore，它的接口就没有那么复杂，你不用告诉semaphore有关锁的信息。而semaphore的调用者也不需要担心lost wakeup的问题，在semaphore的内部实现中考虑了lost wakeup问题。因为定制了up-down计数器，所以semaphore可以在不向接口泄露数据的同时（注，也就是不需要向接口传递condition lock），处理lost wakeup问题。semaphore某种程度来说更简单，尽管它也没那么通用，如果你不是在等待一个计数器，semaphore也就没有那么有用了。这也就是为什么我说sleep和wakeup更通用的原因。

像锁一样，sleep是另一种帮助程序线性化的工具

场景：

1. 等待管道有数据
2. 等待磁盘读取完成
3. wait系统调用，等待子进程完成某个事件

以软件的形式简单模拟：

```JavaScript
Busy wait:
        while pipe buf is empty {} // 这样cpu空转，浪费cpu资源，最好的做法是类似swtch一样学会放弃cpu，直到某个事件发生，才重新拥有cpu控制权，sleep和wakeup就是做这件事情的
        return the data
```

改写的UART的驱动代码

```JavaScript
uartwrite (char buf[], int n) {
// 驱动——处理输入，写入控制台的函数，写入硬件，每次一个字符one by one，由于uart每次只能写入一个字符，但实际情况一般是有很多字符，所以需要等待uart硬件处理完了，才继续发送下一个字符
while (i < n) {  // buf size是16，每16个字符就有一次中断
        while (tx_done == 0) { // 表明硬件还没准备好
                sleep(&tx_chan, &uart_tx_lock); // 硬件没准备好，就去睡觉，不要占着茅坑不拉屎
        }
        // 否则硬件已经准备好了
        WriteReg(THR, buf[i]) // buf是缓冲区的字符集，THR是硬件的传输寄存器
        i += 1;
        tx+ done = 0 // 重置状态
}
}

uartintr() // 驱动——处理中断
{
    acquire(&uart_tx_lock)
    if (ReadReg(LSR) & LSR_TX_IDLE) { // 硬件读取完成读取当前字符
            tx_done = 1; // 重置状态为1
            wakeup(&tx_chan); // 唤醒uartwrite，uartwrite重新掌握cou控制权
    }
}
```

### Wakeup & lost wakeup原理

```JavaScript
broken_sleep(channel) {
        p -> state = SLEEPING;
        p->channel = channel
        swtch()
}

wakeup(channel) {
        for each p in procs[] {
                if (p ->state == SLEEPING && p -> channel == channel) {
                        p->state = RUNNABLE;
                }
        }
}
```

为什么要有channel呢？没有行不行？

```JavaScript
int done
uartwrite(buf) {
        for each c in buf {
                Lock
                while not done {
                        sleep(&tx_chan)
                }
                send c
                done
                unLock
        }
}

intr() {
        Lock
        done = 1
        wakeup(&tx_chan)
        unLock
}
两个例程必须含锁，因为有一个共享数据结构buf；另外它们还共同访问同一个硬件，为了避免竞态条件，必须使用锁。锁加在哪里呢？
上面代码会导致死锁，因为中断触发的时候又加了一次锁，导致never wake up

所以一种解决办法是在sleep之前unlock，这样中断例程就有机会去执行代码，然后wakeup，再在sleep之后acquire，放在在执行write的时候其他进程影响：

uartwrite(buf) {
        for each c in buf {
                Lock
                while not done {
                        unLock
                        sleep(&tx_chan)
                        Lock
                }
                send c
                done
                unLock
        }
}

事实上教授的演示里，是多个核心的，其他核心上确实发生了中断，但是中断发生的时候还没有执行到broken_sleep的线程切换，这就会导致wakeup的时候，并没有唤醒任何东西；然后后面才sleep，这就是所谓的wakeup lost问题

但是有时候wakeup lost会意外地修复自己？
因为这里uart只有一个中断例程，无论是信号输入还是信号完成输出，所以当教授重新键入的时候，触发信号输入，触发中断，会重新wakeup，可能帮助之前的丢失的sleep wakeup了


那sleep到底如何实现呢？
出现上面case的本质原因是release之后，sleep的操作不是原子性的，导致了会被中断，有没有一直方法让broken_sleep成为原子性的？
加另一把锁：

broken_sleep（chan） {
    if (lk != &p->lock) {
            acquire(&p->lock); // 另一把锁，这里直接用了进程锁
            unLock; // 同时释放之前的条件锁，这个时候中断进程可以执行wakeup了，但是由于wakeup的时候，又被进程锁拦截了，导致没办法立即恢复运行，所以就等待了
    }
    // Go to sleep
    p-chan = chan
    p->state = SLEEPING
    
    sched() => 切换到调度器线程schedule的时候，会释放最近一次的运行的进程锁，
    
    p->chan = 0; // 下一次wake up的时候从这里开始
    
    if (lk != &p->lock) {
            release(&p->lock);
            Lock;
    }
}
```

