# OS Lab6 实验报告

计23

鲁逸沁

2012011314

### 练习0
---
1.	<b>proc.c的alloc_proc函数</b>
	
	```
	proc->rq = NULL;
    proc->run_link.prev = proc->run_link.next = NULL;
    proc->time_slice = 0;
    proc->lab6_run_pool.left = proc->lab6_run_pool.right = proc->lab6_run_pool.parent = NULL;
    proc->lab6_stride = 0;
    proc->lab6_priority = 0;
	```

2.	<b>trap.c的trap_dispatch函数</b>

	```
	ticks++;
	sched_class_proc_tick(current);
	```

### 练习1
---
1.	<b>请理解并分析sched_class中各个函数指针的用法，并接合Round Robin调度算法描ucore的调度执行过程。</b>

	> sched_class中各个函数指针的用法
	> * init：初始化进程队列
	> * enqueue：将一个进程加入进程队列
	> * dequeue：将一个进程从进程队列中移出
	> * pick_next：挑选出下一个调度的进程
	> * proc_tick：时钟中断的时候调用，可以用来处理进程调度
	
	> ucore的调度执行过程
	> 1. 初始化一个进程队列rq
	> 2. 将刚开始活动的进程插入队列
	> 3. 等待时间片结束，利用RR算法选择下一个要调度的进程
	> 4. 将当前运行进程转换
	> 5. 若有进程退出，从队列中删除

2.	<b>在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计。</b>

	> * 维护多个队列，每个队列的优先级不同，MAX_TIME_SLICE不同
	> * 调度时，利用算法对不同队列中的进程按次序进行调度
	> * 建立一些哈希表或指针等来维护某个进程属于哪个队列

### 练习2
---
0.	<b>完成常量赋值</b>
	
	>
	```
	#define BIG_STRIDE 0x7FFFFFFF
	```
	
1.	<b>完善stride_init</b>

	> * 根据提示<br/>
	(1) init the ready process list: rq->run_list<br/>
    (2) init the run pool: rq->lab6_run_pool<br/>
    (3) set number of process: rq->proc_num to 0
    ```
    list_init(&(rq->run_list));
	rq->lab6_run_pool = NULL;
	rq->proc_num = 0;
    ```

2.	<b>完善stride_enqueue</b>

	> * 根据提示<br/>
    (1) insert the proc into rq correctly<br/>
    NOTICE: you can use skew_heap or list. Important functions<br/>
    	skew_heap_insert: insert a entry into skew_heap<br/>
    	list_add_before: insert  a entry into the last of list<br/>
    (2) recalculate proc->time_slice<br/>
    (3) set proc->rq pointer to rq<br/>
    (4) increase rq->proc_num
    ```
    rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
	if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
		proc->time_slice = rq->max_time_slice;
	}
	proc->rq = rq;
	rq->proc_num++;
    ```
    
3.	<b>完善stride_dequeue</b>
	
	> * 根据提示<br/>
	(1) remove the proc from rq correctly<br/>
    NOTICE: you can use skew_heap or list. Important functions<br/>
    	skew_heap_remove: remove a entry from skew_heap<br/>
    	list_del_init: remove a entry from the  list<br/>
    ```
    rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
	rq->proc_num--;
    ```

4.	<b>完善stride_pick_next</b>

	> * 根据提示<br/>
	(1) get a  proc_struct pointer p  with the minimum value of stride
    	(1.1) If using skew_heap, we can use le2proc get the p from rq->lab6_run_poll
    	(1.2) If using list, we have to search list to find the p with minimum stride value
    (2) update p;s stride value: p->lab6_stride
    (3) return p
    ```
    if (rq->lab6_run_pool == NULL) return NULL;
	struct proc_struct *p = le2proc(rq->lab6_run_pool, lab6_run_pool);
	#define max(x, y) ((x)>(y)?(x):(y))
	p->lab6_stride += BIG_STRIDE / max(p->lab6_priority, 1);
	return p;
    ```

5.	


### 练习3
---
1.	<b></b>
	
	
2.	<b></b>

### 与标准答案的差异
---
1.	

### 本实验中重要的知识点
---
1.	

### OS原理中很重要但在实验中没有对应上的知识点
---
1.	
