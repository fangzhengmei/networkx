# NetworkX 最大流算法族分析报告

## 1. 最大流计算入口的多实现调度机制

### 1.1 入口函数与调度架构

NetworkX 的最大流计算采用了**策略模式（Strategy Pattern）**设计，通过 `flow_func` 参数将调用路由到不同的算法实现。

#### 核心入口文件
- `networkx/algorithms/flow/maxflow.py` - 统一入口
- `networkx/algorithms/flow/__init__.py` - 模块导出

#### 调度机制代码分析

**默认算法选择**（`maxflow.py:15`）：
```python
default_flow_func = preflow_push
```

**入口函数 `maximum_flow`**（`maxflow.py:23-177`）：
```python
def maximum_flow(flowG, _s, _t, capacity="capacity", flow_func=None, **kwargs):
    if flow_func is None:
        if kwargs:
            raise nx.NetworkXError(
                "You have to explicitly set a flow_func if"
                " you need to pass parameters via kwargs."
            )
        flow_func = default_flow_func  # 默认使用 preflow_push

    if not callable(flow_func):
        raise nx.NetworkXError("flow_func has to be callable.")

    R = flow_func(flowG, _s, _t, capacity=capacity, value_only=False, **kwargs)
    flow_dict = build_flow_dict(flowG, R)
    return (R.graph["flow_value"], flow_dict)
```

### 1.2 可用算法实现

| 算法 | 实现文件 | 时间复杂度 | 适用场景 |
|------|----------|------------|----------|
| **Preflow-Push (最高标签)** | `preflowpush.py` | $O(n^2 \sqrt{m})$ | 通用场景，默认选择 |
| **Edmonds-Karp** | `edmondskarp.py` | $O(n m^2)$ | 稀疏网络，高度偏斜度分布 |
| **Shortest Augmenting Path** | `shortestaugmentingpath.py` | $O(n^2 m)$ | 稠密网络，单位容量网络 |
| **Dinitz** | `dinitz_alg.py` | $O(n^2 m)$ | 通用场景，分层图优化 |
| **Boykov-Kolmogorov** | `boykovkolmogorov.py` | $O(n^2 m |C|)$ | 计算机视觉、图像分割 |

### 1.3 默认选择依据

选择 `preflow_push` 作为默认算法的原因：
1. **理论复杂度最优**：$O(n^2 \sqrt{m})$ 优于其他路径增广算法
2. **全局重标记启发式**（`global_relabel_freq=1`）：加速实际运行
3. **间隙启发式**（Gap Heuristic）：提前识别最小割
4. **两阶段设计**：Phase 1 求最大预流（足够求最小割），Phase 2 转换为最大流

### 1.4 算法接口契约

所有算法必须实现的统一接口：
```python
def algorithm(G, s, t, capacity="capacity", residual=None, value_only=False, **kwargs):
    """
    参数：
        G: 输入图
        s: 源点
        t: 汇点
        capacity: 容量属性名或函数
        residual: 可选的预构建残差网络
        value_only: 是否只计算流值（不构建完整流）
    返回：
        R: 残差网络 DiGraph，包含：
            - R.graph['flow_value']: 最大流值
            - R.graph['algorithm']: 算法标识
            - 各边属性: 'capacity', 'flow'
    """
```

---

## 2. 残差图基础设施

### 2.1 残差网络构建

残差网络是所有最大流算法的共同数据结构基础，由 `build_residual_network` 函数统一构建。

**核心实现**（`utils.py:83-156`）：

```python
def build_residual_network(G, capacity):
    """Build a residual network and initialize a zero flow."""
    if G.is_multigraph():
        raise nx.NetworkXError("MultiGraph and MultiDiGraph not supported (yet).")

    R = nx.DiGraph()
    R.__networkx_cache__ = None  # 禁用缓存以提高性能
    R.add_nodes_from(G)

    capacity = _capacity_function(G, capacity)
    
    # 提取正容量边（排除自环）
    edge_list = [
        (u, v, cap)
        for u, v, attr in G.edges(data=True)
        if u != v and (cap := capacity(u, v, attr)) is not None and cap > 0
    ]

    # 模拟无穷大：3倍有限容量之和
    inf = float("inf")
    inf = 3 * sum(cap for u, v, cap in edge_list if cap != inf) or 1

    if G.is_directed():
        for u, v, cap in edge_list:
            r = min(cap, inf)
            if not R.has_edge(u, v):
                # 成对添加正向和反向边
                R.add_edge(u, v, capacity=r)  # 正向边容量 = 原容量
                R.add_edge(v, u, capacity=0)  # 反向边容量 = 0
            else:
                R[u][v]["capacity"] = r
    else:
        # 无向图：两边容量相等
        for u, v, cap in edge_list:
            r = min(cap, inf)
            R.add_edge(u, v, capacity=r)
            R.add_edge(v, u, capacity=r)

    R.graph["inf"] = inf  # 记录模拟无穷大的值
    return R
```

### 2.2 残差图数据结构规范

```
残差网络 R (DiGraph):
├── 节点: 与原图相同
├── 边: 成对出现 (u,v) 和 (v,u)
│   ├── 属性:
│   │   ├── 'capacity': 残余容量
│   │   └── 'flow': 当前流量（满足 R[u][v]['flow'] == -R[v][u]['flow']）
├── 图属性:
│   ├── 'flow_value': 最大流值
│   ├── 'inf': 模拟无穷大的值
│   └── 'algorithm': 使用的算法标识
```

### 2.3 边容量更新机制

所有算法在增广时遵循相同的**正向/反向边对称更新**模式。

**典型实现（以 Edmonds-Karp 为例）**（`edmondskarp.py:19-38`）：

```python
def augment(path):
    """Augment flow along a path from s to t."""
    # 1. 确定路径残余容量
    flow = inf
    it = iter(path)
    u = next(it)
    for v in it:
        attr = R_succ[u][v]
        flow = min(flow, attr["capacity"] - attr["flow"])  # 残余容量 = 容量 - 流量
        u = v
    
    # 2. 沿路径增广
    it = iter(path)
    u = next(it)
    for v in it:
        R_succ[u][v]["flow"] += flow  # 正向边：增加流量
        R_succ[v][u]["flow"] -= flow  # 反向边：减少流量（等价于增加反向残余容量）
        u = v
    return flow
```

**Preflow-Push 的 push 操作**（`preflowpush.py:90-95`）：

```python
def push(u, v, flow):
    """Push flow units of flow from u to v."""
    R_succ[u][v]["flow"] += flow
    R_succ[v][u]["flow"] -= flow
    R_nodes[u]["excess"] -= flow  # 更新节点超额流
    R_nodes[v]["excess"] += flow
```

### 2.4 容量函数抽象

`_capacity_function` 提供了灵活的容量指定方式（`utils.py:193-231`）：

```python
def _capacity_function(G, capacity):
    """Returns a callable that returns the capacity of an edge."""
    inf = float("inf")
    if callable(capacity):
        return capacity  # 用户自定义容量函数
    # 字符串形式：作为边属性键
    return lambda u, v, data: data.get(capacity, inf)  # 无属性时视为无穷大
```

---

## 3. 算法实现策略对比

### 3.1 路径增广算法族（Ford-Fulkerson 框架）

#### Edmonds-Karp 算法

**核心特点**：
- 使用**双向 BFS** 寻找最短增广路径
- 时间复杂度：$O(n m^2)$
- 路径长度按边数计算（无权最短路径）

**双向 BFS 实现**（`edmondskarp.py:40-69`）：
```python
def bidirectional_bfs():
    """Bidirectional breadth-first search for an augmenting path."""
    pred = {s: None}  # 源点搜索树
    q_s = [s]
    succ = {t: None}  # 汇点搜索树
    q_t = [t]
    while True:
        q = []
        # 选择较小的队列扩展
        if len(q_s) <= len(q_t):
            for u in q_s:
                for v, attr in R_succ[u].items():
                    if v not in pred and attr["flow"] < attr["capacity"]:
                        pred[v] = u
                        if v in succ:  # 相遇
                            return v, pred, succ
                        q.append(v)
            # ...
        else:
            # 从汇点方向扩展（使用 R_pred）
            for u in q_t:
                for v, attr in R_pred[u].items():
                    if v not in succ and attr["flow"] < attr["capacity"]:
                        succ[v] = u
                        if v in pred:
                            return v, pred, succ
                        q.append(v)
```

#### Shortest Augmenting Path 算法

**核心特点**：
- 使用**距离标签**（height）保证每次增广都是最短路径
- 支持**两阶段优化**：Phase 1 用 DFS，Phase 2 切换到 Edmonds-Karp
- 间隙启发式（Gap Heuristic）：当某层无节点时提前终止

**距离标签与间隙启发式**（`shortestaugmentingpath.py:63-66, 123-129`）：
```python
# 初始化各层节点计数
counts = [0] * (2 * n - 1)
for u in R:
    counts[R_nodes[u]["height"]] += 1

# 间隙启发式检查
counts[height] -= 1
if counts[height] == 0:
    # 该层为空，最小割已识别
    R.graph["flow_value"] = flow_value
    return R
```

#### Dinitz 算法

**核心特点**：
- **分层图（Level Graph）**：BFS 构建，只保留从源到汇的最短路径边
- **阻塞流（Blocking Flow）**：在分层图中用 DFS 找到所有增广路径
- 时间复杂度：$O(n^2 m)$，单位容量网络 $O(m \sqrt{n})$

**分层图 + 阻塞流实现**（`dinitz_alg.py:180-234`）：
```python
def breath_first_search():
    """构建分层图：只保留距离 = 当前距离 + 1 的边"""
    parents = {}
    vertex_dist = {s: 0}
    queue = deque([(s, 0)])
    while queue:
        if t in parents:
            break
        u, dist = queue.popleft()
        for v, attr in R_succ[u].items():
            if attr["capacity"] - attr["flow"] > 0:
                if v in parents:
                    if vertex_dist[v] == dist + 1:
                        parents[v].append(u)  # 同层多条路径
                else:
                    parents[v] = deque([u])
                    vertex_dist[v] = dist + 1
                    queue.append((v, dist + 1))
    return parents

def depth_first_search(parents):
    """在分层图中寻找所有阻塞流"""
    total_flow = 0
    # 使用 DFS 回溯找到所有增广路径
    # 当边饱和时从 parents 中移除
```

### 3.2 Boykov-Kolmogorov 算法（双搜索树策略）

**本质差异**：
- 不是传统的 Ford-Fulkerson 框架
- 使用**双搜索树**（源树 + 汇树）并行生长
- 三阶段循环：Growth → Augmentation → Adoption

**核心数据结构**（`boykovkolmogorov.py:350-358`）：
```python
source_tree = {s: None}  # 源点搜索树：节点 → 父节点
target_tree = {t: None}  # 汇点搜索树：节点 → 父节点
active = deque([s, t])   # 活跃节点队列
orphans = deque()        # 孤儿节点（增广后父边饱和）

# 标记启发式数据结构
time = 1
timestamp = {s: time, t: time}  # 节点时间戳
dist = {s: 0, t: 0}             # 节点距离
```

#### 三阶段循环

**1. Growth 阶段**（`boykovkolmogorov.py:203-236`）：
```python
def grow():
    """双向 BFS 生长搜索树，直到找到连接边"""
    while active:
        u = active[0]
        if u in source_tree:
            this_tree = source_tree
            other_tree = target_tree
            neighbors = R_succ  # 源树：正向扩展
        else:
            this_tree = target_tree
            other_tree = source_tree
            neighbors = R_pred  # 汇树：反向扩展
        
        for v, attr in neighbors[u].items():
            if attr["capacity"] - attr["flow"] > 0:
                if v not in this_tree:
                    if v in other_tree:
                        # 找到连接边！
                        return (u, v) if this_tree is source_tree else (v, u)
                    this_tree[v] = u
                    dist[v] = dist[u] + 1
                    timestamp[v] = timestamp[u]
                    active.append(v)
                elif v in this_tree and _is_closer(u, v):
                    # 标记启发式：更新更短路径
                    this_tree[v] = u
                    dist[v] = dist[u] + 1
                    timestamp[v] = timestamp[u]
        _ = active.popleft()
    return None, None
```

**2. Augmentation 阶段**（`boykovkolmogorov.py:238-284`）：
```python
def augment(u, v):
    """沿连接边重构路径并增广"""
    attr = R_succ[u][v]
    flow = min(INF, attr["capacity"] - attr["flow"])
    path = [u]
    
    # 从 u 回溯到 s（源树）
    w = u
    while w != s:
        n = w
        w = source_tree[n]
        attr = R_pred[n][w]
        flow = min(flow, attr["capacity"] - attr["flow"])
        path.append(w)
    path.reverse()
    
    # 从 v 到 t（汇树）
    path.append(v)
    w = v
    while w != t:
        n = w
        w = target_tree[n]
        attr = R_succ[n][w]
        flow = min(flow, attr["capacity"] - attr["flow"])
        path.append(w)
    
    # 增广并标记孤儿
    these_orphans = []
    for u, v in zip(path, path[1:]):
        R_succ[u][v]["flow"] += flow
        R_succ[v][u]["flow"] -= flow
        if R_succ[u][v]["flow"] == R_succ[u][v]["capacity"]:
            # 边饱和，产生孤儿
            if v in source_tree:
                source_tree[v] = None
                these_orphans.append(v)
            if u in target_tree:
                target_tree[u] = None
                these_orphans.append(u)
    orphans.extend(sorted(these_orphans, key=dist.get))
    return flow
```

**3. Adoption 阶段**（`boykovkolmogorov.py:286-326`）：
```python
def adopt():
    """为孤儿节点寻找新父节点或从树中移除"""
    while orphans:
        u = orphans.popleft()
        if u in source_tree:
            tree = source_tree
            neighbors = R_pred
        else:
            tree = target_tree
            neighbors = R_succ
        
        # 尝试为 u 找新父节点
        nbrs = ((n, attr, dist[n]) for n, attr in neighbors[u].items() if n in tree)
        for v, attr, d in sorted(nbrs, key=itemgetter(2)):
            if attr["capacity"] - attr["flow"] > 0:
                if _has_valid_root(v, tree):
                    tree[u] = v
                    dist[u] = dist[v] + 1
                    timestamp[u] = time
                    break
        else:
            # 无法找到有效父节点，从树中移除
            for v, attr, d in nbrs:
                if attr["capacity"] - attr["flow"] > 0:
                    if v not in active:
                        active.append(v)
                if tree[v] == u:
                    tree[v] = None
                    orphans.appendleft(v)
            if u in active:
                active.remove(u)
            del tree[u]
```

### 3.3 算法策略对比总结

| 特性 | 路径增广算法 | Boykov-Kolmogorov |
|------|-------------|-------------------|
| **框架** | Ford-Fulkerson | 双搜索树 |
| **搜索策略** | 单源单汇路径搜索 | 双向并行搜索树生长 |
| **增广单位** | 完整 s-t 路径 | 连接边处汇合 |
| **路径重构** | 每次增广后重新搜索 | 搜索树复用 + Adoption |
| **时间复杂度** | $O(n^2 m)$ 或 $O(n m^2)$ | $O(n^2 m |C|)$（最坏） |
| **实际性能** | 取决于网络结构 | 图像分割等场景极快 |
| **最小割提取** | 需 BFS 可达性 | 直接从搜索树获得 |

---

## 4. 最大流-最小割对偶定理实现

### 4.1 理论基础

**最大流-最小割定理**：
> 在任何网络中，从源点 s 到汇点 t 的最大流值等于分离 s 和 t 的最小割的容量。

**割的定义**：
- 节点划分为两个集合 $S$ 和 $T$，其中 $s \in S, t \in T$
- 割容量：$\sum_{u \in S, v \in T} c(u, v)$
- 最小割：容量最小的 s-t 割

### 4.2 残差图中的最小割提取

**核心原理**：
算法终止时，残差图中**从源点不可达汇点**（无增广路径）。此时：
- $S$ = 源点在残差图中可达的节点集合
- $T$ = 不可达的节点集合
- $(S, T)$ 即为最小割

**`minimum_cut` 实现**（`maxflow.py:320-486`）：

```python
def minimum_cut(flowG, _s, _t, capacity="capacity", flow_func=None, **kwargs):
    """Compute the value and the node partition of a minimum (s, t)-cut."""
    if flow_func is None:
        flow_func = default_flow_func  # preflow_push

    # 1. 计算最大流（value_only=True 可能提前终止）
    R = flow_func(flowG, _s, _t, capacity=capacity, value_only=True, **kwargs)
    
    # 2. 移除饱和边（flow == capacity）
    cutset = [(u, v, d) for u, v, d in R.edges(data=True) 
              if d["flow"] == d["capacity"]]
    R.remove_edges_from(cutset)
    
    # 3. 从汇点反向 BFS 找不可达节点
    # non_reachable = 从 s 到 t 没有路径的节点
    non_reachable = set(nx.shortest_path_length(R, target=_t))
    partition = (set(flowG) - non_reachable, non_reachable)
    
    # 4. 恢复移除的边
    R.add_edges_from(cutset)
    
    return (R.graph["flow_value"], partition)
```

### 4.3 Boykov-Kolmogorov 的特殊优化

BK 算法的双搜索树结构允许**直接提取最小割**，无需额外 BFS：

**搜索树即最小割分区**（`boykovkolmogorov.py:141-151` 文档说明）：
```python
# 算法结束后，搜索树直接定义了最小割
source_tree, target_tree = R.graph["trees"]
partition = (set(source_tree), set(G) - set(source_tree))
# 或等价地
partition = (set(G) - set(target_tree), set(target_tree))
```

**原理**：
- Growth 阶段无法找到连接边时，算法终止
- `source_tree` 包含所有从 s 通过残余边可达的节点
- `target_tree` 包含所有可到达 t 的节点
- 两树之间的边界即为最小割

### 4.4 节点割与边割的转换

`connectivity/cuts.py` 中实现了基于最大流的节点/边割计算：

**边割转换**（`cuts.py:30-159`）：
```python
def minimum_st_edge_cut(G, s, t, flow_func=None, auxiliary=None, residual=None):
    # 构建辅助图：每条边容量为 1
    if auxiliary is None:
        H = build_auxiliary_edge_connectivity(G)
    
    # 最小割的容量即为最少需要移除的边数
    cut_value, partition = nx.minimum_cut(H, s, t, **kwargs)
    reachable, non_reachable = partition
    
    # 提取跨分区边
    cutset = set()
    for u, nbrs in ((n, G[n]) for n in reachable):
        cutset.update((u, v) for v in nbrs if v in non_reachable)
    return cutset
```

**节点割转换**（`cuts.py:167-306`）：
```python
def minimum_st_node_cut(G, s, t, flow_func=None, auxiliary=None, residual=None):
    # 节点分裂技术：每个节点 v 拆分为 vA → vB
    # 边 (vA, vB) 容量为 1（移除该边等价于移除节点 v）
    # 原图边 u→v 变为 uB → vA，容量无穷大
    if auxiliary is None:
        H = build_auxiliary_node_connectivity(G)
    
    mapping = H.graph["mapping"]
    # 源点用 B 节点，汇点用 A 节点
    edge_cut = minimum_st_edge_cut(H, f"{mapping[s]}B", f"{mapping[t]}A", **kwargs)
    # 转换回原图节点
    node_cut = {H.nodes[node]["id"] for edge in edge_cut for node in edge}
    return node_cut - {s, t}
```

---

## 5. 多源多汇场景的统一处理机制

### 5.1 问题描述

多源多汇最大流问题：
- 多个源点 $S = \{s_1, s_2, ..., s_k\}$
- 多个汇点 $T = \{t_1, t_2, ..., t_m\}$
- 求从所有源点到所有汇点的最大总流量

### 5.2 标准归约方法

**引入超级源点和超级汇点**：

1. **添加超级源点 $S^*$**：
   - 添加边 $S^* \to s_i$，容量 = $\infty$（或源点的最大供应量）
   
2. **添加超级汇点 $T^*$**：
   - 添加边 $t_j \to T^*$，容量 = $\infty$（或汇点的最大需求量）

3. **问题转换**：
   - 原问题等价于求 $S^*$ 到 $T^*$ 的单源单汇最大流

### 5.3 NetworkX 中的实现模式

虽然 NetworkX 的 `maximum_flow` 函数本身只接受单源单汇，但这种归约模式在多个模块中被广泛使用：

#### 示例：连通性计算中的超级源/汇

**`connectivity/connectivity.py` 中的模式**（概念性）：
```python
# 多源多汇归约示例
def multi_source_multi_sink_flow(G, sources, targets, capacity="capacity"):
    # 创建临时图
    H = G.copy()
    
    # 添加超级源点
    super_source = "_super_source_"
    H.add_node(super_source)
    for s in sources:
        H.add_edge(super_source, s, capacity=float("inf"))
    
    # 添加超级汇点
    super_sink = "_super_sink_"
    H.add_node(super_sink)
    for t in targets:
        H.add_edge(t, super_sink, capacity=float("inf"))
    
    # 计算单源单汇最大流
    return nx.maximum_flow(H, super_source, super_sink, capacity=capacity)
```

#### 节点连通性中的分裂技术

`build_auxiliary_node_connectivity` 实际上是另一种图变换：

```python
# 每个节点 v 分裂为 v_in 和 v_out
# 边 (v_in, v_out) 容量 = 1（表示"删除"该节点的代价）
# 原图边 u->v 变为 u_out -> v_in，容量 = INF
```

### 5.4 实际应用场景

**场景 1：多商品流简化**
- 当只关心总流量而非各商品独立路径时
- 使用超级源/汇归约为单商品流

**场景 2：全局最小割**
- `minimum_node_cut` / `minimum_edge_cut` 遍历所有节点对
- 本质上是多次单源单汇计算

**场景 3：图像分割（Boykov-Kolmogorov）**
- 前景种子点 = 多个源点
- 背景种子点 = 多个汇点
- 引入超级源连接所有前景种子，超级汇连接所有背景种子

### 5.5 归约的正确性证明

**等价性**：
1. **充分性**：原问题的任一可行流对应归约后问题的可行流
   - 超级源流出 = 各源点流出之和
   - 超级汇流入 = 各汇点流入之和

2. **必要性**：归约后问题的任一可行流对应原问题的可行流
   - 忽略超级源/汇节点
   - 各源/汇点的流量守恒仍然成立

3. **最优性保持**：
   - 最大流值相等（超级源/汇边容量无穷大，不构成瓶颈）

---

## 6. 架构设计总结

### 6.1 层次化架构

```
┌─────────────────────────────────────────────────────────────┐
│                    用户接口层                                  │
│  maximum_flow(), maximum_flow_value(),                      │
│  minimum_cut(), minimum_cut_value()                         │
│  (maxflow.py)                                                │
├─────────────────────────────────────────────────────────────┤
│                    算法实现层                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐  │
│  │ preflow_push│ │ edmonds_karp│ │ boykov_kolmogorov   │  │
│  │ (预流推进)   │ │ (路径增广)   │ │ (双搜索树)           │  │
│  └─────────────┘ └─────────────┘ └─────────────────────┘  │
│  ┌─────────────────┐ ┌─────────────────┐                    │
│  │ shortest_augment│ │ dinitz          │                    │
│  │ (最短增广路径)   │ │ (分层阻塞流)     │                    │
│  └─────────────────┘ └─────────────────┘                    │
├─────────────────────────────────────────────────────────────┤
│                    基础设施层                                  │
│  build_residual_network(), build_flow_dict(),               │
│  CurrentEdge, Level, GlobalRelabelThreshold                 │
│  (utils.py)                                                  │
├─────────────────────────────────────────────────────────────┤
│                    扩展应用层                                  │
│  connectivity/cuts.py (节点/边割),                           │
│  connectivity/connectivity.py (连通性),                      │
│  gomory_hu.py (Gomory-Hu 树)                                │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 关键设计模式

| 设计模式 | 应用位置 | 作用 |
|---------|---------|------|
| **策略模式** | `flow_func` 参数 | 运行时切换算法 |
| **模板方法** | 统一算法接口 | 保证各算法可互换 |
| **享元模式** | `residual` 参数 | 复用残差网络 |
| **图变换** | 超级源/汇、节点分裂 | 问题归约 |
| **启发式优化** | 全局重标记、间隙启发 | 实际性能提升 |

### 6.3 算法选择指南

| 场景 | 推荐算法 | 理由 |
|------|---------|------|
| **通用场景** | `preflow_push`（默认） | 理论复杂度最优，启发式加速 |
| **稀疏网络** | `edmonds_karp` | BFS 对高度偏斜度分布友好 |
| **稠密网络** | `shortest_augmenting_path` | 距离标签减少搜索空间 |
| **单位容量网络** | `shortest_augmenting_path(two_phase=True)` | 两阶段优化 $O(m\sqrt{n})$ |
| **图像分割** | `boykov_kolmogorov` | 双搜索树在网格图上极快 |
| **需要最小割分区** | 任意算法 + `minimum_cut` | 统一接口 |
| **多次计算同一图** | 任意算法 + `residual` 参数 | 复用残差网络 |

---

## 附录：核心文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `networkx/algorithms/flow/maxflow.py` | 统一入口：maximum_flow, minimum_cut 等 |
| `networkx/algorithms/flow/utils.py` | 残差网络构建、流字典构建等基础设施 |
| `networkx/algorithms/flow/preflowpush.py` | 最高标签预流推进算法 |
| `networkx/algorithms/flow/edmondskarp.py` | Edmonds-Karp 算法（双向 BFS） |
| `networkx/algorithms/flow/shortestaugmentingpath.py` | 最短增广路径算法（距离标签） |
| `networkx/algorithms/flow/dinitz_alg.py` | Dinitz 算法（分层阻塞流） |
| `networkx/algorithms/flow/boykovkolmogorov.py` | Boykov-Kolmogorov 算法（双搜索树） |
| `networkx/algorithms/connectivity/cuts.py` | 基于流的节点/边割计算 |
| `networkx/algorithms/connectivity/connectivity.py` | 图连通性计算 |
