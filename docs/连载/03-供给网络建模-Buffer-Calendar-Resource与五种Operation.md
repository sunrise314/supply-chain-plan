# 供给网络建模：Buffer、Calendar、Resource 与五种 Operation

上一章我们把物料、地点、客户、供应商这四类主数据看明白了，但它们还都是"静态档案"。这一章要解决一个更本质的问题：怎么用这些静态档案，搭出一张能被求解器计算的**供给网络**？换句话说，得告诉系统三件事——东西存在哪、产能有多少、东西是怎么造出来的。

frePPLe 用四个核心概念回答这三件事：Buffer（库存点）回答"料在哪、有多少"，Calendar（日历）回答"什么时间可用"，Resource（资源）回答"产能有多大"，Operation（工序）则通过两种连线——物料流 Flow 和产能负载 Load——把它们全部织成一张网。这一章我们一个一个拆。

## Buffer：物料与地点的交叉点

第一章我提过，frePPLe 里没有孤立的"库存"，库存永远是"某个物料在某个地点"的组合，这个组合就是 Buffer（表名 `buffer`）。模型继承我们上一章讲的 `AuditModel`，核心字段长这样：

```python
class Buffer(AuditModel):
    type = models.CharField(...)              # 库存点类型
    batch = models.CharField(default="")      # 批次
    onhand = models.DecimalField(...)         # 当前在手库存
    minimum = models.DecimalField(...)        # 最低库存（安全库存下限）
    minimum_calendar = models.ForeignKey(Calendar, ...)  # 随时间变化的最低库存
    min_interval = models.DurationField(...)  # 最小补货间隔
    maximum = models.DecimalField(...)        # 最高库存
    maximum_calendar = models.ForeignKey(Calendar, ...)  # 随时间变化的最高库存
```

这里最能体现专业度的是它**不把库存上下限当成一个死数字**。`minimum` 是默认的安全库存下限，但 `minimum_calendar` 允许这个下限随时间浮动——比如旺季安全库存拉高、淡季压低；`maximum` 和 `maximum_calendar` 则约束库存上限，防止计划引擎把仓库塞满。`min_interval` 这个 DurationField 表达"两次补货之间至少隔多久"，是个很现实的约束（不可能今天补一点明天补一点）。至于 `onhand` 现有库存、`batch` 批次属性，都和前面一样用高精度 Decimal。

Buffer 在求解器里的地位，我用一句话剧透：它是 ask-reply 递归（第 7 章主角）里"问物料"的应答方。一道工序要吃料，实际是向对应的 Buffer 发问"这个时间点你有这么多料吗"，库存够就立刻答应，不够就继续问"那谁能补、什么时候能补上"，递归由此展开。

## Calendar：把"一切约束都随时间变化"做实

如果只能让我挑一个最能体现"专业计划软件"和"业余排程脚本"差距的设计，我会挑 Calendar。在 frePPLe 里，几乎所有带数值的约束都能挂一个日历，让这个数值成为时间的函数。

`Calendar` 本身很简单：字符串主键 `name` 加一个 `defaultvalue`（默认值）。真正干活的是它下面的一批 `CalendarBucket`：每个桶定义 `starttime`/`endtime`（一天内的时段）、用七个布尔位表示的 `weekdays`（周一到周日哪些天命中）、起止日期 `startdate`/`enddate`、命中时的 `value`，以及一个 `priority`。我读它取值的那段实现时注意到一个细节：多个桶时间段重叠时，它不是报错，而是按 `priority` 决定谁生效，遍历过程中持续记录 `curPriority`，让更具体（优先级更高）的规则覆盖泛化规则。源码里还老老实实地处理了一个边界——`endtime` 是不包含的右端点，它会在比较前把结束时间加一秒，避免"正好卡在结束时刻"的点被漏掉。

这套机制的威力在于复用。你在第 2 章见过 `Location.available` 指向日历；这一章 Resource 的 `maximum_calendar`（两班倒时白班 2 台机、夜班 1 台机）和 `efficiency_calendar`（设备老化期效率打折）也都是日历；连 Buffer 的安全库存上下限都是日历。**一张"三班倒+周末停机+法定节假日+旺季加班"的生产日历建好之后，可以同时驱动地点可用性、产能大小和效率，而不必在每个实体里重复维护排班。** 这是非常漂亮的抽象。

## Resource：四种资源模型

Resource（表名 `resource`）同样继承 `AuditModel` 和 `HierarchyModel`——资源也能分层，比如"一车间 → 一号产线 → 1 号机床"。它的类型枚举我贴在下面：

```python
types = (
    ("default", _("default")),
    ("buckets", _("buckets")),
    ("buckets_day", ...), ("buckets_week", ...), ("buckets_month", ...),
    ("infinite", _("infinite")),
)
```

- **default（默认型）**：把产能看成连续可用的"容量"。`maximum` 表示同时能承载的负载量，默认 1.00，而且允许小数——0.5 可以表示一个人只能投入一半工时。多道工序只要总负载不超过 `maximum` 就能并行占用，非常适合建模人力、并行机台。
- **buckets / buckets_day / week / month（时段型）**：把时间切成日、周、月这样的桶来记账，产能按桶消耗。它牺牲了桶内的精确时序，换来了按天/周/月做粗能力计划和报表聚合的便利，适合中期产能规划。源码注释里坦白这些桶尺寸是"hardcoded"的预定义项，是个已知的不灵活之处。
- **infinite（无限型）**：完全不做产能约束。当某台设备事实上绝不会成为瓶颈时，把它标成无限资源，引擎就不在它身上花算力检查过载。

除了类型，Resource 上还有几个我很喜欢的字段：一个布尔 `constrained` 让你即便对默认型资源也能临时关掉产能约束（相当于保留过载、只用来看计划）；`efficiency` 百分比加 `efficiency_calendar`，效率低于 100% 时工序会自动占用更久（帮助文本写得很直白）；`cost` 用于在多个替代资源间算成本；`setupmatrix` 外键加 `setup` 字段则是换型模型的入口。

劳动力这种"一个人能干多种活"的情况，它用 `Skill` 和 `ResourceSkill` 两张表表达：Skill 定义技能，ResourceSkill 把资源和技能挂钩并带 `priority`。于是"这道工序需要一个会焊接的工人"就不绑定具体某人，而是挂一个技能，引擎在所有具备该技能、且当时有空的人力资源里按优先级挑。

## Operation：五种工序，一张表讲清五种生产形态

终于到了最核心的 Operation（表名 `operation`）。frePPLe 用一个 `type` 字段加自引用的 `owner` 父子结构，用五种类型覆盖了现实里几乎所有工序形态。类型枚举就五行：

```python
types = (
    ("fixed_time", _("fixed_time")),   # 固定耗时
    ("time_per", _("time_per")),       # 按件计时
    ("routing", _("routing")),         # 工艺路线（多步骤）
    ("alternate", _("alternate")),     # 多选一
    ("split", _("split")),             # 拆分并行
)
```

我用一个机械加工厂的例子把它们串起来：

- **fixed_time（固定时间）**：`duration` 固定，和加工数量无关。典型如热处理、灭菌——不管一炉放 10 件还是 100 件，都要烤 8 小时。
- **time_per（按件计时）**：总耗时 = `duration_per`（每件耗时）× 数量，另有一个固定的准备时间。数控车床加工就是这种，每件 3 分钟，100 件就是 300 分钟。
- **routing（工艺路线）**：一个父工序，下面挂一串按 `priority` 排序的子工序（子工序只能是 fixed_time 或 time_per 这种"叶子"），表示"先粗车、再精车、再检验"的多步骤。子工序之间还能用 `OperationDependency` 表达依赖，从而允许步骤交叠——不必等上一步全部做完才开始下一步。
- **alternate（替代）**：父工序下挂多个能力相同的子工序，引擎按 `priority` 和 `search`（选择偏好，如优先成本最低或优先级最高）**只选其中一个**执行。两台都能做同一道活的机床就是 alternate。
- **split（拆分）**：和 alternate 长得像，但语义相反——它把数量**同时拆分到多个子工序并行生产**。一批 1000 件，两台并行机床各分 500 件同时开工，靠 `split` 表达。

模型在 `save()` 里还做了很实在的数据校验，防止你建出无意义的结构：routing 的 owner 只能拥有 time_per/fixed_time 叶子工序，否则抛 `"Only time per or fixed time operations can have an owner of type routing"`；alternate 和 split 下不能再嵌套 alternate/split（`"alternate and split operations cannot have an owner of type alternate or split"`）。这种把领域规则硬编码进保存逻辑的做法，等于在数据入口就挡住了大量"能存进去但永远算不对"的脏模型。

Operation 自身还带了一批计划参数：`sizeminimum`/`sizemultiple`/`sizemaximum` 约束生产批量（最小批量、批量倍数、最大批量），`fence` 是冻结 fence（冻结期内不允许引擎改动已确认计划），`posttime` 是工序结束后的固定后处理时间，`batchwindow` 则告诉引擎可以在需求日前后多大的时间窗里寻找并批（batching）的机会——帮助文本说它会扫描这个窗口内合并批次。

## Flow 与 Load：把工序织进网络的两种连线

光有孤立的工序没有意义，工序要通过两种连线接入网络，这两张表是我认为整个模型里最精妙的设计。

**第一种是 OperationMaterial（物料流，表名 `operationmaterial`，对应引擎里的 Flow）。** 它把一道工序连到一个物料（外加可选的地点），字段 `quantity` 是**每件**的用量、`quantity_fixed` 是与批量无关的固定用量（比如每开机一次固定损耗的耗材）。流量的正负号有严格约定，我专门去 C++ 头文件里核实过：

```cpp
// include/frepple/model.h
/* Returns true if this flow consumes material from the buffer. */
bool isConsumer() const { return quantity < 0 || quantity_fixed < 0; }
/* Returns true if this flow produces material into the buffer. */
bool isProducer() const { return quantity > 0 || quantity_fixed > 0 || ... }
```

也就是说：**正流量表示工序向 Buffer 产出，负流量表示工序从 Buffer 消耗。** 一道装配工序，对着成品 Buffer 挂一条 +1 的流，对着每个零件 Buffer 挂负数的流（每件用 2 个螺丝就是 -2）。Flow 上还有个 `type` 区分在工序的 `start`（开工）还是 `end`（完工）时点扣料/入库，以及 `transfer_batch` 批量转运；`offset` 允许把物料需求在时间上前移或后移。

特别要夸的是 `effective_start`/`effective_end` 这对生效日期（默认 1971-01-01 到 2030-12-31）。它让同一条工序-物料关系可以随时间切换——产品改版，旧 BOM 到 6 月底失效、新 BOM 7 月起生效，全都在一张表里用时间区间表达，这正是工程变更（ECN）在计划系统里该有的样子。而 `name` + `priority` + `search` 三兄弟又一次出现：同名的一组 Flow 构成"替代物料组"，缺首选料时引擎按优先级和偏好自动选替代料。

**第二种是 OperationResource（产能负载，表名 `operationresource`，对应 Load）。** 它把工序连到 Resource，`quantity` 表示这道工序执行时要占用几个单位的资源（一台机床就填 1，需要 3 个工人同时作业就填 3），同样有 `quantity_fixed`、替代组（name/priority/search），以及一个关键字段 `setup`——它是换型矩阵里的"换型后状态"标识。

## SetupMatrix：把换型时间算进计划

最后补上资源侧的换型模型。`SetupMatrix`（表名 `setupmatrix`）只是个命名容器，真正的规则在 `SetupRule`：每条规则有 `fromsetup`、`tosetup`、`priority` 和换型耗时。`fromsetup` 还支持通配符，所以你可以写"从任何浅色油漆换到深色只需 10 分钟、从深色换到浅色必须清洗 60 分钟"这样的规则。Resource 上的 `setup` 字段记录它当前处于什么换型状态，引擎排两道相邻工序时，就用"前一道的 tosetup → 后一道的 tosetup"去矩阵里查出真实的换型耗时并插入排程。忽略换型，是很多业余排程程序算出的计划在车间根本执行不了的直接原因。

## 一张图理解 BOM 和工艺路线为什么被统一了

讲到这里，你可能已经发现 frePPLe 一个大胆而优雅的设计：**它没有像传统 ERP 那样把"BOM（物料清单）"和"工艺路线（Routing）"拆成两套数据。** 在它的世界里，一个 Operation 同时挂着物料流（Flow，回答用什么料、出什么成品）和产能负载（Load，回答用什么设备、占多久），BOM 和工艺路线被统一进了同一个工序节点。

于是"BOM 多层展开"这件事根本不需要单独的 MRP 展开程序：装配工序消耗零件 Buffer（负流），而那个零件 Buffer 的补货方又是另一道制造工序，那道工序又去消耗更底层的原材料 Buffer……顺着负流一路追下去，经过的工序层数，就是 BOM 的层数。第 7 章你会看到，求解器的递归深度恰好等于这个 BOM 深度——这不是巧合，而是数据模型决定的必然结果。

## 小结

这一章我们用 Buffer（库存点）、Calendar（时间函数）、Resource（四种产能）和 Operation（五种工序）搭出了整张供给网络，再用 Flow（正负物料流，含生效区间和替代组）与 Load（产能占用，含换型）两种连线把它们缝合。安全库存是日历、产能是日历、效率是日历；BOM 和工艺路线在工序节点上合二为一。模型搭到这一步，系统已经"知道"一个产品该怎么造了——但它还不知道"未来该造多少"。

下一章，我们走到决策链最上游，看需求从哪来：销售订单怎么建模，frePPLe 又怎么用统计预测方法从销售历史里算出未来需求、并用真实订单去抵消毛预测。
