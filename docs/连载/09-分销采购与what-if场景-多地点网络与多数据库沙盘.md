# 分销、采购与 what-if 场景：多地点网络与多数据库沙盘

前面八章我们的视野基本都在"一个工厂内部"。但真实的供应链是一张网：成品从工厂下线，先到中央仓，再调拨到几个区域仓，最后发给各地客户；原材料则反过来，由不同供应商在不同提前期下喂给工厂。这一章我们把视野拉开，看 frePPLe 怎么解决三个跨地点的问题：**多仓之间怎么自动补货（分销计划）、采购怎么建模、以及怎么在不碰正式计划的前提下做 what-if 沙盘推演。**

## 一个 Buffer 的三种补货来源

先建立一个统一视角。第 5、7 章我们反复看到 Buffer 缺货时会调用 `getProducingOperation()` 去找"补货方"。这个"补货方"在多地点网络里其实有三种身份：

1. **自己造**：补货方是一道制造工序（Operation），产出制造工单；
2. **向外买**：补货方由 `ItemSupplier` 描述——某个物料在某个地点可以向某个供应商采购；
3. **内部调**：补货方由 `ItemDistribution` 描述——某个物料在某个仓缺货，可以从另一个仓调货过来。

frePPLe 最优雅的地方在于：求解器并不需要为这三种来源写三套递归。无论补货是造、是买还是调，对缺货 Buffer 而言都是一次"`(q_qty, q_date)` 问补货方、拿回 `(a_qty, a_date)` 应答"。区别只在应答时间和成本由什么决定——制造看产能和 BOM，采购看供应商提前期，调拨看运输提前期和源仓库存。第 7 章那棵 ask-reply 请求树，就这么自然地从工厂内部延伸到了整个供应网络。

## ItemDistribution：把仓库之间的补货关系画成"通道"

分销计划落在 `ItemDistribution` 这张表（表名 `itemsupplier` 的分销孪生表，表名 `itemdistribution`）。我读了它的字段，几乎是照着"一条仓间补货通道"该有的样子设计的：

```python
class ItemDistribution(AuditModel):
    item = models.ForeignKey(Item, ...)
    location = models.ForeignKey(Location, ...,
        help_text=_("Destination location to be replenished"))   # 目的仓（被补货方）
    origin  = ... # help_text: Source location shipping the item （源仓：发货方）
    leadtime = models.DurationField(...)        # 运输提前期
    sizeminimum / sizemultiple / sizemaximum    # 调拨批量约束（整车/整托）
    batchwindow = models.DurationField(...)     # 并批时间窗
    cost = models.DecimalField(..., help_text=_("Shipping cost per unit"))  # 单位运费
    priority = models.IntegerField(...)         # 多供货仓时的优先级
    resource_qty = models.DecimalField(...)     # 每条调拨占用多少运输资源
    fence = models.DurationField(...)           # 冻结 fence
```

唯一键是 `(item, location, origin, effective_start)` 四元组。这里有两个设计值得专门点出。

**第一，一个目的仓可以有多个 origin，配合 priority 和 cost 形成替代供货。** 华东仓缺货时，模型允许它从华北仓调（优先级 1、运费低但要 3 天）或从华南仓调（优先级 2、运费高但 1 天到）。求解时这和第 3 章讲的替代资源、替代物料完全同构——引擎先问首选通道（华北），源仓没货或来不及再问华南，剩余缺口标在首选上。于是"多仓互为备份"不需要特殊概念，复用的还是同一套 name/priority/alternate 机制。

**第二，通道有生效日期（effective_start 进了唯一键）。** 这让你能表达"10 月起新开一条成都-西安的干线""旧仓库 12 月底关闭后该通道失效"，仓网调整随时间平滑切换，和 BOM 的工程变更生效区间是一个思路。`sizeminimum/multiple/maximum` 在分销语境下对应整车/整托发运规则，`resource_qty` 还能把运输能力本身建模成有限资源（一天只能发几车），`fence` 则冻结近期已安排的调拨不被引擎改。

## DRP 的递归：缺口沿网络逐级上卷

有了 ItemDistribution，分销需求计划（DRP）就是 ask-reply 在地点维度的自然延伸，不需要独立算法。设想三级网络：城市仓→区域仓→中央工厂仓。城市仓库存跌破下限，它向区域仓通道发问；区域仓也没货，区域仓的 Buffer 转而向中央仓通道发问；中央仓再没货，就向工厂的制造工序发问，工厂又触发对原材料和供应商的递归。

你看，**第 7 章那句"递归深度等于 BOM 层数"在这里要扩展成"递归深度等于 BOM 层数乘以网络层数"**——请求既要沿产品结构向下钻入原材料，也要沿地点结构向上钻入上游仓库。每次跨通道发问，应答日期都要加上该段 `leadtime`（运输提前期），成本累加 `cost`（运费）。这也是为什么跨地区的多级网络必须把安全库存放对位置：如果每个层级都按终端需求波动独立备足安全库存，会造成层层放大的牛鞭效应和重复占压；实践中应把缓冲集中放在少数几个"解耦点"（通常是中央仓和工厂），下游仓尽量靠快速补货而非堆库存。frePPLe 的模型能把这种网络算清楚，但缓冲策略仍要靠人按第 5 章的方法设计。

## ItemSupplier：采购侧的提前期与安全冗余

外购料由 `ItemSupplier`（表名 `itemsupplier`）描述，结构和分销通道几乎对称：item + location + supplier 三元关系，`leadtime`（采购提前期，DurationField）是核心，外加和分销同款的批量三件套、成本、优先级、resource_qty、生效日期。我注意到一个很贴现实的字段：

```python
leadtime = models.DurationField(..., help_text=_("Purchasing lead time"))
extra_safety_leadtime = models.DurationField(_("soft safety lead time"), ...)
```

除了硬性的采购提前期，还有一个 **extra_safety_leadtime（软性安全提前期）**。这俩的区别很微妙但重要：硬提前期是供应商白纸黑字承诺的下单到到货时间，不可违反；软安全提前期则是计划员根据"这家供应商老爱延迟一两天"的历史经验额外加的缓冲。把它们分开建模，你既能区分"理论上需要多久"和"我信他几分"，也能在供应商改善后只调软缓冲而不动基础数据。同一个物料同样支持多个供应商（priority+cost 互为替代），首选供应商交期太长就问备选——采购侧的寻源决策和车间、仓网共用一套替代逻辑。

## 五类单据：其实是同一张表的五种身份

递归求解最终吐出的结果是什么？不是五张互不相关的表，而是**统一的一张 `operationplan` 表，用 `type` 字段区分单据身份**。我在模型里读到的类型枚举是：

```python
types = (
    ("STCK", _("inventory")),            # 库存（期初/盘点）
    ("MO",   _("manufacturing order")),   # 制造工单（自己造）
    ("WO",   _("work order")),            # 车间作业单（工艺路线里的具体工序）
    ("PO",   _("purchase order")),        # 采购单（向外买）
    ("DO",   _("distribution order")),    # 调拨单（内部调）
    ("DLVR", _("delivery order")),        # 发货单（满足客户需求）
)
orderstatus = (("proposed", ...), ("approved", ...), ("confirmed", ...),
               ("completed", ...), ("closed", ...))
```

这是个很有启发性的统一设计：造（MO/WO）、买（PO）、调（DO）、交（DLVR），在求解器眼里都是"在某段时间、某个地点、让某个 Buffer 的库存发生一笔增减"的计划记录，于是它们共用一张表、一套时间字段、一套状态机，彼此之间靠 Buffer 的库存时间轴自动咬合——PO 到货入库、DO 在途移动、MO 领料入库，本质都是 FlowPlan 事件。`reference` 用字符串业务单号做主键，和前面所有主数据一脉相承。

状态机是一条单向推进的流水线：**proposed（系统提议）→ approved（计划员审核）→ confirmed（已下达/确认）→ completed（已完成）→ closed（关闭）**。模型的 `save()` 里有大量状态联动逻辑，体现了真实的管控规则：父计划（routing/alternate/split 的父工序）在子计划已确认时不允许还停在 proposed；一个 routing 步骤完工时，会保证前序步骤也都 closed/completed；`quantity_completed` 字段支持部分完工汇报；而 confirmed 之后的单据会被冻结 fence 保护起来，引擎重新跑计划时不能擅自改动已经下达给车间/供应商的承诺。**系统只管提建议（proposed），人审核后才升级成承诺（approved/confirmed），承诺不可被算法悄悄推翻**——这条边界是计划系统能真正落地、被业务信任的关键。

## what-if 场景：多数据库沙箱

讲完网络，最后讲一个把 frePPLe 和大多数玩具排程工具区别开的能力：场景模拟。供应链决策常常是"如果……会怎样"：如果新开一条产线、如果给这个供应商翻倍下单量、如果把安全库存整体下调 10%，晚交订单和库存各会怎么变？你绝不可能拿正式计划去做这种实验。

第 2 章我们埋过伏笔，现在展开：frePPLe 的场景，本质是**同一套 schema 的多个独立 PostgreSQL 数据库**。`Scenario` 模型只有三个状态：

```python
scenarioStatus = (("free", _("free")), ("in use", _("in use")), ("busy", _("busy")))
```

默认库就是名为 Production 的正式场景。你想做沙盘时，系统会**完整复制一个数据库**（结构加数据）成新场景——架构文档把"scenario copying"明确列为由独立 worker 进程执行的长任务之一，和跑计划、数据库备份并列，因为拷贝一个几万张工单的库绝不是 HTTP 请求里能同步完成的事。复制完成后场景置为 "in use"，你就可以在这个完全隔离的库里随意改参数、改主数据、反复跑计划：所有 ORM 操作都靠第 2 章那个 `MultiDBManager` 的 `objects.using(场景名)` 路由到对应数据库，和 Production 物理隔离、零副作用。跑完对比几个关键 KPI（晚交订单数、平均库存、资源过载、问题数量），确认方案更好，再把改动带回正式库。

还记得 `Parameter` 的 `save()` 吗？改全局参数时它会遍历所有 "In use" 的场景库，用 `insert ... on conflict do update` 把基础配置 UPSERT 过去——保证你在多个沙箱里做对比时，底层参数是一致的，差异只来自你真正想验证的那个变量。这种"用一整个数据库当沙箱"的做法乍看笨重，实则极其可靠：隔离是物理级的，不依赖任何复杂的内存事务回滚，而且每个场景还能让不同人各占一个并行推演，互不干扰。

## 一个完整的网络计划长什么样

把这一章和前面串起来，一次完整计划跑完，operationplan 表里会同时出现这样一批协调一致的单据：对每张客户需求，一张 DLVR 发货单锁定交付；往前是若干 MO 制造工单和它们 routing 下的 WO 车间作业单；工厂原材料缺口产生 PO 采购单（带供应商和硬/软提前期）；区域仓、城市仓的库存缺口产生 DO 调拨单（沿 ItemDistribution 通道、占运输资源、满足整车批量）。所有单据的时间由 ask-reply 在 BOM 和仓网两个维度上的递归自动对齐，安全库存补货作为常驻需求先行满足，已确认单据被 fence 冻结，化解不掉的冲突以 Problem 形式标出并可用 pegging 追到受影响订单。这，就是一张覆盖"预测—库存—采购—生产—分销—交付"全链路的协同计划。

## 小结

这一章我们把模型从单厂扩展到了网络：一个 Buffer 有造、买、调三种补货来源，ItemDistribution 用带提前期、运费、优先级、批量和生效期的"通道"把仓库连成网，ItemSupplier 用硬提前期加软性安全提前期描述采购，两者都复用替代组机制；DRP 让缺口沿 BOM 和仓网两个维度递归上卷。所有结果统一落在 operationplan 一张表上，用 STCK/MO/WO/PO/DO/DLVR 区分身份，走 proposed→closed 的受控状态机，confirmed 即被冻结。而多数据库场景用物理隔离的沙箱支持零风险的 what-if 推演。

到这里，frePPLe 的核心能力我们全部走通了。最后一章，我们谈落地：怎么把它通过 REST、CSV、连接器接进你自己的 ERP/MES，怎么做无界面的后台引擎，以及在 MIT 协议的商业边界下，一个 Java 团队该如何与 Timefold 分工、把这套能力真正用到生产里。
