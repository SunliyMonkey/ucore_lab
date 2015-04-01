# report



---

# Lab2 report

## [练习1]实现first-fit 连续物理内存分配算法
* **设计过程**
代码中，其实已经基本实现整个功能。主要注意下标志位的设定。然后在free_pages()的函数当中，需注意“有序插入”，在不满足前两个条件下，并且base<p的情况下，将base插入到p之前。

* **你的first fit算法是否有进一步的改进空间**
有，在查找插入位置的时候，采用的顺序查找方式，相对开销较大，可以采用“跳表”的方式维护整个链接结构，同时能够降低插入开销。

## [练习2]实现寻找虚拟地址对应的页表项
* **设计过程**
首先在页目录表中找到二级页表的入口，如果不存在，在允许创建的情况下，申请新的页表，将页表的物理地址和标志位写入页目录的表项中，然后返回二级页表中对应表项的虚拟地址。

* **如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？**
CPU会把产生异常的线性地址存储在CR2中，并且把表示页访问异常类型的值保存在中断栈中，然后执行中断相关的操作。

* **如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问 异常，请问硬件要做哪些事情？**
返回用户访问失败（这里不确定）

## [练习3]释放某虚地址所在的页并取消对应二级页表项的映射
* **设计过程**
    如果在二级页表中找到对应的表项，则删除对应的值，并且减少该页的引用次数，同时更新快表。

* **数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目 录项和页表项有无对应关系？如果有，其对应关系是啥？**
有关系，页目录表和页表中的每一项均对应的一个具体物理页。而boot_cr3对应了页目录表所在的物理页。

* **如果希望虚拟地址与物理地址相等，则需要如何修改lab2**
在get_pte中，当二级页表对应的表现不存在时，需要新申请物理页作为该虚拟页的对应，在这里可以把虚拟地址传入，强制要求alloc_page为其分配物理地址等于该虚拟地址的page即可。

