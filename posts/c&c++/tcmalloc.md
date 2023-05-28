### **TCMalloc 相比于 PTMalloc的优点**
* TCMalloc比glibc 2.3 malloc （又叫做ptmalloc2，这个库可单独使用）更快。对于小对象，在2.8 GHz 奔腾4上执行malloc/free对，ptmalloc2需要大约300纳秒。相同操作对于TCMalloc实现仅需大约50纳秒。速度对于一个malloc库实现来说非常重要，因为如果malloc库不够快，应用程序编写者倾向于在其之上编写自己的自定义空闲列表。这可能会导致额外的复杂性，并且除非应用程序编写者非常小心适当地调整空闲列表并从中挑选出合适空闲对象，否则会增加内存使用量。

* TCMalloc还可以减少多线程程序的锁争用。对于小对象，几乎没有竞争。对于大对象，TCMalloc尝试使用更细粒度和高效的自旋锁。ptmalloc2也通过使用每个线程的arena来减少锁争用，但是ptmalloc2在使用每个线程的arena时存在一个很大的问题,在ptmalloc2中，空闲内存永远不能从一个arena移动到另一个arena中。这可能导致大量空间浪费。例如，在某个Google应用程序中，第一阶段将为其数据结构分配约300MB的内存。当第一阶段完成后，在同一地址空间中启动第二阶段，比如在另一个线程中开始运行第二阶段的处理，如果此时使用的arena不是与第一阶段使用同一个arena，则第一阶段的空闲内存不会被复用，并且会向地址空间添加另外300MB ，这会导致内存使用出现阶段性的暴涨。

* TCMalloc的另一个好处是分配小对象时，其空间利用更高效。例如，可以分配N个8字节对象，仅使用大约8N * 1.01字节的空间。即，只有百分之一的额外空间开销。ptmalloc2需要为每个对象额外多分配size_t个字节的头部，并且最起码将大小舍入到8字节的倍数，并最终使用16N个字节。事实上，在64位机上，ptmalloc2最小分配的chunk大小默认是32字节，小内存的分配明显空间浪费严重。

### **使用**
在编译的时候加上 -ltcmalloc 连接该库即可。

当然也可以使用LD_PRELOAD 动态连接器的预加载，如果你无法编译你的c/c++程序的话


tcmalloc库包括在gperftools库中，这个库中也包括heap profiler and checker 库，如果只需要tcmalloc,不需要别的，只需要连接libtcmalloc_minimal 库即可。当然如果是从源码编译，需要使用./configure --enableXXX --disableXXX等配置选项。


### **概览**
TCMalloc为每个线程分配一个本地缓存thread-local cache。小的内存分配从thread-local cache中满足。内存对象会在需要时从central data structures 移动到thread-local cache 中，同时会定期将内存从thread-local cache回收回central data structures中

小于等于32K的叫做小对象，大对象直接从central heap中使用页面级分配器分配，大对象被对齐到page size, 占用整数个page.

一系列页面可以被划分为一系列大小相等的小对象。例如，一个页面（4K）可以被划分为32个大小为128字节的对象序列。

### **Small Object Allocation**

每个小对象大小都映射到大约170个size-classes之一。例如，范围在961到1024字节内的所有小对象的分配都会被舍入为1024。这些size-classes之间的间隔是根据8字节、16字节、32字节等方式进行划分。对于大于或等于2K的size-classes，最大间距为256字节。

每个thread-local cache中都包括每个size-class空闲对象的单链表。

在分配小对象时：（1）我们将其大小映射到相应的size-class中。（2）查找当前线程的thread-local cache中相应的空闲列表。（3）如果空闲列表不为空，我们从列表中删除第一个对象并返回它。 

当遵循此快速路径时，TCMalloc根本不会获取任何锁。这有助于显着加快分配速度，因为在2.8 GHz Xeon处理器上进行一次lock/unlock对大约需要100纳秒。


If the free list is empty: (1) We fetch a bunch of objects from a central free list for this size-class (the central free list is shared by all threads). (2) Place them in the thread-local free list. (3) Return one of the newly fetched objects to the applications.

If the central free list is also empty: (1) We allocate a run of pages from the central page allocator. (2) Split the run into a set of objects of this size-class. (3) Place the new objects on the central free list. (4) As before, move some of these objects to the thread-local free list.

### **Large Object Allocation**
一个大对象（> 32K）的大小会被对齐到页面大小（4K），并由central page heap处理。central page heap 也是一组空闲列表。对于i <256，第k个空闲列表中的每个对象的大小是k个页面。第256个空闲列表中每个对象的页面>= 256

通过查找第k个空闲列表来满足k页的分配。如果该空闲列表为空，则我们会查找下一个空闲列表，以此类推。最终，如果必要的话，我们会查找最后一个空闲列表。如果这仍然失败了，我们将从系统中获取内存。

如果分配k个页面，从>k个空闲列表中分配，剩下的页面被插入到适当的空闲页面列表中。

### **Spans**
central page heap 管理的连续页面的由Span对象表示。A span 可以是已分配或空闲的。如果为空闲，则该span 是在空闲页面列表中。如果已分配，则它可能是已移交给应用程序的大型对象，也可能是被拆分为一系列小对象由tcmalloc管理。如果拆分为小对象，则在span中记录了对应的size-class


A central array indexed by page number can be used to find the span to which a page belongs. For example, span a below occupies 2 pages, span b occupies 1 page, span c occupies 5 pages and span d occupies 3 pages.

32位机，共有2^20 个4K的页面，central array的大小为2^20 *4 ，4MB.
如果是64位机器，we use a 3-level radix tree instead of an array to map from a page number to the corresponding span pointer


### **Deallocation**
当一个对象被应用程序释放时，我们首先根据其地址，找到其所在的页面（地址最后的12位mask掉即可），然后根据其页面，从central array中找到其所在的span。The span tells us whether or not the object is small, and its size-class if it is small. 如果是小对象，将其插入到thread-local cache的空闲对象列表中。 如果thread-local-cache的大小超过了预定义(2MB by default)的限制，we run a garbage collector that moves unused objects from the thread cache into central free lists.


If the object is large, the span tells us the range of pages covered by the object. Suppose this range is [p,q]. We also lookup the spans for pages p-1 and q+1. If either of these neighboring spans are free, we coalesce them with the [p,q] span. The resulting span is inserted into the appropriate free list in the central page heap.

### **Central Free Lists for Small Objects**
每个size-class都有其对应的central free list 。central free list由two-level data structure表示，a set of spans, and a linked list of free objects per span.

An object is allocated from a central free list by removing the first entry from the linked list of some span. (If all spans have empty linked lists, a suitably sized span is first allocated from the central page heap.)

An object is returned to a central free list by adding it to the linked list of its containing span. If the linked list length now equals the total number of small objects in the span, this span is now completely free and is returned to the page heap.

### **Garbage Collection of Thread Caches**
A thread cache is garbage collected when the combined size of all objects in the cache exceeds 2MB. The garbage collection threshold is automatically decreased as the number of threads increases so that we don't waste an inordinate amount of memory in a program with lots of threads.
We walk over all free lists in the cache and move some number of objects from the free list to the corresponding central list.

The number of objects to be moved from a free list is determined using a per-list low-water-mark L. L records the minimum length of the list since the last garbage collection. Note that we could have shortened the list by L objects at the last garbage collection without requiring any extra accesses to the central list. We use this past history as a predictor of future accesses and move L/2 objects from the thread cache free list to the corresponding central free list. This algorithm has the nice property that if a thread stops using a particular size, all objects of that size will quickly move from the thread cache to the central free list where they can be used by other threads.

### **Performance Notes**

### **注意事项**
* 如果多线程不是使用的pthread库，tcmalloc可能不会正确工作。It should work on Linux using glibc.
* tcmalloc 相比别的mallocs，可能会消耗更多的内存，但是它一般不会出现像ptmalloc那种的突然内存暴涨现象。 At startup TCMalloc allocates approximately 6 MB of memory
* TCMalloc currently does not return any memory to the system.
* 不要试图在运行过程中通过动态链接库加载tcmalloc,这会导致之前程序中使用别的malloc库分配的内存交给tcmalloc来释放，tcmalloc无法处理这种内存。


本来我还打算看代码，现在我都不想看了，直接用就是了！

tcmalloc比ptmalloc设计好太多了！