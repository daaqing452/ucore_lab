# OS Lab7 实验报告

计23

鲁逸沁

2012011314

### 练习0
---
1.	<b>trap.c的trap_dispatch函数</b>

	>
	```
	ticks++;
	run_timer_list();
	```

### 练习1
---
1.	<b>在实验报告中给出内核级信号量的设计描述，并说其大致执行流流程</b>

	> * 信号量主要对应P操作和V操作
	> * 信号量数据结构
	```
	typedef struct {
		int value;						// 资源数目
		wait_queue_t wait_queue;		// 等待队列，没有申请到资源的进程放在里面
	} semaphore_t;
	```
	> * P操作见sync/sem.c的__down函数
	```
	static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
		bool intr_flag;
		local_intr_save(intr_flag);
		if (sem->value > 0) {										//　判断资源数目是否>0
		    sem->value --;											// 有资源则获取资源
		    local_intr_restore(intr_flag);
		    return 0;
		}
		wait_t __wait, *wait = &__wait;
		wait_current_set(&(sem->wait_queue), wait, wait_state);		//　没有资源则将当前进程变成sleep，加入等待队列
		local_intr_restore(intr_flag);
		schedule();													// 调度，直到资源获得时被唤醒，直接获得资源
		local_intr_save(intr_flag);
		wait_current_del(&(sem->wait_queue), wait);					// 从等待队列里删除当前线程
		local_intr_restore(intr_flag);
		if (wait->wakeup_flags != wait_state) {
		    return wait->wakeup_flags;
		}
		return 0;
	}
	```
	> * V操作见sync/sem.c的__up函数
	```
	static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
		bool intr_flag;
		local_intr_save(intr_flag);
		{
		    wait_t *wait;
		    if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {	// 如果当前进程不在等待队列中，则说明它释放了资源
		        sem->value ++;												// 资源数目加一
		    }
		    else {
		        assert(wait->proc->wait_state == wait_state);
		        wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);		// 如果当前进程在等待队列中，则唤醒它，立刻获得资源
		    }
		}
		local_intr_restore(intr_flag);
	}
	```

2.	<b>在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同</b>

	> 用户态进程和线程信号量的实现可以仿照内核线程的信号量，在用户态虚拟一个信号量，但实际上是内核态管理的。当需要进行信号量操作的时候，通过中断从用户态跳转到内核态，维护内核态管理的信号量，再更新用户态的信号量。

### 练习2
---
1.	<b>完成内核级条件变量和基于内核级条件变量的哲学家就餐问题</b>
	
	> * sync/moniter.c中的cond_signal函数
	```
	if(cvp->count > 0) {
	   cvp->owner->next_count++;
	   up(&(cvp->sem));
	   down(&(cvp->owner->next));
	   cvp->owner->next_count--;
	}
	```
	> * sync/moniter.c中的cond_wait函数
	```
	cvp->count++;
    if(cvp->owner->next_count > 0)
    	up(&(cvp->owner->next));
    else
    	up(&(cvp->owner->mutex));
    down(&(cvp->sem));
    cvp->count--;
	```
	> * sync/check_sync.c中的phi_take_forks_condvar函数
	```
	state_condvar[i] = HUNGRY;
	phi_test_condvar(i);
	if (state_condvar[i] != EATING) {
		cond_wait(&mtp->cv[i]);
	}
	```
	> * sync/check_sync.c中的phi_put_forks_condvar函数
	```
	state_condvar[i] = THINKING;
	phi_test_condvar(LEFT);
	phi_test_condvar(RIGHT);
	```

### 与标准答案的差异
---
1.	在实现哲学家问题的代码中有些细节有差异

### 本实验中重要的知识点
---
1.	信号量和管程的原理及实现
2.	哲学家问题
3.	同步互斥的框架

### OS原理中很重要但在实验中没有对应上的知识点
---
1.	管程的具体实现
2.	同步互斥在原lab上添加的接口
