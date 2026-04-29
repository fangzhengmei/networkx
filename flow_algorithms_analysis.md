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

## 7. 路径增广类算法的双向搜索优化

### 7.1 Edmonds-Karp 的双向 BFS 实现机制

Edmonds-Karp 算法是 Ford-Fulkerson 框架的经典实现，其核心优化在于使用**双向广度优先搜索**（而非单向）来寻找最短增广路径。这一优化在实践中显著减少了搜索空间。

#### 7.1.1 两侧队列的交替扩展逻辑

**核心实现**（`edmondskarp.py:40-69`）：

```python
def bidirectional_bfs():
    """Bidirectional breadth-first search for an augmenting path."""
    pred = {s: None}  # 源点方向的父节点映射：节点 → 前驱
    q_s = [s]          # 源点搜索队列
    succ = {t: None}  # 汇点方向的后继节点映射：节点 → 后继（反向路径）
    q_t = [t]          # 汇点搜索队列
    
    while True:
        q = []
        # 关键优化：始终选择较小的队列进行扩展
        if len(q_s) <= len(q_t):
            # 从源点方向扩展（正向遍历）
            for u in q_s:
                for v, attr in R_succ[u].items():
                    if v not in pred and attr["flow"] < attr["capacity"]:
                        pred[v] = u
                        if v in succ:  # 相遇检测
                            return v, pred, succ
                        q.append(v)
            if not q:
                return None, None, None
            q_s = q
        else:
            # 从汇点方向扩展（反向遍历，使用 R_pred）
            for u in q_t:
                for v, attr in R_pred[u].items():
                    if v not in succ and attr["flow"] < attr["capacity"]:
                        succ[v] = u
                        if v in pred:  # 相遇检测
                            return v, pred, succ
                        q.append(v)
            if not q:
                return None, None, None
            q_t = q
```

#### 7.1.2 交替扩展策略分析

**为什么选择较小的队列扩展？**

这是双向搜索的经典优化策略：

1. **减少总探索节点数**：
   - 单向 BFS：可能需要探索 $O(b^d)$ 个节点（$b$ 为分支因子，$d$ 为距离）
   - 双向 BFS：只需探索 $O(b^{d/2}) + O(b^{d/2})$ 个节点
   - 选择较小队列：保证最坏情况下也是最优的

2. **NetworkX 实现的特殊性**：
   - 源点方向使用 `R_succ`（正向边：`u→v` 的残余容量 = `capacity - flow`）
   - 汇点方向使用 `R_pred`（反向边：等价于搜索 `v→u` 的正向残余容量）

#### 7.1.3 相遇节点检测与路径拼接

**路径拼接逻辑**（`edmondskarp.py:77-88`）：

```python
v, pred, succ = bidirectional_bfs()
if pred is None:
    break

# 1. 从相遇节点回溯到源点
path = [v]
u = v
while u != s:
    u = pred[u]
    path.append(u)
path.reverse()  # 现在 path 是 s → ... → v

# 2. 从相遇节点追溯到汇点
u = v
while u != t:
    u = succ[u]
    path.append(u)  # path 变为 s → ... → v → ... → t
```

**数据结构设计**：

| 方向 | 映射关系 | 含义 | 遍历方式 |
|------|---------|------|---------|
| **源点方向** | `pred[v] = u` | 最短路径中 `v` 的前驱是 `u` | 正向遍历 `R_succ` |
| **汇点方向** | `succ[v] = u` | 最短路径中 `v` 的后继是 `u` | 反向遍历 `R_pred` |

**注意**：汇点方向的 `succ` 命名容易产生误解。实际上：
- `succ` 存储的是**从汇点出发的反向路径**
- `succ[v] = u` 意味着在反向搜索中，`v` 的下一个节点是 `u`
- 这等价于在正向路径中 `u` → `v`

### 7.2 与 Boykov-Kolmogorov 双搜索树的本质差异

虽然 Edmonds-Karp 和 Boykov-Kolmogorov 都采用了"双向"的思想，但它们在**双向的维度**上有本质区别。

#### 7.2.1 双向搜索策略对比

| 维度 | Edmonds-Karp 双向 BFS | Boykov-Kolmogorov 双搜索树 |
|------|---------------------|--------------------------|
| **搜索目标** | 寻找**一条**最短增广路径 | 生长**持久化**的搜索树 |
| **搜索状态** | 每次增广后**全部丢弃** | 增广后**尽量复用** |
| **数据结构** | 临时队列 + 父指针字典 | 持久化搜索树 + 活跃节点队列 |
| **相遇处理** | 找到路径立即返回 | 找到连接边后进入增广阶段 |
| **路径复用** | 无 | 通过 Adoption 阶段修复 |

#### 7.2.2 Edmonds-Karp：一次性路径搜索

**生命周期**：
```
┌─────────────────────────────────────────────────────────────┐
│                     Edmonds-Karp 主循环                       │
├─────────────────────────────────────────────────────────────┤
│  while flow_value < cutoff:                                   │
│      ┌─────────────────────────────────────────────────────┐ │
│      │  bidirectional_bfs()                                 │ │
│      │  - 初始化 pred = {s: None}, succ = {t: None}       │ │
│      │  - 交替扩展 q_s 和 q_t                               │ │
│      │  - 找到相遇节点 v 后返回 (v, pred, succ)            │ │
│      │  - 若未找到，返回 (None, None, None)                 │ │
│      └─────────────────────────────────────────────────────┘ │
│      if pred is None: break                                   │
│      ┌─────────────────────────────────────────────────────┐ │
│      │  augment(path)                                       │ │
│      │  - 计算路径残余容量                                  │ │
│      │  - 沿路径增广流量                                    │ │
│      └─────────────────────────────────────────────────────┘ │
│      ↓ pred 和 succ 被丢弃，下次循环重新构建                  │
└─────────────────────────────────────────────────────────────┘
```

**关键特征**：
1. **无状态性**：每次 `bidirectional_bfs()` 调用都是独立的
2. **最短路径保证**：双向 BFS 天然保证找到的是最短路径（边数最少）
3. **路径长度递增**：每次增广后，最短增广路径的长度**不会减少**（这是 Edmonds-Karp 复杂度证明的关键）

#### 7.2.3 Boykov-Kolmogorov：持久化搜索树

**生命周期**：
```
┌─────────────────────────────────────────────────────────────┐
│              Boykov-Kolmogorov 三阶段循环                     │
├─────────────────────────────────────────────────────────────┤
│  初始化:                                                       │
│    source_tree = {s: None}  ← 持久化源树                     │
│    target_tree = {t: None}  ← 持久化汇树                     │
│    active = deque([s, t])   ← 待扩展节点队列                 │
│                                                               │
│  while flow_value < cutoff:                                   │
│      ┌─────────────────────────────────────────────────────┐ │
│      │  GROW 阶段                                            │ │
│      │  - 从 active 队列取节点扩展                          │ │
│      │  - 源树节点: 正向扩展 R_succ，加入 source_tree       │ │
│      │  - 汇树节点: 反向扩展 R_pred，加入 target_tree       │ │
│      │  - 发现连接边 (u∈source_tree, v∈target_tree)        │ │
│      │    立即返回 (u, v)                                   │ │
│      └─────────────────────────────────────────────────────┘ │
│      if u is None: break  ← 无法生长，算法终止                │
│                                                               │
│      ┌─────────────────────────────────────────────────────┐ │
│      │  AUGMENT 阶段                                         │ │
│      │  - 从 u 回溯到 s: source_tree[u], source_tree[...],  │ │
│      │  - 从 v 追溯到 t: target_tree[v], target_tree[...],  │ │
│      │  - 合并为完整路径 s→...→u→v→...→t                   │ │
│      │  - 沿路径增广流量                                     │ │
│      │  - 标记饱和边对应的节点为孤儿 (orphans)               │ │
│      └─────────────────────────────────────────────────────┘ │
│                                                               │
│      ┌─────────────────────────────────────────────────────┐ │
│      │  ADOPTION 阶段（核心差异！）                          │ │
│      │  - 处理 orphans 队列中的孤儿节点                     │ │
│      │  - 尝试为孤儿找到新父节点（同树内）                   │ │
│      │  - 若找不到，则从树中删除，并将其子节点也变为孤儿    │ │
│      │  - 搜索树被修复，可继续用于下次 GROW                  │ │
│      └─────────────────────────────────────────────────────┘ │
│      ↓ source_tree 和 target_tree 被保留，继续使用            │
└─────────────────────────────────────────────────────────────┘
```

#### 7.2.4 本质差异总结

| 特性 | Edmonds-Karp 双向 BFS | Boykov-Kolmogorov 双搜索树 |
|------|---------------------|--------------------------|
| **双向的含义** | 单次路径搜索的双向加速 | 持久化数据结构的双向维护 |
| **状态保持** | 无状态，每次重建 | 有状态，搜索树持久化 |
| **路径寻找** | 每次寻找完整路径 | 利用已有树结构拼接路径 |
| **增广代价** | 每次需重新搜索 | 只需修复被破坏的树 |
| **复杂度** | $O(n m^2)$（最坏） | $O(n^2 m |C|)$（最坏） |
| **实践性能** | 稀疏网络表现好 | 网格图/图像分割极快 |
| **最短路径** | 保证最短路径 | 不保证，但有标记启发式 |

**核心洞察**：
- **Edmonds-Karp** 的"双向"是**算法层面**的优化：用更少的步骤找到同一条路径
- **Boykov-Kolmogorov** 的"双向"是**数据结构层面**的设计：用两棵树覆盖更多节点，减少重复探索

### 7.3 标记启发式（Marking Heuristic）

Boykov-Kolmogorov 的搜索树虽然不保证最短路径，但通过**时间戳 + 距离**的标记启发式来维护路径质量。

**实现**（`boykovkolmogorov.py:347-348`）：
```python
def _is_closer(u, v):
    """检查 u 是否提供到 v 的更短路径"""
    return timestamp[v] <= timestamp[u] and dist[v] > dist[u] + 1
```

**GROW 阶段中的应用**（`boykovkolmogorov.py:231-234`）：
```python
elif v in this_tree and _is_closer(u, v):
    # v 已在树中，但 u 提供了更短的路径
    this_tree[v] = u           # 更新父指针
    dist[v] = dist[u] + 1      # 更新距离
    timestamp[v] = timestamp[u] # 更新时间戳
```

**时间戳的作用**：
- `time` 变量在每次 AUGMENT 后递增
- `timestamp[v]` 记录节点 v 最后一次被处理的"时间"
- `timestamp[v] <= timestamp[u]` 保证 u 的信息不"过期"

---

## 8. Dinitz 算法的分层图机制

Dinitz 算法是路径增广类算法的重大改进，其核心创新在于**分层图（Level Graph）**和**阻塞流（Blocking Flow）**的组合。

### 8.1 分层图的构建

#### 8.1.1 BFS 分层机制

**核心实现**（`dinitz_alg.py:180-198`）：

```python
def breath_first_search():
    """构建分层图：只保留最短路径上的边"""
    parents = {}              # 记录每个节点的父节点列表（允许多父）
    vertex_dist = {s: 0}      # 节点到源点的距离（层数）
    queue = deque([(s, 0)])
    
    while queue:
        if t in parents:
            break  # 汇点已可达，提前终止（但仍需处理当前层）
        u, dist = queue.popleft()
        
        for v, attr in R_succ[u].items():
            if attr["capacity"] - attr["flow"] > 0:  # 边有残余容量
                if v in parents:
                    # v 已被访问，但可能通过同层另一条路径到达
                    if vertex_dist[v] == dist + 1:
                        parents[v].append(u)  # 增加一个父节点
                else:
                    # v 首次被访问
                    parents[v] = deque([u])
                    vertex_dist[v] = dist + 1
                    queue.append((v, dist + 1))
    return parents
```

#### 8.1.2 分层图的关键性质

**分层图定义**：
- 只包含满足 `dist[v] == dist[u] + 1` 的边 `u→v`
- 所有路径都是从 s 到 t 的**最短路径**（边数最少）
- 是一个 **DAG（有向无环图）**，边只从低层指向高层

**多层父节点设计**：
```python
# 允许多个父节点：v 可以通过 u1 或 u2 到达，距离相同
parents[v] = deque([u1, u2, ...])
```

这允许 DFS 在寻找阻塞流时探索多条路径，而无需重新 BFS。

### 8.2 阻塞流的寻找与推进

#### 8.2.1 DFS 寻找阻塞流

**核心实现**（`dinitz_alg.py:200-234`）：

```python
def depth_first_search(parents):
    """在分层图中寻找所有增广路径（阻塞流）"""
    total_flow = 0
    u = t              # 从汇点开始反向 DFS（巧妙设计！）
    path = [u]         # 路径栈
    
    while True:
        if len(parents[u]) > 0:
            # u 有可用父节点，向前探索
            v = parents[u][0]
            path.append(v)
        else:
            # u 无可用父节点，回溯
            path.pop()
            if len(path) == 0:
                break  # 无更多路径
            v = path[-1]
            parents[v].popleft()  # 移除失效的边
        
        # 检查是否到达源点（找到完整路径）
        if v == s:
            # 1. 计算路径残余容量
            flow = INF
            for u_node, v_node in pairwise(path):
                # path 是 t → ... → s，所以用 R_pred 访问反向边
                flow = min(flow, R_pred[u_node][v_node]["capacity"] - 
                          R_pred[u_node][v_node]["flow"])
            
            # 2. 沿路径增广
            for u_node, v_node in pairwise(reversed(path)):
                # reversed(path) 是 s → ... → t
                R_pred[v_node][u_node]["flow"] += flow
                R_pred[u_node][v_node]["flow"] -= flow
                
                # 3. 检查边是否饱和
                if R_pred[v_node][u_node]["capacity"] - R_pred[v_node][u_node]["flow"] == 0:
                    parents[v_node].popleft()  # 从可用父节点中移除
                    # 回溯到饱和边的起点
                    while path[-1] != v_node:
                        path.pop()
            
            total_flow += flow
            v = path[-1]
        u = v
    return total_flow
```

#### 8.2.2 反向 DFS 的设计洞察

**为什么从汇点开始？**

常规 DFS（从 s 到 t）：
```
s → a → b → c → t  (路径1)
s → a → d → t      (路径2)
```
- 找到路径1后，可能需要多次回溯才能找到路径2

反向 DFS（从 t 到 s）：
```
path = [t]
探索 t 的父节点 → 加入 path
...
直到 path[0] == s
```
- 利用 `parents` 结构保证每步都朝向源点
- 更高效的回溯和饱和边处理

#### 8.2.3 阻塞流的定义

**阻塞流**：在分层图中，**无法再找到任何增广路径**时的流。

注意：阻塞流 ≠ 最大流！
- 阻塞流是**当前分层图**中的最大流
- 增广后，最短路径长度**严格增加**
- 需要重新 BFS 构建新的分层图

### 8.3 饱和边与层次回退

#### 8.3.1 主循环结构

**完整迭代**（`dinitz_alg.py:236-244`）：

```python
flow_value = 0
while flow_value < cutoff:
    # 阶段1: BFS 构建分层图
    parents = breath_first_search()
    if t not in parents:
        break  # 汇点不可达，已达最大流
    
    # 阶段2: DFS 寻找阻塞流
    this_flow = depth_first_search(parents)
    if this_flow * 2 > INF:
        raise nx.NetworkXUnbounded("Infinite capacity path")
    
    flow_value += this_flow
    # 隐式层次回退：下次 BFS 将构建新的分层图
```

#### 8.3.2 与 Edmonds-Karp 的逐条增广对比

**Edmonds-Karp 模式**：
```
循环:
    BFS 找一条最短路径
    增广这条路径
    重复
```
- 每次增广可能只推送少量流量
- 需要 $O(m)$ 次 BFS（最坏情况）

**Dinitz 模式**：
```
循环:
    BFS 构建分层图 (1次)
    DFS 推送阻塞流 (可能多条路径)
    重复
```
- 一次 BFS 后推送尽可能多的流量
- BFS 次数 = 最短路径长度的增加次数 ≤ $O(n)$

#### 8.3.3 效率差异分析

**理论复杂度**：

| 算法 | 时间复杂度 | 单位容量网络 |
|------|-----------|-------------|
| Edmonds-Karp | $O(n m^2)$ | $O(n m^2)$ |
| Dinitz | $O(n^2 m)$ | $O(m \sqrt{n})$ |

**实际差异来源**：

1. **BFS 次数减少**：
   - Edmonds-Karp：每次增广可能需要一次 BFS
   - Dinitz：BFS 次数 = 层次数 ≤ $n$

2. **DFS 效率**：
   - 分层图是 DAG，无环
   - 每层只处理一次
   - 使用 `CurrentEdge` 类优化邻接表遍历

3. **饱和边处理**：
   - 边饱和后从 `parents` 中移除
   - 下次 DFS 不会再探索这些边

### 8.4 Shortest Augmenting Path 的层次优化

Shortest Augmenting Path 算法与 Dinitz 有相似的层次思想，但实现方式不同。

**距离标签（Height Label）**（`shortestaugmentingpath.py:38-61`）：
```python
# 从汇点反向 BFS 初始化高度
heights = {t: 0}
q = deque([(t, 0)])
while q:
    u, height = q.popleft()
    height += 1
    for v, attr in R_pred[u].items():
        if v not in heights and attr["flow"] < attr["capacity"]:
            heights[v] = height
            q.append((v, height))
```

**许可边（Admissible Edge）**：
- 只有满足 `height[u] == height[v] + 1` 的边 `u→v` 才被考虑
- 这保证了每次增广都是最短路径

**间隙启发式（Gap Heuristic）**（`shortestaugmentingpath.py:63-66, 123-129`）：
```python
counts = [0] * (2 * n - 1)
for u in R:
    counts[R_nodes[u]["height"]] += 1

# 当某层计数变为 0 时
counts[height] -= 1
if counts[height] == 0:
    # 间隙出现！更高层的节点无法到达汇点
    R.graph["flow_value"] = flow_value
    return R
```

### 8.5 分层图与距离标签的对比

| 特性 | Dinitz 分层图 | Shortest Augmenting Path 距离标签 |
|------|-------------|----------------------------------|
| **数据结构** | BFS 构建的 `parents` 字典 | 每个节点的 `height` 属性 |
| **更新时机** | 每次阻塞流后重新 BFS | 节点重标签时动态更新 |
| **路径保证** | 只在分层图内搜索 | 检查许可边条件 |
| **提前终止** | 汇点不在 `parents` 中 | 间隙启发式 |
| **复杂度** | $O(n^2 m)$ | $O(n^2 m)$ |
| **单位容量** | - | $O(\min(n^{2/3}, m^{1/2}) m)$ 两阶段优化 |

**核心差异**：
- **Dinitz**：显式构建分层图，一次性推送阻塞流
- **SAP**：隐式通过高度标签维护层次，每次推送一条路径但有更激进的启发式

---

## 9. 连通性模块中的超级源/汇归约

NetworkX 的连通性计算（节点连通度、边连通度）是最大流算法的重要应用场景。这些问题通过**图变换**归约为单源单汇最大流问题。

### 9.1 节点连通度的归约：节点分裂技术

#### 9.1.1 问题背景

**节点连通度**（Node Connectivity）$\kappa(G)$：
- 最少需要删除多少个节点才能使图不连通
- 或使特定的源汇对 $s, t$ 不连通（局部节点连通度）

**难点**：
- 最大流算法天然处理**边**的容量约束
- 如何建模**节点**的"删除代价"？

#### 9.1.2 节点分裂技术

**核心思想**：将每个节点 $v$ 拆分为两个节点 $v_A$ 和 $v_B$，用一条内部边连接。

**辅助图构建**（`connectivity/utils.py:10-60`）：

```python
def build_auxiliary_node_connectivity(G):
    directed = G.is_directed()
    mapping = {}
    H = nx.DiGraph()
    
    # 阶段1: 分裂每个节点
    for i, node in enumerate(G):
        mapping[node] = i
        # 创建两个节点：vA 和 vB
        H.add_node(f"{i}A", id=node)  # 入节点
        H.add_node(f"{i}B", id=node)  # 出节点
        # 添加内部边 vA → vB，容量 = 1
        # 割这条边等价于"删除"原节点 v
        H.add_edge(f"{i}A", f"{i}B", capacity=1)
    
    # 阶段2: 转换原图边
    edges = []
    for source, target in G.edges():
        # 原图边 u→v 变为 uB → vA
        # 容量 = 1（或无穷大，取决于实现）
        edges.append((f"{mapping[source]}B", f"{mapping[target]}A"))
        if not directed:
            # 无向图需要双向边
            edges.append((f"{mapping[target]}B", f"{mapping[source]}A"))
    H.add_edges_from(edges, capacity=1)
    
    H.graph["mapping"] = mapping
    return H
```

#### 9.1.3 分裂后的图结构

**原图**：
```
s -----> a -----> t
 \              /
  \----> b ----/
```

**辅助图**：
```
sA --1--> sB --1--> aA --1--> aB --1--> tA --1--> tB
           \                          /
            \----1--> bA --1--> bB --/
```

**关键观察**：
1. 每个内部边 `vA→vB` 容量为 1
2. 原图边 `u→v` 变为 `uB→vA`，容量为 1
3. 要从 `sB` 到 `tA`，必须经过一系列 `xA→xB` 边

#### 9.1.4 源汇对的选择

**节点割计算**（`connectivity/cuts.py:167-306`）：

```python
def minimum_st_node_cut(G, s, t, flow_func=None, auxiliary=None, residual=None):
    # 构建辅助图
    if auxiliary is None:
        H = build_auxiliary_node_connectivity(G)
    
    mapping = H.graph["mapping"]
    
    # 关键：源点用 B 节点，汇点用 A 节点
    # 源点 s: 从 sB 出发（绕过 sA→sB 边）
    # 汇点 t: 到达 tA 结束（绕过 tA→tB 边）
    edge_cut = minimum_st_edge_cut(H, f"{mapping[s]}B", f"{mapping[t]}A", **kwargs)
    
    # 转换回原图节点
    # 辅助图中的边割 (vA, vB) 对应原图节点 v
    node_cut = {H.nodes[node]["id"] for edge in edge_cut for node in edge}
    return node_cut - {s, t}  # 排除源汇本身
```

**为什么源用 B、汇用 A？**

- 如果源点是 `sA`：割 `sA→sB` 会把源点"删除"，这不是我们想要的
- 如果汇点是 `tB`：割 `tA→tB` 会把汇点"删除"，这也不是我们想要的
- 正确选择：`sB` 出发，`tA` 到达，这样只有**中间节点**的内部边会被割

### 9.2 边连通度的归约

#### 9.2.1 边连通度问题

**边连通度**（Edge Connectivity）$\lambda(G)$：
- 最少需要删除多少条边才能使图不连通

**相比节点连通度**：
- 边连通度更"自然"地对应最大流
- 无需节点分裂，只需简单的图变换

#### 9.2.2 辅助图构建

**实现**（`connectivity/utils.py:63-88`）：

```python
def build_auxiliary_edge_connectivity(G):
    if G.is_directed():
        # 有向图：直接添加 capacity=1
        H = nx.DiGraph()
        H.add_nodes_from(G.nodes())
        H.add_edges_from(G.edges(), capacity=1)
        return H
    else:
        # 无向图：每条边替换为两条反向边
        H = nx.DiGraph()
        H.add_nodes_from(G.nodes())
        for source, target in G.edges():
            # u-v 变为 u→v 和 v→u，各 capacity=1
            H.add_edges_from([(source, target), (target, source)], capacity=1)
        return H
```

#### 9.2.3 无向图的双向边

**原图**（无向）：
```
s ------ a ------ t
 \              /
  \------ b ----/
```

**辅助图**（有向）：
```
s <--1--> a <--1--> t
 \              /
  \<--1--> b <--/
```

每条无向边 `u-v` 变为两条有向边 `u→v` 和 `v→u`，容量均为 1。

### 9.3 两类辅助图的结构对比

| 特性 | 节点连通度辅助图 | 边连通度辅助图 |
|------|-----------------|---------------|
| **节点数** | $2n$（每个原节点分裂为 2 个） | $n$（与原图相同） |
| **边数** | $m + n$（原图边 + 内部边） | $2m$（无向）或 $m$（有向） |
| **核心变换** | 节点分裂 + 内部边 | 无向边转双向有向边 |
| **容量约束** | 内部边 capacity=1 | 所有边 capacity=1 |
| **源汇选择** | 源用 B 节点，汇用 A 节点 | 源汇与原图相同 |
| **割的含义** | 割内部边 = 删除节点 | 割原图边 = 删除边 |

### 9.4 完整归约示例

#### 9.4.1 节点连通度计算示例

**原图**：
```
s -- a -- t
|    |    |
+----b----+
```

这是一个 2-节点连通图（删除 s 或 t 不算，需要删除 a 和 b）。

**辅助图**：
```
sA--1-->sB--1-->aA--1-->aB--1-->tA--1-->tB
 |       |       |       |       |
 |       +--1-->bA--1-->bB--1-->+
 |               |
 +--------1------+
```

**最大流计算**：
- 源点：`sB`
- 汇点：`tA`
- 最大流值 = 2（一条经过 a，一条经过 b）
- 这对应节点连通度 = 2

**最小割**：
- 割集可能包含 `(aA, aB)` 和 `(bA, bB)`
- 对应原图节点 {a, b}

#### 9.4.2 边连通度计算示例

**同一张原图**：

**辅助图**（无向转双向有向）：
```
s <--1--> a <--1--> t
^         ^         ^
|         |         |
+----1----+----1----+
         b
```

**最大流计算**：
- 源点：`s`
- 汇点：`t`
- 最大流值 = 3（s→a→t, s→b→t, s→a→b→t...）
- 实际边连通度 = 2（删除 s-a 和 s-b）

**注意**：边连通度的最大流值可能大于实际连通度，需要考虑所有可能的源汇对。

### 9.5 全局连通度的多源多汇归约

#### 9.5.1 全局最小节点割

**实现**（`connectivity/cuts.py:309-452`）：

```python
def minimum_node_cut(G, s=None, t=None, flow_func=None):
    # 局部最小割：指定源汇
    if s is not None and t is not None:
        return minimum_st_node_cut(G, s, t, flow_func=flow_func)
    
    # 全局最小割：遍历所有节点对
    # 选择度数最小的节点作为基准
    v = min(G, key=G.degree)
    
    # 初始割集：v 的所有邻居
    min_cut = set(G[v])
    
    # 计算 v 到所有非邻居的节点割
    for w in set(G) - set(neighbors(v)) - {v}:
        this_cut = minimum_st_node_cut(G, v, w, **kwargs)
        if len(min_cut) >= len(this_cut):
            min_cut = this_cut
    
    # 还需检查 v 的邻居之间的割
    for x, y in iter_func(neighbors(v), 2):
        if y in G[x]:
            continue  # 直接相连，无法通过节点割分离
        this_cut = minimum_st_node_cut(G, x, y, **kwargs)
        if len(min_cut) >= len(this_cut):
            min_cut = this_cut
    
    return min_cut
```

#### 9.5.2 本质：多次单源单汇计算

全局连通度没有"自然"的超级源/汇归约，但可以：
1. 选择一个基准节点 $v$
2. 计算 $v$ 到所有其他节点的局部连通度
3. 取最小值

这等价于**多次独立的单源单汇最大流计算**，而非单次多源多汇归约。

### 9.6 归约策略总结

| 问题类型 | 归约方法 | 关键变换 |
|---------|---------|---------|
| **局部节点连通度** | 节点分裂 + 单源单汇 | $v \to (vA, vB)$，边 $vA→vB$ capacity=1 |
| **局部边连通度** | 单向/双向边 + 单源单汇 | 无向边转两条有向边，capacity=1 |
| **全局节点连通度** | 多次局部计算 | 遍历节点对，取最小割 |
| **全局边连通度** | 多次局部计算 | 遍历节点对，取最小割 |
| **多源多汇最大流** | 超级源/汇 | 添加 $S^*$ 连所有源，$T^*$ 连所有汇 |

**核心洞察**：
- 节点连通度的**节点分裂**是最巧妙的图变换
- 它将"删除节点"的离散操作转化为"割边"的连续优化问题
- 最大流-最小割定理自然适用于这种变换后的图

---

## 10. 当前边游标机制（CurrentEdge）

### 10.1 问题背景：邻接表重复扫描的开销

在最高标签预流推进算法中，核心操作是**discharge**（释放）：对一个有超额流的活跃节点，扫描其所有出边，尝试将超额流推送到邻居节点。

**朴素实现的问题**：
```python
# 朴素实现（不使用 CurrentEdge）
def discharge_naive(u):
    while excess[u] > 0:
        for v in R_succ[u]:  # 每次都从头扫描所有边！
            if height[u] == height[v] + 1 and residual(u, v) > 0:
                push(u, v, min(excess[u], residual(u, v)))
                if excess[u] == 0:
                    break
        # 如果没有可用边，需要重标记
        height[u] = relabel(u)
```

**问题**：
1. 每次 `discharge` 都从邻接表的**第一个元素**开始扫描
2. 如果前几条边已饱和或高度不满足条件，每次都要重新检查
3. 最坏情况下，每条边可能被检查 $O(n)$ 次，导致 $O(n m)$ 的邻接表扫描开销

### 10.2 CurrentEdge 数据结构设计

**核心实现**（`utils.py:19-48`）：

```python
class CurrentEdge:
    """Mechanism for iterating over out-edges incident to a node in a circular
    manner. StopIteration exception is raised when wraparound occurs.
    """

    __slots__ = ("_edges", "_it", "_curr")

    def __init__(self, edges):
        self._edges = edges
        if self._edges:
            self._rewind()

    def get(self):
        return self._curr

    def move_to_next(self):
        try:
            self._curr = next(self._it)
        except StopIteration:
            self._rewind()
            raise

    def _rewind(self):
        self._it = iter(self._edges.items())
        self._curr = next(self._it)

    def __eq__(self, other):
        return (getattr(self, "_curr", None), self._edges) == (
            (getattr(other, "_curr", None), other._edges)
        )
```

**数据结构解析**：

| 属性 | 类型 | 作用 |
|------|------|------|
| `_edges` | dict | 节点的出边邻接表（`R_succ[u]`） |
| `_it` | iterator | 当前迭代器位置 |
| `_curr` | tuple | 当前边 `(v, attr)` |

**设计特点**：
1. **游标持久化**：游标位置在多次 `discharge` 调用之间保持
2. **循环遍历**：到达末尾时抛出 `StopIteration`，可通过 `_rewind()` 重置
3. **内存优化**：使用 `__slots__` 减少内存开销

### 10.3 在 Discharge 中的使用方式

**初始化**（`preflowpush.py:86-88`）：
```python
# 为每个节点创建 CurrentEdge 游标
for u in R:
    R_nodes[u]["height"] = heights[u] if u in heights else n + 1
    R_nodes[u]["curr_edge"] = CurrentEdge(R_succ[u])  # 游标初始化
```

**Discharge 中的使用**（`preflowpush.py:134-173`）：

```python
def discharge(u, is_phase1):
    """Discharge a node until it becomes inactive or, during phase 1 (see
    below), its height reaches at least n. The node is known to have the
    largest height among active nodes.
    """
    height = R_nodes[u]["height"]
    curr_edge = R_nodes[u]["curr_edge"]  # 获取持久化游标
    next_height = height
    levels[height].active.remove(u)
    
    while True:
        # 使用游标获取当前边（不是从头开始！）
        v, attr = curr_edge.get()
        
        if height == R_nodes[v]["height"] + 1 and attr["flow"] < attr["capacity"]:
            # 发现许可边，尝试推流
            flow = min(R_nodes[u]["excess"], attr["capacity"] - attr["flow"])
            push(u, v, flow)
            activate(v)
            if R_nodes[u]["excess"] == 0:
                # 超额流已推送完毕，节点变为非活跃
                levels[height].inactive.add(u)
                break  # 游标位置保留，下次从这里继续
        
        # 移动到下一条边
        try:
            curr_edge.move_to_next()
        except StopIteration:
            # 遍历完所有边，没有找到许可边
            # 需要重标记来创建新的许可边
            height = relabel(u)  # 重标记会增加高度
            
            if is_phase1 and height >= n - 1:
                # 第一阶段特殊处理：高度 >= n-1 的节点在 S 侧
                levels[height].active.add(u)
                break
            
            next_height = height
    
    R_nodes[u]["height"] = height
    return next_height
```

### 10.4 游标前进与重置的触发条件

**前进时机**：
```python
# 当当前边不满足条件时（高度不匹配 或 已饱和）
if not (height == R_nodes[v]["height"] + 1 and attr["flow"] < attr["capacity"]):
    curr_edge.move_to_next()  # 前进到下一条边
```

**重置与重标记的联动**：

**`move_to_next()` 的内部执行顺序**（`utils.py:34-39`）：
```python
def move_to_next(self):
    try:
        self._curr = next(self._it)
    except StopIteration:
        # 注意执行顺序：
        # 1. 先调用 _rewind() 回绕到首边
        # 2. 再抛出 StopIteration 传播给 discharge
        self._rewind()
        raise
```

这意味着：
- **游标重置和重标记触发是同一个事件的两个结果**
- 当 `StopIteration` 被 `discharge` 捕获时，游标已经回绕到首边了

**`discharge` 中的处理**（`preflowpush.py:1528-1541`）：
```python
# 移动到下一条边
try:
    curr_edge.move_to_next()
except StopIteration:
    # 此时游标已经通过 _rewind() 回绕到首边了！
    # 遍历完所有边，没有找到许可边
    # 需要重标记来创建新的许可边
    height = relabel(u)  # 重标记会增加高度
    
    if is_phase1 and height >= n - 1:
        # 第一阶段特殊处理：高度 >= n-1 的节点在 S 侧
        levels[height].active.add(u)
        break
    
    next_height = height
```

**关键修正**：
- ~~重标记后不重置游标~~ → **重标记发生在游标重置之后**
- 当 `discharge` 捕获 `StopIteration` 时，游标已经通过 `_rewind()` 回绕到首边
- 重标记操作（`height = relabel(u)`）发生在游标重置之后

**为什么重标记后需要从头扫描？**

重标记会增加节点高度：
```python
def relabel(u):
    return min(R_nodes[v]["height"] 
               for v, attr in R_succ[u].items()
               if attr["flow"] < attr["capacity"]) + 1
```

重标记后，新的高度 = 邻居最小高度 + 1。这意味着：
- 原来不满足 `height[u] == height[v] + 1` 的边，现在可能满足了
- 之前被跳过的边（因为高度不匹配），现在可能变成许可边
- 因此需要**从头重新扫描所有边**

这解释了为什么 `move_to_next()` 在抛出 `StopIteration` 前先调用 `_rewind()`：**下一轮扫描需要从首边开始**。

### 10.5 效率提升分析

**朴素实现**：
```
discharge(u):
    从头扫描所有边:
        边1: 饱和 → 跳过
        边2: 高度不匹配 → 跳过
        边3: 许可边 → 推送流量
    下次 discharge(u):
        从头扫描所有边:
            边1: 仍饱和 → 跳过
            边2: 高度仍不匹配 → 跳过
            边3: 可能仍有残余容量 → 继续推送
```

**CurrentEdge 优化**：
```
discharge(u):
    从游标位置开始:
        边1: 饱和 → 前进
        边2: 高度不匹配 → 前进
        边3: 许可边 → 推送流量
    游标停在边4的位置
    下次 discharge(u):
        从边4开始:
            边4: ...
```

**时间复杂度改进**：
- **朴素实现**：每条边可能被检查 $O(n)$ 次，总邻接表扫描 $O(n m)$
- **CurrentEdge 优化**：每条边最多被检查 $O(1)$ 次（除非节点被重标记）
- 重标记次数为 $O(n^2)$（最高标签预流推进），但邻接表扫描开销显著降低

### 10.6 与 Dinitz 指针优化的对比

| 特性 | Preflow-Push CurrentEdge | Dinitz 指针优化 |
|------|--------------------------|-----------------|
| **数据结构** | 独立 `CurrentEdge` 类 | `parents` 列表 |
| **持久化范围** | **同一扫描轮内**跨 discharge 调用保持 | 每次 BFS 后重建 |
| **遍历方向** | 正向遍历出边 | 反向从汇点到源点 |
| **重置时机** | 扫描轮结束时（重标记事件）自动 `_rewind()` | 每次 BFS 后完全重置 |
| **适用场景** | 活跃节点频繁 discharge | 分层图阻塞流搜索 |

**本质差异**：
- **CurrentEdge**：为单个节点的多次 discharge 优化，游标在活跃节点生命周期内保持
- **Dinitz 指针**：为单次阻塞流搜索优化，在分层图生命周期内有效

---

## 11. 全局重标记启发式

### 11.1 问题背景：高度标签的保守性

在预流推进算法中，节点的**高度标签**（height label）决定了推流的方向：
- 只有当 `height[u] == height[v] + 1` 时，才能从 `u` 推流到 `v`
- 当节点无法推流时，需要**重标记**（relabel）来增加高度

**朴素重标记的问题**：
```python
def relabel_naive(u):
    """朴素重标记：取所有邻居高度的最小值 + 1"""
    min_height = INF
    for v, attr in R_succ[u].items():
        if attr["flow"] < attr["capacity"]:  # 边有残余容量
            min_height = min(min_height, R_nodes[v]["height"])
    return min_height + 1
```

**问题**：
1. **保守上界**：朴素重标记得到的高度是**保守上界**，不是精确距离
2. **漂移问题**：随着算法进行，高度标签可能与实际最短路径距离偏差越来越大
3. **效率下降**：高度不准确会导致更多无效的推流尝试

### 11.2 GlobalRelabelThreshold 触发机制

**阈值计算**（`utils.py:61-77`）：

```python
class GlobalRelabelThreshold:
    """Measurement of work before the global relabeling heuristic should be
    applied.
    """

    def __init__(self, n, m, freq):
        # 阈值 = (节点数 + 边数) / 频率
        self._threshold = (n + m) / freq if freq else float("inf")
        self._work = 0  # 累计工作量

    def add_work(self, work):
        self._work += work

    def is_reached(self):
        return self._work >= self._threshold

    def clear_work(self):
        self._work = 0
```

**参数解析**：

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `n` | 图节点数 | 基数计算 |
| `m` | 图边数 | 基数计算 |
| `freq` | `1`（`preflow_push` 默认） | 频率因子 |

**阈值公式**：
$$\text{threshold} = \frac{n + m}{\text{freq}}$$

当 `freq=1` 时：
- 累计工作量达到 $n + m$ 时触发全局重标记
- 这意味着每处理约一条边/一个节点后进行一次全局重标记

### 11.3 工作量的累积方式

**在哪里累积？**（`preflowpush.py:122-132`）：

```python
def relabel(u):
    """Relabel a node to create an admissible edge."""
    # 每次重标记扫描节点的所有出边，工作量 += 出边数
    grt.add_work(len(R_succ[u]))
    return (
        min(
            R_nodes[v]["height"]
            for v, attr in R_succ[u].items()
            if attr["flow"] < attr["capacity"]
        )
        + 1
    )
```

**工作量定义**：
- 每次 `relabel(u)` 扫描 `u` 的所有出边
- 工作量 = 扫描的边数
- 这是一种**近似度量**，不精确但实用

**触发时机**（`preflowpush.py:238-243`）：

```python
while height > 0:
    while True:
        # ... discharge 操作 ...
        height = discharge(u, True)
        
        if grt.is_reached():
            # 阈值已达，执行全局重标记
            height = global_relabel(True)  # from_sink=True
            max_height = height
            grt.clear_work()  # 重置工作量计数
```

### 11.4 精确高度计算：反向 BFS

**核心函数**（`preflowpush.py:53-66`）：

```python
def reverse_bfs(src):
    """Perform a reverse breadth-first search from src in the residual
    network.
    """
    heights = {src: 0}  # 源点高度为 0
    q = deque([(src, 0)])
    while q:
        u, height = q.popleft()
        height += 1
        # 反向遍历：从 v 到 u 的反向边有残余容量
        # 等价于 u 到 v 的正向边有残余容量
        for v, attr in R_pred[u].items():
            if v not in heights and attr["flow"] < attr["capacity"]:
                heights[v] = height
                q.append((v, height))
    return heights
```

**为什么是反向 BFS？**

在残差网络中：
- 我们想知道**从各节点到汇点的最短距离**（边数）
- 正向 BFS 从汇点出发无法遍历所有节点（可能没有正向路径）
- 反向 BFS 从汇点出发，遍历所有有路径到达汇点的节点

**高度的含义**：

| 方向 | BFS 起点 | 高度含义 |
|------|---------|---------|
| **反向 BFS 从 t** | 汇点 | `height[v]` = 从 v 到 t 的最短路径边数 |
| **反向 BFS 从 s** | 源点 | `height[v]` = 从 s 到 v 的最短路径边数 |

### 11.5 全局重标记的完整实现

**`global_relabel` 函数**（`preflowpush.py:188-218`）：

```python
def global_relabel(from_sink):
    """Apply the global relabeling heuristic."""
    # 选择 BFS 起点
    src = t if from_sink else s
    
    # 执行反向 BFS 计算精确高度
    heights = reverse_bfs(src)
    
    if not from_sink:
        # 从源点出发的情况：需要调整高度偏移
        # 因为源点 s 的高度应为 n（不是 0）
        del heights[t]  # 移除汇点
        max_height = max(heights.values())
    else:
        # 从汇点出发的情况
        max_height = max(heights.values())
        
        # 标记无法到达汇点的节点（高度设为 n+1）
        for u in R:
            if u not in heights and R_nodes[u]["height"] < n:
                heights[u] = n + 1
    
    if not from_sink:
        # 从源点出发：高度需要偏移 n
        # 这样 s 的高度 = 0 + n = n，与初始化一致
        for u in heights:
            heights[u] += n
        max_height += n
    
    del heights[src]  # 移除起点本身
    
    # 更新所有节点的高度和层级
    for u, new_height in heights.items():
        old_height = R_nodes[u]["height"]
        if new_height != old_height:
            # 从旧层级移除
            if u in levels[old_height].active:
                levels[old_height].active.remove(u)
                levels[new_height].active.add(u)
            else:
                levels[old_height].inactive.remove(u)
                levels[new_height].inactive.add(u)
            # 更新高度
            R_nodes[u]["height"] = new_height
    
    return max_height
```

### 11.6 两种搜索方向的差异

**Phase 1：从汇点重标记（`from_sink=True`）**

```python
# 第一阶段目标：推送尽可能多的流量到汇点
# 关心的是：节点到汇点的距离
height = global_relabel(True)  # 从汇点反向 BFS
```

**高度含义**：
- `height[v]` = 从 v 到 t 的最短路径边数
- 无法到达 t 的节点：`height = n + 1`（特殊标记）

**Phase 2：从源点重标记（`from_sink=False`）**

```python
# 第二阶段目标：将超额流推送回源点
# 关心的是：源点到各节点的距离（反向意义上）
height = global_relabel(False)  # 从源点反向 BFS
```

**高度含义**：
- 基础值 = 从 s 到 v 的最短路径边数
- 偏移后 = 基础值 + $n$
- 这样 s 的高度 = $0 + n = n$，与初始化一致

### 11.7 精确高度 vs 保守高度

**对比表**：

| 特性 | 精确高度（全局重标记） | 保守高度（朴素重标记） |
|------|----------------------|---------------------|
| **计算方式** | 反向 BFS 遍历整个残差网络 | 扫描单个节点的出边 |
| **高度性质** | 到源/汇的精确最短距离 | 邻居最小高度 + 1（保守上界） |
| **更新频率** | 每 $n+m$ 工作量触发一次 | 节点无法推流时触发 |
| **准确性** | 100% 准确 | 可能比实际距离大很多 |
| **计算开销** | $O(n + m)$ 每次 | $O(\text{出边数})$ 每次 |
| **对算法的影响** | 减少无效推流尝试 | 可能导致高度"漂移" |

**为什么精确高度更好？**

```
假设实际距离：
s ---a--- b --- t   (s到t距离=3)

朴素重标记可能得到：
s(height=5) → a(height=4) → b(height=2) → t(height=0)

问题：s 到 a 的高度差 = 1（正确），但 a 到 b 的高度差 = 2（错误）
     这会导致 a 无法推流到 b，需要额外的重标记

全局重标记得到：
s(height=3) → a(height=2) → b(height=1) → t(height=0)

所有许可边都是正确的最短路径边
```

### 11.8 全局重标记与间隙启发式的关系

**相似之处**：
- 都用于修正高度标签
- 都能提前识别最小割

**不同之处**：

| 特性 | 全局重标记 | 间隙启发式 |
|------|-----------|-----------|
| **触发条件** | 工作量阈值 | 某层节点数变为 0 |
| **操作范围** | 所有节点 | 间隙以上的所有节点 |
| **高度更新** | 设为精确距离 | 设为 $n+1$（特殊标记） |
| **目的** | 提高推流效率 | 提前终止第一阶段 |

**互补关系**：
- 全局重标记**主动**修正高度，提高效率
- 间隙启发式**被动**检测异常，提前终止
- 两者结合使用效果最佳

---

## 12. 间隙启发式与两阶段终止

### 12.1 预流推进的两阶段设计

**整体架构**（`preflowpush.py:220-288`）：

```
┌─────────────────────────────────────────────────────────────┐
│                    预流推进算法主流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  初始化：                                                      │
│    - 从源点饱和推送所有出边                                    │
│    - 源点邻居获得超额流，成为活跃节点                          │
│    - 源点高度设为 n，汇点高度设为 0                            │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Phase 1：第一阶段（求最大预流）                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  while 存在高度 > 0 的活跃节点:                       │    │
│  │      选择最高层的活跃节点 u                            │    │
│  │      discharge(u):                                    │    │
│  │          - 尝试向低层邻居推流                          │    │
│  │          - 无法推流则重标记（增加高度）                 │    │
│  │          - 高度 >= n-1 时停止（在 S 侧）              │    │
│  │                                                        │    │
│  │      检查：                                            │    │
│  │      - 全局重标记阈值？→ 执行全局重标记               │    │
│  │      - 出现间隙？→ 应用间隙启发式，提前终止           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
│  第一阶段结束条件：                                             │
│    1. 无活跃节点（所有超额流已到达汇点或被"困"在 S 侧）       │
│    2. 间隙启发式触发（最小割已识别）                           │
│                                                               │
│  第一阶段结果：                                                 │
│    - R_nodes[t]["excess"] = 最大流值（已正确）               │
│    - 如果 value_only=True，可直接返回                         │
│    - 但流不是合法流（某些节点仍有超额流）                      │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Phase 2：第二阶段（预流转最大流）                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  前置操作：                                            │    │
│  │      global_relabel(False) → 从源点重标记           │    │
│  │      这样节点高度 = 到源点的距离 + n                  │    │
│  │      高度 > n 的节点在 S 侧，有超额流需要推送回源点    │    │
│  │                                                        │    │
│  │  while 存在高度 > n 的活跃节点:                       │    │
│  │      选择最高层的活跃节点 u                            │    │
│  │      discharge(u, is_phase1=False):                   │    │
│  │          - 无高度上限限制（可超过 2n）                │    │
│  │          - 持续推流直到超额流为 0                     │    │
│  │                                                        │    │
│  │      检查：                                            │    │
│  │      - 全局重标记阈值？→ 执行全局重标记               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
│  第二阶段结束条件：                                             │
│    - 无活跃节点（所有超额流已处理）                            │
│    - 源点和汇点可以有超额流（不违反流守恒）                     │
│                                                               │
│  第二阶段结果：                                                 │
│    - 合法最大流：所有中间节点流入 = 流出                       │
│    - 残差图可用于最小割提取                                     │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### 12.2 Level 数据结构设计

**实现**（`utils.py:51-58`）：

```python
class Level:
    """Active and inactive nodes in a level."""

    __slots__ = ("active", "inactive")

    def __init__(self):
        self.active = set()    # 活跃节点：有超额流需要推送
        self.inactive = set()  # 非活跃节点：无超额流
```

**初始化**（`preflowpush.py:104-112`）：

```python
# 创建 2n 个层级（高度范围 0 到 2n-1）
levels = [Level() for i in range(2 * n)]

# 将节点分配到对应层级
for u in R:
    if u != s and u != t:  # 源点和汇点不参与层级管理
        level = levels[R_nodes[u]["height"]]
        if R_nodes[u]["excess"] > 0:
            level.active.add(u)   # 有超额流 → 活跃
        else:
            level.inactive.add(u) # 无超额流 → 非活跃
```

**激活函数**（`preflowpush.py:114-120`）：

```python
def activate(v):
    """Move a node from the inactive set to the active set of its level."""
    if v != s and v != t:
        level = levels[R_nodes[v]["height"]]
        if v in level.inactive:
            level.inactive.remove(v)
            level.active.add(v)  # 成为活跃节点
```

**为什么需要分层？**

最高标签预流推进（Highest-Label Preflow-Push）的核心优化：
- **总是选择最高层的活跃节点进行 discharge**
- 这样可以最小化重标记次数
- 分层数据结构使得选择最高层节点成为 $O(1)$ 操作

### 12.3 间隙启发式的触发条件

**主循环中的检测**（`preflowpush.py:244-251`）：

```python
while height > 0:
    while True:
        # ... discharge 操作 ...
        height = discharge(u, True)
        
        if grt.is_reached():
            # 全局重标记
            height = global_relabel(True)
            max_height = height
            grt.clear_work()
        
        elif not old_level.active and not old_level.inactive:
            # 间隙启发式：old_height 层已空！
            gap_heuristic(old_height)
            height = old_height - 1
            max_height = height
        
        else:
            max_height = max(max_height, height)
```

**触发条件解析**：

```python
not old_level.active and not old_level.inactive
# 等价于
old_level.active == set() and old_level.inactive == set()
```

这意味着：
- `old_height` 层的活跃节点集合为空
- `old_height` 层的非活跃节点集合也为空
- **该层完全没有节点了！**

### 12.4 间隙启发式的实现

**核心函数**（`preflowpush.py:175-186`）：

```python
def gap_heuristic(height):
    """Apply the gap heuristic."""
    # 移动所有高度 > height 的节点到高度 n+1
    # 范围：height+1 到 max_height
    for level in islice(levels, height + 1, max_height + 1):
        # 处理活跃节点
        for u in level.active:
            R_nodes[u]["height"] = n + 1
        # 处理非活跃节点
        for u in level.inactive:
            R_nodes[u]["height"] = n + 1
        
        # 移动到 n+1 层
        levels[n + 1].active.update(level.active)
        level.active.clear()
        levels[n + 1].inactive.update(level.inactive)
        level.inactive.clear()
```

### 12.5 间隙启发式的正确性证明

**为什么间隙意味着最小割？**

**高度标签的性质**：
- 初始化时：`height[t] = 0`，`height[s] = n`
- 对于任何许可边 $u \to v$（有残余容量且 `height[u] == height[v] + 1`）：
  - 如果 `height[u] < n$，则 $u$ 可能在 $S$ 侧或 $T$ 侧
  - 如果 `height[u] >= n$，则 $u$ 在 $S$ 侧（无法通过最短路径到达汇点）

**间隙的含义**：
```
高度分布（示例）：

height=5:  [节点A, 节点B]  ← 活跃
height=4:  []  ← 空！这就是间隙
height=3:  [节点C]
height=2:  [节点D]
height=1:  [节点E]
height=0:  [t]
```

如果高度 `h` 层没有节点：
- 所有高度 `> h` 的节点无法到达高度 `< h` 的节点
- 因为任何路径都需要经过高度递减的许可边
- 但高度 `h` 层为空，无法"跳跃"

**最小割分区**：
- $S$ = 高度 `> h` 的所有节点（包括源点）
- $T$ = 高度 `<= h` 的所有节点（包括汇点）
- 任何从 $S$ 到 $T$ 的边都已饱和（否则许可边会连接两层）

**特殊高度 `n+1`**：
```python
# 间隙启发式将间隙以上的节点设为 n+1
R_nodes[u]["height"] = n + 1
```

这是一个**特殊标记**：
- 表示该节点在 $S$ 侧
- 在第一阶段不再处理（因为 `is_phase1 and height >= n - 1` 时停止）
- 其超额流将在第二阶段被推送回源点

### 12.6 第一阶段的终止条件

**主循环**（`preflowpush.py:223-256`）：

```python
# Phase 1: 推送尽可能多的流量到汇点
height = max_height
while height > 0:
    while True:
        level = levels[height]
        if not level.active:
            # 当前层无活跃节点，下降一层
            height -= 1
            break
        
        # 记录旧层用于间隙检测
        old_height = height
        old_level = level
        
        # 选择一个活跃节点进行 discharge
        u = arbitrary_element(level.active)
        height = discharge(u, True)  # is_phase1=True
        
        # 检查启发式
        if grt.is_reached():
            height = global_relabel(True)
            max_height = height
            grt.clear_work()
        
        elif not old_level.active and not old_level.inactive:
            # 间隙出现！
            gap_heuristic(old_height)
            height = old_height - 1
            max_height = height
        
        else:
            max_height = max(max_height, height)
```

**终止条件详解**：

| 条件 | 含义 | 处理 |
|------|------|------|
| `height > 0` | 仍有节点在 1..max_height 层 | 继续循环 |
| `level.active` 为空 | 当前层无活跃节点 | 下降一层 `height -= 1` |
| 间隙出现 | 某层完全为空 | 应用间隙启发式，下降到 `old_height - 1` |
| `discharge` 返回 `height >= n-1` | 节点在 S 侧，第一阶段不处理 | 移到高层，继续处理其他节点 |

**第一阶段结束的含义**：
- 所有活跃节点要么在汇点（超额流已接收），要么在高度 `>= n` 层（在 S 侧）
- 最小割已隐式确定：$S$ = 高度 `>= n` 的节点，$T$ = 高度 `< n` 的节点

### 12.7 第二阶段：超额流的回收

**为什么需要第二阶段？**

第一阶段结束时：
- 汇点的超额流 = 最大流值（正确）
- 但某些中间节点可能仍有超额流（违反流守恒）
- 这些节点在 $S$ 侧，无法到达汇点

**第二阶段目标**：
- 将 $S$ 侧节点的超额流推送回源点
- 使所有中间节点满足流入 = 流出

**第二阶段实现**（`preflowpush.py:263-287`）：

```python
# Phase 2: 将最大预流转换为最大流
# 通过将超额流推送回源点

# 前置操作：从源点进行全局重标记
height = global_relabel(False)  # from_sink=False
grt.clear_work()

# 第二阶段循环：处理高度 > n 的节点（在 S 侧）
while height > n:
    while True:
        level = levels[height]
        if not level.active:
            # 当前层无活跃节点，下降一层
            height -= 1
            break
        
        # 选择活跃节点 discharge
        u = arbitrary_element(level.active)
        height = discharge(u, False)  # is_phase1=False
        
        if grt.is_reached():
            # 全局重标记
            height = global_relabel(False)
            grt.clear_work()

# 第二阶段结束
R.graph["flow_value"] = R_nodes[t]["excess"]
return R
```

**关键差异：`is_phase1=False`**

在 `discharge` 函数中（`preflowpush.py:161-166`）：

```python
if is_phase1 and height >= n - 1:
    # 第一阶段：高度 >= n-1 的节点停止处理
    # 因为它们在 S 侧，无法到达汇点
    levels[height].active.add(u)
    break
# 第二阶段：无此检查，继续推流直到超额流为 0
```

**第二阶段的推流方向**：

第一阶段：
- 目标：推送流量到汇点
- 许可边：`height[u] == height[v] + 1`（向低层推流）
- 低层 = 更接近汇点

第二阶段（从源点重标记后）：
- 高度 = 到源点的距离 + $n$
- 低层 = 更接近源点
- 向低层推流 = 推送回源点

**示例**：
```
从源点重标记后：
s: height = 0 + n = n
a: height = 1 + n = n+1  (距离 s 为 1)
b: height = 2 + n = n+2  (距离 s 为 2)

许可边 s <- a <- b（向低层推流 = 向源点推流）
```

### 12.8 value_only 参数的优化

**参数定义**（`preflowpush.py:294-346`）：

```python
def preflow_push(
    G, s, t, capacity="capacity", residual=None, 
    global_relabel_freq=1, value_only=False
):
    """
    value_only : bool
        If False, compute a maximum flow; 
        otherwise, compute a maximum preflow which is enough for 
        computing the maximum flow value. Default value: False.
    """
```

**优化点**（`preflowpush.py:257-261`）：

```python
# 第一阶段结束
# R_nodes[t]["excess"] 已是最大流值
if value_only:
    R.graph["flow_value"] = R_nodes[t]["excess"]
    return R  # 直接返回，跳过第二阶段！
```

**为什么这是正确的？**

最大流-最小割定理：
- 最大流值 = 最小割容量
- 第一阶段结束时，最小割已确定
- 汇点的超额流 = 已推送的总流量 = 最大流值
- 第二阶段只是将"被困"在 S 侧的流量推送回源点，不改变汇点的超额流

**适用场景**：
- 只需要最大流值（不需要完整流分配）
- 只需要最小割分区（可从残差图提取）
- `minimum_cut` 函数调用时设置 `value_only=True`

### 12.9 两阶段设计的性能收益

| 阶段 | 目标 | 终止条件 | 性能优化 |
|------|------|---------|---------|
| **Phase 1** | 求最大预流 | 无活跃节点 或 间隙出现 | 间隙启发式提前终止，`value_only` 可跳过 Phase 2 |
| **Phase 2** | 预流转合法流 | 无活跃节点 | 只处理 S 侧节点，工作量减少 |

**间隙启发式的收益**：
- 可能在算法早期就识别最小割
- 避免不必要的推流和重标记操作
- 对于某些图结构（如网格图），间隙可能很早就出现

**value_only 的收益**：
- 完全跳过第二阶段
- 对于最小割计算，这是巨大的优化
- 因为 `minimum_cut` 只需要流值和残差图结构

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
