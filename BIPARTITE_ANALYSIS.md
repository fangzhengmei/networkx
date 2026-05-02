# NetworkX 二部图算法包与通用图接口适配分析

## 目录

1. [概述](#1-概述)
2. [架构设计：无专门二部图类](#2-架构设计无专门二部图类)
3. [通用图接口复用机制](#3-通用图接口复用机制)
4. [节点集合划分传递机制](#4-节点集合划分传递机制)
5. [后端调度与 `_dispatchable` 装饰器](#5-后端调度与-_dispatchable-装饰器)
6. [设计模式与架构权衡](#6-设计模式与架构权衡)
7. [总结](#7-总结)

---

## 1. 概述

### 1.1 二部图算法包定位

NetworkX 的二部图（Bipartite）算法包位于 `networkx/algorithms/bipartite/` 目录下，包含以下模块：

| 模块 | 功能 |
|------|------|
| `basic.py` | 基础操作：二部图判断、着色、集合划分、密度计算 |
| `matching.py` | 匹配算法：Hopcroft-Karp、Eppstein、最小权完美匹配 |
| `projection.py` | 投影算法：单模投影、加权投影 |
| `centrality.py` | 中心性计算：度中心性、中介中心性、接近中心性 |
| `generators.py` | 二部图生成器 |
| `matrix.py` | 邻接矩阵操作 |
| `cluster.py` | 聚类算法 |
| `covering.py` | 覆盖算法 |
| `edgelist.py` | 边列表操作 |
| `redundancy.py` | 冗余分析 |
| `spectral.py` | 谱方法 |
| `extendability.py` | 可扩展性分析 |
| `link_analysis.py` | 链接分析 |

### 1.2 核心设计理念

NetworkX 对二部图的处理采用了**"约定优于配置**的设计哲学：

- **无专门的二部图类**：复用通用的 `Graph`/`DiGraph`/`MultiGraph`/`MultiDiGraph`
- **节点属性约定**：使用 `bipartite` 节点属性标识集合归属（值为 0 或 1）
- **显式参数传递**：大多数算法要求显式传入节点集合参数，避免歧义

---

## 2. 架构设计：无专门二部图类

### 2.1 为什么没有 BipartiteGraph 类？

NetworkX 没有为二部图设计独立的类，这是一个有意识的设计决策。从 `__init__.py` 的文档字符串可以看出：

> "NetworkX does not have a custom bipartite graph class but the Graph() or DiGraph() classes can be used to represent bipartite graphs."

这种设计带来的优势：

1. **代码复用最大化**：二部图可以直接使用所有通用图算法
2. **学习曲线平缓**：用户无需学习新的图类 API
3. **灵活性高**：同一图对象可以在二部图和通用图算法间切换使用

### 2.2 数据结构复用

二部图完全复用通用图的三层字典存储结构：

```
Graph 类的核心数据结构：
├── graph          # 图属性字典
├── _node        # 节点属性字典：{node: {attr1: val1, attr2: val2, ...}}
├── _adj          # 邻接表：{node: {neighbor: {edge_attrs}}}
└── __networkx_cache__  # 后端缓存
```

**关键代码位置**：
- `networkx/classes/graph.py:381-383`

```python
self.graph = self.graph_attr_dict_factory()  # 字典用于图属性
self._node = self.node_dict_factory()        # 空节点属性字典
self._adj = self.adjlist_outer_dict_factory()  # 空邻接表字典
```

---

## 3. 通用图接口复用机制

### 3.1 存储接口复用

二部图算法通过以下方式复用通用图的存储能力：

#### 3.1.1 节点属性存储

二部图的 `bipartite` 属性存储在 `_node` 字典中：

**关键代码位置**：
- `networkx/algorithms/bipartite/generators.py:61-62`

```python
G.add_nodes_from(top, bipartite=0)
G.add_nodes_from(bottom, bipartite=1)
```

`_add_nodes_with_bipartite_label` 辅助函数：
- `networkx/algorithms/bipartite/generators.py:598-602`

```python
def _add_nodes_with_bipartite_label(G, lena, lenb):
    G.add_nodes_from(range(lena + lenb))
    b = dict(zip(range(lena), [0] * lena))
    b.update(dict(zip(range(lena, lena + lenb), [1] * lenb)))
    nx.set_node_attributes(G, b, "bipartite")
    return G
```

#### 3.1.2 邻接关系存储

边的存储完全复用通用图的邻接表结构，无需任何修改。

### 3.2 遍历接口复用

二部图算法大量使用通用图的遍历接口：

| 接口 | 用途 | 使用位置示例 |
|------|------|--------------|
| `G.neighbors(v)` | 获取邻居节点 | `basic.py:62` |
| `G[v]` | 下标访问邻居 | `matching.py:138, 146` |
| `G.adj` | 邻接表视图 | `projection.py:196, 299` |
| `for n in G` | 节点迭代 | `basic.py:65` |
| `G.degree(nodes)` | 获取度数 | `centrality.py:75, 77` |
| `G.edges()` | 边迭代 | `matching.py:411` |
| `G.nodes(data=True)` | 获取节点及属性 | `generators.py:583` |

#### 3.2.1 邻居访问示例

**`color()` 函数中的邻居处理**：
- `networkx/algorithms/bipartite/basic.py:55-62`

```python
if G.is_directed():
    import itertools
    def neighbors(v):
        return itertools.chain.from_iterable([G.predecessors(v), G.successors(v)])
else:
    neighbors = G.neighbors
```

这个设计展示了如何统一处理有向图和无向图的邻居访问。

#### 3.2.2 邻接表在投影算法中的使用

**`projected_graph()` 中的邻居遍历：
- `networkx/algorithms/bipartite/projection.py:106`

```python
nbrs2 = {v for nbr in B[u] for v in B[nbr] if v != u}
```

使用字典推导式通过 `B[u]` 直接访问邻居，然后通过 `B[nbr]` 访问邻居的邻居。

### 3.3 类型判断接口复用

二部图算法需要根据图的类型调整行为：

| 方法 | 用途 | 使用位置 |
|------|------|----------|
| `G.is_directed()` | 判断是否有向图 | `basic.py:55, projection.py:89` |
| `G.is_multigraph()` | 判断是否多重图 | `projection.py:89` |

**示例**：`projection.py` 中根据图类型选择输出图类型：
- `networkx/algorithms/bipartite/projection.py:89-102`

```python
if B.is_multigraph():
    raise nx.NetworkXError("not defined for multigraphs")
if B.is_directed():
    directed = True
    if multigraph:
        G = nx.MultiDiGraph()
    else:
        G = nx.DiGraph()
else:
    directed = False
    if multigraph:
        G = nx.MultiGraph()
    else:
        G = nx.Graph()
```

### 3.4 图操作接口复用

| 操作 | 用途 | 使用位置 |
|------|------|----------|
| `G.subgraph(c)` | 创建子图 | `basic.py:148` |
| `G.add_edge(u, v)` | 添加边 | `projection.py:115, 218` |
| `G.add_nodes_from()` | 批量添加节点 | `projection.py:104` |
| `G.has_edge(u, v)` | 判断边是否存在 | `projection.py:114` |

### 3.5 通用图算法复用

二部图算法还直接调用通用图算法：

| 通用算法 | 用途 | 调用位置 |
|----------|------|----------|
| `nx.is_connected()` | 无向图连通性判断 | `basic.py:209` |
| `nx.is_weakly_connected()` | 有向图弱连通判断 | `basic.py:207` |
| `nx.isolates(G)` | 获取孤立点 | `basic.py:81` |
| `nx.connected_components(G)` | 获取连通分量 | `basic.py:148` |
| `nx.betweenness_centrality()` | 中介中心性 | `centrality.py:177` |
| `nx.single_source_shortest_path_length()` | 单源最短路径 | `centrality.py:265` |

**示例**：`betweenness_centrality()` 复用通用中心性算法：
- `networkx/algorithms/bipartite/centrality.py:177`

```python
betweenness = nx.betweenness_centrality(G, normalized=False, weight=None)
for node in top:
    betweenness[node] /= bet_max_top
for node in bottom:
    betweenness[node] /= bet_max_bot
```

---

## 4. 节点集合划分传递机制

### 4.1 三种传递方式

二部图算法通过三种方式获取节点集合划分：

```
┌─────────────────────────────────────────────────────────────┐
│                    节点集合划分传递方式                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌───────────┐ │
│  │ 1. 显式参数  │───▶│ 2. 自动计算  │───▶│ 3. 节点属性 │ │
│  │  top_nodes  │    │   color()   │    │  bipartite │ │
│  └─────────────┘    └─────────────┘    └───────────┘ │
│       │                    │                    │          │
│       ▼                    ▼                    ▼          │
│  ┌─────────────────────────────────────────────────┐ │
│  │              算法层使用节点集合信息                │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 方式一：显式参数传递

这是最主要的传递方式，大多数二部图算法都接受 `top_nodes` 或 `nodes` 参数。

#### 4.2.1 参数签名示例

| 算法函数 | 参数名 | 位置 |
|----------|--------|------|
| `hopcroft_karp_matching` | `top_nodes` | `matching.py:59` |
| `eppstein_matching` | `top_nodes` | `matching.py:186` |
| `to_vertex_cover` | `top_nodes` | `matching.py:425` |
| `minimum_weight_full_matching` | `top_nodes` | `matching.py:506` |
| `degree_centrality` | `nodes` | `centrality.py:7` |
| `betweenness_centrality` | `nodes` | `centrality.py:82` |
| `closeness_centrality` | `nodes` | `centrality.py:186` |
| `projected_graph` | `nodes` | `projection.py:18` |
| `density` | `nodes` | `basic.py:224` |
| `degrees` | `nodes` | `basic.py:274` |

#### 4.2.2 算法内部处理

以 `hopcroft_karp_matching` 为例：
- `networkx/algorithms/bipartite/matching.py:157`

```python
left, right = bipartite_sets(G, top_nodes)
leftmatches = {v: None for v in left}
rightmatches = {v: None for v in right}
```

算法首先调用 `bipartite_sets()` 函数获取两个节点集合，然后分别初始化匹配字典。

### 4.3 方式二：自动计算（二着色算法）

当未提供 `top_nodes` 时，算法尝试通过 BFS 二着色自动计算。

#### 4.3.1 `color()` 函数实现

**关键代码位置**：
- `networkx/algorithms/bipartite/basic.py:21-82`

```python
@nx._dispatchable
def color(G):
    """Returns a two-coloring of the graph.
    ...
    """
    if G.is_directed():
        import itertools
        def neighbors(v):
            return itertools.chain.from_iterable([G.predecessors(v), G.successors(v)])
    else:
        neighbors = G.neighbors

    color = {}
    for n in G:  # 处理非连通图
        if n in color or len(G[n]) == 0:  # 跳过孤立点
            continue
        queue = [n]
        color[n] = 1  # 节点颜色 (1 or 0)
        while queue:
            v = queue.pop()
            c = 1 - color[v]  # 节点 v 的相反颜色
            for w in neighbors(v):
                if w in color:
                    if color[w] == color[v]:
                        raise nx.NetworkXError("Graph is not bipartite.")
                else:
                    color[w] = c
                    queue.append(w)
    # 孤立点着色为 0
    color.update(dict.fromkeys(nx.isolates(G), 0))
    return color
```

#### 4.3.2 `sets()` 函数的集合划分

**关键代码位置**：
- `networkx/algorithms/bipartite/basic.py:158-220`

```python
@nx._dispatchable
def sets(G, top_nodes=None):
    if G.is_directed():
        is_connected = nx.is_weakly_connected
    else:
        is_connected = nx.is_connected
    if top_nodes is not None:
        X = set(top_nodes)
        Y = set(G) - X
    else:
        if not is_connected(G):
            msg = "Disconnected graph: Ambiguous solution for bipartite sets."
            raise nx.AmbiguousSolution(msg)
        c = color(G)
        X = {n for n, is_top in c.items() if is_top}
        Y = {n for n, is_top in c.items() if not is_top}
    return (X, Y)
```

**重要设计决策**：

1. **非连通图抛出异常**：如果图不连通且未提供 `top_nodes`，抛出 `AmbiguousSolution` 异常
2. **孤立点处理**：孤立点被着色为 0
3. **有向图使用弱连通性**：有向图使用 `is_weakly_connected` 判断连通性

### 4.4 方式三：节点属性约定（推荐但非强制）

NetworkX 推荐使用 `bipartite` 节点属性，但算法本身并不强制依赖它。

#### 4.4.1 生成器自动设置属性

所有二部图生成器都会自动设置 `bipartite` 属性：

**`complete_bipartite_graph` 示例**：
- `networkx/algorithms/bipartite/generators.py:61-62`

```python
G.add_nodes_from(top, bipartite=0)
G.add_nodes_from(bottom, bipartite=1)
```

#### 4.4.2 用户手动获取集合

文档推荐的用法：

```python
# 方式1：通过 color() 函数设置属性
c = nx.bipartite.color(G)
nx.set_node_attributes(G, c, name="bipartite")

# 方式2：通过节点属性获取集合
top_nodes = {n for n, d in G.nodes(data=True) if d["bipartite"] == 0}
bottom_nodes = set(G) - top_nodes
```

#### 4.4.3 为什么算法不直接读取属性？

从设计文档可以看出，算法不直接读取 `bipartite` 属性的原因：

1. **向后兼容**：早期版本可能没有这个属性
2. **灵活性**：用户可能使用其他方式管理节点集合
3. **非强制约定**：文档明确说明 "This convention is not enforced in the source code"

### 4.5 非连通图的模糊性处理

#### 4.5.1 问题本质

非连通二部图存在多种合法的二着色方案：

```
示例：两个不连通的边
G: 1-2, 3-4

方案1：
  X = {1, 3}, Y = {2, 4}
方案2：
  X = {1, 4}, Y = {2, 3}  ← 同样合法！
```

#### 4.5.2 NetworkX 的策略

| 场景 | 处理方式 |
|------|----------|
| 连通图 + 无 top_nodes | 自动二着色，结果唯一 |
| 非连通图 + 无 top_nodes | 抛出 `AmbiguousSolution` |
| 任何图 + 有 top_nodes | 使用用户提供的集合 |

#### 4.5.3 `is_bipartite_node_set()` 的验证

**关键代码位置**：
- `networkx/algorithms/bipartite/basic.py:114-154`

```python
@nx._dispatchable
def is_bipartite_node_set(G, nodes):
    S = set(nodes)
    
    if len(S) < len(nodes):
        raise nx.AmbiguousSolution(
            "The input node set contains duplicates.\n"
            "This may lead to incorrect results when using it in bipartite algorithms.\n"
            "Consider using set(nodes) as the input"
        )
    
    for CC in (G.subgraph(c).copy() for c in connected_components(G)):
        X, Y = sets(CC)
        if not (
            (X.issubset(S) and Y.isdisjoint(S)) or (Y.issubset(S) and X.isdisjoint(S))
        ):
            return False
    return True
```

这个函数验证节点集合是否是有效的二部划分，对每个连通分量分别检查。

---

## 5. 后端调度与 `_dispatchable` 装饰器

### 5.1 装饰器概述

所有二部图算法都使用 `@nx._dispatchable` 装饰器，这是 NetworkX 3.x 引入的后端调度机制。

### 5.2 装饰器参数

**关键代码位置**：
- `networkx/utils/backends.py:215-350`

| 参数 | 说明 | 二部图中的使用 |
|------|------|----------------|
| `name` | 调度名称，避免命名冲突 | `name="bipartite_degree_centrality"` |
| `graphs` | 图参数的位置和名称 | 默认 `"G"`, `"B"` |
| `edge_attrs` | 边属性参数 | `edge_attrs="weight"` |
| `returns_graph` | 是否返回图对象 | 生成器和投影函数使用 |

### 5.3 二部图中的使用示例

#### 5.3.1 基础算法

```python
@nx._dispatchable
def color(G): ...

@nx._dispatchable
def is_bipartite(G): ...

@nx._dispatchable(graphs="B")
def density(B, nodes): ...

@nx._dispatchable(graphs="B", edge_attrs="weight")
def degrees(B, nodes, weight=None): ...
```

#### 5.3.2 中心性算法（命名冲突处理）

```python
@nx._dispatchable(name="bipartite_degree_centrality")
def degree_centrality(G, nodes): ...

@nx._dispatchable(name="bipartite_betweenness_centrality")
def betweenness_centrality(G, nodes): ...

@nx._dispatchable(name="bipartite_closeness_centrality")
def closeness_centrality(G, nodes, normalized=True): ...
```

使用 `name` 参数避免与通用图的 `degree_centrality` 等函数命名冲突。

#### 5.3.3 生成器函数

```python
@nx._dispatchable(graphs=None, returns_graph=True)
@nodes_or_number([0, 1])
def complete_bipartite_graph(n1, n2, create_using=None): ...
```

- `graphs=None` 表示没有输入图参数
- `returns_graph=True` 表示返回图对象

#### 5.3.4 投影函数

```python
@nx._dispatchable(
    graphs="B", preserve_node_attrs=True, preserve_graph_attrs=True, returns_graph=True
)
def projected_graph(B, nodes, multigraph=False): ...
```

- `preserve_node_attrs` 和 `preserve_graph_attrs` 指示后端需要保留属性。

### 5.4 调度机制的作用

`_dispatchable` 装饰器实现了：

1. **后端自动选择**：根据输入图类型自动选择最优后端实现
2. **属性管理**：处理图属性、节点属性、边属性的转换
3. **命名空间管理**：通过 `name` 参数避免不同模块的函数命名冲突
4. **缓存机制**：支持图转换结果的缓存

---

## 6. 设计模式与架构权衡

### 6.1 设计模式应用

#### 6.1.1 约定优于配置（Convention over Configuration）

- **应用**：`bipartite` 节点属性约定
- **位置**：`__init__.py` 文档
- **效果**：减少配置，提高互操作性

#### 6.1.2 策略模式（Strategy Pattern）

- **应用**：`_dispatchable` 装饰器的后端调度
- **位置**：`utils/backends.py`
- **效果**：运行时选择不同的算法实现策略

#### 6.1.3 模板方法模式（Template Method Pattern）

- **应用**：二部图算法的统一结构
- **位置**：所有二部图算法
- **效果**：
  1. 获取节点集合（`top_nodes` 或自动计算`
  2. 执行二部图特定逻辑
  3. 返回结果

#### 6.1.4 适配器模式（Adapter Pattern）

- **应用**：有向图/无向图的统一邻居访问
- **位置**：`basic.py:55-62`
- **效果**：统一处理 `G.neighbors` 和 `G.predecessors/G.successors`

### 6.2 架构权衡分析

#### 6.2.1 优势

| 方面 | 说明 |
|------|------|
| **代码复用** | 二部图无需重新实现图的存储和遍历 |
| **API 一致性** | 用户学习成本低，通用图知识直接应用于二部图 |
| **灵活性** | 同一图可在二部图和通用图算法间切换 |
| **可扩展性** | `_dispatchable` 支持后端插件机制 |

#### 6.2.2 权衡代价

| 方面 | 说明 |
|------|------|
| **无类型安全** | 无法在编译/运行时强制二部图约束（如同类节点无边） |
| **用户负担** | 用户需自行管理节点集合或属性 |
| **异常风险** | 非连通图需显式提供节点集合，否则抛异常 |
| **性能开销** | 每次算法调用都可能需要重新计算节点划分 |

#### 6.2.3 设计决策背后的哲学

NetworkX 的选择反映了以下设计哲学：

1. **"拒绝猜测"（Refuse the temptation to guess**：非连通图不猜测着色方案，强制用户显式提供
2. **"约定优于强制**：提供 `bipartite` 属性约定，但不强制
3. **"组合优于继承"**：通过算法组合而非类继承实现二部图功能

---

## 7. 总结

### 7.1 核心架构要点

1. **无专门二部图类**：复用 `Graph`/`DiGraph`/`MultiGraph`/`MultiDiGraph`

2. **三层数据结构**：
   - `_node` 字典存储 `bipartite` 属性（约定）
   - `_adj` 邻接表存储边关系（完全复用）

3. **接口复用**：
   - 遍历：`G.neighbors()`, `G[v]`, `G.adj`, `G.degree()`
   - 类型判断：`G.is_directed()`, `G.is_multigraph()`
   - 图操作：`G.subgraph()`, `G.add_edge()`
   - 算法复用：`nx.is_connected()`, `nx.betweenness_centrality()` 等

4. **节点集合传递**：
   - 显式参数：`top_nodes`/`nodes` 参数
   - 自动计算：`color()` BFS 二着色
   - 属性约定：`bipartite` 节点属性（推荐）

5. **后端调度**：`@nx._dispatchable` 装饰器支持多后端

### 7.2 与其他图库的对比

| 特性 | NetworkX | 其他图库（如 igraph） |
|------|----------|------------------------|
| 二部图类 | 无专门类 | 通常有 `BipartiteGraph` 类 |
| 类型检查 | 运行时 `is_bipartite()` | 编译/构造时保证 |
| 节点划分 | 参数传递/自动计算/属性约定 | 类内部管理 |
| 算法复用 | 完全复用通用算法 | 二部图算法独立 |

### 7.3 适用场景

NetworkX 的二部图设计最适合：

1. **研究和探索**：需要灵活切换二部图和通用图分析
2. **快速原型**：无需预先定义图类型
3. **教育目的**：学习图算法，理解二部图概念
4. **算法组合**：组合使用二部图和通用图算法

---

## 附录

### A. 关键文件索引

| 文件 | 路径 | 功能 |
|------|------|------|
| 二部图入口 | `networkx/algorithms/bipartite/__init__.py` | 模块文档和导出 |
| 基础操作 | `networkx/algorithms/bipartite/basic.py` | 二着色、集合划分 |
| 匹配算法 | `networkx/algorithms/bipartite/matching.py` | Hopcroft-Karp 等 |
| 投影算法 | `networkx/algorithms/bipartite/projection.py` | 单模投影 |
| 中心性 | `networkx/algorithms/bipartite/centrality.py` | 二部图中心性 |
| 生成器 | `networkx/algorithms/bipartite/generators.py` | 二部图生成 |
| 图类定义 | `networkx/classes/graph.py` | 通用图类 |
| 后端调度 | `networkx/utils/backends.py` | `_dispatchable` 装饰器 |

### B. 参考代码示例

```python
import networkx as nx
from networkx.algorithms import bipartite

# 方式1：使用生成器创建二部图
G = bipartite.complete_bipartite_graph(3, 2)
# 节点自动带有 bipartite 属性
top = {n for n, d in G.nodes(data=True) if d["bipartite"] == 0}
bottom = set(G) - top

# 方式2：手动创建二部图
B = nx.Graph()
B.add_nodes_from([1, 2, 3, 4], bipartite=0)
B.add_nodes_from(["a", "b", "c"], bipartite=1)
B.add_edges_from([(1, "a"), (1, "b"), (2, "b"), (2, "c"), (3, "c"), (4, "a")])

# 方式3：通过二着色获取划分（仅连通图）
if nx.is_connected(B):
    X, Y = bipartite.sets(B)

# 使用二部图算法
matching = bipartite.maximum_matching(B, top_nodes={1,2,3,4})
projection = bipartite.projected_graph(B, {1,2,3,4})
centrality = bipartite.degree_centrality(B, nodes={1,2,3,4})
```
