# NetworkX 二部图算法深度分析：匹配、投影、中心性

## 目录

1. [概述](#1-概述)
2. [三类算法的节点集合划分约束对比](#2-三类算法的节点集合划分约束对比)
3. [非连通图下的自动划分歧义与报错策略](#3-非连通图下的自动划分歧义与报错策略)
4. [后端调度与通用图接口复用的实际调用路径](#4-后端调度与通用图接口复用的实际调用路径)
5. [调用路径流程图](#5-调用路径流程图)
6. [总结](#6-总结)

---

## 1. 概述

本文档深入分析 NetworkX 中二部图算法包的三类核心算法：

| 算法类别 | 核心函数 | 主要用途 |
|---------|---------|---------|
| **匹配算法** | `maximum_matching`, `hopcroft_karp_matching`, `eppstein_matching`, `minimum_weight_full_matching`, `to_vertex_cover` | 最大基数匹配、最小权完美匹配、顶点覆盖转换 |
| **投影算法** | `projected_graph`, `weighted_projected_graph`, `collaboration_weighted_projected_graph`, `overlap_weighted_projected_graph`, `generic_weighted_projected_graph` | 二部图到单模图的投影 |
| **中心性算法** | `degree_centrality`, `betweenness_centrality`, `closeness_centrality` | 二部图特定的中心性度量 |

这三类算法在节点集合划分的输入约束、非连通图处理策略、以及通用图接口复用方式上存在显著差异。

---

## 2. 三类算法的节点集合划分约束对比

### 2.1 约束对比总览

| 维度 | 匹配算法 | 投影算法 | 中心性算法 |
|------|---------|---------|-----------|
| **参数名** | `top_nodes` | `nodes` | `nodes` |
| **是否必需** | 可选（默认 `None`） | **必需**（无默认值） | **必需**（无默认值） |
| **自动划分支持** | ✅ 支持 | ❌ 不支持 | ❌ 不支持 |
| **二部图验证** | 自动计算时验证 | ❌ 不验证 | ❌ 不验证 |
| **非连通图处理** | 无 `top_nodes` 时报错 | 直接使用用户输入 | 直接使用用户输入 |
| **调用 `bipartite_sets()`** | ✅ 是 | ❌ 否 | ❌ 否 |

### 2.2 匹配算法的约束分析

#### 2.2.1 函数签名

```python
@nx._dispatchable
def hopcroft_karp_matching(G, top_nodes=None): ...

@nx._dispatchable
def eppstein_matching(G, top_nodes=None): ...

@nx._dispatchable(edge_attrs="weight")
def minimum_weight_full_matching(G, top_nodes=None, weight="weight"): ...

@nx._dispatchable
def to_vertex_cover(G, matching, top_nodes=None): ...
```

**关键代码位置**：`networkx/algorithms/bipartite/matching.py:59, 186, 506, 425`

#### 2.2.2 节点划分处理流程

匹配算法通过调用 `bipartite_sets()` 函数获取节点划分：

**关键代码位置**：`networkx/algorithms/bipartite/matching.py:157`

```python
# hopcroft_karp_matching 中的关键代码
left, right = bipartite_sets(G, top_nodes)
leftmatches = {v: None for v in left}
rightmatches = {v: None for v in right}
```

**`bipartite_sets()` 函数的逻辑**：
**关键代码位置**：`networkx/algorithms/bipartite/basic.py:158-220`

```python
def sets(G, top_nodes=None):
    if G.is_directed():
        is_connected = nx.is_weakly_connected
    else:
        is_connected = nx.is_connected
    
    if top_nodes is not None:
        # 用户提供了节点集合，直接使用
        X = set(top_nodes)
        Y = set(G) - X
    else:
        # 用户未提供，尝试自动计算
        if not is_connected(G):
            # 非连通图：存在歧义，抛出异常
            msg = "Disconnected graph: Ambiguous solution for bipartite sets."
            raise nx.AmbiguousSolution(msg)
        # 连通图：通过二着色算法自动计算
        c = color(G)
        X = {n for n, is_top in c.items() if is_top}
        Y = {n for n, is_top in c.items() if not is_top}
    return (X, Y)
```

#### 2.2.3 约束特点

1. **灵活性最高**：用户可以选择提供或不提供 `top_nodes`
2. **自动验证**：当 `top_nodes=None` 时，会通过 `color()` 函数验证图是否真的是二部图
3. **歧义避免**：非连通图且无 `top_nodes` 时，明确抛出 `AmbiguousSolution` 异常
4. **算法依赖划分**：Hopcroft-Karp 算法本身依赖左-右节点划分来构建分层图

### 2.3 投影算法的约束分析

#### 2.3.1 函数签名

```python
@nx._dispatchable(
    graphs="B", preserve_node_attrs=True, preserve_graph_attrs=True, returns_graph=True
)
def projected_graph(B, nodes, multigraph=False): ...

@not_implemented_for("multigraph")
@nx._dispatchable(graphs="B", returns_graph=True)
def weighted_projected_graph(B, nodes, ratio=False): ...

@not_implemented_for("multigraph")
@nx._dispatchable(graphs="B", returns_graph=True)
def collaboration_weighted_projected_graph(B, nodes): ...
```

**关键代码位置**：`networkx/algorithms/bipartite/projection.py:18, 123, 224`

#### 2.3.2 节点划分处理流程

投影算法**不调用** `bipartite_sets()`，而是直接使用用户提供的 `nodes` 参数：

**关键代码位置**：`networkx/algorithms/bipartite/projection.py:105-117`

```python
# projected_graph 中的关键代码
G.add_nodes_from((n, B.nodes[n]) for n in nodes)
for u in nodes:
    # 直接使用 B[u] 访问邻居
    nbrs2 = {v for nbr in B[u] for v in B[nbr] if v != u}
    # ...
    G.add_edges_from((u, n) for n in nbrs2)
```

**weighted_projected_graph 中的边界检查**：
**关键代码位置**：`networkx/algorithms/bipartite/projection.py:200-206`

```python
n_top = len(B) - len(nodes)

if n_top < 1:
    raise nx.NetworkXAlgorithmError(
        f"the size of the nodes to project onto ({len(nodes)}) is >= the graph size ({len(B)}).\n"
        "They are either not a valid bipartite partition or contain duplicates"
    )
```

#### 2.3.3 约束特点

1. **`nodes` 是必需参数**：无默认值，用户必须显式提供
2. **不验证二部图性质**：文档明确说明 "No attempt is made to verify that the input graph B is bipartite"
3. **不验证划分有效性**：不检查 `nodes` 是否真的构成一个有效的二分划
4. **只做边界检查**：只检查 `len(nodes) >= len(B)` 这种明显错误的情况
5. **投影的语义**：`nodes` 表示"要投影到的节点集合"，而不是"二部图的一个划分"

### 2.4 中心性算法的约束分析

#### 2.4.1 函数签名

```python
@nx._dispatchable(name="bipartite_degree_centrality")
def degree_centrality(G, nodes): ...

@nx._dispatchable(name="bipartite_betweenness_centrality")
def betweenness_centrality(G, nodes): ...

@nx._dispatchable(name="bipartite_closeness_centrality")
def closeness_centrality(G, nodes, normalized=True): ...
```

**关键代码位置**：`networkx/algorithms/bipartite/centrality.py:7, 82, 186`

#### 2.4.2 节点划分处理流程

中心性算法同样**不调用** `bipartite_sets()`，直接使用集合运算：

**关键代码位置**：`networkx/algorithms/bipartite/centrality.py:72-77`

```python
# degree_centrality 中的关键代码
top = set(nodes)
bottom = set(G) - top
s = 1.0 / len(bottom)
centrality = {n: d * s for n, d in G.degree(top)}
s = 1.0 / len(top)
centrality.update({n: d * s for n, d in G.degree(bottom)})
```

**betweenness_centrality 中的类似处理**：
**关键代码位置**：`networkx/algorithms/bipartite/centrality.py:161-182`

```python
top = set(nodes)
bottom = set(G) - top
n = len(top)
m = len(bottom)
# ... 计算归一化因子 ...
betweenness = nx.betweenness_centrality(G, normalized=False, weight=None)
for node in top:
    betweenness[node] /= bet_max_top
for node in bottom:
    betweenness[node] /= bet_max_bot
```

#### 2.4.3 约束特点

1. **`nodes` 是必需参数**：无默认值
2. **不做任何验证**：不验证图是否二部，不验证 `nodes` 是否有效划分
3. **直接使用补集**：`bottom = set(G) - top` 假设 `nodes` 正好是一个二分划
4. **归一化依赖划分**：二部图中心性的归一化因子依赖两个集合的大小 `n` 和 `m`

### 2.5 为什么存在这些差异？

| 算法类别 | 设计考量 |
|---------|---------|
| **匹配算法** | 算法本身**内在依赖**左-右划分。Hopcroft-Karp 的 BFS 分层、DFS 增广都需要明确区分左右节点。自动划分是为了用户方便，但非连通图的歧义必须明确拒绝。 |
| **投影算法** | 投影的**语义更灵活**。用户可能想投影到任意节点子集，而不一定是完整的二分划。例如，可以投影到一个二分划的子集。不验证二部图性质是为了性能和灵活性。 |
| **中心性算法** | 中心性的**归一化计算**需要知道两个集合的大小。但算法本身（如最短路径计算）不依赖划分。设计选择让用户显式提供，避免歧义。 |

---

## 3. 非连通图下的自动划分歧义与报错策略

### 3.1 歧义的本质

非连通二部图存在多种合法的二着色方案：

```
示例：两个不连通的边构成的图
G: 1-2, 3-4

方案1（标准着色）：
  X = {1, 3}, Y = {2, 4}
  
方案2（交换第二个分量的颜色）：
  X = {1, 4}, Y = {2, 3}  ← 同样合法！
```

从数学上讲，这两种着色都是有效的二部图划分。但从算法角度看：

- **匹配算法**：不同的划分会导致相同的最大匹配（只是左右互换）
- **投影算法**：如果投影到 `{1, 3}` 或 `{1, 4}`，结果会完全不同
- **中心性算法**：不同的划分会导致不同的归一化因子

### 3.2 三类算法的报错策略对比

| 算法类别 | `top_nodes`/`nodes` 提供情况 | 行为 |
|---------|------------------------------|------|
| **匹配算法** | 已提供 | 直接使用，不验证连通性 |
| **匹配算法** | 未提供（`None`）+ 连通图 | 通过 `color()` 自动计算 |
| **匹配算法** | 未提供（`None`）+ 非连通图 | ❌ 抛出 `AmbiguousSolution` |
| **投影算法** | 已提供 | 直接使用，**不检查连通性** |
| **投影算法** | 未提供 | ❌ 类型错误（必需参数） |
| **中心性算法** | 已提供 | 直接使用，**不检查连通性** |
| **中心性算法** | 未提供 | ❌ 类型错误（必需参数） |

### 3.3 错误类型详解

#### 3.3.1 `AmbiguousSolution` 异常

**触发条件**：匹配算法在非连通图上且未提供 `top_nodes`

**错误消息**：
```
Disconnected graph: Ambiguous solution for bipartite sets.
```

**关键代码位置**：`networkx/algorithms/bipartite/basic.py:214-216`

```python
if not is_connected(G):
    msg = "Disconnected graph: Ambiguous solution for bipartite sets."
    raise nx.AmbiguousSolution(msg)
```

**测试验证**：
**关键代码位置**：`networkx/algorithms/bipartite/tests/test_matching.py:136-142`

```python
def test_eppstein_matching_disconnected(self):
    with pytest.raises(nx.AmbiguousSolution):
        match = eppstein_matching(self.disconnected_graph)

def test_hopcroft_karp_matching_disconnected(self):
    with pytest.raises(nx.AmbiguousSolution):
        match = hopcroft_karp_matching(self.disconnected_graph)
```

#### 3.3.2 `NetworkXAlgorithmError` 异常（投影算法）

**触发条件**：`len(nodes) >= len(B)`

**错误消息**：
```
the size of the nodes to project onto ({len(nodes)}) is >= the graph size ({len(B)}).
They are either not a valid bipartite partition or contain duplicates
```

**关键代码位置**：`networkx/algorithms/bipartite/projection.py:202-206`

```python
if n_top < 1:
    raise nx.NetworkXAlgorithmError(
        f"the size of the nodes to project onto ({len(nodes)}) is >= the graph size ({len(B)}).\n"
        "They are either not a valid bipartite partition or contain duplicates"
    )
```

### 3.4 设计哲学："拒绝猜测"

NetworkX 在这里遵循了 Python 的设计哲学：

> **"In the face of ambiguity, refuse the temptation to guess."**
> —— Python 之禅

对于匹配算法：
- 如果用户提供了 `top_nodes`，相信用户知道自己在做什么
- 如果用户未提供且图不连通，**不猜测**使用哪种着色方案，直接报错

对于投影和中心性算法：
- `nodes` 是必需参数，用户**必须**显式指定
- 这实际上是"强制用户消除歧义"的另一种形式

### 3.5 实际使用建议

#### 非连通二部图的正确处理方式

```python
import networkx as nx
from networkx.algorithms import bipartite

# 创建非连通二部图：两个不连通的边
G = nx.Graph()
G.add_edges_from([(1, 2), (3, 4)])

# ❌ 错误做法：不提供 top_nodes
try:
    matching = bipartite.maximum_matching(G)
except nx.AmbiguousSolution as e:
    print(f"错误: {e}")  # 会抛出异常

# ✅ 正确做法 1：显式提供 top_nodes
top_nodes = {1, 3}  # 明确指定一个划分
matching = bipartite.maximum_matching(G, top_nodes=top_nodes)

# ✅ 正确做法 2：按连通分量分别处理
for component in nx.connected_components(G):
    subgraph = G.subgraph(component)
    # 每个连通分量都是连通的，可以自动划分
    X, Y = bipartite.sets(subgraph)
    matching = bipartite.maximum_matching(subgraph)
    # 处理每个分量的结果...
```

---

## 4. 后端调度与通用图接口复用的实际调用路径

### 4.1 `@nx._dispatchable` 装饰器的作用

所有二部图算法都使用 `@nx._dispatchable` 装饰器，这是 NetworkX 3.x 引入的后端调度机制。

#### 4.1.1 装饰器参数对比

| 函数 | `@nx._dispatchable` 参数 | 说明 |
|------|--------------------------|------|
| `hopcroft_karp_matching` | 默认 | 图参数为 `G` (位置 0) |
| `projected_graph` | `graphs="B", preserve_node_attrs=True, preserve_graph_attrs=True, returns_graph=True` | 图参数为 `B`，保留属性，返回图 |
| `degree_centrality` | `name="bipartite_degree_centrality"` | 使用自定义名称避免命名冲突 |
| `minimum_weight_full_matching` | `edge_attrs="weight"` | 包含边属性 `weight` |

#### 4.1.2 为什么中心性算法需要 `name=` 参数？

**关键代码位置**：`networkx/algorithms/bipartite/centrality.py:6`

```python
@nx._dispatchable(name="bipartite_degree_centrality")
def degree_centrality(G, nodes): ...
```

原因：
- `nx.degree_centrality()` 是通用图的中心性算法
- `nx.bipartite.degree_centrality()` 是二部图特定的版本
- 如果不指定 `name=`，两者都会以 `"degree_centrality"` 注册到调度系统
- 这会导致 `KeyError: "Algorithm already exists in dispatch namespace"`

**验证代码**：
**关键代码位置**：`networkx/utils/backends.py:469-473`

```python
if name in _registered_algorithms:
    raise KeyError(
        f"Algorithm already exists in dispatch namespace: {name}. "
        "Fix by assigning a unique `name=` in the `@_dispatchable` decorator."
    )
```

### 4.2 后端调度的完整调用路径

#### 4.2.1 流程图概览

```
用户调用函数
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│           @nx._dispatchable 装饰器包装的函数                │
│                    (__call__ 方法)                           │
└─────────────────────────────────────────────────────────────┘
     │
     ├─────────────── 检查是否有后端安装 ───────────────┐
     │                                                   │
     ▼                                                   ▼
┌─────────────────────┐                   ┌─────────────────────────┐
│  无后端安装          │                   │  有后端安装              │
│  _call_if_no_backends│                   │  _call_if_any_backends  │
│  _installed          │                   │  _installed              │
└─────────────────────┘                   └─────────────────────────┘
     │                                                   │
     │                              ┌────────────────────┼────────────────────┐
     │                              ▼                    ▼                    ▼
     │                     ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
     │                     │ backend=     │    │ 输入图来自   │    │ 尝试配置的   │
     │                     │ 指定后端     │    │ 某后端      │    │ 后端优先级   │
     │                     └──────────────┘    └──────────────┘    └──────────────┘
     │                              │                    │                    │
     │                              └────────────────────┼────────────────────┘
     │                                                   ▼
     │                                          ┌──────────────────────┐
     │                                          │ 后端 can_run?        │
     │                                          │ should_run?          │
     │                                          └──────────────────────┘
     │                                                   │
     │                              ┌────────────────────┼────────────────────┐
     │                              ▼                    ▼                    ▼
     │                     ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
     │                     │ 后端可运行    │    │ 转换到后端图  │    │ fallback到   │
     │                     │ 直接调用      │    │ 然后调用      │    │ networkx     │
     │                     └──────────────┘    └──────────────┘    └──────────────┘
     │                                                   │
     └───────────────────────┬───────────────────────────┘
                             ▼
                  ┌──────────────────────┐
                  │  orig_func(*args,    │
                  │            **kwargs)  │
                  │ (原始 NetworkX 实现)  │
                  └──────────────────────┘
```

#### 4.2.2 无后端时的快速路径

**关键代码位置**：`networkx/utils/backends.py:541-551`

```python
def _call_if_no_backends_installed(self, /, *args, backend=None, **kwargs):
    """Returns the result of the original function (no backends installed)."""
    if backend is not None and backend != "networkx":
        raise ImportError(f"'{backend}' backend is not installed")
    if "networkx" not in self.backends:
        raise NotImplementedError(
            f"'{self.name}' is not implemented by 'networkx' backend. "
            "This function is included in NetworkX as an API to dispatch to "
            "other backends."
        )
    return self.orig_func(*args, **kwargs)  # 直接调用原始函数
```

这是最常见的情况（用户未安装额外后端），此时 `@_dispatchable` 的开销最小，直接调用原始函数。

#### 4.2.3 有后端时的完整流程

**关键代码位置**：`networkx/utils/backends.py:554-899`

`_call_if_any_backends_installed` 的主要步骤：

1. **解析图参数**：从 `args` 和 `kwargs` 中提取图对象
2. **检测输入图的后端**：通过 `__networkx_backend__` 属性识别
3. **确定后端优先级**：
   - 如果用户指定了 `backend=` 参数，使用该后端
   - 否则，如果输入图来自某后端，优先尝试该后端
   - 否则，使用 `nx.config.backend_priority` 配置的优先级
4. **检查后端可用性**：
   - `can_run()`：后端能否处理这些参数？
   - `should_run()`：后端是否应该处理（性能考量）？
5. **执行调用**：
   - 如果输入图已经是后端图类型，直接调用后端函数
   - 否则，尝试转换图到后端类型，然后调用
   - 如果转换失败或后端不支持，fallback 到 NetworkX 原始实现

### 4.3 三类算法对通用图接口的复用

#### 4.3.1 匹配算法的接口复用

**`hopcroft_karp_matching` 中的接口调用**：
**关键代码位置**：`networkx/algorithms/bipartite/matching.py:127-182`

```python
def breadth_first_search():
    for v in left:
        if leftmatches[v] is None:
            distances[v] = 0
            queue.append(v)
        else:
            distances[v] = INFINITY
    distances[None] = INFINITY
    while queue:
        v = queue.popleft()
        if distances[v] < distances[None]:
            for u in G[v]:  # ◀── 使用 G[v] 访问邻居（等价于 G.adj[v]）
                if distances[rightmatches[u]] is INFINITY:
                    distances[rightmatches[u]] = distances[v] + 1
                    queue.append(rightmatches[u])
    return distances[None] is not INFINITY

def depth_first_search(v):
    if v is not None:
        for u in G[v]:  # ◀── 再次使用 G[v]
            if distances[rightmatches[u]] == distances[v] + 1:
                if depth_first_search(rightmatches[u]):
                    rightmatches[u] = v
                    leftmatches[v] = u
                    return True
        distances[v] = INFINITY
        return False
    return True
```

**匹配算法复用的接口**：

| 接口 | 代码位置 | 用途 |
|------|---------|------|
| `G[v]` | `matching.py:138, 146` | 访问节点 `v` 的邻居 |
| `bipartite_sets()` | `matching.py:157` | 获取节点划分（内部调用 `color()`） |
| `nx.is_connected()` | 间接，通过 `bipartite_sets` | 判断图的连通性 |

#### 4.3.2 投影算法的接口复用

**`projected_graph` 中的接口调用**：
**关键代码位置**：`networkx/algorithms/bipartite/projection.py:89-118`

```python
if B.is_multigraph():  # ◀── 判断是否多重图
    raise nx.NetworkXError("not defined for multigraphs")
if B.is_directed():     # ◀── 判断是否有向图
    directed = True
    if multigraph:
        G = nx.MultiDiGraph()  # ◀── 创建输出图
    else:
        G = nx.DiGraph()       # ◀── 创建输出图
else:
    directed = False
    if multigraph:
        G = nx.MultiGraph()     # ◀── 创建输出图
    else:
        G = nx.Graph()          # ◀── 创建输出图

G.graph.update(B.graph)  # ◀── 复制图属性
G.add_nodes_from((n, B.nodes[n]) for n in nodes)  # ◀── 复制节点及属性

for u in nodes:
    nbrs2 = {v for nbr in B[u] for v in B[nbr] if v != u}  # ◀── B[u] 访问邻居
    if multigraph:
        for n in nbrs2:
            if directed:
                links = set(B[u]) & set(B.pred[n])  # ◀── B.pred 访问前驱（有向图）
            else:
                links = set(B[u]) & set(B[n])       # ◀── B[n] 访问邻居
            for l in links:
                if not G.has_edge(u, n, l):  # ◀── 检查边是否存在
                    G.add_edge(u, n, key=l)   # ◀── 添加边
    else:
        G.add_edges_from((u, n) for n in nbrs2)  # ◀── 批量添加边
```

**`weighted_projected_graph` 中的额外接口**：
**关键代码位置**：`networkx/algorithms/bipartite/projection.py:192-218`

```python
if B.is_directed():
    pred = B.pred   # ◀── 有向图使用 pred
    G = nx.DiGraph()
else:
    pred = B.adj     # ◀── 无向图使用 adj
    G = nx.Graph()

for u in nodes:
    unbrs = set(B[u])           # ◀── B[u] 访问邻居
    nbrs2 = {n for nbr in unbrs for n in B[nbr]} - {u}
    for v in nbrs2:
        vnbrs = set(pred[v])    # ◀── 使用 pred/adj
        common = unbrs & vnbrs
        # ...
        G.add_edge(u, v, weight=weight)  # ◀── 添加带权边
```

**投影算法复用的接口**：

| 接口 | 代码位置 | 用途 |
|------|---------|------|
| `B.is_multigraph()` | `projection.py:89` | 判断是否多重图 |
| `B.is_directed()` | `projection.py:91` | 判断是否有向图 |
| `B[u]` | `projection.py:106, 209` | 访问邻居 |
| `B.pred` | `projection.py:110, 193` | 有向图访问前驱 |
| `B.adj` | `projection.py:196` | 无向图访问邻接表 |
| `B.nodes[n]` | `projection.py:104` | 访问节点属性 |
| `B.graph` | `projection.py:103` | 访问图属性字典 |
| `G = nx.Graph()` 等 | `projection.py:96, 102` | 创建输出图 |
| `G.add_nodes_from()` | `projection.py:104` | 批量添加节点 |
| `G.add_edge()` | `projection.py:115, 218` | 添加边 |
| `G.add_edges_from()` | `projection.py:117` | 批量添加边 |
| `G.has_edge()` | `projection.py:114` | 检查边是否存在 |
| `G.graph.update()` | `projection.py:103` | 更新图属性 |

#### 4.3.3 中心性算法的接口复用

**`degree_centrality` 中的接口调用**：
**关键代码位置**：`networkx/algorithms/bipartite/centrality.py:72-78`

```python
top = set(nodes)
bottom = set(G) - top  # ◀── set(G) 获取所有节点（通过 __iter__）
s = 1.0 / len(bottom)
centrality = {n: d * s for n, d in G.degree(top)}  # ◀── G.degree() 获取度数
s = 1.0 / len(top)
centrality.update({n: d * s for n, d in G.degree(bottom)})
```

**`betweenness_centrality` 中的接口调用**：
**关键代码位置**：`networkx/algorithms/bipartite/centrality.py:177`

```python
# 直接调用通用图的 betweenness_centrality
betweenness = nx.betweenness_centrality(G, normalized=False, weight=None)
# 然后使用二部图特定的归一化因子
for node in top:
    betweenness[node] /= bet_max_top
for node in bottom:
    betweenness[node] /= bet_max_bot
```

**`closeness_centrality` 中的接口调用**：
**关键代码位置**：`networkx/algorithms/bipartite/centrality.py:264-290`

```python
path_length = nx.single_source_shortest_path_length  # ◀── 通用图算法

for node in top:
    sp = dict(path_length(G, node))  # ◀── 计算单源最短路径
    totsp = sum(sp.values())
    if totsp > 0.0 and len(G) > 1:
        closeness[node] = (m + 2 * (n - 1)) / totsp  # ◀── 二部图特定公式
        if normalized:
            s = (len(sp) - 1) / (len(G) - 1)
            closeness[node] *= s
    else:
        closeness[node] = 0.0
```

**中心性算法复用的接口**：

| 接口 | 代码位置 | 用途 |
|------|---------|------|
| `set(G)` | `centrality.py:73` | 获取所有节点（通过 `__iter__`） |
| `len(G)` | `centrality.py:163, 268, 276` | 获取节点数（通过 `__len__`） |
| `G.degree(nodes)` | `centrality.py:75, 77` | 获取节点度数 |
| `nx.betweenness_centrality()` | `centrality.py:177` | 调用通用图算法 |
| `nx.single_source_shortest_path_length()` | `centrality.py:265` | 调用通用图算法 |

### 4.4 接口复用的深度对比

| 接口类型 | 匹配算法 | 投影算法 | 中心性算法 |
|---------|---------|---------|-----------|
| **邻居访问** | `G[v]` | `B[u]`, `B.pred`, `B.adj` | 间接（通过调用通用算法） |
| **度数获取** | ❌ 不直接使用 | ❌ 不直接使用 | ✅ `G.degree()` |
| **图类型判断** | ❌ 不判断 | ✅ `is_multigraph()`, `is_directed()` | ❌ 不判断 |
| **图创建** | ❌ 不创建 | ✅ `nx.Graph()`, `nx.DiGraph()` 等 | ❌ 不创建 |
| **节点/边操作** | ❌ 不修改图 | ✅ `add_nodes_from()`, `add_edge()` 等 | ❌ 不修改图 |
| **属性复制** | ❌ 不需要 | ✅ `graph.update()`, `nodes[n]` | ❌ 不需要 |
| **通用算法调用** | ❌ 不调用 | ❌ 不调用 | ✅ `nx.betweenness_centrality()`, `nx.single_source_shortest_path_length()` |
| **二部图特定调用** | ✅ `bipartite_sets()` | ❌ 不调用 | ❌ 不调用 |

---

## 5. 调用路径流程图

### 5.1 `hopcroft_karp_matching` 完整调用路径

```
用户调用: bipartite.maximum_matching(G, top_nodes=None)
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  @nx._dispatchable 装饰器 (__call__)                          │
│  - 检查后端安装情况                                           │
│  - 无后端时直接调用 orig_func                                 │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  hopcroft_karp_matching(G, top_nodes=None)                   │
│  位置: matching.py:59                                         │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  bipartite_sets(G, top_nodes)                                 │
│  位置: matching.py:157 → basic.py:158                        │
└──────────────────────────────────────────────────────────────┘
     │
     ├──────────── top_nodes is None? ────────────┐
     │                                             │
     ▼ (No)                                       ▼ (Yes)
┌─────────────────┐                   ┌─────────────────────────┐
│ X = set(top_nodes) │                   │ is_connected(G)?       │
│ Y = set(G) - X    │                   │ 位置: basic.py:209     │
└─────────────────┘                   └─────────────────────────┘
                                               │
                        ┌──────────────────────┼──────────────────────┐
                        ▼ (Yes)                ▼ (No)
              ┌─────────────────┐    ┌─────────────────────────┐
              │ color(G)        │    │ raise AmbiguousSolution  │
              │ 位置: basic.py:217 │    │ 位置: basic.py:215     │
              └─────────────────┘    └─────────────────────────┘
                        │
                        ▼
              ┌─────────────────────────┐
              │ X = {n for n, is_top    │
              │         in c.items()    │
              │         if is_top}      │
              │ Y = {n for n, is_top    │
              │         in c.items()    │
              │         if not is_top}  │
              └─────────────────────────┘
     │
     └─────────────────────────────────────┐
                                           ▼
┌──────────────────────────────────────────────────────────────┐
│  初始化匹配数据结构                                           │
│  leftmatches = {v: None for v in left}                       │
│  rightmatches = {v: None for v in right}                      │
│  位置: matching.py:158-159                                    │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  Hopcroft-Karp 主循环                                         │
│  while breadth_first_search():                                │
│      for v in left:                                            │
│          if leftmatches[v] is None:                           │
│              depth_first_search(v)                             │
│  位置: matching.py:166-170                                    │
└──────────────────────────────────────────────────────────────┘
     │
     ├──────────────────────────────────────────────────────────┐
     │                                                          │
     ▼                                                          ▼
┌─────────────────┐                                ┌─────────────────┐
│ BFS 分层        │                                │ DFS 增广        │
│ 位置: matching.py:127 │                                │ 位置: matching.py:144 │
└─────────────────┘                                └─────────────────┘
     │                                                          │
     └─────────────────── 通用图接口 ───────────────────────────┘
                              │
                              ▼
              ┌─────────────────────────────────┐
              │  G[v] - 访问邻居                │
              │  位置: matching.py:138, 146    │
              │  等价于 G.adj[v]                │
              └─────────────────────────────────┘
```

### 5.2 `projected_graph` 完整调用路径

```
用户调用: bipartite.projected_graph(B, nodes)
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  @nx._dispatchable 装饰器 (__call__)                          │
│  - graphs="B" 表示图参数是 B                                 │
│  - preserve_node_attrs=True, preserve_graph_attrs=True       │
│  - returns_graph=True 表示返回图对象                         │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  projected_graph(B, nodes, multigraph=False)                  │
│  位置: projection.py:18                                       │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  图类型判断                                                    │
│  if B.is_multigraph(): raise NetworkXError                   │
│  if B.is_directed(): directed = True                          │
│  位置: projection.py:89-98                                    │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  创建输出图                                                    │
│  directed + multigraph ? nx.MultiDiGraph()                   │
│  directed + !multigraph ? nx.DiGraph()                       │
│  !directed + multigraph ? nx.MultiGraph()                    │
│  !directed + !multigraph ? nx.Graph()                        │
│  位置: projection.py:91-102                                   │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  复制属性和节点                                                │
│  G.graph.update(B.graph)                                      │
│  G.add_nodes_from((n, B.nodes[n]) for n in nodes)           │
│  位置: projection.py:103-104                                  │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  投影主循环                                                    │
│  for u in nodes:                                              │
│      nbrs2 = {v for nbr in B[u] for v in B[nbr] if v != u} │
│      if multigraph:                                           │
│          # 多重图处理                                          │
│      else:                                                     │
│          G.add_edges_from((u, n) for n in nbrs2)            │
│  位置: projection.py:105-117                                  │
└──────────────────────────────────────────────────────────────┘
     │
     └────────────────────── 通用图接口 ───────────────────────┘
                              │
                              ▼
         ┌──────────────────────────────────────────────┐
         │  B.is_multigraph() - 判断多重图              │
         │  B.is_directed() - 判断有向图                │
         │  B[u] - 访问邻居                              │
         │  B.pred - 有向图访问前驱                      │
         │  B.adj - 无向图访问邻接表                     │
         │  B.nodes[n] - 访问节点属性                    │
         │  B.graph - 访问图属性                         │
         │  nx.Graph() / nx.DiGraph() - 创建图         │
         │  G.add_nodes_from() - 添加节点               │
         │  G.add_edge() / G.add_edges_from() - 添加边  │
         │  G.has_edge() - 检查边是否存在               │
         │  G.graph.update() - 更新图属性               │
         └──────────────────────────────────────────────┘
```

### 5.3 `betweenness_centrality` 完整调用路径

```
用户调用: bipartite.betweenness_centrality(G, nodes)
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  @nx._dispatchable 装饰器 (__call__)                          │
│  - name="bipartite_betweenness_centrality" (避免命名冲突)    │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  betweenness_centrality(G, nodes)                             │
│  位置: centrality.py:82                                       │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  计算节点划分                                                  │
│  top = set(nodes)                                             │
│  bottom = set(G) - top                                        │
│  n = len(top)                                                 │
│  m = len(bottom)                                              │
│  位置: centrality.py:161-164                                  │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  计算二部图特定的归一化因子                                    │
│  s, t = divmod(n - 1, m)                                      │
│  bet_max_top = (m² * (s+1)² + m*(s+1)*(2t-s-1)             │
│                - t*(2s-t+3)) / 2                              │
│  # 类似地计算 bet_max_bot                                     │
│  位置: centrality.py:165-176                                  │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  调用通用图的 betweenness_centrality                           │
│  betweenness = nx.betweenness_centrality(                     │
│      G, normalized=False, weight=None                         │
│  )                                                             │
│  位置: centrality.py:177                                      │
└──────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────┐
│  使用二部图因子归一化                                          │
│  for node in top:                                             │
│      betweenness[node] /= bet_max_top                         │
│  for node in bottom:                                          │
│      betweenness[node] /= bet_max_bot                         │
│  位置: centrality.py:178-181                                  │
└──────────────────────────────────────────────────────────────┘
     │
     └────────────────────── 通用图接口 ───────────────────────┘
                              │
                              ▼
         ┌──────────────────────────────────────────────┐
         │  set(G) - 获取所有节点 (__iter__)            │
         │  len(G) - 获取节点数 (__len__)               │
         │  nx.betweenness_centrality() - 通用算法调用  │
         └──────────────────────────────────────────────┘
```

---

## 6. 总结

### 6.1 三类算法的核心差异

| 维度 | 匹配算法 | 投影算法 | 中心性算法 |
|------|---------|---------|-----------|
| **参数必需性** | `top_nodes` 可选 | `nodes` 必需 | `nodes` 必需 |
| **自动划分** | 支持（连通图） | 不支持 | 不支持 |
| **非连通图策略** | 无 `top_nodes` 时报 `AmbiguousSolution` | 直接使用用户输入 | 直接使用用户输入 |
| **二部图验证** | 自动计算时验证 | 不验证 | 不验证 |
| **通用接口复用方式** | 直接使用 `G[v]` | 图类型判断 + 图创建 + 节点/边操作 | 调用通用图算法 |

### 6.2 设计哲学

1. **"拒绝猜测"**：非连通图不猜测着色方案，强制用户显式指定或按连通分量处理
2. **"约定优于配置"**：`bipartite` 节点属性约定，但不强制
3. **"组合优于继承"**：通过算法组合而非类继承实现二部图功能
4. **"后端透明"**：`@_dispatchable` 让算法实现与后端调度解耦

### 6.3 实际使用建议

1. **非连通二部图**：始终显式提供 `top_nodes`/`nodes`，或按连通分量分别处理
2. **投影算法**：`nodes` 可以是任意节点子集，不一定是完整二分划
3. **中心性算法**：确保 `nodes` 是正确的二分划，否则归一化因子会错误
4. **性能考量**：投影和中心性算法不验证二部图性质，需要确保输入正确

### 6.4 代码位置速查

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| `hopcroft_karp_matching` | `bipartite/matching.py` | 59 |
| `bipartite_sets` | `bipartite/basic.py` | 158 |
| `color` (二着色) | `bipartite/basic.py` | 21 |
| `AmbiguousSolution` 抛出 | `bipartite/basic.py` | 215 |
| `projected_graph` | `bipartite/projection.py` | 18 |
| `degree_centrality` | `bipartite/centrality.py` | 7 |
| `@_dispatchable` 定义 | `utils/backends.py` | 215 |
| 无后端快速路径 | `utils/backends.py` | 541 |
| 有后端调度逻辑 | `utils/backends.py` | 554 |
