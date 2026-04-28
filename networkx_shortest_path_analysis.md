# NetworkX 最短路径算法族设计分析

## 1. 统一接口设计与分发机制

### 1.1 模块组织结构

NetworkX 的最短路径算法采用了**分层模块化设计**，通过 `__init__.py` 统一导出所有接口：

```python
# networkx/algorithms/shortest_paths/__init__.py
from networkx.algorithms.shortest_paths.generic import *
from networkx.algorithms.shortest_paths.unweighted import *
from networkx.algorithms.shortest_paths.weighted import *
from networkx.algorithms.shortest_paths.astar import *
from networkx.algorithms.shortest_paths.dense import *
```

### 1.2 核心统一接口

`generic.py` 提供了**高层统一接口**，用户无需关心底层算法实现细节：

| 接口函数 | 功能描述 |
|---------|---------|
| `shortest_path()` | 计算最短路径（返回节点列表） |
| `shortest_path_length()` | 计算最短路径长度 |
| `average_shortest_path_length()` | 计算平均最短路径长度 |
| `all_shortest_paths()` | 计算所有最短路径（返回生成器） |
| `single_source_all_shortest_paths()` | 单源到所有节点的所有最短路径 |
| `all_pairs_all_shortest_paths()` | 所有点对的所有最短路径 |

### 1.3 智能分发机制

`shortest_path()` 函数实现了**基于参数的智能算法选择**：

#### 1.3.1 算法选择逻辑

```python
# generic.py:42-178
@nx._dispatchable(edge_attrs="weight")
def shortest_path(G, source=None, target=None, weight=None, method="dijkstra"):
    # 1. 方法参数验证
    if method not in ("dijkstra", "bellman-ford"):
        raise ValueError(f"method not supported: {method}")
    
    # 2. 核心分发规则：根据 weight 参数自动选择
    method = "unweighted" if weight is None else method
```

**分发决策树：**

```
                    shortest_path()
                         │
                    weight is None?
                    ┌────────┴────────┐
                    │                  │
                  Yes                 No
                    │                  │
              method = "unweighted"  method = "dijkstra" or "bellman-ford"
                    │                  │
              使用 BFS 算法         使用带权图算法
```

#### 1.3.2 场景化分发

根据 `source` 和 `target` 的组合，选择不同的调用策略：

| source | target | 调用策略 | 底层函数 |
|--------|--------|---------|---------|
| None | None | 全点对最短路径 | `all_pairs_*` |
| None | 指定 | 全源单目标（有向图反转） | `single_source_*` + 路径反转 |
| 指定 | None | 单源全目标 | `single_source_*` |
| 指定 | 指定 | 单源单目标 | 双向搜索优化版本 |

**关键代码片段（有向图单目标处理）：**

```python
# generic.py:148-160
else:  # target 指定，source 为 None
    # Find paths from all nodes co-accessible to the target.
    if G.is_directed():
        G = G.reverse(copy=False)  # 反转图，将单目标转为单源
    if method == "unweighted":
        paths = nx.single_source_shortest_path(G, target)
    elif method == "dijkstra":
        paths = nx.single_source_dijkstra_path(G, target, weight=weight)
    else:  # method == 'bellman-ford':
        paths = nx.single_source_bellman_ford_path(G, target, weight=weight)
    # 反转路径恢复原始方向
    for target in paths:
        paths[target] = list(reversed(paths[target]))
```

#### 1.3.3 单源单目标优化

对于明确的源点和目标点，使用**双向搜索**进一步优化：

```python
# generic.py:170-177
else:  # source 和 target 都指定
    # Find shortest source-target path.
    if method == "unweighted":
        paths = nx.bidirectional_shortest_path(G, source, target)
    elif method == "dijkstra":
        _, paths = nx.bidirectional_dijkstra(G, source, target, weight)
    else:  # method == 'bellman-ford':
        paths = nx.bellman_ford_path(G, source, target, weight)
```

---

## 2. 有向/无向图与带权/不带权图的适配

### 2.1 无权图算法（BFS 家族）

#### 2.1.1 核心实现

**标准 BFS 实现**（`unweighted.py:83-113`）：

```python
def _single_shortest_path_length(adj, firstlevel, cutoff):
    """Yields (node, level) in a breadth first search"""
    seen = set(firstlevel)
    nextlevel = firstlevel
    level = 0
    n = len(adj)
    for v in nextlevel:
        yield (v, level)
    while nextlevel and cutoff > level:
        level += 1
        thislevel = nextlevel
        nextlevel = []
        for v in thislevel:
            for w in adj[v]:
                if w not in seen:
                    seen.add(w)
                    nextlevel.append(w)
                    yield (w, level)
            if len(seen) == n:
                return
```

#### 2.1.2 有向/无向图适配策略

**单目标 BFS 的适配**（`unweighted.py:116-172`）：

```python
@nx._dispatchable
def single_target_shortest_path_length(G, target, cutoff=None):
    """Compute the shortest path lengths to target from all reachable nodes."""
    if target not in G:
        raise nx.NodeNotFound(f"Target {target} is not in G")
    if cutoff is None:
        cutoff = float("inf")
    
    # 关键适配：根据图类型选择不同的邻接表视图
    # 有向图：使用 _pred（前驱节点）进行反向搜索
    # 无向图：使用 _adj（邻接节点），因为无向边是双向的
    adj = G._pred if G.is_directed() else G._adj
    nextlevel = [target]
    return dict(_single_shortest_path_length(adj, nextlevel, cutoff))
```

**双向 BFS 的适配**（`unweighted.py:292-341`）：

```python
def _bidirectional_pred_succ(G, source, target):
    """Bidirectional shortest path helper."""
    # 处理有向图和无向图的邻居访问
    if G.is_directed():
        Gpred = G.pred   # 有向图：前驱节点用于反向搜索
        Gsucc = G.succ   # 有向图：后继节点用于正向搜索
    else:
        Gpred = G.adj    # 无向图：两者相同
        Gsucc = G.adj
    
    # 前驱和后继字典
    pred = {source: None}  # 正向搜索：记录每个节点的前驱
    succ = {target: None}  # 反向搜索：记录每个节点的后继
    
    # 初始化搜索边界
    forward_fringe = [source]
    reverse_fringe = [target]
    
    while forward_fringe and reverse_fringe:
        # 选择较小的边界进行扩展（优化策略）
        if len(forward_fringe) <= len(reverse_fringe):
            # 正向扩展
            this_level = forward_fringe
            forward_fringe = []
            for v in this_level:
                for w in Gsucc[v]:  # 使用后继访问
                    if w not in pred:
                        forward_fringe.append(w)
                        pred[w] = v
                    if w in succ:  # 两个搜索相遇
                        return pred, succ, w
        else:
            # 反向扩展
            this_level = reverse_fringe
            reverse_fringe = []
            for v in this_level:
                for w in Gpred[v]:  # 使用前驱访问
                    if w not in succ:
                        succ[w] = v
                        reverse_fringe.append(w)
                    if w in pred:  # 两个搜索相遇
                        return pred, succ, w
```

#### 2.1.3 无权图算法体系

| 函数 | 功能 | 适用场景 |
|------|------|---------|
| `single_source_shortest_path()` | 单源最短路径 | 从一个源点到所有可达节点 |
| `single_target_shortest_path()` | 单目标最短路径 | 从所有可达节点到一个目标点 |
| `bidirectional_shortest_path()` | 双向 BFS | 单源单目标的优化搜索 |
| `all_pairs_shortest_path()` | 全点对最短路径 | 所有节点对的最短路径 |
| `predecessor()` | 前驱字典 | 用于重构所有最短路径 |

### 2.2 带权图算法（Dijkstra / Bellman-Ford 家族）

#### 2.2.1 权重函数抽象

`_weight_function` 实现了**权重的统一抽象**，支持三种权重表示方式：

```python
# weighted.py:41-78
def _weight_function(G, weight):
    """Returns a function that returns the weight of an edge."""
    if callable(weight):
        # 情况1：用户自定义权重函数
        # 签名：weight(u, v, edge_attr_dict) -> number or None
        return weight
    
    # 情况2：字符串指定边属性名
    if G.is_multigraph():
        # 多重图：选择平行边中的最小权重
        return lambda u, v, d: min(attr.get(weight, 1) for attr in d.values())
    # 简单图：直接获取边属性，不存在则默认为 1
    return lambda u, v, data: data.get(weight, 1)
```

**权重函数的高级用法：**

```python
# 示例1：使用自定义权重函数隐藏某些边
weight = lambda u, v, d: 1 if d['color']=="red" else None

# 示例2：包含节点权重的边权重
def func(u, v, d):
    node_u_wt = G.nodes[u].get("node_weight", 1)
    node_v_wt = G.nodes[v].get("node_weight", 1)
    edge_wt = d.get("weight", 1)
    return node_u_wt / 2 + node_v_wt / 2 + edge_wt
```

#### 2.2.2 Dijkstra 算法的图类型适配

**核心 Dijkstra 实现**（`weighted.py:784-907`）：

```python
def _dijkstra_multisource(
    G, sources, weight, pred=None, paths=None, cutoff=None, target=None
):
    """Uses Dijkstra's algorithm to find shortest weighted paths"""
    
    # 关键：使用 G._adj 统一访问，同时支持有向图和无向图
    # 对于有向图，G._adj == G._succ（后继节点）
    # 对于无向图，G._adj 就是邻接表
    G_succ = G._adj  # For speed-up (and works for both directed and undirected graphs)
    
    dist = {}  # 最终距离字典
    seen = {}   # 已访问节点的临时距离
    
    # 优先队列：(distance, counter, node)
    # counter 用于避免节点比较
    c = count()
    fringe = []
    
    # 初始化多源点
    for source in sources:
        seen[source] = 0
        heappush(fringe, (0, next(c), source))
    
    while fringe:
        (dist_v, _, v) = heappop(fringe)
        
        # 懒删除：如果节点已有最终距离，跳过
        if v in dist:
            continue
        dist[v] = dist_v
        
        # 提前终止：找到目标节点
        if v == target:
            break
        
        # 遍历邻居
        for u, e in G_succ[v].items():
            cost = weight(v, u, e)
            if cost is None:
                continue  # 跳过隐藏边
            
            vu_dist = dist_v + cost
            
            # 截止距离优化
            if cutoff is not None and vu_dist > cutoff:
                continue
            
            # 负权检测
            if u in dist:
                u_dist = dist[u]
                if vu_dist < u_dist:
                    raise ValueError("Contradictory paths found:", "negative weights?")
                elif pred is not None and vu_dist == u_dist:
                    # 记录所有最短路径的前驱
                    pred_dict[u].append(v)
            
            # 更新距离
            elif u not in seen or vu_dist < seen[u]:
                seen[u] = vu_dist
                heappush(fringe, (vu_dist, next(c), u))
                if pred_dict is not None:
                    pred_dict[u] = [v]
            elif pred is not None and vu_dist == seen[u]:
                # 等长路径，添加前驱
                pred_dict[u].append(v)
```

#### 2.2.3 双向 Dijkstra 的图类型适配

**双向 Dijkstra 实现**（`weighted.py:2310-2459`）：

```python
@nx._dispatchable(edge_attrs="weight")
def bidirectional_dijkstra(G, source, target, weight="weight"):
    """Dijkstra's algorithm for shortest paths using bidirectional search."""
    
    # 邻居访问策略：根据图类型选择不同的视图
    if G.is_directed():
        # 有向图：
        # - 正向搜索：使用 _succ（后继节点）
        # - 反向搜索：使用 _pred（前驱节点）
        neighbors = [G._succ, G._pred]
    else:
        # 无向图：两者相同，都使用 _adj
        neighbors = [G._adj, G._adj]
    
    # 双向搜索的状态管理
    dists = [{}, {}]           # [正向最终距离, 反向最终距离]
    preds = [{source: None}, {target: None}]  # [正向前驱, 反向后继]
    fringe = [[], []]          # [正向优先队列, 反向优先队列]
    seen = [{source: 0}, {target: 0}]  # [正向临时距离, 反向临时距离]
    
    # 初始化两个方向的优先队列
    c = count()
    heappush(fringe[0], (0, next(c), source))
    heappush(fringe[1], (0, next(c), target))
    
    finaldist = None
    meetnode = None
    direction = 1  # 1 表示下次选择反向搜索
    
    while fringe[0] and fringe[1]:
        # 交替选择方向进行扩展
        direction = 1 - direction  # 0=正向, 1=反向
        
        # 弹出当前方向的最小距离节点
        (dist, _, v) = heappop(fringe[direction])
        
        # 懒删除：已处理过的节点跳过
        if v in dists[direction]:
            continue
        
        dists[direction][v] = dist
        
        # 相遇检测：如果节点在另一个方向已处理
        if v in dists[1 - direction]:
            # 两个方向都处理过此节点，找到最短路径
            return (finaldist, path(meetnode, 0) + path(preds[1][meetnode], 1))
        
        # 扩展邻居
        for w, d in neighbors[direction][v].items():
            # 权重计算：正向和反向方向不同
            if direction == 0:
                # 正向：u -> v，权重为 weight(u, v, data)
                cost = weight(v, w, d)
            else:
                # 反向：搜索是从 target 到 source，但边是 u -> v
                # 所以实际权重应该是 weight(w, v, data)
                cost = weight(w, v, d)
            
            if cost is None:
                continue
            
            vwLength = dist + cost
            
            # 负权检测
            if w in dists[direction]:
                if vwLength < dists[direction][w]:
                    raise ValueError("Contradictory paths found: negative weights?")
            
            # 更新距离
            elif w not in seen[direction] or vwLength < seen[direction][w]:
                seen[direction][w] = vwLength
                heappush(fringe[direction], (vwLength, next(c), w))
                preds[direction][w] = v
                
                # 潜在相遇点：记录更短的路径
                if w in seen[1 - direction]:
                    finaldist_w = vwLength + seen[1 - direction][w]
                    if finaldist is None or finaldist > finaldist_w:
                        finaldist, meetnode = finaldist_w, w
```

#### 2.2.4 Bellman-Ford 算法的图类型适配

**Bellman-Ford 核心实现**（`weighted.py:1389-1510`）：

```python
def _inner_bellman_ford(
    G,
    sources,
    weight,
    pred,
    dist=None,
    heuristic=True,
):
    """Inner Relaxation loop for Bellman–Ford algorithm.
    
    This is an implementation of the SPFA variant.
    See https://en.wikipedia.org/wiki/Shortest_Path_Faster_Algorithm
    """
    
    # 统一使用 G._adj，同时支持有向图和无向图
    G_succ = G._adj  # For speed-up (and works for both directed and undirected graphs)
    
    inf = float("inf")
    n = len(G)
    
    # SPFA 优化：使用队列管理待松弛节点
    count = {}       # 节点入队次数（用于负环检测）
    q = deque(sources)
    in_q = set(sources)
    
    while q:
        u = q.popleft()
        in_q.remove(u)
        
        # 优化：如果前驱还在队列中，跳过当前节点
        # 因为前驱的松弛会更有效
        if all(pred_u not in in_q for pred_u in pred[u]):
            dist_u = dist[u]
            
            # 遍历所有出边进行松弛
            for v, e in G_succ[u].items():
                dist_v = dist_u + weight(u, v, e)
                
                if dist_v < dist.get(v, inf):
                    # 成功松弛
                    
                    # 负环启发式检测
                    if heuristic:
                        if v in recent_update[u]:
                            # 发现负环：节点在更新路径上出现两次
                            pred[v].append(u)
                            return v
                    
                    # 入队管理
                    if v not in in_q:
                        q.append(v)
                        in_q.add(v)
                        count_v = count.get(v, 0) + 1
                        
                        # 标准负环检测：入队次数 >= 节点数
                        if count_v == n:
                            return v
                        
                        count[v] = count_v
                    
                    # 更新距离和前驱
                    dist[v] = dist_v
                    pred[v] = [u]
                
                elif dist.get(v) is not None and dist_v == dist.get(v):
                    # 等长路径，添加前驱
                    pred[v].append(u)
    
    # 没有负环
    return None
```

---

## 3. Dijkstra 算法的优先队列选择与懒删除机制

### 3.1 优先队列的设计选择

#### 3.1.1 队列元素结构

NetworkX 的 Dijkstra 实现使用 Python 标准库的 `heapq` 模块，队列元素采用**三元组结构**：

```python
# weighted.py:845-853
# fringe is heapq with 3-tuples (distance,c,node)
# use the count c to avoid comparing nodes (may not be able to)
c = count()
fringe = []
for source in sources:
    seen[source] = 0
    heappush(fringe, (0, next(c), source))
```

**设计原因分析：**

| 组件 | 作用 | 必要性 |
|------|------|--------|
| `distance` | 优先队列的主键，决定弹出顺序 | 必须（Dijkstra 核心） |
| `count` | 唯一计数器，避免节点比较 | 必须（解决节点不可比较问题） |
| `node` | 实际的节点对象 | 必须 |

**为什么需要 `count`？**

```python
# 问题场景：节点可能不可比较
class CustomNode:
    def __init__(self, name):
        self.name = name

# 如果队列元素是 (distance, node)
# 当两个节点距离相同时，heapq 会尝试比较节点
# 如果节点没有定义 __lt__，会抛出 TypeError

# 解决方案：插入唯一计数器
# 当距离相同时，比较计数器（总是可比较的整数）
heappush(fringe, (distance, next(c), node))
```

#### 3.1.2 与标准 Dijkstra 的对比

**标准 Dijkstra 伪代码：**
```
1. dist[source] = 0, dist[others] = ∞
2. Q = 所有节点（优先级队列）
3. while Q 非空:
4.     u = 抽取最小距离节点
5.     for 每个邻居 v:
6.         if dist[v] > dist[u] + weight(u,v):
7.             dist[v] = dist[u] + weight(u,v)
8.             减小 v 的优先级（DECREASE-KEY 操作）
```

**NetworkX 的实现差异：**

```python
# NetworkX 使用"多次插入 + 懒删除"策略
# 第7步不执行 DECREASE-KEY，而是直接插入新条目
elif u not in seen or vu_dist < seen[u]:
    seen[u] = vu_dist
    heappush(fringe, (vu_dist, next(c), u))  # 直接插入新条目
```

### 3.2 懒删除（Lazy Deletion）机制详解

#### 3.2.1 核心实现

```python
# weighted.py:855-859
while fringe:
    (dist_v, _, v) = heappop(fringe)
    
    # 懒删除的关键检查
    if v in dist:
        continue  # 已找到更短路径，跳过此过期条目
    
    # 否则，这是最短路径
    dist[v] = dist_v
```

#### 3.2.2 工作原理

**场景示例：**

假设有一个图 `A --2--> B --1--> C`，同时 `A --5--> C`

```
初始状态：
  fringe = [(0, 0, A)]
  dist = {}
  seen = {A: 0}

步骤1：弹出 (0, 0, A)
  - A 不在 dist 中，处理
  - dist[A] = 0
  - 邻居 B: 距离 0+2=2，插入 fringe = [(2, 1, B)]
  - 邻居 C: 距离 0+5=5，插入 fringe = [(2, 1, B), (5, 2, C)]
  - seen = {A: 0, B: 2, C: 5}

步骤2：弹出 (2, 1, B)
  - B 不在 dist 中，处理
  - dist[B] = 2
  - 邻居 C: 距离 2+1=3 < 5，发现更短路径！
  - 不执行 DECREASE-KEY，直接插入新条目
  - heappush(fringe, (3, 3, C))
  - seen[C] = 3
  - 现在 fringe = [(3, 3, C), (5, 2, C)]  <-- 注意：有两个 C 的条目！

步骤3：弹出 (3, 3, C)
  - C 不在 dist 中，处理
  - dist[C] = 3
  - 处理完毕

步骤4：弹出 (5, 2, C)
  - C 已在 dist 中（dist[C] = 3 < 5）
  - 执行懒删除：continue 跳过
  - 此过期条目被忽略
```

#### 3.2.3 设计权衡

| 策略 | 优点 | 缺点 |
|------|------|------|
| **懒删除（NetworkX 采用）** | 1. 实现简单，无需复杂的优先级队列<br>2. 利用 Python heapq 的标准操作<br>3. 代码可读性高 | 1. 优先队列可能包含过期条目<br>2. 内存占用略高<br>3. 弹出操作次数略多 |
| **标准 DECREASE-KEY** | 1. 优先队列大小始终可控<br>2. 理论时间复杂度更优 | 1. 需要支持 DECREASE-KEY 的特殊堆（如 Fibonacci 堆）<br>2. Python 标准库不支持<br>3. 实现复杂 |

#### 3.2.4 时间复杂度分析

**理论复杂度：**
- 标准 Dijkstra（Fibonacci 堆）：$O(V \log V + E)$
- NetworkX 实现（二叉堆 + 懒删除）：$O(E \log E)$

**实际表现：**
- 每条边可能导致一次堆插入：$O(E)$ 次插入
- 每次插入 $O(\log E)$ 时间
- 弹出次数最多 $O(E)$ 次，每次 $O(\log E)$
- 实际中，过期条目通常不多，性能接近最优

### 3.3 多源 Dijkstra 扩展

NetworkX 的 Dijkstra 实现**原生支持多源点**：

```python
# weighted.py:784-854
def _dijkstra_multisource(
    G, sources, weight, pred=None, paths=None, cutoff=None, target=None
):
    """Uses Dijkstra's algorithm to find shortest weighted paths
    
    Parameters
    ----------
    sources : non-empty iterable of nodes
        Starting nodes for paths. If this is just an iterable containing
        a single node, then all paths computed by this function will
        start from that node. If there are two or more nodes in this
        iterable, the computed paths may begin from any one of the start
        nodes.
    """
    # ...
    for source in sources:
        seen[source] = 0
        heappush(fringe, (0, next(c), source))
    # ...
```

**应用场景：**
- 计算从多个源点到所有节点的最短距离
- 例如：计算城市中所有消防站到任意点的最短距离

---

## 4. 其他算法实现分析

### 4.1 A* 算法

#### 4.1.1 核心实现

A* 算法是 Dijkstra 算法的**启发式优化版本**：

```python
# astar.py:12-171
@nx._dispatchable(edge_attrs="weight", preserve_node_attrs="heuristic")
def astar_path(G, source, target, heuristic=None, weight="weight", *, cutoff=None):
    """Returns a list of nodes in a shortest path between source and target
    using the A* ("A-star") algorithm."""
    
    # 默认启发式函数：h=0，退化为 Dijkstra
    if heuristic is None:
        def heuristic(u, v):
            return 0
    
    weight = _weight_function(G, weight)
    G_succ = G._adj
    
    # 队列元素：(f_cost, counter, node, g_cost, parent)
    # f_cost = g_cost + h_cost（实际代价 + 启发式估计）
    c = count()
    queue = [(0, next(c), source, 0, None)]
    
    # 已入队节点的信息：{node: (g_cost, h_cost)}
    enqueued = {}
    # 已探索节点的父节点：{node: parent}
    explored = {}
    
    while queue:
        # 弹出 f_cost 最小的节点
        _, __, curnode, dist, parent = heappop(queue)
        
        # 找到目标
        if curnode == target:
            # 重构路径
            path = [curnode]
            node = parent
            while node is not None:
                path.append(node)
                node = explored[node]
            path.reverse()
            return path
        
        # 懒删除检查
        if curnode in explored:
            if explored[curnode] is None:
                continue  # 源点已处理
            # 检查是否有更短的路径已入队
            qcost, h = enqueued[curnode]
            if qcost < dist:
                continue
        
        explored[curnode] = parent
        
        # 扩展邻居
        for neighbor, w in G_succ[curnode].items():
            cost = weight(curnode, neighbor, w)
            if cost is None:
                continue
            
            ncost = dist + cost  # g_cost 实际代价
            
            # 检查是否需要更新
            if neighbor in enqueued:
                qcost, h = enqueued[neighbor]
                if qcost <= ncost:
                    continue  # 已有更短路径
            else:
                h = heuristic(neighbor, target)  # 计算启发式估计
            
            # 截止优化
            if cutoff and ncost + h > cutoff:
                continue
            
            # 入队
            enqueued[neighbor] = ncost, h
            heappush(queue, (ncost + h, next(c), neighbor, ncost, curnode))
    
    raise nx.NetworkXNoPath(f"Node {target} not reachable from {source}")
```

#### 4.1.2 A* 与 Dijkstra 的对比

| 特性 | Dijkstra | A* |
|------|----------|-----|
| 优先级键 | `g(n)`（实际代价） | `f(n) = g(n) + h(n)` |
| 启发式 | 无 | 必须有（默认 h=0） |
| 完备性 | 保证找到最短路径 | 当启发式可采纳时保证 |
| 最优性 | 保证 | 当启发式可采纳且一致时保证 |
| 适用场景 | 所有带权图（非负权） | 有好的启发式函数的场景（如网格路径） |

**可采纳性（Admissibility）：**
- 启发式函数 $h(n)$ 永远不会高估到目标的实际代价
- 即 $h(n) \leq h^*(n)$，其中 $h^*(n)$ 是实际最短距离

**一致性（Consistency）：**
- 对于任意节点 $n$ 和其后继 $n'$，满足：
  $h(n) \leq c(n, n') + h(n')$
- 一致性蕴含可采纳性

### 4.2 Bellman-Ford 算法（SPFA 变体）

#### 4.2.1 核心特性

Bellman-Ford 算法的**独特能力**：
1. 支持**负权边**（Dijkstra 不支持）
2. 能够**检测负环**（负权环使最短路径无意义）

#### 4.2.2 SPFA 优化

标准 Bellman-Ford 时间复杂度为 $O(VE)$，SPFA（Shortest Path Faster Algorithm）通过队列优化显著提升实际性能：

```python
# weighted.py:1458-1500
# SPFA 优化：使用队列管理待松弛节点
count = {}       # 节点入队次数（用于负环检测）
q = deque(sources)
in_q = set(sources)

while q:
    u = q.popleft()
    in_q.remove(u)
    
    # 额外优化：如果前驱还在队列中，跳过当前节点
    if all(pred_u not in in_q for pred_u in pred[u]):
        dist_u = dist[u]
        
        for v, e in G_succ[u].items():
            dist_v = dist_u + weight(u, v, e)
            
            if dist_v < dist.get(v, inf):
                # 成功松弛，入队
                if v not in in_q:
                    q.append(v)
                    in_q.add(v)
                    count_v = count.get(v, 0) + 1
                    
                    # 负环检测：入队次数 >= 节点数
                    if count_v == n:
                        return v  # 发现负环
                    
                    count[v] = count_v
                
                dist[v] = dist_v
                pred[v] = [u]
```

#### 4.2.3 负环检测机制

**两种负环检测策略：**

1. **标准策略**（入队次数）：
   ```python
   count_v = count.get(v, 0) + 1
   if count_v == n:  # n 是节点数
       return v  # 发现负环
   ```
   - 理论依据：最短路径最多包含 $V-1$ 条边
   - 如果节点入队 $V$ 次，说明存在负环

2. **启发式策略**（更新路径追踪）：
   ```python
   # weighted.py:1478-1491
   if heuristic:
       if v in recent_update[u]:
           # 发现负环：节点在更新路径上出现两次
           pred[v].append(u)
           return v
       
       # 追踪更新路径
       if v in pred_edge and pred_edge[v] == u:
           recent_update[v] = recent_update[u]
       else:
           recent_update[v] = (u, v)
   ```
   - 更快速的负环检测
   - 通过追踪松弛路径，检测是否形成环

#### 4.2.4 负环查找

```python
# weighted.py:2214-2306
def find_negative_cycle(G, source, weight="weight"):
    """Returns a cycle with negative total weight if it exists."""
    weight = _weight_function(G, weight)
    pred = {source: []}
    
    # 运行 Bellman-Ford 检测负环
    v = _inner_bellman_ford(G, [source], weight, pred=pred)
    if v is None:
        raise nx.NetworkXError("No negative cycles detected.")
    
    # 从前驱字典中重构负环
    neg_cycle = []
    stack = [(v, list(pred[v]))]
    seen = {v}
    
    while stack:
        node, preds = stack[-1]
        if v in preds:
            # 找到环
            neg_cycle.extend([node, v])
            neg_cycle = list(reversed(neg_cycle))
            return neg_cycle
        
        if preds:
            nbr = preds.pop()
            if nbr not in seen:
                stack.append((nbr, list(pred[nbr])))
                neg_cycle.append(node)
                seen.add(nbr)
        else:
            stack.pop()
            if neg_cycle:
                neg_cycle.pop()
```

### 4.3 Johnson 算法

Johnson 算法是**全点对最短路径**算法，特别适合**稀疏图**：

```python
# weighted.py:2462-2542
@nx._dispatchable(edge_attrs="weight")
def johnson(G, weight="weight"):
    r"""Uses Johnson's Algorithm to compute shortest paths.
    
    Johnson's Algorithm finds a shortest path between each pair of
    nodes in a weighted graph even if negative weights are present.
    """
    # 步骤1：添加超级源点，连接到所有其他节点
    # （在 _bellman_ford 内部处理）
    
    # 步骤2：用 Bellman-Ford 计算重赋权值 h(v)
    dist = {v: 0 for v in G}
    pred = {v: [] for v in G}
    weight = _weight_function(G, weight)
    
    dist_bellman = _bellman_ford(G, list(G), weight, pred=pred, dist=dist)
    
    # 步骤3：重赋权，消除负权
    # new_weight(u, v) = weight(u, v) + h(u) - h(v)
    # 这样所有边权变为非负
    def new_weight(u, v, d):
        return weight(u, v, d) + dist_bellman[u] - dist_bellman[v]
    
    # 步骤4：对每个节点运行 Dijkstra
    def dist_path(v):
        paths = {v: [v]}
        _dijkstra(G, v, new_weight, paths=paths)
        return paths
    
    return {v: dist_path(v) for v in G}
```

**Johnson 算法的时间复杂度：**
- Bellman-Ford 重赋权：$O(VE)$
- $V$ 次 Dijkstra：$O(V E \log V)$（二叉堆）
- 总复杂度：$O(V E \log V)$

**与 Floyd-Warshall 对比：**
- Floyd-Warshall：$O(V^3)$，适合稠密图
- Johnson：$O(V E \log V)$，适合稀疏图

---

## 5. 总结与设计亮点

### 5.1 架构设计总结

```
┌─────────────────────────────────────────────────────────────┐
│                      用户层接口                               │
│  shortest_path(), shortest_path_length(), all_shortest_paths()│
└─────────────────────────┬───────────────────────────────────┘
                          │ 参数分发
              ┌───────────┴───────────┐
              │                       │
        weight is None           weight specified
              │                       │
              ▼                       ▼
    ┌─────────────────┐    ┌─────────────────────────┐
    │   无权图算法    │    │      带权图算法         │
    │  (BFS 家族)    │    │  (Dijkstra/Bellman-Ford)│
    └────────┬────────┘    └───────────┬─────────────┘
             │                          │
    ┌────────┴────────┐      ┌─────────┴────────┐
    │  单源/单目标/   │      │   单源/单目标/    │
    │  全点对 变体    │      │   全点对 变体     │
    └─────────────────┘      └──────────────────┘
```

### 5.2 关键设计亮点

| 设计点 | 实现策略 | 优势 |
|--------|---------|------|
| **统一接口** | `generic.py` 作为分发层 | 用户无需关心底层算法 |
| **智能分发** | 基于 `weight` 参数自动选择 | API 简洁，自动最优选择 |
| **权重抽象** | `_weight_function` 统一处理 | 支持属性名、函数、多重图 |
| **懒删除** | 优先队列过期条目跳过 | 实现简单，利用标准 heapq |
| **计数器** | 三元组 `(dist, count, node)` | 避免节点不可比较问题 |
| **多源支持** | Dijkstra 原生支持多源 | 灵活应对多种场景 |
| **双向搜索** | 单源单目标使用双向版本 | 时间/空间复杂度减半 |
| **SPFA 优化** | Bellman-Ford 队列优化 | 实际性能显著提升 |
| **负环检测** | 入队次数 + 启发式追踪 | 快速检测负权环 |

### 5.3 算法选择指南

| 场景 | 推荐算法 | 原因 |
|------|---------|------|
| 无权图 | BFS（双向 BFS） | 时间复杂度最优 $O(V+E)$ |
| 带权图（非负权） | Dijkstra | 高效稳定 |
| 带权图（有负权） | Bellman-Ford (SPFA) | 唯一能处理负权的选择 |
| 带权图（有好的启发式） | A* | 启发式引导，搜索更快 |
| 全点对（稠密图） | Floyd-Warshall | $O(V^3)$ 实现简单 |
| 全点对（稀疏图） | Johnson | $O(VE \log V)$ 更优 |
| 需要检测负环 | Bellman-Ford | 内置负环检测能力 |

### 5.4 代码设计的工程价值

1. **可维护性**：
   - 清晰的模块划分
   - 统一的编码风格
   - 详细的文档字符串

2. **可扩展性**：
   - 权重函数抽象支持自定义
   - 启发式函数支持自定义
   - 易于添加新的最短路径算法

3. **性能优化**：
   - 双向搜索减少搜索空间
   - SPFA 优化 Bellman-Ford
   - 截止距离提前终止
   - 懒删除简化实现

4. **健壮性**：
   - 完善的参数验证
   - 清晰的异常类型
   - 负权/负环检测

---

## 附录：关键函数索引

### 统一接口层（generic.py）

| 函数 | 行号 | 功能 |
|------|------|------|
| `shortest_path()` | 42-178 | 统一最短路径接口 |
| `shortest_path_length()` | 181-322 | 统一路径长度接口 |
| `average_shortest_path_length()` | 325-438 | 平均最短路径长度 |
| `all_shortest_paths()` | 441-516 | 所有最短路径 |
| `_build_paths_from_predecessors()` | 652-714 | 从前驱字典重构路径 |

### 无权图层（unweighted.py）

| 函数 | 行号 | 功能 |
|------|------|------|
| `_single_shortest_path_length()` | 83-113 | 标准 BFS 实现 |
| `single_target_shortest_path_length()` | 116-172 | 单目标 BFS（有向图适配） |
| `_bidirectional_pred_succ()` | 292-341 | 双向 BFS 核心 |
| `predecessor()` | 540-625 | 前驱字典计算（所有最短路径） |

### 带权图层（weighted.py）

| 函数 | 行号 | 功能 |
|------|------|------|
| `_weight_function()` | 41-78 | 权重函数抽象 |
| `_dijkstra_multisource()` | 784-907 | Dijkstra 核心实现（多源） |
| `bidirectional_dijkstra()` | 2310-2459 | 双向 Dijkstra |
| `_inner_bellman_ford()` | 1389-1510 | Bellman-Ford (SPFA) 核心 |
| `find_negative_cycle()` | 2214-2306 | 负环查找 |
| `johnson()` | 2462-2542 | Johnson 全点对算法 |

### A* 算法层（astar.py）

| 函数 | 行号 | 功能 |
|------|------|------|
| `astar_path()` | 12-171 | A* 最短路径 |
| `astar_path_length()` | 174-239 | A* 路径长度 |

---

## 6. 稠密图专用全点对算法：Floyd-Warshall 实现分析

### 6.1 问题背景：全点对最短路径

**全点对最短路径**（All-Pairs Shortest Paths, APSP）问题要求计算图中任意两点之间的最短路径。对于不同的图特性，有不同的最优算法：

| 图类型 | 推荐算法 | 时间复杂度 |
|--------|---------|-----------|
| 无权图 | $V$ 次 BFS | $O(V(V+E))$ |
| 带权图（非负权，稀疏） | Johnson 算法 | $O(VE \log V)$ |
| 带权图（稠密） | Floyd-Warshall | $O(V^3)$ |
| 带权图（负权） | Johnson 或 Floyd-Warshall | 同上 |

**Floyd-Warshall 的优势**：
1. 实现极其简洁
2. 支持负权边（但不能有负环）
3. 对于稠密图（$E \approx V^2$），常数因子很小
4. 易于并行化和向量化（NumPy 版本）

### 6.2 Floyd-Warshall 算法原理

#### 6.2.1 动态规划状态定义

Floyd-Warshall 是一个**动态规划**算法，核心思想是考虑**中间节点**。

**状态定义**：
- 设 $dist[k][i][j]$ 表示从节点 $i$ 到节点 $j$，中间只经过编号为 $\{1, 2, ..., k\}$ 的节点的最短路径长度。

**状态转移方程**：
```
dist[k][i][j] = min(
    dist[k-1][i][j],                    # 不经过第 k 个节点
    dist[k-1][i][k] + dist[k-1][k][j]   # 经过第 k 个节点
)
```

**边界条件**：
- $dist[0][i][j] = weight(i, j)$ 如果有边 $(i, j)$
- $dist[0][i][j] = \infty$ 如果没有边
- $dist[0][i][i] = 0$ 节点到自身的距离为 0

#### 6.2.2 空间优化

注意到状态转移只依赖于 $k-1$ 层的状态，可以**优化空间**：
- 使用二维数组 $dist[i][j]$ 代替三维数组
- 原地更新，无需额外空间

**优化后的算法**：
```
for k from 1 to V:
    for i from 1 to V:
        for j from 1 to V:
            dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j])
```

### 6.3 NetworkX 纯 Python 实现分析

#### 6.3.1 数据结构选择

NetworkX 使用**嵌套 defaultdict** 存储距离和前驱：

```python
# dense.py:17-36
def _init_pred_dist(G, weight):
    """Initialize pred and dist to be used in Floyd Warshall algorithms"""
    # dictionary-of-dictionaries representation for dist and pred
    # use some defaultdict magick here
    # for dist the default is the floating point inf value
    dist = defaultdict(lambda: defaultdict(lambda: float("inf")))
    pred = defaultdict(dict)
    # initialize path distance dictionary to be the adjacency matrix
    # also set the distance to self to 0 (zero diagonal)
    weight = _weight_function(G, weight)
    for u, unbr in G._adj.items():
        for v, e_attr in unbr.items():
            cost = weight(u, v, e_attr)
            if cost is not None:  # for hidden edge, weight() returns None
                dist[u][v] = cost
                pred[u][v] = u
        if dist[u][u] < 0:  # negative self loop
            raise nx.NetworkXUnbounded("Negative cycle detected.")
        dist[u][u] = 0
    return pred, dist
```

**设计分析**：

| 数据结构 | 用途 | 优势 |
|---------|------|------|
| `dist[u][v]` | 存储 $u$ 到 $v$ 的最短距离 | 支持任意可哈希节点类型，不限于整数 |
| `pred[u][v]` | 存储从 $u$ 到 $v$ 的最短路径中，$v$ 的前驱节点 | 用于路径重构 |

**为什么不用列表/矩阵？**
- NetworkX 的节点可以是任意可哈希对象（字符串、元组等）
- 使用 `defaultdict` 自动处理不存在的键（返回 `inf`）
- 对于稀疏图，节省空间（只存储存在的边）

#### 6.3.2 核心三重循环

```python
# dense.py:271-353
@nx._dispatchable(edge_attrs="weight")
def floyd_warshall_predecessor_and_distance(G, weight="weight"):
    """Find all-pairs shortest path lengths using Floyd's algorithm."""
    pred, dist = _init_pred_dist(G, weight)
    for w in G:
        dist_w = dist[w]  # save recomputation
        for u in G:
            dist_u = dist[u]  # save recomputation
            for v in G:
                d = dist_u[w] + dist_w[v]
                if dist_u[v] > d:
                    dist_u[v] = d
                    pred[u][v] = pred[w][v]
    if any(dist[u][u] < 0 for u in G):
        raise nx.NetworkXUnbounded("Negative cycle detected.")
    return dict(pred), dict(dist)
```

**关键点分析**：

1. **循环变量选择**：
   - `w`：中间节点（对应理论中的 $k$）
   - `u`：源节点
   - `v`：目标节点

2. **局部变量优化**：
   ```python
   dist_w = dist[w]  # save recomputation
   dist_u = dist[u]  # save recomputation
   ```
   - 避免重复的字典查找
   - Python 中局部变量访问比属性访问更快

3. **状态转移**：
   ```python
   d = dist_u[w] + dist_w[v]
   if dist_u[v] > d:
       dist_u[v] = d
       pred[u][v] = pred[w][v]  # 关键：前驱的更新
   ```
   - 这里的 `pred[u][v] = pred[w][v]` 非常巧妙
   - 表示：如果经过 $w$ 到 $v$ 更近，那么 $v$ 的前驱就是"从 $w$ 到 $v$ 的最短路径中 $v$ 的前驱"

#### 6.3.3 前驱更新机制详解

**为什么 `pred[u][v] = pred[w][v]` 是正确的？**

让我们通过一个例子来理解：

```
图：A → B → C → D
    A → D（很长的边）

初始状态（k=0，不经过中间节点）：
  dist[A][D] = 长距离
  pred[A][D] = A  # 直接从 A 到 D

当 w = B 时：
  dist[A][B] + dist[B][D] 可能更短
  如果 dist[A][D] > dist[A][B] + dist[B][D]:
      dist[A][D] = dist[A][B] + dist[B][D]
      pred[A][D] = pred[B][D]  # 这是关键！

当 w = C 时：
  类似地更新...
```

**理解**：
- 当我们发现从 $u$ 经过 $w$ 到 $v$ 是更优路径时
- 这条路径的完整形式是：$u \to ... \to w \to ... \to v$
- $w$ 到 $v$ 的部分已经是最优的（由 `pred[w][v]` 记录）
- 所以 $v$ 的前驱应该就是 `pred[w][v]`

#### 6.3.4 负环检测

```python
# dense.py:351-352
if any(dist[u][u] < 0 for u in G):
    raise nx.NetworkXUnbounded("Negative cycle detected.")
```

**原理**：
- 如果存在从 $u$ 到 $u$ 的路径，其总长度小于 0
- 说明经过了一个**负权环**
- 负权环使最短路径无意义（可以无限循环使路径长度趋近于 $-\infty$）

**初始化时也检测**：
```python
# dense.py:33-34
if dist[u][u] < 0:  # negative self loop
    raise nx.NetworkXUnbounded("Negative cycle detected.")
```
- 自环（self-loop）的负权直接就是负环

### 6.4 路径重建机制

#### 6.4.1 专用路径重建函数

NetworkX 提供了 `reconstruct_path` 函数来从 `pred` 字典重构路径：

```python
# dense.py:356-397
@nx._dispatchable(graphs=None)
def reconstruct_path(source, target, predecessors):
    """Reconstruct a path from source to target using the predecessors
    dict as returned by floyd_warshall_predecessor_and_distance"""
    if source == target:
        return []
    prev = predecessors[source]  # 获取从 source 出发的所有前驱信息
    curr = prev[target]          # target 的前驱
    path = [target, curr]
    while curr != source:
        curr = prev[curr]        # 继续向前追溯
        path.append(curr)
    return list(reversed(path))   # 反转得到从 source 到 target 的路径
```

#### 6.4.2 路径重建示例

假设有图：`A → B → C → D`，且 Floyd-Warshall 已计算完成。

**`predecessors` 字典内容**：
```python
predecessors = {
    'A': {
        'B': 'A',   # A→B 的路径：B 的前驱是 A
        'C': 'B',   # A→C 的路径：C 的前驱是 B（即 A→B→C）
        'D': 'C'    # A→D 的路径：D 的前驱是 C（即 A→B→C→D）
    },
    'B': {
        'C': 'B',
        'D': 'C'
    },
    # ...
}
```

**重构路径 `reconstruct_path('A', 'D', predecessors)`**：

```
步骤1：source='A', target='D'
       prev = predecessors['A'] = {'B': 'A', 'C': 'B', 'D': 'C'}
       
步骤2：curr = prev['D'] = 'C'
       path = ['D', 'C']

步骤3：curr != 'A'，继续：
       curr = prev['C'] = 'B'
       path = ['D', 'C', 'B']

步骤4：curr != 'A'，继续：
       curr = prev['B'] = 'A'
       path = ['D', 'C', 'B', 'A']

步骤5：curr == 'A'，停止
       返回 reversed(path) = ['A', 'B', 'C', 'D']
```

### 6.5 NumPy 加速版本分析

#### 6.5.1 实现差异

NetworkX 提供了 `floyd_warshall_numpy` 函数，利用 NumPy 的向量化操作大幅提升性能：

```python
# dense.py:39-122
@nx._dispatchable(edge_attrs="weight")
def floyd_warshall_numpy(G, nodelist=None, weight="weight"):
    """Find all-pairs shortest path lengths using Floyd's algorithm.
    
    This algorithm for finding shortest paths takes advantage of
    matrix representations of a graph and works well for dense
    graphs where all-pairs shortest path lengths are desired.
    """
    import numpy as np
    
    # 步骤1：转换为 NumPy 邻接矩阵
    A = nx.to_numpy_array(
        G, nodelist, multigraph_weight=min, weight=weight, nonedge=np.inf
    )
    n, m = A.shape
    
    # 步骤2：预处理（负自环检测）
    if np.any(np.diag(A) < 0):
        raise nx.NetworkXUnbounded("Negative cycle detected.")
    np.fill_diagonal(A, 0)  # diagonal elements should be zero
    
    # 步骤3：核心三重循环（向量化版本）
    for i in range(n):
        # 关键：使用 NumPy 广播（broadcasting）
        # A[i, :] 是行向量：从 i 到所有节点的距离
        # A[:, i] 是列向量：从所有节点到 i 的距离
        # A[:, i][:, np.newaxis] + A[i, :][np.newaxis, :]
        #   = 外积式的加法：每个 (u, v) 计算 dist[u][i] + dist[i][v]
        A = np.minimum(A, A[i, :][np.newaxis, :] + A[:, i][:, np.newaxis])
    
    # 步骤4：负环检测
    if np.any(np.diag(A) < 0):
        raise nx.NetworkXUnbounded("Negative cycle detected.")
    return A
```

#### 6.5.2 纯 Python vs NumPy 版本对比

| 特性 | 纯 Python 版本 | NumPy 版本 |
|------|--------------|-----------|
| **数据结构** | `defaultdict` 嵌套 | NumPy 2D 数组 |
| **节点类型** | 任意可哈希类型 | 必须映射为整数索引 |
| **路径重建** | 支持（通过 `pred` 字典） | 不直接支持（只返回距离矩阵） |
| **性能** | 对于大图较慢 | 向量化操作，快 10-100 倍 |
| **内存** | 稀疏友好（只存存在的边） | 稠密矩阵（总是 $O(V^2)$） |
| **负权边** | 支持 | 支持 |
| **负环检测** | 支持 | 支持 |

#### 6.5.3 NumPy 广播机制详解

**核心向量化操作**：
```python
A = np.minimum(A, A[i, :][np.newaxis, :] + A[:, i][:, np.newaxis])
```

让我们展开这个操作：

**假设有 3 个节点，当前中间节点是 i=1**：

```
A[i, :] = A[1, :] = [d10, d11, d12]  # 行向量
A[:, i] = A[:, 1] = [d01, d11, d21]  # 列向量

# 将行向量扩展为形状 (1, 3)
A[i, :][np.newaxis, :] = [[d10, d11, d12]]

# 将列向量扩展为形状 (3, 1)
A[:, i][:, np.newaxis] = [[d01],
                          [d11],
                          [d21]]

# 广播加法：(3, 1) + (1, 3) = (3, 3)
[[d01],   +   [[d10, d11, d12]]   =   [[d01+d10, d01+d11, d01+d12],
 [d11],                                [d11+d10, d11+d11, d11+d12],
 [d21]]                                [d21+d10, d21+d11, d21+d12]]

# 结果矩阵的每个元素 (u, v) = dist[u][i] + dist[i][v]
```

**广播的优势**：
- 所有运算在 C 层面执行，比 Python 循环快得多
- 利用 CPU 缓存和 SIMD 指令
- 代码更简洁

#### 6.5.4 版本选择建议

| 场景 | 推荐版本 | 原因 |
|------|---------|------|
| 需要路径重建 | 纯 Python 版本 | NumPy 版本不返回 `pred` |
| 节点非整数（字符串等） | 纯 Python 版本 | NumPy 需要整数索引 |
| 大图（1000+ 节点） | NumPy 版本 | 性能差异显著 |
| 稠密图 | NumPy 版本 | 向量化更优 |
| 稀疏图 | 可考虑 Johnson | Floyd-Warshall 都是 $O(V^3)$ |

### 6.6 Floyd-Warshall 变体：Tree 版本

NetworkX 还提供了一个优化版本 `floyd_warshall_tree`，基于 Brodnik 等人的研究：

```python
# dense.py:125-268
@nx._dispatchable(edge_attrs="weight")
def floyd_warshall_tree(G, weight="weight"):
    r"""Find all-pairs shortest path lengths using a Tree-based
    modification of Floyd's algorithm.
    
    This variant implements the Tree algorithm of Brodnik, Grgurovic and Pozar.
    It differs from the classical Floyd Warshall algorithm by using a shortest
    path tree rooted at each intermediate vertex w.
    
    For complete directed graphs with independent edge weights drawn from the
    uniform distribution on [0, 1], the expected running time is
    $O(|V|^2 (\log |V|)^2)$, as shown in [1]_.
    """
```

**核心优化思想**：
1. 为每个中间节点 $w$ 构建最短路径树 $OUT_w$
2. 使用深度优先搜索（DFS）遍历树
3. 如果某个节点 $v$ 的松弛不能改进任何距离，**跳过整个子树**

**适用场景**：
- 完全有向图，边权服从均匀分布
- 期望时间复杂度从 $O(V^3)$ 降到 $O(V^2 \log^2 V)$
- 但最坏情况仍是 $O(V^3)$

---

## 7. Johnson 算法深度分析：重赋权与批量 Dijkstra

### 7.1 问题背景

**Dijkstra 算法的限制**：
- 只能处理**非负权边**
- 如果存在负权边，可能得到错误结果

**Bellman-Ford 算法的问题**：
- 可以处理负权边
- 但单源时间复杂度是 $O(VE)$
- 全点对就是 $O(V^2 E)$，对于稠密图是 $O(V^4)$，不可接受

**Johnson 算法的创新**：
1. 使用 Bellman-Ford 进行**重赋权**（reweighting），消除负权
2. 然后对每个节点运行**高效的 Dijkstra 算法**
3. 时间复杂度：$O(VE + V^2 \log V)$

### 7.2 算法原理

#### 7.2.1 重赋权技术

**核心思想**：找到一个节点势函数 $h(v)$，使得：
- 新的边权 $\hat{w}(u, v) = w(u, v) + h(u) - h(v)$ 都是非负的
- 最短路径结构保持不变

**为什么这样的变换保持最短路径？**

考虑路径 $P = u \to v_1 \to v_2 \to ... \to v_k \to v$：

原路径长度：
$$w(P) = w(u, v_1) + w(v_1, v_2) + ... + w(v_k, v)$$

新路径长度：
$$\hat{w}(P) = \hat{w}(u, v_1) + \hat{w}(v_1, v_2) + ... + \hat{w}(v_k, v)$$
$$= [w(u, v_1) + h(u) - h(v_1)] + [w(v_1, v_2) + h(v_1) - h(v_2)] + ... + [w(v_k, v) + h(v_k) - h(v)]$$

**望远镜展开**（telescoping sum）：
$$= w(u, v_1) + w(v_1, v_2) + ... + w(v_k, v) + h(u) - h(v)$$
$$= w(P) + h(u) - h(v)$$

**关键发现**：
- 对于固定的 $u$ 和 $v$，$h(u) - h(v)$ 是**常数**
- 因此，$\hat{w}(P) = w(P) + \text{常数}$
- 最小化 $\hat{w}(P)$ 等价于最小化 $w(P)$
- **最短路径不变！**

#### 7.2.2 如何找到势函数 $h$？

**势函数条件**：
- 对于所有边 $(u, v)$，需要 $\hat{w}(u, v) \geq 0$
- 即 $w(u, v) + h(u) - h(v) \geq 0$
- 变形：$h(v) \leq h(u) + w(u, v)$

**这看起来很熟悉...**
- 这正是最短路径的**三角不等式**！
- 如果 $h(v)$ 是从某个源点 $s$ 到 $v$ 的最短距离，那么：
  $$dist(s, v) \leq dist(s, u) + w(u, v)$$

**解决方案**：
1. 添加一个**超级源点** $s$，它到所有其他节点都有一条权为 0 的边
2. 使用 Bellman-Ford 计算从 $s$ 到所有节点的最短距离 $h(v)$
3. 这个 $h(v)$ 就是我们需要的势函数！

#### 7.2.3 完整算法流程

```
Johnson(G):
    1. 创建 G' = G ∪ {s}，添加边 (s, v)，权为 0，对所有 v ∈ V
    
    2. 使用 Bellman-Ford(G', s) 计算 h(v) = 从 s 到 v 的最短距离
       - 如果 Bellman-Ford 检测到负环，报错
       
    3. 重赋权：
       对每条边 (u, v)，计算新权：
           ŵ(u, v) = w(u, v) + h(u) - h(v)
    
    4. 对每个节点 u ∈ V：
       使用 Dijkstra 在重赋权后的图上计算从 u 到所有 v 的最短距离 d̂(u, v)
    
    5. 恢复原始距离：
       d(u, v) = d̂(u, v) - h(u) + h(v)
    
    6. 返回 d(u, v)
```

### 7.3 NetworkX 实现分析

#### 7.3.1 核心实现

```python
# weighted.py:2462-2542
@nx._dispatchable(edge_attrs="weight")
def johnson(G, weight="weight"):
    r"""Uses Johnson's Algorithm to compute shortest paths.
    
    Johnson's Algorithm finds a shortest path between each pair of
    nodes in a weighted graph even if negative weights are present.
    """
    # 步骤1 & 2：使用 Bellman-Ford 计算势函数 h(v)
    # 注意：NetworkX 的实现有些巧妙，没有显式添加超级源点
    # 而是将所有节点作为源点传入 _bellman_ford
    
    dist = {v: 0 for v in G}  # 初始距离都是 0
    pred = {v: [] for v in G}
    weight = _weight_function(G, weight)
    
    # 将所有节点作为源点，效果等价于添加超级源点
    dist_bellman = _bellman_ford(G, list(G), weight, pred=pred, dist=dist)
    
    # 步骤3：定义重赋权后的权重函数
    def new_weight(u, v, d):
        return weight(u, v, d) + dist_bellman[u] - dist_bellman[v]
    
    # 步骤4 & 5：对每个节点运行 Dijkstra，自动恢复距离
    def dist_path(v):
        paths = {v: [v]}
        # _dijkstra 使用 new_weight，但返回的路径是原始节点顺序
        # 距离的恢复：d(u, v) = d̂(u, v) - h(u) + h(v)
        # 注意：这里 _dijkstra 内部只计算路径，距离的恢复是隐式的
        # （因为我们只关心路径本身，不是距离数值）
        _dijkstra(G, v, new_weight, paths=paths)
        return paths
    
    # 步骤6：返回所有路径
    return {v: dist_path(v) for v in G}
```

#### 7.3.2 实现细节解析

**问题：NetworkX 为什么没有显式添加超级源点？**

查看 `_bellman_ford` 的实现：

```python
# weighted.py:1295-1386
def _bellman_ford(
    G,
    source,          # 这里 source 是一个列表！
    weight,
    pred=None,
    paths=None,
    dist=None,       # 可以传入初始距离
    target=None,
    heuristic=True,
):
    """Calls relaxation loop for Bellman–Ford algorithm and builds paths
    
    This is an implementation of the SPFA variant.
    """
    if pred is None:
        pred = {v: [] for v in source}
    
    if dist is None:
        dist = {v: 0 for v in source}  # 所有源点初始距离为 0
    # ...
```

**关键洞察**：
- `_bellman_ford` 支持**多源点**
- 当 `source = list(G)` 且 `dist = {v: 0 for v in G}` 时
- 效果等价于：添加了一个超级源点 $s$，它到所有节点的边权为 0
- 因为从"任一源点"出发的最短距离就是从 $s$ 出发的最短距离

#### 7.3.3 距离恢复

**为什么返回的路径是正确的？**

虽然 `johnson()` 函数内部使用重赋权后的边权运行 Dijkstra，但：
1. 重赋权只改变边权的数值
2. **不改变最短路径的节点顺序**
3. 因为如前所述，变换是保序的

**如果需要距离数值**：
- `johnson()` 只返回路径，不返回距离
- 如果需要距离，可以：
  ```python
  # 方法1：使用 floyd_warshall（如果图不太大）
  dist = nx.floyd_warshall(G, weight='weight')
  
  # 方法2：手动从路径计算
  paths = nx.johnson(G, weight='weight')
  # 然后根据边权累加
  ```

### 7.4 时间复杂度分析

| 步骤 | 算法 | 时间复杂度 |
|------|------|-----------|
| 势函数计算 | Bellman-Ford (SPFA) | $O(VE)$ |
| 重赋权 | 遍历所有边 | $O(E)$ |
| 单源最短路径 | $V$ 次 Dijkstra | $O(V \cdot E \log V)$（二叉堆） |
| **总计** | | **$O(VE \log V)$** |

**与 Floyd-Warshall 对比**：
- Floyd-Warshall：$O(V^3)$
- Johnson：$O(VE \log V)$

**对于不同图类型**：
- 稀疏图（$E = O(V)$）：Johnson = $O(V^2 \log V)$ < Floyd-Warshall = $O(V^3)$ ✓
- 稠密图（$E = O(V^2)$）：Johnson = $O(V^3 \log V)$ > Floyd-Warshall = $O(V^3)$ ✗

### 7.5 适用场景对比

| 场景 | 推荐算法 | 原因 |
|------|---------|------|
| 稠密图 + 负权 | Floyd-Warshall | 常数因子小，实现简单 |
| 稀疏图 + 负权 | Johnson | $O(VE \log V)$ 优于 $O(V^3)$ |
| 稠密图 + 非负权 | Floyd-Warshall 或 $V$ 次 Dijkstra | 视具体情况 |
| 稀疏图 + 非负权 | $V$ 次 Dijkstra | 无需 Bellman-Ford 预处理 |
| 需要路径重建 | 任意 | 都支持 |
| 需要 NumPy 加速 | Floyd-Warshall (NumPy 版本) | Johnson 无 NumPy 版本 |

---

## 8. 全点对最短路径算法综合对比

### 8.1 算法特性汇总

| 特性 | Floyd-Warshall | Johnson | $V$ 次 BFS/Dijkstra |
|------|---------------|---------|---------------------|
| **时间复杂度** | $O(V^3)$ | $O(VE \log V)$ | $O(V(V+E))$ 或 $O(VE \log V)$ |
| **空间复杂度** | $O(V^2)$ | $O(V+E)$ | $O(V+E)$ |
| **负权边** | 支持 | 支持 | BFS: 否, Dijkstra: 否 |
| **负环检测** | 支持 | 支持 | 不适用 |
| **路径重建** | 支持 | 支持 | 支持 |
| **NumPy 加速** | 支持 | 不支持 | 不支持 |
| **实现难度** | 极简单 | 中等 | 简单 |

### 8.2 图类型与算法选择

```
                    全点对最短路径问题
                            │
              ┌─────────────┴─────────────┐
              │                           │
          无权图                        带权图
              │                           │
              ▼              ┌────────────┴────────────┐
       V 次 BFS              │                         │
    O(V(V+E))              有负权?                 无非负权
                            │                         │
                    ┌───────┴───────┐                 │
                    │               │                 │
                 稠密图          稀疏图           V 次 Dijkstra
                    │               │            O(VE log V)
                    ▼               ▼
            Floyd-Warshall    Johnson 算法
               O(V³)          O(VE log V)
```

### 8.3 工程实践建议

**选择 Floyd-Warshall 的情况**：
1. 图的规模较小（如 $V < 1000$）
2. 图是稠密的（$E \approx V^2$）
3. 需要处理负权边
4. 代码简洁性优先
5. 需要 NumPy 向量化加速

**选择 Johnson 的情况**：
1. 图是稀疏的（$E = O(V)$ 或 $O(V \log V)$）
2. 需要处理负权边
3. 图的规模中等（$V > 1000$ 但不太大）

**选择 $V$ 次 Dijkstra 的情况**：
1. 所有边权非负
2. 图是稀疏的
3. 性能最优（无需 Bellman-Ford 预处理）

**性能预期（估算）**：

| 节点数 $V$ | Floyd-Warshall | Johnson (稀疏图) |
|-----------|----------------|------------------|
| 100 | 0.001s | 0.005s |
| 500 | 0.125s | 0.1s |
| 1000 | 1s | 0.5s |
| 2000 | 8s | 2s |
| 5000 | 125s | 10s |

*注：假设稀疏图 $E = O(V)$，实际性能因实现和硬件而异*

### 8.4 NetworkX 中的统一接口

回顾 `average_shortest_path_length` 函数，它展示了 NetworkX 如何智能选择全点对算法：

```python
# generic.py:325-438
@nx._dispatchable(edge_attrs="weight")
def average_shortest_path_length(G, weight=None, method=None):
    r"""Returns the average shortest path length."""
    single_source_methods = ["unweighted", "dijkstra", "bellman-ford"]
    all_pairs_methods = ["floyd-warshall", "floyd-warshall-numpy"]
    supported_methods = single_source_methods + all_pairs_methods
    
    if method is None:
        method = "unweighted" if weight is None else "dijkstra"
    # ...
    
    if method in single_source_methods:
        # 使用单源方法迭代：O(V * 单源时间)
        s = sum(l for u in G for l in path_length(u).values())
    else:
        if method == "floyd-warshall":
            # 使用 Floyd-Warshall：O(V³)
            all_pairs = nx.floyd_warshall(G, weight=weight)
            s = sum(sum(t.values()) for t in all_pairs.values())
        elif method == "floyd-warshall-numpy":
            # 使用 NumPy 加速版本
            s = float(nx.floyd_warshall_numpy(G, weight=weight).sum())
```

**设计思想**：
1. 提供 `method` 参数让用户选择
2. 默认选择：
   - 无权图：`"unweighted"`（$V$ 次 BFS）
   - 带权图：`"dijkstra"`（$V$ 次 Dijkstra）
3. 用户可显式指定 `"floyd-warshall"` 或 `"floyd-warshall-numpy"`

---

## 附录 B：稠密图算法函数索引

### Floyd-Warshall 家族（dense.py）

| 函数 | 行号 | 功能 |
|------|------|------|
| `_init_pred_dist()` | 17-36 | 初始化距离和前驱字典 |
| `floyd_warshall_numpy()` | 39-122 | NumPy 加速版 Floyd-Warshall |
| `floyd_warshall_tree()` | 125-268 | 优化版（最短路径树） |
| `floyd_warshall_predecessor_and_distance()` | 271-353 | 标准版，返回 pred 和 dist |
| `reconstruct_path()` | 356-397 | 从前驱字典重建路径 |
| `floyd_warshall()` | 400-466 | 简化版，只返回距离 |

### Johnson 算法（weighted.py）

| 函数 | 行号 | 功能 |
|------|------|------|
| `johnson()` | 2462-2542 | Johnson 全点对最短路径 |

---

## 附录 C：算法选择决策树

```
开始：需要计算全点对最短路径？
│
├── 图是无权图？
│   ├── 是 → 使用 V 次 BFS
│   │        时间复杂度：O(V(V+E))
│   │
│   └── 否（带权图）→ 继续
│
├── 有负权边？
│   ├── 是 → 继续
│   │   │
│   │   ├── 图是稠密图？
│   │   │   ├── 是 → Floyd-Warshall
│   │   │   │        时间复杂度：O(V³)
│   │   │   │        优点：实现简单，支持 NumPy 加速
│   │   │   │
│   │   │   └── 否（稀疏图）→ Johnson 算法
│   │   │            时间复杂度：O(VE log V)
│   │   │            优点：对于稀疏图比 O(V³) 快
│   │   │
│   │   └── 需要处理负环？
│   │       ├── 是 → Bellman-Ford 可以检测
│   │       │        Floyd-Warshall 也可以检测
│   │       │
│   │       └── 否 → 同上
│   │
│   └── 否（非负权）→ 继续
│       │
│       ├── 图是稠密图？
│       │   ├── 是 → 可选择：
│       │   │        - Floyd-Warshall：实现简单
│       │   │        - V 次 Dijkstra：可能更快
│       │   │
│       │   └── 否（稀疏图）→ V 次 Dijkstra
│       │            时间复杂度：O(VE log V)
│       │            优点：比 Johnson 少了 Bellman-Ford 预处理
│       │
│       └── 需要路径重建？
│           ├── 是 → 所有方法都支持
│           │
│           └── 否 → 同上
│
└── 节点数规模？
    ├── 小规模 (V < 500) → Floyd-Warshall 通常足够
    │
    ├── 中等规模 (500 < V < 5000) → 视稀疏性选择
    │
    └── 大规模 (V > 5000) → 需要专门优化
               （NetworkX 可能不是最佳选择）
```
