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


### 练习2
---
1.	<b>你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。</b>
	> * 根据提示中需要初始化的内容进行初始化
	> * 1. call alloc_proc to allocate a proc_struct<br/>
	如果没有成功alloc_proc成功，则如同上面代码return的一样，跳到fork_out
	```
	if ((proc = alloc_proc()) == NULL) {
    	goto fork_out;
    }
	```
	> * 2. call setup_kstack to allocate a kernel stack for child process<br/>
	如果没有成功，就要将之前alloc_proc的资源释放，这时就需要跳到bad_fork_cleanup_proc
	```
	if (setup_kstack(proc)) {
    	goto bad_fork_cleanup_proc;
    }
	```
	> * 3. call copy_mm to dup OR share mm according clone_flag<br/>
	如果没有成功，就要将之前alloc_proc的资源释放，并且将栈资源也释放，这时就需要跳到bad_fork_cleanup_kstack
	```
	if (copy_mm(clone_flags, proc)) {
    	goto bad_fork_cleanup_kstack;
    }
	```
	> * 4. call copy_thread to setup tf & context in proc_struct<br/>
	5. insert proc_struct into hash_list && proc_list<br/>
    6. call wakup_proc to make the new child process RUNNABLE
	```
	copy_thread(proc, stack, tf);
    hash_proc(proc);
    list_add(&proc_list, &(proc->list_link));
    wakeup_proc(proc);
	```
	> * 7. set ret vaule using child proc's pid
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
	> *

### 与标准答案的差异
---
1.	标准答案中都有assert错误信息，我并没有意识到这一点

### 本实验中重要的知识点
---
1.	了解虚拟存储的实现方式和大致框架
2.	了解了如何判断、处理页缺失异常，以及换入换出操作
3.	了解FIFO算法的具体实现

### OS原理中很重要但在实验中没有对应上的知识点
---
1.	其他页替换算法
2.	具体换入换出操作的实现流程
3.	整个流程的具体线索，从页缺失，到中断，到调用虚拟存储，到换入换出，到重新执行指令
