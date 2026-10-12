SQL 语句的处理流程

一条 SQL 从字符串到结果，依次经过 **解析 → 语义分析 → 重写 → 规划 → 执行** 五个阶段，每个阶段有自己的一套树表示。

一、SQL 解析（Parse）

词法分析得到一个个有效的 token（词元，包含关键字 select、标识符表名等），接着语法分析将 token 流转换为**原始解析树（raw parse tree）**。

二、语义分析（Analyze）

将对象名称（表、列名等）映射到 oid 上，进行数据类型的一致性检查，细化节点（如，一个 FuncCall 节点，在查询树中会根据解析结果，被转换为 FuncExpr（普通函数）或 Aggref（聚合函数）节点），由解析树得到**查询树（query tree）**，相当于由抽象的语法树，变成了更具体一些的树。

三、查询重写（Rewrite）

语义分析之后、规划之前还有一步：**视图展开**成它的定义查询、**规则（RULE）**的触发、RETURNING 列表的展开等，都在这里完成。产出的仍然是查询树，只是其中不再有视图这类"虚表"——规划器看到的是一棵完全展开、只含真实表的查询树。

四、查询规划（Planner/Optimizer 负责）

第一阶段：预处理 (Preprocessing)

简化查询，将 FROM 子句中的简单子查询"扁平化"（pull up），合并到主查询中，消除嵌套结构；此外还有常量折叠、条件下推、外连接消除等。

第二阶段：扫描/连接规划 (Scan/Join Planning)

为查询中的每个表生成所有可能的扫描路径，再通过连接顺序搜索和连接方法评估，一步步拼出整个查询的最优路径树。

1. 生成基础关系路径

- 根据统计信息估算 where 过滤后的行数与行宽。
- 生成扫描路径：为每个表生成所有可能的访问方式，包含：
  - SeqScan：顺序扫描，通常总会生成作为基准。例外：关系被约束排除判定为空（dummy rel，例如分区全被排除、或对分区表用 ONLY）时，不会生成 SeqScan，而是给一条空的 Append 路径，最终在计划里表现为 Result（One-Time Filter: false）。
  - IndexScan：当 WHERE 条件匹配索引时生成。
  - IndexOnlyScan：索引包含查询所需全部列时生成。但真要"只扫索引"，还需要可见性映射（VM）表明相应页面全部可见，否则仍要回表确认可见性——索引覆盖是必要条件，不是充分条件。
  - BitmapHeapScan：适合 OR 条件或索引元组较多的情况。注意它底下必然挂着 BitmapIndexScan（真正查索引的那一半），OR/AND 条件多时中间还会插入 BitmapOr / BitmapAnd 做位图合并。
- 成本估算与保留：按代价比较，**保留一组互不支配的路径**，而不是只留一条最便宜的——包括总成本最低的、启动成本最低的、以及**每个有用排序（pathkey）下最便宜的那条**。最后一条尤其关键：上层 Merge Join、GroupAggregate、ORDER BY 需要的可能是"已经有序"的路径，一条更便宜但无序的路径替代不了它。

2. 连接顺序搜索

通过动态规划（表数较少时）或者遗传算法对连接顺序进行选择。动态规划按 joinrel 的大小逐层拼接，并不是严格按"表数分层"的教科书式 DP；遗传算法的触发阈值由 geqo_threshold 控制（默认 12 个 FROM 项），另外 from_collapse_limit / join_collapse_limit 决定子查询能被展平多少。

例子：三表查询 customers c、orders o、order_items oi。

Level 1：三个基表。

Level 2：
{customers, orders}：有连接条件 o.customer_id = c.id，生成连接路径。
{orders, order_items}：有连接条件 oi.order_id = o.id，生成连接路径。
{customers, order_items}：无直接连接条件，会生成笛卡尔积路径，但成本极高，通常不会成为最优。

Level 3：
由 {customers, orders} 与 {order_items} 连接。
由 {orders, order_items} 与 {customers} 连接。
由 {customers, order_items} 与 {orders} 连接。
比较所有 Level 3 路径的成本，选出总成本最低者。

3. 生成连接路径

针对每一对可连接的关系，评估不同的连接方法：

- Nested Loop Join：外表每一行，扫描内表。若内表连接列有索引，可用参数化 IndexScan，成本很低；内表也可以套 Materialize 或 Memoize，避免对同一个参数重复计算。
- Merge Join：要求输入按连接键排序，可能需显式 Sort，或利用索引顺序。
- Hash Join：对 inner（通常是代价较低、较小的一侧）建哈希表，用 outer 探测。只支持可哈希的等值连接；两边都较大、嵌套循环代价爆炸时最有用。内存不够（work_mem）时会分批（multi-batch）落盘，成本随之上升。

五、执行器执行

PostgreSQL 使用 **Volcano 模型**：数据从计划树的叶子节点向上流动，每个节点被其父节点"拉取"时，返回一个元组（tuple）。

PostgreSQL 执行器包含丰富的算子（Plan Node），每个算子实现一种特定的关系代数操作，执行器通过组合这些算子来构建执行计划树。

执行器用三个统一入口做分发，它们**不是各算子自己实现的**——各算子实现的是带自己名字的那一份：

| 统一入口（分发） | 各算子实现 | Volcano 对应 |
| --- | --- | --- |
| ExecInitNode | ExecInit<算子名>（如 ExecInitSeqScan） | open |
| ExecProcNode | Exec<算子名>（如 ExecSeqScan） | next |
| ExecEndNode | ExecEnd<算子名>（如 ExecEndSeqScan） | close |

其中 ExecInitNode / ExecEndNode 是 execProcnode.c 里的大 switch；ExecProcNode 则是 executor.h 里的内联函数，通过节点自己的函数指针跳转，并在发现参数变化（chgParam）时先自动调用重扫。

此外还有第四个必不可少的接口 **ExecReScan**（ExecReScan<算子名>）：嵌套循环的内表、参数化路径每换一次参数都要重扫。

算子分类：

| 类别 | 作用 | 算子 |
| --- | --- | --- |
| 扫描算子 (Scan Nodes) | 从表中读取数据 | Seq Scan, Index Scan, Index Only Scan, Bitmap Index Scan, Bitmap Heap Scan, Tid Scan, Tid Range Scan, Sample Scan, Function Scan, Table Function Scan, Values Scan, Subquery Scan, CTE Scan, Named Tuplestore Scan, WorkTable Scan, Foreign Scan, Custom Scan |
| 连接算子 (Join Nodes) | 将两个数据集按条件组合 | Nested Loop, Merge Join, Hash Join |
| 聚合与分组算子 (Aggregation Nodes) | 对数据进行聚合计算或分组 | Agg（唯一的聚合节点类型，按聚合策略显示为 Aggregate 无 GROUP BY / GroupAggregate 排序分组 / HashAggregate 哈希分组 / MixedAggregate grouping sets 混合策略）；Group（另一个独立节点，用于有 GROUP BY 但**没有聚合函数**的查询，要求输入已按分组列排序） |
| 排序与去重算子 (Sorting & Deduplication Nodes) | 对数据进行排序或去重 | Sort, Incremental Sort, Unique |
| 物化与缓存算子 (Materialization Nodes) | 缓存子节点结果，避免重复计算 | Materialize（物化整个子节点结果）, Memoize（按参数缓存子节点结果，本质是"结果缓存"而不是物化） |
| 控制与辅助算子 (Control & Auxiliary Nodes) | 管理结果集、执行限制或并行协调 | Result, Append, Merge Append, Recursive Union, ProjectSet, BitmapAnd, BitmapOr, Limit, LockRows, SetOp, HashSetOp, WindowAgg, Gather, Gather Merge, Hash |
| 数据修改算子 (Modification Nodes) | 执行 INSERT/UPDATE/DELETE/MERGE | ModifyTable |
