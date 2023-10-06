## xv6启动过程(Boot)

首先，写在**ROM**中的**boot loader**，将内核装载入内存中，并将执行流跳转到固定的入口点(代码写死在ROM中，自然跳转的入口点也是固定的)

因此，编译内核时，需要将内核的入口函数**_entry**(kernel/entry.S:6)编译到**boot loader**指定的地址处(xv6是0x80000000)

而为了*让生活更美好*，**_entry**函数使用汇编指令初始化栈后，跳转到使用C语言编写的**start**(kernel/start.c:20)。 **start**函数的主要工作就是设置**CSRs**(control and state registers)，从而切换到**S-mode**(supervisor mode)，并将执行流设置成**main**(kernel/main.c:10)。**start**函数实现的非常巧妙，其通过设置**CSRs**，伪造一个异常处理保存的上下文，其上下文的特权级是**S-mode**，PC是**main**。执行**mret**指令后，通过恢复上下文，完成特权级和执行流的转换

在**main**函数中，即初始化内核的子系统，并执行**userinit**(kernel/proc.c:211)，创建第一个进程。 **userinit**函数申请进程描述符和虚拟地址空间等资源，将**initcode.S**(user/initcode.S:1)的汇编代码映射入进程中并设置为进程入口函数。

**initcode.S**(user/initcode.S:1)中，将**/init加载到内存中，它**是user/init.c编译的可执行程序，其初始化中断设备，初始化文件描述符，并启动**sh**

```C
void
main()
{
  if(cpuid() == 0){ 
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // 初始化内存
    kvminit();       // 为设备和kernel建设页表映射
    kvminithart();   // 设置SATP寄存器为刚才的kernel_pagetable变量，当这里这条指令执行之后，下一个指令的地址会发生什么？
    所以这条指令的执行时刻是一个非常重要的时刻。因为整个地址翻译从这条指令之后开始生效，之后的每一个使用的内存地址都可能对应到与之不同的物理内存地址。因为在这条指令之前，我们使用的都是物理内存地址，这条指令之后page table开始生效，所有的内存地址都变成了另一个含义，也就是虚拟内存地址。
这里能正常工作的原因是值得注意的。因为前一条指令还是在物理内存中，而后一条指令已经在虚拟内存中了。比如，下一条指令地址是0x80001110就是一个虚拟内存地址。等等，0x80001110看起来好像还是物理地址，这是怎么回事？因为kernel page的映射关系中，虚拟地址到物理地址是完全相等的。所以，在我们打开虚拟地址翻译硬件之后，地址翻译硬件会将一个虚拟地址翻译到相同的物理地址。所以实际上，我们最终还是能通过内存地址执行到正确的指令，因为经过地址翻译0x80001110还是对应0x80001110。

    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode table
    fileinit();      // 文件系统初始化
    virtio_disk_init(); // emulated hard disk
    userinit();      // 启动用户进程
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;   
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();    
}


// 进程初始化，为每个进程分配kernel stack
void
procinit(void)
{
  struct proc *p; 
  
  initlock(&pid_lock, "nextpid");
  initlock(&wait_lock, "wait_lock");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");
      p->kstack = KSTACK((int) (p - proc));
  }
}

void
trapinit(void)
{
  initlock(&tickslock, "time");
}

// set up to take exceptions and traps while in the kernel.
void
trapinithart(void)
{
  w_stvec((uint64)kernelvec); // 设置stvec指向kernelvec，kernelvec是一个.S文件，当有中断产生时，会通过ecall进入管理员模式，同时由ecall指令设置stvec指向trampoline，这是硬件层面做的吗？在代码里并没x有看到任何像w_stvec(trampolone)的代码。
}

// Set up first user process.
void
userinit(void)
{
  struct proc *p; 

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}

// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();
  p->state = USED;

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}

int
growproc(int n)
{
  uint sz; 
  struct proc *p = myproc();

  sz = p->sz; // va每次递增都是基于之前的sz大小进行递增的，初始的时候是0，所以va起始于0
  if(n > 0){ 
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1; 
    }   
  } else if(n < 0){ 
    sz = uvmdealloc(p->pagetable, sz, sz + n); 
  }
  p->sz = sz; 
  return 0;
}

pagetable_t
proc_pagetable(struct proc *p) 
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){ 
    uvmfree(pagetable, 0); 
    return 0;
  }

  // map the trapframe just below TRAMPOLINE, for trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){ 
    uvmunmap(pagetable, TRAMPOLINE, 1, 0); 
    uvmfree(pagetable, 0); 
    return 0;
  }

  return pagetable;
}
```

虚拟内存和物理内存之间映射其中有几点需要注意下：

它被映射到虚拟地址空间的顶端，用户和内核的页表里都有这一项映射，摆放的位置相同，很快我们将在下一章介绍这一页的作用。trampoline页被映射了两次，一次映射到虚拟地址空间的顶端，一次是直接映射。为什么要这样设计呢？

为了在user space和kernel space之间切换时，trampoline的汇编代码在切换前后仍然能够继续执行。

 code to switch between user and kernel space.

 this code is mapped at the same virtual address

   (TRAMPOLINE) in user and kernel space so that

it continues to work when it switches page tables.