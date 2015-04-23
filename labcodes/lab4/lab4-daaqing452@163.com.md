# OS Lab4 实验报告

计23

鲁逸沁

2012011314

### 练习1
---
1.	<b>alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结构，用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。</b>

	> * 根据提示中需要初始化的内容进行初始化
	> * below fields in proc_struct need to be initialized<br/>
	enum proc_state state;                      // Process state<br/>
	int pid;                                    // Process ID<br/>
	int runs;                                   // the running times of Proces<br/>
	uintptr_t kstack;                           // Process kernel stack<br/>
	volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?<br/>
	struct context context;                     // Switch here to run process<br/>
	struct trapframe *tf;                       // Trap frame for current interrupt<br/>
	uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)<br/>
	uint32_t flags;                             // Process flag<br/>
	char name[PROC_NAME_LEN + 1];               // Process name<br/>
	
	```
	proc->state = PROC_UNINIT;
	proc->pid = -1;
	proc->runs = 0;
	proc->kstack = 0;
	proc->need_resched = 0;
	proc->parent = NULL;
	proc->mm = NULL;
	memset(&(proc->context), 0, sizeof(struct context));
	proc->tf = NULL;
	proc->cr3 = boot_cr3;
	proc->flags = 0;
	memset(proc->name, 0, sizeof(proc->name));
	```

2.	<b>请说明proc_struct中```struct context context```和```struct trapframe *tf```成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）</b>
	
	> * 查看struct context的结构
	```
	struct context {
		uint32_t eip;
		uint32_t esp;
		uint32_t ebx;
		uint32_t ecx;
		uint32_t edx;
		uint32_t esi;
		uint32_t edi;
		uint32_t ebp;
	};
	```
	可以发现它是用来保存寄存器的，同时也包含了堆栈的信息，总之就是保存了进程运行时的状态。并且在调度过程proc_run函数中出现了代码
	```
	switch_to(&(prev->context), &(next->context));
	```
	则可以判断它时用来保存进程运行状态的数据结构
	> * struct trapframe时调用系统中断时使用的数据结构，它在kernel_thread函数和copy_thread函数里完成了一些初始化工作，因此它应该是为了后续某个需要系统中断的工作准备的。有一些代码有一点信息
	```
	tf.tf_regs.reg_ebx = (uint32_t)fn;
	```
	这个应该是指定了新进程需要加载的函数指针。还有
	```
	proc->tf->tf_esp = esp;
	```
	记录了它的堆栈。因此我猜测这个数据结构时用来fork新的内核线程后，加载新用户进程，产生系统中断而用的
	

### 练习2
---
1.	<b>你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。</b>

	> * 根据提示中需要初始化的内容进行初始化
	> * (1) call alloc_proc to allocate a proc_struct<br/>
	如果没有成功alloc_proc成功，则如同上面代码return的一样，跳到fork_out
	```
	if ((proc = alloc_proc()) == NULL) {
    	goto fork_out;
    }
	```
	> * (2) call setup_kstack to allocate a kernel stack for child process<br/>
	如果没有成功，就要将之前alloc_proc的资源释放，这时就需要跳到bad_fork_cleanup_proc
	```
	if (setup_kstack(proc)) {
    	goto bad_fork_cleanup_proc;
    }
	```
	> * (3) call copy_mm to dup OR share mm according clone_flag<br/>
	如果没有成功，就要将之前alloc_proc的资源释放，并且将栈资源也释放，这时就需要跳到bad_fork_cleanup_kstack
	```
	if (copy_mm(clone_flags, proc)) {
    	goto bad_fork_cleanup_kstack;
    }
	```
	> * (4) call copy_thread to setup tf & context in proc_struct<br/>
	(5) insert proc_struct into hash_list && proc_list<br/>
    (6) call wakup_proc to make the new child process RUNNABLE<br/>
	```
	copy_thread(proc, stack, tf);
    hash_proc(proc);
    list_add(&proc_list, &(proc->list_link));
    wakeup_proc(proc);
	```
	> * (7) set ret vaule using child proc's pid
	```
	ret = proc->pid;
	```
	不过这时可以发现，proc的pid并没有被赋值，因此之前要进行一些初始化，包括对其pid、父进程、nr_process等的处理
	```
	proc->pid = get_pid();
    proc->parent = current;
    nr_process++;
	```
	这段代码插入的位置可以通过不断调试位置make qemu查看是否正确
	> * 最终代码
	```
	if ((proc = alloc_proc()) == NULL) {
    	goto fork_out;
    }
    if (setup_kstack(proc)) {
    	goto bad_fork_cleanup_proc;
    }
    if (copy_mm(clone_flags, proc)) {
    	goto bad_fork_cleanup_kstack;
    }
    proc->pid = get_pid();
    proc->parent = current;
    nr_process++;
    copy_thread(proc, stack, tf);
    hash_proc(proc);
    list_add(&proc_list, &(proc->list_link));
    wakeup_proc(proc);
    ret = proc->pid;
	```

2.	<b>请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。</b>

	> 是。<br/>
	通过查看get_pid函数，可以发现，它每次分配pid的时候，都扫描了整个进程队列，取出从0开始最小的没用过的数字作为新的pid，因此整个进程队列里不会出现两个相同的pid，即每次fork新进程的时候都能获得唯一的id。<br/>
	但是，根据这种算法，一个被释放的进程和一个新fork的进程可能会拥有相同的pid。

### 练习3
---
1.	<b>在本实验的执行过程中，创建且运行了几个内核线程？</b>
	
	> 创建且运行了两个内核线程，分别对应idleproc和initproc
	
2.	<b>语句```local_intr_save(intr_flag);....local_intr_restore(intr_flag);```在这里有何作用？请说明理由。</b>

	> 这段语句在这里是禁用了硬件中断，保证在进程切换的时间里没有硬件中断产生，保证进程切换的顺利进行。因为这一段代码涉及特权级的改变、堆栈的改变、寄存器的备份和恢复，如果产生一个中断，那么寄存器和堆栈的信息就会在中途被破坏，进程切换就不能完成。

### 与标准答案的差异
---
1.	标准答案中在do_fork函数的加入进程队列的阶段，也禁用了中断
2.	标准答案中在do_fork函数里一些语句的顺序与我的不一样

### 本实验中重要的知识点
---
1.	了解了进程生命周期，从创建、等待、被调度、运行、结束等阶段
2.	了解了ucore操作系统中进程的初始化工作，包括idelproc和initproc的创建
3.	了解了do_fork函数的工作原理，了解进程复制的相关过程
4.	了解了调度的过程，如果切换进程的大致过程

### OS原理中很重要但在实验中没有对应上的知识点
---
1.	新内核线程的创建尝试
2.	进程切换的具体细节
3.	一些关键函数，比如kernel_thread函数
