# OS Lab5 实验报告

计23

鲁逸沁

2012011314

### 练习1
---
1.	<b>加载应用程序并执行。</b>

	> * 根据提示填写代码
	> * tf_cs should be USER_CS segment (see memlayout.h)<br/>
	tf_ds=tf_es=tf_ss should be USER_DS segmenttf_esp<br/>
	should be the top addr of user stack (USTACKTOP)<br/>
	tf_eip should be the entry point of this binary program (elf->e_entry)<br/>
	tf_eflags should be set to enable computer to produce Interrupt<br/>
	```
	tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = FL_IF;
	```
	其中最后一个值参考了答案

2.	<b>描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。</b>
	
	> * init_main中调用kernel_thread再调用do_fork复制了一个内核线程user_main
	> * 在Makefile中制定了user_main需要运行的程序X，放在宏TEST中，user_main最终调用kernel_execve产生一个SYS_exec的系统调用
	> * trap中处理了这个系统调用，调用了sys_exec，进一步调用了do_execve
	> * do_execve中完成了对旧内存空间、旧页表等的释放，调用load_icode来加载ELF格式的程序X
	> * load_icode中，完成了对ELF文件的读取、页表的建立、堆栈的建立，trap_frame的返回用户态构造
	> * 等到该用户进程（对应了一个内核线程）被调度到时，用户进程运行，context中的eip指向了forkret，在forkret中调用了trapentry.S中的__trapret，其中调用了iret。然而在load_icode中trap_frame的参数已经被设置成了用户态的相关值，则调用iret后系统进入了用户态
	> * 在load_icode中trap_frame的指令指针eip已经对应了新的程序X的第一条指令，这时程序就会从X的第一条指令开始执行，直到被调度

### 练习2
---
1.	<b>补充copy_range的实现，确保能够正确执行。</b>
	> * 根据提示补全代码
	> * (1) find src_kvaddr: the kernel virtual address of page<br/>
	(2) find dst_kvaddr: the kernel virtual address of npage
	```
	void *src_kvaddr = page2kva(page);
	void *dst_kvaddr = page2kva(npage);
	```
	> * (3) memory copy from src_kvaddr to dst_kvaddr, size is PGSIZE
	```
	memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
	```
	> * (4) build the map of phy addr of  nage with the linear addr start
	```
	ret = page_insert(to, npage, start, perm);
	```
	> * 观察发现在trap.c的trap_dispatch中也有一段lab5需要填充的代码，仔细观察发现时处理时钟中断时，需要将当前进程的need_resched属性置位1，这个可能时lab5中最初级的调度算法，即每个时间片调度一次

2.	<b></b>

### 练习3
---
1.	<b></b>
	
	
2.	<b></b>

### 与标准答案的差异
---
1.	
2.	

### 本实验中重要的知识点
---
1.	
2.	
3.	
4.	

### OS原理中很重要但在实验中没有对应上的知识点
---
1.	
2.	
3.	
