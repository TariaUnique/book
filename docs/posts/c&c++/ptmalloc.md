GNU C library使用的是ptmalloc的内存管理程序

### **version**
glibc2.17 


### **chunk**

用户请求分配的空间在 ptmalloc 中都使用一个 chunk 来表示,chunk为用户需要使用的内存空间，加上ptmalloc内部管理chunk的metadata
```c
#ifndef INTERNAL_SIZE_T
#define INTERNAL_SIZE_T size_t
#endif

struct malloc_chunk {

  INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;

  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};

typedef struct malloc_chunk* mchunkptr;

```
![](../../images/c%26c%2B%2B/03.png)

第一个域表示前一个chunk的大小，第二个域中，最后三个字节，A表示该chunk属于main arena还是non main arena，M表示该chunk是直接调用mmap分配的mmaped chunk，不属于任何arena, 还是从某个arena分配的。P表示之前的chunk是否空闲。如果P为1，表示前一个chunk正在使用中，则第一个表示其大小的域是无效的，被前一个域用来存储数据了，如果P为0，则前一个chunk是空闲的。

如果chunk是空闲的，且被arena管理，会在chunk中用于用户存储数据的地方，存储四个指针：

fd 和 bk：指针 fd 和 bk 只有当该 chunk 块空闲时才存在，其作用是用于将对应的空闲chunk 块加入到空闲 chunk 块链表中统一管理

fd_nextsize 和 bk_nextsize:当当前的 chunk 存在于 large bins 中时，large bins 中的空闲chunk 是按照从大到小的，但同一个大小的 chunk 可能有多个，增加了这两个字段可以加快遍历空闲 chunk，并查找满足需要的空闲 chunk，fd_nextsize 指向下一个比当前 chunk 大小小的第一个空闲 chunk，bk_nextszie 指向前一个比当前 chunk 大小大的第一个空闲 chunk。

为了使得 chunk 所占用的空间最小，ptmalloc 使用了空间复用，一个 chunk 或者正在被使用，或者已经被 free 掉是空闲的，所以 chunk 的中的一些域可以在使用状态和空闲状态表示不同的意义，来达到空间复用的效果。

空闲时，一个 chunk 中至少需要 4个 size_t大小的空间，用来储 prev_size，size，fd 和 bk，64位系统是32B

同时，每个chunk的大小要对齐到2*size_t。这里是16B.

当一个 chunk 处于使用状态时，它的下一个 chunk 的 prev_size域肯定是无效的。所以实际上，这个空间也可以被当前 chunk 使用。故而实际上，一个使用中的 chunk 的大小的计算公式应该是：in_use_size = (用户请求大小+ 16 - 8 ) align to 16B，这里加 16 是因为需要存储 prev_size 和 size，但又因为向下一个 chunk“借”了 8B，所以要减去 8。

最后，因为空闲的 chunk 和使用中的chunk 使用的是同一块空间。所以肯定要取其中最大者作为实际的分配空间。即最终的分配空间 chunk_size = max(in_use_size, 32)。这就是当用户请求内存分配时，ptmalloc 实际需要分配的内存大小。在后面的介绍中，如果不是特别指明的地方，指的都是这个经过转换的实际需要分配的内存大小，而不是用户请求的内存分配大小

```c

/* conversion from malloc headers to user pointers, and back */

#define chunk2mem(p)   ((void*)((char*)(p) + 2*SIZE_SZ))
#define mem2chunk(mem) ((mchunkptr)((char*)(mem) - 2*SIZE_SZ))

/* The smallest possible chunk */
#define MIN_CHUNK_SIZE        (offsetof(struct malloc_chunk, fd_nextsize))


#define MALLOC_ALIGNMENT       (2 * SIZE_SZ)
#define MALLOC_ALIGN_MASK      (MALLOC_ALIGNMENT - 1)

/* The smallest size we can malloc is an aligned minimal chunk */

#define MINSIZE  \
  (unsigned long)(((MIN_CHUNK_SIZE+MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK))

/* pad request bytes into a usable size -- internal version */

#define request2size(req)                                         \
  (((req) + SIZE_SZ + MALLOC_ALIGN_MASK < MINSIZE)  ?             \
   MINSIZE :                                                      \
   ((req) + SIZE_SZ + MALLOC_ALIGN_MASK) & ~MALLOC_ALIGN_MASK)


/* size field is or'ed with PREV_INUSE when previous adjacent chunk in use */
#define PREV_INUSE 0x1

/* extract inuse bit of previous chunk */
#define prev_inuse(p)       ((p)->size & PREV_INUSE)


/* size field is or'ed with IS_MMAPPED if the chunk was obtained with mmap() */
#define IS_MMAPPED 0x2

/* check for mmap()'ed chunk */
#define chunk_is_mmapped(p) ((p)->size & IS_MMAPPED)


/* size field is or'ed with NON_MAIN_ARENA if the chunk was obtained
   from a non-main arena.  This is only set immediately before handing
   the chunk to the user, if necessary.  */
#define NON_MAIN_ARENA 0x4

/* check for chunk from non-main arena */
#define chunk_non_main_arena(p) ((p)->size & NON_MAIN_ARENA)

#define SIZE_BITS (PREV_INUSE|IS_MMAPPED|NON_MAIN_ARENA)

/* Get size, ignoring use bits */
#define chunksize(p)         ((p)->size & ~(SIZE_BITS))


/* Ptr to next physical malloc_chunk. */
#define next_chunk(p) ((mchunkptr)( ((char*)(p)) + ((p)->size & ~SIZE_BITS) ))

/* Ptr to previous physical malloc_chunk */
#define prev_chunk(p) ((mchunkptr)( ((char*)(p)) - ((p)->prev_size) ))

```

### **bin**
空闲chunk链表叫做bin,在ptmalloc中有4中类型的bin
* small bin
* large bin
* fast bin
* unsorted bin




#### **small bin**
small bin中的chunk大小相同，共62个small bin, 每个small bin的chunk大小相差MALLOC_ALIGNMENT，64位机上，最小的small bin 存储32B的chunk,最大的small bin存储1008B的chunk

small bin中chunk的分配是精确匹配的。（因为分配的chunk大小都被alignment了，而small bins的大小正好相差alignment）

```c
#define NBINS             128
#define NSMALLBINS         64  
#define SMALLBIN_WIDTH    MALLOC_ALIGNMENT
#define SMALLBIN_CORRECTION (MALLOC_ALIGNMENT > 2 * SIZE_SZ) //此值为0
#define MIN_LARGE_SIZE    ((NSMALLBINS - SMALLBIN_CORRECTION) * SMALLBIN_WIDTH) // 64*16,1024

#define in_smallbin_range(sz)  \
  ((unsigned long)(sz) < (unsigned long)MIN_LARGE_SIZE)

//32/16=2,1008/16=63
#define smallbin_index(sz) \
  ((SMALLBIN_WIDTH == 16 ? (((unsigned)(sz)) >> 4) : (((unsigned)(sz)) >> 3)) \
   + SMALLBIN_CORRECTION)
```

#### **large bins**

large bins 中的每一个 bin 分别包含了一个给定大小范围内的 chunk，其中的 chunk 按从大到小排列。

在64位机上，大于等于 1024B 的空闲 chunk，由 large bins管理。Large bins 一共包括 63 个 bin，每个bin的chunk大小并不是一样的，而是在一定范围内从大到小排列的。所有的bins分成6组，每组中的每个bin的最小chunk的大小，组成固定公差的等差数列，每组的 bin 数量依次为 32、16、8、4、2、1，公差依次为 64B、512B、4096B、32768B、262144B等。

以64位机为例，第一组 large bins 的第一个bin 的最小 chunk 为 1024B，共 32 个 bin，公差为 64B，等差数列满足如下关系：`Chunk_size=1024 + 64 * index`，第二组 large bins的第一个bin 的最小 chunk 为第一组 bin 的结束 chunk 大小，满足如下关系`Chunk_size=1024 + 64 * 32 + 512 * index`

```c
#define largebin_index_64(sz)                                                \
(((((unsigned long)(sz)) >>  6) <= 48)?  48 + (((unsigned long)(sz)) >>  6): \
 ((((unsigned long)(sz)) >>  9) <= 20)?  91 + (((unsigned long)(sz)) >>  9): \
 ((((unsigned long)(sz)) >> 12) <= 10)? 110 + (((unsigned long)(sz)) >> 12): \
 ((((unsigned long)(sz)) >> 15) <=  4)? 119 + (((unsigned long)(sz)) >> 15): \
 ((((unsigned long)(sz)) >> 18) <=  2)? 124 + (((unsigned long)(sz)) >> 18): \
					126)

#define largebin_index(sz) \
  (SIZE_SZ == 8 ? largebin_index_64 (sz)                                     \
   : MALLOC_ALIGNMENT == 16 ? largebin_index_32_big (sz)                     \
   : largebin_index_32 (sz))

#define bin_index(sz) \
 ((in_smallbin_range(sz)) ? smallbin_index(sz) : largebin_index(sz))
```
smallbin_index()返回的值为2-63，largebin_index()返回的值为64-126. 1 为unsorted bin的index

#### **unsorted bin**
unsorted bin 可以看作是 small bins 和 large bins 的 cache，只有一个 unsorted bin,其中的chunk不排序，fast bins中的chunk在合并后先放到unsorted bin中，分配时，如果在 unsorted bin 中没有合适的 chunk，就会把 unsorted bin 中的chunk加入到对应的small bin或large bin中

unsorted bin的index是1


#### **unsorted bin,small bin, large bin的管理**

所有的small bins，large bins，和unsorted bin存储在arena的bin数组中，如下
```c
struct malloc_state {

//......
  mchunkptr        bins[NBINS * 2 - 2];

};

```
之间说过，unsorted bin只有一个，small bin 62个，large bin 63个。

而NBINS宏等于128，这样，bins数组中存储254个元素，比所有的bins数量要多。
![](../../images/c%26c%2B%2B/05.png)
bins中存储的不是每个bin的第一个chunk的指针，而是每个bin的第一个chunk的fd和bk,分别指向该bin的第一个chunk和最后一个chunk。
unsorted bin的两个指针存储在0,1位置，small bins存储在2-125，large bins存储在126-251，252,253不适用。

当我们要从arena中获取unsorted bin时，使用如下的代码
```c
p=bin_at(a->bins,1);
first_chunk=p->fd; //first_chunk指向unsorted bin的第一个chunk
last_chunk=p->bk; //last_chunk指向unsorted bin的最后一个chunk
```
如果是根据大小sz获取到small bin,或者large bin，代码如下
```c
idx=bin_index(sz);
p=bin_at(a->bins,idx);
first_chunk=p->fd;
last_chunk=p->bk;
```
bin_at的实现
```c
typedef struct malloc_chunk* mbinptr;

//其中m为指向arena的指针，i为相应bin的索引，unsorted bin为1，small bins为2-63，large bins 为64-126
#define bin_at(m, i) \
  (mbinptr) (((char *) &((m)->bins[((i) - 1) * 2]))			      \
	     - offsetof (struct malloc_chunk, fd))
```
以bin_at(m,1)为例：

1. &((m)->bins[((i)-1)*2]),该式子根据bin的索引i，i为1时，即最开始的bin（unsorted bin）。获得bins[0]的地址。

2. offsetof(struct malloc_chunk,fd):得到fd成员在malloc_chunk结构中的偏移量。在64位系统系统下为16。

3. 把第一步得到地址（令其为pt）转化为char*型，pt-16,bins[-2]的地址（假设bins[-2]存在）

4. 将该地址强行转换成mbinptr，也就是chunk的指针，chunk中fd的偏移量为16，bk的偏移量为24，
当调用p->fd,p->bk时，刚好访问到bins[0],bins[1]


#### **fast bins**

fast bins 小内存块的缓存，大小小于等于 DEFAULT_MXFAST 的 chunk 分配与回收都会在 fast bins 中先查找，在64位上为 128字节，这个参数可以通过 mallopt 函数进行修改，最大值为 160B。
```c

#ifndef DEFAULT_MXFAST
#define DEFAULT_MXFAST     (64 * SIZE_SZ / 4) //128
#endif

typedef struct malloc_chunk* mfastbinptr;

#define MAX_FAST_SIZE     (80 * SIZE_SZ / 4) //160

#define fastbin_index(sz) \
  ((((unsigned int)(sz)) >> (SIZE_SZ == 8 ? 4 : 3)) - 2)

#define fastbin(ar_ptr, idx) ((ar_ptr)->fastbinsY[idx])

//request2size(MAX_FAST_SIZE)=176,fastbin_index(176)=9,NFASTBINS=10
#define NFASTBINS  (fastbin_index(request2size(MAX_FAST_SIZE))+1)

struct malloc_state {
//....
  /* Fastbins */
  mfastbinptr      fastbinsY[NFASTBINS];

};

#define FASTBIN_CONSOLIDATION_THRESHOLD  (65536UL)

```
FASTBIN_CONSOLIDATION_THRESHOLD 表示当回收的 chunk 与相邻的 chunk 合并后大于该值， 64k，则合并 fast bins 中所有的 chunk 放回到 unsorted bin。


### **arena**
一个arean是一个分配区，在最早版本的实现中只有一个主分配区（main arena），每次分配内存都必须对主分配区加锁，分配完成后释放锁。在多线程环境下，锁的争用严重影响内存分配的效率，现在这一版本增加了非主分配区（non main arena）支持. 所有的分配区用arena中的next连接.每一个分配区利用互斥锁（mutex）使线程对于该分配区的访问互斥。


每个进程只有一个主分配区，但可能存在多个非主分配区，ptmalloc 根据系统对分配区的争用情况动态增加非主分配区的数量，分配区的数量一旦增加，就不会再减少了。

同时，分配区的数量也是有限制的，默认情况下最多`2*num_cores `或者`8*num_cores`,可通过mallopt设置更大的值。当arena的数量达到上限时，便不会再增加arena

主分配区可以访问进程的 堆地址空间 区域和 memory map 映射区域，也就是说主分配区可以使用 sbrk 和mmap向操作系统申请虚拟内存。而非主分配区只能访问进程的 mmap 映射区域


当某一线程需要调用 malloc()分配内存空间时，该线程先查看线程本地存储是否已经存储指向某个分配区的指针，如果有，调用mutex_lock对该分配区加锁。 如果没有，并且arena的数量没有达到上线，则会创建新的分配区并绑定到当前线程的本地存储中。如果arena的数量达到上限了，则对所有arena连接成的连表一一尝试加锁，加锁成功，则绑定该arena并使用。也就是说，如果线程数量少于arena数量的限制，每个线程都会有自己的arena,如果线程数量多于arena数量的限制，则会存在多个线程使用同一个arena，从而造成锁争用的情况。在释放操作中，首先根据释放的chunk找到对应的arena，如果chunk大小小于max_fast,直接使用cas原子操作释放回fast bins中。否则对arena加锁，释放回arena中的bins数组中，或者和top chunk合并。

因为释放回fastbins中，只需要访问每个fastbin的头节点，采用头插法，使用原子CAS操作即可。fastbin中的每个chunk只需要连接成单链表，也就是只设置fd变量。 而释放回bins数组中，最起码要访问两个数组元素。
。


```c

#define FASTCHUNKS_BIT        (1U)

#define have_fastchunks(M)     (((M)->flags &  FASTCHUNKS_BIT) == 0)
#define clear_fastchunks(M)    catomic_or (&(M)->flags, FASTCHUNKS_BIT)
#define set_fastchunks(M)      catomic_and (&(M)->flags, ~FASTCHUNKS_BIT)

#define NONCONTIGUOUS_BIT     (2U)

#define contiguous(M)          (((M)->flags &  NONCONTIGUOUS_BIT) == 0)
#define noncontiguous(M)       (((M)->flags &  NONCONTIGUOUS_BIT) != 0)
#define set_noncontiguous(M)   ((M)->flags |=  NONCONTIGUOUS_BIT)
#define set_contiguous(M)      ((M)->flags &= ~NONCONTIGUOUS_BIT)


struct malloc_state {
  /* Serialize access.  */
  mutex_t mutex;

  /* Flags (formerly in max_fast).  */
  int flags;


  /* Fastbins */
  mfastbinptr      fastbinsY[NFASTBINS];

  /* Base of the topmost chunk -- not otherwise kept in a bin */
  mchunkptr        top;

  /* The remainder from the most recent split of a small request */
  mchunkptr        last_remainder;

  /* Normal bins packed as described above */
  mchunkptr        bins[NBINS * 2 - 2];

  /* Bitmap of bins */
  unsigned int     binmap[BINMAPSIZE];

  /* Linked list */
  struct malloc_state *next;

#ifdef PER_THREAD 
  /* Linked list for free arenas.  */
  struct malloc_state *next_free;
#endif

  /* Memory allocated from the system in this arena.  */
  INTERNAL_SIZE_T system_mem;
  INTERNAL_SIZE_T max_system_mem;
};

typedef struct malloc_state *mstate;
```
* flags，bit0 表示是否有 fast bin chunk，bit1 表示是否能返回连续的地址空间，显然只有主分配区才能做到，因为只有主分配区调用的sbrk系统调用，分配的堆地址空间，而非主分配区使用的mmmap,分配的memory map的地址空间。

* binmap，标识 bit 指向的 bin 是否有空闲 chunk。

#### **bitmap**
```c

#define BINMAPSHIFT      5
#define BITSPERMAP       (1U << BINMAPSHIFT) //32
#define BINMAPSIZE       (NBINS / BITSPERMAP)  //4

#define idx2block(i)     ((i) >> BINMAPSHIFT)
#define idx2bit(i)       ((1U << ((i) & ((1U << BINMAPSHIFT)-1))))

#define mark_bin(m,i)    ((m)->binmap[idx2block(i)] |=  idx2bit(i))
#define unmark_bin(m,i)  ((m)->binmap[idx2block(i)] &= ~(idx2bit(i)))
#define get_binmap(m,i)  ((m)->binmap[idx2block(i)] &   idx2bit(i))
```

idx2block(63):
* 63>>5=1, 也就是binmap[1],这个unsigned int中的某一位

idx2bit(63):
* 63 & 31 = 31
* 1<<31,也就是unsigned int中的最高位被置1


#### **top chunk**
top chunk并不是组织在bin中。ptmalloc每次在向系统申请内存时，申请的都是大块的内存，其中从低地址部分开始的chunk返回给用户使用，而剩下的chunk便是top chunk,从未被使用过。

如果是main arena,调用sbrk从堆地址空间获取内存，低地址部分返回给用户，剩下的为main arena的top chunk。 如果是非main arena, 调用mmap从memory map地址空间获取内存，这个内存被称为heap, non main arena的heap可以有多个，地址不是连续的，所有的heap链接成连表。 non main arena的top chunk指向当前使用的heap的top chunk.


当分配内存时，如果fast bins和bins中都无法满足要求，则从top chunk中分配，如果top chunk也无法满足，main arena会调用sbrk()系统调用拓展top chunk大小，non main arena则调用mmap()分配新的heap，并将top chunk指向当前新分配的heap.

如果需要收缩ptmalloc管理的内存，释放空闲内存回系统，也是收缩的top chunk。 对于main arena,收缩的就是堆地址空间对应的top chunk, 对于非main arena,从当前使用的heap对应的top chunk开始收缩，如果当前的heap可以完全释放回系统，则将top chunk指向当前heap之前的heap,继续判断是否可以收缩。


#### **last_remainder chunk** 
如果分配区上次分配的chunk还有剩余，剩下的chunk由arena的last_remainder指向，并且该chunk被加入unsorted bin中

此类型的chunk用于提高连续malloc(small chunk)的效率和内存分配的局部性。

### **mmaped chunk**
到目前位置，一个arena中管理的chunk,有各种bins中的chunk, top chunk, last_remainder chunk.

mapped chunk不被arena管理，当需要分配的 chunk 足够大，ptmalloc会直接使用mmap分配chunk给用户使用。这样的 chunk 在被 free 时将直接调用unmap归还给系统。

mapped chunk,其size 字段的M表示被设置了，`chunk_is_mmapped(p)`宏返回1


### **配置参数**
可通过mallopt调整配置参数
```c
int mallopt(int param, int value);
```
其中param主要有如下几个选项

**M_MXFAST**

fast bin中保存的chunk大小的最大值。默认为`64*sizeof(size_t)/4`,64位机上是128B,这个值的范围为0 - 80*sizeof(size_t)/4，64位机上最大值是160B. 如果设置为0，disable the use of fastbins

**M_MMAP_THRESHOLD**

如果请求分配的内存大小大于等于M_MMAP_THRESHOLD，并且ptmalloc中各种bins和top chunk无法满足，则直接调用mmap分配chunk给用户，mmaped chunk在释放时直接调用munmap()释放回操作系统。

当然，相比从ptmalloc中分配chunk，这种直接调用mmap系统调用的方式，有如下的缺点
* 释放时直接调munmap释放回系统，不被ptmalloc缓存，每次都从系统申请内存，调用mmap系统调用陷入内核，开销大
* memory map映射大小要对齐到page frame,也就是4kb，如果请求的是小内存，则内部碎片大
* mmap的内存，在分配物理页面时，分配的物理页面需要清零。<font color=red>这个不能算缺点，堆内存在分配物理页面时，也需要清零。 内核在page fault handler中处理缺页异常时，anonymous memory mapping 的缺页处理都是调用的`alloc_page(GFP_HIGHUSER | __GFP_ZERO);`，堆地址空间的memory region也是属于anonymous memory mapping。更何况，清零又不是在分配地址空间时发生的，内核都是demand paging, 在page fault handler 中才实际分配物理页面</font>.当然linux map page 手册中确实写了这个缺点，只能说写手册的人水平也一般
* 多线程共享同一个进程地址空间，如果都是直接调用的mmap，在系统调用中并发访问进程的地址空间，需要加锁。

M_MMAP_THRESHOLD 的默认值是128k,范围为0-512k(32bit),0-32M(64bit),最大值的参数是DEFAULT_MMAP_THRESHOLD_MAX

mmap threshold是动态调整的，初始值是128k,如果munmap释放的chunk大小大于mmap threshold并且小于等于DEFAULT_MMAP_THRESHOLD_MAX ，则mmap threshold调整为释放的chunk大小。并且threshold for trimming 设置为chunk大小的两倍。这个动态调整机制是默认开启的，如果设置了M_TRIM_THRESHOLD，M_MMAP_THRESHOLD，M_TOP_PAD，M_MMAP_MAX其中的任意一个参数，则该机制关闭。


**M_TRIM_THRESHOLD**

M_TRIM_THRESHOLD 用于设置 mmap 收缩阈值，默认值为 128KB。收缩只会在 free时才发生，如果当前 free 的 chunk 大小加上前后能合并 chunk 的大小大于 64KB，并且 top chunk 的大小达到 M_TRIM_THRESHOLD，对于非主分配区，调用 heap_trim()返回一部分内存给操作系统，对于主分配区，调用 systrim()返回一部分内存给操作系统。

M_TRIM_THRESHOLD 的值必须设置为页大小对齐，设置为-1 会关闭内存收缩设置。

**M_MMAP_MAX**
This parameter specifies the maximum number of allocation requests that may be simultaneously serviced using mmap(2).  This parameter exists because some systems have a limited number of internal tables for use by mmap(2),and using more than a few of them may degrade performance.The default value is 65,536, a value which has no special significance and which serves only as a safeguard.Setting this parameter to 0 disables the use of mmap(2) for servicing large allocation requests.

**M_TOP_PAD** 
默认值是128KB
当需要对top chunk做拓展时，这个值会被增加到需要拓展的大小
当需要对top chunk做收缩时，会保留这个大小。
In either case, the amount of padding is always rounded to a system page boundary.

**M_ARENA_MAX**
这个值规定了可以创建的arenas的上限。默认是0，可以创建的arenas的上限由M_ARENA_TEST决定。
从glibc2.15开始，多线程的arenas优化就是默认开启的了，在这之前，2.10版本开始，需要通过
--enable-experimental-malloc 才能开启多线程arenas的优化。(应该对应代码中的PER_THREAD宏)

**M_ARENA_TEST**
在32位机上，默认值是2， 在64位机上，默认值是8. 如果没有设置M_ARENA_MAX，则该值乘以cpu的核心数，是能创建的arenas的上限。
如果设置了M_ARENA_MAX,则这个值不会被使用。

这里的多线程优化，是PER_THREAD宏开启的，这个宏开启后，如果线程本地存储中没有绑定arena，如果arenas的数量没有达到上线，则直接创建arena.如果达到上限了，则从已经存在的arena中获取。 如果线程本地存储中已经有arena了，则直接调用mutex_lock加锁。

开启这个机制，只会在线程数大于arena的数量限制时，才有可能多个线程共享同一个arena，从而造成锁争用。

如果没有开启这个优化，在试图从线程本地存储中获取arena并加锁失败后，首先是试图从已经存在的arena中循环尝试，这里面有获取锁的开销, 而且虽然在线程本地存储中存储了上一次获取内存使用的arena，但这arena并不是跟线程一一对应的，多线程在试图获取arena时，增大了锁冲突的概率。而PER_THREAD优化，在线程数小于arena的上限时，arena是跟线程一一对应的。

这个优化目前在glibc2.17版本中默认开启的。


### **初始化**
glibc中定义了ptmalloc的初始化函数

```c
static void
ptmalloc_init (void)
{
  if(__malloc_initialized >= 0) return;
  __malloc_initialized = 0;
//...

  tsd_key_create(&arena_key, NULL);
  //当前线程的线程本地存储中存储时的main_arena
  tsd_setspecific(arena_key, (void *)&main_arena);

  //设置了在fork前后需要调用的函数，主要是arena全部加锁和解锁。fork系统调用需要拷贝父进程的地址空间，这时候除了父子进程之外，如果并发的其他线程也访问地址空间，则可能造成地址空间处于不一致的状态，这里给全部的anera加上锁，最起码保证其他线程如果要访问arena,需要等到fork完毕。
  thread_atfork(ptmalloc_lock_all, ptmalloc_unlock_all, ptmalloc_unlock_all2);
  const char *s = NULL;
  //开始处理环境变量，跟据环境变量调用__libc_mallopt设置配置参数
  if (__builtin_expect (_environ != NULL, 1))
  {
      char **runp = _environ;
      char *envline;

      while (__builtin_expect ((envline = next_env_entry (&runp)) != NULL,
			       0))
	    {
	      size_t len = strcspn (envline, "=");

	      if (envline[len] != '=')
	    /* This is a "MALLOC_" variable at the end of the string
	       without a '=' character.  Ignore it since otherwise we
	       will access invalid memory below.  */
	        continue;

	      switch (len)
	      {
          //case....
	        case 8:
	          if (! __builtin_expect (__libc_enable_secure, 0))
        		{
        		  if (memcmp (envline, "TOP_PAD_", 8) == 0)
        		    __libc_mallopt(M_TOP_PAD, atoi(&envline[9]));
        		  else if (memcmp (envline, "PERTURB_", 8) == 0)
        		    __libc_mallopt(M_PERTURB, atoi(&envline[9]));
        		}
        	  break;
	        case 9:
	          if (! __builtin_expect (__libc_enable_secure, 0))
		        {
        		  if (memcmp (envline, "MMAP_MAX_", 9) == 0)
        		    __libc_mallopt(M_MMAP_MAX, atoi(&envline[10]));
        #ifdef PER_THREAD
        		  else if (memcmp (envline, "ARENA_MAX", 9) == 0)
        		    __libc_mallopt(M_ARENA_MAX, atoi(&envline[10]));
        #endif
        		}
	          break;
#ifdef PER_THREAD
        	case 10:
        	  if (! __builtin_expect (__libc_enable_secure, 0))
        		{
        		  if (memcmp (envline, "ARENA_TEST", 10) == 0)
        		    __libc_mallopt(M_ARENA_TEST, atoi(&envline[11]));
        		}
        	  break;
#endif
	        case 15:
	          if (! __builtin_expect (__libc_enable_secure, 0))
        		{
        		  if (memcmp (envline, "TRIM_THRESHOLD_", 15) == 0)
        		    __libc_mallopt(M_TRIM_THRESHOLD, atoi(&envline[16]));
        		  else if (memcmp (envline, "MMAP_THRESHOLD_", 15) == 0)
        		    __libc_mallopt(M_MMAP_THRESHOLD, atoi(&envline[16]));
        		}
	          break;
	        default:
	          break;
	      }
	    }
  }
  //.....
  __malloc_initialized = 1;
}
```
初始化主要就是将main_arena设置到当前线程的本地存储中，然后根据环境变量设置好配置参数。 

main_arena是static global variable
```c
static struct malloc_state main_arena =
  {
    .mutex = MUTEX_INITIALIZER,
    .next = &main_arena
  };
```

全局变量`int __malloc_initialized `表示是否初始化完成了。

什么时候初始化呢？

首先ptmalloc定义了一个malloc_hook,如下
```c
__malloc_ptr_t weak_variable (*__malloc_hook)
     (size_t __size, const __malloc_ptr_t) = malloc_hook_ini;
```
__malloc_hook是个函数指针，类型为 `__malloc_ptr_t (size_t __size, const __malloc_ptr_t)`

```c
# define __malloc_ptr_t  void *

# define weak_variable weak_function

# define weak_function __attribute__ ((weak))
```

**关于weak function,answer from GPT-3.5**

In C and C++, functions can be declared as weak functions, which means that they can be overwritten by a stronger function with the same name at runtime. This feature is commonly used in embedded systems and device driver programming where you may need to provide a default implementation of a function that can be overridden by a more complete or customized implementation later.

When a function is declared as weak, it can be overridden if a stronger version of the same function exists in the code. If there is no stronger version of the function, the weak version will be used. 

Here is an example of how to declare a weak function in C:
```
// Declare a weak function
__attribute__((weak)) void myWeakFunction() {
  // Default implementation of myWeakFunction
}
```

In this example, we declare a weak version of the 

`myWeakFunction()`

 function. If a stronger version of 

`myWeakFunction()`

 is defined later in the code, it will overwrite this weak version. If no stronger version of the function is defined, then the default implementation provided will be used.


malloc_hook_ini定义如下,首先将__malloc_hook清空，然后调用ptmalloc_init初始化。
```c
static void*
malloc_hook_ini(size_t sz, const __malloc_ptr_t caller)
{
  __malloc_hook = NULL;
  ptmalloc_init();
  return __libc_malloc(sz);
}
```

当用户程序中调用malloc时，会调用glibc中的__libc_malloc,首先会检查__malloc_hook是否设置了，如果设置了hook，直接调用hook返回。

除了malloc之外，realloc中也设置了相应的hook,如下
```c
static void*
realloc_hook_ini(void* ptr, size_t sz, const __malloc_ptr_t caller)
{
  __malloc_hook = NULL;
  __realloc_hook = NULL;
  ptmalloc_init();
  return __libc_realloc(ptr, sz);
}
```
除此之外，代码中也在`__libc_mallopt`,`__libc_mallinfo`等中设置了检查
```c
  if(__malloc_initialized < 0)
    ptmalloc_init ();
```
后续我们将首先调用了初始化的线程，也就是绑定了main_arena的线程，叫做主线程。其他的线程叫做非主线程。

### **malloc**

在glibc中，malloc为__libc_malloc的强符号
```c
strong_alias (__libc_malloc, malloc)
```
所以，当我们调用malloc是，其实调用的是glibc中的__libc_malloc函数

```c
void*
__libc_malloc(size_t bytes)
{
  mstate ar_ptr; //arena分配区
  void *victim;

//首先检查是否设置了hook,如果设置了则直接调用该hook并返回

  __malloc_ptr_t (*hook) (size_t, const __malloc_ptr_t)
    = force_reg (__malloc_hook); 
  if (__builtin_expect (hook != NULL, 0)) //__builtin_expect:分支预测，这里预测其没有设置hook
    return (*hook)(bytes, RETURN_ADDRESS (0));

//查找一个分配区
  arena_lookup(ar_ptr);
//给该分配区加锁
  arena_lock(ar_ptr, bytes);
  if(!ar_ptr)
    return 0;
//_int_malloc，进入arena分配
  victim = _int_malloc(ar_ptr, bytes);
  if(!victim) {
    ar_ptr = arena_get_retry(ar_ptr, bytes);
    if (__builtin_expect(ar_ptr != NULL, 1)) {
      victim = _int_malloc(ar_ptr, bytes);
      (void)mutex_unlock(&ar_ptr->mutex);
    }
  } else
    (void)mutex_unlock(&ar_ptr->mutex);
  assert(!victim || chunk_is_mmapped(mem2chunk(victim)) || ar_ptr == arena_for_chunk(mem2chunk(victim)));
  return victim;
}
```

#### **arena_lookup**
```c
#define arena_lookup(ptr) do { \
  void *vptr = NULL; \
  ptr = (mstate)tsd_getspecific(arena_key, vptr); \
} while(0)
```
这里只是获取线程本地存储中的值到ptr中，如果是非主线程，这个值可能会NULL.

我们看下如果使用pthread线程库，线程本地存储是怎么实现的
```c
typedef int tsd_key_t[1];	/* no key data structure, libc magic does it */
static tsd_key_t arena_key;


__libc_tsd_define (static, void *, MALLOC)	/* declaration/common definition */
#define tsd_key_create(key, destr)	((void) (key)) 
#define tsd_setspecific(key, data)	__libc_tsd_set (void *, MALLOC, (data))
#define tsd_getspecific(key, vptr)	((vptr) = __libc_tsd_get (void *, MALLOC))


#define __libc_tsd_define(CLASS, TYPE, KEY)	\
  CLASS __thread TYPE __libc_tsd_##KEY attribute_tls_model_ie;

#define __libc_tsd_address(TYPE, KEY)		(&__libc_tsd_##KEY)
#define __libc_tsd_get(TYPE, KEY)		(__libc_tsd_##KEY)
#define __libc_tsd_set(TYPE, KEY, VALUE)	(__libc_tsd_##KEY = (VALUE))

```

可以看到，使用pthread创建线程本地存储的全局变量，就是加上 `__thread`关键字。这里实际线程本地存储的全局变量的变量名实际是

```c
static __thread void * __libc_tsd_MALLOC
```

而`tsd_getspecific(arena_key, vptr)`直接拓展成
```c
vptr=__libc_tsd_MALLOC
```
直接返回线程本地存储的全局变量就可以了。



### **arena_lock**
```c
# define arena_lock(ptr, size) do { \
  if(ptr) \
    (void)mutex_lock(&ptr->mutex); \
  else \
    ptr = arena_get2(ptr, (size), NULL); \
} while(0)

```
可以看到,如果是主线程，直接调用mutex_lock加完锁就可以了，如果是非主线程，且之前没有在线程本地存储中绑定arena,则这里会调用arena_get2。 也就是说，arena_get2函数只会在非主线程中调用，而其创建的arena，也是non main arena,其中使用的heap是调用mmap获取的内存。

### **arena_get2**
```c
static mstate
internal_function
arena_get2(mstate a_tsd, size_t size, mstate avoid_arena)
{
  mstate a;
  static size_t narenas_limit;
  a = get_free_list ();
  if (a == NULL)
  {
    
    //这里会根据arena_max和arena_test设置能创建的arena上限。
    //当然在初始的几次arena_get2中可能都不会设置arena_max。
    //mp_.arena_test在32位机上时2，64位机上是8
    if (narenas_limit == 0)
	  {
	    if (mp_.arena_max != 0)
	      narenas_limit = mp_.arena_max;
	    else if (narenas > mp_.arena_test)
	    {
	      int n  = __get_nprocs ();

	      if (n >= 1)
		      narenas_limit = NARENAS_FROM_NCORES (n);
	      else
		    /* We have no information about the system.  Assume two
		    cores.  */
		      narenas_limit = NARENAS_FROM_NCORES (2);
	    }
	  }
    repeat:;
      size_t n = narenas;
      /* NB: the following depends on the fact that (size_t)0 - 1 is a
	 very large number and that the underflow is OK.  If arena_max
	 is set the value of arena_test is irrelevant.  If arena_test
	 is set but narenas is not yet larger or equal to arena_test
	 narenas_limit is 0.  There is no possibility for narenas to
	 be too big for the test to always fail since there is not
	 enough address space to create that many arenas.  */

   //如果同时需要创建arena的线程很多，而narenas_limit又是0，比如上来就启动很多线程都来malloc,都并发走到了这里，这时候，可能创建的arena比narenas_limit多的多。
   //这段代码实在是不怎么样，跟临时拼凑的一样
      if (__builtin_expect (n <= narenas_limit - 1, 0))
	    {
	      if (catomic_compare_and_exchange_bool_acq (&narenas, n + 1, n))
	        goto repeat;
	      a = _int_new_arena (size);
	      if (__builtin_expect (a == NULL, 0))
	        catomic_decrement (&narenas);
	    }
      else
	      a = reused_arena (avoid_arena);
    }
    return a;
}

# define NARENAS_FROM_NCORES(n) ((n) * (sizeof(long) == 4 ? 2 : 8))
```

#### **_int_new_arena**
该函数只有非主线程会调用，创建一个non main arena。


```c
static mstate
_int_new_arena(size_t size)
{
  mstate a;
  heap_info *h; //一个arena可以有多个heap
  char *ptr;
  unsigned long misalign;

  h = new_heap(size + (sizeof(*h) + sizeof(*a) + MALLOC_ALIGNMENT),
	       mp_.top_pad);
  if(!h) {
    /* Maybe size is too large to fit in a single heap.  So, just try
       to create a minimally-sized arena and let _int_malloc() attempt
       to deal with the large request via mmap_chunk().  */
    h = new_heap(sizeof(*h) + sizeof(*a) + MALLOC_ALIGNMENT, mp_.top_pad);
    if(!h)
      return 0;
  }
  //一个heap中，第一块内存中存储heap本身的描述符，第二块内存中存储heap所属于的arena的描述符。
  a = h->ar_ptr = (mstate)(h+1);

  malloc_init_state(a); //arena的初始化逻辑，主要是设置好bins
  /*a->next = NULL;*/
  a->system_mem = a->max_system_mem = h->size;
  arena_mem += h->size;

  /* Set up the top chunk, with proper alignment. */
  ptr = (char *)(a + 1);
  misalign = (unsigned long)chunk2mem(ptr) & MALLOC_ALIGN_MASK;
  if (misalign > 0)
    ptr += MALLOC_ALIGNMENT - misalign;
  top(a) = (mchunkptr)ptr; //top(a) : #define top(ar_ptr) ((ar_ptr)->top) 
  set_head(top(a), (((char*)h + h->size) - ptr) | PREV_INUSE); 

  tsd_setspecific(arena_key, (void *)a);
  mutex_init(&a->mutex);
  (void)mutex_lock(&a->mutex);


  (void)mutex_lock(&list_lock);


  /* Add the new arena to the global list.  */
  a->next = main_arena.next;
  atomic_write_barrier ();
  main_arena.next = a;


  (void)mutex_unlock(&list_lock);

  return a;
}

/* Set size at head, without disturbing its use bit */
#define set_head_size(p, s)  ((p)->size = (((p)->size & SIZE_BITS) | (s)))

/* Set size/use field */
#define set_head(p, s)       ((p)->size = (s))

```
可以看到这个_int_new_arena创建了一个非main arena,设置好了heap地址空间，top chunk,bins等等，可以直接使用了。

于此同时，如果第一次调用malloc的是主线程，主线程的main arena到目前为止还是全局变量的初始值，基本没有任何设置。

#### **new_heap**
此函数是给non main arena分配地址空间，一块地址空间叫heap,用mmap分配，基本没什么可说的。
```c
static heap_info *
internal_function
new_heap(size_t size, size_t top_pad)
{
  size_t page_mask = GLRO(dl_pagesize) - 1;
  char *p1, *p2;
  unsigned long ul;
  heap_info *h;

  if(size+top_pad < HEAP_MIN_SIZE)
    size = HEAP_MIN_SIZE;
  else if(size+top_pad <= HEAP_MAX_SIZE)
    size += top_pad;
  else if(size > HEAP_MAX_SIZE)
    return 0;
  else
    size = HEAP_MAX_SIZE;
  size = (size + page_mask) & ~page_mask;

  /* A memory region aligned to a multiple of HEAP_MAX_SIZE is needed.
     No swap space needs to be reserved for the following large
     mapping (on Linux, this is the case for all non-writable mappings
     anyway). */
  p2 = MAP_FAILED;
  if(aligned_heap_area) {
    p2 = (char *)MMAP(aligned_heap_area, HEAP_MAX_SIZE, PROT_NONE,
		      MAP_NORESERVE);
    aligned_heap_area = NULL;
    if (p2 != MAP_FAILED && ((unsigned long)p2 & (HEAP_MAX_SIZE-1))) {
      //没有对齐，取消映射的地址
      __munmap(p2, HEAP_MAX_SIZE);
      p2 = MAP_FAILED;
    }
  }
  if(p2 == MAP_FAILED) {
    p1 = (char *)MMAP(0, HEAP_MAX_SIZE<<1, PROT_NONE, MAP_NORESERVE);
    if(p1 != MAP_FAILED) {
      p2 = (char *)(((unsigned long)p1 + (HEAP_MAX_SIZE-1))
		    & ~(HEAP_MAX_SIZE-1));
      ul = p2 - p1;
      if (ul)
	      __munmap(p1, ul); //取消前面没有对齐的部分
      else
	      aligned_heap_area = p2 + HEAP_MAX_SIZE; //下次映射的起始地址
      __munmap(p2 + HEAP_MAX_SIZE, HEAP_MAX_SIZE - ul); //取消后面没有对齐的部分
    } else {
      /* Try to take the chance that an allocation of only HEAP_MAX_SIZE
	 is already aligned. */
      p2 = (char *)MMAP(0, HEAP_MAX_SIZE, PROT_NONE, MAP_NORESERVE);
      if(p2 == MAP_FAILED)
	      return 0;
      if((unsigned long)p2 & (HEAP_MAX_SIZE-1)) {
	      __munmap(p2, HEAP_MAX_SIZE);
	      return 0;
      }
    }
  }
  if(__mprotect(p2, size, PROT_READ|PROT_WRITE) != 0) {
    __munmap(p2, HEAP_MAX_SIZE);
    return 0;
  }
  h = (heap_info *)p2;
  h->size = size;
  h->mprotect_size = size;
  return h;
}

#define DEFAULT_MMAP_THRESHOLD_MAX (4 * 1024 * 1024 * sizeof(long)) //32M

#define HEAP_MIN_SIZE (32*1024)  //32k

#define HEAP_MAX_SIZE (2 * DEFAULT_MMAP_THRESHOLD_MAX)  //64M

```
没啥可分析的，每次分配的地址大小是HEAP_MAX_SIZE，一定要是HEAP_MAX_SIZE对齐的，其中的前size部分被设置成可读写，其他部分暂时无法访问。

#### **_int_malloc**
该函数就是在arena中分配chunk了，这个函数比较长，分段看。 注意到这里， 如果是main arena第一次调用malloc,main arena还没有初始化，里面的值，要么为0，要么为NULL,堆地址空间ptmalloc到目前为止也还没有使用，brk==brk_start

首先c语言，需要先将局部变量初始化在前，然后调用`checked_request2size(bytes, nb);`将请求大小转为chunk大小。转换逻辑之前已经说过了。
```c
static void*
_int_malloc(mstate av, size_t bytes)
{
  INTERNAL_SIZE_T nb;               /* normalized request size */
  unsigned int    idx;              /* associated bin index */
  mbinptr         bin;              /* associated bin */

  mchunkptr       victim;           /* inspected/selected chunk */
  INTERNAL_SIZE_T size;             /* its size */
  int             victim_index;     /* its bin index */

  mchunkptr       remainder;        /* remainder from a split */
  unsigned long   remainder_size;   /* its size */

  unsigned int    block;            /* bit map traverser */
  unsigned int    bit;              /* bit map traverser */
  unsigned int    map;              /* current word of binmap */

  mchunkptr       fwd;              /* misc temp for linking */
  mchunkptr       bck;              /* misc temp for linking */

  const char *errstr = NULL;
  //将请求大小转为chunk大小
  checked_request2size(bytes, nb);
  //....
```

然后处理fastbin的分配，这里对fastbin的访问使用了cas,虽然在分配的时候给访问的arena加锁了，但在释放的时候，如果只释放回fastbin,也是使用的cas,没有加锁。

然后注意，如果fastbin中都是NULL,这段代码不会有问题。
```c
  if ((unsigned long)(nb) <= (unsigned long)(get_max_fast ())) {
    idx = fastbin_index(nb);
    mfastbinptr* fb = &fastbin (av, idx);
    mchunkptr pp = *fb;
    do{
	    victim = pp;
	    if (victim == NULL)
	      break;
    }while ((pp = catomic_compare_and_exchange_val_acq (fb, victim->fd, victim))!= victim);
    if (victim != 0) {
      //...
      void *p = chunk2mem(victim);
      if (__builtin_expect (perturb_byte, 0))
	      alloc_perturb (p, bytes);
      return p;
    }
  }
//...

```

然后是对small requrest处理。 small request的处理需要访问bins，对于non main arena中bins已经初始化过了，而非main arena的bins的初始化，这个逻辑放到了malloc_consolidate中，感觉很别扭。

```c
  if (in_smallbin_range(nb)) {
    idx = smallbin_index(nb);
    bin = bin_at(av,idx);

    if ( (victim = last(bin)) != bin) { //如果链表是空的，last(bin)==bin,fast(bin)==bin
      if (victim == 0) /* initialization check */
	      malloc_consolidate(av); //这里是处理main arena中bins的初始化，非main arena不会走到这里。
      else {
        bck = victim->bk;
        //....
  	    set_inuse_bit_at_offset(victim, nb);
  	    bin->bk = bck;
  	    bck->fd = bin;

  	    if (av != &main_arena)
  	      victim->size |= NON_MAIN_ARENA;
        //...
  	    void *p = chunk2mem(victim);
  	    if (__builtin_expect (perturb_byte, 0))
  	      alloc_perturb (p, bytes);
  	    return p;
      }
    }
  }

#define smallbin_index(sz) \
  ((SMALLBIN_WIDTH == 16 ? (((unsigned)(sz)) >> 4) : (((unsigned)(sz)) >> 3)) \
   + SMALLBIN_CORRECTION)

#define first(b)     ((b)->fd)
#define last(b)      ((b)->bk)

#define set_inuse_bit_at_offset(p, s)\
 (((mchunkptr)(((char*)(p)) + (s)))->size |= PREV_INUSE)

```
small bin中的分配，根据分配的大小找到small bin的链表，如果不空，取链表最后一个chunk，这个逻辑很直接。

看下malloc_consolidate中对main arena的初始化
```c
static void malloc_consolidate(mstate av)
{
//声明了一堆函数局部变量

  /*
    If max_fast is 0, we know that av hasn't
    yet been initialized, in which case do so below
  */

  if (get_max_fast () != 0) {
    //这其中是fastbin中chunk的合并逻辑
  }else {
    malloc_init_state(av); // non main arena在创建的时候（_int_new_arena)就调用了这个函数了，main arena的调用放在了这里
    //...
  }
}

static void malloc_init_state(mstate av)
{
  int     i;
  mbinptr bin;

  /* Establish circular links for normal bins */
  //bins的初始化，每个bins都是空链表
  for (i = 1; i < NBINS; ++i) {
    bin = bin_at(av,i);
    bin->fd = bin->bk = bin;
  }


  if (av != &main_arena)
    set_noncontiguous(av);
  if (av == &main_arena) 
    set_max_fast(DEFAULT_MXFAST);  
  av->flags |= FASTCHUNKS_BIT; //当前fastbin中还没有chunk
  av->top            = initial_top(av);  //把main arena的top设置为指向了unsorteb bin, 也就是为bins数组的首地址-16，也就是main arena中的top成员的地址。如果是non main arena，随后又设置了top chunk指向第一个创建的heap.
}

#define initial_top(M)              (unsorted_chunks(M))

```


```c
  else {
    idx = largebin_index(nb);
    if (have_fastchunks(av))
      malloc_consolidate(av);
  }
```
不属于small bin的话，首先malloc_consolidate合并fast bin中的chunk，并入unsorted bin中

```c
  for(;;) {

    int iters = 0;
    //循环处理unsorted bin中的chunk，最多处理10000个chunk。并且是从链表后开始处理
    while ( (victim = unsorted_chunks(av)->bk) != unsorted_chunks(av)) {
      bck = victim->bk;
      //...
      size = chunksize(victim);

      if (in_smallbin_range(nb) && bck == unsorted_chunks(av) && victim == av->last_remainder && (unsigned long)(size) > (unsigned long)(nb + MINSIZE)) {

        	/* split and reattach remainder */
        	remainder_size = size - nb;
        	remainder = chunk_at_offset(victim, nb);
        	unsorted_chunks(av)->bk = unsorted_chunks(av)->fd = remainder;
        	av->last_remainder = remainder;
        	remainder->bk = remainder->fd = unsorted_chunks(av);
        	if (!in_smallbin_range(remainder_size))
      	  {
      	    remainder->fd_nextsize = NULL;
      	    remainder->bk_nextsize = NULL;
      	  }

        	set_head(victim, nb | PREV_INUSE | (av != &main_arena ? NON_MAIN_ARENA : 0));
        	set_head(remainder, remainder_size | PREV_INUSE);
        	set_foot(remainder, remainder_size);
          //...
        	void *p = chunk2mem(victim);
        	if (__builtin_expect (perturb_byte, 0))
        	  alloc_perturb (p, bytes);
        	return p;
      }
      //....

#define unsorted_chunks(M)          (bin_at(M, 1))

/* Treat space at ptr + offset as a chunk */
#define chunk_at_offset(p, s)  ((mchunkptr)(((char*)(p)) + (s)))

/* Set size/use field */
#define set_head(p, s)       ((p)->size = (s))

/* Set size at footer (only when chunk is not in use) */
#define set_foot(p, s)       (((mchunkptr)((char*)(p) + (s)))->prev_size = (s))

```
循环处理unsortd bin中的chunk,最多循环10000次

后续这个if判断，条件很多，分别是，分配的大小属于small bin, unsorted bin中只有一个bin,这个bin是个last_remainder bin, 并且last_raminder bin的大小大于 需要分配的chunk大小加上MINSIZE(64位机上是32byte)
```c
if (in_smallbin_range(nb) && bck == unsorted_chunks(av) && victim == av->last_remainder && (unsigned long)(size) > (unsigned long)(nb + MINSIZE))
```
如果这些条件都满足，就切割这个last_remainder chunk并返回。

否则的话，将其从unsorted bin中移除，如果其大小正好满足要求，设置好相关chunk的metadata并返回
```c
      /* remove from unsorted list */
      unsorted_chunks(av)->bk = bck;
      bck->fd = unsorted_chunks(av);

      /* Take now instead of binning if exact fit */

      if (size == nb) {
	      set_inuse_bit_at_offset(victim, size);
	      if (av != &main_arena)
	        victim->size |= NON_MAIN_ARENA;
	      //...
	      void *p = chunk2mem(victim);
	      if (__builtin_expect (perturb_byte, 0))
	        alloc_perturb (p, bytes);
	      return p;
      }

```
否则的话，加入small bin或large bin中,small bin放入表头，large bin放入合适的位置。
```c
      /* place chunk in bin */
      if (in_smallbin_range(size)) {
	      victim_index = smallbin_index(size);
	      bck = bin_at(av, victim_index);
	      fwd = bck->fd; 
      }
      else {
	      victim_index = largebin_index(size);
	      bck = bin_at(av, victim_index);
	      fwd = bck->fd;
      	/* maintain large bins in sorted order */
      	if (fwd != bck) {
      	  /* Or with inuse bit to speed comparisons */
      	  size |= PREV_INUSE;
      	  /* if smaller than smallest, bypass loop below */
      	  assert((bck->bk->size & NON_MAIN_ARENA) == 0); 
      	  if ((unsigned long)(size) < (unsigned long)(bck->bk->size)) {
      	    fwd = bck;
      	    bck = bck->bk;

      	    victim->fd_nextsize = fwd->fd;
      	    victim->bk_nextsize = fwd->fd->bk_nextsize;
      	    fwd->fd->bk_nextsize = victim->bk_nextsize->fd_nextsize = victim;
      	  }else {
      	    assert((fwd->size & NON_MAIN_ARENA) == 0);
      	    while ((unsigned long) size < fwd->size)
    	      {
          		fwd = fwd->fd_nextsize;
          		assert((fwd->size & NON_MAIN_ARENA) == 0);
    	      }
      	    if ((unsigned long) size == (unsigned long) fwd->size)
      	      /* Always insert in the second position.  */
      	      fwd = fwd->fd;
      	    else{
          		victim->fd_nextsize = fwd;
          		victim->bk_nextsize = fwd->bk_nextsize;
          		fwd->bk_nextsize = victim;
          		victim->bk_nextsize->fd_nextsize = victim;
      	    }
      	    bck = fwd->bk;
      	  }
      	}else
      	  victim->fd_nextsize = victim->bk_nextsize = victim;
      }

      mark_bin(av, victim_index);
      victim->bk = bck;
      victim->fd = fwd;
      fwd->bk = victim;
      bck->fd = victim;

    #define MAX_ITERS	10000
      if (++iters >= MAX_ITERS)  //unsorted bin最多判断10000次
	      break;
    }


#define mark_bin(m,i)    ((m)->binmap[idx2block(i)] |=  idx2bit(i))

```
最后的大括号对应最开始的while循环。

后续代码是unsorted bin处理完之后的逻辑。

如果大小属于largebins的范围，找到对应的large bin, large bin是从大到小排序的，找到大于size的最小chunk，如果需要切割，切割后的chunk放到unsorted bin的表头。
```c

    if (!in_smallbin_range(nb)) {
      bin = bin_at(av, idx);

      /* skip scan if empty or largest chunk is too small */
      if ((victim = first(bin)) != bin && (unsigned long)(victim->size) >= (unsigned long)(nb)) {

	      victim = victim->bk_nextsize; //这个是当前large bin中的最小的bin
	      while (((unsigned long)(size = chunksize(victim)) <(unsigned long)(nb)))
	        victim = victim->bk_nextsize;

	      /* Avoid removing the first entry for a size so that the skip
	      list does not have to be rerouted.  */
	      if (victim != last(bin) && victim->size == victim->fd->size)
	        victim = victim->fd;

	      remainder_size = size - nb;
	      unlink(victim, bck, fwd);

	      /* Exhaust */
	      if (remainder_size < MINSIZE)  {
	        set_inuse_bit_at_offset(victim, size);
	        if (av != &main_arena)
	          victim->size |= NON_MAIN_ARENA;
	      }
	      /* Split */
	      else {
	        remainder = chunk_at_offset(victim, nb);
	  /* We cannot assume the unsorted list is empty and therefore
	     have to perform a complete insert here.  */
	        bck = unsorted_chunks(av);
	        fwd = bck->fd;
          //...
      	  remainder->bk = bck;
      	  remainder->fd = fwd;
      	  bck->fd = remainder;
      	  fwd->bk = remainder;
      	  if (!in_smallbin_range(remainder_size))
      	  {
      	      remainder->fd_nextsize = NULL;
      	      remainder->bk_nextsize = NULL;
      	  }
	        set_head(victim, nb | PREV_INUSE | (av != &main_arena ? NON_MAIN_ARENA : 0));
	        set_head(remainder, remainder_size | PREV_INUSE);
	        set_foot(remainder, remainder_size);
	      }
	      //...
	      void *p = chunk2mem(victim);
	      if (__builtin_expect (perturb_byte, 0))
	        alloc_perturb (p, bytes);
	      return p;
      }
    }

```


后续代码，是要么属于small bins的范围,要么是对应large bin中找不到满足大小的chunk，要从后续的large bin中找

后续处理需要遍历bins,根据bitmap来优化.

如果是small bins中找到的，直接返回chunk即可。如果是从large bins中找到的，肯定是满足要求的最小chunk,也就是说，找到idx之后的第一个bit被设置的链表，取这个链表的最后一个chunk,然后根据大小做切割。这一个逻辑可以满足不管是small bin中获取，还是large bin中获取的所有情况。

这个代码的循环逻辑还是很厉害的
```c
    ++idx; 
    bin = bin_at(av,idx);
    block = idx2block(idx);
    map = av->binmap[block];
    bit = idx2bit(idx);

    for (;;) {

      /* Skip rest of block if there are no more set bits in this block.  */
      if (bit > map || bit == 0) {
	      do {
	        if (++block >= BINMAPSIZE)  /* out of bins */
	          goto use_top;
	      } while ( (map = av->binmap[block]) == 0);

	      bin = bin_at(av, (block << BINMAPSHIFT));
	      bit = 1;
      }

      /* Advance to bin with set bit. There must be one. */
      while ((bit & map) == 0) {
	      bin = next_bin(bin);
	      bit <<= 1;
	      assert(bit != 0);
      }

      /* Inspect the bin. It is likely to be non-empty */
      victim = last(bin);

      /*  If a false alarm (empty bin), clear the bit. */
      if (victim == bin) {
	      av->binmap[block] = map &= ~bit; /* Write through */
	      bin = next_bin(bin);
	      bit <<= 1;
      }
      else {
	      size = chunksize(victim);

  	    /*  We know the first chunk in this bin is big enough to use. */
  	    assert((unsigned long)(size) >= (unsigned long)(nb));

  	    remainder_size = size - nb;

	      /* unlink */
	      unlink(victim, bck, fwd);

  	    if (remainder_size < MINSIZE) {
  	      set_inuse_bit_at_offset(victim, size);
  	      if (av != &main_arena)
  	        victim->size |= NON_MAIN_ARENA;
  	    }else {
          remainder = chunk_at_offset(victim, nb);

    	    /* We cannot assume the unsorted list is empty and therefore
    	     have to perform a complete insert here.  */
    	    bck = unsorted_chunks(av);
    	    fwd = bck->fd;
          //...
    	    remainder->bk = bck;
    	    remainder->fd = fwd;
    	    bck->fd = remainder;
    	    fwd->bk = remainder;

    	    /* advertise as last remainder */
    	    if (in_smallbin_range(nb))
    	      av->last_remainder = remainder;
    	    if (!in_smallbin_range(remainder_size))
    	    {
    	      remainder->fd_nextsize = NULL;
    	      remainder->bk_nextsize = NULL;
    	    }
    	    set_head(victim, nb | PREV_INUSE |(av != &main_arena ? NON_MAIN_ARENA : 0));
    	    set_head(remainder, remainder_size | PREV_INUSE);
    	    set_foot(remainder, remainder_size);
  	    }
	      //...
	      void *p = chunk2mem(victim);
	      if (__builtin_expect (perturb_byte, 0))
	        alloc_perturb (p, bytes);
	      return p;
      }
    }

```


到这里，就从top中找了

```c
  use_top:
    victim = av->top;
    size = chunksize(victim);  //如果是main arena,top chunk没有初始化，只是被设置为了unsorted bin,也就是bins数组的起始地址-16， 这里chunksize返回的是main arena中last_remainder的值，显然，也是0.

    if ((unsigned long)(size) >= (unsigned long)(nb + MINSIZE)) {
      remainder_size = size - nb;
      remainder = chunk_at_offset(victim, nb);
      av->top = remainder;
      set_head(victim, nb | PREV_INUSE |(av != &main_arena ? NON_MAIN_ARENA : 0));
      set_head(remainder, remainder_size | PREV_INUSE);

      //...
      void *p = chunk2mem(victim);
      if (__builtin_expect (perturb_byte, 0))
	      alloc_perturb (p, bytes);
      return p;
    }

    /* When we are using atomic ops to free fast chunks we can get
       here for all block sizes.  */
    else if (have_fastchunks(av)) {
      malloc_consolidate(av);
      /* restore original bin index */
      if (in_smallbin_range(nb))
	      idx = smallbin_index(nb);
      else
	      idx = largebin_index(nb);
    }

    /*
       Otherwise, relay to handle system-dependent cases
    */
    else {
      void *p = sysmalloc(nb, av);
      if (p != NULL && __builtin_expect (perturb_byte, 0))
	      alloc_perturb (p, bytes);
      return p;
    }
  } //对应for(;;),开始unsorted bin 处理那里
}//_int_malloc结束

```



###  **sysmalloc**
该函数只在这一处被调用，也就是top也无法满足分配时。

这个函数也很长，我们分段来看

首先还是定义一堆函数局部变量
```c
static void* sysmalloc(INTERNAL_SIZE_T nb, mstate av)
{
  mchunkptr       old_top;        /* incoming value of av->top */
  INTERNAL_SIZE_T old_size;       /* its size */
  char*           old_end;        /* its end address */

  long            size;           /* arg to first MORECORE or mmap call */
  char*           brk;            /* return value from MORECORE */

  long            correction;     /* arg to 2nd MORECORE call */
  char*           snd_brk;        /* 2nd return val */

  INTERNAL_SIZE_T front_misalign; /* unusable bytes at front of new space */
  INTERNAL_SIZE_T end_misalign;   /* partial page left at end of new space */
  char*           aligned_brk;    /* aligned offset into brk */

  mchunkptr       p;              /* the allocated/returned chunk */
  mchunkptr       remainder;      /* remainder from allocation */
  unsigned long   remainder_size; /* its size */

  unsigned long   sum;            /* for updating stats */

  size_t          pagemask  = GLRO(dl_pagesize) - 1;
  bool            tried_mmap = false;

```
如果大小大于mmap_shreshold,并且mmap次数满足最大mamp次数，直接调用mmap分配chunk。需要注意的是，最后的chunk的metadata中要设置好IS_MMAPPED标志，表示这是一个mmaped chunk。 
```c

  if ((unsigned long)(nb) >= (unsigned long)(mp_.mmap_threshold) &&
      (mp_.n_mmaps < mp_.n_mmaps_max)) {

    char* mm;             /* return value from mmap call*/

  try_mmap:
    //mmap的大小要是page size的整数倍。
    if (MALLOC_ALIGNMENT == 2 * SIZE_SZ)
      size = (nb + SIZE_SZ + pagemask) & ~pagemask;
    else
      size = (nb + SIZE_SZ + MALLOC_ALIGN_MASK + pagemask) & ~pagemask;
    tried_mmap = true;

    /* Don't try if size wraps around 0 */
    if ((unsigned long)(size) > (unsigned long)(nb)) {

      mm = (char*)(MMAP(0, size, PROT_READ|PROT_WRITE, 0));

      if (mm != MAP_FAILED) {
            //对齐有关的设置....
            set_head(p, size|IS_MMAPPED);
        //更新一些统计数据...
	    return chunk2mem(p);
      }
    }
  }
```

如果不能直接调用mmap获取mmaped chunk返回，就需要拓展top chunk了。

首先看如果是non main arena如何拓展
```c
  old_top  = av->top;
  old_size = chunksize(old_top);
  old_end  = (char*)(chunk_at_offset(old_top, old_size));
  //首先将brk和snd_brk初始化为0，
  brk = snd_brk = (char*)(MORECORE_FAILURE);

   //如果是main arena,第一次调用malloc,其top没有初始化，只是临时指向了bins的起始地址减去16，也就是top字段的地址
 //如果是非main arena,在非main arena创建的时候就初始化好了top,其中pre_inuse metadata被设置了。
  assert((old_top == initial_top(av) && old_size == 0) ||
	 ((unsigned long) (old_size) >= MINSIZE &&
	  prev_inuse(old_top) &&
	  ((unsigned long)old_end & pagemask) == 0));

  assert((unsigned long)(old_size) < (unsigned long)(nb + MINSIZE));

   if (av != &main_arena) {
    //这里面是非main_arena的处理
    heap_info *old_heap, *heap;
    size_t old_heap_size;
    old_heap = heap_for_ptr(old_top); //根据当前top地址，取出所在heap的起始地址
    old_heap_size = old_heap->size;
    if ((long) (MINSIZE + nb - old_size) > 0 && grow_heap(old_heap, MINSIZE + nb - old_size) == 0) {
        //直接在之前的heap上拓展
      av->system_mem += old_heap->size - old_heap_size;
      arena_mem += old_heap->size - old_heap_size;
      set_head(old_top, (((char *)old_heap + old_heap->size) - (char *)old_top) | PREV_INUSE);
    }
    else if ((heap = new_heap(nb + (MINSIZE + sizeof(*heap)), mp_.top_pad))) {
      //分配新的heap
      heap->ar_ptr = av;
      heap->prev = old_heap;  //non main arena所有分配的heap都链接成链表
      av->system_mem += heap->size;
      arena_mem += heap->size;
      /* Set up the new top.  */
      top(av) = chunk_at_offset(heap, sizeof(*heap));
      set_head(top(av), (heap->size - sizeof(*heap)) | PREV_INUSE);

      //之前的heap的top chunk中，设置fencepost chunk,剩下的chunk free掉
      old_size = (old_size - MINSIZE) & ~MALLOC_ALIGN_MASK;
      set_head(chunk_at_offset(old_top, old_size + 2*SIZE_SZ), 0|PREV_INUSE);
      if (old_size >= MINSIZE) {
	    set_head(chunk_at_offset(old_top, old_size), (2*SIZE_SZ)|PREV_INUSE);
	    set_foot(chunk_at_offset(old_top, old_size), (2*SIZE_SZ));
	    set_head(old_top, old_size|PREV_INUSE|NON_MAIN_ARENA);
	    _int_free(av, old_top, 1);
      } else {
	    set_head(old_top, (old_size + 2*SIZE_SZ)|PREV_INUSE);
	    set_foot(old_top, (old_size + 2*SIZE_SZ));
      }
    }
    else if (!tried_mmap)
      /* We can at least try to use to mmap memory.  */
      goto try_mmap;

  } else { /* av == main_arena */
  //...
  }


#define heap_for_ptr(ptr) \
 ((heap_info *)((unsigned long)(ptr) & ~(HEAP_MAX_SIZE-1)))

```
在non main arena中分配新的heap时，每个heap映射的地址空间大小是HEAP_MAX_SIZE，在64位机上是64M,只不过除了需要的size地址空间被设置为可读写，剩下的空间为不可访问。如果heap可以拓展，只需要将所需的额外空间设置为可读写就可以了
```c
static int
grow_heap(heap_info *h, long diff)
{
  size_t page_mask = GLRO(dl_pagesize) - 1;
  long new_size;
  diff = (diff + page_mask) & ~page_mask;
  new_size = (long)h->size + diff;
  if((unsigned long) new_size > (unsigned long) HEAP_MAX_SIZE)
    return -1;
  if((unsigned long) new_size > h->mprotect_size) {
    if (__mprotect((char *)h + h->mprotect_size,
		   (unsigned long) new_size - h->mprotect_size,
		   PROT_READ|PROT_WRITE) != 0)
      return -2;
    h->mprotect_size = new_size;
  }

  h->size = new_size;
  return 0;
}
```

再看如果是main arena如何拓展top chunk
```c
  if (av != &main_arena) {
    //....

  } else { /* av == main_arena */

    size = nb + mp_.top_pad + MINSIZE;

    //main arena被初始化为contiguous arena,如果在某次调用sbrk拓展top时失败了，而后续调用mmap重新设置top成功，则会将main arena改为uncontiguous arena
    //如果是contiguous,则说明main arena一直都使用的堆空间，其地址是连续的。
    if (contiguous(av))
        size -= old_size;

    size = (size + pagemask) & ~pagemask;

    if (size > 0)
        brk = (char*)(MORECORE(size)); //这里调用sbrk系统调用拓展堆地址空间，如果成功，返回拓展的地址空间的起始地址。需要注意的是，不管main arena当前的top是堆地址空间的top,还是memroy map地址空间的top，拓展top首先都是调用的sbrk.

    if (brk != (char*)(MORECORE_FAILURE)) {
        /* Call the `morecore' hook if necessary.  */
        void (*hook) (void) = force_reg (__after_morecore_hook);
        if (__builtin_expect (hook != NULL, 0))
            (*hook) ();
    } else {
        if (contiguous(av))
            size = (size + old_size + pagemask) & ~pagemask;

        //MMAP_AS_MORECORE_SIZE是1M
        if ((unsigned long)(size) < unsigned long(MMAP_AS_MORECORE_SIZE))
            size = MMAP_AS_MORECORE_SIZE;

        if ((unsigned long)(size) > (unsigned long)(nb)) {

            char *mbrk = (char*)(MMAP(0, size, PROT_READ|PROT_WRITE, 0));

            if (mbrk != MAP_FAILED) {
                brk = mbrk;
                snd_brk = brk + size;
                set_noncontiguous(av);
            }
        }
    }
    //brk!=0,即要么sbrk调用成功，要么mmap调用成功
    if (brk != (char*)(MORECORE_FAILURE)) {
        //...
        av->system_mem += size;
        if (brk == old_end && snd_brk == (char*)(MORECORE_FAILURE))
            //调用sbrk成功，并且，brk == old_end, 直接拓展之前的top即可
            set_head(old_top, (size + old_size) | PREV_INUSE);
        else if (contiguous(av) && old_size && brk < old_end) {
            malloc_printerr (3, "break adjusted to free malloc space", brk);
        }
        else {
            //这里有三种情况，一是调用mmap 成功，二是调用sbrk成功，但是之前的top不是堆地址空间的top，三是调用sbrk成功，之前的top也是堆地址空间的top，但是地址不连续。这可能是因为用户程序中除了调用ptmalloc库分配内存之外，也在别的地方调用了brk系统调用

            front_misalign = 0;
            end_misalign = 0;
            correction = 0;
            aligned_brk = brk;

            if (contiguous(av)) {
              //这个分支对应sbrk调用成功，之前的top也是堆地址空间的top,但是地址却不连续。也就是用户程序中别的地方调用了brk,处理略
            }else {
              	if (MALLOC_ALIGNMENT == 2 * SIZE_SZ)
	                assert(((unsigned long)chunk2mem(brk) & MALLOC_ALIGN_MASK) == 0);
	              else {
	                front_misalign = (INTERNAL_SIZE_T)chunk2mem(brk) & MALLOC_ALIGN_MASK;
	                if (front_misalign > 0) {
	                  aligned_brk += MALLOC_ALIGNMENT - front_misalign;
	                }
	              }       
	              if (snd_brk == (char*)(MORECORE_FAILURE)) {
                  //这里对应的就是，调用sbrk成功，但是之前使用的top不是堆空间的top.
                   snd_brk = (char*)(MORECORE(0));
	              }
            }
            if (snd_brk != (char*)(MORECORE_FAILURE)) {
                av->top = (mchunkptr)aligned_brk;
                set_head(av->top, (snd_brk - aligned_brk + correction) | PREV_INUSE);
                av->system_mem += correction;

                //top chunk已经更新到新的一块地址空间了，之前的top chunk,设置两个fencepost chunk,剩下的chunk free掉
                if (old_size != 0) {
                    old_size = (old_size - 4*SIZE_SZ) & ~MALLOC_ALIGN_MASK;
                    set_head(old_top, old_size | PREV_INUSE);
                    chunk_at_offset(old_top, old_size)->size =
                        (2*SIZE_SZ)|PREV_INUSE;
                    chunk_at_offset(old_top, old_size + 2*SIZE_SZ)->size =
                        (2*SIZE_SZ)|PREV_INUSE;
                    if (old_size >= MINSIZE) {
                        _int_free(av, old_top, 1);
                    }

                }
            }
        }
    }

 } //av==main_arena

#define MORECORE_FAILURE 0
#define MMAP_AS_MORECORE_SIZE (1024 * 1024)
```

可以看到，main arena在拓展top的时候，首先调用的是sbrk,如果sbrk失败，会再试图调用mmap。这种混用sbrk和mmap的机制，在收缩的时候是不好处理的，main arena也没有像非main arena那样将分配的内存用heap来链接，main arena在top 收缩的时候，判断如下
```c
current_brk = (char*)(MORECORE(0));
if (current_brk == (char*)(av->top) + top_size){
  //收缩top逻辑...
}
```
也就是只会收缩堆地址空间分配的top。 mmap分配的top是不会去收缩的。而且这种处理逻辑，在某些情况下也会造成堆地址空间分配的top也不会收缩。也就是说，main arena中很可能会缓存很多空闲的内存而不去收缩。

其实不用sbrk,只调用mmap分配内存是最好的，尤其是在64位机的情况下。

拓展完top后就从top中取出需要的chunk返回
```c

  /* finally, do the allocation */
    p = av->top;
    size = chunksize(p);

  /* check that one of the above allocation paths succeeded */
    if ((unsigned long)(size) >= (unsigned long)(nb + MINSIZE)) {
        remainder_size = size - nb;
        remainder = chunk_at_offset(p, nb);
        av->top = remainder;
        set_head(p, nb | PREV_INUSE | (av != &main_arena ? NON_MAIN_ARENA : 0));
        set_head(remainder, remainder_size | PREV_INUSE);
        check_malloced_chunk(av, p, nb);
        return chunk2mem(p);
    }
    __set_errno (ENOMEM);
    return 0;
} //sysmalloc
```

至此，分配的代码就看完了。



### **__libc_free**
当用户调用glibc的free函数，实际调用的是__libc_free

```c
void
__libc_free(void* mem)
{
  mstate ar_ptr;
  mchunkptr p;                         
  //有hook,直接调用hook返回
  void (*hook) (__malloc_ptr_t, const __malloc_ptr_t)
    = force_reg (__free_hook);
  if (__builtin_expect (hook != NULL, 0)) {
    (*hook)(mem, RETURN_ADDRESS (0));
    return;
  }

  if (mem == 0)                              /* free(0) has no effect */
    return;

  p = mem2chunk(mem);

  if (chunk_is_mmapped(p))                       /* release mmapped memory. */
  {
    //mp_.mmap_threshold和mp_.trim_threshold的动态调整机制
    if (!mp_.no_dyn_threshold
	&& p->size > mp_.mmap_threshold
	&& p->size <= DEFAULT_MMAP_THRESHOLD_MAX)
      {
	mp_.mmap_threshold = chunksize (p);
	mp_.trim_threshold = 2 * mp_.mmap_threshold;
      }
    munmap_chunk(p);
    return;
  }

  ar_ptr = arena_for_chunk(p);
  _int_free(ar_ptr, p, 0);
}

#define arena_for_chunk(ptr) \
 (chunk_non_main_arena(ptr) ? heap_for_ptr(ptr)->ar_ptr : &main_arena)
```

### **int_free**

以下代码中省略了跟lock相关的设置，以及一些错误检查。
在将chunk放入fastbin中时使用的是cas,没有加锁。
其他时候该加锁的要加锁。
```c
static void
_int_free(mstate av, mchunkptr p, int have_lock)
{
  INTERNAL_SIZE_T size;        /* its size */
  mfastbinptr*    fb;          /* associated fastbin */
  mchunkptr       nextchunk;   /* next contiguous chunk */
  INTERNAL_SIZE_T nextsize;    /* its size */
  int             nextinuse;   /* true if nextchunk is used */
  INTERNAL_SIZE_T prevsize;    /* size of previous contiguous chunk */
  mchunkptr       bck;         /* misc temp for linking */
  mchunkptr       fwd;         /* misc temp for linking */

  const char *errstr = NULL;
  int locked = 0;

  size = chunksize(p);

//....



   if ((unsigned long)(size) <= (unsigned long)(get_max_fast ())) {

//...
        set_fastchunks(av);
        unsigned int idx = fastbin_index(size);
        fb = &fastbin (av, idx);

        mchunkptr fd;
        mchunkptr old = *fb;
        do{
            //...
            p->fd = fd = old;
          }while ((old = catomic_compare_and_exchange_val_rel (fb, p, fd)) != fd);
    }


    else if (!chunk_is_mmapped(p)) {
        //...
        nextchunk = chunk_at_offset(p, size);
        //...

        nextsize = chunksize(nextchunk);
        //...

        /* consolidate backward */
        if (!prev_inuse(p)) {
            prevsize = p->prev_size;
            size += prevsize;
            p = chunk_at_offset(p, -((long) prevsize));
            unlink(p, bck, fwd);
        }

        if (nextchunk != av->top) {
            /* get and clear inuse bit */
            nextinuse = inuse_bit_at_offset(nextchunk, nextsize);

            /* consolidate forward */
            if (!nextinuse) {
                unlink(nextchunk, bck, fwd);
                size += nextsize;
            } else
                clear_inuse_bit_at_offset(nextchunk, 0);

            bck = unsorted_chunks(av);
            fwd = bck->fd;
            p->fd = fwd;
            p->bk = bck;
            if (!in_smallbin_range(size))
            {
                p->fd_nextsize = NULL;
                p->bk_nextsize = NULL;
            }
            bck->fd = p;
            fwd->bk = p;

            set_head(p, size | PREV_INUSE);
            set_foot(p, size);

        }else {
            size += nextsize;
            set_head(p, size | PREV_INUSE);
            av->top = p;
        }
        if ((unsigned long)(size) >= FASTBIN_CONSOLIDATION_THRESHOLD) {
            if (have_fastchunks(av))
                malloc_consolidate(av);

            if (av == &main_arena) {
                if ((unsigned long)(chunksize(av->top)) >= (unsigned long)(mp_.trim_threshold))
                    systrim(mp_.top_pad, av);

            } else {
                heap_info *heap = heap_for_ptr(top(av));
                assert(heap->ar_ptr == av);
                heap_trim(heap, mp_.top_pad);
            }
        }

            //...
    }else {
        munmap_chunk (p);
    }
}

#define FASTBIN_CONSOLIDATION_THRESHOLD  (65536UL)

```
1. 如果释放的chunk大小小于max_fast,则使用cas直接放入到fastbin中。否则进入下一步
2. 判断释放的chunk是否是mmaped chunk，如果是，进入下一步.否则跳到第6步
3. 判断之前的chunk是否空闲，如果是空闲的，则合并
4. 再判断其下一个chunk是否是top chunk，如果不是top chunk，并且是空闲的，也要合并。并将合并后的chunk放入unsorted bin中。如果是top chunk,将该chunk合并入top chunk。
5.  最后判断合并后的chunk大小，如果大于FASTBIN_CONSOLIDATION_THRESHOLD，则执行top chunk的收缩工作。在执行收缩工作之前，如果fast bin中有chunk，则执行fastbin chunk的合并。
6.  对于mmaped chunk,直接unmap就可以了。


### **systrim**
如果是main arena,并且main arena的top chunk大于mp_.trim_threshold，则调用systrim收缩,pad是需要留下的大小。
```c
static int systrim(size_t pad, mstate av)
{
  long  top_size;        /* Amount of top-most memory */
  long  extra;           /* Amount to release */
  long  released;        /* Amount actually released */
  char* current_brk;     /* address returned by pre-check sbrk call */
  char* new_brk;         /* address returned by post-check sbrk call */
  size_t pagesz;

  pagesz = GLRO(dl_pagesize);
  top_size = chunksize(av->top);

  /* Release in pagesize units, keeping at least one page */
  extra = (top_size - pad - MINSIZE - 1) & ~(pagesz - 1);

  if (extra > 0) {

    //如果当前top属于堆地址空间，才收缩
    current_brk = (char*)(MORECORE(0));
    if (current_brk == (char*)(av->top) + top_size) {



      MORECORE(-extra);

      void (*hook) (void) = force_reg (__after_morecore_hook);
      if (__builtin_expect (hook != NULL, 0))
	      (*hook) ();

      new_brk = (char*)(MORECORE(0));

      if (new_brk != (char*)MORECORE_FAILURE) {
	      released = (long)(current_brk - new_brk);

    	  if (released != 0) {
    	    /* Success. Adjust top. */
    	    av->system_mem -= released;
    	    set_head(av->top, (top_size - released) | PREV_INUSE);
    	    //...
    	    return 1;
    	  }
      }
    }
  }
  return 0;
}
```

### **heap_trim**
```c
static int
internal_function
heap_trim(heap_info *heap, size_t pad)
{
  mstate ar_ptr = heap->ar_ptr;
  unsigned long pagesz = GLRO(dl_pagesize);
  mchunkptr top_chunk = top(ar_ptr), p, bck, fwd;
  heap_info *prev_heap;
  long new_size, top_size, extra, prev_size, misalign;

  /* Can this heap go away completely? */
  while(top_chunk == chunk_at_offset(heap, sizeof(*heap))) {
    prev_heap = heap->prev;
    prev_size = prev_heap->size - (MINSIZE-2*SIZE_SZ);
    p = chunk_at_offset(prev_heap, prev_size); 
    //...
    assert(p->size == (0|PREV_INUSE)); /* must be fencepost */
    p = prev_chunk(p);
    new_size = chunksize(p) + (MINSIZE-2*SIZE_SZ) + misalign;
    assert(new_size>0 && new_size<(long)(2*MINSIZE));
    if(!prev_inuse(p))
      new_size += p->prev_size;
    assert(new_size>0 && new_size<HEAP_MAX_SIZE); //到目前为止，这个new size 是prev heap中top chunk的大小，如果这个大小满足收缩后剩余大小的要求，则当前这个heap可以完全释放回系统
    if(new_size + (HEAP_MAX_SIZE - prev_heap->size) < pad + MINSIZE + pagesz)
      break;
    ar_ptr->system_mem -= heap->size;
    arena_mem -= heap->size;
    delete_heap(heap);
    heap = prev_heap;
    if(!prev_inuse(p)) { /* consolidate backward */
      p = prev_chunk(p);
      unlink(p, bck, fwd); //需要并入top chunk中，从当前空闲bin中去除
    }
    assert(((unsigned long)((char*)p + new_size) & (pagesz-1)) == 0);
    assert( ((char*)p + new_size) == ((char*)heap + heap->size) );
    top(ar_ptr) = top_chunk = p;
    set_head(top_chunk, new_size | PREV_INUSE);
  }
  top_size = chunksize(top_chunk);
  extra = (top_size - pad - MINSIZE - 1) & ~(pagesz - 1);
  if(extra < (long)pagesz)
    return 0;
  /* Try to shrink. */
  if(shrink_heap(heap, extra) != 0)
    return 0;
  ar_ptr->system_mem -= extra;
  arena_mem -= extra;

  /* Success. Adjust top accordingly. */
  set_head(top_chunk, (top_size - extra) | PREV_INUSE);
  /*check_chunk(ar_ptr, top_chunk);*/
  return 1;
}

#define prev_chunk(p) ((mchunkptr)( ((char*)(p)) - ((p)->prev_size) ))

static int
shrink_heap(heap_info *h, long diff)
{
  long new_size;

  new_size = (long)h->size - diff;
  if(new_size < (long)sizeof(*h))
    return -1;
  if (__builtin_expect (check_may_shrink_heap (), 0))
  {
    //将这段地址空间重新映射为不可访问。
    if((char *)MMAP((char *)h + new_size, diff, PROT_NONE,MAP_FIXED) == (char *) MAP_FAILED)
	      return -2;
    h->mprotect_size = new_size;
  }
  else
    //只是给操作系统一个hint,表示程序在将来的一段时间不会再访问这段地址空间，其对应的物理页面可以被回收。
    //但如果后续程序中还是访问到了这段地址空间，也不会出错。
    __madvise ((char *)h + new_size, diff, MADV_DONTNEED);

  h->size = new_size;
  return 0;
}

#define delete_heap(heap) \
  do {								\
    if ((char *)(heap) + HEAP_MAX_SIZE == aligned_heap_area)	\
      aligned_heap_area = NULL;					\
    __munmap((char*)(heap), HEAP_MAX_SIZE);			\
  } while (0)


/* Force an unmap when the heap shrinks in a secure exec.  This ensures that
   the old data pages immediately cease to be accessible.  */
static inline bool
check_may_shrink_heap (void)
{
  return __libc_enable_secure;
}

```

到这里，free chunk的代码就结束了。
















1. 线程先查看线程本地存储中指向分配区的指针，是否已经指向一个分配区，如果是，尝试对该分配区加锁，如果加锁成功，使用该分配区分配内存，否则，该线程搜索分配区循环链表试图获得一个空闲（没有加锁）的分配区。如果所有的分配区都已经加锁，那么 ptmalloc 会开辟一个新的分配区，把该分配区加入到分配区循环链表中。当确定使用哪个分配区并对其加锁成功后，将线程本地存储中指向分配区的指针设置为该分配区的地址。然后使用该分配区进行分配操作。

2.  将用户的请求大小转换为实际需要分配的 chunk 空间大小。

3.  判断所需分配chunk的大小是否满足chunk_size <= max_fast (max_fast 默认为 64B)，
如果是的话，则转下一步，否则跳到第 5 步。

1. 首先尝试在 fast bins 中取一个所需大小的 chunk 分配给用户。如果可以找到，则分
配结束。否则转到下一步。

1. 如果所需分配的 chunk 的大小属于small bins的范围，从对应的bin 的尾部摘取一个恰好满足大小的 chunk。

2. 到这一步，要么是small bins中找不到合适的，要么是分配的大小大于small bins的范围，ptmalloc 首先会遍历 fast bins 中的 chunk，将相邻的 chunk 进行合并，并链接到 unsorted bin 中，然后遍历 unsorted bin 中的 chunk,

如果 unsorted bin 只有一个 chunk，并且这个 chunk 在上次分配时被使用过，并且所需分配的 chunk 大小属于 small bins，并且 该chunk 的大小大于等于需要分配的大小，这种情况下就直 接将该 chunk 进行切割，分配结束，否则将遍历的每个 chunk 根据其的空间大小将其放入 small bins 或是 large bins 中，遍历完成后，转入下一步

7. 从 bins 中按照“smallest-first，best-fit”原则，找一个合适的 chunk，从
中划分一块所需大小的 chunk，并将剩下的部分链接回到 bins 中。若操作成功，则
分配结束，否则转到下一步。
1. 如果搜索 fast bins 和 bins 都没有找到合适的 chunk，那么就需要操作 top chunk 来
进行分配了。判断 top chunk 大小是否满足所需 chunk 的大小，如果是，则从 top
chunk 中分出一块来。否则转到下一步。
1. 到了这一步，说明 top chunk 也不能满足分配要求，所以，于是就有了两个选择: 如
果是主分配区，调用 sbrk()，增加 top chunk 大小；如果是非主分配区，调用 mmap
来分配一个新的 sub-heap，增加 top chunk 大小；或者使用 mmap()来直接分配。在
这里，需要依靠 chunk 的大小来决定到底使用哪种方法。判断所需分配的 chunk
大小是否大于等于 mmap 分配阈值，如果是的话，则转下一步，调用 mmap 分配，
否则跳到第 12 步，增加 top chunk 的大小。
1.  使用 mmap 系统调用为程序的内存空间映射一块 chunk_size align 4kB 大小的空间。
然后将内存指针返回给用户。
1.  判断是否为第一次调用 malloc，若是主分配区，则需要进行一次初始化工作，分配一块大小为(chunk_size + 128KB) align 4KB 大小的空间作为初始的 heap。若已经初
始化过了，主分配区则调用 sbrk()增加 heap 空间，非主分配区则在 top chunk 中切
割出一个 chunk，使之满足分配需求，并将内存指针返回给用户。





