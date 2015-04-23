# OS Lab4 实验报告

计23

鲁逸沁

2012011314

### 练习1
---
1.	<b>分配并初始化一个进程控制块。</b>
	> * 根据提示初始化该初始化的内容
	> * below fields in proc_struct need to be initialized<br/>
			enum proc_state state;                      // Process state<br/>
			int pid;                                    // Process ID<br/>
			int runs;                                   // the running times of Proces<br/>
     *       uintptr_t kstack;                           // Process kernel stack<br/>
     *       volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?<br/>
     *       struct proc_struct *parent;                 // the parent process<br/>
     *       struct mm_struct *mm;                       // Process's memory management field<br/>
     *       struct context context;                     // Switch here to run process<br/>
     *       struct trapframe *tf;                       // Trap frame for current interrupt<br/>
     *       uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)<br/>
     *       uint32_t flags;                             // Process flag<br/>
     *       char name[PROC_NAME_LEN + 1];               // Process name<br/>
	```
	
	```


### 练习2
---
1.	<b>完成vmm.c中的do_pgfault函数。</b>

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
