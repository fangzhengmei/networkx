# NetworkX 过滤视图跨模块一致性分析

## 一、整体架构概览

### 1.1 视图创建时的数据包装

`subgraph_view` 创建视图时，通过**三层包装**实现过滤与数据共享：

```
┌─────────────────────────────────────────────────────────────────┐
│                    subgraph_view(G, filter_node, filter_edge)   │
└─────────────────────────────┬───────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │  _NODE_OK       │ │  _EDGE_OK       │ │  _graph          │
    │  (过滤条件)      │ │  (过滤条件)      │ │  (底层图引用)    │
    └────────┬────────┘ └────────┬────────┘ └────────┬────────┘
             │                    │                    │
             └────────────────────┼────────────────────┘
                                  ▼
                    ┌─────────────────────────┐
                    │   数据结构包装层         │
                    └───────────┬─────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
   ┌──────────────┐    ┌──────────────────┐  ┌──────────────────┐
   │  _node =     │    │  _adj =          │  │  _succ/_pred =   │
   │FilterAtlas   │    │FilterAdjacency   │  │FilterAdjacency   │
   │(G._node, fn) │    │(G._adj, fn, fe)  │  │(G._succ/_pred,   │
   └──────────────┘    └──────────────────┘  │   fn, fe/rev_fe)  │
                                              └──────────────────┘
```

### 1.2 关键代码位置

| 组件 | 文件位置 | 行号 |
|-----|---------|-----|
| `subgraph_view` 主函数 | `networkx/classes/graphviews.py` | 135-233 |
| `FilterAtlas` 类 | `networkx/classes/coreviews.py` | 267-311 |
| `FilterAdjacency` 类 | `networkx/classes/coreviews.py` | 314-365 |
| `FilterMultiAdjacency` 类 | `networkx/classes/coreviews.py` | 413-435 |

---

## 二、常见图操作入口的数据流分析

### 2.1 操作入口分类

根据实现方式，图操作可分为三类：

| 分类 | 操作示例 | 实现方式 | 过滤状态 |
|-----|---------|---------|---------|
| **直接访问 `_node`** | `len(G)`, `for n in G`, `n in G` | 直接使用 `self._node` | ✅ 受过滤保护 |
| **通过 `_adj`/`_succ`/`_pred`** | `G[n]`, `G.adj`, `G.neighbors(n)` | 直接使用 `self._adj` 等 | ✅ 受过滤保护 |
| **通过 ReportView** | `G.nodes`, `G.edges`, `G.degree` | 创建 View 对象，内部使用 `_node`/`_adj` | ✅ 受过滤保护 |
| **特殊方法** | `G.subgraph()`, `G.copy(as_view=True)` | 可能绕过或重置过滤 | ⚠️ 需特别注意 |

### 2.2 详细数据流追踪

#### 2.2.1 节点访问路径

**路径 1：`len(G)` → 节点计数**

```python
# graph.py:506
def __len__(self):
    return len(self._node)

# 视图中 _node 是 FilterAtlas
# FilterAtlas.__len__:
# coreviews.py:284-291
def __len__(self):
    if hasattr(self.NODE_OK, "length"):
        return self.NODE_OK.length
    if hasattr(self.NODE_OK, "nodes"):
        return len(self.NODE_OK.nodes & self._atlas.keys())
    return sum(1 for n in self._atlas if self.NODE_OK(n))
```

**过滤生效点**：`FilterAtlas.__len__` 会应用 `NODE_OK` 过滤条件。

---

**路径 2：`for n in G` → 节点迭代**

```python
# graph.py:470
def __iter__(self):
    return iter(self._node)

# FilterAtlas.__iter__:
# coreviews.py:293-300
def __iter__(self):
    try:
        node_ok_shorter = 2 * len(self.NODE_OK.nodes) < len(self._atlas)
    except AttributeError:
        node_ok_shorter = False
    if node_ok_shorter:
        return (n for n in self.NODE_OK.nodes if n in self._atlas)
    return (n for n in self._atlas if self.NODE_OK(n))
```

**过滤生效点**：生成器表达式中应用 `NODE_OK` 过滤。

---

**路径 3：`n in G` → 成员检查**

```python
# graph.py:481-484
def __contains__(self, n):
    try:
        return n in self._node
    except TypeError:
        return False

# FilterAtlas.__contains__ (继承自 Mapping)
# 会调用 __getitem__ 检查
```

**过滤生效点**：`FilterAtlas.__getitem__` 会检查 `NODE_OK(key)`。

---

#### 2.2.2 `G.nodes` - NodeView 路径

```python
# graph.py:755-846
@cached_property
def nodes(self):
    return NodeView(self)

# reportviews.py:288-304
class NodeView(Mapping, Set):
    def __init__(self, graph):
        self._nodes = graph._node  # 引用视图的 _node (FilterAtlas)
    
    def __len__(self):
        return len(self._nodes)  # 调用 FilterAtlas.__len__
    
    def __iter__(self):
        return iter(self._nodes)  # 调用 FilterAtlas.__iter__
    
    def __getitem__(self, n):
        return self._nodes[n]  # 调用 FilterAtlas.__getitem__
```

**过滤生效点**：`NodeView` 直接使用 `graph._node`，而视图的 `_node` 已经是 `FilterAtlas`。

---

#### 2.2.3 邻接访问路径

**路径 1：`G.adj` → AdjacencyView**

```python
# graph.py:392-409
@cached_property
def adj(self):
    return AdjacencyView(self._adj)  # self._adj 是 FilterAdjacency

# coreviews.py:66-86
class AdjacencyView(AtlasView):
    def __getitem__(self, name):
        return AtlasView(self._atlas[name])  # self._atlas 是 FilterAdjacency
```

**注意**：`adj` 是 `cached_property`，但视图通过 `_CachedPropertyResetterAdj` 描述符确保 `_adj` 被重新赋值时清除缓存。

```python
# graph.py:22-44
class _CachedPropertyResetterAdj:
    def __set__(self, obj, value):
        od = obj.__dict__
        od["_adj"] = value
        # 重置缓存属性
        props = ["adj", "edges", "degree"]
        for prop in props:
            if prop in od:
                del od[prop]
```

---

**路径 2：`G[n]` → 下标访问**

```python
# graph.py:508-532
def __getitem__(self, n):
    return self.adj[n]  # 通过 adj 属性访问
```

**路径 3：`G.neighbors(n)` → 邻居迭代**

```python
# graph.py:1343-1384
def neighbors(self, n):
    try:
        return iter(self._adj[n])  # 直接访问 _adj (FilterAdjacency)
    except KeyError as err:
        raise nx.NetworkXError(...) from err
```

**路径 4：`G.adjacency()` → 邻接迭代**

```python
# graph.py:1489-1507
def adjacency(self):
    return iter(self._adj.items())  # 直接访问 _adj (FilterAdjacency)
```

---

#### 2.2.4 FilterAdjacency 的链式过滤机制

`FilterAdjacency` 在访问时会动态创建嵌套的过滤条件：

```python
# coreviews.py:351-358
def __getitem__(self, node):
    if node in self._atlas and self.NODE_OK(node):
        # 动态创建新的过滤条件：邻居节点 + 边过滤
        def new_node_ok(nbr):
            return self.NODE_OK(nbr) and self.EDGE_OK(node, nbr)
        
        # 返回嵌套的 FilterAtlas
        return FilterAtlas(self._atlas[node], new_node_ok)
    raise KeyError(f"Key {node} not found")
```

**完整访问链示例**：

```
视图 view = subgraph_view(G, filter_node=fn, filter_edge=fe)

访问 view.adj[1][2] 的执行流程：

1. view.adj → AdjacencyView(view._adj)
              → AdjacencyView(FilterAdjacency(G._adj, fn, fe))

2. view.adj[1] → FilterAdjacency.__getitem__(1)
   检查：1 in G._adj AND fn(1)
   通过则创建：new_node_ok(nbr) = fn(nbr) AND fe(1, nbr)
   返回：FilterAtlas(G._adj[1], new_node_ok)

3. view.adj[1][2] → FilterAtlas.__getitem__(2)
   检查：2 in G._adj[1] AND new_node_ok(2)
      → 2 in G._adj[1] AND fn(2) AND fe(1, 2)
   通过则返回：G._adj[1][2] (边属性字典)
```

---

#### 2.2.5 边访问路径

**`G.edges` → EdgeView**

```python
# graph.py:1386-1441
@cached_property
def edges(self):
    return EdgeView(self)

# reportviews.py:1168-1202
class OutEdgeView:
    def __init__(self, G):
        self._graph = G
        self._adjdict = G._succ if hasattr(G, "succ") else G._adj
        # G._adj 是 FilterAdjacency
    
    def __iter__(self):
        for n, nbrs in self._nodes_nbrs():  # _adjdict.items
            for nbr in nbrs:
                yield (n, nbr)
        # 迭代 FilterAdjacency，自动应用过滤
    
    def __getitem__(self, e):
        u, v = e
        return self._adjdict[u][v]
        # FilterAdjacency[u][v] 应用双重过滤
```

---

#### 2.2.6 度计算路径

**`G.degree` → DegreeView**

```python
# graph.py:1509-1540+
@cached_property
def degree(self):
    return DegreeView(self)

# reportviews.py:529-574
class DiDegreeView:
    def __init__(self, G, nbunch=None, weight=None):
        self._graph = G
        self._succ = G._succ if hasattr(G, "_succ") else G._adj
        self._pred = G._pred if hasattr(G, "_pred") else G._adj
        # _succ/_pred 是 FilterAdjacency
    
    def __getitem__(self, n):
        succs = self._succ[n]  # FilterAdjacency[n]
        preds = self._pred[n]  # FilterAdjacency[n]
        if weight is None:
            return len(succs) + len(preds)  # len(FilterAtlas) 应用过滤
        # 加权度计算同样遍历过滤后的邻居
    
    def __iter__(self):
        for n in self._nodes:
            succs = self._succ[n]
            preds = self._pred[n]
            # 计算度时使用过滤后的邻居
            yield (n, len(succs) + len(preds))
```

---

### 2.3 操作路径汇总表

| 操作 | 代码位置 | 数据来源 | 过滤生效位置 | 状态 |
|-----|---------|---------|-------------|------|
| `len(G)` | `graph.py:506` | `self._node` | `FilterAtlas.__len__` | ✅ 受保护 |
| `for n in G` | `graph.py:470` | `self._node` | `FilterAtlas.__iter__` | ✅ 受保护 |
| `n in G` | `graph.py:481` | `self._node` | `FilterAtlas.__getitem__` | ✅ 受保护 |
| `G.nodes` | `graph.py:755` | `graph._node` | `FilterAtlas` 包装 | ✅ 受保护 |
| `G.nodes[n]` | `reportviews.py:298` | `self._nodes[n]` | `FilterAtlas.__getitem__` | ✅ 受保护 |
| `G.adj` | `graph.py:392` | `self._adj` | `FilterAdjacency` 包装 | ✅ 受保护 |
| `G[n]` | `graph.py:508` | `self.adj[n]` | 链式过滤 | ✅ 受保护 |
| `G[n][m]` | `coreviews.py:351` | 嵌套 `FilterAtlas` | 节点+边双重过滤 | ✅ 受保护 |
| `G.neighbors(n)` | `graph.py:1343` | `self._adj[n]` | `FilterAdjacency` | ✅ 受保护 |
| `G.adjacency()` | `graph.py:1489` | `self._adj.items()` | `FilterAdjacency` | ✅ 受保护 |
| `G.edges` | `graph.py:1386` | `G._adj`/`G._succ` | `FilterAdjacency` | ✅ 受保护 |
| `G.edges[u, v]` | `reportviews.py:1190` | `self._adjdict[u][v]` | 链式过滤 | ✅ 受保护 |
| `G.degree` | `graph.py:1509` | `G._succ`/`G._pred` | `FilterAdjacency` | ✅ 受保护 |
| `G.degree[n]` | `reportviews.py:550` | `len(succs) + len(preds)` | `FilterAtlas.__len__` | ✅ 受保护 |

---

## 三、透传路径识别：绕过过滤的风险点

### 3.1 风险点 1：`_graph` 属性 - 直接访问底层图

**代码位置**：`graphviews.py:112, 211`

```python
# generic_graph_view 和 subgraph_view 都设置：
newG._graph = G  # 直接引用底层图
```

**风险说明**：
- `view._graph` 直接指向原始图，完全绕过过滤
- 任何通过 `_graph` 的操作都不会应用过滤条件

**示例**：
```python
G = nx.path_graph(6)  # 0-1-2-3-4-5
view = nx.subgraph_view(G, filter_node=lambda n: n < 4)

# 视图中只能看到 0,1,2,3
print(list(view.nodes))  # [0, 1, 2, 3]

# 但 _graph 可以访问全部
print(list(view._graph.nodes))  # [0, 1, 2, 3, 4, 5]
print(list(view._graph.edges))  # [(0,1), (1,2), (2,3), (3,4), (4,5)]
```

**使用场景**：
- `_graph` 主要用于内部优化（如 `subgraph()` 方法）
- 普通用户代码应避免直接使用 `_graph`

---

### 3.2 风险点 2：`G.subgraph(nodes)` 优化路径

**代码位置**：`graph.py:1868-1875`

```python
def subgraph(self, nodes):
    induced_nodes = nx.filters.show_nodes(self.nbunch_iter(nodes))
    # if already a subgraph, don't make a chain
    subgraph = nx.subgraph_view
    if hasattr(self, "_NODE_OK"):
        # 关键：直接在底层图上创建新视图
        return subgraph(
            self._graph, filter_node=induced_nodes, filter_edge=self._EDGE_OK
        )
    return subgraph(self, filter_node=induced_nodes)
```

**行为分析**：

| 条件 | 行为 | 结果 |
|-----|------|-----|
| 非视图对象 | `subgraph(self, filter_node=...)` | 在当前对象上创建视图 |
| 已有 `_NODE_OK` 属性 | `subgraph(self._graph, ...)` | **解包到底层图** |

**关键语义变化**：

```python
G = nx.path_graph(10)  # 0-1-2-3-4-5-6-7-8-9

# 第一步：创建过滤掉奇数节点的视图
view1 = nx.subgraph_view(G, filter_node=lambda n: n % 2 == 0)
print(list(view1.nodes))  # [0, 2, 4, 6, 8]

# 第二步：在视图上调用 subgraph
view2 = view1.subgraph([0, 2, 4])

# 问题：view2 的 _NODE_OK 是什么？
# 答案：show_nodes([0, 2, 4])，完全替换了原来的 n % 2 == 0
# 但 view2 的 _EDGE_OK 保留了 view1._EDGE_OK（如果有的话）
```

**注意事项**：
1. `subgraph()` 在视图上调用时，会**解包**到底层原始图
2. 节点过滤条件会被**替换**为新的 `show_nodes(nodes)`
3. 边过滤条件 `_EDGE_OK` 会被**保留**
4. 这是为了避免"视图链"性能问题，但改变了语义

---

### 3.3 风险点 3：`copy(as_view=True)` - 无过滤的通用视图

**代码位置**：`graph.py:1591-1678`

```python
def copy(self, as_view=False):
    if as_view is True:
        return nx.graphviews.generic_graph_view(self)
    # ... 常规复制逻辑
```

**`generic_graph_view` 的行为**：

```python
# graphviews.py:41-132
def generic_graph_view(G, create_using=None):
    # ...
    newG._graph = G
    newG.graph = G.graph
    
    # 直接共享数据结构，不包装过滤！
    newG._node = G._node
    if newG.is_directed():
        if G.is_directed():
            newG._succ = G._succ
            newG._pred = G._pred
        else:
            newG._succ = G._adj
            newG._pred = G._adj
    elif G.is_directed():
        if G.is_multigraph():
            newG._adj = UnionMultiAdjacency(G._succ, G._pred)
        else:
            newG._adj = UnionAdjacency(G._succ, G._pred)
    else:
        newG._adj = G._adj
    return newG
```

**关键差异**：

| 方法 | 数据结构包装 | 过滤条件 |
|-----|-------------|---------|
| `subgraph_view(G, fn, fe)` | `FilterAtlas` / `FilterAdjacency` | 应用 `_NODE_OK` / `_EDGE_OK` |
| `generic_graph_view(G)` | 直接引用 `G._node`, `G._adj` | **无过滤** |

**风险示例**：

```python
G = nx.path_graph(6)
view = nx.subgraph_view(G, filter_node=lambda n: n < 4)
print(list(view.nodes))  # [0, 1, 2, 3]

# copy(as_view=True) 绕过过滤！
view_copy = view.copy(as_view=True)
print(list(view_copy.nodes))  # [0, 1, 2, 3, 4, 5] - 看到了全部节点！
```

**原因**：`view._node` 是 `FilterAtlas`，但 `generic_graph_view` 直接使用：
```python
newG._node = G._node  # 即 FilterAtlas
```

等等，这应该仍有过滤啊？让我再仔细看...

实际上：
- `view._node` 是 `FilterAtlas(G._node, filter_node)`
- `generic_graph_view(view)` 会设置 `newG._node = view._node`
- 所以 `newG._node` 仍然是 `FilterAtlas`

但问题在于：`generic_graph_view` 不会设置 `_NODE_OK` 和 `_EDGE_OK`，也不会重新包装。

更深层的问题：如果 `view` 是一个视图，`view._adj` 已经是 `FilterAdjacency`，那么 `generic_graph_view(view)` 会：
```python
newG._adj = view._adj  # 这是 FilterAdjacency
```

所以实际上 `copy(as_view=True)` 应该**保留过滤**...

让我重新分析：

```python
# 假设 G 是原始图，G._adj 是普通 dict
view = subgraph_view(G, filter_node=fn, filter_edge=fe)
# view._node = FilterAtlas(G._node, fn)
# view._adj = FilterAdjacency(G._adj, fn, fe)

view_copy = generic_graph_view(view)
# view_copy._node = view._node = FilterAtlas(G._node, fn)
# view_copy._adj = view._adj = FilterAdjacency(G._adj, fn, fe)
```

所以 `copy(as_view=True)` 实际上**保留了过滤**，因为数据结构已经被包装了。

但这里有一个细微差别：`generic_graph_view` 不会设置 `_NODE_OK` 和 `_EDGE_OK` 属性，这意味着：
1. `view_copy.subgraph()` 的行为可能不同（因为没有 `_NODE_OK` 检查）
2. 无法直接访问过滤条件

让我验证 `subgraph()` 的行为：

```python
# 如果 view_copy 没有 _NODE_OK：
def subgraph(self, nodes):
    if hasattr(self, "_NODE_OK"):  # False
        return subgraph(self._graph, ...)
    return subgraph(self, filter_node=induced_nodes)  # 走这个分支
```

所以 `view_copy.subgraph()` 会在 `view_copy` 上创建新视图，而不是解包到底层图。

**修正后的风险评估**：

`copy(as_view=True)`：
- ✅ 数据结构保持过滤包装（`FilterAtlas`/`FilterAdjacency`）
- ⚠️ 丢失 `_NODE_OK`/`_EDGE_OK` 属性
- ⚠️ `subgraph()` 行为改变（不解包到底层图）
- ⚠️ 可能形成更深的视图链

---

### 3.4 风险点 4：`to_directed(as_view=True)` / `to_undirected(as_view=True)`

**代码位置**：`graph.py:1680-1734, 1736+`

```python
def to_directed(self, as_view=False):
    graph_class = self.to_directed_class()
    if as_view is True:
        return nx.graphviews.generic_graph_view(self, graph_class)
    # ... 深拷贝逻辑
```

**同样使用 `generic_graph_view`**：
- 与 `copy(as_view=True)` 相同的行为
- 数据结构保持过滤包装
- 丢失 `_NODE_OK`/`_EDGE_OK`

---

### 3.5 风险点 5：`reverse_view` - 有条件保留过滤

**代码位置**：`graphviews.py:236-268`

```python
@not_implemented_for("undirected")
def reverse_view(G):
    newG = generic_graph_view(G)  # 先创建通用视图
    # 然后交换 succ 和 pred
    newG._succ, newG._pred = G._pred, G._succ
    return newG
```

**行为分析**：

| 输入类型 | `_node`/`_adj` | `_NODE_OK`/`_EDGE_OK` |
|---------|---------------|----------------------|
| 原始图 | 直接引用 | 无 |
| 已有视图 | `FilterAtlas`/`FilterAdjacency`（保留） | **丢失**（generic_graph_view 不设置） |

**关键问题**：`reverse_view` 使用 `generic_graph_view`，所以：
- ✅ 数据过滤包装保留（因为 `G._node` 已经是 `FilterAtlas`）
- ⚠️ `_NODE_OK`/`_EDGE_OK` 属性丢失
- ⚠️ `subgraph()` 行为改变

**示例**：

```python
G = nx.DiGraph([(0, 1), (1, 2), (2, 3)])
view = nx.subgraph_view(G, filter_node=lambda n: n < 3)

print(list(view.nodes))  # [0, 1, 2]
print(hasattr(view, '_NODE_OK'))  # True

rview = nx.reverse_view(view)
print(list(rview.nodes))  # [0, 1, 2] - 过滤保留
print(hasattr(rview, '_NODE_OK'))  # False - 属性丢失
```

---

### 3.6 透传路径汇总表

| 操作 | 代码位置 | 数据过滤 | `_NODE_OK`/`_EDGE_OK` | 风险等级 |
|-----|---------|---------|----------------------|---------|
| `view._graph` | `graphviews.py:211` | ❌ 完全绕过 | - | **高** |
| `view.subgraph(nodes)` | `graph.py:1871` | ⚠️ 条件重置 | ⚠️ 节点过滤替换 | **中** |
| `view.copy(as_view=True)` | `graph.py:1668` | ✅ 保留包装 | ❌ 丢失 | **低** |
| `view.to_directed(as_view=True)` | `graph.py:1723` | ✅ 保留包装 | ❌ 丢失 | **低** |
| `view.to_undirected(as_view=True)` | 类似 | ✅ 保留包装 | ❌ 丢失 | **低** |
| `reverse_view(view)` | `graphviews.py:265` | ✅ 保留包装 | ❌ 丢失 | **低** |

---

## 四、一致性保证机制

### 4.1 缓存失效机制

**问题**：`G.nodes`, `G.edges`, `G.degree`, `G.adj` 都是 `cached_property`，如果数据结构改变，缓存会失效吗？

**解决方案**：描述符 `_CachedPropertyResetterAdj` 和 `_CachedPropertyResetterNode`

```python
# graph.py:22-44
class _CachedPropertyResetterAdj:
    def __set__(self, obj, value):
        od = obj.__dict__
        od["_adj"] = value
        # 重置相关缓存
        props = ["adj", "edges", "degree"]
        for prop in props:
            if prop in od:
                del od[prop]

# graph.py:47-67
class _CachedPropertyResetterNode:
    def __set__(self, obj, value):
        od = obj.__dict__
        od["_node"] = value
        if "nodes" in od:
            del od["nodes"]

# 类定义中使用
class Graph:
    _adj = _CachedPropertyResetterAdj()
    _node = _CachedPropertyResetterNode()
```

**工作原理**：
1. `_adj` 和 `_node` 被定义为描述符
2. 当 `subgraph_view` 给 `newG._adj = FilterAdjacency(...)` 赋值时
3. 描述符的 `__set__` 方法被调用
4. 相关的 `cached_property` 缓存被清除

**视图创建时的缓存状态**：

```python
def subgraph_view(G, *, filter_node=no_filter, filter_edge=no_filter):
    newG = nx.freeze(G.__class__())  # 创建空图对象
    newG._NODE_OK = filter_node
    newG._EDGE_OK = filter_edge
    
    newG._graph = G
    newG.graph = G.graph
    
    # 这些赋值会触发缓存清除
    newG._node = FilterAtlas(G._node, filter_node)
    # 触发：清除 "nodes" 缓存
    
    newG._adj = Adj(G._adj, filter_node, filter_edge)  # 或 _succ/_pred
    # 触发：清除 "adj", "edges", "degree" 缓存
    
    return newG
```

**保证**：由于是新创建的 `G.__class__()` 对象，本来就没有缓存，所以这主要是防御性编程。

---

### 4.2 只读保护机制

**问题**：如何防止通过视图意外修改底层图？

**解决方案**：`freeze()` 函数

```python
# function.py:198-245
def freeze(G):
    # 将所有修改方法替换为抛出异常
    G.add_node = frozen
    G.add_nodes_from = frozen
    G.remove_node = frozen
    G.remove_nodes_from = frozen
    G.add_edge = frozen
    G.add_edges_from = frozen
    G.add_weighted_edges_from = frozen
    G.remove_edge = frozen
    G.remove_edges_from = frozen
    G.clear = frozen
    G.clear_edges = frozen
    G.frozen = True
    return G

def frozen(*args, **kwargs):
    raise nx.NetworkXError("Frozen graph can't be modified")
```

**在视图创建时应用**：

```python
# graphviews.py:206
newG = nx.freeze(G.__class__())
```

**保护范围**：

| 操作 | 普通图 | 视图（frozen） |
|-----|-------|---------------|
| `G.add_node(n)` | ✅ 成功 | ❌ `NetworkXError` |
| `G.add_edge(u, v)` | ✅ 成功 | ❌ `NetworkXError` |
| `G.remove_node(n)` | ✅ 成功 | ❌ `NetworkXError` |
| `G.nodes[n]['attr'] = value` | ✅ 成功 | ⚠️ **可能成功** |
| `G.edges[u, v]['attr'] = value` | ✅ 成功 | ⚠️ **可能成功** |

**关键注意**：`freeze()` 只保护**结构修改**（增删节点/边），但**不保护属性修改**。

属性修改仍然可以通过：
```python
view = nx.subgraph_view(G, ...)
view.nodes[0]['color'] = 'red'  # 可能成功！
view.edges[0, 1]['weight'] = 100  # 可能成功！
```

这是因为：
1. `view._node` 是 `FilterAtlas`，`view._node[0]` 返回 `G._node[0]`（底层节点属性字典）
2. `NodeView[n]` 返回 `self._nodes[n]`，即底层字典
3. 该字典是普通的 `dict`，可以修改

**设计意图**：属性共享是视图"实时"特性的一部分。如果需要完全独立的副本，应该使用 `.copy()`。

---

### 4.3 实时性保证

**问题**：底层图修改后，视图能立即看到变化吗？

**答案**：是的，因为所有数据结构都是**引用共享**。

**引用关系图**：

```
┌─────────────────────────────────────────────────────────────┐
│                         视图 (view)                           │
├─────────────────────────────────────────────────────────────┤
│  _graph ───────────────────────────────────────────────────┼──┐
│  _node = FilterAtlas(...)                                   │  │
│     │                                                       │  │
│     └──► _atlas ───────────────────────────────────────────┼──┼──► G._node
│  _adj = FilterAdjacency(...)                               │  │
│     │                                                       │  │
│     └──► _atlas ───────────────────────────────────────────┼──┼──► G._adj
│  graph ────────────────────────────────────────────────────┼──┼──► G.graph
└─────────────────────────────────────────────────────────────┘  │
                                                                  │
                        ┌─────────────────────────────────────────┘
                        ▼
              ┌─────────────────────────┐
              │      底层图 (G)         │
              ├─────────────────────────┤
              │  _node = {node: attrs}  │
              │  _adj = {node: nbrs}   │
              │  graph = {...}          │
              └─────────────────────────┘
```

**验证示例**（来自测试）：

```python
# test_subgraphviews.py:299-306
def test_add_node(self):
    # 添加节点到底层图，视图会看到
    self.G.add_node(5)
    assert [0, 1, 3, 4] == sorted(self.H.nodes)  # H 是 edge_subgraph
    self.G.remove_node(5)

def test_remove_node(self):
    # 从底层图删除节点，视图也会看不到
    self.G.remove_node(0)
    assert [1, 3, 4] == sorted(self.H.nodes)
    self.G.add_node(0, name="node0")
    self.G.add_edge(0, 1, name="edge01")
```

**属性共享验证**：

```python
# test_subgraphviews.py:318-332
def test_node_attr_dict(self):
    # H 是 edge_subgraph 视图
    for v in self.H:
        assert self.G.nodes[v] == self.H.nodes[v]
    
    # 修改 G 的属性，H 也看到变化
    self.G.nodes[0]["name"] = "foo"
    assert self.G.nodes[0] == self.H.nodes[0]
    
    # 修改 H 的属性，G 也看到变化（同一字典）
    self.H.nodes[1]["name"] = "bar"
    assert self.G.nodes[1] == self.H.nodes[1]
```

---

### 4.4 迭代一致性

**问题**：迭代视图时，如果底层图被修改，会发生什么？

**答案**：与 Python 字典迭代相同的行为。

**关键代码**：

```python
# FilterAtlas.__iter__
def __iter__(self):
    # 方案 1：如果过滤集合更小，遍历过滤集合
    if node_ok_shorter:
        return (n for n in self.NODE_OK.nodes if n in self._atlas)
    # 方案 2：否则遍历底层字典
    return (n for n in self._atlas if self.NODE_OK(n))
```

**风险**：
- 如果使用方案 2（遍历 `self._atlas`），在迭代中修改 `G._node` 可能导致 `RuntimeError: dictionary changed size during iteration`
- 如果使用方案 1（遍历 `self.NODE_OK.nodes`），则取决于过滤集合的类型

**建议**：
- 迭代视图时避免修改底层图
- 如果需要修改，先将视图转换为列表：`list(view.nodes)`

---

## 五、跨模块操作的一致性验证

### 5.1 算法模块如何使用视图

**问题**：当把视图传递给 `nx.shortest_path`, `nx.bfs_tree` 等算法时，过滤条件生效吗？

**答案**：是的，因为算法通过标准图操作接口访问数据。

**示例分析**：以 BFS 为例

```python
# 假设 BFS 算法使用以下操作：
# 1. for node in G 或 G.nodes
# 2. G.neighbors(node) 或 G[node]
# 3. G.has_edge(u, v)
# 4. 检查节点是否存在：n in G

# 所有这些操作都通过过滤层：
# - for node in G → iter(self._node) → FilterAtlas.__iter__
# - G.neighbors(n) → iter(self._adj[n]) → FilterAdjacency
# - G.has_edge(u, v) → v in G._adj[u] → FilterAdjacency
# - n in G → n in self._node → FilterAtlas
```

**验证**：查看测试

```python
# test_subgraphviews.py:364-371
@pytest.mark.parametrize("multigraph", (nx.MultiGraph, nx.MultiDiGraph))
def test_multigraph_filtered_edges(self, multigraph):
    G = multigraph([("a", "b"), ("a", "c"), ("c", "b")])
    H = nx.edge_subgraph(G, [("a", "b", 0), ("c", "b", 0)])
    assert "c" not in H["a"]  # 邻居访问受过滤
    assert not H.has_edge("a", "c")  # has_edge 受过滤
```

---

### 5.2 `subgraph()` 方法的特殊一致性

**问题**：`G.subgraph(nodes)` 对视图和普通图行为一致吗？

**答案**：语义上有微妙差异，但设计是有意的。

**普通图上的行为**：
```python
G = nx.path_graph(10)
H = G.subgraph([0, 1, 2])
# H 是 subgraph_view(G, filter_node=show_nodes([0, 1, 2]))
# H._graph = G
# H._NODE_OK = show_nodes([0, 1, 2])
```

**视图上的行为**：
```python
G = nx.path_graph(10)
view1 = nx.subgraph_view(G, filter_node=lambda n: n < 5)  # 节点 0-4
view2 = view1.subgraph([0, 1, 2])

# 由于 view1 有 _NODE_OK：
# view2 = subgraph_view(view1._graph, 
#                      filter_node=show_nodes([0, 1, 2]),
#                      filter_edge=view1._EDGE_OK)
# 
# 即 view2 直接基于 G，不是 view1
```

**设计意图**（来自 `graphviews.py` 文档）：
> "For the common simple case of node induced subgraphs created from the graph class, we short-cut the chain by returning a subgraph of the original graph directly rather than a subgraph of a subgraph. We are careful not to disrupt any edge filter in the middle subgraph."

**关键点**：
1. 避免视图链（性能原因）
2. 保留边过滤条件 `_EDGE_OK`
3. 替换节点过滤条件为新的 `show_nodes`

**潜在语义变化**：
```python
G = nx.path_graph(10)

# 复杂的节点过滤条件
def complex_filter(n):
    # 某些复杂逻辑，无法用 show_nodes 表示
    return n < 5 and n != 2

view1 = nx.subgraph_view(G, filter_node=complex_filter)
print(list(view1.nodes))  # [0, 1, 3, 4]

view2 = view1.subgraph([0, 1, 2, 3])
# view2._NODE_OK = show_nodes([0, 1, 2, 3])
# complex_filter 被替换了！

print(list(view2.nodes))  # [0, 1, 2, 3] - 节点 2 现在可见了！
```

**这是设计上的权衡**：
- 性能优化优先（避免视图链）
- 假设 `subgraph()` 用于创建节点诱导子图
- 复杂过滤条件应该重新评估

---

### 5.3 图属性的一致性

**问题**：`G.graph`, `G.nodes[n]`, `G.edges[u, v]` 这些属性在视图和底层图之间如何共享？

**答案**：完全共享同一字典对象。

```python
# test_subgraphviews.py:350-355
def test_graph_attr_dict(self):
    assert self.G.graph is self.H.graph  # 同一对象

# graphviews.py 中的设置
newG.graph = G.graph  # 直接引用
```

**节点属性**：
```python
# NodeView.__getitem__
def __getitem__(self, n):
    return self._nodes[n]

# FilterAtlas.__getitem__
def __getitem__(self, key):
    if key in self._atlas and self.NODE_OK(key):
        return self._atlas[key]  # 返回 G._node[key]
```

**边属性**：
```python
# FilterAdjacency.__getitem__ 返回 FilterAtlas
# FilterAtlas.__getitem__ 返回 self._atlas[key]
# 即 G._adj[u][v]
```

**一致性验证**：

| 属性访问 | 视图代码路径 | 返回对象 | 与底层图关系 |
|---------|-------------|---------|-------------|
| `view.graph` | 直接引用 `G.graph` | 同 `G.graph` | **同一对象** |
| `view.nodes[n]` | `FilterAtlas[n]` → `G._node[n]` | 同 `G.nodes[n]` | **同一对象** |
| `view.edges[u, v]` | `FilterAdjacency[u][v]` → `G._adj[u][v]` | 同 `G.edges[u, v]` | **同一对象** |

---

## 六、一致性边界与最佳实践

### 6.1 一致性保证范围

**保证一致的操作**：

```
┌─────────────────────────────────────────────────────────────────┐
│                     受过滤保护的操作                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  节点访问：                                                       │
│  ✅ len(G)          ✅ for n in G      ✅ n in G               │
│  ✅ G.nodes         ✅ G.nodes[n]       ✅ list(G.nodes.data()) │
│                                                                 │
│  邻接访问：                                                       │
│  ✅ G.adj           ✅ G[n]             ✅ G[n][m]              │
│  ✅ G.neighbors(n)  ✅ G.adjacency()    ✅ G.has_edge(u, v)    │
│                                                                 │
│  边访问：                                                         │
│  ✅ G.edges         ✅ G.edges[u, v]    ✅ list(G.edges.data())│
│  ✅ G.in_edges      ✅ G.out_edges                             │
│                                                                 │
│  度计算：                                                         │
│  ✅ G.degree        ✅ G.degree[n]      ✅ G.in_degree         │
│  ✅ G.out_degree                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**不保证一致的操作**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    绕过过滤的风险操作                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  直接访问底层图：                                                 │
│  ❌ view._graph          → 完全绕过过滤                         │
│                                                                 │
│  丢失过滤属性的视图操作：                                          │
│  ⚠️ view.copy(as_view=True)                                      │
│  ⚠️ view.to_directed(as_view=True)                              │
│  ⚠️ view.to_undirected(as_view=True)                            │
│  ⚠️ nx.reverse_view(view)                                        │
│  （数据过滤保留，但 _NODE_OK/_EDGE_OK 丢失）                      │
│                                                                 │
│  语义变化的操作：                                                 │
│  ⚠️ view.subgraph(nodes)                                         │
│  （节点过滤被替换，边过滤保留）                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 6.2 最佳实践

#### 实践 1：避免直接使用 `_graph`

```python
# 错误：绕过过滤
for n in view._graph.nodes:
    process(n)  # 处理了被过滤的节点！

# 正确：使用标准接口
for n in view.nodes:
    process(n)  # 只处理可见节点
```

#### 实践 2：复杂过滤避免使用 `subgraph()` 链式调用

```python
# 问题：complex_filter 会被替换
view1 = subgraph_view(G, filter_node=complex_filter)
view2 = view1.subgraph([0, 1, 2])  # complex_filter 丢失！

# 正确：手动合并过滤条件
def combined_filter(n):
    return complex_filter(n) and n in {0, 1, 2}

view2 = subgraph_view(G, filter_node=combined_filter)
```

#### 实践 3：需要独立副本时使用 `.copy()`

```python
# 视图：实时、共享、只读（结构）
view = subgraph_view(G, filter_node=fn)

# 副本：独立、可修改
copied = view.copy()  # 或 nx.Graph(view)
```

#### 实践 4：迭代时避免修改底层图

```python
# 风险：可能导致 RuntimeError
for n in view.nodes:
    if condition(n):
        G.remove_node(n)  # 修改底层图

# 安全：先获取快照
nodes_to_remove = [n for n in view.nodes if condition(n)]
for n in nodes_to_remove:
    G.remove_node(n)
```

#### 实践 5：属性修改注意共享性

```python
# 通过视图修改属性会影响底层图
view = subgraph_view(G, filter_node=fn)
view.nodes[0]['status'] = 'processed'
# G.nodes[0]['status'] 也变成 'processed'

# 如果需要独立属性，先复制
copied = view.copy()
copied.nodes[0]['status'] = 'processed'
# G.nodes[0]['status'] 不变
```

---

## 七、代码索引

### 7.1 核心文件

| 文件 | 描述 |
|-----|------|
| `networkx/classes/graphviews.py` | 视图创建函数（`subgraph_view`, `generic_graph_view`, `reverse_view`） |
| `networkx/classes/coreviews.py` | 数据结构视图类（`FilterAtlas`, `FilterAdjacency` 等） |
| `networkx/classes/reportviews.py` | 报告视图类（`NodeView`, `EdgeView`, `DegreeView` 等） |
| `networkx/classes/graph.py` | Graph 基类（操作入口、缓存机制） |
| `networkx/classes/function.py` | `freeze()` 函数 |

### 7.2 关键代码位置

| 功能 | 文件 | 行号 |
|-----|-----|-----|
| `subgraph_view` 主函数 | `graphviews.py` | 135-233 |
| `_NODE_OK`/`_EDGE_OK` 设置 | `graphviews.py` | 207-208 |
| `_graph` 引用设置 | `graphviews.py` | 112, 211 |
| `FilterAtlas` 类 | `coreviews.py` | 267-311 |
| `FilterAdjacency` 类 | `coreviews.py` | 314-365 |
| `FilterAdjacency.__getitem__` 链式过滤 | `coreviews.py` | 351-358 |
| `NodeView` 类 | `reportviews.py` | 226- |
| `EdgeView` 类 | `reportviews.py` | 1150- |
| `DegreeView` 类 | `reportviews.py` | 586- |
| `Graph.__len__` | `graph.py` | 506 |
| `Graph.__iter__` | `graph.py` | 470 |
| `Graph.__getitem__` | `graph.py` | 508-532 |
| `Graph.adj` (cached_property) | `graph.py` | 392-409 |
| `Graph.nodes` (cached_property) | `graph.py` | 755-846 |
| `Graph.edges` (cached_property) | `graph.py` | 1386-1441 |
| `Graph.degree` (cached_property) | `graph.py` | 1509- |
| `Graph.neighbors` | `graph.py` | 1343-1384 |
| `Graph.adjacency` | `graph.py` | 1489-1507 |
| `Graph.subgraph` | `graph.py` | 1868-1875 |
| `Graph.copy` | `graph.py` | 1591-1678 |
| `Graph.to_directed` | `graph.py` | 1680-1734 |
| `_CachedPropertyResetterAdj` | `graph.py` | 22-44 |
| `_CachedPropertyResetterNode` | `graph.py` | 47-67 |
| `freeze` 函数 | `function.py` | 198-245 |
