# OS Lab2 实验报告

计23

鲁逸沁

2012011314

### 练习1
---
1.	<b>实现first-fit连续物理内存分配算法。</b>

	> * 阅读memlayout.h，了解结构体Page的成员基本属性<br/>
	ref属性如注释中操作即可<br/>
	flags有两个重要的位，PG_reserved表示当前页是否可分配（空闲），1表示不可分配，0表示可分配；PG_property表示当前页是否为一段连续空闲块的头，1表示为头，0表示不为头<br/>
	如果当前页是一段连续空闲块的头页，则property属性为该空闲块中的页数<br/>
	page_link表示该页在first-fit空闲块链表中的对应的list_entry_t，可以与le2page一起作为Page和list_entry_t之间的互相转换<br/>
	将flag中的两个重要的位设为1或者0，可以分别使用SetPageReserved和ClearPageReserved，以及SetPageProperty和ClearPageProperty来实现<br/>
	> * 了解free_list的结构，其为一个类似环状的结构，从头开始依据页的地址从小到大排列，每一个空闲块在链表中保存其头页
	> * 编写函数default_init
	```
	list_init(&free_list);
    nr_free = 0;
	```
	> * 编写函数default_init_memmap<br/>
	首先将目前page开始的n页进行初始化，这里要特殊考虑第一页，因为它的flags中PG_property位要设为1，property属性要设为n
	```
	for (; p != base + n; p ++) {
		ClearPageReserved(p);
		if (p == base) {
			SetPageProperty(p);
			p->property = n;
		} else {
			ClearPageProperty(p);
		}
		set_page_ref(p, 0);
	}
	```
	> 然后将第一页加入空闲块链表free_list，并将总块数更新
	```
	nr_free += n;
    list_add_before(&free_list, &(base->page_link));
	```
	> * 编写函数default_alloc_pages<br/>
	首先从free_list里找到第一个符合条件的块fp，若没有直接退出
	```
	struct Page *fp = NULL;
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {		//	get the first page
        struct Page *p = le2page(le, page_link);
        if (p->property >= n) {
            fp = p;
            break;
        }
    }
    if (fp == NULL) return fp;
	```
	然后把连续的n个页置为不可被分配，并在free_list中删除该空闲块
	```
    for (i = 0; i < n; ++i) {							//	reserve n pages
    	struct Page *p = fp + i;
    	SetPageReserved(p);
    	ClearPageProperty(p);
    }
    list_del(le);
	```
	若该空闲块在分配n页后还有剩余remain，则将剩余部分重新加入free_list，并重置该剩余部分的头页
	```
	list_entry_t *lenext = le->next;
	if (remain > 0) {									//	if remain
    	struct Page *newp = fp + n;
    	SetPageProperty(newp);
    	newp->property = remain;
    	list_add_before(lenext, &(newp->page_link));
    }
	```
	更新总空闲块数，返回
	```
	nr_free -= n;
    return fp;
	```
	> * 编写函数default_free_pages<br/>
	先将空闲块中的n个连续页置为可用
	```
	struct Page *p = base;
    for (; p != base + n; p ++) {							//	free
        ClearPageReserved(p);
        if (p == base) {
        	SetPageProperty(p);
        	p->property = n;
        } else {
        	ClearPageProperty(p);
        }
        set_page_ref(p, 0);
    }
	```
	再在free_list中找到这个空闲块该插入的位置，位于prevp和nextp之间插入
	```
	nextp = le2page(le, page_link);
    while ((le = list_next(le)) != &free_list) {			//	find position
    	prevp = nextp;
    	nextp = le2page(le, page_link);
    	if (nextp > base) break;
    }
    list_add_before(le, &(base->page_link));				//	add
	```
	最后判断prevp和nextp的长度，如果可以合并就将后一个的长度加到前一个上，并将后一个销毁
	```
	if (base + base->property == nextp) {					//	combine next
    	base->property += nextp->property;
    	ClearPageProperty(nextp);
    	list_del(&(nextp->page_link));
    }
    if (prevp + prevp->property == base) {					//	combine prev
    	prevp->property += base->property;
    	ClearPageProperty(base);
    	list_del(&(base->page_link));
    }
    nr_free += n;
	```
2.	<b>你的first fit算法是否有进一步的改进空间？</b>
	> 如果使用平衡树代替链表来维护空闲块，则可以使查找插入和删除都做到O(logn)的复杂度。

### 练习2
---
1.	<b>实现寻找虚拟地址对应的页表项。</b>

	> * 实现get_pte函数，根据其提示实现每个步骤
	> * (1) find page directory entry<br/>
	使用提示中的PDX取出la的前10位，作为索引在pgdir目录中找到对应项pde。
	```
	pde_t *pde = &pgdir[PDX(la)];
	```
	> * (2) check if entry is not present<br/>
	根据提示，判断pde中的PTE_P位。
	```
	if (!(*pde & PTE_P)) {
	```
	> * (3) check if creating is needed, then alloc page for page table
	```
		if (!create) return NULL;
		struct Page *pz = alloc_page();
		if (pz == NULL) return NULL;
	```
	> * (4) set page reference
	```
		set_page_ref(pz, 1);
	```
	> * (5) get linear address of page<br/>
	使用提示中的page2pa来获取物理地址，再用KADDR来转换成kernel中的地址。
	```
		uintptr_t paz = page2pa(pz);
		uintptr_t kaz = KADDR(paz);
	```
	> * (6) clear page content using memset
	```
	memset(kaz, 0, PGSIZE);
	```
	> * (7) set page directory entry's permission<br/>
	将物理地址paz中的某些开关开启，放入pde中作为其内容。
	```
		*pde = paz | PTE_U | PTE_W | PTE_P;
	```
	> * (8) return page table entry<br/>
	通过PDE_ADDR函数获得没有的开关物理地址，再通过KADDR转换成kernel中的地址，这个地址就是页表地址。此时再通过PTX将la中的中间10位取出作为索引，在页表pt中查找到相应页表项的指针。
	```
	uintptr_t papde = PDE_ADDR(*pde);
	uintptr_t kapde = KADDR(papde);
	pte_t *pt = (pte_t *)kapde;
	return pt + PTX(la);
	```
2.	<b>请描述页目录项（Pag Director Entry）和页表项（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。</b>

	> * 页目录项PDE为一个字节32位，前20位为对应页表物理地址的前20位，后12位包含了一些该页目录项的信息，比如保留位、访问位、可写位、权限位等等。
	> * 页表项PTE为一个字节32位，前20位位对应页帧物理地址的前20位，后12位包含了一些该页表项的信息，比如保留位、访问位、可写位、权限位等等。
	> * 对ucore而言，我们了解这部分的含义，可以帮助正确的操作也目录项和页表项，并通过它们判断合法性，以及算出正确的地址。

3.	<b>如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？</b>

	> * CPU：产生中断，再交由操作系统处理中断
	> * 内存、硬盘：导入相应缺失页

### 练习3
---
1.	<b>释放某虚地址所在的页并取消对应二级页表项的映射。</b>

	> * 实现get_pte函数，根据其提示实现每个步骤
	> * (1) check if this page table entry is present
	```
	if (!(*ptep & PTE_P)) return;
	```
	> * (2) find corresponding page to pte
	```
	struct Page *p = pte2page(*ptep);
	```
	> * (3) decrease page reference<br/>
	(4) and free this page when page reference reachs 0<br/>
	使用提示中的page_ref_dec函数和free_page函数。
	```
	if (page_ref_dec(p) == 0) free_page(p);
	```
	> * (5) clear second page table entry<br/>
	由于PTE只是一个字节，因此清零就赋值为0即可。
	```
	*ptep = 0;
	```
	> * (6) flush tlb<br/>
	根据提示中的tlb_invalidate函数编写语句。
	```
	tlb_invalidate(pgdir, la);
	```

2.	<b>数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？</b>

	> 没分配的Page无对应关系；已分配的Page地址的前20位与某些页表项的前20位相等，对应了一个通过页机制转换之前的虚拟地址。

3.	<b>如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ </b>

	> * 页目录项和页表项放在内存前面一部分（0开始的地址），规定应用程序不许使用这段地址，页目录项和页表项做对等映射
	> * 在传入虚拟地址求物理地址时，强制分配地址相同的物理地址

### 与标准答案的差异
---
1.	练习1中，虽然使用的算法都时first-fit，但是我的实现方式与标准答案完全不同。标准答案是将所有页，无论是否是头页都放入free_list；而我只将头页放入free_list。由于这个原因，我的实现中有些情况下对PG_reserved位和PG_property位的处理与标准答案也略有不同。
2.	练习1的default_free_pages函数中，我实现空闲块前后合并的方法也和标准答案不同，更加简单和清晰。

### 本实验中重要的知识点
---
1.	空闲内存分配算法的实现。原理中讲了最先分配、最优分配、最差分配、buddy system四种，实验中实现了最先分配。
2.	页模式的机制。原理中讲了两层页表的具体页映射流程，实验中继续巩固了这一知识，并将页目录项、页表项的关系、其组成部分、虚拟地址线性地址物理地址的关系等了解的更透彻一点。

### OS原理中很重要但在实验中没有对应上的知识点
---
1.	段机制的部分。
2.	具体整个页映射流程。
3.	具体整个虚拟地址到物理地址映射的流程。
