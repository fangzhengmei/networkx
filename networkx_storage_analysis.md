# NetworkX 图对象内部存储设计分析报告

## 1. 概述

NetworkX 是一个用于创建、操作和研究复杂网络结构的 Python 库。其核心设计采用了**嵌套字典（nested dictionary）**的数据结构来表示图的邻接关系，并通过**惰性视图（lazy views）**机制提供高效的图结构访问能力。

本报告深入分析 NetworkX 中四种图类型的内部存储设计以及惰性视图的实现机制。

---

## 2. 四种图类型的存储结构

NetworkX 提供了四种基本图类型：

| 图类型 | 类名 | 方向性 | 多重边 |
|--------|------|--------|--------|
| 无向图 | `Graph` | 无向 | 不支持 |
| 有向图 | `DiGraph` | 有向 | 不支持 |
| 多重无向图 | `MultiGraph` | 无向 | 支持 |
| 多重有向图 | `MultiDiGraph` | 有向 | 支持 |

### 2.1 核心数据成员

所有图类型都包含以下核心数据成员：

```python
# 图级属性字典
self.graph = {}  # 存储图的全局属性，如名称等

# 节点属性字典
self._node = {}  # {node: {attr_key: attr_value, ...}, ...}

# 邻接关系字典（根据图类型有所不同）
self._adj = {}   # 或 self._succ / self._pred
```

### 2.2 Graph（无向图）的存储结构

**存储层级：dict-of-dict-of-dict（三层嵌套字典）**

```
self._node = {
    node1: {attr1: val1, attr2: val2, ...},
    node2: {...},
    ...
}

self._adj = {
    node1: {
        neighbor1: {edge_attr1: val1, edge_attr2: val2, ...},
        neighbor2: {...},
        ...
    },
    node2: {...},
    ...
}
```

**关键设计特点**：

1. **边属性字典共享引用**：在 `graph.py:979-982` 中，添加边时：
```python
datadict = self._adj[u].get(v, self.edge_attr_dict_factory())
datadict.update(attr)
self._adj[u][v] = datadict
self._adj[v][u] = datadict  # 同一字典对象的引用
```
这意味着 `G._adj[u][v]` 和 `G._adj[v][u]` 指向**同一个字典对象**，修改其中一个会同时影响另一个。

2. **自环处理**：自环边在 `_adj` 中只存储一次，但在计算度时需要特殊处理（`graph.py:634` 中的 `n in nbrs` 检查）。

**示例**：
```python
G = nx.Graph()
G.add_edge(1, 2, weight=3.0, color='red')

# 内部存储结构
G._node = {
    1: {},
    2: {}
}

G._adj = {
    1: {
        2: {'weight': 3.0, 'color': 'red'}
    },
    2: {
        1: {'weight': 3.0, 'color': 'red'}  # 与上面是同一个字典对象
    }
}

# 验证共享引用
assert G._adj[1][2] is G._adj[2][1]  # True
```

### 2.3 DiGraph（有向图）的存储结构

**存储层级：使用两个独立的三层嵌套字典**

```
self._node = {
    node1: {attr1: val1, ...},
    ...
}

self._succ = self._adj = {  # 后继节点（出边）
    node1: {
        neighbor1: {edge_attr1: val1, ...},
        ...
    },
    ...
}

self._pred = {  # 前驱节点（入边）
    node1: {
        neighbor1: {edge_attr1: val1, ...},
        ...
    },
    ...
}
```

**关键设计特点**：

1. **_adj 与 _succ 是同一对象**：在 `digraph.py:329-331` 中：
```python
_adj = _CachedPropertyResetterAdjAndSucc()
_succ = _adj  # 同一描述符
_pred = _CachedPropertyResetterPred()
```

2. **边属性字典双向共享**：在 `digraph.py:741-744` 中：
```python
datadict = self._adj[u].get(v, self.edge_attr_dict_factory())
datadict.update(attr)
self._succ[u][v] = datadict
self._pred[v][u] = datadict  # 同一字典对象的引用
```

3. **方向独立性**：边 `(u, v)` 存储在 `_succ[u][v]` 和 `_pred[v][u]`，而边 `(v, u)` 存储在 `_succ[v][u]` 和 `_pred[u][v]`，两者是完全独立的。

**示例**：
```python
G = nx.DiGraph()
G.add_edge(1, 2, weight=3.0)  # 1 -> 2
G.add_edge(2, 1, weight=5.0)  # 2 -> 1

# 内部存储
G._succ = {
    1: {2: {'weight': 3.0}},
    2: {1: {'weight': 5.0}}
}

G._pred = {
    1: {2: {'weight': 5.0}},  # 2 是 1 的前驱
    2: {1: {'weight': 3.0}}   # 1 是 2 的前驱
}

# 验证：1->2 的边属性在 _succ[1][2] 和 _pred[2][1] 共享
assert G._succ[1][2] is G._pred[2][1]  # True

# 验证：两条边是独立的
assert G._succ[1][2] is not G._succ[2][1]  # True
```

### 2.4 MultiGraph（多重无向图）的存储结构

**存储层级：dict-of-dict-of-dict-of-dict（四层嵌套字典）**

多重图需要支持同一对节点间存在多条边，因此增加了**边键（edge key）**层来区分不同的边。

```
self._node = {
    node1: {attr1: val1, ...},
    ...
}

self._adj = {
    node1: {
        neighbor1: {
            edge_key1: {edge_attr1: val1, ...},
            edge_key2: {...},
            ...
        },
        neighbor2: {...},
        ...
    },
    node2: {...},
    ...
}
```

**关键设计特点**：

1. **边键自动生成**：在 `multigraph.py:413-440` 的 `new_edge_key` 方法：
```python
def new_edge_key(self, u, v):
    try:
        keydict = self._adj[u][v]
    except KeyError:
        return 0
    key = len(keydict)
    while key in keydict:
        key += 1
    return key
```
默认使用整数作为边键，从 0 开始递增。

2. **边键层字典共享**：在 `multigraph.py:531-534` 中：
```python
keydict = self.edge_key_dict_factory()
keydict[key] = datadict
self._adj[u][v] = keydict
self._adj[v][u] = keydict  # 边键字典共享
```

3. **单个边属性字典**：每条边（由边键标识）有自己独立的属性字典。

**示例**：
```python
G = nx.MultiGraph()
key1 = G.add_edge(1, 2, weight=3.0)  # 返回 0
key2 = G.add_edge(1, 2, weight=5.0)  # 返回 1

# 内部存储
G._adj = {
    1: {
        2: {
            0: {'weight': 3.0},
            1: {'weight': 5.0}
        }
    },
    2: {
        1: {
            0: {'weight': 3.0},  # 与上面是同一个边键字典
            1: {'weight': 5.0}
        }
    }
}

# 验证边键字典共享
assert G._adj[1][2] is G._adj[2][1]  # True

# 访问特定边
print(G[1][2][0])  # {'weight': 3.0}
print(G[1][2][1])  # {'weight': 5.0}
```

### 2.5 MultiDiGraph（多重有向图）的存储结构

**存储层级：结合 DiGraph 和 MultiGraph 的四层嵌套字典**

```
self._node = {
    node1: {attr1: val1, ...},
    ...
}

self._succ = self._adj = {
    node1: {
        neighbor1: {
            edge_key1: {edge_attr1: val1, ...},
            edge_key2: {...},
            ...
        },
        ...
    },
    ...
}

self._pred = {
    node1: {
        neighbor1: {
            edge_key1: {edge_attr1: val1, ...},
            ...
        },
        ...
    },
    ...
}
```

**关键设计特点**：

1. **多重继承**：`MultiDiGraph` 继承自 `MultiGraph` 和 `DiGraph`（`multidigraph.py:22`）：
```python
class MultiDiGraph(MultiGraph, DiGraph):
```

2. **方法解析顺序（MRO）**：通过 MRO 机制正确组合两个父类的行为。

3. **边键层双向共享**：在 `multidigraph.py:518-521` 中：
```python
self._succ[u][v] = keydict
self._pred[v][u] = keydict  # 边键字典在后继和前驱间共享
```

**示例**：
```python
G = nx.MultiDiGraph()
key1 = G.add_edge(1, 2, weight=3.0)  # 1->2, key=0
key2 = G.add_edge(1, 2, weight=5.0)  # 1->2, key=1
key3 = G.add_edge(2, 1, weight=7.0)  # 2->1, key=0

# 内部存储
G._succ = {
    1: {2: {0: {'weight': 3.0}, 1: {'weight': 5.0}}},
    2: {1: {0: {'weight': 7.0}}}
}

G._pred = {
    1: {2: {0: {'weight': 7.0}}},
    2: {1: {0: {'weight': 3.0}, 1: {'weight': 5.0}}}
}

# 验证边键字典共享
assert G._succ[1][2] is G._pred[2][1]  # True
```

### 2.6 多重继承与方法解析顺序（MRO）

`MultiDiGraph` 是 NetworkX 中最复杂的图类型，它同时继承了 `MultiGraph` 和 `DiGraph`，这种多重继承设计需要仔细处理方法解析顺序（MRO）。

#### 2.6.1 继承层次与 MRO

**类定义**（`multidigraph.py:22`）：
```python
class MultiDiGraph(MultiGraph, DiGraph):
```

**继承层次**：
```
object
  │
  └── Graph
        │
        ├── DiGraph
        │
        └── MultiGraph
              │
              └── MultiDiGraph (同时继承 DiGraph 和 MultiGraph)
```

**Python MRO（C3 线性化）**：
```python
import networkx as nx
print([c.__name__ for c in nx.MultiDiGraph.__mro__])
# 输出: ['MultiDiGraph', 'MultiGraph', 'DiGraph', 'Graph', 'object']
```

注意：`MultiDiGraph` 列在第一位，然后是 `MultiGraph`，然后是 `DiGraph`，然后是 `Graph`。这意味着方法查找时优先检查 `MultiDiGraph` 自己，然后是 `MultiGraph`，再是 `DiGraph`。

#### 2.6.2 关键操作的方法解析

##### 1. `__init__` 方法

`MultiDiGraph.__init__` **显式调用** `DiGraph.__init__`，而不是通过 `super()` 使用 MRO（`multidigraph.py:360-373`）：

```python
def __init__(self, incoming_graph_data=None, multigraph_input=None, **attr):
    # ... 参数处理 ...
    else:
        DiGraph.__init__(self, incoming_graph_data, **attr)  # 显式调用
```

**设计意图**：
- `DiGraph.__init__` 初始化 `_succ` 和 `_pred` 两个字典（这是有向图的核心存储结构）
- 而 `Graph.__init__` 只初始化一个 `_adj` 字典
- 显式调用确保了正确的存储结构被初始化

##### 2. `add_edge` 方法

四种图类型都有**独立实现**的 `add_edge` 方法，因为它们的存储结构不同：

| 图类型 | add_edge 位置 | 存储操作 |
|--------|--------------|----------|
| `Graph` | `graph.py:968-994` | `self._adj[u][v] = self._adj[v][u] = datadict` |
| `DiGraph` | `digraph.py:730-756` | `self._succ[u][v] = self._pred[v][u] = datadict` |
| `MultiGraph` | `multigraph.py:442-536` | `self._adj[u][v] = self._adj[v][u] = keydict`（边键层） |
| `MultiDiGraph` | `multidigraph.py:427-523` | `self._succ[u][v] = self._pred[v][u] = keydict`（边键层） |

**`MultiDiGraph.add_edge` 的实现细节**（`multidigraph.py:507-521`）：

```python
if key is None:
    key = self.new_edge_key(u, v)  # 调用 MultiGraph.new_edge_key
if v in self._succ[u]:
    keydict = self._adj[u][v]
    datadict = keydict.get(key, self.edge_attr_dict_factory())
    datadict.update(attr)
    keydict[key] = datadict
else:
    datadict = self.edge_attr_dict_factory()
    datadict.update(attr)
    keydict = self.edge_key_dict_factory()
    keydict[key] = datadict
    self._succ[u][v] = keydict
    self._pred[v][u] = keydict  # 边键字典共享引用
```

**关键设计点**：
1. 调用 `self.new_edge_key(u, v)` 生成边键——这个方法继承自 `MultiGraph`
2. 边键字典（`keydict`）在 `_succ[u][v]` 和 `_pred[v][u]` 间共享引用
3. 每条边有独立的属性字典（`datadict`）

##### 3. `degree` 属性

`MultiDiGraph` 用 `@cached_property` 自己定义了 `degree` 属性（`multidigraph.py:722-768`）：

```python
@cached_property
def degree(self):
    return DiMultiDegreeView(self)
```

**不同 DegreeView 的度计算逻辑**：

| 视图类 | 适用图类型 | `__getitem__` 计算方式 |
|--------|-----------|------------------------|
| `DiDegreeView` | DiGraph | `len(succs) + len(preds)` |
| `DegreeView` | Graph | `len(nbrs) + (n in nbrs)`（自环修正） |
| `MultiDegreeView` | MultiGraph | `sum(len(keys) for keys in nbrs.values()) + (n in nbrs and len(nbrs[n]))` |
| `DiMultiDegreeView` | MultiDiGraph | `sum(len(keys) for keys in succs.values()) + sum(len(keys) for keys in preds.values())` |

**`DiMultiDegreeView` 的实现**（`reportviews.py:739-781`）：

```python
class DiMultiDegreeView(DiDegreeView):
    def __getitem__(self, n):
        weight = self._weight
        succs = self._succ[n]
        preds = self._pred[n]
        if weight is None:
            # 统计出边键数 + 入边键数
            return sum(len(keys) for keys in succs.values()) + sum(
                len(keys) for keys in preds.values()
            )
        # 加权度：遍历所有边键的属性字典
        deg = sum(
            d.get(weight, 1) for key_dict in succs.values() for d in key_dict.values()
        ) + sum(
            d.get(weight, 1) for key_dict in preds.values() for d in key_dict.values()
        )
        return deg
```

#### 2.6.3 对四层嵌套字典存储的影响

多重继承设计对 `MultiDiGraph` 的存储结构有以下具体影响：

##### 1. 存储结构初始化

由于显式调用 `DiGraph.__init__`，`MultiDiGraph` 初始化了**两个独立的四层嵌套字典**：

```python
# DiGraph.__init__ 中初始化
self._succ = self.adjlist_outer_dict_factory()  # 四层结构
self._pred = self.adjlist_outer_dict_factory()  # 四层结构
self._adj = self._succ  # _adj 是 _succ 的别名
```

而如果继承 `Graph`，只会初始化一个 `_adj` 字典。

##### 2. 边键字典的共享引用

`MultiDiGraph.add_edge` 中的共享引用机制：

```python
# 添加新边时
keydict = self.edge_key_dict_factory()  # 边键字典
keydict[key] = datadict                  # 边键 → 属性字典
self._succ[u][v] = keydict               # 存储到出边表
self._pred[v][u] = keydict               # 存储到入边表（同一引用）
```

**验证共享引用**：
```python
G = nx.MultiDiGraph()
key1 = G.add_edge(1, 2, weight=3.0)  # 1->2, key=0
key2 = G.add_edge(1, 2, weight=5.0)  # 1->2, key=1

# 边键字典共享
assert G._succ[1][2] is G._pred[2][1]  # True

# 每条边有独立的属性字典
assert G._succ[1][2][0] is not G._succ[1][2][1]  # True

# 通过 _succ 修改，通过 _pred 也能看到
G._succ[1][2][0]['weight'] = 10.0
print(G._pred[2][1][0]['weight'])  # 10.0
```

##### 3. 与 `MultiGraph` 的对比

| 特性 | MultiGraph | MultiDiGraph |
|------|-----------|--------------|
| 邻接字典 | 单个 `_adj` | `_succ` + `_pred` |
| 边键字典共享 | `_adj[u][v] is _adj[v][u]` | `_succ[u][v] is _pred[v][u]` |
| 方向 | 无向 | 有向 |
| 反向边 | 不适用（同一存储） | `_succ[v][u]` 是独立的 |

**反向边的独立性**：
```python
G = nx.MultiDiGraph()
G.add_edge(1, 2, weight=3.0)  # 1->2
G.add_edge(2, 1, weight=5.0)  # 2->1

# 这是两条完全独立的边
assert G._succ[1][2] is not G._succ[2][1]  # True
assert G._succ[1][2] is not G._pred[1][2]   # True（_pred[1][2] 是 2->1 的边键字典）
```

#### 2.6.4 方法重写与 MRO 的交互

`MultiDiGraph` 重写了大部分关键方法，避免了 MRO 带来的歧义：

| 方法 | 是否重写 | 调用来源 |
|------|----------|----------|
| `__init__` | 是 | 显式调用 `DiGraph.__init__` |
| `add_edge` | 是 | 自己的实现 |
| `remove_edge` | 是 | 自己的实现 |
| `adj/succ/pred` | 是 | `@cached_property` 返回 `MultiAdjacencyView` |
| `edges/out_edges` | 是 | `@cached_property` 返回 `OutMultiEdgeView` |
| `in_edges` | 是 | `@cached_property` 返回 `InMultiEdgeView` |
| `degree` | 是 | `@cached_property` 返回 `DiMultiDegreeView` |
| `new_edge_key` | 否 | 继承自 `MultiGraph` |
| `is_multigraph` | 是 | 返回 `True` |
| `is_directed` | 是 | 返回 `True` |

**设计原则**：
1. **核心修改操作**（`add_edge`, `remove_edge`）完全重写，因为存储结构不同
2. **视图属性**（`adj`, `edges`, `degree`）重写，返回适合多重有向图的视图类
3. **辅助方法**（`new_edge_key`）从 `MultiGraph` 继承，因为逻辑相同
4. **类型检查方法**（`is_multigraph`, `is_directed`）重写，返回正确的类型标识

### 2.7 四种图类型存储结构对比

| 特性 | Graph | DiGraph | MultiGraph | MultiDiGraph |
|------|-------|---------|------------|--------------|
| **嵌套层数** | 3层 | 3层（×2个字典） | 4层 | 4层（×2个字典） |
| **邻接字典** | `_adj` | `_succ` + `_pred` | `_adj` | `_succ` + `_pred` |
| **边键层** | 无 | 无 | 有 | 有 |
| **边属性共享** | `_adj[u][v]` 与 `_adj[v][u]` 共享 | `_succ[u][v]` 与 `_pred[v][u]` 共享 | 边键字典共享 | 边键字典共享 |
| **度计算** | 邻居数 + 自环修正 | 出度 + 入度 | 各邻居的边键数之和 + 自环修正 | 出边键数 + 入边键数 |

---

## 3. 惰性视图（Lazy Views）机制

NetworkX 的视图设计遵循 Python 字典视图（`dict.keys()`, `dict.values()`, `dict.items()`）的设计哲学：**不复制数据，只提供访问接口**。

### 3.1 视图的核心特性

1. **惰性求值（Lazy Evaluation）**：数据只在访问时才计算，不预先存储
2. **内存高效**：不复制底层数据，仅保存引用
3. **实时更新**：视图反映图的当前状态，图修改后视图自动更新
4. **只读结构**：视图本身不可修改（但可通过视图修改属性字典）

### 3.2 核心视图类（coreviews.py）

`coreviews.py` 定义了底层数据结构的视图类，主要用于邻接关系的访问。

#### 3.2.1 AtlasView

**用途**：只读的 `dict-of-dict` 视图

**继承**：`collections.abc.Mapping`

**核心实现**（`coreviews.py:23-63`）：
```python
class AtlasView(Mapping):
    __slots__ = ("_atlas",)  # 仅保存对底层字典的引用
    
    def __init__(self, d):
        self._atlas = d  # 保存引用，不复制
    
    def __len__(self):
        return len(self._atlas)
    
    def __iter__(self):
        return iter(self._atlas)
    
    def __getitem__(self, key):
        return self._atlas[key]  # 直接返回内部字典（可写）
```

**关键设计**：
- 使用 `__slots__` 减少内存开销
- `__getitem__` 返回内部字典本身，因此可以修改边属性：
  ```python
  G.adj[1][2]['weight'] = 5.0  # 有效，修改的是内部属性字典
  ```

#### 3.2.2 AdjacencyView

**用途**：只读的 `dict-of-dict-of-dict` 视图（用于非多重图的邻接关系）

**继承**：`AtlasView`

**核心实现**（`coreviews.py:66-86`）：
```python
class AdjacencyView(AtlasView):
    __slots__ = ()  # 复用父类的 _atlas
    
    def __getitem__(self, name):
        return AtlasView(self._atlas[name])  # 包装为 AtlasView
```

**设计意图**：
- 外层是 `AdjacencyView`（只读 Mapping）
- 内层是 `AtlasView`（只读 Mapping）
- 最内层是实际的属性字典（可写）

**访问示例**：
```python
G = nx.Graph()
G.add_edge(1, 2, weight=3.0)

adj_view = G.adj  # AdjacencyView
node1_adj = adj_view[1]  # AtlasView({2: {'weight': 3.0}})
edge_attr = node1_adj[2]  # {'weight': 3.0} (实际字典，可修改)

# 修改边属性
edge_attr['weight'] = 5.0  # 有效
print(G[1][2]['weight'])  # 5.0
```

#### 3.2.3 MultiAdjacencyView

**用途**：只读的 `dict-of-dict-of-dict-of-dict` 视图（用于多重图的邻接关系）

**继承**：`AdjacencyView`

**核心实现**（`coreviews.py:88-107`）：
```python
class MultiAdjacencyView(AdjacencyView):
    __slots__ = ()
    
    def __getitem__(self, name):
        return AdjacencyView(self._atlas[name])  # 返回 AdjacencyView
```

**访问层级**：
```
G.adj          # MultiAdjacencyView
G.adj[u]       # AdjacencyView (邻居 → 边键字典)
G.adj[u][v]    # AtlasView (边键 → 属性字典)
G.adj[u][v][k] # 属性字典（可写）
```

#### 3.2.4 Union* 系列视图

用于有向图中合并 `_succ` 和 `_pred` 的视图：

| 视图类 | 用途 | 合并的数据 |
|--------|------|------------|
| `UnionAtlas` | 合并两个 dict-of-dict | `G.succ[node]` + `G.pred[node]` |
| `UnionAdjacency` | 合并两个 dict-of-dict-of-dict | `G.succ` + `G.pred` |
| `UnionMultiInner` | 多重图的 UnionAtlas | 多重图的 `_succ[node]` + `_pred[node]` |
| `UnionMultiAdjacency` | 多重图的 UnionAdjacency | 多重图的 `_succ` + `_pred` |

**核心实现**（以 `UnionAtlas` 为例，`coreviews.py:110-162`）：
```python
class UnionAtlas(Mapping):
    __slots__ = ("_succ", "_pred")
    
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

**设计要点**：
- 动态计算两个字典的键的并集
- 不复制数据，仅在访问时合并视图
- 用于有向图的无向视图（如 `degree` 计算需要同时考虑入边和出边）

#### 3.2.5 有向图的无向视图合并机制

当有向图需要以无向方式访问时，NetworkX 使用 `Union*` 系列视图来动态合并 `_succ` 和 `_pred` 两个邻接表。

##### 1. 触发条件

**主要触发场景**（`graphviews.py:125-129`）：

```python
def generic_graph_view(G, create_using=None):
    # ...
    if newG.is_directed():
        # 有向视图，直接使用 _succ 和 _pred
    elif G.is_directed():
        # 原图是有向图，但新视图要求无向
        if G.is_multigraph():
            newG._adj = UnionMultiAdjacency(G._succ, G._pred)  # 多重图
        else:
            newG._adj = UnionAdjacency(G._succ, G._pred)      # 非多重图
```

**触发方式**：

| API 调用 | 触发条件 |
|----------|----------|
| `G.to_undirected(as_view=True)` | 明确要求创建无向视图 |
| `generic_graph_view(G, nx.Graph)` | 指定 `create_using` 为无向图类型 |
| `subgraph_view` 内部处理 | 某些子图操作可能触发 |

**示例**：
```python
DG = nx.DiGraph()
DG.add_edge(1, 2, weight=3.0)  # 1->2
DG.add_edge(2, 3, weight=5.0)  # 2->3
DG.add_edge(3, 1, weight=7.0)  # 3->1

# 创建无向视图
UG = DG.to_undirected(as_view=True)
# UG._adj 是 UnionAdjacency(DG._succ, DG._pred)

# 以无向方式访问
print(list(UG.edges))  # [(1, 2), (2, 3), (3, 1)]
print(UG.has_edge(1, 2))  # True
print(UG.has_edge(2, 1))  # True（无向视图中是同一条边）
```

##### 2. 合并逻辑详解

Union* 视图的核心是**惰性合并**——不复制数据，只在访问时动态计算并集。

###### 2.1 UnionAdjacency（非多重图）

**核心实现**（`coreviews.py:165-214`）：

```python
class UnionAdjacency(Mapping):
    __slots__ = ("_succ", "_pred")
    
    def __init__(self, succ, pred):
        # 断言：两个字典的键（节点）应该相同
        assert len(set(succ.keys()) ^ set(pred.keys())) == 0
        self._succ = succ
        self._pred = pred
    
    def __len__(self):
        return len(self._succ)  # 两个字典长度相同
    
    def __iter__(self):
        return iter(self._succ)
    
    def __getitem__(self, nbr):
        # 返回 UnionAtlas，合并该节点的出边和入边邻居
        return UnionAtlas(self._succ[nbr], self._pred[nbr])
```

**设计要点**：
- 外层节点迭代直接使用 `_succ` 的键（因为 `_succ` 和 `_pred` 的节点集相同）
- 访问 `G.adj[u]` 时返回 `UnionAtlas(self._succ[u], self._pred[u])`

###### 2.2 UnionAtlas（邻居层合并）

**核心实现**（`coreviews.py:110-162`）：

```python
class UnionAtlas(Mapping):
    __slots__ = ("_succ", "_pred")
    
    def __init__(self, succ, pred):
        self._succ = succ  # 出边邻居：{neighbor: edge_attr_dict}
        self._pred = pred  # 入边邻居：{neighbor: edge_attr_dict}
    
    def __len__(self):
        # 并集的大小：出边邻居 + 入边邻居 - 重复的
        return len(self._succ.keys() | self._pred.keys())
    
    def __iter__(self):
        # 迭代并集
        return iter(set(self._succ.keys()) | set(self._pred.keys()))
    
    def __getitem__(self, key):
        # 优先从出边找，找不到从入边找
        try:
            return self._succ[key]
        except KeyError:
            return self._pred[key]
```

**合并逻辑示例**：

```python
DG = nx.DiGraph()
DG.add_edge(1, 2, weight=3.0)  # 1->2
DG.add_edge(3, 1, weight=5.0)  # 3->1
DG.add_edge(1, 4, weight=7.0)  # 1->4
DG.add_edge(2, 1, weight=9.0)  # 2->1

# 对于节点 1：
# _succ[1] = {2: {'weight': 3.0}, 4: {'weight': 7.0}}  出边
# _pred[1] = {3: {'weight': 5.0}, 2: {'weight': 9.0}}  入边

# UnionAtlas(_succ[1], _pred[1]) 的行为：
# len() = len({2,4} | {3,2}) = 3  (邻居: 2, 3, 4)
# __getitem__(2) = _succ[2]? 不，是 _succ[1][2] = {'weight': 3.0}
# __getitem__(3) = _pred[1][3] = {'weight': 5.0}
```

**注意**：当同一对节点间存在双向边时，`__getitem__` 优先返回出边的属性字典。这是设计上的权衡。

###### 2.3 多重图的 UnionMultiAdjacency 和 UnionMultiInner

对于多重有向图，使用 `UnionMultiAdjacency` 和 `UnionMultiInner` 来处理四层嵌套字典。

**UnionMultiInner**（`coreviews.py:217-246`）：

```python
class UnionMultiInner(UnionAtlas):
    __slots__ = ()  # 复用 _succ, _pred
    
    def __getitem__(self, node):
        in_succ = node in self._succ
        in_pred = node in self._pred
        if in_succ:
            if in_pred:
                # 双向都有，返回 UnionAtlas
                return UnionAtlas(self._succ[node], self._pred[node])
            return UnionAtlas(self._succ[node], {})
        return UnionAtlas({}, self._pred[node])
```

**关键差异**：
- 多重图中，`_succ[node] 是 `{neighbor: {edge_key: edge_attr}}`
- 所以需要额外的 `UnionMultiInner` 来处理边键层的合并

**访问层级对比**：

| 图类型 | `G.adj[u] 返回 | `G.adj[u][v]` 返回 |
|--------|------------------|---------------------|
| DiGraph（无向视图） | `UnionAtlas` | 边属性字典 |
| MultiDiGraph（无向视图） | `UnionMultiInner` | `UnionAtlas`（边键层） |

##### 3. 实时更新保证

Union* 视图通过以下机制保证实时更新：

###### 3.1 仅保存引用，不复制数据

```python
class UnionAtlas(Mapping):
    def __init__(self, succ, pred):
        self._succ = succ  # 仅保存引用
        self._pred = pred  # 仅保存引用
```

视图对象不存储任何数据的副本，只保存对底层 `_succ` 和 `_pred` 字典的引用。

###### 3.2 每次访问实时计算

```python
def __len__(self):
    # 每次访问都重新计算并集
    return len(self._succ.keys() | self._pred.keys())

def __iter__(self):
    # 每次迭代都重新计算并集
    return iter(set(self._succ.keys()) | set(self._pred.keys()))

def __getitem__(self, key):
    # 每次访问都从原始字典获取
    try:
        return self._succ[key]
    except KeyError:
        return self._pred[key]
```

**实时更新示例**：

```python
DG = nx.DiGraph()
DG.add_edge(1, 2)
DG.add_edge(2, 3)

# 创建无向视图
UG = DG.to_undirected(as_view=True)

print(list(UG.edges))  # [(1, 2), (2, 3)]
print(UG.degree(1))      # 1

# 修改原图
DG.add_edge(3, 1)  # 添加 3->1

# 视图自动更新
print(list(UG.edges))  # [(1, 2), (2, 3), (3, 1)]
print(UG.degree(1))      # 2（1->2 和 3->1）
```

###### 3.3 与 DegreeView 的实时性

`DiDegreeView` 在计算度时也是实时访问 `_succ` 和 `_pred`：

```python
class DiDegreeView:
    def __getitem__(self, n):
        succs = self._succ[n]  # 实时访问
        preds = self._pred[n]  # 实时访问
        if weight is None:
            return len(succs) + len(preds)  # 实时计算
```

##### 4. 设计权衡与注意事项

###### 4.1 双向边的属性访问

当存在双向边时，`UnionAtlas.__getitem__` 优先返回出边的属性字典：

```python
DG = nx.DiGraph()
DG.add_edge(1, 2, weight=3.0, direction='out')  # 1->2
DG.add_edge(2, 1, weight=5.0, direction='in')   # 2->1

UG = DG.to_undirected(as_view=True)

# 访问 UG.adj[1][2]
# 优先返回 _succ[1][2] = {'weight': 3.0, 'direction': 'out'}
print(UG.adj[1][2])  # {'weight': 3.0, 'direction': 'out'}

# 但这两条边在度计算时都会被统计
print(UG.degree(1))  # 2（出边和入边各算一条）
```

这意味着在无向视图中访问边属性时，可能只看到其中一条边的属性。这是设计上的限制。

###### 4.2 性能考虑

Union* 视图的操作有额外开销：

| 操作 | 普通视图 | Union* 视图 |
|------|----------|-------------|
| `len()` | O(1) | O(k) 计算并集 |
| `iter()` | O(k) | O(k) 计算并集 + 迭代 |
| `__getitem__` | O(1) | O(1) 两次字典查找 |

其中 k 是邻居数量。对于大型图，频繁调用 `len()` 或迭代可能会有性能影响。

###### 4.3 内存效率

虽然有运行时开销，但 Union* 视图的内存效率很高：

```python
# 每个 Union* 对象只保存两个引用
class UnionAtlas:
    __slots__ = ("_succ", "_pred")  # 非常小的内存占用
```

相比之下，如果创建真正的无向图副本需要 O(m) 内存存储边数据。

#### 3.2.5 Filter* 系列视图

用于子图过滤的视图：

| 视图类 | 用途 | 过滤层级 |
|--------|------|----------|
| `FilterAtlas` | 过滤节点的 dict-of-dict 视图 | 节点层 |
| `FilterAdjacency` | 过滤节点和边的邻接视图 | 节点 + 边 |
| `FilterMultiInner` | 多重图的过滤内层视图 | 节点 + 边键 |
| `FilterMultiAdjacency` | 多重图的过滤邻接视图 | 节点 + 边 + 边键 |

**核心实现**（以 `FilterAdjacency` 为例，`coreviews.py:314-365`）：
```python
class FilterAdjacency(Mapping):
    def __init__(self, d, NODE_OK, EDGE_OK):
        self._atlas = d
        self.NODE_OK = NODE_OK  # 节点过滤函数
        self.EDGE_OK = EDGE_OK  # 边过滤函数
    
    def __getitem__(self, node):
        if node in self._atlas and self.NODE_OK(node):
            def new_node_ok(nbr):
                return self.NODE_OK(nbr) and self.EDGE_OK(node, nbr)
            return FilterAtlas(self._atlas[node], new_node_ok)
        raise KeyError(f"Key {node} not found")
```

**设计要点**：
- 使用函数/可调用对象作为过滤条件
- 惰性过滤：只在访问时检查条件
- 用于 `subgraph()` 等方法创建子图视图

---

## 3.3 有向图的无向视图合并机制

当有向图需要以无向方式访问时，NetworkX 使用 `Union*` 系列视图来动态合并 `_succ`（出边表）和 `_pred`（入边表）两个邻接表。这是一个典型的**惰性视图合并**设计，不复制任何数据，只在访问时动态计算。

### 3.3.1 触发条件：什么接口调用会走到这条路径？

无向视图的合并机制由 **`graphviews.py`** 中的视图创建逻辑触发，主要有以下三种触发方式：

#### 方式一：`to_undirected(as_view=True)`

这是最常用的触发方式，用户显式要求创建无向视图：

```python
import networkx as nx

DG = nx.DiGraph()
DG.add_edge(1, 2, weight=3.0)
DG.add_edge(2, 3, weight=5.0)

# 触发方式：as_view=True
UG = DG.to_undirected(as_view=True)
```

**关键代码路径**（`digraph.py:578-603`）：
```python
def to_undirected(self, reciprocal=False, as_view=False):
    if as_view:
        from networkx.classes.graphviews import generic_graph_view
        # 调用 generic_graph_view，create_using=nx.Graph
        return generic_graph_view(self, create_using=Graph)
    # ... 否则创建真正的无向图副本
```

#### 方式二：`generic_graph_view(G, create_using=nx.Graph)`

这是内部核心触发点，在 `graphviews.py:125-129` 中：

```python
def generic_graph_view(G, create_using=None):
    # ... 省略前置代码 ...
    
    # 核心判断逻辑
    if newG.is_directed():
        # 新视图也是有向图：直接使用 _succ 和 _pred
        pass
    elif G.is_directed():
        # 原图是有向图，但新视图要求无向 → 触发 Union* 视图合并
        if G.is_multigraph():
            # 多重有向图 → 无向视图
            newG._adj = UnionMultiAdjacency(G._succ, G._pred)
        else:
            # 普通有向图 → 无向视图
            newG._adj = UnionAdjacency(G._succ, G._pred)
    
    # ... 省略后续代码 ...
```

**触发条件的关键判断**：

| 条件 | 值 | 含义 |
|------|-----|------|
| `G.is_directed()` | `True` | 原图是有向图 |
| `newG.is_directed()` | `False` | 新视图要求无向 |
| **触发合并** | **是** | 使用 `Union*` 视图 |

#### 方式三：间接触发（如某些视图操作）

某些视图操作可能间接触发这个机制，例如：

```python
# 某些子图操作可能内部使用 generic_graph_view
from networkx.classes.graphviews import subgraph_view

# 取决于具体实现，某些 subgraph_view 操作可能触发
```

#### 完整触发流程图

```
用户调用
    │
    ├── DG.to_undirected(as_view=True)
    │         │
    │         └── generic_graph_view(DG, create_using=Graph)
    │                   │
    │                   └── 判断：G.is_directed()=True, newG.is_directed()=False
    │                           │
    │                           └── 是 → newG._adj = UnionAdjacency(DG._succ, DG._pred)
    │
    ├── generic_graph_view(DG, create_using=nx.Graph)
    │         │
    │         └── 同上
    │
    └── 其他间接调用
              │
              └── 内部调用 generic_graph_view
```

### 3.3.2 合并逻辑：内部如何把出边表和入边表合并成惰性视图？

合并逻辑由 **`coreviews.py`** 中的 `Union*` 系列视图类实现，核心是**惰性合并**——不复制数据，只在访问时动态计算并集。

#### 整体架构：两层 Union 视图

对于非多重有向图，合并涉及两层视图：

```
用户访问 UG.adj
         │
         └── UnionAdjacency（合并 _succ 和 _pred 两个邻接表）
                   │
                   ├── 外层节点：直接使用 _succ 的键（节点集相同）
                   │
                   └── 访问 UG.adj[u] 时 → 返回 UnionAtlas
                             │
                             └── UnionAtlas（合并 _succ[u] 和 _pred[u] 两个邻居字典）
                                       │
                                       ├── len()：计算两个字典的键的并集大小
                                       ├── iter()：迭代两个字典的键的并集
                                       └── __getitem__(v)：优先查 _succ[u][v]，再查 _pred[u][v]
```

#### 第一层：UnionAdjacency（邻接表层合并）

**定义位置**：`coreviews.py:165-214`

**核心实现**：

```python
class UnionAdjacency(Mapping):
    __slots__ = ("_succ", "_pred")  # 仅保存两个引用，不复制数据
    
    def __init__(self, succ, pred):
        # 断言：两个字典的键（节点）必须相同
        # 因为有向图中 _succ 和 _pred 的节点集总是一致的
        assert len(set(succ.keys()) ^ set(pred.keys())) == 0
        self._succ = succ  # 仅保存引用
        self._pred = pred  # 仅保存引用
    
    def __len__(self):
        # 两个字典长度相同，直接返回 _succ 的长度
        return len(self._succ)
    
    def __iter__(self):
        # 直接迭代 _succ 的键（节点集相同）
        return iter(self._succ)
    
    def __getitem__(self, node):
        # 关键：访问具体节点的邻居时，返回 UnionAtlas
        # UnionAtlas 负责合并该节点的出边邻居和入边邻居
        return UnionAtlas(self._succ[node], self._pred[node])
```

**设计要点**：
1. **节点集假设**：`_succ` 和 `_pred` 的键（节点）完全相同
   - 这是有向图的基本不变式：任何节点必须同时出现在 `_succ` 和 `_pred` 中
   - 即使节点没有出边或入边，也会有一个空字典 `{}`
2. **延迟合并**：`UnionAdjacency` 本身不做任何合并
   - `__len__` 和 `__iter__` 直接使用 `_succ`
   - 只有当访问具体节点 `G.adj[node]` 时，才返回 `UnionAtlas` 进行实际合并

#### 第二层：UnionAtlas（邻居层合并）

**定义位置**：`coreviews.py:110-162`

**核心实现**：

```python
class UnionAtlas(Mapping):
    __slots__ = ("_succ", "_pred")  # 仅保存两个引用
    
    def __init__(self, succ, pred):
        # succ: 出边邻居字典 {neighbor: edge_attr_dict, ...}
        # pred: 入边邻居字典 {neighbor: edge_attr_dict, ...}
        self._succ = succ
        self._pred = pred
    
    def __len__(self):
        # 核心：每次访问都动态计算并集大小
        # _succ.keys() | _pred.keys() = 出边邻居 ∪ 入边邻居
        return len(self._succ.keys() | self._pred.keys())
    
    def __iter__(self):
        # 核心：每次迭代都动态计算并集
        return iter(set(self._succ.keys()) | set(self._pred.keys()))
    
    def __getitem__(self, key):
        # 核心：优先从出边找，找不到从入边找
        try:
            return self._succ[key]
        except KeyError:
            return self._pred[key]
```

#### 合并逻辑的具体示例

让我们用一个具体例子来理解合并过程：

```python
DG = nx.DiGraph()
DG.add_edge(1, 2, weight=3.0)  # 1→2（出边）
DG.add_edge(3, 1, weight=5.0)  # 3→1（对1来说是入边）
DG.add_edge(1, 4, weight=7.0)  # 1→4（出边）
DG.add_edge(2, 1, weight=9.0)  # 2→1（对1来说是入边，双向边）
```

**底层存储结构**：

```python
# 节点 1 的出边邻居
DG._succ[1] = {
    2: {'weight': 3.0},  # 1→2
    4: {'weight': 7.0}   # 1→4
}

# 节点 1 的入边邻居
DG._pred[1] = {
    3: {'weight': 5.0},  # 3→1
    2: {'weight': 9.0}   # 2→1（注意：这是独立的边）
}
```

**创建无向视图后**：

```python
UG = DG.to_undirected(as_view=True)

# UG._adj 是 UnionAdjacency(DG._succ, DG._pred)

# 访问 UG.adj[1] → 返回 UnionAtlas(DG._succ[1], DG._pred[1])
adj_1 = UG.adj[1]  # UnionAtlas 实例
```

**UnionAtlas 的行为**：

| 操作 | 实际计算 | 结果 |
|------|----------|------|
| `len(adj_1)` | `len({2,4} \| {3,2})` | `3`（邻居：2, 3, 4） |
| `list(adj_1)` | `list({2,4} \| {3,2})` | `[2, 3, 4]`（顺序不确定） |
| `adj_1[2]` | 优先查 `_succ[1][2]` | `{'weight': 3.0}`（出边属性） |
| `adj_1[3]` | 查 `_succ[1][3]` 失败，查 `_pred[1][3]` | `{'weight': 5.0}` |

**注意双向边的处理**：
- 节点 2 同时在 `_succ[1]` 和 `_pred[1]` 中
- `adj_1[2]` 优先返回 `_succ[1][2]` 的属性（`weight=3.0`）
- 但 `_pred[1][2]` 中的边（`weight=9.0`）在度计算时仍然会被统计

#### 多重图的特殊处理：UnionMultiAdjacency + UnionMultiInner

对于多重有向图（`MultiDiGraph`），存储结构是四层嵌套字典，需要额外的视图层：

**存储结构**：
```
_succ[node] = {
    neighbor: {
        edge_key_0: {edge_attr},  # 边 0
        edge_key_1: {edge_attr},  # 边 1
        ...
    },
    ...
}
```

**视图层次**：
```
用户访问 UG.adj[u]
         │
         └── UnionMultiInner（合并 _succ[u] 和 _pred[u]）
                   │
                   └── 访问 UG.adj[u][v] 时 → 返回 UnionAtlas
                             │
                             └── UnionAtlas（合并 _succ[u][v] 和 _pred[u][v] 两个边键字典）
```

**UnionMultiInner 实现**（`coreviews.py:217-246`）：

```python
class UnionMultiInner(UnionAtlas):
    __slots__ = ()  # 复用 UnionAtlas 的 _succ 和 _pred
    
    def __getitem__(self, node):
        in_succ = node in self._succ
        in_pred = node in self._pred
        
        if in_succ:
            if in_pred:
                # 双向都有边 → 返回 UnionAtlas 合并两个边键字典
                return UnionAtlas(self._succ[node], self._pred[node])
            return UnionAtlas(self._succ[node], {})
        return UnionAtlas({}, self._pred[node])
```

**访问层级对比**：

| 图类型 | `G.adj[u]` 返回 | `G.adj[u][v]` 返回 |
|--------|------------------|---------------------|
| DiGraph（无向视图） | `UnionAtlas` | 边属性字典 |
| MultiDiGraph（无向视图） | `UnionMultiInner` | `UnionAtlas`（边键层） |

### 3.3.3 实时更新保证：合并视图如何保证实时反映图的变化？

Union* 系列视图通过**三个核心机制**保证实时更新，即：当原图发生修改时，视图会自动反映这些变化。

#### 机制一：仅保存引用，不复制数据

所有 Union* 视图类都只保存对底层字典的**引用**，不复制任何数据：

```python
class UnionAtlas(Mapping):
    __slots__ = ("_succ", "_pred")  # 仅两个引用
    
    def __init__(self, succ, pred):
        self._succ = succ  # 保存引用，不复制
        self._pred = pred  # 保存引用，不复制
```

**内存对比**：

| 方式 | 内存占用 | 实时性 |
|------|----------|--------|
| 复制数据（如 `dict.copy()`） | O(n) 复制整个字典 | ❌ 快照式，不实时 |
| 保存引用（Union* 视图） | O(1) 仅两个指针 | ✅ 实时访问原数据 |

#### 机制二：每次访问实时计算

Union* 视图的所有访问方法都**直接访问原始字典**，不缓存任何结果：

```python
class UnionAtlas(Mapping):
    def __len__(self):
        # 每次访问都重新计算并集
        return len(self._succ.keys() | self._pred.keys())
    
    def __iter__(self):
        # 每次迭代都重新计算并集
        return iter(set(self._succ.keys()) | set(self._pred.keys()))
    
    def __getitem__(self, key):
        # 每次访问都从原始字典获取
        try:
            return self._succ[key]
        except KeyError:
            return self._pred[key]
```

**关键设计**：
- 没有 `self._cache` 或类似的缓存属性
- 每次调用都重新计算 `_succ.keys() | _pred.keys()`
- `__getitem__` 直接查 `_succ` 和 `_pred` 字典

#### 机制三：与 DegreeView 等其他视图的一致性

无向视图的 `degree` 等属性也依赖实时访问：

```python
# DiDegreeView（用于有向图的度计算）
class DiDegreeView:
    def __getitem__(self, n):
        succs = self._succ[n]  # 实时访问 _succ
        preds = self._pred[n]  # 实时访问 _pred
        if weight is None:
            return len(succs) + len(preds)  # 实时计算
```

#### 实时更新的完整示例

```python
import networkx as nx

# 1. 创建有向图
DG = nx.DiGraph()
DG.add_edge(1, 2)
DG.add_edge(2, 3)

# 2. 创建无向视图
UG = DG.to_undirected(as_view=True)

# 3. 检查初始状态
print("初始状态：")
print(f"  UG.edges: {list(UG.edges)}")  # [(1, 2), (2, 3)]
print(f"  UG.degree(1): {UG.degree(1)}")  # 1（只有 1→2 的出边）

# 4. 修改原图
print("\n修改原图：添加 3→1 的边")
DG.add_edge(3, 1)  # 直接修改原图，不涉及视图

# 5. 视图自动更新
print("\n视图自动更新后：")
print(f"  UG.edges: {list(UG.edges)}")  # [(1, 2), (2, 3), (3, 1)]
print(f"  UG.degree(1): {UG.degree(1)}")  # 2（1→2 和 3→1）

# 6. 再修改：删除边
print("\n再修改：删除 1→2 的边")
DG.remove_edge(1, 2)

# 7. 视图再次更新
print("\n视图再次更新后：")
print(f"  UG.edges: {list(UG.edges)}")  # [(2, 3), (3, 1)]
print(f"  UG.degree(1): {UG.degree(1)}")  # 1（只有 3→1）
```

**为什么能自动更新？**

当执行 `DG.add_edge(3, 1)` 时：
1. `DG._succ[3][1]` 被添加
2. `DG._pred[1][3]` 被添加
3. `UG._adj` 是 `UnionAdjacency(DG._succ, DG._pred)`
4. 当访问 `UG.edges` 时，迭代器最终会访问 `_succ` 和 `_pred`
5. `UnionAtlas.__iter__()` 会计算 `_succ.keys() | _pred.keys()`
6. 新添加的边会被包含在并集中

#### 实时更新的边界与注意事项

**什么情况下视图会实时更新？**

| 操作类型 | 示例 | 实时性 |
|----------|------|--------|
| 添加边 | `DG.add_edge(u, v)` | ✅ 实时 |
| 删除边 | `DG.remove_edge(u, v)` | ✅ 实时 |
| 添加节点 | `DG.add_node(n)` | ✅ 实时 |
| 删除节点 | `DG.remove_node(n)` | ✅ 实时 |
| 修改边属性 | `DG.edges[u, v]['weight'] = 5` | ✅ 实时（属性字典是同一引用） |

**什么情况下可能出现问题？**

1. **迭代过程中修改图**：

```python
# 这可能导致问题
for u, v in UG.edges:
    DG.remove_edge(u, v)  # 迭代过程中修改字典
```

这与 Python 的 `RuntimeError: dictionary changed size during iteration` 是同一类问题。

2. **双向边的属性访问歧义**：

```python
DG = nx.DiGraph()
DG.add_edge(1, 2, weight=3.0)  # 1→2，weight=3
DG.add_edge(2, 1, weight=5.0)  # 2→1，weight=5

UG = DG.to_undirected(as_view=True)

# 访问 UG.adj[1][2]
# 优先返回 _succ[1][2] = {'weight': 3.0}
print(UG.adj[1][2])  # {'weight': 3.0}

# 但度计算会统计两条边
print(UG.degree(1))  # 2（出边和入边各一条）
```

这是设计权衡：边属性只返回其中一条，但度计算会正确统计。

### 3.3.4 设计权衡与性能分析

#### 性能开销对比

| 操作 | 普通视图（AdjacencyView） | Union* 视图 | 差异 |
|------|---------------------------|-------------|------|
| `__len__` | O(1) | O(k) 计算并集 | Union* 更慢 |
| `__iter__` | O(k) | O(k) 计算并集 + O(k) 迭代 | Union* 略慢 |
| `__getitem__` | O(1) 一次查找 | O(1) 最多两次查找 | 差异很小 |

其中 k 是邻居数量。

#### 内存效率对比

| 方式 | 内存占用 | 适用场景 |
|------|----------|----------|
| 创建真正的无向图副本 | O(m) 复制所有边数据 | 需要持久化修改 |
| Union* 视图 | O(1) 仅保存引用 | 临时访问、内存敏感场景 |

#### 设计决策总结

NetworkX 选择 Union* 视图而不是复制数据，基于以下权衡：

| 维度 | Union* 视图方案 | 复制数据方案 |
|------|-----------------|--------------|
| **内存** | O(1) 极低 | O(m) 复制所有边 |
| **实时性** | ✅ 自动反映原图变化 | ❌ 快照式，需要手动同步 |
| **性能** | 访问有额外开销（计算并集） | 访问无额外开销 |
| **实现复杂度** | 需要 Union* 系列视图类 | 简单，直接复制 |
| **适用场景** | 临时无向访问、大型图 | 需要持久化无向图 |

**核心设计哲学**：
> 内存优先，实时性优先。通过惰性计算和引用语义，在保持实时性的同时最小化内存开销。

---

### 3.4 高级视图类（reportviews.py）

`reportviews.py` 定义了面向用户的高级视图，提供更丰富的接口。

#### 3.3.1 NodeView

**用途**：节点的集合视图 + 字典视图

**继承**：`Mapping` + `Set`（同时支持 dict-like 和 set-like 操作）

**核心实现**（`reportviews.py:226-392`）：
```python
class NodeView(Mapping, Set):
    __slots__ = ("_nodes",)
    
    def __init__(self, graph):
        self._nodes = graph._node  # 仅保存引用
    
    # Mapping 协议
    def __len__(self):
        return len(self._nodes)
    
    def __iter__(self):
        return iter(self._nodes)
    
    def __getitem__(self, n):
        return self._nodes[n]  # 返回节点属性字典
    
    # Set 协议
    def __contains__(self, n):
        return n in self._nodes
    
    @classmethod
    def _from_iterable(cls, it):
        return set(it)  # 支持集合操作
    
    # DataView 工厂方法
    def __call__(self, data=False, default=None):
        if data is False:
            return self
        return NodeDataView(self._nodes, data, default)
    
    def data(self, data=True, default=None):
        if data is False:
            return self
        return NodeDataView(self._nodes, data, default)
```

**使用示例**：
```python
G = nx.Graph()
G.add_node(1, color='red', size=10)
G.add_node(2, color='blue')

nodes = G.nodes  # NodeView

# Set-like 操作
print(1 in nodes)           # True
print(len(nodes))           # 2
print(nodes & {1, 2, 3})    # {1, 2}

# Mapping-like 操作
print(nodes[1])             # {'color': 'red', 'size': 10}
print(dict(nodes))          # {1: {'color': 'red', 'size': 10}, 2: {'color': 'blue'}}

# DataView
for n, color in nodes.data('color', default='white'):
    print(n, color)  # 1 red, 2 blue
```

#### 3.3.2 EdgeView

**用途**：边的集合视图 + 字典视图

**继承**：`Set` + `Mapping` + `EdgeViewABC`

**核心特点**：

1. **无向图的边去重**：`EdgeView` 在迭代时确保每条边只出现一次（`reportviews.py:1376-1383`）：
```python
def __iter__(self):
    seen = {}
    for n, nbrs in self._nodes_nbrs():
        for nbr in list(nbrs):
            if nbr not in seen:
                yield (n, nbr)
        seen[n] = 1
    del seen
```

2. **有向图的边方向**：`OutEdgeView` 和 `InEdgeView` 分别处理出边和入边

3. **多重图的边键**：`MultiEdgeView` 和 `OutMultiEdgeView` 包含边键信息

**使用示例**：
```python
# 无向图
G = nx.Graph()
G.add_edge(1, 2, weight=3.0)
G.add_edge(2, 3, weight=5.0)

edges = G.edges  # EdgeView

# Set-like 操作
print((1, 2) in edges)      # True
print((2, 1) in edges)      # True（无向图）
print(len(edges))           # 2

# Mapping-like 操作
print(edges[1, 2])          # {'weight': 3.0}
print(edges[2, 1])          # {'weight': 3.0}（同一字典）

# DataView
for u, v, w in edges.data('weight', default=1):
    print(u, v, w)  # 1 2 3.0, 2 3 5.0

# 多重图
MG = nx.MultiGraph()
MG.add_edge(1, 2, weight=3.0)  # key=0
MG.add_edge(1, 2, weight=5.0)  # key=1

medges = MG.edges  # MultiEdgeView
print(list(medges))           # [(1, 2, 0), (1, 2, 1)]（包含边键）
print(medges[1, 2, 0])        # {'weight': 3.0}（需要边键访问）
```

#### 3.3.3 DegreeView

**用途**：节点度的视图

**核心特点**：

1. **惰性计算**：度值不存储，每次访问时实时计算

2. **多种实现**：针对不同图类型有不同的计算方式：

| 视图类 | 适用图类型 | 度计算方式 |
|--------|-----------|-----------|
| `DegreeView` | Graph | `len(nbrs) + (n in nbrs)`（邻居数 + 自环修正） |
| `DiDegreeView` | DiGraph | `len(succs) + len(preds)`（出度 + 入度） |
| `OutDegreeView` | DiGraph | `len(succs)`（仅出度） |
| `InDegreeView` | DiGraph | `len(preds)`（仅入度） |
| `MultiDegreeView` | MultiGraph | `sum(len(keys) for keys in nbrs.values())`（边键数之和） |
| `DiMultiDegreeView` | MultiDiGraph | 出边键数 + 入边键数 |

3. **加权度支持**：通过 `weight` 参数计算加权度

**核心实现**（以 `DegreeView` 为例，`reportviews.py:586-651`）：
```python
class DegreeView(DiDegreeView):
    def __getitem__(self, n):
        weight = self._weight
        nbrs = self._succ[n]
        if weight is None:
            return len(nbrs) + (n in nbrs)  # 自环被计算两次
        return sum(dd.get(weight, 1) for dd in nbrs.values()) + (
            n in nbrs and nbrs[n].get(weight, 1)
        )
```

**自环处理说明**：
- 在无向图中，自环边在 `_adj[n]` 中只出现一次（`_adj[n][n]`）
- 但度的计算应该将自环计为 2（一条边贡献两个端点）
- 因此需要 `+ (n in nbrs)` 来额外加 1

**使用示例**：
```python
G = nx.Graph()
G.add_edge(1, 2)
G.add_edge(2, 3)
G.add_edge(3, 3)  # 自环

degree = G.degree  # DegreeView

print(degree[1])   # 1
print(degree[2])   # 2
print(degree[3])   # 2（自环计为 2）

# 加权度
WG = nx.Graph()
WG.add_edge(1, 2, weight=3.0)
WG.add_edge(2, 3, weight=5.0)

wdegree = WG.degree(weight='weight')
print(wdegree[1])  # 3.0
print(wdegree[2])  # 8.0
print(dict(wdegree))  # {1: 3.0, 2: 8.0, 3: 5.0}
```

### 3.4 缓存属性机制

NetworkX 使用 `cached_property` 装饰器来缓存视图对象，并在图修改时自动失效。

#### 3.4.1 cached_property 装饰器

在 `graph.py:392-409` 中：
```python
@cached_property
def adj(self):
    return AdjacencyView(self._adj)
```

`cached_property` 是 Python 标准库装饰器，它：
- 将方法转换为属性
- 第一次访问时计算并缓存结果
- 后续访问直接返回缓存值

#### 3.4.2 缓存失效机制

NetworkX 使用**数据描述符（Data Descriptor）**来实现缓存的自动失效。

以 `_CachedPropertyResetterAdj` 为例（`graph.py:22-44`）：
```python
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

**工作原理**：

1. `Graph` 类中定义：
```python
_adj = _CachedPropertyResetterAdj()
```

2. 当 `G._adj = some_value` 被赋值时：
   - 描述符的 `__set__` 方法被调用
   - 它将值存储到实例的 `__dict__["_adj"]`
   - 同时删除 `__dict__` 中缓存的 `adj`, `edges`, `degree` 等属性

3. 下次访问 `G.adj` 时：
   - `cached_property` 发现 `__dict__` 中没有缓存
   - 重新计算并缓存新的视图对象

#### 3.4.3 图修改时的缓存清理

除了描述符机制，NetworkX 在图修改操作中还显式调用 `nx._clear_cache(self)`：

```python
def add_edge(self, u, v, **attr):
    # ... 添加边的逻辑 ...
    nx._clear_cache(self)  # 显式清理缓存

def remove_node(self, n):
    # ... 移除节点的逻辑 ...
    nx._clear_cache(self)
```

这确保了任何修改图结构的操作都会使缓存失效。

### 3.5 视图设计总结

#### 3.5.1 视图层级结构

```
用户访问
    │
    ├── G.nodes ──────────────────────→ NodeView
    │                                       │
    │                                       ├── 迭代: iter(G._node)
    │                                       ├── 包含: n in G._node
    │                                       ├── 查找: G._node[n]
    │                                       └── G.nodes(data='attr') → NodeDataView
    │
    ├── G.edges ──────────────────────→ EdgeView / OutEdgeView / MultiEdgeView
    │                                       │
    │                                       ├── 迭代: 遍历 _adj/_succ
    │                                       ├── 包含: (u,v) 检查
    │                                       ├── 查找: G._adj[u][v]
    │                                       └── G.edges(data='attr') → EdgeDataView
    │
    ├── G.adj / G.succ ───────────────→ AdjacencyView / MultiAdjacencyView
    │   (G.pred 用于有向图)
    │                                       │
    │                                       ├── 迭代: iter(G._adj)
    │                                       ├── 查找: G._adj[u] → AtlasView
    │                                       └── G.adj[u][v] → 属性字典
    │
    └── G.degree ─────────────────────→ DegreeView / DiDegreeView / ...
                                            │
                                            ├── 迭代: 实时计算每个节点的度
                                            └── 查找: 实时计算指定节点的度
```

#### 3.5.2 设计模式总结

| 设计模式 | 应用场景 | 实现方式 |
|----------|----------|----------|
| **视图模式** | 所有视图类 | 保存引用，不复制数据 |
| **惰性求值** | DegreeView, Filter*View | 访问时才计算 |
| **描述符模式** | 缓存失效 | `_CachedPropertyResetter*` 类 |
| **协议继承** | 视图接口 | 继承 `Mapping`, `Set` ABC |
| **组合模式** | 高级视图 | 组合多个底层视图 |

---

## 4. 关键设计决策与权衡

### 4.1 嵌套字典 vs 其他数据结构

**选择嵌套字典的原因**：

1. **O(1) 平均时间复杂度**：
   - 节点查找：`node in G._node` → O(1)
   - 边查找：`v in G._adj[u]` → O(1)
   - 属性访问：`G._adj[u][v]['weight']` → O(1)

2. **灵活性**：
   - 节点可以是任意可哈希对象（不限于整数）
   - 边属性可以是任意键值对
   - 易于动态添加/删除节点和边

3. **Python 原生支持**：
   - 字典是 Python 内置高效数据结构
   - 语法简洁直观
   - 内存管理由 Python 解释器优化

**权衡**：

| 优势 | 劣势 |
|------|------|
| 访问速度快 | 内存开销较大（每个字典有额外开销） |
| 实现简单 | 缓存局部性差（链表式结构） |
| 高度灵活 | 数值算法需要转换为矩阵 |
| 动态性好 | 不适合大规模稠密图 |

### 4.2 边属性字典共享引用

**设计决策**：无向图中 `G._adj[u][v]` 和 `G._adj[v][u]` 指向同一字典对象。

**优势**：

1. **内存节省**：每条边只存储一份属性字典
2. **一致性保证**：修改任一端的属性都会反映到另一端
3. **操作简化**：不需要同时更新两个字典

**潜在风险**：

```python
G = nx.Graph()
G.add_edge(1, 2, weight=3.0)

# 获取边属性
attrs = G[1][2]

# 直接修改字典
attrs['weight'] = 5.0  # 这会修改图中的边属性

# 验证
print(G[2][1]['weight'])  # 5.0（已被修改）
```

这种行为是有意设计的——允许通过视图修改属性，但用户需要知道他们正在操作的是实际的图数据。

### 4.3 惰性视图 vs  eager 容器

**选择惰性视图的原因**：

1. **内存效率**：
   - 大型图中，创建边列表需要 O(m) 内存
   - 视图只需要 O(1) 内存（仅保存引用）

2. **实时性**：
   - 视图反映图的当前状态
   - 不需要在图修改后重新生成容器

3. **链式操作**：
   ```python
   # 不需要中间容器
   for u, v in G.edges:
       if G.edges[u, v].get('weight', 1) > threshold:
           process(u, v)
   ```

**权衡**：

| 惰性视图 | Eager 容器 |
|----------|------------|
| 内存 O(1) | 内存 O(n) 或 O(m) |
| 实时更新 | 快照式（需要手动更新） |
| 迭代时图不能修改 | 可安全修改图 |
| 不支持索引切片 | 支持 `list(G.edges)[0]` |

**推荐实践**：
```python
# 需要修改图时，先具体化
edges_to_remove = list(G.edges)[:10]
for u, v in edges_to_remove:
    G.remove_edge(u, v)

# 简单迭代时，直接使用视图
for n in G.nodes:
    process(n)
```

### 4.4 缓存策略

NetworkX 使用了两级缓存策略：

1. **`cached_property` 缓存视图对象**：
   - 视图对象本身轻量，但创建仍有开销
   - 缓存避免重复创建相同的视图

2. **显式失效机制**：
   - 数据描述符在 `_adj` 等被重新赋值时失效
   - `nx._clear_cache()` 在图修改时调用

**为什么不使用弱引用或观察者模式？**

1. **简单性**：显式失效比复杂的通知机制更容易实现和调试
2. **控制粒度**：可以精确控制何时失效（不是每次字典操作都失效）
3. **性能**：避免了每次修改时的通知开销

---

## 5. 代码示例与验证

### 5.1 验证存储结构

```python
import networkx as nx

def test_storage_structures():
    """验证四种图类型的内部存储结构"""
    
    print("=" * 60)
    print("1. Graph（无向图）存储验证")
    print("=" * 60)
    
    G = nx.Graph()
    G.add_edge(1, 2, weight=3.0)
    G.add_edge(2, 3, weight=5.0)
    
    # 验证边属性字典共享
    print(f"G._adj[1][2] is G._adj[2][1]: {G._adj[1][2] is G._adj[2][1]}")  # True
    
    # 修改验证
    G._adj[1][2]['weight'] = 10.0
    print(f"G._adj[2][1]['weight'] after modification: {G._adj[2][1]['weight']}")  # 10.0
    
    print("\n" + "=" * 60)
    print("2. DiGraph（有向图）存储验证")
    print("=" * 60)
    
    DG = nx.DiGraph()
    DG.add_edge(1, 2, weight=3.0)
    DG.add_edge(2, 1, weight=5.0)
    
    # 验证 _succ 和 _pred 的关系
    print(f"DG._adj is DG._succ: {DG._adj is DG._succ}")  # True
    
    # 验证边属性共享
    print(f"DG._succ[1][2] is DG._pred[2][1]: {DG._succ[1][2] is DG._pred[2][1]}")  # True
    
    # 验证两条边独立
    print(f"DG._succ[1][2] is DG._succ[2][1]: {DG._succ[1][2] is DG._succ[2][1]}")  # False
    
    print("\n" + "=" * 60)
    print("3. MultiGraph（多重无向图）存储验证")
    print("=" * 60)
    
    MG = nx.MultiGraph()
    key1 = MG.add_edge(1, 2, weight=3.0)
    key2 = MG.add_edge(1, 2, weight=5.0)
    
    print(f"Edge keys: {key1}, {key2}")  # 0, 1
    
    # 验证边键字典共享
    print(f"MG._adj[1][2] is MG._adj[2][1]: {MG._adj[1][2] is MG._adj[2][1]}")  # True
    
    # 验证各边独立
    print(f"MG._adj[1][2][0] is MG._adj[1][2][1]: {MG._adj[1][2][0] is MG._adj[1][2][1]}")  # False
    
    print("\n" + "=" * 60)
    print("4. MultiDiGraph（多重有向图）存储验证")
    print("=" * 60)
    
    MDG = nx.MultiDiGraph()
    MDG.add_edge(1, 2, weight=3.0)
    MDG.add_edge(1, 2, weight=5.0)
    
    # 验证边键字典共享
    print(f"MDG._succ[1][2] is MDG._pred[2][1]: {MDG._succ[1][2] is MDG._pred[2][1]}")  # True

if __name__ == "__main__":
    test_storage_structures()
```

### 5.2 验证视图惰性

```python
import networkx as nx

def test_view_laziness():
    """验证视图的惰性和实时更新特性"""
    
    print("=" * 60)
    print("验证视图的实时更新特性")
    print("=" * 60)
    
    G = nx.Graph()
    G.add_edge(1, 2)
    G.add_edge(2, 3)
    
    # 获取视图
    nodes_view = G.nodes
    edges_view = G.edges
    adj_view = G.adj
    
    print(f"初始状态:")
    print(f"  nodes: {list(nodes_view)}")  # [1, 2, 3]
    print(f"  edges: {list(edges_view)}")  # [(1, 2), (2, 3)]
    
    # 修改图
    G.add_node(4)
    G.add_edge(3, 4)
    
    print(f"\n添加节点 4 和边 (3,4) 后:")
    print(f"  nodes: {list(nodes_view)}")  # [1, 2, 3, 4]（自动更新）
    print(f"  edges: {list(edges_view)}")  # [(1, 2), (2, 3), (3, 4)]（自动更新）
    
    print("\n" + "=" * 60)
    print("验证视图不复制数据")
    print("=" * 60)
    
    # 修改节点属性
    G.nodes[1]['color'] = 'red'
    
    # 通过不同方式访问
    print(f"G.nodes[1]: {G.nodes[1]}")           # {'color': 'red'}
    print(f"G._node[1]: {G._node[1]}")             # {'color': 'red'}
    print(f"Same object? {G.nodes[1] is G._node[1]}")  # True
    
    print("\n" + "=" * 60)
    print("验证度的惰性计算")
    print("=" * 60)
    
    degree_view = G.degree
    
    print(f"初始度数:")
    print(f"  degree(1): {degree_view[1]}")  # 1
    
    G.add_edge(1, 4)
    
    print(f"\n添加边 (1,4) 后:")
    print(f"  degree(1): {degree_view[1]}")  # 2（实时计算）

if __name__ == "__main__":
    test_view_laziness()
```

---

## 6. 总结

### 6.1 核心设计要点

NetworkX 的图存储设计体现了以下核心思想：

1. **嵌套字典邻接表**：
   - 简单直观的数据结构
   - 灵活的节点和边属性支持
   - O(1) 平均时间复杂度的访问

2. **共享引用优化**：
   - 无向图边属性双向共享
   - 有向图 `_succ` 和 `_pred` 共享边属性
   - 减少内存占用，保证数据一致性

3. **多层视图抽象**：
   - 核心视图（`AtlasView`, `AdjacencyView`）提供底层访问
   - 高级视图（`NodeView`, `EdgeView`, `DegreeView`）提供用户友好接口
   - 过滤视图（`Filter*`）支持子图操作

4. **智能缓存策略**：
   - `cached_property` 缓存视图对象
   - 数据描述符实现自动失效
   - 显式缓存清理确保一致性

### 6.2 设计权衡

| 决策 | 优势 | 代价 |
|------|------|------|
| 嵌套字典 | 简单、灵活、快速 | 内存开销大、缓存局部性差 |
| 共享引用 | 内存节省、一致性 | 意外修改风险 |
| 惰性视图 | 内存高效、实时更新 | 迭代时不能修改图 |
| 缓存机制 | 减少重复创建 | 实现复杂度增加 |

### 6.3 适用场景

NetworkX 的存储设计最适合以下场景：

1. **中小型图分析**：节点数在数千到数十万级别
2. **需要频繁修改的图**：动态添加/删除节点和边
3. **丰富的属性存储**：节点和边需要携带大量元数据
4. **交互式探索**：在 Python REPL 或 Jupyter notebook 中进行图分析

对于超大规模图（百万级节点以上）或需要数值计算的场景，可能需要考虑：
- 使用 `networkx.convert_matrix` 转换为 NumPy/SciPy 矩阵
- 考虑专用图库（如 Graph-tool、igraph）
- 使用 NetworkX 的后端机制集成其他存储引擎

---

## 附录：类继承关系

### A.1 图类继承关系

```
object
  │
  └── Graph
        │
        ├── DiGraph
        │     │
        │     └── MultiDiGraph
        │
        └── MultiGraph
              │
              └── MultiDiGraph (多重继承)
```

### A.2 核心视图类继承关系

```
Mapping (collections.abc)
  │
  ├── AtlasView
  │     │
  │     └── AdjacencyView
  │           │
  │           └── MultiAdjacencyView
  │
  ├── UnionAtlas
  │
  ├── UnionAdjacency
  │
  ├── UnionMultiInner (──→ UnionAtlas)
  │
  ├── UnionMultiAdjacency (──→ UnionAdjacency)
  │
  ├── FilterAtlas
  │
  └── FilterAdjacency
        │
        ├── FilterMultiInner
        │
        └── FilterMultiAdjacency
```

### A.3 高级视图类继承关系

```
# NodeView
Mapping, Set ──→ NodeView

# EdgeView
Set, Mapping, EdgeViewABC ──→ OutEdgeView
                                      │
                                      ├── EdgeView
                                      │
                                      ├── InEdgeView
                                      │
                                      └── OutMultiEdgeView
                                                │
                                                ├── MultiEdgeView
                                                │
                                                └── InMultiEdgeView

# DegreeView
DiDegreeView
      │
      ├── DegreeView
      ├── OutDegreeView
      ├── InDegreeView
      └── MultiDegreeView
            │
            ├── DiMultiDegreeView
            ├── OutMultiDegreeView
            └── InMultiDegreeView
```

---

## 参考文献

1. NetworkX 源代码：
   - `networkx/classes/graph.py` - 无向图实现
   - `networkx/classes/digraph.py` - 有向图实现
   - `networkx/classes/multigraph.py` - 多重无向图实现
   - `networkx/classes/multidigraph.py` - 多重有向图实现
   - `networkx/classes/coreviews.py` - 核心视图实现
   - `networkx/classes/reportviews.py` - 高级视图实现

2. Python 文档：
   - `collections.abc.Mapping` - 映射抽象基类
   - `collections.abc.Set` - 集合抽象基类
   - `functools.cached_property` - 缓存属性装饰器

3. NetworkX 官方文档：
   - https://networkx.org/documentation/stable/reference/classes/index.html
