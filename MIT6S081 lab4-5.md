## Lab4 traps

![img](https://picx.zhimg.com/80/v2-7fbc970c68ce151ee618859611e580bd_720w.jpeg?source=d16d100b)



### 4.1 RISC-V assembly

略

### 4.2 Backtrace

> 在 kernel/printf.c 中实现一个 backtrace() 函数。 在sys_sleep中插入对这个函数的调用，然后运行bttest，调用sys_sleep。 您的输出应如下所示： backtrace： 0x0000000080002cda 0x0000000080002bb6 0x0000000080002898

本节实验要求实现backtrace()并在sys_sleep时调用，用于搞清楚函数调用栈。实验的目的是帮助我们更好地理解函数调用栈的结构。

![img](https://picx.zhimg.com/80/v2-f5314cf51d36d87d8bae7bc710beea95_720w.png?source=d16d100b)

- 函数栈帧在用户空间的栈中，而栈在xv6里默认只占一页。这是用来判断frame pointer是否到终点的方法
- printf("%p", arg);    我觉得可以直接理解为，以十六进制打印变量的值
- 如果arg是指针，那么其值就是保存的地址，打印出来的就是地址的十六进制表示
- 如果arg是uint64等，那么打印的其实也是pointer值的十六进制表示。

```c
void backtrace(void){
  uint64 fp = r_fp();
  uint64 up_limit = PGROUNDUP(fp),down_limit = PGROUNDDOWN(fp);
  printf("backtrace:\n");
  while(fp>=down_limit && fp<=up_limit){
    printf("%p\n",*(uint64*)(fp-8));
    fp = *(uint64*)(fp-16);
  }
}
```

### 4.3 Alarm

> 在本练习中，您将向 xv6 添加一项功能，该功能会在进程使用 CPU 时间时定期向其发出警报。 这对于想要限制它们占用多少 CPU 时间的受计算限制的进程，或者对于想要计算但也想采取一些定期操作的进程来说可能很有用。 更一般地说，您将实现一种原始形式的用户级中断/故障处理程序； 例如，您可以使用类似的方法来处理应用程序中的页面错误。 如果您的解决方案通过了警报测试和用户测试，那么它就是正确的。

本节实验要求实现系统调用sigalarm(ticks, fn)和sigreturn. 其行为就是，统计时钟ticks的个数，当达到设定值时会调用用户函数fn.

一些提示：

- 时钟中断是模拟不断产生的，其在kernel/trap.c中处理。可以定位到which_dev == 2处。实验指导给出的思想是：**系统调用sigalarm的作用，就是将ticks和fn送入内核中**。那么，我们在proc结构体中记录下该值。同时还要记录当前的tick数，到达设定值时再清零。这个变量同样要存到proc结构体中。

```c
  if(which_dev == 2){
    if(p->is_handling == 0){
        if(p->intervals!=0||p->handler!=0){
        p->ticks_cnt++;
        if(p->ticks_cnt == p->intervals){
          p->ticks_cnt = 0;
          p->is_handling = 1;
          memmove(&p->tmp_trapframe,p->trapframe,sizeof(struct trapframe));
          p->trapframe->epc = p->handler;
        }
      }
    }
    yield();
  }
```

- 如何调用用户函数？我们传递的用户函数的地址，其是用户地址空间中的虚拟地址，该地址在内核态是无效的。所以我们要**返回用户态，在用户态运行. 考虑到我们在处理trap最后要回到用户态，于是做法便是将fn的地址送入epc. 在sret时，就会将epc->pc, 从而返回到fn函数的地址。**
- **至此，Alarm部分的test0便完成了。**
- 但是，有没有感觉有一些不对劲？是的，我们覆盖了原有的epc，导致我们无法再回到时钟中断发生的地址继续执行代码。同时，即使我们回到了原epc地址，我们的寄存器值也因为fn的运行导致截然不同，这显然也不符合要求。因此我们需要对各寄存器进行保存。
- 最简单的做法就是，在proc里再新增一个trapframe结构体用于备份。
- (1) 在覆盖epc前，保存trap->frame.  (memmove)
- (2) Xv6已经为你考虑好了解决方案：fn函数的最后会调用 sigreturn()系统调用，这意味着你可以在该系统调用的实现中，加入对trapframe的恢复。这样, sigreturn作为系统调用也会返回到用户态，我们将时钟中断的trapframe覆盖此时的trapframe, 那么就实现了状态的切回。
- **至此，Alarm部分的test1便完成了。**
- 最后test2要求不得在handle的途中再次触发fn, 实际上我们只需要用一个变量is_handling进行加锁即可。Alarm部分的test2也完成了。
- **xv6 2022fall 补充test3:**
  - 要求sigreturn不能改变a0的值
  - 代码流程是, sigreturn先恢复trapframe, 之后在syscall()中再将返回值写入到trapframe的a0. 
  - 那就让sigreturn的返回值等于想要trapframe的a0就行了嘛.

```c
uint64
sys_sigreturn(void){
  memmove(myproc()->trapframe,&(myproc()->tmp_trapframe),sizeof(struct trapframe));
  myproc()->is_handling = 0;
  return myproc()->trapframe->a0;
}
```

### 4.4 收获

- gdb用得越来越熟练了
- printf %p的含义
- 加深了对函数调用栈帧的理解
- 了解了如何利用sepc来实现中断返回时的定向、非原跳转。
- 加深用户地址空间地址、页表转换的理解，用户态传到内核态的地址，在内核态中不能直接使用。通常的解决方法是内核里再创建变量，然后copyin/out. 





## Lab5 Copy-on-Write Fork for xv6 

![img](https://picx.zhimg.com/80/v2-6077d39baff1111123e6c9d1784dd660_720w.jpeg?source=d16d100b)



### 5.1 流程

这大概是我头一次在实验面前栽跟头吧，有点恶心

实验的要求就是，实现fork的copy-on-write.

整理思路：

**朴素：**

1. 不能在fork时直接分配物理页、复制了，而是应该在子进程的页表中复制父进程的PTE.  同时将PTE的PTE_W位清除。 定位到uvmcopy().
2. 之后我们访问到这些PTE时会触发page fault. 由于是用户态，所以定位到usertrap(). 识别到scause == 15. 在这里需要重新分配物理页(kalloc)、拷贝原物理页内容(memmove)、修改PTE的flag。
3. copyout()是内核触发页错误的方式，不归usertrap()管。因此也要复用usertrap()里关于page fault的处理。通过设置PTE_COW位(1<<8L)，并检验将要访问的物理页是否是COW且非W，来确定是否要采取措施。

**计数：**

1. 由于一个物理页可能被多个页表映射。因此不能轻易free掉。根据讲义，要以pa为idx维护一个引用数数组。思来想去在kalloc.c中定义全局变量比较合适。
2. **ref=1:**    kalloc时，该page的reference被设置为1.
3. **ref-- :**     kfree时，先--reference, 如果==0则free，否则跳过. 【所以要在cow pagefault分配新物理页后，对原物理页进行一次kfree】
4. **ref++:**    uvmcopy时，page的reference++. 可以在kalloc中将对应的操作封装，然后在vm.c中调用。
5.  bug: 由于kinit时会调用freerange, 将所有页面都kfree(加入到freelist中)。这样初始各页的ref为-1，因此要在freerange中将各页面设置ref为1再调用kfree.

**再想：**

1. 页reference数组作为全局变量，会被许多进程共享。因此需要加锁。比葫芦画瓢，用一个结构体将spinlock和reference变量打包。在kinit时初始化锁。在任何情况下访问ref都要acquire.
2. 在cow page fault处理时，面对kalloc失败的情况，设置p->killed=1或者exit.
3. 一个优化：在cow page fault时，如果当前页只有一个ref, 那么直接修改pte的flags即可。

**走过的弯路：**

一开始没有想对ref值变动的时机。

将++的时机放在了uvmmap(), 将--的时机放在了uvmunmap().  

对于前者，我原本设计的方案是kalloc时初始化为0，uvmmap时++.  但是忽略了还有kvmmap, 凭什么人家物理页只能让你用户页表映射啊。

对于后者，同理。

**部分代码：**

```c
extern struct{
  struct spinlock lock;
  int reference_cnt[PHYSTOP/PGSIZE];
} kref;

// 由usertrap的scause == 15调用
int cowfault(pagetable_t pagetable, uint64 va){
  if(va >= MAXVA)
    return -1;
  if(va % PGSIZE != 0)
    return -1;
  pte_t* pte;
  if((pte = walk(pagetable,va,0)) == 0)
    return -1;
  if(!(*pte & PTE_COW) || !(*pte & PTE_V))
    return -1;
  if((*pte & PTE_W))
    return -1;
  uint64 pa = PTE2PA(*pte);
  acquire(&kref.lock);
  if(kref.reference_cnt[pa/PGSIZE] > 1){
    release(&kref.lock);
    char* mem;
    if((mem = kalloc()) == 0){
      return -1;
    }
    memmove(mem, (char*)pa, PGSIZE);
    *pte = PA2PTE((uint64)mem) | PTE_V | PTE_R | PTE_X | PTE_W | PTE_U;
    kfree((void*)pa);
  }
  else{
    release(&kref.lock);
    *pte = PA2PTE(pa) | PTE_V | PTE_R | PTE_X | PTE_W | PTE_U;
  }
  return 0;
}

int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    *pte &= ~PTE_W;
    *pte |= PTE_COW;
    flags = PTE_FLAGS(*pte);
    increment_ref(pa);
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```



### 5.2 收获

1. 多用panic.
2. 高屋建瓴的设计。
3. va在walk页表前，要先向下对齐。