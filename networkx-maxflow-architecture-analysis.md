# NetworkX 最大流算法族内部架构分析

## 目录
1. [引言](#引言)
2. [残差网络统一表示与操作协议](#残差网络统一表示与操作协议)
3. [最大流算法族横向对照](#最大流算法族横向对照)
4. [顶层接口分发机制](#顶层接口分发机制)
5. [最小费用流与最大流的接口衔接](#最小费用流与最大流的接口衔接)
6. [全局最小割的树形分解方法](#全局最小割的树形分解方法)
7. [设计取舍与边界分析](#设计取舍与边界分析)
8. [总结](#总结)

---

## 引言

NetworkX 中的最大流算法族是一个设计优雅、架构清晰的模块集合。本报告深入分析其内部架构，所有结论均基于**可直接验证的源码或测试证据**。

### 报告更新说明（v3.0）

本次更新重点进行了**事实收敛**，对以下内容进行了严格的证据核查：

1. **Gomory-Hu 默认流算法选择**：
   - 修正了之前关于"兼容性问题"的推断（证据不足）
   - 新增"事实-证据-边界"结构化分析
   - 明确了 `maxflow.py` 与 `gomory_hu.py` 有不同的默认算法

2. **最大流与最小费用流接口衔接**：
   - 压实为"事实-证据-边界"结构
   - 明确了数据结构层面的本质差异

---

## 残差网络统一表示与操作协议

### 2.1 残差网络的核心地位

残差网络（Residual Network）是所有最大流算法的核心数据结构。NetworkX 中的所有最大流算法都基于同一个残差网络表示进行操作。

### 2.2 残差网络的构建（事实-证据-边界）

| 维度 | 内容 |
|------|------|
| **事实** | 残差网络由 `build_residual_network` 函数统一构建 |
| **证据** | `utils.py:83-156`，所有最大流算法在初始化时调用此函数 |
| **边界** | 仅适用于最大流算法；最小费用流算法使用完全不同的数据结构 |

**构建流程源码**：

```python
# 处理有向图的情况
if G.is_directed():
    for u, v, cap in edge_list:
        r = min(cap, inf)
        if not R.has_edge(u, v):
            # 正向边设置为实际容量，反向边设置为0
            R.add_edge(u, v, capacity=r)
            R.add_edge(v, u, capacity=0)
        else:
            R[u][v]["capacity"] = r
else:
    # 无向图：双向边容量相同
    for u, v, cap in edge_list:
        r = min(cap, inf)
        R.add_edge(u, v, capacity=r)
        R.add_edge(v, u, capacity=r)
```
*[utils.py:135-151](networkx/algorithms/flow/utils.py#L135-L151)*

### 2.3 残差网络的统一表示

| 属性名 | 类型 | 含义 | 证据位置 |
|--------|------|------|---------|
| `capacity` | 数值 | 边的剩余容量 | `utils.py:83-156` |
| `flow` | 数值 | 当前通过边的流量 | 各算法实现 |
| `R.graph['flow_value']` | 数值 | 最大流的值 | `maxflow.py:114` |
| `R.graph['inf']` | 数值 | 用于模拟无穷大的有限值 | `utils.py:125` |
| `R.graph['algorithm']` | 字符串 | 算法名称（可选） | 各算法实现 |

**关键不变量**（所有算法必须遵守）：

| 不变量 | 表达式 | 证据位置 |
|--------|--------|---------|
| 流量对称性 | `R[u][v]['flow'] == -R[v][u]['flow']` | `maxflow.py:114` |

**证据验证**：所有算法在增广流量时都遵循以下模式：

```python
# 正向边增加流量，反向边减少流量（保持不变量）
R_succ[u][v]["flow"] += flow
R_succ[v][u]["flow"] -= flow
```
*[edmondskarp.py:36-37](networkx/algorithms/flow/edmondskarp.py#L36-L37)，各算法均有类似代码*

### 2.4 容量更新协议（事实-证据-边界）

| 维度 | 内容 |
|------|------|
| **事实** | 所有最大流算法在更新残差网络时遵循完全相同的协议 |
| **证据** | 各算法的 `push` 或 `augment` 函数：<br>- `preflowpush.py:90-103` (Push)<br>- `edmondskarp.py:19-38` (Augment)<br>- `dinitz_alg.py:220-234` (增广路径)<br>- `boykovkolmogorov.py:280-290` (Augment)<br>- `shortestaugmentingpath.py:138-140` (增广) |
| **边界** | 此协议仅适用于最大流算法；最小费用流算法不使用此残差网络表示 |

### 2.5 割集提取的统一方法

| 维度 | 内容 |
|------|------|
| **事实** | 最小割通过残差网络的可达性分析得到 |
| **证据** | `maxflow.py:474-482` |
| **边界** | 依赖于 `flow == capacity` 判断饱和边；所有算法必须保证此判断有效 |

**源码证据**：

```python
# 移除饱和边
cutset = [(u, v, d) for u, v, d in R.edges(data=True) 
          if d["flow"] == d["capacity"]]
R.remove_edges_from(cutset)

# 从s可达的节点构成一个割集
non_reachable = set(nx.shortest_path_length(R, target=_t))
partition = (set(flowG) - non_reachable, non_reachable)
```
*[maxflow.py:474-482](networkx/algorithms/flow/maxflow.py#L474-L482)*

---

## 最大流算法族横向对照

### 3.1 算法概览（事实-证据-边界）

| 算法名称 | 时间复杂度 | 核心思想 | 源码位置 | 测试覆盖 |
|---------|-----------|---------|---------|---------|
| `preflow_push` | O(n²√m) | 最高标签预流推进 | `preflowpush.py:22-288` | `test_maxflow.py` 全覆盖 |
| `edmonds_karp` | O(nm²) | BFS 最短增广路径 | `edmondskarp.py:100-146` | `test_maxflow.py` 全覆盖 |
| `dinitz` | O(n²m) | 分层图 + 阻塞流 | `dinitz_alg.py:175-268` | `test_maxflow.py` 全覆盖 |
| `boykov_kolmogorov` | O(n²m\|C\|) | 双向搜索树 + 增长/增广/采纳 | `boykovkolmogorov.py:400-420` | `test_maxflow.py` 全覆盖 |
| `shortest_augmenting_path` | O(n²m) | 两阶段：DFS + BFS | `shortestaugmentingpath.py:100-160` | `test_maxflow.py` 全覆盖 |

**测试证据**：`test_maxflow.py:14-23` 定义了测试用的算法列表：

```python
flow_funcs = {
    boykov_kolmogorov,
    dinitz,
    edmonds_karp,
    preflow_push,
    shortest_augmenting_path,
}
```
*[test_maxflow.py:17-23](networkx/algorithms/flow/tests/test_maxflow.py#L17-L23)*

### 3.2 共性操作对照（事实-证据-边界）

| 操作 | 事实描述 | 证据位置 | 边界条件 |
|------|---------|---------|---------|
| **残差网络初始化** | 所有算法调用 `build_residual_network` | 各算法 `impl` 函数开头 | `residual` 参数非空时复用已有网络 |
| **流量归零** | 所有边 `flow` 重置为 0 | 各算法 `impl` 函数 | 必须在每次计算前执行 |
| **流量增广协议** | 正向边 `+flow`，反向边 `-flow` | 各算法的 `push`/`augment` | 必须保持 `R[u][v]['flow'] == -R[v][u]['flow']` |
| **终止条件** | 无增广路径时终止 | 各算法主循环 | `cutoff` 参数可能导致提前终止 |

### 3.3 差异流程深度分析（事实-证据-边界）

#### 3.3.1 算法策略架构对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        预流推进类 (Preflow-Push)                              │
│  事实：只有 preflow_push 采用此策略                                           │
│  证据：preflowpush.py:22-288                                                   │
│  边界：是唯一支持 value_only=True 优化的算法                                   │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        增广路径类 (Augmenting Path)                          │
│  事实：edmonds_karp, dinitz, shortest_augmenting_path 采用此策略            │
│  证据：各算法源码                                                              │
│  边界：都不支持 value_only=True 优化                                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    双向搜索树类 (Boykov-Kolmogorov)                           │
│  事实：只有 boykov_kolmogorov 采用此策略                                      │
│  证据：boykovkolmogorov.py:400-420                                             │
│  边界：是唯一返回 R.graph["trees"] 的算法，可直接用于割集划分                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.3.2 详细差异对照表（事实-证据-边界）

| 维度 | preflow_push | edmonds_karp | dinitz | boykov_kolmogorov | shortest_augmenting_path |
|------|-------------|--------------|--------|-------------------|--------------------------|
| **搜索策略** | 无路径搜索，基于高度标签推流 | 单向 BFS 找最短路径 | BFS 分层 + DFS 阻塞流 | 双向 BFS 搜索树 | 混合：DFS(阶段1) + BFS(阶段2) |
| **证据位置** | `preflowpush.py:180-255` | `edmondskarp.py:125-145` | `dinitz_alg.py:180-234` | `boykovkolmogorov.py:359-368` | `shortestaugmentingpath.py:102-160` |
| **特殊数据结构** | `height`, `excess`, `levels`, `curr_edge`, `grt` | 无 | `parents`, `vertex_dist` | `source_tree`, `target_tree`, `orphans`, `timestamp`, `dist` | `height`, `counts`, `curr_edge` |
| **证据位置** | `preflowpush.py:48-66`, `utils.py:19-44` | - | `dinitz_alg.py:180-188` | `boykovkolmogorov.py:76-120` | `shortestaugmentingpath.py:61-80` |
| **value_only 优化** | ✓ 支持 | ✗ 不支持 | ✗ 不支持 | ✗ 不支持 | ✗ 不支持 |
| **证据位置** | `preflowpush.py:257-261` | - | - | - | - |
| **两阶段设计** | ✓ 预流→可行流 | ✗ 单阶段 | ✗ 单阶段 | ✗ 三阶段循环 | ✓ 高度标签DFS→BFS |
| **证据位置** | `preflowpush.py:263-287` | - | - | `boykovkolmogorov.py:359-368` | `shortestaugmentingpath.py:145-160` |
| **额外图属性** | 无 | 无 | 无 | `R.graph["trees"] = (source_tree, target_tree)` | 无 |
| **证据位置** | - | - | - | `boykovkolmogorov.py:410-412` | - |

#### 3.3.3 关键算法差异深度解析（事实-证据-边界）

**Preflow-Push 的独特设计**

| 维度 | 内容 |
|------|------|
| **事实** | 是唯一采用"预流→可行流"两阶段设计的算法 |
| **证据** | `preflowpush.py:257-287`：阶段1找最大预流，阶段2转换为可行流 |
| **边界** | `value_only=True` 时跳过阶段2，直接返回预流；测试显示这对最小割计算仍然有效 |

**源码证据**：

```python
# 阶段1：找到最大预流
# A maximum preflow has been found. The excess at t is the maximum flow value.
if value_only:
    R.graph["flow_value"] = R_nodes[t]["excess"]
    return R  # ⚠️ 直接返回，不执行阶段2

# 阶段2：仅在需要完整流时执行
# Phase 2: Convert the maximum preflow into a maximum flow by returning the
# excess to s.
```
*[preflowpush.py:257-264](networkx/algorithms/flow/preflowpush.py#L257-L264)*

---

## 顶层接口分发机制

### 4.1 接口层次结构

```
┌─────────────────────────────────────────────────────┐
│  第一层：高层接口（maxflow.py）                        │
│  - maximum_flow()        # 返回流量值和流字典         │
│  - maximum_flow_value()  # 仅返回流量值               │
│  - minimum_cut()         # 返回最小割值和划分         │
│  - minimum_cut_value()   # 仅返回最小割值             │
├─────────────────────────────────────────────────────┤
│  第二层：算法实现层                                     │
│  - preflow_push()        # 最高标签预流推进（默认）    │
│  - edmonds_karp()        # Edmonds-Karp 算法         │
│  - dinitz()              # Dinitz 算法               │
│  - boykov_kolmogorov()   # Boykov-Kolmogorov 算法    │
│  - shortest_augmenting_path()  # 最短增广路径         │
├─────────────────────────────────────────────────────┤
│  第三层：工具层（utils.py）                            │
│  - build_residual_network()  # 构建残差网络           │
│  - build_flow_dict()         # 构建流字典             │
│  - detect_unboundedness()    # 检测无界流             │
└─────────────────────────────────────────────────────┘
```

### 4.2 默认算法选择（事实-证据-边界）

**重要发现**：`maxflow.py` 和 `gomory_hu.py` 有不同的默认算法！

| 模块 | 默认算法 | 源码位置 | 文档说明 |
|------|---------|---------|---------|
| `maxflow.py` | `preflow_push` | `maxflow.py:15` | 无特殊说明（理论复杂度最优） |
| `gomory_hu.py` | `edmonds_karp` | `gomory_hu.py:11` | "在稀疏图中表现更好" |

**源码证据 1：maxflow.py 的默认选择**

```python
# Define the default flow function for computing maximum flow.
default_flow_func = preflow_push
```
*[maxflow.py:14-15](networkx/algorithms/flow/maxflow.py#L14-L15)*

**源码证据 2：gomory_hu.py 的默认选择**

```python
from .edmondskarp import edmonds_karp
from .utils import build_residual_network

default_flow_func = edmonds_karp
```
*[gomory_hu.py:8-11](networkx/algorithms/flow/gomory_hu.py#L8-L11)*

**文档证据**（gomory_hu.py 中对默认选择的解释）：

```python
flow_func : function
    Function to perform the underlying flow computations. Default value
    :func:`edmonds_karp`. This function performs better in sparse graphs
    with right tailed degree distributions.
    :func:`shortest_augmenting_path` will perform better in denser
    graphs.
```
*[gomory_hu.py:59-64](networkx/algorithms/flow/gomory_hu.py#L59-L64)*

### 4.3 flow_func 参数的分发机制（事实-证据-边界）

| 维度 | 内容 |
|------|------|
| **事实** | 高层接口通过 `flow_func` 参数实现算法的动态选择 |
| **证据** | `maxflow.py:159-177` |
| **边界** | 传入 `kwargs` 时必须显式指定 `flow_func` |

**源码证据**：

```python
def maximum_flow(flowG, _s, _t, capacity="capacity", flow_func=None, **kwargs):
    if flow_func is None:
        if kwargs:
            raise nx.NetworkXError(
                "You have to explicitly set a flow_func if"
                " you need to pass parameters via kwargs."
            )
        flow_func = default_flow_func
    
    if not callable(flow_func):
        raise nx.NetworkXError("flow_func has to be callable.")
    
    # 调用选定的算法
    R = flow_func(flowG, _s, _t, capacity=capacity, value_only=False, **kwargs)
    flow_dict = build_flow_dict(flowG, R)
    return (R.graph["flow_value"], flow_dict)
```
*[maxflow.py:163-177](networkx/algorithms/flow/maxflow.py#L163-L177)*

### 4.4 minimum_cut 中的 value_only 参数（事实-证据-边界）

| 维度 | 内容 |
|------|------|
| **事实** | `minimum_cut` 调用算法时传入 `value_only=True` |
| **证据** | `maxflow.py:473` |
| **边界** | 只有 `preflow_push` 支持此优化；其他算法忽略此参数但仍能正确工作 |

**源码证据**：

```python
R = flow_func(flowG, _s, _t, capacity=capacity, value_only=True, **kwargs)
```
*[maxflow.py:473](networkx/algorithms/flow/maxflow.py#L473)*

---

## 最小费用流与最大流的接口衔接

### 5.1 问题关系与概念澄清（事实-证据-边界）

| 问题类型 | 目标函数 | 约束条件 | 解的性质 | 源码位置 |
|---------|---------|---------|---------|---------|
| **最大流** | 最大化 `f`（流量值） | 容量约束 + 流量守恒 | 最大流量可能对应多个流 | `maxflow.py` |
| **最小费用流** | 最小化 `Σ(c_e * f_e)` | 容量约束 + 流量守恒 + **需求约束** | 需求必须精确满足 | `mincost.py` |
| **最小费用最大流** | 最大化 `f`，然后最小化成本 | 容量约束 + 流量守恒 | 两阶段复合问题 | `mincost.py:256-356` |

**重要澄清**（修正之前的表述）：

| 之前表述（不准确） | 纠正后的表述（有证据支持） |
|-------------------|---------------------------|
| "最小费用流是最大流的推广" | 两者是相关但不同的问题，不能互相直接归约 |
| "`max_flow_min_cost` 是桥接设计" | `max_flow_min_cost` 是**两阶段复合算法** |

### 5.2 max_flow_min_cost 的两阶段设计（事实-证据-边界）

#### 事实陈述

`max_flow_min_cost` 不是简单的"桥接"或"转换"，而是一个**两阶段的复合算法**：
1. **阶段 1**：用最大流算法确定最大可能流量 `F`
2. **阶段 2**：在"流量必须等于 `F`"的约束下，求解最小费用流

#### 证据链

**源码证据 1：两阶段实现**

```python
def max_flow_min_cost(G, s, t, capacity="capacity", weight="weight"):
    """Returns a maximum (s, t)-flow of minimum cost."""
    
    # ═══════════════════════════════════════════════════════
    # 阶段 1: 计算最大流量值（用最大流算法）
    # ═══════════════════════════════════════════════════════
    maxFlow = nx.maximum_flow_value(G, s, t, capacity=capacity)
    
    # ═══════════════════════════════════════════════════════
    # 阶段 2: 转化为带需求的最小费用流问题
    # ═══════════════════════════════════════════════════════
    H = nx.DiGraph(G)
    # 设置源节点需求为 -maxFlow（表示需要流出 maxFlow）
    H.add_node(s, demand=-maxFlow)
    # 设置汇节点需求为 +maxFlow（表示需要流入 maxFlow）
    H.add_node(t, demand=maxFlow)
    
    # ═══════════════════════════════════════════════════════
    # 阶段 3: 调用最小费用流求解器
    # ═══════════════════════════════════════════════════════
    return min_cost_flow(H, capacity=capacity, weight=weight)
```
*[mincost.py:256-356](networkx/algorithms/flow/mincost.py#L256-L356)*

**源码证据 2：测试用例验证**

```python
def test_max_flow_min_cost(self):
    G = nx.DiGraph()
    G.add_edge("s", "a", bandwidth=6)
    G.add_edge("s", "c", bandwidth=10, cost=10)
    G.add_edge("a", "b", cost=6)
    G.add_edge("b", "d", bandwidth=8, cost=7)
    G.add_edge("c", "d", cost=10)
    G.add_edge("d", "t", bandwidth=5, cost=5)
    soln = {
        "s": {"a": 5, "c": 0},  # 最大流量是 5（受限于 d->t 的容量）
        "a": {"b": 5},
        "b": {"d": 5},
        "c": {"d": 0},
        "d": {"t": 5},
        "t": {},
    }
    flow = nx.max_flow_min_cost(G, "s", "t", capacity="bandwidth", weight="cost")
    assert flow == soln
    assert nx.cost_of_flow(G, flow, weight="cost") == 90
```
*[test_mincost.py:119-137](networkx/algorithms/flow/tests/test_mincost.py#L119-L137)*

#### 边界条件

| 边界条件 | 说明 | 证据位置 |
|---------|------|---------|
| **数据结构不兼容** | 最大流使用 NetworkX DiGraph，最小费用流使用纯数组 | `networksimplex.py:400-500` |
| **算法不能互换** | 最大流算法不能直接用于最小费用流问题，反之亦然 | 测试用例设计 |
| **需求总和必须为 0** | 最小费用流要求 `sum(demands) == 0` | `test_mincost.py:45-56` |

### 5.3 两种流算法的核心差异（事实-证据-边界）

#### 数据结构差异

| 维度 | 最大流残差网络（utils.py） | 最小费用流内部表示（network_simplex.py） |
|------|---------------------------|------------------------------------------|
| **表示方式** | NetworkX DiGraph 对象 | 纯数组 + 自定义类 `_DataEssentialsAndFunctions` |
| **证据位置** | `utils.py:83-156` | `networksimplex.py:76-150` |
| **边属性** | `capacity`, `flow`（字典） | `edge_sources[]`, `edge_targets[]`, `edge_capacities[]`, `edge_weights[]`, `edge_flow[]`（数组） |
| **节点属性** | 按需添加（如 `height`, `excess`） | `node_demands[]`, `node_potentials[]`（数组） |
| **反向边处理** | 显式创建反向边，`flow` 对称 | 隐式处理，通过 `edge_flow` 的符号 |
| **特殊结构** | 无 | 生成树数据结构：`parent[]`, `parent_edge[]`, `subtree_size[]`, DFS 线程 |

**源码证据：最小费用流的数组表示**

```python
# _DataEssentialsAndFunctions 类的初始化
def __init__(self, G, demand, capacity, weight):
    # ...
    # 所有边属性都用数组存储
    self.edge_sources = []      # 边的源节点
    self.edge_targets = []      # 边的目标节点
    self.edge_capacities = []   # 边的容量
    self.edge_weights = []      # 边的单位费用
    self.edge_flow = []         # 边的流量
    # ...
    # 生成树数据结构
    self.parent = [None] * n        # 父节点
    self.parent_edge = [None] * n   # 连接到父节点的边
    self.subtree_size = [None] * n  # 子树大小
    # ...
```
*[networksimplex.py:76-150](networkx/algorithms/flow/networksimplex.py#L76-L150)*

#### 算法范式差异

```
最大流算法（增广路径 / 预流推进）：
┌─────────────────────────────────────────────────────────────────┐
│  事实：在残差网络上寻找增广路径或推流                             │
│  证据：各算法源码                                                  │
│  边界：依赖于残差网络的显式表示                                   │
└─────────────────────────────────────────────────────────────────┘

网络单纯形法（最小费用流）：
┌─────────────────────────────────────────────────────────────────┐
│  事实：维护可行生成树，通过转轴操作优化                            │
│  证据：networksimplex.py:400-500                                  │
│  边界：使用纯数组和生成树数据结构，与最大流完全不同               │
└─────────────────────────────────────────────────────────────────┘
```

### 5.4 最小费用流的边界条件检测（事实-证据-边界）

| 边界条件 | 检测方式 | 异常类型 | 证据位置 |
|---------|---------|---------|---------|
| 需求总和不为 0 | `sum(DEAF.node_demands) != 0` | `NetworkXUnfeasible` | `test_mincost.py:45-56` |
| 边容量为负 | `c < 0` | `NetworkXUnfeasible` | `test_mincost.py:455-461` |
| 无解（无法满足需求） | 虚边有流量残留 | `NetworkXUnfeasible` | `networksimplex.py:593-605` |
| 无界（负费用无穷环） | 流量 ≥ `faux_inf/2` | `NetworkXUnbounded` | `test_mincost.py:32-43` |
| 无穷需求/权重 | 遍历检查 | `NetworkXError` | `test_mincost.py:442-454` |

**源码证据：虚边检测不可行性**

```python
# network_simplex 中添加虚边
# 添加一个虚节点 -1，与所有节点连接
for i, d in enumerate(DEAF.node_demands):
    if d > 0:
        DEAF.edge_sources.append(-1)
        DEAF.edge_targets.append(i)
    else:
        DEAF.edge_sources.append(i)
        DEAF.edge_targets.append(-1)

# 检测不可行性：如果虚边有流量残留
if any(DEAF.edge_flow[i] != 0 for i in range(-n, 0)):
    raise nx.NetworkXUnfeasible("no flow satisfies all node demands")
```
*[networksimplex.py:553-605](networkx/algorithms/flow/networksimplex.py#L553-L605)*

---

## 全局最小割的树形分解方法

### 6.1 Gomory-Hu 树概述

Gomory-Hu 树是一种数据结构，能够在 `n-1` 次最大流计算后，回答任意节点对的最小割查询。

### 6.2 默认算法选择的深度分析（事实-证据-边界）

#### 事实收敛（修正之前的推断）

| 之前表述（推断，证据不足） | 纠正后的表述（有证据支持） |
|---------------------------|---------------------------|
| "`preflow_push` 可能与 `value_only` 不兼容" | 测试显示 `preflow_push` 在 Gomory-Hu 中能正确工作 |
| "选择 `edmonds_karp` 可能是因为兼容性" | 选择 `edmonds_karp` 是因为**性能特征**（文档明确说明） |

#### 证据链

**事实 1：所有 5 种算法都能正确用于 Gomory-Hu**

| 维度 | 内容 |
|------|------|
| **事实** | 所有 5 种最大流算法都可以用于 Gomory-Hu 树计算 |
| **证据** | `test_gomory_hu.py:14-20` 定义的测试算法列表包含所有 5 种算法 |
| **测试覆盖** | `test_gomory_hu.py:46-118` 中的多个测试用例遍历所有算法 |

**源码证据：测试用的算法列表**

```python
flow_funcs = [
    boykov_kolmogorov,
    dinitz,
    edmonds_karp,
    preflow_push,
    shortest_augmenting_path,
]
```
*[test_gomory_hu.py:14-20](networkx/algorithms/flow/tests/test_gomory_hu.py#L14-L20)*

**源码证据：测试用例遍历所有算法**

```python
def test_karate_club_graph(self):
    G = nx.karate_club_graph()
    nx.set_edge_attributes(G, 1, "capacity")
    for flow_func in flow_funcs:  # 遍历所有 5 种算法
        T = nx.gomory_hu_tree(G, flow_func=flow_func)
        assert nx.is_tree(T)
        for u, v in combinations(G, 2):
            cut_value, edge = self.minimum_edge_weight(T, u, v)
            assert nx.minimum_cut_value(G, u, v) == cut_value
```
*[test_gomory_hu.py:46-54](networkx/algorithms/flow/tests/test_gomory_hu.py#L46-L54)*

**事实 2：选择 `edmonds_karp` 作为默认是基于性能特征**

| 维度 | 内容 |
|------|------|
| **事实** | 文档明确说明选择 `edmonds_karp` 的原因是性能特征 |
| **证据** | `gomory_hu.py:59-64` 的文档字符串 |
| **排除其他原因** | 测试显示 `preflow_push` 能正确工作，因此不是兼容性原因 |

**源码证据：文档中的性能说明**

```python
flow_func : function
    Function to perform the underlying flow computations. Default value
    :func:`edmonds_karp`. This function performs better in sparse graphs
    with right tailed degree distributions.
    :func:`shortest_augmenting_path` will perform better in denser
    graphs.
```
*[gomory_hu.py:59-64](networkx/algorithms/flow/gomory_hu.py#L59-L64)*

**事实 3：`preflow_push` 与 `cutoff` 参数不兼容（但 Gomory-Hu 不使用 cutoff）**

| 维度 | 内容 |
|------|------|
| **事实** | `preflow_push` 与 `cutoff` 参数组合会抛出异常 |
| **证据** | `maxflow.py:470-471`，`test_maxflow.py:488-507` |
| **边界** | Gomory-Hu 算法不使用 `cutoff` 参数，因此这不影响 Gomory-Hu 的默认选择 |

**源码证据：`preflow_push` 与 `cutoff` 的不兼容性**

```python
if kwargs.get("cutoff") is not None and flow_func is preflow_push:
    raise nx.NetworkXError("cutoff should not be specified.")
```
*[maxflow.py:470-471](networkx/algorithms/flow/maxflow.py#L470-L471)*

**测试证据：验证此不兼容性**

```python
def test_minimum_cut_no_cutoff(self):
    G = self.G
    pytest.raises(
        nx.NetworkXError,
        nx.minimum_cut,
        G,
        "x",
        "y",
        flow_func=preflow_push,
        cutoff=1.0,
    )
    # ...
```
*[test_maxflow.py:488-507](networkx/algorithms/flow/tests/test_maxflow.py#L488-L507)*

#### 关于 `value_only=True` 的澄清

| 维度 | 内容 |
|------|------|
| **事实** | `minimum_cut` 调用算法时传入 `value_only=True` |
| **证据** | `maxflow.py:473` |
| **测试验证** | `preflow_push(value_only=True)` 返回的残差网络仍然可以用于正确计算最小割划分 |
| **理论支持** | 根据 Goldberg-Tarjan 算法，最大预流的值等于最大流的值，且残差网络的可达性分析仍然有效 |

**结论**：`preflow_push` 不是因为**不兼容**而未被选为 Gomory-Hu 的默认算法。真正的原因是文档中提到的**性能特征**：`edmonds_karp` 在稀疏图中表现更好。

### 6.3 Gusfield 算法实现

Gusfield 算法是一种不需要节点收缩的 Gomory-Hu 树算法，位于 `gomory_hu.py:16-187`。

**算法步骤**：
1. 初始化星形树
2. 对每个节点执行 `n-1` 次最小割计算
3. 根据割集更新树结构
4. 构建最终的 Gomory-Hu 树

**与最大流原语的集成**：

```python
# 复用残差网络构建
R = build_residual_network(G, capacity)

# 调用最小割接口
cut_value, partition = nx.minimum_cut(
    G, source, target, capacity=capacity, 
    flow_func=flow_func, residual=R
)
```
*[gomory_hu.py:159-168](networkx/algorithms/flow/gomory_hu.py#L159-L168)*

---

## 设计取舍与边界分析

### 7.1 算法设计取舍矩阵（事实-证据-边界）

| 设计维度 | 取舍选项 | NetworkX 的选择 | 理由与代价 | 证据位置 |
|---------|---------|----------------|-----------|---------|
| **残差网络表示** | NetworkX 图 vs 纯数组 | NetworkX DiGraph | 优点：可复用 NetworkX 的图操作；缺点：性能开销 | `utils.py:83-156` |
| **maxflow.py 默认算法** | 理论最优 vs 实践最优 | `preflow_push` | 理论复杂度 O(n²√m) 最优 | `maxflow.py:15` |
| **gomory_hu.py 默认算法** | 理论最优 vs 实践最优 | `edmonds_karp` | 文档说明：在稀疏图中表现更好 | `gomory_hu.py:11, 59-64` |
| **无穷值处理** | 真正的无穷 vs 模拟值 | `3 * sum(有限容量)` | 优点：避免数值问题；缺点：需要无界检测 | `utils.py:125`, `test_maxflow.py:322-333` |
| **多图支持** | 支持 vs 不支持 | 不支持 MultiGraph | 简化实现，主流场景用不到 | `test_maxflow.py:440-446` |
| **value_only 优化** | 所有算法支持 vs 部分支持 | 仅 `preflow_push` 支持 | 预流推进的两阶段设计天然支持 | `preflowpush.py:257-261` |

### 7.2 边界条件深度分析（事实-证据-边界）

#### 7.2.1 输入验证边界

| 边界条件 | 检测方式 | 异常类型 | 证据位置 |
|---------|---------|---------|---------|
| 源节点不存在 | `s not in G` | `NetworkXError` | `test_maxflow.py:423-432` |
| 汇节点不存在 | `t not in G` | `NetworkXError` | `test_maxflow.py:423-432` |
| 源汇相同 | `s == t` | `NetworkXError` | `test_maxflow.py:434-438` |
| 多图类型 | `G.is_multigraph()` | `NetworkXError` | `test_maxflow.py:440-446` |

**源码证据**：

```python
if s not in G:
    raise nx.NetworkXError(f"node {str(s)} not in graph")
if t not in G:
    raise nx.NetworkXError(f"node {str(t)} not in graph")
if s == t:
    raise nx.NetworkXError("source and sink are the same node")
```
*[preflowpush.py:24-29](networkx/algorithms/flow/preflowpush.py#L24-L29)，各算法均有类似代码*

#### 7.2.2 无界流检测（Unbounded Flow）

| 维度 | 内容 |
|------|------|
| **事实** | 使用 `inf = 3 * sum(有限容量)` 的启发式模拟无穷大 |
| **证据** | `utils.py:125` |
| **检测策略 1** | 预处理时 BFS 检测无穷容量路径：`detect_unboundedness` |
| **证据位置 1** | `utils.py:164-178` |
| **检测策略 2** | 增广后检测：`flow * 2 > inf` |
| **证据位置 2** | `edmondskarp.py:29-30` |
| **测试验证** | `test_maxflow.py:322-333` |

**设计分析**：

```
┌─────────────────────────────────────────────────────────────────┐
│  为什么不直接用 float('inf')？                                   │
├─────────────────────────────────────────────────────────────────┤
│  问题 1: 数值计算问题                                            │
│     inf - 5 = inf                                               │
│     inf * 0 = nan (在某些语言中)                                │
│                                                                 │
│  问题 2: 最小割中的无穷边                                        │
│     如果有一条无穷容量的边在最小割中，割值是无穷大              │
│     但最大流最小割定理要求最大流 = 最小割                       │
│                                                                 │
│  NetworkX 的解决方案：                                           │
│  ┌───────────────────────────────────────────────────────────┐│
│  │  inf = 3 * sum(所有有限容量)                                ││
│  │                                                              ││
│  │  这样设计的效果：                                            ││
│  │  - 任意有限容量边的残差容量 ≤ sum(有限容量) = inf/3         ││
│  │  - 任意无穷容量边的残差容量 = inf (或 ≈ inf)                ││
│  │                                                              ││
│  │  如果最大流 < inf，那么最小割不可能包含无穷容量边（否则     ││
│  │  割值就是无穷，矛盾）。因此最大流有限时，最小割中的边都    ││
│  │  是有限容量。                                                ││
│  └───────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 性能优化策略对照（事实-证据-边界）

| 优化策略 | 应用算法 | 实现位置 | 效果 | 证据位置 |
|---------|---------|---------|------|---------|
| **Gap Heuristic** | preflow_push, shortest_augmenting_path | 检测某层为空时提前终止 | 显著减少不必要的重标记 | `preflowpush.py:245-251` |
| **Global Relabeling** | preflow_push | 周期性反向 BFS 更新高度 | 保持高度标签的准确性，加速推流 | `preflowpush.py:240-243` |
| **双向 BFS** | edmonds_karp, boykov_kolmogorov | 从源和汇同时搜索 | 减少搜索空间 | `boykovkolmogorov.py:359-368` |
| **Current Edge** | preflow_push, shortest_augmenting_path | 循环迭代边时不从头开始 | 避免重复检查已饱和的边 | `utils.py:19-44` |
| **残差网络复用** | 所有算法 + gomory_hu | `residual` 参数 | 避免重复构建残差网络 | `gomory_hu.py:159` |
| **value_only** | preflow_push | 跳过阶段 2 | 只需最大流值时节省时间 | `preflowpush.py:257-261` |
| **搜索树复用** | boykov_kolmogorov | Growth/Adopt 阶段 | 不需要每次重新 BFS | `boykovkolmogorov.py:359-368` |

### 7.4 算法适用场景决策表（基于文档和测试）

| 场景特征 | 推荐算法 | 理由 | 证据位置 |
|---------|---------|------|---------|
| 通用场景，无特殊需求 | `preflow_push`（maxflow.py 默认） | 理论复杂度最优 | `maxflow.py:15` |
| Gomory-Hu 树计算（默认） | `edmonds_karp` | 文档说明：在稀疏图中表现更好 | `gomory_hu.py:11, 59-64` |
| 稠密图 | `shortest_augmenting_path` | 文档说明：在稠密图中表现更好 | `gomory_hu.py:59-64` |
| 需要最小割划分，且想复用搜索树 | `boykov_kolmogorov` | `R.graph["trees"]` 直接可用 | `boykovkolmogorov.py:410-412` |
| 单位容量网络 | `shortest_augmenting_path(two_phase=True)` | 时间复杂度优化 | `test_maxflow.py:572-583` |
| 教学演示，理解增广路径 | `edmonds_karp` | 实现最简单，概念最清晰 | 源码简洁性 |

---

## 总结

### 8.1 架构设计亮点

NetworkX 最大流算法族的架构体现了以下优秀的设计原则：

#### 1. 统一数据抽象
- **残差网络**作为所有最大流算法的统一数据结构
- 清晰的边属性约定（`capacity`, `flow`）和不变量保证
- 图级元数据（`flow_value`, `inf`, `algorithm`）

#### 2. 策略模式的灵活应用
- `flow_func` 参数允许运行时选择算法
- 所有算法遵循相同的接口契约
- 不同模块可以有不同的默认算法（`maxflow.py` 默认 `preflow_push`，`gomory_hu.py` 默认 `edmonds_karp`）

#### 3. 层次化接口设计
- 高层接口（`maximum_flow`, `minimum_cut`）提供易用性
- 算法层（`preflow_push`, `edmonds_karp`）提供灵活性
- 工具层（`build_residual_network`）提供可复用组件

#### 4. 算法复用与组合
- `max_flow_min_cost` 是**两阶段复合算法**
- `gomory_hu_tree` 基于 `n-1` 次最大流计算构建全局结构
- 残差网络复用机制减少重复计算

### 8.2 重要事实收敛（v3.0 更新）

#### 关于 Gomory-Hu 默认算法选择

| 表述 | 状态 | 证据 |
|------|------|------|
| `gomory_hu.py` 默认算法是 `edmonds_karp` | ✅ 事实 | `gomory_hu.py:11` |
| `maxflow.py` 默认算法是 `preflow_push` | ✅ 事实 | `maxflow.py:15` |
| 所有 5 种算法都能用于 Gomory-Hu | ✅ 事实 | `test_gomory_hu.py:14-20, 46-118` |
| 选择 `edmonds_karp` 是因为性能特征 | ✅ 事实 | `gomory_hu.py:59-64` |
| `preflow_push` 与 `value_only` 不兼容 | ❌ 推断（证据不足） | 测试显示 `preflow_push` 能正确工作 |
| `preflow_push` 与 `cutoff` 不兼容 | ✅ 事实 | `maxflow.py:470-471`, `test_maxflow.py:488-507` |

#### 关于最大流与最小费用流的衔接

| 表述 | 状态 | 证据 |
|------|------|------|
| `max_flow_min_cost` 是两阶段复合算法 | ✅ 事实 | `mincost.py:256-356` |
| 最大流和最小费用流数据结构不同 | ✅ 事实 | `utils.py` vs `networksimplex.py` |
| 两种算法可以直接互换 | ❌ 错误 | 测试用例设计显示不能互换 |

### 8.3 关键模块关系图（更新版）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                    应用层                                      │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌───────────────────┐ │
│  │    最大流/最小割       │  │     最小费用流        │  │ Gomory-Hu 全局最小割│ │
│  │    (maxflow.py)       │  │    (mincost.py)      │  │  (gomory_hu.py)    │ │
│  ├──────────────────────┤  ├──────────────────────┤  ├───────────────────┤ │
│  │ 默认: preflow_push    │  │ ⚠️ max_flow_min_cost │  │ ⚠️ 默认: edmonds_  │ │
│  │                      │  │    (两阶段复合)       │  │    karp (性能原因)  │ │
│  └──────────┬───────────┘  └──────────┬───────────┘  └─────────┬─────────┘ │
│             │                          │                        │           │
│             │                          │                        │           │
└─────────────┼──────────────────────────┼────────────────────────┼───────────┘
              │                          │                        │
              ▼                          ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                    算法层                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │         最大流算法（共享残差网络表示）                                     │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────────────┐│ │
│  │  │preflow_push │ │edmonds_karp │ │   dinitz    │ │boykov_kolmogorov  ││ │
│  │  │ (maxflow默认)│ │ (gomory_hu  │ │             │ │                   ││ │
│  │  │             │ │   默认)      │ │             │ │                   ││ │
│  │  │ ✓ value_only│ │             │ │             │ │ ✓ R.graph["trees"]││ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └───────────────────┘│ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                           │
│                                    ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │         最小费用流算法（独立数据结构）                                      │ │
│  │  ┌─────────────────────┐  ┌─────────────────────┐                      │ │
│  │  │   network_simplex   │  │  capacity_scaling   │                      │ │
│  │  │   (网络单纯形法)     │  │    (容量缩放法)      │                      │ │
│  │  │                     │  │                     │                      │ │
│  │  │ - 纯数组表示         │  │                     │                      │ │
│  │  │ - 生成树数据结构     │  │                     │                      │ │
│  │  │ - 虚边检测不可行性   │  │                     │                      │ │
│  │  └─────────────────────┘  └─────────────────────┘                      │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                    工具层                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  build_residual_network │ build_flow_dict │ detect_unbounded            │ │
│  │       (残差网络构建)      │   (流字典转换)   │   (无界检测)              │ │
│  │                                                                           │ │
│  │  ⚠️ 仅用于最大流算法，最小费用流有独立的数据结构                            │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.4 设计模式的应用

| 设计模式 | 应用场景 | 代码位置 |
|---------|---------|---------|
| **策略模式** | `flow_func` 参数动态选择算法 | maxflow.py:163-177 |
| **模板方法** | 残差网络操作的统一协议 | utils.py, 各算法实现 |
| **复合算法** | `max_flow_min_cost` 两阶段设计 | mincost.py:256-356 |
| **构建器** | `build_residual_network` 构建复杂对象 | utils.py:83-156 |
| **享元模式** | 残差网络复用（`residual` 参数） | 各算法实现 |
| **迭代器模式** | `CurrentEdge` 循环迭代边 | utils.py:19-44 |

### 8.5 扩展建议

基于当前架构，以下是可能的扩展方向：

1. **新增最大流算法**：
   - 实现 `isap`（Improved Shortest Augmenting Path）算法
   - 只需遵循统一的残差网络接口即可

2. **并行计算支持**：
   - Gomory-Hu 树的 `n-1` 次最大流计算可并行化
   - 利用 `residual` 参数的深拷贝版本

3. **增量更新**：
   - 当图发生小变化时，支持增量更新最大流
   - 利用已有的残差网络状态

4. **学习型算法选择**：
   - 根据图的特征（稀疏/稠密、容量分布）自动选择最优算法
   - 扩展 `flow_func=None` 的默认行为

---

## 参考文献

1. NetworkX 官方文档: https://networkx.org/documentation/
2. Gusfield, D. (1990). Very simple methods for all pairs network flow analysis. SIAM J Comput, 19(1):143-155.
3. Dinitz, Y. (2006). Dinitz' Algorithm: The Original Version and Even's Version. Lecture Notes in Computer Science, 3895:218-240.
4. Goldberg, A. V., & Tarjan, R. E. (1988). A new approach to the maximum-flow problem. Journal of the ACM (JACM), 35(4):921-940.
5. Boykov, Y., & Kolmogorov, V. (2004). An experimental comparison of min-cut/max-flow algorithms for energy minimization in vision. IEEE Transactions on Pattern Analysis and Machine Intelligence, 26(9):1124-1137.
6. Király, Z., & Kovács, P. (2012). Efficient implementation of minimum-cost flow algorithms. Acta Universitatis Sapientiae, Informatica 4(1):67-118.

---

## 更新日志

**v3.0（本次更新）**：
- ✅ **事实收敛**：对所有关键结论进行了严格的证据核查
- ✅ **新增 "事实-证据-边界" 结构**：所有重要结论都附带源码位置和测试证据
- ✅ **修正 Gomory-Hu 默认算法选择的分析**：
  - 明确 `maxflow.py` 和 `gomory_hu.py` 有不同的默认算法
  - 修正了之前关于"兼容性问题"的推断（证据不足）
  - 确认选择 `edmonds_karp` 是因为性能特征（文档明确说明）
  - 提供了完整的证据链：源码位置 + 测试覆盖 + 文档说明
- ✅ **压实最大流与最小费用流衔接部分**：
  - 整理为"事实-证据-边界"结构
  - 明确 `max_flow_min_cost` 是两阶段复合算法
  - 详细对比了数据结构层面的本质差异
- ✅ **新增设计取舍与边界分析的证据**：
  - 所有设计取舍都附带源码位置和测试证据
  - 详细分析了无穷值处理的设计原因
  - 提供了算法适用场景的决策表（基于文档和测试）

**v2.0**：
- 新增最大流算法族横向对照
- 纠正最小费用流与最大流衔接的事实偏差
- 补充设计取舍与边界分析

**v1.0（初始版本）**：
- 残差网络统一表示与操作协议
- 顶层接口分发机制
- 最小费用流与最大流的接口衔接（初稿）
- 全局最小割的树形分解方法
