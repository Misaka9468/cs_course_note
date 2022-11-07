## 遇到的bug

- 在make grade时，报错 /usr/bin/env: ‘python3\r’: No such file or directory.
- 很显然是文件编码格式，万恶的换行方式
- grade-lab-xxx  用vim打开   :set ff=unix 然后退出即可

## Lab1 Xv6 and Unix utilities

- [Lab: Xv6 and Unix utilities](https://pdos.csail.mit.edu/6.828/2021/labs/util.html)

![img](https://picx1.zhimg.com/80/v2-4bb914afc5bfdf2948f62c15f988d303_720w.png?source=d16d100b)



### 1.1 sleep (easy)

没有难度，主要是教会你如何使用系统调用.

### 1.2 pingpong (easy)

- fork()后，子进程将会继承父进程的文件描述符表，会共享pipe这个特殊的文件.
- p[0]为read-end, p[1]为write-end.
- 使用pipe()时要注意的几个原则
- 该进程不需要的端，尽早close
- 使用完的端，尽早close
- read-end只有在write-end的所有link都被close掉才会接收EOF.

### 1.3 primes (moderate/hard)

![img](https://pic1.zhimg.com/80/v2-1fbbfe501fd807dfaf21db3d6d829fe7_720w.png?source=d16d100b)



**伪代码:**

```
p = get a number from left neighbor
print p
loop:
    n = get a number from left neighbor
    if (p does not divide n)
        send n to right neighbor
```

- 考虑用递归来解决fork()的层层使用
- 管道可以作为参数; 遵循管道的close原则
- 针对该题, 每个进程只有一个子进程(子进程的子进程似乎不是我的子进程，存疑?) 所以不用考虑waitall()的实现.

### 1.4 find (moderate)

- argv[0]为程序名
- fstat(fd, &st)用于获得文件的信息，存放在struct stat st中.
- Directory is a file containing a sequence of dirent structures.  其中struct dirent{} 包含了该目录下各文件的inum以及filename. 在实验中，通过filename以及其所在目录来获得完整文件路径，从而利用fstat来获得该目录下各文件的stat信息.

### 1.5  xargs (moderate)

- 再次理解管道: 左侧程序的输出，送入右侧程序的standard input.
- [xargs的用途](https://www.runoob.com/linux/linux-comm-xargs.html)
- 对于exec(char*,char** ), 第二个参数为执行文件的参数列表, argv[0]为执行文件名.  

## Lab2 system calls

- [Lab: System calls](https://pdos.csail.mit.edu/6.828/2021/labs/syscall.html)

![img](https://picx1.zhimg.com/80/v2-86c552c30196f7d6db622301e16056e7_720w.png?source=d16d100b)

### 2.0 讲解

syscall在xv-6的流程：

- user/*.c中，调用例如fork()函数. 函数原型在user/user.h中. 实际上并不会call系统调用，而是将系统调用号存入a7, 之后调用ecall. 这个实现是由usys.pl生成的usys.S中的汇编代码实现。回顾，编译过程只要看到原型就认为user代码中的函数调用合法，之后在链接时，usys.S中的函数实现已经为.o文件，因此user中的函数调用自然会重定向到这些这些汇编实现的代码。
- ecall后会进入kernel设置好的entry. trampoline.S会调用usertrap(kernel/trap.c:37). 该函数会处理来自用户态的异常、中断与系统调用。其会调用syscall函数(kernel/syscall.c:163).
- 在syscall()函数中，其判断p->trapframe->a7的系统调用号，根据下标与对应sys_函数的映射，调用对应的系统调用函数。完成后会返回。

### 2.1 System call tracing

some hint

- 大胆地在proc结构体中加入新成员
- 在内核空间中，myproc()将返回当前进程的指针，可以对其信息进行读/写。
- 对于系统调用的参数：根据函数原型中参数的顺序，其被存储在p->trapframe->a?中。使用argraw和argint、argaddr等进行读取。

### 2.2 Sysinfo

some hint

- 在user.h中添加sysinfo的原型时，应该声明参数的结构体，告知编译器。
- 如何填充在user space下的sysinfo结构体呢？做法是将kernel的、已经更新的sysinfo结构体复制到user space里的sysinfo结构体中。利用copyout(pagetable_t, uint64, char* *,* uint64);  其中，char* 指向内核态里某地址，uint64为复制的长度，该函数将以这个地址为起始，复制len个字节到进程页表的指定地址。
- 根据以上分析可知，我们需要得到user space中sysinfo的地址。获取该系统调用的参数，使用argaddr(0,&addr).  将地址赋给addr. 
- 对于getnproc: 遍历proc[]数组、判断state即可。
- 对于getfreemem: 观察kalloc.c可以大致得出结论。struct run{}结构体用于page的遍历, 其长度为64bit (也就是RISC-V中一个地址/指针的长度)。类似一个链表结构。kmem.freelist维护空闲页组成链表的头部。若要对kmem进行修改，还需要对锁进行acquire和release.



## Lab3 page tables

![img](https://pic1.zhimg.com/80/v2-aba7c011423022233bb8abc2f0357f11_720w.jpeg?source=d16d100b)



编辑切换为居中

添加图片注释，不超过 140 字（可选）

### 3.1 Speed up system calls

给定了虚拟地址USYSCALL (TRAPFRAME - PGSIZE), 以及user的方法ugetid(). 在该函数中，直接从该地址中取得一个uint64值作为当前进程的pid. 

实验即要求我们实现该虚拟地址到实际物理地址的映射。 整个运行的逻辑应该是，测试程序调用ugetpid() -> 访问虚拟地址 USYSCALL -> 经过页表转化为对物理地址的访问 -> 从我们分配、填写好的物理页中，取得想要的值。

一些hint:

- allocproc()中，需要申请一个物理页，并将当前pid记录到物理页中。
- 我们在proc_pagetable()中，才将虚拟地址映射(mappages)到刚才申请的页，所以我们需要记录该物理页的地址。考虑到每个进程都要有一个私有的页来存放该值，因此在proc结构体中加入指针struct usyscall*（类比trapframe的做法）。
- freeproc(): 
- free进程时，首先要将该物理页释放。同样类比trapframe, 利用kfree()来释放一个物理页。
- 之后调用proc_freepagetable(), 则是将映射关系free. 即让PTE失效. 注意在这里需要加入uvmunmap(pagetable, USYSCALL, 1, 0); 
- 只有当所有子PTE都失效，才能free掉该页表。这就是为什么在uvmfree之前，要uvmunmap所有PTE.

### 3.2 Print a page table

遍历、按照格式打印页表内容即可

一些提示

- pagetable_t类型是 uint64*.*  pte  = pagetable[idx]，其实就是 *(pagetable + idx * sizeof(uint64)).  也就是页表中第i个项的uint64值。
- pte_t本身是一个uint64, 就代表了该项pte的内容。转化为pa后，可以赋给 pagetable_t类型，从而使得pagetable_t这个指针指向pa，也就是指向物理地址。
- 前3页，应该是text和data合用一页，然后stack和保护页各用一页；后3页，就是trampoline, trapframe, USYSCALL.

### 3.3 Detecting which pages have been accessed

要求你实现系统调用pgacesss(). 其接收参数va, npages, addr. 分别为虚拟地址起始，页数以及一个变量地址。该系统调用要实现的内容是，检测以va开头的、被read或write过的页。结果用mask的形式映射到一个unsigned int变量中。

一些hint

- xv6已经为你设置好了系统调用需要的东西（原型、调用号），直接在sysproc.c中对应函数中填写即可。
- argint解析32bit int ;  argaddr解析64bit uint64.
- 尽管没法找到xv6 read或write页的代码（也可能是我愚钝了），但是任何访问虚存的操作，都要通过walk()找到*pte。我们跟踪的实际上是物理页被访问，其体现在该物理页PTE的flag值。所以我们找到第三级页表的对应PTE，指定PTE_A即可。PTE_A的值在PTE的结构图中可以找到。
- 综上，所以应在walk()中对物理页的PTE的PTE_A进行设置。
- 在系统调用函数中，我们不能再调用walk()找pte,否则得到的PTE都会被设置PTE_A. 所以要复用一下walk的代码。
- 为什么要在pgacess后置零PTE? 
- 根据debug来看，会将 1<<0 的PTE_A也置为1
- 思考一下，这是因为buf作为起始地址，应当在malloc时被访问过一次？这就是为什么要先pgaccess一次，目的是让这次访问设置的PTE_A被清除。