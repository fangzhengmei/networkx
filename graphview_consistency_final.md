# NetworkX 过滤视图一致性机制：校准版分析报告

> **版本说明**：本报告基于代码精确审计，校准了之前分析中的结论表述，并补充了有向图和多重图场景的完整证据链。

---

## 一、视图创建的精确数据流

### 1.1 `subgraph_view` 核心逻辑

**文件位置**：`networkx/classes/graphviews.py:135-233`

```python
def subgraph_view(G, *, filter_node=no_filter, filter_edge=no_filter):
    newG = nx.freeze(G.__class__())
    newG._NODE_OK = filter_node      # 属性 1：节点过滤条件
    newG._EDGE_OK = filter_edge      # 属性 2：边过滤条件
    
    # 引用共享
    newG._graph = G                  # 属性 3：底层图引用
    newG.graph = G.graph             # 图级属性直接引用
    
    # 数据结构包装
    newG._node = FilterAtlas(G._node, filter_node)
    
    # 根据图类型选择包装器
    if G.is_multigraph():
        Adj = FilterMultiAdjacency
        def reverse_edge(u, v, k=None):
            return filter_edge(v, u, k)
    else:
        Adj = FilterAdjacency
        def reverse_edge(u, v, k=None):
            return filter_edge(v, u)
    
    # 有向图的特殊处理：_succ 和 _pred 使用不同的边过滤
    if G.is_directed():
        newG._succ = Adj(G._succ, filter_node, filter_edge)      # 出边：原始顺序
        newG._pred = Adj(G._pred, filter_node, reverse_edge)      # 入边：参数反转
    else:
        newG._adj = Adj(G._adj, filter_node, filter_edge)
    
    return newG
```

### 1.2 关键设计决策

| 决策点 | 代码位置 | 设计意图 |
|-------|---------|---------|
| `_NODE_OK` / `_EDGE_OK` 属性 | `graphviews.py:207-208` | 供 `subgraph()` 方法识别视图并优化 |
| `_graph` 属性 | `graphviews.py:211` | 指向原始图，用于解包视图链 |
| `_succ` / `_pred` 分别包装 | `graphviews.py:227-229` | 有向图边过滤的方向一致性 |
| `reverse_edge` 闭包 | `graphviews.py:218-225` | 入边视图保持与出边相同的过滤语义 |

---

## 二、过滤类的精确行为校准

### 2.1 `FilterAtlas` - 节点级过滤

**文件位置**：`networkx/classes/coreviews.py:267-311`

```python
class FilterAtlas(Mapping):
    def __init__(self, d, NODE_OK):
        self._atlas = d           # 底层字典引用（不复制）
        self.NODE_OK = NODE_OK    # 过滤条件
    
    def __len__(self):
        # 优化：如果过滤条件有 length/nodes 属性
        if hasattr(self.NODE_OK, "length"):
            return self.NODE_OK.length
        if hasattr(self.NODE_OK, "nodes"):
            return len(self.NODE_OK.nodes & self._atlas.keys())
        # 否则遍历计数
        return sum(1 for n in self._atlas if self.NODE_OK(n))
    
    def __iter__(self):
        try:  # 优化路径：过滤集合更小时
            node_ok_shorter = 2 * len(self.NODE_OK.nodes) < len(self._atlas)
        except AttributeError:
            node_ok_shorter = False
        
        if node_ok_shorter:
            return (n for n in self.NODE_OK.nodes if n in self._atlas)
        return (n for n in self._atlas if self.NODE_OK(n))
    
    def __getitem__(self, key):
        if key in self._atlas and self.NODE_OK(key):
            return self._atlas[key]  # 返回底层字典的值（属性字典）
        raise KeyError(f"Key {key} not found")
```

**关键行为**：
- `__getitem__` 返回**底层数据的直接引用**（节点属性字典）
- 这意味着通过视图修改属性会影响底层图

### 2.2 `FilterAdjacency` - 普通图邻接过滤

**文件位置**：`networkx/classes/coreviews.py:314-365`

```python
class FilterAdjacency(Mapping):
    def __init__(self, d, NODE_OK, EDGE_OK):
        self._atlas = d
        self.NODE_OK = NODE_OK
        self.EDGE_OK = EDGE_OK
    
    def __getitem__(self, node):
        if node in self._atlas and self.NODE_OK(node):
            # 动态创建嵌套过滤条件
            def new_node_ok(nbr):
                return self.NODE_OK(nbr) and self.EDGE_OK(node, nbr)
            
            return FilterAtlas(self._atlas[node], new_node_ok)
        raise KeyError(f"Key {node} not found")
```

**调用链示例**：访问 `view.adj[1][2]`

```
步骤 1: view.adj[1]
        → FilterAdjacency.__getitem__(1)
        检查: 1 in G._adj AND NODE_OK(1)
        通过 → 创建 new_node_ok(nbr) = NODE_OK(nbr) AND EDGE_OK(1, nbr)
              返回 FilterAtlas(G._adj[1], new_node_ok)

步骤 2: [2] (对返回的 FilterAtlas)
        → FilterAtlas.__getitem__(2)
        检查: 2 in G._adj[1] AND new_node_ok(2)
              = 2 in G._adj[1] AND NODE_OK(2) AND EDGE_OK(1, 2)
        通过 → 返回 G._adj[1][2] (边属性字典)
```

### 2.3 `FilterMultiAdjacency` - 多重图邻接过滤

**文件位置**：`networkx/classes/coreviews.py:413-435`

```python
class FilterMultiAdjacency(FilterAdjacency):
    def __getitem__(self, node):
        if node in self._atlas and self.NODE_OK(node):
            # 边过滤需要包含边键
            def edge_ok(nbr, key):
                return self.NODE_OK(nbr) and self.EDGE_OK(node, nbr, key)
            
            return FilterMultiInner(self._atlas[node], self.NODE_OK, edge_ok)
        raise KeyError(f"Key {node} not found")
```

### 2.4 `FilterMultiInner` - 多重图内层过滤

**文件位置**：`networkx/classes/coreviews.py:368-410`

```python
class FilterMultiInner(FilterAdjacency):
    def __getitem__(self, nbr):
        if (
            nbr in self._atlas
            and self.NODE_OK(nbr)
            and any(self.EDGE_OK(nbr, key) for key in self._atlas[nbr])
        ):
            # 此时 self.EDGE_OK 是 edge_ok(nbr, key) 的部分应用
            def new_node_ok(key):
                return self.EDGE_OK(nbr, key)
            
            return FilterAtlas(self._atlas[nbr], new_node_ok)
        raise KeyError(f"Key {nbr} not found")
```

**多重图调用链**：访问 `view.adj[1][2][0]`（节点 1→2 的第 0 条边）

```
步骤 1: view.adj[1]
        → FilterMultiAdjacency.__getitem__(1)
        检查: 1 in G._adj AND NODE_OK(1)
        通过 → 创建 edge_ok(nbr, key) = NODE_OK(nbr) AND EDGE_OK(1, nbr, key)
              返回 FilterMultiInner(G._adj[1], NODE_OK, edge_ok)

步骤 2: [2] (对 FilterMultiInner)
        → FilterMultiInner.__getitem__(2)
        检查: 2 in G._adj[1]
              AND NODE_OK(2)
              AND any(EDGE_OK(2, key) for key in G._adj[1][2])
              = any(edge_ok(2, key) for key in ...)
              = any(NODE_OK(2) AND EDGE_OK(1, 2, key) for key in ...)
        通过 → 创建 new_node_ok(key) = EDGE_OK(2, key) = edge_ok(2, key)
              返回 FilterAtlas(G._adj[1][2], new_node_ok)

步骤 3: [0] (对 FilterAtlas)
        → FilterAtlas.__getitem__(0)
        检查: 0 in G._adj[1][2] AND new_node_ok(0)
              = 0 in G._adj[1][2] AND edge_ok(2, 0)
              = 0 in G._adj[1][2] AND NODE_OK(2) AND EDGE_OK(1, 2, 0)
        通过 → 返回 G._adj[1][2][0] (边属性字典)
```

**关键校准**：
- 多重图中 `filter_edge` 接收三个参数：`(u, v, key)`
- `FilterMultiAdjacency.EDGE_OK` = 原始 `filter_edge`
- `FilterMultiInner.EDGE_OK` = `edge_ok(nbr, key)` 闭包（已捕获 `node`）
- 最终检查：`EDGE_OK(node, nbr, key)`

---

## 三、有向图场景的过滤一致性

### 3.1 `_succ` 与 `_pred` 的不同处理

**关键代码**：`graphviews.py:227-229`

```python
if G.is_directed():
    newG._succ = Adj(G._succ, filter_node, filter_edge)      # 出边：原始顺序
    newG._pred = Adj(G._pred, filter_node, reverse_edge)      # 入边：参数反转
```

**`reverse_edge` 的定义**：`graphviews.py:224-225`

```python
# 普通有向图
def reverse_edge(u, v, k=None):
    return filter_edge(v, u)

# 多重有向图 (graphviews.py:218-219)
def reverse_edge(u, v, k=None):
    return filter_edge(v, u, k)
```

### 3.2 一致性验证

考虑有向边 `u→v`：

| 访问方式 | 使用的数据结构 | 调用的 `filter_edge` | 实际参数 |
|---------|---------------|---------------------|---------|
| `view.succ[u]` | `_succ` | 原始 `filter_edge` | `filter_edge(u, v)` |
| `view.pred[v]` | `_pred` | `reverse_edge` | `reverse_edge(v, u)` → `filter_edge(u, v)` |
| `view.out_edges` | `_succ` | 原始 `filter_edge` | `filter_edge(u, v)` |
| `view.in_edges` | `_pred` | `reverse_edge` | `reverse_edge(v, u)` → `filter_edge(u, v)` |

**关键发现**：
- 对于边 `u→v`，无论从出边方向还是入边方向访问，**最终都调用 `filter_edge(u, v)`**
- 这要求 `filter_edge` 对有向图使用**不对称**的签名：`filter_edge(source, target)`

### 3.3 内置过滤函数的设计

**文件位置**：`networkx/classes/filters.py`

```python
# 有向图边过滤：不对称检查
def show_diedges(edges):
    edges = {(u, v) for u, v in edges}
    return lambda u, v: (u, v) in edges  # 严格检查顺序

def hide_diedges(edges):
    edges = {(u, v) for u, v in edges}
    return lambda u, v: (u, v) not in edges

# 无向图边过滤：对称检查（同时检查 (u,v) 和 (v,u)）
def show_edges(edges):
    alledges = set(edges) | {(v, u) for (u, v) in edges}
    return lambda u, v: (u, v) in alledges

def hide_edges(edges):
    alledges = set(edges) | {(v, u) for (u, v) in edges}
    return lambda u, v: (u, v) not in alledges
```

**测试验证**：`test_subgraphviews.py:44-56`

```python
def test_hidden_edges(self):
    hide_edges = [(2, 3), (8, 7), (222, 223)]
    edges_gone = self.hide_edges_filter(hide_edges)  # 对 DiGraph 是 hide_diedges
    G = self.gview(self.G, filter_edge=edges_gone)
    
    if G.is_directed():
        assert self.G.edges - G.edges == {(2, 3)}  # 只有 (2,3) 边被隐藏
        assert list(G[2]) == []                       # 2 的出边为空
        assert list(G.pred[3]) == []                  # 3 的入边为空
        assert list(G.pred[2]) == [1]                 # 2 的入边仍然有 1
    else:
        assert self.G.edges - G.edges == {(2, 3), (7, 8)}  # 无向图两个方向都隐藏
```

---

## 四、ReportView 层的过滤传递

### 4.1 `NodeView`

**文件位置**：`networkx/classes/reportviews.py:226-`

```python
class NodeView(Mapping, Set):
    def __init__(self, graph):
        self._nodes = graph._node  # 直接引用视图的 _node
        # 如果 graph 是视图，graph._node 已经是 FilterAtlas
    
    def __len__(self):
        return len(self._nodes)  # 调用 FilterAtlas.__len__
    
    def __iter__(self):
        return iter(self._nodes)  # 调用 FilterAtlas.__iter__
    
    def __getitem__(self, n):
        return self._nodes[n]  # 调用 FilterAtlas.__getitem__
```

**过滤传递**：`view.nodes` 完全依赖 `view._node`，而 `view._node` 已经是 `FilterAtlas`。

### 4.2 `OutEdgeView` / `InEdgeView`

**文件位置**：`networkx/classes/reportviews.py:1149-`

```python
class OutEdgeView(Set, Mapping, EdgeViewABC):
    def __init__(self, G):
        self._graph = G
        self._adjdict = G._succ if hasattr(G, "succ") else G._adj
        # 对有向图视图：G._succ 是 FilterAdjacency
        # 对无向图视图：G._adj 是 FilterAdjacency

class InEdgeView(OutEdgeView):
    def __init__(self, G):
        self._graph = G
        self._adjdict = G._pred if hasattr(G, "pred") else G._adj
        # 对有向图视图：G._pred 是 FilterAdjacency（使用 reverse_edge）
```

**关键一致性**：
- `OutEdgeView` 使用 `_succ` → 边过滤按原始顺序 `filter_edge(u, v)`
- `InEdgeView` 使用 `_pred` → 边过滤按反转顺序 `reverse_edge(u, v)` = `filter_edge(v, u)`
- 对于边 `u→v`，两种方式最终都检查 `filter_edge(u, v)`

### 4.3 `DegreeView`

**文件位置**：`networkx/classes/reportviews.py:586-`

```python
class DiDegreeView:
    def __init__(self, G, nbunch=None, weight=None):
        self._graph = G
        self._succ = G._succ if hasattr(G, "_succ") else G._adj
        self._pred = G._pred if hasattr(G, "_pred") else G._adj
        # 对视图：_succ 和 _pred 都是 FilterAdjacency
    
    def __getitem__(self, n):
        succs = self._succ[n]  # FilterAdjacency[n] → FilterAtlas
        preds = self._pred[n]  # 同上
        if weight is None:
            return len(succs) + len(preds)  # len(FilterAtlas) 应用过滤
        # 加权度：遍历过滤后的邻居
        deg = sum(
            succs[nbr].get(weight, 1) for nbr in succs
        ) + sum(
            preds[nbr].get(weight, 1) for nbr in preds
        )
        return deg
```

---

## 五、标准操作的过滤生效路径汇总

### 5.1 完整调用链表

| 操作 | 代码位置 | 数据来源 | 过滤生效位置 | 状态 |
|-----|---------|---------|-------------|------|
| `len(G)` | `graph.py:506` | `self._node` | `FilterAtlas.__len__` | ✅ 受保护 |
| `for n in G` | `graph.py:470` | `self._node` | `FilterAtlas.__iter__` | ✅ 受保护 |
| `n in G` | `graph.py:481` | `self._node` | `FilterAtlas.__getitem__` | ✅ 受保护 |
| `G.nodes` | `graph.py:755` | `graph._node` | `FilterAtlas` 包装 | ✅ 受保护 |
| `G.nodes[n]` | `reportviews.py:298` | `self._nodes[n]` | `FilterAtlas.__getitem__` | ✅ 受保护 |
| `G.adj` | `graph.py:392` | `self._adj` | `FilterAdjacency` 包装 | ✅ 受保护 |
| `G.succ` (DiGraph) | `digraph.py:~380` | `self._succ` | `FilterAdjacency` 包装 | ✅ 受保护 |
| `G.pred` (DiGraph) | `digraph.py:~400` | `self._pred` | `FilterAdjacency(reverse_edge)` | ✅ 受保护 |
| `G[n]` | `graph.py:508` | `self.adj[n]` | 链式过滤 | ✅ 受保护 |
| `G[n][m]` | `coreviews.py:351` | 嵌套 `FilterAtlas` | 节点+边双重过滤 | ✅ 受保护 |
| `G[n][m][k]` (MultiGraph) | `coreviews.py:409` | 多层 `FilterAtlas` | 节点+边+边键过滤 | ✅ 受保护 |
| `G.neighbors(n)` | `graph.py:1343` | `self._adj[n]` | `FilterAdjacency` | ✅ 受保护 |
| `G.adjacency()` | `graph.py:1489` | `self._adj.items()` | `FilterAdjacency` | ✅ 受保护 |
| `G.edges` | `graph.py:1386` | `G._succ`/`G._adj` | `FilterAdjacency` | ✅ 受保护 |
| `G.out_edges` (DiGraph) | `digraph.py:1033` | `G._succ` | `FilterAdjacency` | ✅ 受保护 |
| `G.in_edges` (DiGraph) | `digraph.py:1039` | `G._pred` | `FilterAdjacency(reverse_edge)` | ✅ 受保护 |
| `G.edges[u, v]` | `reportviews.py:1190` | `self._adjdict[u][v]` | 链式过滤 | ✅ 受保护 |
| `G.degree` | `graph.py:1509` | `G._succ`/`G._pred` | `FilterAdjacency` | ✅ 受保护 |
| `G.degree[n]` | `reportviews.py:550` | `len(succs) + len(preds)` | `FilterAtlas.__len__` | ✅ 受保护 |

### 5.2 一致性保障机制

#### 缓存失效机制

**文件位置**：`networkx/classes/graph.py:22-67`

```python
class _CachedPropertyResetterAdj:
    def __set__(self, obj, value):
        od = obj.__dict__
        od["_adj"] = value
        # 重置相关缓存
        props = ["adj", "edges", "degree"]
        for prop in props:
            if prop in od:
                del od[prop]

class _CachedPropertyResetterNode:
    def __set__(self, obj, value):
        od = obj.__dict__
        od["_node"] = value
        if "nodes" in od:
            del od["nodes"]

# 类定义
class Graph:
    _adj = _CachedPropertyResetterAdj()
    _node = _CachedPropertyResetterNode()
```

**DiGraph 的扩展**：`networkx/classes/digraph.py:21-85`

```python
class _CachedPropertyResetterAdjAndSucc:
    def __set__(self, obj, value):
        od = obj.__dict__
        od["_adj"] = value
        od["_succ"] = value
        # 重置更多缓存
        props = ["adj", "succ", "edges", "out_edges", "degree", "out_degree", "in_degree"]
        for prop in props:
            if prop in od:
                del od[prop]

class _CachedPropertyResetterPred:
    def __set__(self, obj, value):
        od = obj.__dict__
        od["_pred"] = value
        props = ["pred", "in_edges", "degree", "out_degree", "in_degree"]
        for prop in props:
            if prop in od:
                del od[prop]
```

**视图创建时的缓存状态**：
```python
def subgraph_view(G, *, filter_node=no_filter, filter_edge=no_filter):
    newG = nx.freeze(G.__class__())  # 新创建的对象，无缓存
    
    # 这些赋值会触发缓存清除（但新对象本来就没有）
    newG._node = FilterAtlas(G._node, filter_node)
    if G.is_directed():
        newG._succ = Adj(G._succ, filter_node, filter_edge)
        newG._pred = Adj(G._pred, filter_node, reverse_edge)
    else:
        newG._adj = Adj(G._adj, filter_node, filter_edge)
    
    return newG
```

---

## 六、透传路径与语义边界校准

### 6.1 `view._graph` - 设计上的底层图访问

**文件位置**：`graphviews.py:112, 211`

```python
newG._graph = G  # 直接引用底层图
```

**行为**：
- `view._graph` 指向原始图，完全绕过过滤
- 这是**设计上的内部属性**，供 `subgraph()` 等方法使用
- 普通用户代码应避免直接访问

**风险等级**：**高**（完全绕过过滤）

### 6.2 `view.subgraph(nodes)` - 设计上的视图链优化

**文件位置**：`networkx/classes/graph.py:1868-1875`

```python
def subgraph(self, nodes):
    induced_nodes = nx.filters.show_nodes(self.nbunch_iter(nodes))
    subgraph = nx.subgraph_view
    if hasattr(self, "_NODE_OK"):
        # 关键：在底层图上创建新视图，不是在当前视图上
        return subgraph(
            self._graph, filter_node=induced_nodes, filter_edge=self._EDGE_OK
        )
    return subgraph(self, filter_node=induced_nodes)
```

**行为校准**（之前的"冲突"实际是设计意图）：

| 条件 | 行为 | 语义 |
|-----|------|-----|
| 普通图 | `subgraph(self, filter_node=...)` | 在当前对象上创建视图 |
| 已有视图（有 `_NODE_OK`） | `subgraph(self._graph, filter_node=..., filter_edge=self._EDGE_OK)` | **解包到底层图** |

**设计意图**（来自 `graphviews.py:16-24` 文档）：
> "For the common simple case of node induced subgraphs created from the graph class, we short-cut the chain by returning a subgraph of the original graph directly rather than a subgraph of a subgraph. We are careful not to disrupt any edge filter in the middle subgraph."

**关键语义**：
1. ✅ 避免视图链（性能优化）
2. ✅ 边过滤条件 `_EDGE_OK` 被**保留**
3. ⚠️ 节点过滤条件被**替换**为 `show_nodes(nodes)`
4. ⚠️ 新视图的 `_graph` 指向**原始底层图**，不是当前视图

**这不是"冲突"，而是明确的设计权衡**。

**风险等级**：**中**（语义变化但有文档说明）

### 6.3 `copy(as_view=True)` / `to_directed(as_view=True)` - 通用视图

**文件位置**：
- `graph.py:1591-1678` (`copy`)
- `digraph.py:1680-1734` (`to_directed`)

```python
def copy(self, as_view=False):
    if as_view is True:
        return nx.graphviews.generic_graph_view(self)
    # ... 深拷贝逻辑
```

**`generic_graph_view` 的行为**：`graphviews.py:41-132`

```python
def generic_graph_view(G, create_using=None):
    # ...
    newG._node = G._node           # 直接引用，不重新包装
    if newG.is_directed():
        if G.is_directed():
            newG._succ = G._succ   # 直接引用
            newG._pred = G._pred   # 直接引用
        # ...
    # 不设置 _NODE_OK 和 _EDGE_OK！
    return newG
```

**行为校准**：

| 方面 | 行为 | 状态 |
|-----|------|------|
| 数据结构过滤 | `G._node`/`G._succ`/`G._pred` 已经是 `Filter*` 包装 → 直接引用 → 过滤保留 | ✅ 过滤生效 |
| `_NODE_OK`/`_EDGE_OK` | `generic_graph_view` 不设置这些属性 | ❌ 属性丢失 |
| `subgraph()` 行为 | 没有 `_NODE_OK` → 不会解包到底层图 | ⚠️ 行为改变 |

**这不是"冲突"，而是两种不同的视图类型**：
- `subgraph_view`：有过滤条件的视图（设置 `_NODE_OK`/`_EDGE_OK`）
- `generic_graph_view`：通用类型转换视图（不设置过滤属性）

**风险等级**：**低**（过滤保留，仅属性丢失）

### 6.4 `reverse_view` - 反向视图

**文件位置**：`graphviews.py:236-268`

```python
def reverse_view(G):
    newG = generic_graph_view(G)  # 使用通用视图
    newG._succ, newG._pred = G._pred, G._succ  # 交换
    return newG
```

**行为校准**：
- 与 `generic_graph_view` 相同：数据过滤保留，`_NODE_OK`/`_EDGE_OK` 丢失
- 额外：`_succ` 和 `_pred` 交换

**风险等级**：**低**

### 6.5 透传路径汇总（校准版）

| 操作 | 数据过滤 | `_NODE_OK`/`_EDGE_OK` | 行为说明 | 风险等级 |
|-----|---------|----------------------|---------|---------|
| `view._graph` | ❌ 完全绕过 | - | 直接访问底层图 | **高** |
| `view.subgraph(nodes)` | ✅ 新过滤应用 | ⚠️ 节点过滤替换 | 设计上的视图链优化 | **中** |
| `view.copy(as_view=True)` | ✅ 保留包装 | ❌ 丢失 | 通用视图类型 | **低** |
| `view.to_directed(as_view=True)` | ✅ 保留包装 | ❌ 丢失 | 通用视图类型 | **低** |
| `reverse_view(view)` | ✅ 保留包装 | ❌ 丢失 | 通用视图类型 | **低** |

---

## 七、一致性边界与最佳实践

### 7.1 保证一致的场景

**所有标准图操作都受过滤保护**：

```python
# 节点操作
len(view)           # FilterAtlas.__len__
for n in view:      # FilterAtlas.__iter__
n in view           # FilterAtlas.__contains__
view.nodes          # NodeView 使用 view._node
view.nodes[n]       # FilterAtlas.__getitem__

# 邻接操作
view.adj            # AdjacencyView 使用 view._adj
view[n]             # FilterAdjacency.__getitem__
view[n][m]          # 嵌套 FilterAtlas
view.neighbors(n)   # iter(self._adj[n])

# 边操作
view.edges          # OutEdgeView 使用 view._succ/view._adj
view.in_edges       # InEdgeView 使用 view._pred
view.edges[u, v]    # 链式过滤访问

# 度计算
view.degree         # DiDegreeView 使用 view._succ/view._pred
view.degree[n]      # len(FilterAtlas)
```

### 7.2 需要注意的场景

#### 场景 1：`_graph` 内部属性

```python
# ❌ 错误：绕过过滤
for n in view._graph.nodes:
    process(n)  # 处理了被过滤的节点！

# ✅ 正确：使用标准接口
for n in view.nodes:
    process(n)  # 只处理可见节点
```

#### 场景 2：`subgraph()` 在视图上的语义

```python
G = nx.path_graph(10)

# 复杂节点过滤
def complex_filter(n):
    return n < 5 and n != 2  # 节点 0,1,3,4 可见

view1 = nx.subgraph_view(G, filter_node=complex_filter)
print(list(view1.nodes))  # [0, 1, 3, 4]

# subgraph() 会替换节点过滤条件
view2 = view1.subgraph([0, 1, 2, 3])
print(list(view2.nodes))  # [0, 1, 2, 3] - 节点 2 现在可见！

# ✅ 正确做法：手动合并过滤
def combined(n):
    return complex_filter(n) and n in {0, 1, 2, 3}

view2_correct = nx.subgraph_view(G, filter_node=combined)
print(list(view2_correct.nodes))  # [0, 1, 3] - 正确
```

#### 场景 3：属性共享

```python
# 通过视图修改属性会影响底层图
view = nx.subgraph_view(G, filter_node=fn)
view.nodes[0]['status'] = 'processed'
# G.nodes[0]['status'] 也变成 'processed'

# ✅ 如果需要独立属性，先复制
copied = view.copy()
copied.nodes[0]['status'] = 'processed'
# G.nodes[0]['status'] 不变
```

### 7.3 过滤函数的一致性要求

#### 有向图

```python
# ✅ 正确：不对称的有向边过滤
def my_diedge_filter(u, v):
    # u 是源，v 是目标
    return (u, v) in allowed_edges  # 严格顺序

# 或使用内置工厂
filter_edge = nx.filters.show_diedges([(0, 1), (2, 3)])
```

#### 无向图

```python
# ✅ 正确：对称的无向边过滤
def my_edge_filter(u, v):
    return (u, v) in allowed_edges or (v, u) in allowed_edges

# 或使用内置工厂（自动处理对称）
filter_edge = nx.filters.show_edges([(0, 1), (2, 3)])
# 实际上检查 {(0,1), (1,0), (2,3), (3,2)}
```

#### 多重图

```python
# ✅ 正确：包含边键
def my_multiedge_filter(u, v, k):
    # k 是边键
    return k == 0  # 只显示键为 0 的边

# 或使用内置工厂
filter_edge = nx.filters.show_multiedges([(0, 1, 0), (0, 1, 2)])
```

---

## 八、代码索引（校准版）

### 8.1 核心文件

| 文件 | 描述 |
|-----|------|
| `networkx/classes/graphviews.py` | 视图创建函数（`subgraph_view`, `generic_graph_view`, `reverse_view`） |
| `networkx/classes/coreviews.py` | 数据结构视图类（`FilterAtlas`, `FilterAdjacency`, `FilterMultiInner`, `FilterMultiAdjacency`） |
| `networkx/classes/filters.py` | 过滤函数工厂（对称/不对称、普通/多重） |
| `networkx/classes/reportviews.py` | 报告视图类（`NodeView`, `OutEdgeView`, `InEdgeView`, `DegreeView`） |
| `networkx/classes/graph.py` | Graph 基类（操作入口、缓存机制、`subgraph()`） |
| `networkx/classes/digraph.py` | DiGraph 类（有向图扩展、`_succ`/`_pred` 缓存） |

### 8.2 精确代码位置

| 功能 | 文件 | 行号 |
|-----|-----|-----|
| `subgraph_view` 主函数 | `graphviews.py` | 135-233 |
| `_NODE_OK`/`_EDGE_OK` 设置 | `graphviews.py` | 207-208 |
| `_graph` 引用设置 | `graphviews.py` | 112, 211 |
| 有向图 `_succ`/`_pred` 分别包装 | `graphviews.py` | 227-229 |
| `reverse_edge` 闭包定义 | `graphviews.py` | 218-225 |
| `generic_graph_view` 函数 | `graphviews.py` | 41-132 |
| `reverse_view` 函数 | `graphviews.py` | 236-268 |
| `FilterAtlas` 类 | `coreviews.py` | 267-311 |
| `FilterAdjacency` 类 | `coreviews.py` | 314-365 |
| `FilterMultiInner` 类 | `coreviews.py` | 368-410 |
| `FilterMultiAdjacency` 类 | `coreviews.py` | 413-435 |
| `show_diedges`/`hide_diedges` | `filters.py` | 74-77, 32-35 |
| `show_edges`/`hide_edges` | `filters.py` | 80-83, 38-41 |
| `show_multiedges`/`hide_multiedges` | `filters.py` | 92-95, 50-53 |
| `Graph.subgraph` 方法 | `graph.py` | 1868-1875 |
| `_CachedPropertyResetterAdj` | `graph.py` | 22-44 |
| `_CachedPropertyResetterAdjAndSucc` | `digraph.py` | 21-57 |
| `OutEdgeView.__init__` (`_succ`) | `reportviews.py` | 1170 |
| `InEdgeView.__init__` (`_pred`) | `reportviews.py` | 1407 |

---

## 九、结论校准

### 9.1 之前分析的校准点

| 之前的表述 | 校准后的准确表述 |
|-----------|-----------------|
| "`subgraph()` 语义变化" | "`subgraph()` 设计上的视图链优化，节点过滤被替换，边过滤被保留" |
| "冲突结论" | "不同视图类型的设计差异：`subgraph_view` 带过滤属性，`generic_graph_view` 不带" |
| "`copy(as_view=True)` 过滤保留但属性丢失" | "`copy(as_view=True)` 创建通用视图，数据过滤保留（因为 `Filter*` 对象被直接引用），但 `_NODE_OK`/`_EDGE_OK` 属性不设置" |

### 9.2 最终一致性模型

```
┌─────────────────────────────────────────────────────────────────────┐
│                         过滤视图一致性模型                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    受过滤保护的操作                            │  │
│  │  len(G), for n in G, n in G                                 │  │
│  │  G.nodes, G.nodes[n]                                         │  │
│  │  G.adj, G[n], G[n][m], G.neighbors(n)                       │  │
│  │  G.edges, G.in_edges, G.out_edges, G.edges[u, v]           │  │
│  │  G.degree, G.degree[n]                                       │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    语义边界（设计意图）                        │  │
│  │                                                                 │  │
│  │  view._graph          → 内部属性，直接访问底层图               │  │
│  │  view.subgraph(nodes) → 优化：解包到底层图，替换节点过滤       │  │
│  │  view.copy(as_view)   → 通用视图：过滤保留，属性不设置         │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    有向图一致性保证                            │  │
│  │                                                                 │  │
│  │  边 u→v 的访问：                                               │  │
│  │    - view.succ[u][v]  → filter_edge(u, v)                   │  │
│  │    - view.pred[v][u]  → reverse_edge(v, u) = filter_edge(u, v) │  │
│  │    - view.out_edges    → 使用 _succ → filter_edge(u, v)    │  │
│  │    - view.in_edges     → 使用 _pred → filter_edge(u, v)    │  │
│  │                                                                 │  │
│  │  要求：filter_edge 对有向图使用不对称签名 filter_edge(src, dst) │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    多重图一致性保证                            │  │
│  │                                                                 │  │
│  │  边 (u, v, k) 的访问：                                         │  │
│  │    - view.adj[u][v][k] → filter_edge(u, v, k)              │  │
│  │                                                                 │  │
│  │  调用链：                                                       │  │
│  │    FilterMultiAdjacency[u] → 创建 edge_ok(nbr, key)         │  │
│  │    FilterMultiInner[v]    → 检查 edge_ok(v, key)            │  │
│  │    FilterAtlas[k]         → 返回边属性                         │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.3 关键设计原则

1. **零拷贝共享**：所有 `Filter*` 类都引用底层数据结构，不复制
2. **延迟过滤**：过滤条件在访问时刻通过 `__len__`/`__iter__`/`__getitem__` 应用
3. **方向一致性**：有向图的 `_pred` 使用 `reverse_edge` 闭包，确保边 `u→v` 从两个方向访问都检查 `filter_edge(u, v)`
4. **视图链优化**：`subgraph()` 在视图上调用时解包到底层图，避免性能问题（但改变节点过滤语义）
5. **属性共享**：`FilterAtlas.__getitem__` 返回底层数据的直接引用，属性修改影响底层图
