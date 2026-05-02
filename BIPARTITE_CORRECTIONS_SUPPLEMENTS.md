# NetworkX 二部图算法深度分析：修正与补充

## 目录

1. [修正：投影算法输入校验的准确描述](#1-修正投影算法输入校验的准确描述)
2. [补充：中心性算法在异常节点划分下的边界行为](#2-补充中心性算法在异常节点划分下的边界行为)
3. [补充：最小权完整匹配与普通最大匹配的对比](#3-补充最小权完整匹配与普通最大匹配的对比)
4. [总结](#4-总结)

---

## 1. 修正：投影算法输入校验的准确描述

### 1.1 之前描述的不准确点

**之前的描述（不准确）**：
> "所有投影算法都有 `len(nodes) >= len(B)` 的边界检查"

**实际情况**：只有 **`weighted_projected_graph`** 有这个检查，其他投影函数**没有**！

### 1.2 各投影函数的输入校验对比

| 投影函数 | `len(nodes) >= len(B)` 检查 | 多重图检查 | 代码位置 |
|---------|-----------------------------|------------|----------|
| `projected_graph` (无加权) | ❌ **没有** | ✅ 有 (`is_multigraph()` 检查) | `projection.py:18-118` |
| `weighted_projected_graph` | ✅ **有** | ✅ 有 (`@not_implemented_for("multigraph")`) | `projection.py:123-219` |
| `collaboration_weighted_projected_graph` | ❌ **没有** | ✅ 有 (`@not_implemented_for("multigraph")`) | `projection.py:224-313` |
| `overlap_weighted_projected_graph` | ❌ **没有** | ✅ 有 (`@not_implemented_for("multigraph")`) | `projection.py:318-413` |
| `generic_weighted_projected_graph` | ❌ **没有** | ✅ 有 (`@not_implemented_for("multigraph")`) | `projection.py:418-525` |

### 1.3 `weighted_projected_graph` 的边界检查

**关键代码位置**：`networkx/algorithms/bipartite/projection.py:200-206`

```python
n_top = len(B) - len(nodes)

if n_top < 1:
    raise nx.NetworkXAlgorithmError(
        f"the size of the nodes to project onto ({len(nodes)}) is >= the graph size ({len(B)}).\n"
        "They are either not a valid bipartite partition or contain duplicates"
    )
```

**检查逻辑**：
- `n_top < 1` 等价于 `len(B) - len(nodes) < 1`
- 即 `len(nodes) >= len(B)` 时抛出异常

**测试验证**：
**关键代码位置**：`networkx/algorithms/bipartite/tests/test_project.py:57-61`

```python
def test_path_weighted_projected_graph(self):
    G = nx.path_graph(4)

    with pytest.raises(nx.NetworkXAlgorithmError):
        bipartite.weighted_projected_graph(G, [1, 2, 3, 3])  # 包含重复节点
```

### 1.4 `projected_graph` 无边界检查的验证

**测试验证**：
**关键代码位置**：`networkx/algorithms/bipartite/tests/test_project.py:99-101`

```python
def test_star_projected_graph(self):
    G = nx.star_graph(3)
    # ...
    P = bipartite.projected_graph(G, [0])  # 投影到单个节点！
    assert nodes_equal(list(P), [0])
    assert edges_equal(list(P.edges()), [])
```

`projected_graph` 可以投影到**只有一个节点**的集合，这说明它**没有** `len(nodes) >= len(B)` 的检查。

### 1.5 为什么 `weighted_projected_graph` 需要这个检查？

看 `weighted_projected_graph` 的 `ratio` 参数：
**关键代码位置**：`networkx/algorithms/bipartite/projection.py:214-217`

```python
if not ratio:
    weight = len(common)
else:
    weight = len(common) / n_top  # n_top = len(B) - len(nodes)
```

当 `ratio=True` 时，权重计算使用 `n_top` 作为分母。如果 `n_top < 1`：
- `n_top == 0`：除零错误
- `n_top < 0`：无意义的负数

因此，**只有 `weighted_projected_graph` 需要这个检查**，因为它可能需要 `n_top` 作为分母。

---

## 2. 补充：中心性算法在异常节点划分下的边界行为

### 2.1 中心性算法的输入处理

所有三个中心性算法都有相同的输入处理模式：

**关键代码位置**：`networkx/algorithms/bipartite/centrality.py:72-73`

```python
def degree_centrality(G, nodes):
    top = set(nodes)
    bottom = set(G) - top  # 直接使用集合差集，不验证！
    # ...
```

**关键发现**：中心性算法**不验证** `nodes` 是否是有效的二分划！它们只是简单地执行 `set(G) - set(nodes)` 来获取另一个集合。

### 2.2 异常场景分析

我们分析以下 4 种异常场景：

| 场景 | 描述 | `degree_centrality` | `betweenness_centrality` | `closeness_centrality` |
|------|------|---------------------|--------------------------|------------------------|
| **场景 A** | 非连通二部图 | ✅ 正常运行 | ✅ 正常运行 | ✅ 正常运行 |
| **场景 B** | 无边图（两个孤立节点） | ✅ 返回 0.0 | ✅ 返回 0.0 | ✅ 返回 0.0 |
| **场景 C** | `nodes` 包含混合划分（两个集合的节点都有） | ❌ 不报错，但归一化错误 | ❌ 不报错，但归一化错误 | ❌ 不报错，但归一化错误 |
| **场景 D** | `nodes` 包含图中不存在的节点 | ❌ 可能 `KeyError` | ❌ 无意义结果 | ❌ 无意义结果 |

### 2.3 场景 A：非连通二部图

**测试验证**：
**关键代码位置**：`networkx/algorithms/bipartite/tests/test_centrality.py:58-72`

```python
def test_bipartite_closeness_centrality_unconnected(self):
    G = nx.complete_bipartite_graph(3, 3)
    G.add_edge(6, 7)  # 再加一个不连通的边（第二个二部分量）
    # 图结构：
    # 分量1: K_{3,3} - 节点 0,1,2 (top) 和 3,4,5 (bottom)
    # 分量2: 边 (6,7) - 节点 6 和 7
    
    c = bipartite.closeness_centrality(G, [0, 2, 4, 6], normalized=False)
    # 可以正常运行！不会报错！
    
    answer = {
        0: 10.0 / 7,   # 分量1 中的节点
        2: 10.0 / 7,
        4: 10.0 / 7,
        6: 10.0,        # 分量2 中的节点（到自己分量的距离更近）
        1: 10.0 / 7,
        3: 10.0 / 7,
        5: 10.0 / 7,
        7: 10.0,
    }
    assert c == answer
```

**行为分析**：

| 算法 | 非连通图处理 |
|------|-------------|
| `degree_centrality` | 只依赖 `G.degree()`，完全不关心连通性 |
| `betweenness_centrality` | 依赖 `nx.betweenness_centrality()`，非连通图中不连通的节点对贡献 0 |
| `closeness_centrality` | 依赖最短路径，可达部分正常计算，不可达部分不影响（因为只计算可达节点的路径和） |

**关键发现**：`closeness_centrality` 在非连通图上**可以正常运行**，因为它只计算从源节点可达的节点的路径长度。

### 2.4 场景 B：无边图（孤立节点）

**测试验证**：
**关键代码位置**：`networkx/algorithms/bipartite/tests/test_centrality.py:50-56`

```python
def test_closeness_centrality(self):
    G = nx.Graph()
    G.add_node(0)
    G.add_node(1)  # 两个节点，无边
    
    c = bipartite.closeness_centrality(G, [0])
    assert c == {0: 0.0, 1: 0.0}  # 返回 0.0，不报错
    
    c = bipartite.closeness_centrality(G, [1])
    assert c == {0: 0.0, 1: 0.0}
```

**行为分析**：

| 算法 | 无边图处理 |
|------|-----------|
| `degree_centrality` | 所有节点度数为 0，返回 `{node: 0.0, ...}` |
| `betweenness_centrality` | 没有最短路径经过任何节点，返回 `{node: 0.0, ...}` |
| `closeness_centrality` | 没有路径，`totsp = 0`，返回 0.0 |

**关键发现**：无边图不会导致报错，只是返回全 0 或无意义的结果。

### 2.5 场景 C：`nodes` 包含混合划分

这是**最危险**的场景，因为算法**不会报错**但会产生**无意义的结果**。

**示例分析**：

```python
# 正确的二部图
B = nx.complete_bipartite_graph(3, 2)
# top = {0, 1, 2}, bottom = {3, 4}

# 正确用法
c_correct = bipartite.degree_centrality(B, [0, 1, 2])
# top = {0, 1, 2}, bottom = {3, 4}
# 归一化因子：1/len(bottom) = 1/2, 1/len(top) = 1/3

# 错误用法：nodes 包含两个集合的节点
c_wrong = bipartite.degree_centrality(B, [0, 1, 3])  # 混合了 top 和 bottom！
# top = {0, 1, 3}, bottom = {2, 4}  ← 错误的划分！
# 归一化因子：1/len(bottom) = 1/2, 1/len(top) = 1/3
# 结果无意义，因为 0,1,3 不在同一个二分划中
```

**为什么不报错？**

看代码：
```python
top = set(nodes)
bottom = set(G) - top  # 只是简单的集合运算，不验证任何约束！
```

`set(G) - set(nodes)` 总是有效的，无论 `nodes` 是什么。

### 2.6 场景 D：`nodes` 包含图中不存在的节点

**示例分析**：

```python
B = nx.complete_bipartite_graph(3, 2)
# 节点：{0, 1, 2, 3, 4}

# 错误用法：nodes 包含不存在的节点
c = bipartite.degree_centrality(B, [0, 1, 100])  # 100 不存在！

# 实际计算：
# top = {0, 1, 100}
# bottom = {2, 3, 4}  # 100 不在 G 中，所以不影响
# 
# 然后调用 G.degree(top)：
# G.degree({0, 1, 100}) - 这取决于 NetworkX 如何处理不存在的节点
```

**可能的行为**：
- `degree_centrality`：调用 `G.degree(top)` 时，如果 `top` 包含不存在的节点，可能抛出 `KeyError`
- `betweenness_centrality` 和 `closeness_centrality`：`set(G) - top` 会忽略不存在的节点，但归一化因子会错误

### 2.7 中心性算法的"沉默失败"问题

**关键发现**：中心性算法存在**"沉默失败"（silent failure）**问题：

| 问题 | 表现 |
|------|------|
| **不验证二部图性质** | 即使输入图不是二部图，算法也会运行 |
| **不验证 `nodes` 有效性** | 即使 `nodes` 不是一个二分划，算法也会运行 |
| **归一化依赖 `nodes`** | 错误的 `nodes` 导致错误的归一化因子 |
| **无警告或错误** | 用户可能不知道结果是无意义的 |

**文档提示**：

中心性算法的文档中**没有**提到会验证 `nodes` 的有效性。用户需要自行确保：
1. 输入图确实是二部图
2. `nodes` 参数确实是二部图的一个二分划

---

## 3. 补充：最小权完整匹配与普通最大匹配的对比

### 3.1 对比总览

| 维度 | `hopcroft_karp_matching` (最大基数匹配) | `minimum_weight_full_matching` (最小权完整匹配) |
|------|------------------------------------------|------------------------------------------------|
| **算法目标** | 最大化匹配边数 | 最小化匹配边权值和 |
| **匹配要求** | 最大基数（不一定完整/完美） | **必须完整**：`|M| = min(|U|, |V|)` |
| **图类型支持** | 无向图 (`hopcroft_karp_matching`)<br>有向图 (`eppstein_matching`) | **仅无向图** |
| **外部依赖** | ❌ 无（纯 Python 实现） | ✅ 依赖 **NumPy + SciPy** |
| **失败模式 1** | 无 `top_nodes` 时非连通图抛出 `AmbiguousSolution` | 无 `top_nodes` 时非连通图抛出 `AmbiguousSolution` |
| **失败模式 2** | 不匹配的节点**不出现**在结果中 | 不存在完整匹配时**抛出 `ValueError`** |
| **实现方式** | BFS 分层 + DFS 增广（直接操作图） | 邻接矩阵 → `linear_sum_assignment` |

### 3.2 "完整匹配"的定义

**最小权完整匹配**中的"完整"（full）有明确的数学定义：

**文档说明**：
**关键代码位置**：`networkx/algorithms/bipartite/matching.py:507-523`

```
Let :math:`G = ((U, V), E)` be a weighted bipartite graph ...
This function then produces a matching
:math:`M \subseteq E` with cardinality

.. math::
   \lvert M \rvert = \min(\lvert U \rvert, \lvert V \rvert),

which minimizes the sum of the weights ...

When :math:`\lvert U \rvert = \lvert V \rvert`, this is commonly
referred to as a perfect matching; here, since we allow
:math:`\lvert U \rvert` and :math:`\lvert V \rvert` to differ, we
follow Karp and refer to the matching as *full*.
```

**关键概念**：

| 概念 | 定义 |
|------|------|
| **完整匹配 (Full Matching)** | 匹配大小 = `min(|U|, |V|)`。较小的集合必须完全匹配。 |
| **完美匹配 (Perfect Matching)** | 特殊的完整匹配，要求 `|U| = |V|` 且 `|M| = |U| = |V|`。 |

**示例**：
- 如果 `|U| = 3`, `|V| = 5`，完整匹配要求匹配大小为 3
- 如果 `|U| = 5`, `|V| = 3`，完整匹配要求匹配大小为 3
- 如果 `|U| = |V| = 4`，完整匹配 = 完美匹配，要求匹配大小为 4

### 3.3 依赖差异

**最小权完整匹配的依赖导入**：
**关键代码位置**：`networkx/algorithms/bipartite/matching.py:571-572`

```python
import numpy as np
import scipy as sp
```

**测试中的依赖检查**：
**关键代码位置**：`networkx/algorithms/bipartite/tests/test_matching.py:216-218`

```python
class TestMinimumWeightFullMatching:
    @classmethod
    def setup_class(cls):
        pytest.importorskip("scipy")  # 如果 SciPy 未安装，跳过整个测试类
```

**对比**：

| 函数 | 依赖 | 未安装时的行为 |
|------|------|----------------|
| `hopcroft_karp_matching` | 无 | 正常运行 |
| `eppstein_matching` | 无 | 正常运行 |
| `minimum_weight_full_matching` | NumPy + SciPy | 运行时抛出 `ImportError` |

### 3.4 实现方式差异

#### 3.4.1 Hopcroft-Karp 算法（直接操作图）

**关键代码位置**：`networkx/algorithms/bipartite/matching.py:127-170`

```python
def hopcroft_karp_matching(G, top_nodes=None):
    # ...
    left, right = bipartite_sets(G, top_nodes)
    leftmatches = {v: None for v in left}
    rightmatches = {v: None for v in right}
    # ...
    
    def breadth_first_search():
        # ...
        while queue:
            v = queue.popleft()
            if distances[v] < distances[None]:
                for u in G[v]:  # ◀── 直接使用 G[v] 访问邻居
                    if distances[rightmatches[u]] is INFINITY:
                        distances[rightmatches[u]] = distances[v] + 1
                        queue.append(rightmatches[u])
        # ...
    
    def depth_first_search(v):
        if v is not None:
            for u in G[v]:  # ◀── 直接使用 G[v] 访问邻居
                if distances[rightmatches[u]] == distances[v] + 1:
                    if depth_first_search(rightmatches[u]):
                        rightmatches[u] = v
                        leftmatches[v] = u
                        return True
            distances[v] = INFINITY
            return False
        return True
```

**特点**：
- 直接使用 `G[v]` 访问邻接表
- 纯 Python 实现，无依赖
- 操作匹配字典 `leftmatches` 和 `rightmatches`

#### 3.4.2 最小权完整匹配（矩阵方法）

**关键代码位置**：`networkx/algorithms/bipartite/matching.py:574-589`

```python
def minimum_weight_full_matching(G, top_nodes=None, weight="weight"):
    # 运行时导入依赖
    import numpy as np
    import scipy as sp

    left, right = nx.bipartite.sets(G, top_nodes)
    U = list(left)
    V = list(right)
    
    # 步骤 1：转换为二部邻接矩阵（稀疏格式）
    weights_sparse = biadjacency_matrix(
        G, row_order=U, column_order=V, weight=weight, format="coo"
    )
    
    # 步骤 2：转换为稠密矩阵，无穷大表示无边
    weights = np.full(weights_sparse.shape, np.inf)
    weights[weights_sparse.row, weights_sparse.col] = weights_sparse.data
    
    # 步骤 3：调用 SciPy 的线性和分配求解器
    left_matches = sp.optimize.linear_sum_assignment(weights)
    
    # 步骤 4：转换结果格式
    d = {U[u]: V[v] for u, v in zip(*left_matches)}
    d.update({v: u for u, v in d.items()})  # 添加反向映射
    return d
```

**流程示意图**：

```
输入: 二部图 G, top_nodes
     │
     ▼
┌─────────────────────┐
│ nx.bipartite.sets() │ 获取左右节点集合
└─────────────────────┘
     │
     ▼
┌─────────────────────┐
│ biadjacency_matrix()│ 转换为稀疏邻接矩阵
└─────────────────────┘
     │
     ▼
┌─────────────────────┐
│ np.full(..., np.inf)│ 转换为稠密矩阵，无边=inf
└─────────────────────┘
     │
     ▼
┌──────────────────────────────────┐
│ sp.optimize.linear_sum_assignment│ SciPy 求解器
│ (线性和分配问题)                   │
└──────────────────────────────────┘
     │
     ▼
┌─────────────────────┐
│ 构造匹配字典         │ 返回结果
└─────────────────────┘
```

### 3.5 失败模式差异

#### 3.5.1 共同的失败模式：非连通图 + 无 `top_nodes`

两个算法都会在这种情况下抛出 `AmbiguousSolution`：

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

#### 3.5.2 不同的失败模式：不存在完整匹配

这是**最小权完整匹配独有的**失败模式。

**测试验证**：
**关键代码位置**：`networkx/algorithms/bipartite/tests/test_matching.py:230-240`

```python
def test_minimum_weight_full_matching_with_no_full_matching(self):
    B = nx.Graph()
    B.add_nodes_from([1, 2, 3], bipartite=0)   # 左集合：3 个节点
    B.add_nodes_from([4, 5, 6], bipartite=1)    # 右集合：3 个节点
    B.add_edge(1, 4, weight=100)   # 1 只连接到 4
    B.add_edge(2, 4, weight=100)   # 2 只连接到 4
    B.add_edge(3, 4, weight=50)    # 3 连接到 4, 5, 6
    B.add_edge(3, 5, weight=50)
    B.add_edge(3, 6, weight=50)
    
    # 图结构：
    #   1 --\
    #        >-- 4 -- 3 -- 5
    #   2 --/           \
    #                    >-- 6
    #
    # 最大匹配大小 = 2（例如：1-4, 3-5）
    # 不存在完整匹配（大小为 3），因为 1 和 2 竞争 4
    
    with pytest.raises(ValueError):
        minimum_weight_full_matching(B)  # ❌ 抛出 ValueError
```

**对比**：

| 场景 | `hopcroft_karp_matching` | `minimum_weight_full_matching` |
|------|--------------------------|--------------------------------|
| 不存在完整匹配 | ✅ 返回最大匹配（部分节点不匹配） | ❌ **抛出 `ValueError`** |

**普通最大匹配的行为**：

```python
# 同样的图
matching = hopcroft_karp_matching(B, top_nodes=[1, 2, 3])
# 可能返回：{1: 4, 3: 5, 4: 1, 5: 3}
# 节点 2 和 6 没有匹配，不出现在结果中
# 不报错！
```

### 3.6 其他差异

#### 3.6.1 有向图支持

| 函数 | 有向图支持 |
|------|-----------|
| `hopcroft_karp_matching` | ❌ 仅无向图 |
| `eppstein_matching` | ✅ 支持有向图 |
| `minimum_weight_full_matching` | ❌ 仅无向图 |

**注意**：`maximum_matching` 是 `hopcroft_karp_matching` 的别名，因此也不支持有向图。

#### 3.6.2 边权重处理

| 函数 | 权重处理 |
|------|---------|
| `hopcroft_karp_matching` | ❌ 忽略权重（只关心基数） |
| `minimum_weight_full_matching` | ✅ 使用 `weight` 参数指定的边属性 |

#### 3.6.3 结果格式

两个函数的结果格式相同：

```python
# 返回字典，包含双向映射
matching = {
    left_node1: right_node1,
    left_node2: right_node2,
    right_node1: left_node1,   # 反向映射
    right_node2: left_node2,   # 反向映射
}
```

**未匹配的节点不出现在结果中**（对于 `hopcroft_karp_matching`）。

---

## 4. 总结

### 4.1 修正要点

| 原描述 | 修正后描述 |
|--------|-----------|
| "所有投影算法都有 `len(nodes) >= len(B)` 检查" | **只有 `weighted_projected_graph`** 有这个检查。其他投影函数（`projected_graph`, `collaboration_weighted_projected_graph`, `overlap_weighted_projected_graph`, `generic_weighted_projected_graph`）**没有**这个检查。 |
| （未提及） | `weighted_projected_graph` 需要这个检查是因为 `ratio=True` 时 `n_top` 作为分母。 |

### 4.2 补充要点：中心性算法的边界行为

| 场景 | 行为 | 风险 |
|------|------|------|
| 非连通图 | 正常运行 | 无 |
| 无边图（孤立节点） | 返回全 0.0 | 无 |
| `nodes` 包含混合划分 | 不报错，但归一化错误 | ⚠️ **沉默失败** |
| `nodes` 包含不存在的节点 | 可能 `KeyError` 或无意义结果 | ⚠️ 潜在错误 |

**关键发现**：中心性算法**不验证**输入的有效性，存在"沉默失败"风险。

### 4.3 补充要点：匹配算法对比

| 维度 | 最大基数匹配 (`hopcroft_karp_matching`) | 最小权完整匹配 (`minimum_weight_full_matching`) |
|------|------------------------------------------|------------------------------------------------|
| **目标** | 最大化边数 | 最小化权值和 |
| **匹配要求** | 最大基数 | **必须完整**：`|M| = min(|U|, |V|)` |
| **依赖** | 无 | **NumPy + SciPy** |
| **不存在完整匹配时** | 返回部分匹配，不报错 | **抛出 `ValueError`** |
| **实现方式** | BFS/DFS 直接操作图 | 邻接矩阵 → SciPy `linear_sum_assignment` |

### 4.4 实际使用建议

#### 投影算法

```python
from networkx.algorithms import bipartite

# projected_graph 可以投影到任意节点子集
G = nx.star_graph(3)
P = bipartite.projected_graph(G, [0])  # 投影到单个节点，合法！

# 但 weighted_projected_graph 有边界检查
G = nx.path_graph(4)
P = bipartite.weighted_projected_graph(G, [0, 1, 2, 3])  # ❌ NetworkXAlgorithmError
P = bipartite.weighted_projected_graph(G, [0, 2])  # ✅ 正常
```

#### 中心性算法

```python
# ⚠️ 风险：不会验证 nodes 是否是有效的二分划
B = nx.complete_bipartite_graph(3, 2)

# ✅ 正确用法
c = bipartite.degree_centrality(B, [0, 1, 2])  # top 集合

# ❌ 错误用法（不报错，但结果无意义）
c = bipartite.degree_centrality(B, [0, 1, 3])  # 混合了 top 和 bottom！

# ✅ 建议：先验证
from networkx.algorithms.bipartite import is_bipartite_node_set
assert is_bipartite_node_set(B, [0, 1, 2])  # 验证是否是有效划分
```

#### 匹配算法

```python
B = nx.Graph()
B.add_nodes_from([1, 2, 3], bipartite=0)
B.add_nodes_from([4, 5, 6], bipartite=1)
B.add_edges_from([(1, 4), (2, 4), (3, 4), (3, 5), (3, 6)])

# 不存在完整匹配（1 和 2 竞争 4）

# ✅ hopcroft_karp_matching：返回最大匹配，不报错
matching = bipartite.hopcroft_karp_matching(B, top_nodes=[1, 2, 3])
# 可能返回 {1: 4, 3: 5, 4: 1, 5: 3}

# ❌ minimum_weight_full_matching：抛出 ValueError
try:
    matching = bipartite.minimum_weight_full_matching(B, top_nodes=[1, 2, 3])
except ValueError:
    print("不存在完整匹配！")

# ✅ 建议：先检查是否存在完美/完整匹配
if bipartite.is_perfect_matching(B, matching):
    # ...
```
