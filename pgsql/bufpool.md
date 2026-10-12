bufpool有两个地方承载
1.共享内存 
2.每个进程私有的bufpool(local buffer)，用于临时表，同时不需要wal日志跟checkpoint

关键数据结构
BufferTag
相当于pageid，用于唯一标识
state
64bit的标识位，包含了pin数目、时钟扫描计数、buf的状态（脏页，无效页等）、内容锁计数器，使用一个uint64因此可以原子性更改
BM_LOCKED占其中的1bit，用于保护bufdesc，state为violate变量，原子操作使用__sync_val_compare_and_swap（自带全屏障），以此实现bufdesc的并发控制
置位 BM_LOCKED 时，表示当前有进程持有缓冲区头部锁，拥有对 state 的独占修改权；其他进程看到后必须等待，不能修改 state。不置位时，多个进程可以通过 CAS 竞争修改 state，所以“有可能可以更改”，但必须遵循 CAS 协议。
在大多数热路径上（如 Pin/Unpin），对 Refcount 的修改通过原子加减完成，无需获取缓冲区头锁（同时需要保证此时lock标记为不为1，避免lock的语义被破坏），这是性能的关键
volatile作用：
避免编译器的优化，保证编译器每次读取都会生成读取指令，而非从cpu寄存器中直接拿到值，但是虽然会生成读取指令，结果可能来源于内存或者cpu cache，因此其余cpu无法立刻感知更改
寄存器由编译器/汇编显示管理，就是避免一直复用某个寄存器的值
内存屏障作用：保证屏障前与后的内存读写顺序不会被打乱

Clock-Sweep Algorithm
在bufpool初始化的时候，初始化连续的bufdesc数组，有一个时钟指针（数组下标），指向下个可能淘汰的页，数组下标使用原子操作进行更新，避免了加锁
每个进程原子性的获取下标并给其+1（CAS），随后检查拿到的那个下标对应的desc是否满足refcount为0，不为0则跳过（说明还有进程在持有这个页），随后检查usagecount（时钟扫描计数），如果不为0则usagecount--并扫描下一个
如果usagecount为0，则先给bufdesc的refcount++，保证不会被淘汰，随后所属权改成自己
CAS 可能失败需重试、数组下标回绕需同步 completePasses（当前访问了多少圈，用于给后台写入进程来评估扫描进度，从而调整刷脏节奏。）
trycounter 防止无限扫描，trycounter 会在refcount不为0的时候--，在refcount为0&&usage不为赋值为NBuffers（数组大小），当trycounter 为0说明所有页都被pin住，此时会中止扫描（同时有一条异常日志）
当以normal模式访问页的时候，会调用pinbuffer，将refcount跟usagecount都++。


buf访问策略
批量读（BAS_BULKREAD）、批量写（BAS_BULKWRITE）、for vacuum（BAS_VACUUM），读的buf最小256KB，写的最大16MB，vacuum 2MB
对于批量读跟批量写，会有专门的读写ring buf进行淘汰管理，ring buf是每个进程私有的动态内存，里面不存放页面内存，而是存放了bufid
ring buf由GetAccessStrategy申请，由FreeAccessStrategy进行释放，生命周期由调用方进行管理，是操作级别，比如heap_beginscan开始的时候申请，在scanend的时候释放
当有ringbuf时，如果是BAS_BULKREAD方式拿页，那么会优先从ringbuf寻找可淘汰的页，当ringbuf对应的页都被pin住，就会回退为普通的clock淘汰，同时，BAS_BULKREAD拿页时只会在usagecount为0时才会++至1，而如果使用normal拿同一个页，最多可以到5

io
bufpool通过readstream实现了异步io，readstream提供了两个接口，一个是startRead（提交读页申请），一个是waitforread（等待读页完成），io_method可以设置为：
sync：同步io，就是进程自己读，调用pread，调用的时候，系统会归还cpu资源
worker：使用专门的io进程进行读取
io_uring: readstream在linux通过io_uring实现异步io，io_uring有两个队列(用户态与内核态共享的两个无锁环形缓冲区，通过mmap共享)，一个提交队列（SQ），一个完成队列（CQ），用户将请求放到SQ中，操作系统读完后，将结果放到CQ，可能会并发读取，因此CQ中的顺序与SQ未必一致，对于用户而言，有专门的检查缓冲区的接口跟示意缓冲区消费完成的接口，通过检查接口判断是否有新的io完成了，检查接口有阻塞式（io_uring_wait_cqe）跟非阻塞式（io_uring_peek_cqe），对于阻塞式接口，内核会将线程挂起，完成再唤醒，每个进程都会创建一个iouring实例，但是是放到共享内存的，提交io仅允许在自己的ioring实例上进行，但是检查io是否完成允许其余进程进行辅助（使用lwlock进行保护），提交隔离是为了降低提交竞争，检查共享是为了避免互相等待死锁，并及时获取结果
startRead：将pageid批量传入，readstream内部可能会实现预读+批量读取，（分别怎么实现？），随后接口返回
传入后，readstream不会主动查询io是否完成，上层调用时，才会处理，同时处理一批数据
waitforread：检查bufdesc的状态（BM_IO_IN_PROGRESS ），是否还为io进行中，如果是非阻塞式检查一个ioring，看是否有新的io结果（可查看其余进程发起的io_uring）
1.如果有，就会使用io_uring_cq_ready检查当前有多少个结果了，然后使用io_uring_peek_batch_cqe将结果都取出来，对于每个结果，io_uring_cqe_seen，并设置页valid与!BM_IO_IN_PROGRESS，然后通过唤醒等待队列里的每个进程，一次会将队列内的全部去除放到bufpool，不会刻意只等一个
2.如果没有就会pgaio_wref_wait，bufdesc上承载了PgAioWaitRef（io的引用或者说索引），通过索引，找到PgAioHandle（io句柄），PgAioHandle承载了io的状态、目标文件、偏移量等信息，同时还承载了条件变量（本质就是锁保护的等待队列），等待队列的元素为进程号，PgAioHandle是进程共享的，wait就是把自己加入到等待队列，随后让出cpu资源，wait底层由重量级锁（latch）支持


bufdesc把本进程加入到PgAioHandle的等待队列里，当io直接结束，



ReadStream 
预读  主要收益对象：顺序扫描，位图堆扫描，VACUUM与ANALYZE，索引预加载

6.双层buf，跟系统的缓存怎么交互
7.动态调整怎么实现
8.读写页怎么控制的