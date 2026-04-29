# NetworkX 图生成器设计模式与格式转换机制分析

## 一、图生成器设计模式分析

### 1.1 核心设计模式

NetworkX 的图生成器采用了**工厂方法模式**与**策略模式**相结合的设计：

- **工厂方法模式**：通过统一的函数接口创建不同拓扑结构的图
- **策略模式**：通过 `create_using` 参数允许用户灵活指定返回的图类型

### 1.2 统一接口机制

所有图生成器函数都遵循相同的接口模式，核心是 `create_using` 参数和 `empty_graph` 基础函数。

#### `create_using` 参数设计

`create_using` 参数支持三种形式：

| 形式 | 说明 | 示例 |
|------|------|------|
| `None` | 使用默认图类型（通常是 `Graph`） | `nx.complete_graph(5)` |
| 图类 | 创建该类型的新图 | `nx.complete_graph(5, nx.DiGraph)` |
| 图实例 | 清空并重用该实例 | `nx.complete_graph(5, my_graph)` |

#### `empty_graph` 基础函数

`empty_graph` 是所有图生成器的基础构造函数，位于 `networkx/generators/classic.py:586`：

```python
def empty_graph(n=0, create_using=None, default=Graph):
    if create_using is None:
        G = default()
    elif isinstance(create_using, type):
        G = create_using()
    elif not hasattr(create_using, "adj"):
        raise TypeError("create_using is not a valid NetworkX graph type or instance")
    else:
        create_using.clear()
        G = create_using
    
    _, nodes = n
    G.add_nodes_from(nodes)
    return G
```

#### `check_create_using` 验证函数

`check_create_using` 函数用于验证 `create_using` 参数的有效性，位于 `networkx/utils/misc.py:667`：

```python
def check_create_using(create_using, *, directed=None, multigraph=None, default=None):
    if default is None:
        default = nx.Graph
    G = create_using if create_using is not None else default

    G_directed = G.is_directed(None) if isinstance(G, type) else G.is_directed()
    G_multigraph = G.is_multigraph(None) if isinstance(G, type) else G.is_multigraph()

    if directed is not None:
        if directed and not G_directed:
            raise nx.NetworkXError("create_using must be directed")
        if not directed and G_directed:
            raise nx.NetworkXError("create_using must not be directed")

    if multigraph is not None:
        if multigraph and not G_multigraph:
            raise nx.NetworkXError("create_using must be a multi-graph")
        if not multigraph and G_multigraph:
            raise nx.NetworkXError("create_using must not be a multi-graph")
    return G
```

### 1.3 典型图生成器实现分析

#### 完全图生成器 (`complete_graph`)

完全图生成器展示了有向图与无向图的核心差异处理，位于 `networkx/generators/classic.py:316`：

```python
def complete_graph(n, create_using=None):
    _, nodes = n
    G = empty_graph(nodes, create_using)
    if len(nodes) > 1:
        if G.is_directed():
            edges = itertools.permutations(nodes, 2)  # 有向图：排列，n*(n-1) 条边
        else:
            edges = itertools.combinations(nodes, 2)  # 无向图：组合，n*(n-1)/2 条边
        G.add_edges_from(edges)
    return G
```

**关键差异**：
- **有向图**：使用 `itertools.permutations`，每个节点对生成两条有向边（u→v 和 v→u）
- **无向图**：使用 `itertools.combinations`，每个节点对只生成一条无向边

#### 网格图生成器 (`grid_2d_graph`)

网格图生成器展示了周期性边界和有向图的特殊处理，位于 `networkx/generators/lattice.py:36`：

```python
def grid_2d_graph(m, n, periodic=False, create_using=None):
    G = empty_graph(0, create_using)
    row_name, rows = m
    col_name, cols = n
    G.add_nodes_from((i, j) for i in rows for j in cols)
    G.add_edges_from(((i, j), (pi, j)) for pi, i in pairwise(rows) for j in cols)
    G.add_edges_from(((i, j), (i, pj)) for i in rows for pj, j in pairwise(cols))

    # 周期性边界处理
    try:
        periodic_r, periodic_c = periodic
    except TypeError:
        periodic_r = periodic_c = periodic

    if periodic_r and len(rows) > 2:
        first = rows[0]
        last = rows[-1]
        G.add_edges_from(((first, j), (last, j)) for j in cols)
    if periodic_c and len(cols) > 2:
        first = cols[0]
        last = cols[-1]
        G.add_edges_from(((i, first), (i, last)) for i in rows)
    
    # 有向图需要添加反向边
    if G.is_directed():
        G.add_edges_from((v, u) for u, v in G.edges())
    return G
```

**关键设计**：
- 首先创建单向边（行连接和列连接）
- 对于有向图，显式添加反向边以形成对称的网格结构
- 支持周期性边界条件

#### 随机图生成器 (`fast_gnp_random_graph`)

随机图生成器展示了参数验证和图类型推断，位于 `networkx/generators/random_graphs.py:43`：

```python
def fast_gnp_random_graph(n, p, seed=None, directed=False, *, create_using=None):
    # 根据 directed 参数确定默认图类型
    default = nx.DiGraph if directed else nx.Graph
    
    # 验证 create_using 参数与 directed 参数的一致性
    create_using = check_create_using(
        create_using, directed=directed, multigraph=False, default=default
    )
    
    if p <= 0 or p >= 1:
        return nx.gnp_random_graph(
            n, p, seed=seed, directed=directed, create_using=create_using
        )

    G = empty_graph(n, create_using=create_using)
    lp = math.log(1.0 - p)

    # 有向图的边生成逻辑
    if directed:
        v = 1
        w = -1
        while v < n:
            lr = math.log(1.0 - seed.random())
            w = w + 1 + int(lr / lp)
            while w >= v and v < n:
                w = w - v
                v = v + 1
            if v < n:
                G.add_edge(w, v)

    # 无向图的边生成逻辑
    v = 1
    w = -1
    while v < n:
        lr = math.log(1.0 - seed.random())
        w = w + 1 + int(lr / lp)
        while w >= v and v < n:
            w = w - v
            v = v + 1
        if v < n:
            G.add_edge(v, w)
    return G
```

**关键设计**：
- 使用 `directed` 参数控制图类型
- 使用 `check_create_using` 确保参数一致性
- 有向图和无向图使用不同的边索引计算逻辑

### 1.4 图生成器分类

NetworkX 的图生成器按拓扑结构分类如下：

| 类别 | 模块文件 | 典型生成器 | 说明 |
|------|----------|------------|------|
| 经典图 | `classic.py` | `complete_graph`, `cycle_graph`, `path_graph`, `star_graph`, `wheel_graph` | 基础拓扑结构 |
| 随机图 | `random_graphs.py` | `gnp_random_graph`, `barabasi_albert_graph`, `watts_strogatz_graph` | 概率生成模型 |
| 网格/格点图 | `lattice.py` | `grid_2d_graph`, `grid_graph`, `triangular_lattice_graph`, `hexagonal_lattice_graph` | 规则网格结构 |
| 树图 | `trees.py` | `random_tree`, `prefix_tree` | 树状结构 |
| 社区图 | `community.py` | `stochastic_block_model`, `planted_partition_graph` | 社区结构 |
| 几何图 | `geometric.py` | `random_geometric_graph`, `soft_random_geometric_graph` | 几何空间中的图 |

---

## 二、图类结构分析

### 2.1 继承层次结构

NetworkX 提供了四种核心图类，形成清晰的继承层次：

```
┌─────────────────────────────────────────────────────────────┐
│                         Graph (基类)                          │
│                    无向图，单条边                              │
├─────────────────────────┬───────────────────────────────────┤
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                       │
│              │      DiGraph        │                       │
│              │   有向图，单条边     │                       │
│              └─────────────────────┘                       │
│                         │                                   │
▼                         ▼                                   ▼
┌─────────────────────────────────────────────────────────────┐
│                      MultiGraph                              │
│                  无向图，多条边                               │
├─────────────────────────┬───────────────────────────────────┤
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                       │
│              │    MultiDiGraph     │                       │
│              │   有向图，多条边     │                       │
│              │  (多重继承)          │                       │
│              └─────────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心数据结构

#### Graph 类（无向图）

`Graph` 类使用邻接表存储结构，位于 `networkx/classes/graph.py`：

```python
class Graph:
    """Base class for undirected graphs."""
    
    # 数据描述符：用于重置缓存属性
    adjacency_attr = _CachedPropertyResetterAdj()
    node_attr = _CachedPropertyResetterNode()
    
    def __init__(self, incoming_graph_data=None, **attr):
        self.graph = {}  # 图级属性
        self._node = {}   # 节点属性字典
        self._adj = {}    # 邻接表
    
    def is_directed(self):
        return False
    
    def is_multigraph(self):
        return False
```

**邻接表结构**：
- `_adj[u][v]` 存储边 (u, v) 的属性字典
- 对于无向图，`_adj[u][v]` 和 `_adj[v][u]` 指向同一个字典对象

#### DiGraph 类（有向图）

`DiGraph` 类继承自 `Graph`，使用分离的后继和前驱表，位于 `networkx/classes/digraph.py`：

```python
class DiGraph(Graph):
    """Base class for directed graphs."""
    
    # 数据描述符
    adjacency_attr = _CachedPropertyResetterAdjAndSucc()
    predecessor_attr = _CachedPropertyResetterPred()
    
    def __init__(self, incoming_graph_data=None, **attr):
        self.graph = {}
        self._node = {}
        self._adj = {}    # 别名：_succ，后继表
        self._pred = {}   # 前驱表
    
    def is_directed(self):
        return True
```

**关键数据结构**：
- `_succ[u][v]`：存储从 u 到 v 的出边属性
- `_pred[v][u]`：存储从 u 到 v 的入边属性
- `_adj` 是 `_succ` 的别名，保持与 Graph 类的接口一致性

#### MultiGraph 类（多重无向图）

`MultiGraph` 类继承自 `Graph`，支持同节点对之间的多条边，位于 `networkx/classes/multigraph.py`：

```python
class MultiGraph(Graph):
    """An undirected graph class that can store multiedges."""
    
    def is_multigraph(self):
        return True
```

**多重边存储**：
- `_adj[u][v][key]`：通过 key 区分同节点对的不同边
- 每条边有唯一的整数 key

#### MultiDiGraph 类（多重有向图）

`MultiDiGraph` 类使用多重继承，同时继承 `MultiGraph` 和 `DiGraph`，位于 `networkx/classes/multidigraph.py`：

```python
class MultiDiGraph(MultiGraph, DiGraph):
    """A directed graph class that can store multiedges."""
```

**方法解析顺序 (MRO)**：
- `MultiDiGraph` → `MultiGraph` → `DiGraph` → `Graph`
- `is_directed()` 返回 `True`（来自 `DiGraph`）
- `is_multigraph()` 返回 `True`（来自 `MultiGraph`）

### 2.3 类型判断方法

所有图类都提供了统一的类型判断接口：

| 方法 | Graph | DiGraph | MultiGraph | MultiDiGraph |
|------|-------|---------|------------|--------------|
| `is_directed()` | `False` | `True` | `False` | `True` |
| `is_multigraph()` | `False` | `False` | `True` | `True` |

这些方法在图生成器和格式转换中被广泛使用，用于根据图类型执行不同的逻辑。

---

## 三、格式转换机制分析

### 3.1 统一转换入口

NetworkX 提供了 `to_networkx_graph` 函数作为统一的转换入口，位于 `networkx/convert.py:34`：

```python
def to_networkx_graph(data, create_using=None, multigraph_input=False):
    """Make a NetworkX graph from a known data structure."""
    
    # 1. NetworkX 图对象
    if hasattr(data, "adj"):
        try:
            result = from_dict_of_dicts(
                data.adj,
                create_using=create_using,
                multigraph_input=data.is_multigraph(),
            )
            # 复制图属性和节点属性
            result.graph.update(data.graph)
            for n, dd in data.nodes.items():
                result._node[n].update(dd)
            return result
        except Exception as err:
            raise nx.NetworkXError("Input is not a correct NetworkX graph.") from err

    # 2. 字典格式（dict-of-dicts 或 dict-of-lists）
    if isinstance(data, dict):
        try:
            return from_dict_of_dicts(
                data, create_using=create_using, multigraph_input=multigraph_input
            )
        except Exception as err1:
            if multigraph_input is True:
                raise nx.NetworkXError(...)
            try:
                return from_dict_of_lists(data, create_using=create_using)
            except Exception as err2:
                raise TypeError("Input is not known type.") from err2

    # 3. 边列表
    if isinstance(data, list | tuple | nx.reportviews.EdgeViewABC | Iterator):
        try:
            return from_edgelist(data, create_using=create_using)
        except:
            pass

    # 4. NumPy 数组
    try:
        import numpy as np
        if isinstance(data, np.ndarray):
            return nx.from_numpy_array(data, create_using=create_using)
    except ImportError:
        pass

    # 5. SciPy 稀疏数组
    try:
        import scipy as sp
        if hasattr(data, "format"):
            return nx.from_scipy_sparse_array(data, create_using=create_using)
    except ImportError:
        pass

    # ... 其他格式
```

**支持的输入格式**：
1. NetworkX 图对象
2. 字典格式（dict-of-dicts、dict-of-lists）
3. 边列表（list、tuple、iterator）
4. NumPy 数组（邻接矩阵）
5. SciPy 稀疏数组
6. Pandas DataFrame
7. PyGraphviz 图

### 3.2 字典格式转换

#### 图转字典 (`to_dict_of_dicts`)

将图转换为邻接字典表示，位于 `networkx/convert.py:253`：

```python
def to_dict_of_dicts(G, nodelist=None, edge_data=None):
    """Returns adjacency representation of graph as a dictionary of dictionaries."""
    
    if nodelist is None:
        nodelist = G
    
    dod = {}
    for n in nodelist:
        dod[n] = {}
        for nbr, dd in G.adj[n].items():
            if nodelist is not None and nbr not in nodelist:
                continue
            if edge_data is None:
                dod[n][nbr] = dd
            else:
                dod[n][nbr] = edge_data
    return dod
```

**输出格式**：
- **普通图**：`{node: {neighbor: edge_attr_dict, ...}, ...}`
- **多重图**：`{node: {neighbor: {key: edge_attr_dict, ...}, ...}, ...}`

**有向图与无向图的差异**：
- **无向图**：每条边在两个方向都出现（u 在 v 的邻接表中，v 也在 u 的邻接表中）
- **有向图**：只有出边出现在邻接表中（u→v 只在 u 的邻接表中）

#### 字典转图 (`from_dict_of_dicts`)

从邻接字典创建图，位于 `networkx/convert.py:374`：

```python
def from_dict_of_dicts(d, create_using=None, multigraph_input=False):
    G = nx.empty_graph(0, create_using)
    G.add_nodes_from(d)
    
    if multigraph_input:
        # 多重图输入格式
        if G.is_directed():
            if G.is_multigraph():
                # 有向多重图
                G.add_edges_from(
                    (u, v, key, data)
                    for u, nbrs in d.items()
                    for v, datadict in nbrs.items()
                    for key, data in datadict.items()
                )
            else:
                # 有向图（非多重）
                G.add_edges_from(
                    (u, v, data)
                    for u, nbrs in d.items()
                    for v, datadict in nbrs.items()
                    for key, data in datadict.items()
                )
        else:  # Undirected
            if G.is_multigraph():
                # 无向多重图：需要避免重复添加
                seen = set()
                for u, nbrs in d.items():
                    for v, datadict in nbrs.items():
                        for key, data in datadict.items():
                            if (v, u, key) not in seen:
                                seen.add((u, v, key))
                                G.add_edge(u, v, key=key, **data)
            else:
                # 无向图（非多重）：需要避免重复添加
                seen = set()
                for u, nbrs in d.items():
                    for v, datadict in nbrs.items():
                        if (v, u) not in seen:
                            seen.add((u, v))
                            G.add_edge(u, v, **datadict)
    else:
        # 普通图输入格式
        if G.is_directed():
            # 有向图：直接添加所有边
            G.add_edges_from(
                (u, v, data)
                for u, nbrs in d.items()
                for v, data in nbrs.items()
            )
        else:
            # 无向图：需要避免重复添加
            seen = set()
            for u, nbrs in d.items():
                for v, data in nbrs.items():
                    if (v, u) not in seen:
                        seen.add((u, v))
                        G.add_edge(u, v, **data)
    return G
```

**关键差异处理**：

| 图类型 | 处理方式 | 原因 |
|--------|----------|------|
| 有向图 | 直接添加所有边 | 每条边都是单向的，u→v 和 v→u 是独立的 |
| 无向图 | 使用 `seen` 集合避免重复 | 无向边在字典中可能出现两次（u→v 和 v→u） |

### 3.3 矩阵格式转换

#### 图转 NumPy 数组 (`to_numpy_array`)

将图转换为邻接矩阵，位于 `networkx/convert_matrix.py:893`：

```python
def to_numpy_array(
    G,
    nodelist=None,
    dtype=None,
    order=None,
    multigraph_weight=sum,
    weight="weight",
    nonedge=0.0,
):
    """Returns the graph adjacency matrix as a NumPy array."""
    
    if nodelist is None:
        nodelist = list(G)
    
    nlen = len(nodelist)
    index = dict(zip(nodelist, range(nlen)))
    
    # 初始化矩阵
    A = np.full((nlen, nlen), nonedge, dtype=dtype, order=order)
    
    # 填充矩阵
    for u, v, data in G.edges(data=True):
        if u in index and v in index:
            # 获取边权重
            if weight is None:
                value = data
            else:
                value = data.get(weight, 1)
            
            i, j = index[u], index[v]
            
            if G.is_multigraph() and not G.is_directed():
                # 无向多重图：需要特殊处理
                if A[i, j] == nonedge:
                    A[i, j] = value
                    if i != j:  # 非自环
                        A[j, i] = value
                else:
                    # 多重边：使用 multigraph_weight 合并
                    A[i, j] = multigraph_weight([A[i, j], value])
                    if i != j:
                        A[j, i] = multigraph_weight([A[j, i], value])
            elif G.is_directed():
                # 有向图：只设置 A[i,j]
                if G.is_multigraph():
                    if A[i, j] == nonedge:
                        A[i, j] = value
                    else:
                        A[i, j] = multigraph_weight([A[i, j], value])
                else:
                    A[i, j] = value
            else:
                # 无向图（非多重）：设置对称位置
                A[i, j] = value
                if i != j:
                    A[j, i] = value
    return A
```

**矩阵元素含义**：
- **有向图**：`A[i, j]` 表示从节点 `i` 到节点 `j` 的边权重
- **无向图**：`A[i, j] = A[j, i]`，表示节点 `i` 和 `j` 之间的边权重
- **自环**：`A[i, i]` 表示节点 `i` 的自环权重

#### NumPy 数组转图 (`from_numpy_array`)

从邻接矩阵创建图，位于 `networkx/convert_matrix.py:714`：

```python
def from_numpy_array(
    A,
    parallel_edges=False,
    create_using=None,
    edge_attr="weight",
    *,
    nonedge=0,
):
    """Returns a graph from NumPy array."""
    
    G = nx.empty_graph(0, create_using)
    
    # 获取节点数量
    n = A.shape[0]
    
    # 添加节点
    G.add_nodes_from(range(n))
    
    # 根据图类型处理边
    if G.is_directed():
        # 有向图：遍历所有非零元素
        for i, j in zip(*np.where(A != nonedge)):
            if i == j:
                # 自环
                G.add_edge(i, j, **{edge_attr: A[i, j]})
            else:
                G.add_edge(i, j, **{edge_attr: A[i, j]})
    else:
        # 无向图：只遍历上三角（包括对角线）
        for i, j in zip(*np.where(np.triu(A) != nonedge)):
            G.add_edge(i, j, **{edge_attr: A[i, j]})
    
    return G
```

**关键差异**：
- **有向图**：遍历整个矩阵，`A[i, j]` 表示 i→j 的边
- **无向图**：只遍历上三角矩阵，避免重复添加边

### 3.4 稀疏矩阵格式转换

#### 图转 SciPy 稀疏数组 (`to_scipy_sparse_array`)

将图转换为 SciPy 稀疏数组，位于 `networkx/convert_matrix.py:496`：

```python
def to_scipy_sparse_array(G, nodelist=None, dtype=None, weight="weight", format="csr"):
    """Returns the graph adjacency matrix as a SciPy sparse array."""
    
    if nodelist is None:
        nodelist = list(G)
    
    nlen = len(nodelist)
    index = dict(zip(nodelist, range(nlen)))
    
    # 构建 COO 格式的三元组
    row, col, data = [], [], []
    
    for u, v, edge_data in G.edges(data=True):
        if u in index and v in index:
            i, j = index[u], index[v]
            
            # 获取权重
            if weight is None:
                value = 1
            else:
                value = edge_data.get(weight, 1)
            
            # 处理有向图和无向图
            if G.is_directed():
                # 有向图：只添加 (i, j)
                row.append(i)
                col.append(j)
                data.append(value)
            else:
                # 无向图：添加对称位置
                row.append(i)
                col.append(j)
                data.append(value)
                if i != j:  # 非自环
                    row.append(j)
                    col.append(i)
                    data.append(value)
    
    # 创建稀疏数组并转换为指定格式
    import scipy.sparse as sp
    A = sp.coo_array((data, (row, col)), shape=(nlen, nlen), dtype=dtype)
    
    if format == "coo":
        return A
    return A.asformat(format)
```

**关键差异**：
- **有向图**：每条边只存储一个条目 (i, j)
- **无向图**：每条边存储两个对称条目 (i, j) 和 (j, i)

### 3.5 转换机制总结

| 转换方向 | 函数 | 有向图处理 | 无向图处理 |
|----------|------|-----------|-----------|
| 图 → 字典 | `to_dict_of_dicts` | 只存储出边 | 存储双向引用 |
| 字典 → 图 | `from_dict_of_dicts` | 直接添加所有边 | 使用 `seen` 集合去重 |
| 图 → NumPy 数组 | `to_numpy_array` | `A[i,j]` 表示 i→j | 对称矩阵 `A[i,j] = A[j,i]` |
| NumPy 数组 → 图 | `from_numpy_array` | 遍历整个矩阵 | 只遍历上三角 |
| 图 → 稀疏数组 | `to_scipy_sparse_array` | 单一条目 (i, j) | 对称条目 (i, j) 和 (j, i) |

---

## 四、GraphML 格式转换分析

### 4.1 GraphML 格式简介

GraphML 是一种基于 XML 的图数据格式，支持：
- 有向图、无向图、混合图
- 节点和边的属性
- 嵌套图（子图）
- 超边（NetworkX 不支持）

### 4.2 写操作分析

#### 写入流程

GraphML 写入的核心是 `GraphMLWriter` 类，位于 `networkx/readwrite/graphml.py`：

```python
class GraphMLWriter:
    """Writer for GraphML files."""
    
    def add_graph_element(self, G):
        """Add a graph element to the XML."""
        
        # 1. 确定默认边类型
        if G.is_directed():
            default_edge_type = "directed"
        else:
            default_edge_type = "undirected"
        
        # 2. 创建 graph 元素
        graphid = G.graph.pop("id", None)
        if graphid is None:
            graph_element = self.myElement("graph", edgedefault=default_edge_type)
        else:
            graph_element = self.myElement(
                "graph", edgedefault=default_edge_type, id=graphid
            )
        
        # 3. 添加图属性
        data = {
            k: v
            for (k, v) in G.graph.items()
            if k not in ["node_default", "edge_default"]
        }
        self.add_attributes("graph", graph_element, data, default)
        
        # 4. 添加节点和边
        self.add_nodes(G, graph_element)
        self.add_edges(G, graph_element)
```

#### 边的写入

```python
def add_edges(self, G, graph_element):
    """Add edges to the graph element."""
    
    # 根据图类型处理边
    if G.is_directed():
        # 有向图：遍历所有出边
        edges = G.edges(data=True, keys=True) if G.is_multigraph() else G.edges(data=True)
        for edge in edges:
            # 添加边元素，不指定 directed 属性（使用默认值）
            self.add_edge(G, edge, graph_element)
    else:
        # 无向图：遍历所有边
        edges = G.edges(data=True, keys=True) if G.is_multigraph() else G.edges(data=True)
        for edge in edges:
            # 添加边元素
            self.add_edge(G, edge, graph_element)
```

#### 输出的 XML 结构

**无向图**：
```xml
<graph edgedefault="undirected">
  <node id="0"/>
  <node id="1"/>
  <edge source="0" target="1"/>
</graph>
```

**有向图**：
```xml
<graph edgedefault="directed">
  <node id="0"/>
  <node id="1"/>
  <edge source="0" target="1"/>
</graph>
```

### 4.3 读操作分析

#### 读取流程

GraphML 读取的核心是 `GraphMLReader` 类：

```python
class GraphMLReader:
    """Reader for GraphML files."""
    
    def make_graph(self, graph_xml, graphml_keys, defaults, G=None):
        """Create a graph from the XML element."""
        
        # 1. 获取默认边类型
        edgedefault = graph_xml.get("edgedefault", "undirected")
        
        # 2. 创建图对象
        if G is None:
            if edgedefault == "directed":
                G = nx.DiGraph()
            else:
                G = nx.Graph()
        
        # 3. 处理节点和边
        for node_xml in graph_xml.findall(f"{{{self.NS_GRAPHML}}}node"):
            self.add_node(G, node_xml, graphml_keys, defaults)
        
        for edge_xml in graph_xml.findall(f"{{{self.NS_GRAPHML}}}edge"):
            self.add_edge(G, edge_xml, graphml_keys)
        
        # 4. 最终确定图类型
        G = nx.DiGraph(G) if G.is_directed() else nx.Graph(G)
        return G
```

#### 边类型验证

`add_edge` 方法会验证边的 `directed` 属性与图类型的一致性：

```python
def add_edge(self, G, edge_element, graphml_keys):
    """Add an edge to the graph."""
    
    # 获取边的 directed 属性
    directed = edge_element.get("directed")
    
    # 验证一致性
    if G.is_directed() and directed == "false":
        msg = "directed=false edge found in directed graph."
        raise nx.NetworkXError(msg)
    
    if (not G.is_directed()) and directed == "true":
        msg = "directed=true edge found in undirected graph."
        raise nx.NetworkXError(msg)
    
    # 添加边
    source = self.node_type(edge_element.get("source"))
    target = self.node_type(edge_element.get("target"))
    data = self.decode_data_elements(graphml_keys, edge_element)
    
    G.add_edge(source, target, **data)
```

### 4.4 有向图与无向图的差异处理

#### 边类型确定规则

GraphML 中边的类型由以下规则确定（优先级从高到低）：

1. **边的 `directed` 属性**：如果边元素显式指定了 `directed="true"` 或 `directed="false"`
2. **图的 `edgedefault` 属性**：如果边没有显式指定，使用图级别的默认值

#### NetworkX 的限制

NetworkX 不支持**混合图**（同时包含有向和无向边），因此：

| 场景 | 处理方式 |
|------|----------|
| 读取有向图时遇到 `directed="false"` 的边 | 抛出 `NetworkXError` |
| 读取无向图时遇到 `directed="true"` 的边 | 抛出 `NetworkXError` |
| 边没有 `directed` 属性 | 使用图的 `edgedefault` |

#### 写入时的处理

| 图类型 | `edgedefault` 值 | 边是否显式指定 `directed` |
|--------|------------------|---------------------------|
| 无向图 | `"undirected"` | 否（使用默认值） |
| 有向图 | `"directed"` | 否（使用默认值） |

### 4.5 示例

#### 写入示例

```python
import networkx as nx

# 无向图
G_undir = nx.Graph()
G_undir.add_edge(0, 1, weight=2.5)
nx.write_graphml(G_undir, "undirected.graphml")

# 有向图
G_dir = nx.DiGraph()
G_dir.add_edge(0, 1, weight=2.5)
nx.write_graphml(G_dir, "directed.graphml")
```

**输出的 undirected.graphml**：
```xml
<?xml version='1.0' encoding='utf-8'?>
<graphml xmlns="http://graphml.graphdrawing.org/xmlns">
  <key attr.name="weight" attr.type="double" for="edge" id="d0"/>
  <graph edgedefault="undirected">
    <node id="0"/>
    <node id="1"/>
    <edge source="0" target="1">
      <data key="d0">2.5</data>
    </edge>
  </graph>
</graphml>
```

**输出的 directed.graphml**：
```xml
<?xml version='1.0' encoding='utf-8'?>
<graphml xmlns="http://graphml.graphdrawing.org/xmlns">
  <key attr.name="weight" attr.type="double" for="edge" id="d0"/>
  <graph edgedefault="directed">
    <node id="0"/>
    <node id="1"/>
    <edge source="0" target="1">
      <data key="d0">2.5</data>
    </edge>
  </graph>
</graphml>
```

#### 读取示例

```python
# 读取并自动识别图类型
G = nx.read_graphml("directed.graphml")
print(G.is_directed())  # True

G2 = nx.read_graphml("undirected.graphml")
print(G2.is_directed())  # False
```

---

## 五、设计模式与最佳实践总结

### 5.1 图生成器设计模式总结

#### 核心模式

| 模式 | 应用场景 | 实现方式 |
|------|----------|----------|
| **工厂方法模式** | 创建不同拓扑结构的图 | 每个生成器函数是一个工厂方法 |
| **策略模式** | 灵活指定图类型 | `create_using` 参数 |
| **模板方法模式** | 统一的生成流程 | `empty_graph` + 边添加 |

#### 关键设计原则

1. **统一接口**：所有生成器都支持 `create_using` 参数
2. **类型安全**：使用 `check_create_using` 验证参数一致性
3. **灵活性**：支持图类和图实例两种形式的 `create_using`
4. **可扩展性**：用户可以通过自定义图类扩展

### 5.2 格式转换设计模式总结

#### 核心模式

| 模式 | 应用场景 | 实现方式 |
|------|----------|----------|
| **适配器模式** | 不同格式之间的转换 | `to_*` 和 `from_*` 函数 |
| **工厂方法模式** | 从数据创建图 | `to_networkx_graph` |
| **策略模式** | 根据图类型选择转换逻辑 | `is_directed()` 和 `is_multigraph()` 判断 |

#### 关键设计原则

1. **双向转换**：每种格式都有 `to_*` 和 `from_*` 函数
2. **类型感知**：转换逻辑根据图类型（有向/无向、单条/多条边）自动调整
3. **损失最小化**：尽可能保留所有图信息（属性、权重等）
4. **渐进式支持**：可选依赖（NumPy、SciPy、Pandas）不影响核心功能

### 5.3 有向图与无向图差异处理总结

#### 处理策略分类

| 场景 | 有向图处理 | 无向图处理 | 核心原因 |
|------|-----------|-----------|----------|
| **边生成** | `permutations` | `combinations` | 有向边是单向的 |
| **边存储** | `_succ` + `_pred` | 单个 `_adj` | 有向图需要区分方向 |
| **字典转图** | 直接添加 | `seen` 集合去重 | 无向边可能出现两次 |
| **矩阵转图** | 遍历整个矩阵 | 遍历上三角 | 无向矩阵是对称的 |
| **稀疏矩阵** | 单一条目 | 对称条目 | 存储效率考虑 |
| **GraphML** | `edgedefault="directed"` | `edgedefault="undirected"` | 格式规范要求 |

#### 关键 API 汇总

**图类型判断**：
- `G.is_directed()`：判断是否为有向图
- `G.is_multigraph()`：判断是否为多重图

**图生成器**：
- `create_using` 参数：指定图类型
- `empty_graph(n, create_using)`：创建空图
- `check_create_using()`：验证图类型参数

**格式转换**：
- `to_dict_of_dicts` / `from_dict_of_dicts`：字典格式
- `to_numpy_array` / `from_numpy_array`：NumPy 矩阵
- `to_scipy_sparse_array` / `from_scipy_sparse_array`：稀疏矩阵
- `write_graphml` / `read_graphml`：GraphML 格式

### 5.4 最佳实践建议

1. **选择合适的图类型**：
   - 不需要区分边方向时使用 `Graph`
   - 需要区分边方向时使用 `DiGraph`
   - 需要同节点对多条边时使用 `MultiGraph`/`MultiDiGraph`

2. **使用 `create_using` 参数**：
   - 在图生成器中显式指定图类型
   - 避免依赖默认行为（可能随版本变化）

3. **格式转换注意事项**：
   - 无向图转矩阵后是对称的
   - 有向图的邻接矩阵不一定对称
   - 多重图转换为矩阵时会合并边权重

4. **GraphML 兼容性**：
   - NetworkX 不支持混合图
   - 读取外部 GraphML 文件时注意边类型一致性
   - 写入时会自动设置正确的 `edgedefault`

---

## 六、参考代码位置

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| `empty_graph` | `networkx/generators/classic.py` | 586 |
| `check_create_using` | `networkx/utils/misc.py` | 667 |
| `complete_graph` | `networkx/generators/classic.py` | 316 |
| `grid_2d_graph` | `networkx/generators/lattice.py` | 36 |
| `fast_gnp_random_graph` | `networkx/generators/random_graphs.py` | 43 |
| `Graph` 类 | `networkx/classes/graph.py` | 70 |
| `DiGraph` 类 | `networkx/classes/digraph.py` | 87 |
| `MultiGraph` 类 | `networkx/classes/multigraph.py` | 15 |
| `MultiDiGraph` 类 | `networkx/classes/multidigraph.py` | 22 |
| `to_networkx_graph` | `networkx/convert.py` | 34 |
| `to_dict_of_dicts` | `networkx/convert.py` | 253 |
| `from_dict_of_dicts` | `networkx/convert.py` | 374 |
| `to_numpy_array` | `networkx/convert_matrix.py` | 893 |
| `to_scipy_sparse_array` | `networkx/convert_matrix.py` | 496 |
| `write_graphml` | `networkx/readwrite/graphml.py` | 62 |
| `read_graphml` | `networkx/readwrite/graphml.py` | （见 `GraphMLReader` 类） |

---

*报告生成时间：2026-04-29*

*分析基于 NetworkX 源代码版本：位于 `g:\fangzheng\solo-dogfeeding\code\networkx-8945`*