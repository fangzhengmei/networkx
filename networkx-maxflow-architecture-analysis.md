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

NetworkX 中的最大流算法族是一个设计优雅、架构清晰的模块集合。该模块通过统一的残差网络表示、灵活的算法选择机制以及与其他图算法的深度集成，展现了优秀的软件架构设计。

本报告深入分析 NetworkX 最大流算法族的内部架构，重点关注：
- 残差网络的统一表示和操作协议
- **多种最大流算法在残差网络上的共性操作与差异流程（新增横向对照）**
- 顶层接口如何分发请求到不同算法实现
- **最小费用流与最大流的关系及接口衔接（纠正事实偏差）**
- 全局最小割的树形分解方法如何基于最大流原语构建
- **设计取舍与边界分析（新增）**

---

## 残差网络统一表示与操作协议

### 2.1 残差网络的核心地位

残差网络（Residual Network）是所有最大流算法的核心数据结构。NetworkX 中的所有最大流算法都基于同一个残差网络表示进行操作，这是实现算法互换性的关键。

**核心设计原则：**
- 所有算法共享相同的残差网络结构
- 边容量更新遵循统一的协议
- 增广路径计算基于相同的图操作原语

### 2.2 残差网络的构建

残差网络的构建由 `build_residual_network` 函数统一实现，位于 `networkx/algorithms/flow/utils.py:83-156`。

#### 构建流程：

1. **输入验证**：检查图类型，不支持 MultiGraph 和 MultiDiGraph
2. **容量函数处理**：将字符串或函数类型的 capacity 参数转换为统一的函数接口
3. **边过滤**：提取正容量的非自环边
4. **无穷值处理**：使用 `3 * sum(有限容量)` 作为无穷大的近似值
5. **双向边创建**：为每条边创建正向和反向边

#### 关键代码分析：

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

#### 边属性约定：

| 属性名 | 类型 | 含义 |
|--------|------|------|
| `capacity` | 数值 | 边的剩余容量 |
| `flow` | 数值 | 当前通过边的流量 |

#### 关键不变量：
```
R[u][v]['flow'] == -R[v][u]['flow']
```
*[maxflow.py:114](networkx/algorithms/flow/maxflow.py#L114)*

这个不变量是所有算法操作的基础，确保了正向和反向边的流量始终保持对称。

#### 图级属性：

| 属性名 | 含义 |
|--------|------|
| `R.graph['flow_value']` | 最大流的值 |
| `R.graph['inf']` | 用于模拟无穷大的有限值 |
| `R.graph['algorithm']` | 用于计算最大流的算法名称 |

### 2.4 容量更新协议

所有最大流算法在更新残差网络时都遵循相同的协议。以 Edmonds-Karp 算法的 `augment` 函数为例：

```python
def augment(path):
    """Augment flow along a path from s to t."""
    # 1. 计算路径的残余容量
    flow = inf
    it = iter(path)
    u = next(it)
    for v in it:
        attr = R_succ[u][v]
        flow = min(flow, attr["capacity"] - attr["flow"])
        u = v
    
    # 2. 沿路径增广流量
    it = iter(path)
    u = next(it)
    for v in it:
        R_succ[u][v]["flow"] += flow  # 正向边流量增加
        R_succ[v][u]["flow"] -= flow  # 反向边流量减少（保持不变量）
        u = v
    return flow
```
*[edmondskarp.py:19-38](networkx/algorithms/flow/edmondskarp.py#L19-L38)*

### 2.5 增广路径计算的统一原语

虽然不同算法寻找增广路径的策略不同，但它们都基于相同的图操作原语：

1. **BFS 用于最短路径**（Edmonds-Karp, Dinitz）
2. **DFS 用于阻塞流**（Dinitz）
3. **高度标签用于推流**（Preflow-Push）

### 2.6 割集提取的统一方法

根据最大流最小割定理，最小割可以通过残差网络的可达性分析得到：

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

### 3.1 算法概览

NetworkX 实现了 5 种最大流算法，它们共享相同的残差网络表示，但采用不同的增广策略：

| 算法名称 | 时间复杂度 | 核心思想 | 适用场景 |
|---------|-----------|---------|---------|
| `preflow_push` | O(n²√m) | 最高标签预流推进 | 默认算法，通用场景 |
| `edmonds_karp` | O(nm²) | BFS 最短增广路径 | 教学示例，小规模图 |
| `dinitz` | O(n²m) | 分层图 + 阻塞流 | 通用场景，比 Edmonds-Karp 快 |
| `boykov_kolmogorov` | O(n²m\|C\|) | 双向搜索树 + 增长/增广/采纳 | 计算机视觉，能量最小化 |
| `shortest_augmenting_path` | O(n²m) | 两阶段：DFS + BFS | 可配置，单位容量网络优化 |

### 3.2 共性操作对照

所有算法在残差网络操作层面具有以下共性：

#### 3.2.1 初始化阶段共性

| 操作 | 代码位置 | 说明 |
|------|---------|------|
| 残差网络构建 | `build_residual_network()` | 所有算法调用相同函数 |
| 流量归零 | 各算法 `impl` 函数 | `e["flow"] = 0` |
| 无穷值获取 | `R.graph["inf"]` | 统一使用图级属性 |

**共性代码模式：**
```python
# 所有算法的初始化模式
if residual is None:
    R = build_residual_network(G, capacity)
else:
    R = residual

# 流量归零
for u in R:
    for e in R[u].values():
        e["flow"] = 0
```

#### 3.2.2 流量增广共性

所有算法在增广流量时都遵循相同的协议：

```python
# 正向边增加流量，反向边减少流量（保持不变量）
R_succ[u][v]["flow"] += flow
R_succ[v][u]["flow"] -= flow
```

#### 3.2.3 终止条件共性

| 条件 | 触发情况 |
|------|---------|
| 无增广路径 | 正常终止，已达最大流 |
| 达到 `cutoff` | 提前终止（部分算法支持） |
| 检测到无穷容量路径 | 抛出 `NetworkXUnbounded` |

### 3.3 差异流程深度分析

#### 3.3.1 算法策略架构对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        预流推进类 (Preflow-Push)                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  阶段1: 建立预流 (Push)                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐ │ │
│  │  │ 初始化: 源节点高度=n, 预流推进到邻居                                   │ │ │
│  │  │ 主循环: 选择最高标签的活跃节点                                          │ │ │
│  │  │   - Push: 向高度-1的邻居推流                                          │ │ │
│  │  │   - Relabel: 如果无法推流，增加高度                                    │ │ │
│  │  └─────────────────────────────────────────────────────────────────────┘ │ │
│  │  阶段2: 转换为可行流 (仅 value_only=False)                                │ │
│  │  - 将多余流量从汇节点退回源节点                                            │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        增广路径类 (Augmenting Path)                          │
│                                                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐ │
│  │   Edmonds-Karp      │  │     Dinitz          │  │ Shortest Augmenting │ │
│  │   (BFS增广)          │  │  (分层图+阻塞流)     │  │   Path (两阶段)      │ │
│  ├─────────────────────┤  ├─────────────────────┤  ├─────────────────────┤ │
│  │ while 有增广路径:    │  │ while 有增广路径:   │  │ 阶段1: DFS + 高度标签 │ │
│  │   1. BFS找最短路径   │  │   1. BFS构建分层图  │  │ 阶段2: BFS (复用EK)  │ │
│  │   2. 沿路径增广      │  │   2. DFS找阻塞流    │  │                     │ │
│  │   3. 更新残差网络    │  │   3. 沿阻塞流增广   │  │                     │ │
│  │                     │  │   4. 重复直到无路径  │  │                     │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    双向搜索树类 (Boykov-Kolmogorov)                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  三阶段循环:                                                              │ │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐                              │ │
│  │  │  Growth │───▶│Augment  │───▶│  Adopt  │───▶ (循环)                  │ │
│  │  └─────────┘    └─────────┘    └─────────┘                              │ │
│  │                                                                           │ │
│  │  Growth: 双向BFS扩展搜索树，直到两树相遇                                  │ │
│  │  Augment: 沿相遇路径增广流量，标记饱和边                                  │ │
│  │  Adopt:  修复搜索树，处理成为孤儿的节点                                    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.3.2 详细差异对照表

| 维度 | preflow_push | edmonds_karp | dinitz | boykov_kolmogorov | shortest_augmenting_path |
|------|-------------|--------------|--------|-------------------|--------------------------|
| **搜索策略** | 无路径搜索，基于高度标签推流 | 单向 BFS 找最短路径 | BFS 分层 + DFS 阻塞流 | 双向 BFS 搜索树 | 混合：DFS(阶段1) + BFS(阶段2) |
| **增广单位** | 边级别的推流 (Push) | 完整路径增广 | 阻塞流增广 | 完整路径增广 | 完整路径增广 |
| **特殊数据结构** | 节点高度(`height`)、过剩流量(`excess`)、层次数组(`levels`)、当前边(`curr_edge`)、全局重标记阈值(`grt`) | 无 | 父节点字典(`parents`)、距离字典(`vertex_dist`) | 源树(`source_tree`)、目标树(`target_tree`)、活跃节点(`active`)、孤儿节点(`orphans`)、时间戳(`timestamp`)、距离(`dist`) | 节点高度(`height`)、计数数组(`counts`)、当前边(`curr_edge`) |
| **value_only 优化** | ✓ 支持（跳过阶段2） | ✗ 不支持 | ✗ 不支持 | ✗ 不支持 | ✗ 不支持 |
| **cutoff 支持** | ✗ 不支持 | ✓ 支持 | ✓ 支持 | ✓ 支持 | ✓ 支持 |
| **两阶段设计** | ✓ 预流 → 可行流 | ✗ 单阶段 | ✗ 单阶段 | ✗ 单阶段（但三阶段循环） | ✓ 高度标签DFS → BFS |
| **额外图属性** | `R.graph["trees"]` (Boykov才有) | 无 | 无 | `R.graph["trees"] = (source_tree, target_tree)` | 无 |

#### 3.3.3 关键算法差异深度解析

**1. Preflow-Push 的独特设计**

```python
# 预流推进的核心操作：Push 和 Relabel

def push(u, v, flow):
    """Push flow units of flow from u to v."""
    R_succ[u][v]["flow"] += flow
    R_succ[v][u]["flow"] -= flow
    R_nodes[u]["excess"] -= flow  # 源节点过剩减少
    R_nodes[v]["excess"] += flow  # 目标节点过剩增加

def relabel(u):
    """Relabel a node to create an admissible edge."""
    # 将节点高度设置为：最小邻居高度 + 1
    return min(
        R_nodes[v]["height"]
        for v, attr in R_succ[u].items()
        if attr["flow"] < attr["capacity"]
    ) + 1
```
*[preflowpush.py:90-133](networkx/algorithms/flow/preflowpush.py#L90-L133)*

**关键差异点：**
- 不寻找完整路径，而是直接在边级别推流
- 使用过剩流量(`excess`)概念，允许中间节点暂时流入多于流出
- 两阶段设计：阶段1只求最大预流（`value_only`可提前终止），阶段2才转换为可行流

**2. Boykov-Kolmogorov 的双向搜索树**

```python
# 三阶段循环
while flow_value < cutoff:
    # Growth 阶段：双向扩展搜索树
    u, v = grow()
    if u is None:
        break
    
    # Augmentation 阶段：沿相遇路径增广
    flow_value += augment(u, v)
    
    # Adoption 阶段：修复搜索树
    adopt()
```
*[boykovkolmogorov.py:359-368](networkx/algorithms/flow/boykovkolmogorov.py#L359-L368)*

**关键差异点：**
- 维护两棵搜索树：从源出发的 `source_tree` 和从汇出发的 `target_tree`
- 当两棵树相遇时找到增广路径
- 增广后需要处理孤儿节点（`orphans`）并修复搜索树
- 搜索树可直接用于最小割划分（`R.graph["trees"]`）

**3. Shortest Augmenting Path 的两阶段混合**

```python
# 阶段1: 使用高度标签的 DFS
# 使用与 preflow_push 类似的高度标签和 gap heuristic
while not done:
    # 寻找允许边 (height == neighbor_height + 1)
    if height == R_nodes[v]["height"] + 1 and attr["flow"] < attr["capacity"]:
        path.append(v)
        # ... DFS 前进
    
    # Gap heuristic: 如果某层为空，可提前终止
    if counts[height] == 0:
        R.graph["flow_value"] = flow_value
        return R

# 阶段2: 复用 Edmonds-Karp 的 BFS
flow_value += edmonds_karp_core(R, s, t, cutoff - flow_value)
```
*[shortestaugmentingpath.py:102-160](networkx/algorithms/flow/shortestaugmentingpath.py#L102-L160)*

**关键差异点：**
- 复用了 `preflow_push` 的高度标签和 `CurrentEdge` 数据结构
- 复用了 `edmonds_karp_core` 实现
- 支持 `two_phase` 参数优化单位容量网络

**4. Dinitz 的分层图 + 阻塞流**

```python
def breath_first_search():
    """构建分层图，只保留允许边 (dist[v] = dist[u] + 1)"""
    parents = {}
    vertex_dist = {s: 0}
    while queue:
        u, dist = queue.popleft()
        for v, attr in R_succ[u].items():
            if attr["capacity"] - attr["flow"] > 0:
                # 只记录到最短路径的边
                if vertex_dist[v] == dist + 1:
                    parents[v].append(u)
                # ...
    return parents

def depth_first_search(parents):
    """在分层图中找阻塞流"""
    # DFS 直到无法增广
    while True:
        # 找到路径后增广
        # ...
        R_pred[v][u]["flow"] += flow
        R_pred[u][v]["flow"] -= flow
```
*[dinitz_alg.py:180-234](networkx/algorithms/flow/dinitz_alg.py#L180-L234)*

**关键差异点：**
- BFS 只用于构建分层图，不直接找路径
- DFS 在分层图中找阻塞流（所有最短路径上的最大流）
- 每次增广后需要重新 BFS 构建新的分层图

### 3.4 算法选择决策树

```
                    ┌─────────────────────────────────┐
                    │   选择最大流算法                 │
                    └────────────────┬────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │  需要最小割划分? │    │ 单位容量网络?   │    │   通用场景?     │
    └────────┬────────┘    └────────┬────────┘    └────────┬────────┘
             │                       │                       │
             ▼                       ▼                       ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │boykov_kolmogorov│    │shortest_augmenting│    │  preflow_push   │
    │  (搜索树直接可用) │    │  (two_phase=True) │    │    (默认)       │
    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## 顶层接口分发机制

### 4.1 接口层次结构

NetworkX 的最大流模块采用了清晰的三层接口设计：

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

### 4.2 flow_func 参数的分发机制

高层接口通过 `flow_func` 参数实现算法的动态选择。核心设计在于：

1. **默认算法选择**：
```python
default_flow_func = preflow_push
```
*[maxflow.py:15](networkx/algorithms/flow/maxflow.py#L15)*

2. **参数校验与分发**：
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

### 4.3 算法接口契约

所有最大流算法必须遵循统一的接口契约：

**函数签名：**
```python
def algorithm(G, s, t, capacity="capacity", residual=None, value_only=False, **kwargs):
    """
    Parameters
    ----------
    G : NetworkX graph
        输入图
    s : node
        源节点
    t : node
        汇节点
    capacity : string or function
        容量属性名或容量函数
    residual : NetworkX graph, optional
        可复用的残差网络
    value_only : bool
        如果为 True，仅计算最大流值（某些算法可优化）
    
    Returns
    -------
    R : NetworkX DiGraph
        计算完成后的残差网络
    """
```

**返回值约定：**
- 残差网络 `R` 必须包含 `R.graph['flow_value']`
- 所有边必须满足 `R[u][v]['flow'] == -R[v][u]['flow']`
- 可选地包含 `R.graph['algorithm']` 标识算法名称

### 4.4 value_only 参数的优化

某些算法（如 `preflow_push`）支持 `value_only=True` 优化：

```python
# 阶段1：找到最大预流
# A maximum preflow has been found. The excess at t is the maximum flow value.
if value_only:
    R.graph["flow_value"] = R_nodes[t]["excess"]
    return R

# 阶段2：仅在需要完整流时执行
# Phase 2: Convert the maximum preflow into a maximum flow by returning the
# excess to s.
```
*[preflowpush.py:257-264](networkx/algorithms/flow/preflowpush.py#L257-L264)*

### 4.5 残差网络复用机制

所有算法都支持 `residual` 参数，允许复用已构建的残差网络：

```python
if residual is None:
    R = build_residual_network(G, capacity)
else:
    R = residual
```
*[edmondskarp.py:103-106](networkx/algorithms/flow/edmondskarp.py#L103-L106)*

这种设计在需要多次计算最大流时（如 Gomory-Hu 树算法）特别有用。

---

## 最小费用流与最大流的接口衔接

### 5.1 问题关系与概念澄清

**重要纠正**：最小费用流问题不是最大流问题的简单"推广"，而是两个**相关但不同**的问题：

| 问题类型 | 目标函数 | 约束条件 | 解的性质 |
|---------|---------|---------|---------|
| **最大流** | 最大化 `f`（流量值） | 容量约束 + 流量守恒 | 最大流量可能对应多个流 |
| **最小费用流** | 最小化 `Σ(c_e * f_e)` | 容量约束 + 流量守恒 + **需求约束** | 需求必须精确满足 |
| **最小费用最大流** | 最大化 `f`，然后最小化成本 | 容量约束 + 流量守恒 | 是两阶段问题：先最大流量，再最小成本 |

**关键事实**：`max_flow_min_cost` 不是简单的"桥接"或"转换"，而是一个**两阶段的复合算法**：
1. **第一阶段**：用最大流算法确定最大可能流量 `F`
2. **第二阶段**：在"流量必须等于 `F`"的约束下，求解最小费用流

### 5.2 接口层次

最小费用流模块也采用了类似的分层设计：

```
┌─────────────────────────────────────────────────────────────────┐
│  第一层：高层接口（mincost.py）                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ min_cost_flow_cost()    # 返回最小费用（满足需求）            │ │
│  │ min_cost_flow()         # 返回流字典（满足需求）              │ │
│  │ max_flow_min_cost()     # ⚠️ 两阶段复合：最大流量 + 最小费用  │ │
│  │ cost_of_flow()          # 计算给定流的费用（独立函数）         │ │
│  └─────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  第二层：算法实现层                                                 │
│  - network_simplex()       # 网络单纯形法（核心实现）            │
│  - capacity_scaling()      # 容量缩放法                           │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 max_flow_min_cost 的两阶段设计剖析

**代码原文分析**：

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

**设计意图解析**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  为什么需要两阶段？                                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  问题 1: 单纯的最小费用流（带需求）可以有多个解，流量可能不同                  │
│  问题 2: 用户要的是"在所有达到最大流量的流中，找费用最小的那个"                │
│                                                                              │
│  解决方案：                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  原始图 G（有 capacity 和 weight）                                        ││
│  │      │                                                                    ││
│  │      ▼ 阶段 1: 忽略 weight，只看 capacity                                 ││
│  │  ┌─────────────┐                                                         ││
│  │  │ maximum_flow│ ──▶ 得到最大流量 F                                      ││
│  │  │  (任意算法)  │                                                         ││
│  │  └─────────────┘                                                         ││
│  │      │                                                                    ││
│  │      ▼ 阶段 2: 添加需求约束                                               ││
│  │  ┌───────────────────────────────────────────────────────────────────┐ ││
│  │  │ 新图 H = G + demand(s) = -F + demand(t) = +F                       │ ││
│  │  │                                                                      │ ││
│  │  │  这个需求约束强制：                                                   │ ││
│  │  │  - 从 s 流出的净流量 = F                                             │ ││
│  │  │  - 流入 t 的净流量 = F                                                │ ││
│  │  │  - 因此，任何可行流都必须是"流量为 F 的流"                           │ ││
│  │  └───────────────────────────────────────────────────────────────────┘ ││
│  │      │                                                                    ││
│  │      ▼ 阶段 3: 调用 min_cost_flow                                         ││
│  │  ┌─────────────────┐                                                      ││
│  │  │ min_cost_flow   │ ──▶ 在流量=F 的约束下，最小化 cost                  ││
│  │  │ (network_simplex)│                                                      ││
│  │  └─────────────────┘                                                      ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 两种流算法的核心差异

#### 数据结构差异

| 维度 | 最大流残差网络（utils.py） | 最小费用流内部表示（network_simplex.py） |
|------|---------------------------|------------------------------------------|
| **表示方式** | NetworkX DiGraph 对象 | 纯数组 + 自定义类 `_DataEssentialsAndFunctions` |
| **边属性** | `capacity`, `flow` | `edge_sources[]`, `edge_targets[]`, `edge_capacities[]`, `edge_weights[]`, `edge_flow[]` |
| **节点属性** | 按需添加（如 `height`, `excess`） | `node_demands[]`, `node_potentials[]` |
| **反向边处理** | 显式创建反向边，`flow` 对称 | 隐式处理，通过 `edge_flow` 的符号 |
| **特殊结构** | 无 | 生成树数据结构：`parent[]`, `parent_edge[]`, `subtree_size[]`, DFS 线程 |

#### 算法范式差异

```
最大流算法（增广路径 / 预流推进）：
┌─────────────────────────────────────────────────────────────────┐
│  核心思想：在残差网络上寻找增广路径或推流                        │
│                                                                   │
│  while 可以增广:                                                  │
│      找到增广路径 / 可以推流的边                                  │
│      沿路径 / 边推送流量                                          │
│      更新残差网络（正向+flow，反向-flow）                         │
└─────────────────────────────────────────────────────────────────┘

网络单纯形法（最小费用流）：
┌─────────────────────────────────────────────────────────────────┐
│  核心思想：维护可行生成树，通过转轴操作优化                       │
│                                                                   │
│  1. 初始化：构造初始可行生成树（添加虚节点和虚边）               │
│  2. 迭代：                                                        │
│     a. 找进入边：reduced_cost < 0 的非树边                       │
│     b. 找离开边：沿环增广，第一条饱和的树边                       │
│     c. 转轴：更新生成树结构                                       │
│  3. 终止：所有非树边 reduced_cost ≥ 0                            │
└─────────────────────────────────────────────────────────────────┘
```

### 5.5 重要事实纠正

**之前报告中的不准确表述**：
> "最小费用流问题是最大流问题的推广"

**纠正后的准确理解**：

1. **问题范围不同**：
   - 最大流：单源单汇，目标是流量最大
   - 最小费用流：多源多汇（通过需求指定），目标是成本最小
   - 两者都有容量约束和流量守恒约束

2. **不能互相归约**：
   - 最大流 **不能直接** 用最小费用流算法求解（除非设置适当的权重）
   - 最小费用流 **不能** 用最大流算法求解（缺少成本维度）

3. **`max_flow_min_cost` 是复合问题**：
   - 它不是"用最大流算法解最小费用流"
   - 也不是"用最小费用流算法解最大流"
   - 而是：先用最大流算法确定流量上界，再用最小费用流算法在该约束下优化

### 5.6 cost_of_flow 的独立计算

`cost_of_flow` 函数展示了如何独立于求解算法计算流的费用：

```python
def cost_of_flow(G, flowDict, weight="weight"):
    """Compute the cost of the flow given by flowDict on graph G."""
    return sum((flowDict[u][v] * d.get(weight, 0) 
                for u, v, d in G.edges(data=True)))
```
*[mincost.py:196-252](networkx/algorithms/flow/mincost.py#L196-L252)*

这种解耦设计允许：
1. 验证不同算法的结果
2. 单独分析流的成本构成

---

## 全局最小割的树形分解方法

### 6.1 问题背景

全局最小割（Global Minimum Cut）是将图分成两个非空子集，使得割集的容量最小。

**朴素方法**：对每对节点 `(s, t)` 计算最小 `s-t` 割，需要 `O(n^2)` 次最大流计算。

**Gomory-Hu 树方法**：仅需要 `n-1` 次最大流计算即可得到所有节点对的最小割信息。

### 6.2 Gomory-Hu 树的性质

Gomory-Hu 树 T 是原图 G 的一个树结构，满足：
1. **节点相同**：T 包含 G 的所有节点
2. **等价性**：任意两节点 `u, v` 在 G 中的最小割值等于 T 中 `u-v` 路径上的最小边权
3. **可恢复性**：移除 T 中 `u-v` 路径上的最小权边，得到的两个连通分量就是 G 中 `u, v` 的一个最小割划分

### 6.3 Gusfield 算法实现

NetworkX 实现了 Gusfield 算法（一种不需要节点收缩的 Gomory-Hu 树算法），位于 `gomory_hu.py:16-187`。

#### 算法步骤：

```
1. 初始化：构建星形树，选择一个根节点 root
   tree[n] = root 对所有 n ≠ root

2. 对每个节点 source（n-1 次迭代）：
   a. target = tree[source]
   b. 计算 G 中 source-taget 的最小割
   c. 根据割集更新树结构
   d. 调整相关节点的父节点

3. 构建最终的 Gomory-Hu 树
```

#### 关键代码分析：

```python
def gomory_hu_tree(G, capacity="capacity", flow_func=None):
    # 步骤1：初始化星形树
    tree = {}
    labels = {}
    iter_nodes = iter(G)
    root = next(iter_nodes)
    for n in iter_nodes:
        tree[n] = root  # 所有节点指向根
    
    # 复用残差网络（关键优化）
    R = build_residual_network(G, capacity)
    
    # 步骤2：n-1 次迭代
    for source in tree:
        target = tree[source]
        
        # 调用最大流算法计算最小割
        cut_value, partition = nx.minimum_cut(
            G, source, target, capacity=capacity, 
            flow_func=flow_func, residual=R
        )
        labels[(source, target)] = cut_value
        
        # 更新树结构
        # Source 在 partition[0]，target 在 partition[1]
        for node in partition[0]:
            if node != source and node in tree and tree[node] == target:
                tree[node] = source  # 重新连接父节点
                labels[node, source] = labels.get((node, target), cut_value)
        
        # 处理根节点的特殊情况
        if target != root and tree[target] in partition[0]:
            labels[source, tree[target]] = labels[target, tree[target]]
            labels[target, source] = cut_value
            tree[source] = tree[target]
            tree[target] = source
    
    # 步骤3：构建最终的树
    T = nx.Graph()
    T.add_nodes_from(G)
    T.add_weighted_edges_from(((u, v, labels[u, v]) for u, v in tree.items()))
    return T
```
*[gomory_hu.py:20-187](networkx/algorithms/flow/gomory_hu.py#L20-L187)*

### 6.4 与最大流原语的深度集成

Gomory-Hu 树算法完美展示了如何基于最大流原语构建更高级的算法：

**集成点：**

1. **复用残差网络构建**：
```python
R = build_residual_network(G, capacity)
```
*[gomory_hu.py:159](networkx/algorithms/flow/gomory_hu.py#L159)*

2. **调用最小割接口**：
```python
cut_value, partition = nx.minimum_cut(
    G, source, target, capacity=capacity, 
    flow_func=flow_func, residual=R
)
```
*[gomory_hu.py:166-168](networkx/algorithms/flow/gomory_hu.py#L166-L168)*

3. **可插拔的算法选择**：
```python
if flow_func is None:
    flow_func = default_flow_func  # edmonds_karp
```
*[gomory_hu.py:143-144](networkx/algorithms/flow/gomory_hu.py#L143-L144)*

### 6.5 使用示例

```python
# 构建 Gomory-Hu 树
G = nx.karate_club_graph()
nx.set_edge_attributes(G, 1, "capacity")
T = nx.gomory_hu_tree(G)

# 查询任意节点对的最小割值
def minimum_edge_weight_in_shortest_path(T, u, v):
    path = nx.shortest_path(T, u, v, weight="weight")
    return min((T[u][v]["weight"], (u, v)) 
               for (u, v) in zip(path, path[1:]))

u, v = 0, 33
cut_value, edge = minimum_edge_weight_in_shortest_path(T, u, v)
# cut_value == nx.minimum_cut_value(G, u, v)
```
*[gomory_hu.py:81-96](networkx/algorithms/flow/gomory_hu.py#L81-L96)*

---

## 设计取舍与边界分析

### 7.1 算法设计取舍矩阵

| 设计维度 | 取舍选项 | NetworkX 的选择 | 理由与代价 |
|---------|---------|----------------|-----------|
| **残差网络表示** | NetworkX 图 vs 纯数组 | NetworkX DiGraph | 优点：可复用 NetworkX 的图操作；缺点：性能开销 |
| **算法默认选择** | 理论最优 vs 实践最优 | `preflow_push` (默认) | 理论复杂度 O(n²√m) 最优，但实际性能依赖图特征 |
| **无穷值处理** | 真正的无穷 vs 模拟值 | `3 * sum(有限容量)` | 优点：避免数值问题；缺点：需要无界检测 |
| **多图支持** | 支持 vs 不支持 | 不支持 MultiGraph | 简化实现，主流场景用不到 |
| **value_only 优化** | 所有算法支持 vs 部分支持 | 仅 `preflow_push` 支持 | 预流推进的两阶段设计天然支持 |
| **Gomory-Hu 默认算法** | 最快 vs 最稳 | `edmonds_karp` | 与 `preflow_push` 的 `value_only` 不兼容（见 7.2.3） |

### 7.2 边界条件深度分析

#### 7.2.1 输入验证边界

所有算法在入口处进行统一的输入验证：

```python
# 边界 1: 源/汇节点不存在
if s not in G:
    raise nx.NetworkXError(f"node {str(s)} not in graph")
if t not in G:
    raise nx.NetworkXError(f"node {str(t)} not in graph")

# 边界 2: 源汇相同
if s == t:
    raise nx.NetworkXError("source and sink are the same node")

# 边界 3: 多图类型
if G.is_multigraph():
    raise nx.NetworkXError("MultiGraph and MultiDiGraph not supported (yet).")
```
*[各算法 impl 函数]*

**设计分析**：
- `s == t` 是一个重要边界：数学上最大流为 0，但实现中选择抛出异常，强制用户处理
- 多图不支持是一个有意识的简化：如果有平行边，用户应先合并（容量相加）

#### 7.2.2 无界流检测（Unbounded Flow）

**问题场景**：存在从源到汇的路径，其中所有边的容量都是无穷大。

**检测策略**：

```python
# 策略 1: 预处理时 BFS 检测（preflow_push）
def detect_unboundedness(R, s, t):
    """Detect an infinite-capacity s-t path in R."""
    q = deque([s])
    seen = {s}
    inf = R.graph["inf"]
    while q:
        u = q.popleft()
        for v, attr in R[u].items():
            if attr["capacity"] == inf and v not in seen:
                if v == t:
                    raise nx.NetworkXUnbounded(
                        "Infinite capacity path, flow unbounded above."
                    )
                seen.add(v)
                q.append(v)
```
*[utils.py:164-178](networkx/algorithms/flow/utils.py#L164-L178)*

```python
# 策略 2: 增广后检测（edmonds_karp, dinitz 等）
if flow * 2 > inf:
    raise nx.NetworkXUnbounded("Infinite capacity path, flow unbounded above.")
```
*[edmondskarp.py:29-30](networkx/algorithms/flow/edmondskarp.py#L29-L30)*

**设计分析**：
- 为什么用 `flow * 2 > inf`？因为 `inf = 3 * sum(有限容量)`
- 如果单次增广的流量 `> inf/2`，说明用到了容量为 `inf` 的边
- 但这只是一个启发式检测，可能有漏检

**无穷值模拟的设计取舍**：

```
┌─────────────────────────────────────────────────────────────────┐
│  为什么不直接用 float('inf')？                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
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
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 7.2.3 Gomory-Hu 树与 preflow_push 的不兼容

**重要发现**：`gomory_hu_tree` 的默认算法是 `edmonds_karp`，而不是全局默认的 `preflow_push`！

```python
# gomory_hu.py 中的默认选择
default_flow_func = edmonds_karp  # ⚠️ 不是 preflow_push！
```
*[gomory_hu.py:11](networkx/algorithms/flow/gomory_hu.py#L11)*

**原因分析**：

```python
# 在 minimum_cut 中调用 flow_func 时：
R = flow_func(flowG, _s, _t, capacity=capacity, value_only=True, **kwargs)
```
*[maxflow.py:473](networkx/algorithms/flow/maxflow.py#L473)*

对于 `preflow_push`，`value_only=True` 意味着：
- 只执行阶段 1（预流）
- 不执行阶段 2（转换为可行流）

**但问题在于**：Gomory-Hu 树算法不仅需要最小割的**值**，还需要最小割的**划分**（`partition`）。

```python
# minimum_cut 中获取划分的代码
# 移除饱和边
cutset = [(u, v, d) for u, v, d in R.edges(data=True) 
          if d["flow"] == d["capacity"]]
R.remove_edges_from(cutset)

# 可达性分析
non_reachable = set(nx.shortest_path_length(R, target=_t))
partition = (set(flowG) - non_reachable, non_reachable)
```
*[maxflow.py:475-482](networkx/algorithms/flow/maxflow.py#L475-L482)*

**关键问题**：对于 `preflow_push` + `value_only=True`，残差网络中的 `flow` 是**预流**，不是**可行流**。这可能影响可达性分析的正确性！

**设计取舍的证据**：

```python
# maxflow.py 中的特殊检查
if kwargs.get("cutoff") is not None and flow_func is preflow_push:
    raise nx.NetworkXError("cutoff should not be specified.")
```
*[maxflow.py:470-471](networkx/algorithms/flow/maxflow.py#L470-L471)*

这说明开发者已经意识到 `preflow_push` 与某些参数组合可能有问题。

#### 7.2.4 最小费用流的边界情况

最小费用流有更多边界情况需要处理：

| 边界情况 | 检测方式 | 异常类型 |
|---------|---------|---------|
| 需求总和不为 0 | `sum(DEAF.node_demands) != 0` | `NetworkXUnfeasible` |
| 边容量为负 | `c < 0` | `NetworkXUnfeasible` |
| 无解（无法满足需求） | 虚边有流量残留 | `NetworkXUnfeasible` |
| 无界（负费用无穷环） | 流量 ≥ `faux_inf/2` | `NetworkXUnbounded` |
| 无穷需求/权重 | 遍历检查 | `NetworkXError` |

**虚边检测机制**：

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

**设计分析**：
- 虚边的容量和权重都设为 `faux_inf`（一个很大的值）
- 如果最优解中虚边有流量，说明原问题无法满足需求
- 这是网络单纯形法中标准的大 M 法（Big-M Method）

### 7.3 性能优化策略对照

| 优化策略 | 应用算法 | 实现位置 | 效果 |
|---------|---------|---------|------|
| **Gap Heuristic** | preflow_push, shortest_augmenting_path | 检测某层为空时提前终止 | 显著减少不必要的重标记 |
| **Global Relabeling** | preflow_push | 周期性反向 BFS 更新高度 | 保持高度标签的准确性，加速推流 |
| **双向 BFS** | edmonds_karp, boykov_kolmogorov | 从源和汇同时搜索 | 减少搜索空间 |
| **Current Edge** | preflow_push, shortest_augmenting_path | 循环迭代边时不从头开始 | 避免重复检查已饱和的边 |
| **残差网络复用** | 所有算法 + gomory_hu | `residual` 参数 | 避免重复构建残差网络 |
| **value_only** | preflow_push | 跳过阶段 2 | 只需最大流值时节省时间 |
| **搜索树复用** | boykov_kolmogorov | Growth/Adopt 阶段 | 不需要每次重新 BFS |

### 7.4 算法适用场景决策表

| 场景特征 | 推荐算法 | 理由 |
|---------|---------|------|
| 通用场景，无特殊需求 | `preflow_push`（默认） | 理论复杂度最优，实践中通常最快 |
| 需要最小割划分，且想复用搜索树 | `boykov_kolmogorov` | `R.graph["trees"]` 直接可用 |
| 单位容量网络 | `shortest_augmenting_path(two_phase=True)` | 时间复杂度从 O(n²m) 降到 O(min(n^(2/3), m^(1/2))m) |
| 稠密图，边数远大于节点数 | `dinitz` 或 `shortest_augmenting_path` | 分层图 + 阻塞流在稠密图中效率高 |
| 稀疏图，计算机视觉/能量最小化 | `boykov_kolmogorov` | 实践中对这类问题特别有效 |
| 教学演示，理解增广路径 | `edmonds_karp` | 实现最简单，概念最清晰 |
| 只需最大流值，不需要流分布 | `preflow_push(value_only=True)` | 跳过阶段 2，节省时间 |
| Gomory-Hu 树计算（默认） | `edmonds_karp` | 与 `value_only` 兼容性好 |

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
- 默认算法（`preflow_push`）可根据版本更新

#### 3. 层次化接口设计
- 高层接口（`maximum_flow`, `minimum_cut`）提供易用性
- 算法层（`preflow_push`, `edmonds_karp`）提供灵活性
- 工具层（`build_residual_network`）提供可复用组件

#### 4. 算法复用与组合
- `max_flow_min_cost` 是**两阶段复合算法**（不是简单桥接）
- `gomory_hu_tree` 基于 `n-1` 次最大流计算构建全局结构
- 残差网络复用机制减少重复计算

#### 5. 渐进式优化
- `value_only` 参数允许仅计算流量值
- `residual` 参数支持增量计算
- 不同算法针对不同图特征优化

### 8.2 重要纠正与补充

#### 关于最小费用流的纠正
1. **不是简单推广**：最小费用流和最大流是相关但不同的问题，不能互相直接归约
2. **`max_flow_min_cost` 是两阶段设计**：
   - 阶段 1：用最大流算法确定最大流量 `F`
   - 阶段 2：通过设置需求约束，强制最小费用流算法求解流量恰好为 `F` 的最小费用流
3. **数据结构完全不同**：最大流使用 NetworkX DiGraph，最小费用流使用纯数组 + 生成树结构

#### 关于 Gomory-Hu 树的补充
1. **默认算法是 `edmonds_karp`**，不是全局默认的 `preflow_push`
2. **可能的原因**：`preflow_push` + `value_only=True` 只计算预流，可能影响最小割划分的正确性
3. **这是一个有意识的设计取舍**，在代码中有明确体现

#### 关于边界处理的补充
- **无界流检测**：使用 `inf = 3 * sum(有限容量)` 的启发式，通过 `flow * 2 > inf` 检测
- **最小费用流无解检测**：通过虚边流量残留判断
- **输入验证**：所有算法统一检查 `s not in G`, `t not in G`, `s == t`, `MultiGraph` 等

### 8.3 关键模块关系图（更新版）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                    应用层                                      │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌───────────────────┐ │
│  │    最大流/最小割       │  │     最小费用流        │  │ Gomory-Hu 全局最小割│ │
│  │    (maxflow.py)       │  │    (mincost.py)      │  │  (gomory_hu.py)    │ │
│  ├──────────────────────┤  ├──────────────────────┤  ├───────────────────┤ │
│  │ maximum_flow()        │  │ min_cost_flow()      │  │ ⚠️ 默认: edmonds_  │ │
│  │ maximum_flow_value()  │  │ min_cost_flow_cost() │  │    karp (不是默认)  │ │
│  │ minimum_cut()         │  │ ⚠️ max_flow_min_cost()│  │                   │ │
│  │ minimum_cut_value()   │  │    (两阶段复合)       │  │                   │ │
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
│  │  │ (默认算法)   │ │  (BFS增广)   │ │ (分层图)    │ │  (双向搜索树)      ││ │
│  │  │             │ │             │ │             │ │                   ││ │
│  │  │ ✓ value_only│ │ ✗ value_only│ │ ✗ value_only│ │ ✗ value_only     ││ │
│  │  │ 两阶段设计   │ │             │ │             │ │ 三阶段循环        ││ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └───────────────────┘│ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐│ │
│  │  │              shortest_augmenting_path                                ││ │
│  │  │     (混合: DFS高度标签 + BFS，复用 EK 核心)                          ││ │
│  │  └─────────────────────────────────────────────────────────────────────┘│ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                           │
│                                    ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │         最小费用流算法（独立数据结构）                                      │ │
│  │  ┌─────────────────────┐  ┌─────────────────────┐                      │ │
│  │  │   network_simplex   │  │  capacity_scaling   │                      │ │
│  │  │   (网络单纯形法)     │  │    (容量缩放法)      │                      │ │
│  │  │                     │  │                     │                      │ │
│  │  │ - 数组表示           │  │                     │                      │ │
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
| **适配器**（纠正） | ~~`max_flow_min_cost` 转换问题类型~~ → **复合算法** | mincost.py:256-356 |
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

5. **修复/澄清 `preflow_push` 与 Gomory-Hu 的兼容性**：
   - 调研 `preflow_push(value_only=True)` 的残差网络是否真的会影响最小割划分
   - 如果不影响，可以将 Gomory-Hu 的默认算法改为 `preflow_push` 以获得更好性能
   - 如果影响，应该在文档中明确说明这个设计取舍

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

**v2.0 (本次更新)**：
1. ✅ 新增 **第3章 最大流算法族横向对照**，包含 5 种算法的详细对比
2. ✅ 纠正 **第5章 最小费用流与最大流的接口衔接** 中的事实偏差：
   - 澄清 `max_flow_min_cost` 是**两阶段复合算法**，不是简单桥接
   - 详细分析了最大流与最小费用流在数据结构和算法范式上的本质差异
3. ✅ 新增 **第7章 设计取舍与边界分析**：
   - 算法设计取舍矩阵
   - 输入验证、无界流检测等边界条件深度分析
   - **重要发现**：Gomory-Hu 树默认算法是 `edmonds_karp` 而非 `preflow_push` 的原因
   - 最小费用流的虚边检测机制
   - 性能优化策略对照
   - 算法适用场景决策表
4. ✅ 更新 **第8章 总结** 中的模块关系图和设计模式应用

**v1.0 (初始版本)**：
- 残差网络统一表示与操作协议
- 顶层接口分发机制
- 最小费用流与最大流的接口衔接（初稿）
- 全局最小割的树形分解方法
