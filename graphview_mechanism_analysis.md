# NetworkX 过滤视图（GraphView / subgraph_view）机制设计分析

## 一、设计概述

NetworkX 的过滤视图机制是一种**零拷贝**的图数据访问模式，允许在不复制底层数据的情况下，通过过滤条件动态限制节点和边的可见集合。这种设计借鉴了 Python 字典视图（dict views）的理念，提供了对底层数据的实时、只读投影。

### 核心设计目标

1. **零拷贝（Zero-copy）**：视图与底层图共享数据存储，不复制节点和边数据
2. **延迟过滤（Lazy filtering）**：过滤条件在访问时应用，而非视图创建时
3. **实时反映（Live view）**：底层图的修改会即时反映在视图中
4. **只读保护（Read-only）**：防止通过视图意外修改底层数据
5. **类型兼容**：支持 Graph/DiGraph/MultiGraph/MultiDiGraph 所有图类型

---

## 二、核心架构组件

### 2.1 模块层次结构

```
networkx/
└── classes/
    ├── graphviews.py      # 视图创建入口函数
    ├── coreviews.py       # 核心数据结构视图类
    ├── filters.py         # 过滤函数/类工厂
    └── function.py        # freeze 冻结机制
```

### 2.2 关键组件关系图

```
┌─────────────────────────────────────────────────────────────┐
│                      subgraph_view()                         │
│                    (graphviews.py:135)                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌─────────┐   ┌─────────┐   ┌─────────┐
        │_NODE_OK │   │_EDGE_OK │   │ _graph  │
        │(过滤器)  │   │(过滤器)  │   │(底层图) │
        └────┬────┘   └────┬────┘   └────┬────┘
             │              │              │
             └──────────────┼──────────────┘
                            ▼
              ┌─────────────────────────┐
              │   FilterAtlas /         │
              │   FilterAdjacency /     │
              │   FilterMultiAdjacency  │
              │     (coreviews.py)      │
              └───────────┬─────────────┘
                          │
                          ▼
              ┌─────────────────────────┐
              │   底层图数据结构         │
              │   _node, _adj, _succ,   │
              │   _pred                  │
              └─────────────────────────┘
```

---

## 三、过滤条件的延迟应用机制

### 3.1 视图创建 vs 数据访问

**关键洞察**：`subgraph_view` 创建视图时**不执行任何过滤操作**，只记录过滤条件。实际过滤发生在**数据访问时刻**。

```python
# graphviews.py:206-233
def subgraph_view(G, *, filter_node=no_filter, filter_edge=no_filter):
    newG = nx.freeze(G.__class__())
    newG._NODE_OK = filter_node      # 仅保存过滤函数引用
    newG._EDGE_OK = filter_edge       # 仅保存过滤函数引用
    
    # 创建视图：将底层数据结构包装在 Filter* 类中
    newG._node = FilterAtlas(G._node, filter_node)
    if G.is_directed():
        newG._succ = Adj(G._succ, filter_node, filter_edge)
        newG._pred = Adj(G._pred, filter_node, reverse_edge)
    else:
        newG._adj = Adj(G._adj, filter_node, filter_edge)
    return newG
```

### 3.2 FilterAtlas：节点级延迟过滤

`FilterAtlas` 是节点字典的视图包装器，实现了 `collections.abc.Mapping` 协议。

```python
# coreviews.py:267-311
class FilterAtlas(Mapping):
    def __init__(self, d, NODE_OK):
        self._atlas = d           # 引用底层字典，不复制
        self.NODE_OK = NODE_OK    # 保存过滤条件
    
    def __len__(self):
        # 访问时才计数：遍历并应用过滤
        if hasattr(self.NODE_OK, "length"):
            return self.NODE_OK.length
        if hasattr(self.NODE_OK, "nodes"):
            return len(self.NODE_OK.nodes & self._atlas.keys())
        return sum(1 for n in self._atlas if self.NODE_OK(n))
    
    def __iter__(self):
        # 访问时才过滤：生成器模式
        try:
            node_ok_shorter = 2 * len(self.NODE_OK.nodes) < len(self._atlas)
        except AttributeError:
            node_ok_shorter = False
        if node_ok_shorter:
            # 优化：如果过滤集合更小，遍历过滤集合
            return (n for n in self.NODE_OK.nodes if n in self._atlas)
        # 否则遍历底层数据并过滤
        return (n for n in self._atlas if self.NODE_OK(n))
    
    def __getitem__(self, key):
        # 访问时检查过滤条件
        if key in self._atlas and self.NODE_OK(key):
            return self._atlas[key]
        raise KeyError(f"Key {key} not found")
```

**核心特性**：
- **零拷贝**：`self._atlas = d` 仅保存引用
- **延迟计算**：`__len__`、`__iter__`、`__getitem__` 都在调用时才应用过滤
- **智能优化**：`__iter__` 会根据过滤集合大小选择更高效的遍历策略

### 3.3 FilterAdjacency：边级延迟过滤

`FilterAdjacency` 处理邻接表结构（`{node: {neighbor: attr_dict}}`），实现节点和边的双重过滤。

```python
# coreviews.py:314-365
class FilterAdjacency(Mapping):
    def __init__(self, d, NODE_OK, EDGE_OK):
        self._atlas = d
        self.NODE_OK = NODE_OK
        self.EDGE_OK = EDGE_OK
    
    def __getitem__(self, node):
        # 第一层：检查源节点是否通过过滤
        if node in self._atlas and self.NODE_OK(node):
            # 动态创建新的过滤条件：目标节点 + 边过滤
            def new_node_ok(nbr):
                return self.NODE_OK(nbr) and self.EDGE_OK(node, nbr)
            
            # 返回嵌套的 FilterAtlas，实现链式过滤
            return FilterAtlas(self._atlas[node], new_node_ok)
        raise KeyError(f"Key {node} not found")
```

**链式过滤机制**：
```
访问 view.adj[1][2] 的执行流程：
1. view.adj → FilterAdjacency 实例
2. [1] → 检查 NODE_OK(1)，通过则返回 FilterAtlas(邻居字典, new_node_ok)
3. [2] → FilterAtlas 检查 new_node_ok(2)，即 NODE_OK(2) AND EDGE_OK(1, 2)
```

### 3.4 多重图的特殊处理

对于 `MultiGraph`/`MultiDiGraph`，需要额外处理边键（edge key）：

```python
# coreviews.py:413-435
class FilterMultiAdjacency(FilterAdjacency):
    def __getitem__(self, node):
        if node in self._atlas and self.NODE_OK(node):
            # 边过滤需要包含边键参数
            def edge_ok(nbr, key):
                return self.NODE_OK(nbr) and self.EDGE_OK(node, nbr, key)
            
            return FilterMultiInner(self._atlas[node], self.NODE_OK, edge_ok)
        raise KeyError(f"Key {node} not found")

# coreviews.py:368-410
class FilterMultiInner(FilterAdjacency):
    def __getitem__(self, nbr):
        if (nbr in self._atlas 
            and self.NODE_OK(nbr) 
            and any(self.EDGE_OK(nbr, key) for key in self._atlas[nbr])):
            
            def new_node_ok(key):
                return self.EDGE_OK(nbr, key)
            
            return FilterAtlas(self._atlas[nbr], new_node_ok)
        raise KeyError(f"Key {nbr} not found")
```

**多重图访问路径**：
```
view.adj[1][2][0]  (访问节点1→2的第0条边)
│       │    │
│       │    └── FilterAtlas: 检查边键过滤
│       └── FilterMultiInner: 检查目标节点 + 边键
└── FilterMultiAdjacency: 检查源节点
```

---

## 四、共享存储与一致性保证

### 4.1 数据结构共享机制

`subgraph_view` 创建的新图对象与底层图**共享所有核心数据结构**：

```python
# graphviews.py:210-232
newG._graph = G           # 保存对底层图的引用
newG.graph = G.graph      # 图级属性字典：直接引用

# 节点数据：通过 FilterAtlas 包装
newG._node = FilterAtlas(G._node, filter_node)

# 邻接数据：根据图类型选择不同的 Filter 类
if G.is_directed():
    newG._succ = Adj(G._succ, filter_node, filter_edge)
    newG._pred = Adj(G._pred, filter_node, reverse_edge)
else:
    newG._adj = Adj(G._adj, filter_node, filter_edge)
```

**共享引用关系**：
```
┌──────────────────────┐
│    View (newG)       │
├──────────────────────┤
│ _graph ──────────────┼──────► G (底层图)
│ _node = FilterAtlas──┼──┐
│ _adj = FilterAdjacency┼─┤
└──────────────────────┘ │
                          │
                          ▼
                 ┌────────────────┐
                 │ G._node (dict) │
                 │ G._adj (dict)  │
                 │ G._succ/_pred  │
                 └────────────────┘
```

### 4.2 实时一致性验证

视图是**实时（live）**的，底层图的修改会立即反映在视图中：

```python
# 来自 graphviews.py 文档示例
G = nx.Graph()
G.add_edge(1, 2, weight=0.3)
G.add_edge(2, 3, weight=0.5)

# 创建通用视图
viewG = nx.graphviews.generic_graph_view(G)

# 修改底层图
G.remove_edge(2, 3)

# 视图自动反映变化
print(G.edges(data=True))
# EdgeDataView([(1, 2, {'weight': 0.3})])

print(viewG.edges(data=True))
# EdgeDataView([(1, 2, {'weight': 0.3})])  # 同步更新
```

### 4.3 只读保护机制

视图通过 `freeze()` 函数设置为只读，防止意外修改：

```python
# function.py:198-245
def freeze(G):
    # 将所有修改方法替换为抛出异常的 dummy 方法
    G.add_node = frozen
    G.add_nodes_from = frozen
    G.remove_node = frozen
    G.remove_nodes_from = frozen
    G.add_edge = frozen
    G.add_edges_from = frozen
    G.remove_edge = frozen
    G.remove_edges_from = frozen
    G.clear = frozen
    G.clear_edges = frozen
    G.frozen = True
    return G

def frozen(*args, **kwargs):
    raise nx.NetworkXError("Frozen graph can't be modified")
```

**注意**：`freeze` 只阻止**结构修改**（增删节点/边），但**允许修改属性数据**：

```python
view = nx.subgraph_view(G, filter_node=...)
view.frozen  # True

# 结构修改会失败
view.add_node(99)  # NetworkXError: Frozen graph can't be modified

# 属性修改可能成功（取决于底层数据是否可写）
# 这是设计允许的，因为属性是共享引用
```

---

## 五、过滤函数设计

### 5.1 过滤函数协议

NetworkX 定义了灵活的过滤函数协议：

| 过滤类型 | 签名 | 返回值含义 |
|---------|------|-----------|
| `filter_node` | `func(node) -> bool` | `True` 表示节点可见 |
| `filter_edge` (普通图) | `func(u, v) -> bool` | `True` 表示边可见 |
| `filter_edge` (多重图) | `func(u, v, key) -> bool` | `True` 表示边可见 |

### 5.2 内置过滤工具

`filters.py` 提供了常用的过滤函数工厂：

```python
# filters.py:21-95

# 默认不过滤
def no_filter(*items):
    return True

# 隐藏指定节点
def hide_nodes(nodes):
    nodes = set(nodes)
    return lambda node: node not in nodes

# 显示指定节点（类实现，支持 pickle）
class show_nodes:
    def __init__(self, nodes):
        self.nodes = set(nodes)  # 附加 nodes 属性供优化使用
    
    def __call__(self, node):
        return node in self.nodes

# 边过滤
def hide_edges(edges):
    alledges = set(edges) | {(v, u) for (u, v) in edges}
    return lambda u, v: (u, v) not in alledges
```

### 5.3 优化：带属性的过滤类

注意 `show_nodes` 被实现为**类**而非 lambda，这是为了：

1. **支持 pickle 序列化**：lambda 无法被 pickle
2. **提供性能优化属性**：`FilterAtlas.__iter__` 和 `__len__` 会检查 `nodes` 属性

```python
# coreviews.py:293-300 (FilterAtlas.__iter__)
def __iter__(self):
    try:  # 检查 NODE_OK 是否有 'nodes' 属性
        node_ok_shorter = 2 * len(self.NODE_OK.nodes) < len(self._atlas)
    except AttributeError:
        node_ok_shorter = False
    
    if node_ok_shorter:
        # 优化路径：遍历更小的过滤集合
        return (n for n in self.NODE_OK.nodes if n in self._atlas)
    # 普通路径：遍历底层数据
    return (n for n in self._atlas if self.NODE_OK(n))
```

---

## 六、其他视图类型

### 6.1 generic_graph_view：通用类型转换视图

用于在不复制数据的情况下转换图类型（如 Graph ↔ DiGraph）：

```python
# graphviews.py:41-132
def generic_graph_view(G, create_using=None):
    if create_using is None:
        newG = G.__class__()
    else:
        newG = nx.empty_graph(0, create_using)
    
    newG = nx.freeze(newG)
    newG._graph = G
    newG.graph = G.graph
    newG._node = G._node  # 直接共享
    
    if newG.is_directed():
        if G.is_directed():
            newG._succ = G._succ
            newG._pred = G._pred
        else:
            # 无向图转有向图：邻接表同时作为 succ 和 pred
            newG._succ = G._adj
            newG._pred = G._adj
    elif G.is_directed():
        # 有向图转无向图：使用 Union* 合并 succ 和 pred
        if G.is_multigraph():
            newG._adj = UnionMultiAdjacency(G._succ, G._pred)
        else:
            newG._adj = UnionAdjacency(G._succ, G._pred)
    else:
        newG._adj = G._adj
    return newG
```

### 6.2 reverse_view：反向视图

用于有向图的边方向反转：

```python
# graphviews.py:236-268
def reverse_view(G):
    newG = generic_graph_view(G)
    # 简单交换 succ 和 pred 的引用，不复制数据
    newG._succ, newG._pred = G._pred, G._succ
    return newG
```

### 6.3 Union* 系列：合并视图

用于将有向图的 `succ` 和 `pred` 合并为无向视图：

```python
# coreviews.py:110-162
class UnionAtlas(Mapping):
    def __init__(self, succ, pred):
        self._succ = succ
        self._pred = pred
    
    def __len__(self):
        return len(self._succ.keys() | self._pred.keys())
    
    def __iter__(self):
        return iter(set(self._succ.keys()) | set(self._pred.keys()))
    
    def __getitem__(self, key):
        try:
            return self._succ[key]
        except KeyError:
            return self._pred[key]
```

---

## 七、视图链（View Chaining）与性能注意事项

### 7.1 视图链的形成

可以在视图上再创建视图，形成视图链：

```python
G = nx.path_graph(10)
view1 = nx.subgraph_view(G, filter_node=lambda n: n < 8)
view2 = nx.subgraph_view(view1, filter_edge=lambda u, v: u != 3)
# view2._graph → view1 → G
```

### 7.2 性能问题

文档中明确警告：

> "Note: Since graphviews look like graphs, one can end up with view-of-view-of-view chains. Be careful with chains because they become very slow with about 15 nested views."

**原因**：每次访问都需要逐层应用过滤条件：

```
访问 view2.adj[5] 的执行路径：
1. view2._adj[5] → FilterAdjacency 检查 NODE_OK2(5)
2. 底层是 view1._adj → FilterAdjacency 检查 NODE_OK1(5)
3. 底层是 G._adj → 实际字典访问

每层都有函数调用开销，O(n) 层 = O(n) 倍开销
```

### 7.3 部分优化：子图快捷方式

对于简单的节点诱导子图，NetworkX 尝试优化视图链：

```python
# graphviews.py 文档注释
# "For the common simple case of node induced subgraphs created
# from the graph class, we short-cut the chain by returning a
# subgraph of the original graph directly rather than a subgraph
# of a subgraph."
```

但这种优化是有限的，文档建议：

> "Often it is easiest to use .copy() to avoid chains."

---

## 八、一致性保证与边界情况

### 8.1 底层图修改的影响

由于视图是实时的，底层图的修改可能导致意外行为：

```python
G = nx.Graph()
G.add_edges_from([(0, 1), (1, 2), (2, 3)])

# 过滤显示偶数节点
view = nx.subgraph_view(G, filter_node=lambda n: n % 2 == 0)
print(list(view.nodes()))  # [0, 2]

# 迭代过程中修改底层图
nodes_iter = iter(view.nodes())
print(next(nodes_iter))  # 0
G.add_node(4)
print(next(nodes_iter))  # 2 或 4? 取决于迭代器实现
```

**建议**：在迭代视图时避免修改底层图。

### 8.2 边过滤的完整性约束

`subgraph_view` 允许过滤掉连接两个可见节点的边：

```python
G = nx.path_graph(4)  # 0-1-2-3

# 节点都可见，但边 (1,2) 不可见
view = nx.subgraph_view(
    G,
    filter_node=lambda n: True,
    filter_edge=lambda u, v: (u, v) not in {(1, 2), (2, 1)}
)

print(list(view.edges()))  # [(0, 1), (2, 3)]
# 图看起来是不连通的两个分量
```

这是**设计允许的**，但用户需要意识到这可能导致与"节点诱导子图"不同的语义。

### 8.3 属性的共享性

视图与底层图**共享属性字典**：

```python
G = nx.Graph()
G.add_edge(0, 1, weight=1.0)

view = nx.subgraph_view(G)
print(view[0][1]['weight'])  # 1.0

# 通过视图修改属性（如果底层数据结构允许）
G[0][1]['weight'] = 2.0
print(view[0][1]['weight'])  # 2.0，同步更新
```

---

## 九、总结

### 9.1 核心设计模式

| 设计模式 | 实现方式 | 关键代码位置 |
|---------|---------|-------------|
| **虚拟代理（Virtual Proxy）** | Filter* 类延迟应用过滤 | `coreviews.py:267-435` |
| **引用共享** | 直接赋值底层数据结构引用 | `graphviews.py:211-232` |
| **方法替换** | `freeze()` 替换修改方法 | `function.py:198-245` |
| **生成器模式** | `__iter__` 返回生成器实现延迟过滤 | `coreviews.py:293-300` |
| **策略模式** | 过滤函数作为参数注入 | `graphviews.py:135` |

### 9.2 关键设计决策权衡

| 决策 | 优点 | 代价 |
|-----|------|-----|
| 零拷贝共享 | 内存高效、实时同步 | 底层修改可能影响视图 |
| 延迟过滤 | 创建视图 O(1)、灵活 | 访问时 O(k) 过滤开销 |
| 只读保护（结构级） | 防止意外修改 | 属性仍可修改 |
| 视图链支持 | 组合灵活 | 深度链性能差 |

### 9.3 与复制方式的对比

| 特性 | subgraph_view | G.subgraph().copy() |
|-----|---------------|---------------------|
| 内存占用 | O(1)（仅视图对象） | O(n + m) |
| 创建时间 | O(1) | O(n + m) |
| 访问时间 | O(k) 过滤开销 | O(1) 直接访问 |
| 实时反映底层修改 | 是 | 否 |
| 可修改性 | 只读（结构） | 完全可写 |
| 适用场景 | 临时分析、算法中间态 | 需要独立副本 |

---

## 十、代码索引

| 功能 | 文件位置 | 关键行号 |
|-----|---------|---------|
| `subgraph_view` 函数 | `networkx/classes/graphviews.py` | 135-233 |
| `generic_graph_view` 函数 | `networkx/classes/graphviews.py` | 41-132 |
| `reverse_view` 函数 | `networkx/classes/graphviews.py` | 236-268 |
| `FilterAtlas` 类 | `networkx/classes/coreviews.py` | 267-311 |
| `FilterAdjacency` 类 | `networkx/classes/coreviews.py` | 314-365 |
| `FilterMultiAdjacency` 类 | `networkx/classes/coreviews.py` | 413-435 |
| `show_nodes` 过滤类 | `networkx/classes/filters.py` | 57-71 |
| `freeze` 函数 | `networkx/classes/function.py` | 198-245 |
