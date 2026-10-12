postmaster 进程，是主进程
1.初始化内存跟信号量池
2.监听前端请求
3.创建用于处理成功认证的后台进程
4.管理全局操作，类似于启动、恢复等
5.这个进程不参与共享内存的访问，以此避免了后台进程崩溃的影响

postgres每个链接都是一个进程，这保证其可以在不支持多线程的环境上运行（SSL or PAM）

辅助进程：
WAL Summarizer：扫描wal日志并发给备库
Background Workers: 通过拓展来支持的可插拔worker
Parallel Workers：给并行查询用的工作进程

共享内存：
有一个固定大小的共享内存段
初始化流程：
各模块通过RegisterShmemCallbacks注册回调函数
计算大小，随后申请并初始化
共享实体：
ProcGlobal：一个用于保存每个进程状态（PROC_HDR）的数组
WalSummarizerCtl：WAL summarization的控制状态
BackgroundWorkerData：可插拔worker的槽位
通过Lightweight Locking (LWLock)来控制共享内存的并发
对于不支持fork的系统，有一个EXEC_BACKEND 模式，使用CreateProcess（windows上的创建进程）


术语：
OID：
是"一个具体对象"的编号，不是"一类对象"的编号。所以：表这个概念没有 OID；每一张具体的表有自己的 OID（就是 pg_class.oid，等价于你说的 tableid）；每一个具体的函数也有自己的OID。而且由于一个表会顺带产生行类型、数组类型、索引等对象，一次 CREATE TABLE 实际会消耗多个 OID。
MultiXactId：
是**"一组事务"的编号**。它存在的唯一理由是：元组头里只够放一个 xmax 字段，但一行可能同时被多个事务锁住。
  - 元组头的 xmax 里放一个 MultiXactId，同时元组头中有个标志位声明"这个 xmax 是 Multixact，不是普通事务 ID"。这就是读元组时第一个要判断的分叉。
  - 名单本身存在数据目录下的 pg_multixact/ 里。所以元组里只有一个数字，成员列表在外部。
LogSequenceNumber (LSN)：
wal使用的日志序列号
ItemPointer (TID)：行号，(BlockNumber, OffsetNumber)


mvcc术语
Visibility Map (VM)
一个可见性的掩码，用于标识一个页上，每个元组的是否都可见或者都是frozen
TransactionId
仅给会修改数据的事务分配
Hint Bits
在HeapTupleHeader承载插入或者删除是否commit或者abort
TupleTableSlot
元组的一个抽象表示，用于执行层统一处理元祖逻辑，这里的元组抽象了 heap上的元组、索引、纯数组等
PostgreSQL 执行器（executor）中元组在节点之间传递的标准容器

pmmaster启动的时候
postmaster 启动 startup 子进程，由它检测并执行 recovery"
关闭时：

停止接受新连接（smart 模式先等现有 client backend 自然结束）。
PM_STOP_BACKENDS：给所有 backend、autovacuum、bgwriter、walwriter（以及 recovery 相关的 startup/walreceiver）发 SIGTERM；checkpointer、archiver、stats、syslogger 暂不动。
PM_WAIT_BACKENDS：等上述进程退出。
PM_SHUTDOWN：启动/保留 checkpointer，发 SIGUSR2 → checkpointer 写 shutdown checkpoint（这是最终的刷脏 + WAL 同步点），写完退出。
PM_SHUTDOWN_2：唤醒 archiver 做最后一次归档、walsender 发完剩余 WAL，等它们退出。
PM_WAIT_DEAD_END：等剩余辅助子进程退出。
postmaster 进程退出。
后端进程启动：
启动进程---认证---连接数据库并加载catalog-------获取共享内存的PGPROC条目------加入到全局进程数组中-----进入主循环

PostgreSQL 解决"共享内存里指针跨进程"问题的办法是：

共享内存有postmaster申请，随后每个进程通过fork方式创建，直接继承了共享内存的地址，因此可通过指针访问共享内存
每个进程内有一个全局变量指针，指向在共享内存中的head

PostgreSQL 采用懒分配：事务 BEGIN 时不取号，直到首次执行写操作、调用 GetCurrentTransactionId() 等入口，才经 AssignTransactionId → GetNewTransactionId 分配。

分配流程（varsup.c，全程持 XidGenLock）：

检查 wraparound 阈值（xidVacLimit/xidWarnLimit/xidStopLimit），接近危险线则触发 autovacuum、告警或拒绝分配；
先扩展 CLOG/CommitTs/Subtrans 对应页；
推进共享计数器 nextXid（64 位 FullTransactionId = 32 位 epoch + 32 位 xid）；
在释放锁前，将 XID 写入 MyProc->xid（顶层）或 MyProc->subxids（子事务，满 64 个置 overflow 标志）。
子事务分配前先确保父事务有 XID，保证子号大于父号。epoch 用于持久化恢复时确定当前圈数，32 位 xid 回绕后靠它维持绝对先后序。备库不分配，靠 WAL 回放推进 nextXid。