height weight lock
重量级锁用于数据库内逻辑实体的并发控制，和闩锁的区别是：闩锁保护内存里的数据结构、生命周期极短、不检测死锁；重量级锁锁的是用户可见的逻辑对象，从库、表、页、行，一直到事务和用户自定义的键

表锁状态：共有8个，主要思想是读与写不冲突，ddl要独占，在线维护尽量不阻塞读写
代表性状态：
1.access share,对应操作为普通的select，仅被ACCESS EXCLUSIVE阻塞
2.ROW EXCLUSIVE，对应操作为insert、update、delete，不挡普通读，挡create index跟ddl
3.SHARE UPDATE EXCLUSIVE（共享更新排他锁），对应操作为vacuum、analyze、create index concurrently，不阻塞读写，阻塞ddl跟其余维护操作
4.SHARE，对应操作为CREATE INDEX（非 CONCURRENTLY），会阻塞写，不阻塞读
5.ACCESS EXCLUSIVE，对应ALTER/DROP/TRUNCATE/VACUUM FULL，会阻塞所有操作
行锁状态：共有四个，会改唯一键就使用for update、不改唯一键就使用for no key update，这个是为了unique列的一致性，外键检查会拿父表 FOR KEY SHARE，所以更新非键列不会阻塞子表插入。
1.for update：最强。改唯一键、DELETE、SELECT FOR UPDATE 会用。
2.for no key update:次强。UPDATE 不改唯一键时自动用
3.FOR SHARE:共享。SELECT FOR SHARE 用，挡写，允许多个共享。
4.for key share:最弱。外键检查自动用，只挡 FOR UPDATE。

锁的管理:由三层结构来管理
1.LOCK 对象，lock对象直接承载了锁的状态（可能不止一个），每个状态有多少个进程持有或者在等待，以此可以快速判断是否有冲突，通过链表可以间接知道每个进程对于这个lock的状态，使用在共享内存的hash表进行承载，key为LOCKTAG（唯一地定义了一个“可锁定对象”），value为lock对象
typedef struct LOCK
{
    LOCKTAG     tag;                    /* 锁对象的唯一标识 */

    LOCKMASK    grantMask;              /* 已授予的锁模式位掩码 */
    LOCKMASK    waitMask;               /* 正在等待的锁模式位掩码 */

    dlist_head  procLocks;              /* 该锁上所有 PROCLOCK 的链表 */
    ProcQueue   waitProcs;              /* 该锁的等待队列 */

    int         requested[MAX_LOCKMODES]; /* 每种模式被请求的次数 */
    int         nRequested;             /* requested[] 之和 */

    int         granted[MAX_LOCKMODES]; /* 每种模式被授予的次数 */
    int         nGranted;               /* granted[] 之和 */
} LOCK;
2.PROCLOCK 对象，这是 LOCK 与具体后端进程（PGPROC）之间的关联结构，它记录了某个后端进程在特定 LOCK 对象上持有或等待的锁信息。PROCLOCK 同样存储在共享内存哈希表中。
typedef struct PROCLOCKTAG
{
    LOCK       *lock;   // 指向共享内存中的 LOCK 对象
    PGPROC     *proc;   // 指向后端进程的 PGPROC
} PROCLOCKTAG;
hashmap键是 PROCLOCKTAG，value是 PROCLOCK 结构体，用于快速找到某个进程的某个锁是什么状态，需要保证状态的唯一性，同时实现高效的插入与删除的空间管理
LockMethodProcLockHash (共享内存哈希表)
  Key = PROCLOCKTAG { LOCK*, PGPROC* }
  Value = PROCLOCK
    │
    ├─ tag          → { myLock, myProc }       唯一标识
    ├─ groupLeader  → PGPROC*                  锁组组长
    ├─ holdMask     → 已授予的锁模式位掩码
    ├─ releaseMask  → 正在释放的锁模式位掩码
    ├─ lockLink     → 挂到 lock->procLocks     按锁维度遍历
    └─ procLink     → 挂到 proc->procLocks     按进程维度遍历

3.LOCALLOCK 对象，这个是每个进程内部私有的变量，当进程需要拿锁，优先在本地查找LOCALLOCK，没找到时才需访问共享内存，LOCALLOCK包含了LOCK对象跟PROCLOCK对象的指针，以及该锁被持有的次数（nLocks）等信息，用于支持锁的重入。使用hashmap承载，key为 LOCALLOCKTAG 与value为 LOCALLOCK
typedef struct LOCALLOCKTAG
{
    LOCKTAG     lock;   /* 共享内存中锁对象的标识 */
    LOCKMODE    mode;   /* 请求的锁模式（如 AccessShareLock） */
} LOCALLOCKTAG;
同一个锁对象上的不同锁模式，在本地缓存中会生成不同的 LOCALLOCK 条目。

举例（不考虑 Fast Path Locking优化）：
链路一：首次获取锁（以 SELECT * FROM accounts 请求 ACCESS SHARE 为例）
1.本地查找LOCALLOCK，不存在，初始化一个
2.共享 LOCK 查找/创建，不存在，创建一个LOCK对象
3.共享 PROCLOCK 查找/创建，创建PROCLOCK，并挂载到lock->procLocks、MyProc->procLocks两个链表上
4.冲突判断与授予，无冲突，回填本地 LOCALLOCK
链路二：事务提交释放所有锁（LockReleaseAll）
1.遍历本地 LOCALLOCK 哈希
2.通过 LOCALLOCK 定位共享结构
3.清除 PROCLOCK 的持有模式，仅清除LOCALLOCK对应的一个锁模式
4.清理 LOCK 对象，若 LOCK 对象上已无任何 PROCLOCK（lock->procLocks 为空）且无等待者则删除lock条目
5.清理本地 LOCALLOCK，循环遍历以上流程

冲突处理：
首先检查 waitMask：如果 conflictTab[lockmode] & lock->waitMask 非零，说明等待队列中已有与本请求冲突的等待者，必须排队，不能插队。这一步独立于 grantMask，用于保证公平性。
如果 waitMask 无冲突，再检查 grantMask：如果 conflictTab[lockmode] & lock->grantMask 为零，说明已授予的锁中无冲突，直接授予。
如果 grantMask 有冲突，通过 granted[] 和自己的 holdMask 判断冲突是否仅来自自己：对于每个冲突模式 i，如果 granted[i] == 1 且自己持有模式 i，说明只有自己持有，从冲突中排除；如果 granted[i] > 1，说明除了自己还有别人持有，是真冲突；如果自己不持有模式 i 但 granted[i] > 0，也一定是真冲突。
如果排除自己后无真冲突，直接授予新模式（自己原有的模式保留，语义上相当于锁升级）。
如果存在真冲突，将自己加入 lock->waitProcs，设置 MyProc->waitLockMode = lockmode，并更新 lock->waitMask |= LOCKBIT_ON(lockmode)，然后休眠等待唤醒。

Fast Path Locking：
对于弱表锁（access share、ROW SHARE、ROW EXCLUSIVE）、同时仅锁单个库（区别于全局角色/用户认证信息等）的锁，进行了单独的优化，倾向于将其记到PGPROC名下而不放到锁表管理，但还是在共享内存（PGPROC结构体内），因为其余进程需要检测（上强表锁的扫描）
解决的问题：在 PostgreSQL 里，每执行一条 SQL，都需要对涉及的关系（表、索引、视图等）加表级锁。这些锁大多是 ACCESS SHARE 这种非常弱的锁，读读之间根本不冲突。但问题在于：每一次加锁/放锁，都要去访问全局共享锁表（shared lock table），并申请 LWLock:lock_manager 这个轻量级锁来保护它
锁空间被分成了1024个锁分区（LOCKTAG的hash对1024取模），锁分区会记录这个空间强表锁的数目，如果为0，就说明没有强表锁，这种情况下，弱表锁就可以不放入锁表
如果有进程想要上强表锁，先给分区对应强锁数++（避免新的弱锁），接着会检测其余的所有进程是否有上对应分区的弱锁（扫描每一个后端进程的PGPROC），如果有就迁移到锁表进行管理，等迁移完后，再真正尝试在锁表里上强锁

行锁：
行锁存储在行头，不存到共享内存表，以此避免锁表的膨胀，涉及的字段为
xmax：这个字段可以表示持有锁的xid、MultiXactId（指向多个持有锁信息的地址）、删除该行的xid
t_infomask / t_infomask2（行头的标记位），可以用于标识是否删除、锁的具体模式等
冲突时，休眠与唤醒会复用height weight lock的逻辑，在共享内存表锁里面新建一个行锁lock对象
每后端同时最多只有一条行级等待记录，且只在存在活跃冲突时才建 → 所以行锁再多也不会撑爆锁表，这就是 PostgreSQL 没有锁升级（行锁升级为表锁）的原因
一个例外：会话已持有该行某种锁、想升级到更强时，不需要加入到行锁lock对象的等待队列中，可以升级

休眠与唤醒实现：
借助了操作系统的能力（对于 Linux 是 epoll），epoll 实现了跟踪 fd，当 fd 状态发生变化时，就唤醒进程的能力，一个epoll实例可以高效的跟踪多个fd的状态。
当某进程没有拿到锁，就会主动进行休眠，释放 CPU 资源。
对于进程 p2，当其释放锁资源后，会检查 lock 的等待队列。从队首开始，对每个等待者做判断：如果等待者想要的模式与当前锁已授予的模式（包括前面已经唤醒的进程）没有冲突，就会在共享内存中授予它锁，并唤醒这个进程（p2 使被唤醒进程对应 latch 的 fd 变成可读）。如果有冲突，就会跳过这个进程，并接着往后遍历
对于被唤醒的进程，会检查锁的状态（MyProc->waitStatus）。如果是真的拿到了锁（GRANTED），就继续执行后续操作；如果发现自己被选为死锁牺牲者（DIED），就中止事务并释放锁；否则会重新进入休眠（避免由于超时、postmaster 死亡检测、或其他信号导致的虚假唤醒）。

死锁检测与恢复：
当等待的进程等待时间超过了 deadlock_timeout（默认1s），这个进程自己会被操作系统唤醒，随后它会构建等待图来检查是否陷入死锁（遍历）。
图中的点为进程（事务），边为等待关系。如果是 P1 等待 P2 已经持有的锁，那么就是hard edge（遍历procLocks）；如果 P1 与 P2 都在等待同一个锁，且 P1 排在 P2 后面，那么就是soft edge（遍历 waitProcs）。会递归进行遍历，p1被唤醒后，先得到p1的waitlock，以此为入口开始遍历，通过visitedProcs[]来判断是否节点曾经遍历过，从而判断是否有环。对于当前进程等待的锁，先遍历 procLocks 找到硬边阻塞者，再遍历 waitProcs 找到软边阻塞者，对每个阻塞者递归执行同样的流程。
如果死锁环全部由实边构成，被唤醒的这个进程就会中止自己的事务（即“自杀”）。
如果环中存在虚边，系统会首先尝试对相关的等待队列进行重排（拓扑排序），以消除虚边、避免中止任何事务。只有当重排无法解决时，才会触发中止。

light weight lock
轻量级锁，仅有共享跟独占两种模式，用于保护共享内存的内容，尝试获取锁通过原子操作CAS实现，等待通过等待队列实现，通过信号量实现唤醒，信号量是进程级别，每个进程一个
上共享锁，state++,解共享锁--，当为0时，锁的现在持有方从等待队首往后遍历，唤醒多个读进程或者一个写进程，会将目前能唤醒的读进程全部唤醒
