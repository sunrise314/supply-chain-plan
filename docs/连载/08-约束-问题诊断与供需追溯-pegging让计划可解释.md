# 约束、问题诊断与供需追溯：让计划不仅算得出，还说得清

一张计划算出来，如果只能看到一张漂亮的甘特图，那它对实际工作的价值不到一半。计划员每天真正面对的是一堆麻烦：哪张订单要晚交、为什么晚、是卡在设备还是卡在物料、这颗缺了的螺丝会连累哪些客户订单、要不要现在就去催那批采购？这些问题的答案，才是计划系统的核心产出。frePPLe 用两套机制回答它们：**Problem（问题标记）负责"哪里不对劲"，Pegging（供需追溯）负责"谁连累了谁"。** 这一章我们看它怎么把第 7 章那个递归求解过程，变成一份能拿去开会决策的诊断报告。

## Problem：求解过程中随时记账

先纠正一个常见误解：很多人以为问题是计划跑完后另外做一遍检查发现的。其实不是。在 frePPLe 里，**发现问题和求解是同时发生的**。还记得第 7 章 `checkOperation()` 里那个 `data.constraints` 容器吗？求解器在试探排程的过程中，一旦发现某个候选安排违反了约束，就立刻往这个容器里 push 一个问题对象，用来决定要不要移动工序、要不要换替代。求解结束时，那些通过调整已经被化解的问题会随回滚消失，**最终仍然化解不掉的，就固化成挂在实体上的 Problem 记录，展示在问题报告里。**

所有问题都继承自 `Problem` 基类，共享一组统一的属性：归属实体（owner，是某个资源、某个库存点还是某道工序）、问题类别 `getEntity()`、一个时间区间 `getDates()`、涉及数量，以及最关键的一个布尔判断 `isFeasible()`。我在 `include/frepple/model.h` 里读到了一批具体的问题类，这里把最常用的整理成一张表：

| 问题类 | 类别 | isFeasible | 触发场景 |
| --- | --- | --- | --- |
| `ProblemCapacityOverload` | capacity 产能 | 否 | 资源负荷超过 `maximum`，带超载量和时间段 |
| `ProblemMaterialShortage` | material 物料 | 否 | Buffer 在某时间段库存被打到负数，带缺口量 |
| `ProblemBeforeCurrent` | operation 工序 | 否 | 要满足需求必须在"今天之前"就开工 |
| `ProblemPrecedence` | operation 工序 | 否 | 工艺路线里前后工序的先后依赖被违反 |
| `ProblemSyncDemand` | operation 工序 | — | 同组需求需要同步交付，本需求无法独立安排 |
| `ProblemInvalidData` | — | — | 数据错误，比如计划期内根本找不到可用时间 |
| `ConstraintPurchasingLeadTime` | material 物料 | 否 | 要求的到货早于采购提前期允许的最早时间 |
| `ProblemAwaitSupply` | material 物料 | **是** | 在等待已确认的供给（在途采购/在制）到货 |
| `ProblemOutlier` | forecast 预测 | — | 销量历史中的离群点 |

我特别想讲两类，因为它们体现了设计者的细腻。

**第一是"硬问题"和"软告警"的区分，也就是 isFeasible。** 你看 `ProblemCapacityOverload` 和 `ProblemMaterialShortage` 的 `isFeasible()` 都返回 `false`——它们让计划变得不可行，必须处理。但 `ProblemAwaitSupply`（等待已确认供给）返回的是 `true`：

```cpp
class ProblemAwaitSupply : public Problem {
  string getDescription() const override {
    ...
    ch << "' awaits confirmed supply";   // "某 Buffer/工序 在等待已确认供给"
  }
  bool isFeasible() const override { return true; }   // 这其实是可行的，只是提醒
};
```

这太真实了：一颗料库存为 0，但有一张已下给供应商、确定 5 天后到货的采购单，计划把工序排在 5 天后——这不是不可行，只是提醒计划员"这批活押在那张在途订单上，记得盯着点"。把这类提示和真正的缺料/超载混为一谈，会让问题报告全是狼来了；区分可行与否，计划员才能先救火、再看提醒。

**第二是问题都带可读的描述和精确的时间段、数量。** 过载问题有超载量 `qty` 和时间区间 `dr`，缺料问题有缺口量和缺口持续区间，数据错误直接写出 `"No available time found on operation '...'"`。这意味着报告不是一句干巴巴的"有问题"，而是"11 月 18 日下午，数控车床超载 1.3 台、持续 4 小时"这种可以直接行动的信息。

## 三种计划模式，本质是三种"对待问题的态度"

把问题机制和第 6 章连起来，你会理解得更深。同一个排不过来的场景，三种 planType 的区别，几乎可以完全用"问题怎么处理"来重述：

- **无约束（planType=3）**：引擎不强行化解冲突，订单全部账面准时足量，但把 `ProblemCapacityOverload`、`ProblemMaterialShortage` 原封不动地留在计划上。所以**无约束计划的全部价值就体现在那一片问题标记里**——它如实告诉你"要让这些订单全准时，你缺多少产能、多少料"。
- **智能无约束（planType=2）**：先穷尽替代资源/替代物料，能化解的问题在求解中消掉，只把真正无法替代的缺口作为问题留下。
- **完全约束（planType=1）**：引擎主动用"晚交"或"短交"去消除硬问题，宁可改交期也不留不可行计划；于是你在问题报告里看到的更多是"晚交"类结果，而不是赤裸裸的超载/缺料。

这就是为什么我在第 6 章强调"跑完务必看问题报告，别只看甘特图"——甘特图是被问题处理策略加工过的表象，问题列表才是约束的原始真相。

## 替代与换型：问题是怎么被"化解"的

顺着问题这条线，把第 3 章埋的两个模型在这里闭环。当资源产能出问题时，引擎并非只能报 `ProblemCapacityOverload`，它会先尝试替代。回忆 `OperationResource`（Load）上的 `name` + `priority` + `search` 三兄弟：同名的一组 Load 构成替代资源池，首选资源满了，引擎会按优先级（以及 search 指定的偏好，比如优先成本最低）去问池子里的其他资源——包括通过 `ResourceSkill` 关联的、具备所需技能的人力。物料侧同理，同名的 `OperationMaterial`（Flow）构成替代物料组，首选料短缺就去问替代料的 Buffer。

换型则通过 `SetupMatrix`/`SetupRule` 影响时间：两道相邻工序在同一台设备上的前后状态（fromsetup→tosetup）决定要不要插入清洗、换模时间。如果排程把两道强冲突的产品挨在一起、白白吞下一大段换型，问题诊断里就能看出大量时间被 setup 吃掉，优化方向就是调整加工顺序、让相似产品成组（sequence-dependent setup）。智能无约束模式之所以更有说服力，正是因为它报告的缺口是"连替代资源、替代物料、成组换型都试过之后"剩下的。

## Pegging：沿着第 7 章那棵请求树双向追溯

问题告诉你"数控车床 18 号超载了"，但计划员下一句一定是："**那会连累哪些客户订单？**" 回答这个问题靠的是 Pegging（供需追溯/钉连）。它本质上是把第 7 章求解时走过的那棵 ask-reply 请求树记录下来，让你能沿着它正反向遍历。

引擎在头文件里提供了四个方向的追溯入口：

```cpp
PeggingIterator getPeggingUpstream() const;          // 向上游追：我依赖哪些料和产能
PeggingIterator getPeggingUpstreamFirstLevel() const;
PeggingIterator getPeggingDownstream() const;        // 向下游追：谁在依赖我
PeggingIterator getPeggingDownstreamFirstLevel() const;
```

而 `PeggingIterator` 可以从任何一个节点出发构造——从一张需求、一个工序计划、一条物料流计划甚至一条产能负载计划开始都行：

```cpp
PeggingIterator(const Demand*, short = -1);
PeggingIterator(const OperationPlan*, bool = true, short = -1);
PeggingIterator(const FlowPlan*, bool = true);
PeggingIterator(LoadPlan*, bool = true);
```

两个方向对应两类最高频的业务追问：

**向上游追（upstream）：这张订单到底用到了什么？** 从一张客户订单出发，经过它的交付工序、装配工序，沿物料流一路向下，列出它牵动的每一张生产工单、每一张采购申请、每一道资源占用。销售想知道"这张单现在进展到哪了、卡在哪一步"，向上游追一遍就是完整的订单全链路视图（这也正是 WMS/TMS 里我们想做的"全程可视"在计划层的对应物）。

**向下游追（downstream）：这个缺口会砸到谁头上？** 这是供应链里最有杀伤力的一问。某个物料 Buffer 出现 `ProblemMaterialShortage`，从这个 Buffer 向下游追，能列出所有消耗它的工序、再到所有最终受影响的客户订单及各自交期。配合专门的 `PeggingDemandIterator`（直接枚举受影响的需求），你能立刻回答："这颗芯片 18 号到不了，会导致 A、B、C 三张订单分别晚 3 天、5 天、5 天，涉及这三个客户。" 有了这份名单，催料、跟客户沟通改期、调整优先级才有依据——你知道该先救谁。

反过来也一样有用：采购紧急催回一批料、提前两天到货，从这批料向下游追，就能看到它能把哪些订单从晚交名单里救回来，从而决定值不值得为这趟加急付空运费。

## 计划员拿到报告后的标准动作

把 Problem 和 Pegging 组合起来，一个专业计划员的晨会工作流其实非常清晰：

1. **按影响排问题**：打开问题报告，先看 isFeasible=false 的硬问题，按时间和数量排序，锁定最早、最大的产能过载和物料缺口；
2. **向下游追订单**：对每个缺口做 downstream pegging，列出受影响的客户订单，结合订单优先级判断该保谁；
3. **向上游找对策**：对要保的订单做 upstream pegging，看瓶颈到底在哪个资源或哪张采购单；
4. **对症下药**：能加班/调班扩产能的扩产能，能换替代料/替代资源的换（问题报告里残留的缺口本就是替代穷尽后的真缺口），实在不行就用 pegging 名单提前跟客户谈改期或协商短交；
5. **回到场景里验证**：把这些措施在 what-if 场景（下一章）里试算，确认问题真的消了再带回正式计划。

在这个流程里，求解器不是一个"给出完美答案的神谕"，而是一个"光速试算并把代价如实列出来"的助手；最终的权衡（牺牲哪张单、花多少加急费）仍然是人基于商业判断做的。我认为这才是 APS 正确的人机分工。

## 小结

这一章我们看到，frePPLe 在求解的同时就在记账：化解不掉的冲突固化成带类别、时间段、数量和可读描述的 Problem，并区分"不可行硬问题"（产能过载、物料短缺、必须排到过去、采购提前期）和"可行软提醒"（等待已确认供给、离群点）。三种计划模式的差异本质是对待这些问题的态度。而 Pegging 把求解时的请求树变成可双向遍历的追溯链：upstream 回答订单用到了什么、downstream 回答缺口连累了谁，PeggingDemandIterator 直接给出受影响订单清单。计划因此从"一张图"升级成"一份能追责、能决策的诊断报告"。

下一章我们把视野从单厂内部拉到整个供应网络：多个仓库之间怎么自动补货（分销计划），采购提前期和供应商怎么建模，以及怎么用多数据库场景安全地做 what-if 沙盘推演。
