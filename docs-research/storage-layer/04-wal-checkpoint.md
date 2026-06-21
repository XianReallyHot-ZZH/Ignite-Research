# 第 4 阶:机器崩了怎么不丢数据——WAL + Checkpoint

> **对应天花板文档**:`docs-research/03-ignite-storage-layer.md` §6
> **本阶只管一件事**:让"在内存里改的数据"在崩溃后能恢复,做到持久化。

---

## 开场:第 3 阶留下的悬念

前三阶我们搭好了"又快":数据在堆外页里、能按 key 秒查。但有个要命的隐患:**这一切都在内存里。** 机器一断电、进程一崩,内存里那些**还没落盘的页**就全没了——用户刚 `put` 成功的数据,说没就没。

"又快又靠谱"的**靠谱**还没解决。本阶就解决它,靠两件配套利器:**WAL**(先写日志)和 **Checkpoint**(定期刷盘)。

---

## 台阶一:WAL——改页之前,先把"改了啥"记下来

> 术语:**WAL(Write-Ahead Log,预写日志)** = 一个**只追加**(append-only,只往末尾写、不改旧内容)的日志文件。每次改页**之前**,先往这里写一条"我改了什么"。

**痛点** — 直接改内存页很快,但崩了就丢。能不能"既快、崩了又能恢复"?

**类比** — 像**商店的流水账本**:每卖一笔,先在本子上记一行(append,只往后写);即使店里货物(内存页)被火灾烧了,凭账本也能一笔笔重算出该有什么。WAL 就是存储引擎的"流水账"。

**原理** — WAL 的规矩叫**"先写日志,再改页"**(Write-Ahead):每次修改,先往 WAL 追加一条记录,再去改内存页。这样即便页丢了,**重放 WAL** 就能把改动重新做一遍,恢复到崩溃前。

一条普通行写入,会产生**两条**WAL 记录(各管一摊):

| 记录 | 类型 | 重放时能恢复什么 |
|---|---|---|
| `DataPageInsertRecord` | **物理**(页字节增量) | 重建"数据页里那几个字节"长啥样 |
| `DataRecord(DataEntry)` | **逻辑**(缓存操作) | 重做"对某 key 做了 put"这个操作 |

> 物理记录重建**页字节**,逻辑记录重建**缓存语义**——两套各司其职。

**为什么这么设计** — 为什么不只留一种?因为纯物理重放最快(直接还原字节),但它绑定页的物理布局;纯逻辑重放更通用(重做操作),但慢。Ignite **两套都写**,恢复时先用物理快速重建页、再用逻辑补全语义。

📍 **代码锚点**:WAL 实现入口 `FileWriteAheadLogManager.java:172`;一条记录的地址叫 **WALPointer**(`(idx, fileOff, len)`,`WALPointer.java:27`)。对应 03 §6.1。

---

## 台阶二:WAL 的持久程度——四种模式,持久 vs 速度的权衡

**原理** — "写日志"到底写多牢?Ignite 给四档(默认 `LOG_ONLY`):

| 模式 | 含义 | 抗崩溃 |
|---|---|---|
| **`LOG_ONLY`(默认)** | flush 到 OS 的文件缓存(page cache) | 抗进程崩溃,**不抗掉电** |
| `FSYNC` | 每条提交都强行刷到物理磁盘 | **抗掉电** |
| `BACKGROUND` | 后台定时刷 | 最后几次更新可能丢 |
| `NONE` | 关掉 WAL | 只有优雅关机才不丢 |

> 术语补两个:**page cache** 是操作系统帮程序缓存磁盘文件的内存;**fsync** 是"强制把数据真正写到物理磁盘"的系统调用(否则可能还停在 OS 缓存里,掉电会丢)。

**为什么这么设计** — 默认选 `LOG_ONLY` 是为了**速度**(不每条都等磁盘);要抗掉电就显式设 `FSYNC`。这是把"持久性 vs 性能"的开关交给用户。

📍 **代码锚点**:默认值 `DFLT_WAL_MODE = LOG_ONLY`(`DataStorageConfiguration.java:138`)。对应 03 §6.1。

---

## 台阶三:Checkpoint——定期把脏页批量刷盘,收口持久化

> 术语:**Checkpoint(检查点)** = 定期把内存里所有**脏页**(改过但还没落盘的页)批量写到磁盘,并记一个"存档点"标记。

**痛点** — 有了 WAL 崩了能恢复,但**WAL 会越来越长**:崩溃后要从头重放,越久没存档,恢复越慢。而且内存里的脏页迟早要落盘,不能永远只靠 WAL 兜底。

**类比** — 玩游戏**存档**:一直不存档,挂了就要从很久之前重来(重放 WAL 太长);定期存档,挂了只需从上一个存档点重来。Checkpoint 就是"存档"。

**原理** — Checkpoint 干两件事:

```mermaid
flowchart LR
    A["后台 Checkpoint 线程定期触发<br/>(默认 3 分钟一次)"] --> B["把所有脏页批量刷到磁盘 part-N.bin"]
    B --> C["记一个'存档点'标记<br/>(写到 WAL 里的 CheckpointRecord)"]
    C --> D["推进 WAL 截断点<br/>→ 旧 WAL 段可以删了"]
```

做完一次 Checkpoint:**脏页都落盘了 + WAL 里有了标记** → 这之前的旧 WAL 日志就**可以删除了**(恢复时只需重放标记之后的部分)。

**为什么 WAL 和 Checkpoint 必须配套** — 只用 WAL:日志无限长、恢复慢;只用 Checkpoint(不写 WAL 直接刷页):每次改都要刷盘,慢得没法用。**WAL 保证每次改都快(只追加日志),Checkpoint 定期收口(批量刷 + 截断旧日志)**——这是数据库持久化的标准组合拳。

📍 **代码锚点**:Checkpointer 后台线程(`Checkpointer.body`);默认周期 `DFLT_CHECKPOINT_FREQ = 180000`(3 分钟,`DataStorageConfiguration.java:104`);磁盘页文件 `part-N.bin`。对应 03 §6.2。

---

## 台阶四(进阶速览):一致性快照怎么不堵住写?——COW + 双标记

这一台阶是 Checkpointer 的"高级机制",先有个印象即可,细节留给 03。

**矛盾** — Checkpoint 要刷的是"某时刻"的脏页快照,但刷盘很慢,期间用户还在不停改页。怎么既刷到一致的快照,又不卡住写?

**两个关键设计**:

1. **Checkpoint 读写锁**(一种**读写锁**——多个读可并发、写独占):普通"改页"操作持**读锁**;Checkpoint 只在"拍快照"那一瞬间持**写锁**。所以写线程只在极短的快照瞬间被挡一下,之后继续写。

2. **Copy-on-Write(写时复制)快照**:快照期间,如果某个正被刷的页又被用户改了,系统**把这页复制一份**给 Checkpoint 刷(刷的是冻结的副本),用户接着改原件。→ Checkpoint 看到一致快照,写几乎不阻塞。

3. **START / END 双标记**:每次 Checkpoint 写一对 `START` / `END` 标记文件。恢复时:有 `END` = 那次完整成功;只有 `START` 没 `END` = **上次崩在 Checkpoint 中途**,恢复时把这轮补刷完。这让 Checkpoint **对崩溃是原子的**。

**崩溃恢复** = 补做中断的 Checkpoint + 重放 WAL(先物理、后逻辑)。

📍 **代码锚点**:读写锁 `CheckpointReadWriteLock.java:33`;COW 快照 `PageMemoryImpl.postWriteLockPage:1630`;双标记 `CheckpointMarkersStorage`。对应 03 §6.2、§6.3。

---

## 深入(选读):WAL/Checkpoint 文件层与接线

> 台阶一~四讲的是 WAL/Checkpoint 的**概念**。这一节把文件层、记录层的**真实结构,以及它们怎么接起来**讲透,全部对着源码。初读可跳过。

### 1. WAL 段文件 + 追加写 + 记录落盘格式

WAL 切成定长段(默认 64MB),按递增序号命名:

```
${work}/db/wal/                       ← 活跃段(正在写)
  0000000000000017.wal
${work}/db/wal/archive/               ← 封存段(可压缩/删除)
  0000000000000016.wal  ...
```

段内是**连续追加**的记录;写满则 `rollOver`(关闭→移到 archive→开下一段)。每条记录的**落盘格式(V2)**自带类型、自己的指针、CRC:

```
┌────────────┬────────────────────────────┬──────────────┬─────────┐
│ recordType │ WALPointer(idx,fileOff,len)│ record body  │ CRC32   │
└────────────┴────────────────────────────┴──────────────┴─────────┘
```

### 2. 记录类型 + 两条主角

| 类别 | 代表记录 | 重放时干啥 |
|---|---|---|
| **PHYSICAL** | `DataPageInsertRecord`、`PageSnapshot`、各种 `PageDeltaRecord` | 重建页字节 |
| **LOGICAL** | `DataRecord`/`DataEntry`、`TxRecord` | 重做缓存操作/事务 |
| **标记** | `CheckpointRecord`、`SwitchSegmentRecord` | 划定重放点/换段 |

一次普通行写产生两条主角:

```
DataPageInsertRecord(PHYSICAL)              DataRecord(LOGICAL) + DataEntry
┌────────┬────────┬───────────┐            ┌──────────┬──────────────┐
│ grpId  │ pageId │payload(b[])│            │timestamp │List<DataEntry>│
└────────┴────────┴───────────┘            └──────────┴──────────────┘
applyDelta → io.addRow(payload)            DataEntry: cacheId,key,val,op,
(直接改页字节)                               writeVer,nearXidVer,expireTime,
                                             partId,partCnt,flags
                                             (op=CREATE/UPDATE/DELETE)
```

物理记录 = "页字节怎么改"(快、绑定布局);逻辑记录 = "业务做了什么"(通用、可重放语义)。

### 3. WALPointer:每条记录的 1:1 地址

```
┌─────────────┬───────────┬───────────┐
│ idx (long)  │ fileOff   │ len (int) │  POINTER_SIZE=16
│ 段文件序号  │ 段内字节偏移│ 记录长度   │
└─────────────┴───────────┴───────────┘
```

- **1:1 绑定**:`log(record)` 每写一条就生成并返回它的 WALPointer;这指针**同时被写进记录自己的落盘帧**(记录自描述位置)。
- **全序**:按 `(idx, fileOff)` 可比大小,所以 WAL 流是有序的;`next()=(idx,fileOff+len,0)` 指下一条。
- **用途**:当 Checkpoint 的 cpMark、恢复的起点、`truncate` 删段的边界——WAL/Checkpoint/恢复之间流通的"货币"。

### 4. part-N.bin:全量页存储 + 与 WAL 的关系

```
part-N.bin(V2,头部=一整页)
┌──────────────────────────┬──────────────────────────────────────┐
│ 头部(pageSize 字节)      │ page0 │ page1 │ ...                  │
│ 签名+版本+type+pageSize   │ 每页 pageSize 字节,带 CRC(FastCrc)  │
└──────────────────────────┴──────────────────────────────────────┘
页位置: pageOffset(pageId) = pageIndex * pageSize + headerSize(V2=pageSize)
```

```
WAL        = 增量日志(谁改了什么)    追加、短期、被 checkpoint 截断
part-N.bin = 全量页存储(页的最终字节) checkpoint 把脏页刷进来
持久化状态 = part-N.bin(全量) + WAL(cpPtr 之后的增量)
恢复      = 加载 part-N.bin 页 + 重放 cpPtr 之后的 WAL
```

### 5. CheckpointRecord 与 cp 标记文件

`CheckpointRecord`(type=CHECKPOINT_RECORD)字段:`cpId(UUID)`、`end(恒false,已废弃)`、`cacheGrpStates`、`cpMark`。它是**一条 WAL 记录**,不是时间/操作/数据,是个标记。

**最新的 checkpoint 标记不从 WAL 里按类型扫**,而是从 `db/cp/` 的标记文件读:

```
db/cp/
  <cpTs>-<cpId>-START.bin   ← checkpoint 开始时写
  <cpTs>-<cpId>-END.bin     ← checkpoint 结束时写
每个文件里存:(cpId, WALPointer) —— 指向该次 CheckpointRecord 在 WAL 的位置
```

恢复 `readCheckpointStatus`:读最近一对 START/END → 拿 `cpId` + 它的 WALPointer → 再去 WAL 那个位置定位。**用标记文件而不是扫 WAL**:① 小、读得快;② START/END 双文件能判断"上次 checkpoint 是否完整"(有 START 没 END = 崩在中途)。

### 6. cpMark(cpPtr)怎么来:快照 + writeLock 屏障

> 真正用于恢复的"checkpoint mark" = `cpPtr` = CheckpointRecord 自己在 WAL 里的位置(由 `wal.log(cpRec)` 返回,写进标记文件);CheckpointRecord 内部那个 `cpMark` 字段正常 checkpoint 时其实是 null(特例才填)。

看 `CheckpointWorkflow.markCheckpointBegin` 的顺序:

```
① writeLock()                         ← 快照瞬间,改页线程暂停
② beginAllCheckpoints() 冻结脏页快照   ← 把当前所有脏页"照相"
③ cpPtr = wal.log(cpRec)             ← 紧接着写 CheckpointRecord,拿位置
④ writeUnlock()                      ← 放锁
```

①~③ 都在 writeLock 里、②③ 之间无缝 → `cpPtr` 是 WAL 流里一道**干净屏障**:

```
WAL: [data] [data] ... [data] | cpRec@cpPtr | [data] ...
     ╰─ 全在②冻结的快照里,本次刷进 part-N.bin ─╯   ╰─ 快照之后,本次不刷
```

所以 **cpPtr 之前 = 已被快照捕获 = 本次刷盘 → 刷完就 durably 落在 part-N.bin**;之后 = 留着重放。之后 `notchLastCheckpointPtr(cpPtr)` 登记它,`truncate` 就能删 cpPtr 之前的旧段。

### 7. 崩溃重放:物理 + 逻辑(WALPointer 当 seek 游标)

WALPointer 在重放里的角色 = **全序 + seek 游标**(不是逐条过滤):

```
启动 → readCheckpointStatus(读 db/cp/ 标记)→ 得到 cpPtr
        │
        ▼
wal.replay(cpPtr)   ← 把 WAL 读取器 seek 到 cpPtr(段 idx, 段内偏移)
        │
【A. 物理重放 restoreBinaryMemory】从 cpPtr 顺序读
        │   PageDeltaRecord.applyDelta → 把增量打回内存页;PageSnapshot → 整页覆盖
        ▼
   写 MemoryRecoveryRecord(标记:此前的物理 delta 以后不再重放)
【B. 逻辑重放 applyLogicalUpdates】继续读
        │   DataRecord/DataEntry → 经事务管理器重做 put/remove;TxRecord → 重做/回滚
        ▼
   resumeLogging → 启动 checkpointer + 强制一次 "node started" checkpoint
```

**cpPtr 之前的记录不读**(读取器 seek 过去了,对应的页改动已在 part-N.bin);**cpPtr 及之后**逐条重放(物理先、逻辑后)。分区是靠"从哪开始读"切开的。

### 8. Checkpoint 八阶段详图(+ COW + 双标记)

```
① 改页线程平时持 checkpointReadLock(读锁)——所有改页的必备闸
② Checkpoint 线程拿 writeLock——"快照瞬间",改页线程暂停排干
③ beginCheckpoint 冻结脏页快照(复制 dirtyPages→checkpointPages 并清空;safeToUpdate=true)
④ 往 WAL 写 CheckpointRecord(含 cpPtr)            [还持写锁]
⑤ writeUnlock——改页线程恢复(新脏页进下一轮)
   —— 以下无锁,不阻塞写线程 ——
⑥ wal.flush(cpPtr, fsync=true)
⑦ 写 START 标记(临时文件→force→原子改名);拆分排序脏页;多线程刷每个脏页到 part-N.bin;每个 PageStore sync()
⑧ finishCheckpoint 清快照;写 END 标记;notchLastCheckpointPtr(cpPtr) → 旧 WAL 段可删
```

- **COW 快照**(`PageMemoryImpl.postWriteLockPage`):无锁刷页阶段,若某页正被刷又被前台改,拷贝一份冻结副本给 checkpoint 刷,前台改原件 → 一致快照、写几乎不阻塞。
- **START/END 双标记**:有 END=完整成功;只 START 没 END=崩在中途,`finalizeCheckpointOnRecovery` 补刷那轮 → Checkpoint **对崩溃原子**。

### 9. WAL 写入的并发控制:环形缓冲 + 单刷盘者

`log()` 多线程并发调用,既不能让两条记录拿到同一位置(WALPointer 重复),也不能让字节交错写坏段文件。Ignite **没有"一把大锁串到底"**(那会退化成单线程),而是把"**占位置 + 填缓冲**"(并发)和"**写文件**"(串行)解耦:

```
线程A ─┐
线程B ─┼─► buf.offer(rec.size)   [原子地划一段独占切片]
线程C ─┘         │
                 ├─ pos = seg.position() − rec.size          ← 切片起点,唯一
                 ├─ ptr = new WALPointer(idx, pos, size)     ← WALPointer,唯一
                 └─ fillBuffer(seg, rec)                     ← 只写自己那段,不交错
                 ▼
        SegmentedRingByteBuffer(内存环形缓冲)
                 ▼
        刷盘者(单线程,按 pos 顺序刷)▼
        段文件 0000...{idx}.wal   ← 文件写入串行,字节顺序 = 缓冲顺序
```

两个风险,各由不同机制挡:

| 风险 | 由谁挡住 | 怎么挡 |
|---|---|---|
| **WALPointer 重复**(两条记录同位置) | `SegmentedRingByteBuffer.offer(size)` | 原子为每条记录划一段独占切片,起点 `pos` 单调递增、互不重叠 → `fileOff` 必唯一 |
| **段号 idx 错乱**(换段撞车) | `SegmentAware` | 专管"当前是哪段、何时 rollover",换段时 `reserve`/`release` 段号 → `idx` 一致 |
| **字节交错 / 写坏文件** | **单刷盘者** | 只有它往段文件写,按 `pos` 顺序刷 → 文件字节序 = 缓冲序,不交错 |

> 澄清:`FileWriteHandleImpl` 里那把 `ReentrantLock lock`(`:104`,带 `fsync`/`nextSegment` 两个 Condition)是用于 **fsync 等待 / 换段协调**,**不是**串行每条记录的占位——占位的并发安全全交给环形缓冲的原子 `offer`;`written` 位置用 CAS 更新(`:266`)。
>
> 一句话:**靠"环形缓冲原子切片(并发占位)+ SegmentAware(段号)+ 单刷盘者(串行落盘)"把并发和正确性分开管** → WALPointer 不重复、段文件不写坏,同时吞吐高。

📍 **代码锚点**:段文件/追加 `FileWriteAheadLogManager`(`rollOver:1380`);记录落盘格式 `RecordV2Serializer`;WALPointer `WALPointer.java:27`;记录类 `DataPageInsertRecord`/`DataRecord`/`DataEntry`/`CheckpointRecord`;part-N.bin `FilePageStore`(`HEADER_SIZE:76`、`pageOffset:807`);cp 标记 `CheckpointMarkersStorage`(`:554`);cpPtr 派生 `CheckpointWorkflow.markCheckpointBegin:221`;重放 `GridCacheDatabaseSharedManager.restoreBinaryMemory`/`applyLogicalUpdates`;八阶段 `Checkpointer.doCheckpoint`+`CheckpointWorkflow`;**并发控制** `FileWriteHandleImpl.addRecord:222`、`SegmentedRingByteBuffer.offer`、`SegmentAware`、单刷盘者。对应 03 §6.1、§6.2、§6.3。

---

## 你现在应该能回答

1. 为什么每次改页之前要先写 WAL?不写会怎样?
2. WAL 和 Checkpoint 为什么必须配套?只用一个行不行?
3. 默认 WAL 模式是 `LOG_ONLY`,它抗进程崩溃却不抗掉电,为什么?(提示:page cache vs 物理磁盘)

---

## 对应到 03 文档

本阶覆盖 03 的 **§6 全部**:WAL(§6.1)、Checkpoint 八阶段 + COW(§6.2)、崩溃恢复(§6.3)。本阶把八阶段简化成了"做了哪几件事",逐阶段行号明细看 03 §6.2 的时序图。

---

## 留给下一阶的悬念

到这,单机的"又快又靠谱"齐了:内存里快速存取 + 崩了能恢复。

但单机有两个硬伤:**装不下**(数据量超过单机内存/磁盘)和**单点故障**(一台机器挂了,它负责的数据就不可用了)。要扛住真正的生产规模,必须**横向扩展到多台机器**。

这就是第 5 阶的主题:**分区 + 亲和性 + rebalance**。
