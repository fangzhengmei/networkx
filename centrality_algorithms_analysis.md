# NetworkX 中心性度量算法分析报告

## 1. 统一接口设计与差异化实现方式

### 1.1 函数签名对比

| 算法 | 函数签名 | 关键参数 |
|------|----------|----------|
| 介数中心性 | `betweenness_centrality(G, k=None, normalized=True, weight=None, endpoints=False, seed=None)` | normalized, weight, k, endpoints |
| 接近度中心性 | `closeness_centrality(G, u=None, distance=None, wf_improved=True, *, sp=None)` | distance, wf_improved |
| 特征向量中心性 | `eigenvector_centrality(G, max_iter=100, tol=1.0e-6, nstart=None, weight=None)` | weight, max_iter, tol |
| PageRank | `pagerank(G, alpha=0.85, personalization=None, max_iter=100, tol=1.0e-6, nstart=None, weight="weight", dangling=None)` | alpha, weight, personalization |
| Katz 中心性 | `katz_centrality(G, alpha=0.1, beta=1.0, max_iter=1000, tol=1.0e-6, nstart=None, normalized=True, weight=None)` | alpha, beta, normalized, weight |

### 1.2 共享参数的差异化处理策略

#### 1.2.1 `normalized` 参数的差异化语义

同一个参数名 `normalized` 在不同算法中代表完全不同的数学含义：

**介数中心性 (betweenness.py:51-53)**：
```python
normalized : bool, optional (default=True)
    If `True`, the betweenness values are rescaled by dividing by the number of
    possible $(s, t)$-pairs in the graph.
```
- 语义：除以可能的 (s,t) 节点对数量进行缩放
- 数学含义：将原始计数转换为 0-1 范围内的比值
- 有向图：归一化因子为 $N(N-1)$
- 无向图：归一化因子为 $N(N-1)/2$

**Katz 中心性 (katz.py:74-75)**：
```python
normalized : bool, optional (default=True)
    If True normalize the resulting values.
```
- 语义：将结果向量归一化为单位欧氏范数
- 数学含义：$\|x\|_2 = 1$
- 实现：`s = 1.0 / math.hypot(*x.values())`

**特征向量中心性**：
- 无显式 `normalized` 参数，但结果自动归一化为单位欧氏范数
- 实现 (eigenvector.py:189)：`norm = math.hypot(*x.values()) or 1`

**PageRank**：
- 无显式 `normalized` 参数，但结果是概率分布（自动归一化）
- 所有中心性值之和为 1

**接近度中心性**：
- 无 `normalized` 参数，使用 `wf_improved` 参数控制 Wasserman-Faust 改进公式
- 当 `wf_improved=True` 时，按可达节点比例进行缩放

#### 1.2.2 `weight` 参数的差异化语义

**介数中心性**：
- 权重被解释为**距离**（用于最短路径计算）
- 实现：使用 Dijkstra 算法，权重越大，路径越长

**接近度中心性**：
- 通过 `distance` 参数指定（而非 `weight`）
- 语义：边的距离，用于最短路径计算

**特征向量中心性**：
- 权重被解释为**连接强度**
- 语义：权重越大，邻居的影响越大
- 实现 (eigenvector.py:183)：`w = G[n][nbr].get(weight, 1) if weight else 1`

**PageRank**：
- 权重用于计算随机跳转概率
- 语义：出边的权重决定跳转概率分布

**Katz 中心性**：
- 权重被解释为**连接强度**
- 语义：与特征向量中心性相同

### 1.3 有向图与无向图的语义差别

#### 介数中心性

**有向图**：
- 考虑有序对 $(s, t)$，每个方向独立计数
- 归一化因子：$N(N-1)$（包含端点）或 $(N-1)(N-2)$（不包含端点）

**无向图**：
- 考虑无序对 $\{s, t\}$，每条路径只计数一次
- 归一化因子：$N(N-1)/2$
- 实现 (betweenness.py:527-535)：
  ```python
  if not directed:
      correction = 2
  else:
      correction = 1
  scale = N / (K_source * correction)
  ```

#### 接近度中心性

**有向图**：
- 默认使用**入边距离**（从其他节点到目标节点的距离）
- 实现 (closeness.py:118-127)：
  ```python
  if G.is_directed():
      G = G.reverse()  # create a reversed graph view
  ```
- 如需使用出边距离，需手动调用 `G.reverse()`

**无向图**：
- 距离是对称的，无方向问题

#### 特征向量中心性

**有向图**：
- 计算**左特征向量**（对应入边）
- 数学含义：$\lambda x^T = x^T A$
- 语义：节点的中心性等于其前驱节点中心性之和

**无向图**：
- 邻接矩阵对称，左右特征向量相同
- 数学含义：$Ax = \lambda x$

#### PageRank

**有向图**：
- 原始设计，直接使用邻接矩阵

**无向图**：
- 不同实现版本处理方式不同：

  **1. 纯 Python 版本 (`_pagerank_python`)**：
  - 显式转换为有向图，每条边变为两个方向的边
  - 实现 (pagerank_alg.py:128)：`D = G.to_directed()`

  **2. SciPy 稀疏矩阵版本 (`_pagerank_scipy`，默认版本)**：
  - 不调用 `to_directed()`，直接通过 `nx.to_scipy_sparse_array(G, ...)` 构建邻接矩阵
  - 对于无向图，`to_scipy_sparse_array` 生成**对称矩阵**，隐式表示双向边
  - 实现 (pagerank_alg.py:460)：`A = nx.to_scipy_sparse_array(G, nodelist=nodelist, weight=weight, dtype=float)`

  **3. NumPy 稠密矩阵版本 (`_pagerank_numpy`)**：
  - 调用 `google_matrix()`，内部通过 `nx.to_numpy_array(G, ...)` 构建矩阵
  - 同样依赖无向图邻接矩阵的**对称性**隐式处理双向边
  - `google_matrix` 的文档声明："Undirected graphs will be converted to a directed graph with two directed edges for each undirected edge."，但实际是通过对称矩阵实现的

- 最终效果：所有版本在无向图上每条边都被视为双向的（入边和出边都计数）

#### Katz 中心性

**有向图**：
- 计算**左特征向量**（对应入边）
- 实现 (katz.py:325)：`A = nx.adjacency_matrix(G, ...).todense().T`

**无向图**：
- 邻接矩阵对称，结果对称

---

## 2. 介数中心性的 Brandes 算法实现

### 2.1 精确算法与近似采样版本的结构差异

#### 精确算法流程 (betweenness.py:216-244)

```python
def betweenness_centrality(G, k=None, normalized=True, weight=None, endpoints=False, seed=None):
    betweenness = dict.fromkeys(G, 0.0)
    if k == len(G):
        k = None
    if k is None:
        nodes = G  # 所有节点作为源
    else:
        nodes = seed.sample(list(G.nodes()), k)  # 采样节点
    
    for s in nodes:
        # 1. 单源最短路径计算
        if weight is None:
            S, P, sigma, _ = _single_source_shortest_path_basic(G, s)
        else:
            S, P, sigma, _ = _single_source_dijkstra_path_basic(G, s, weight)
        
        # 2. 反向累加路径贡献
        if endpoints:
            betweenness, _ = _accumulate_endpoints(betweenness, S, P, sigma, s)
        else:
            betweenness, _ = _accumulate_basic(betweenness, S, P, sigma, s)
    
    # 3. 归一化
    betweenness = _rescale(...)
    return betweenness
```

#### 关键差异点

| 特性 | 精确算法 | 近似采样版本 |
|------|----------|--------------|
| 源节点 | 所有节点 $V$ | 采样 $k$ 个节点 |
| 时间复杂度 | $O(n(m + n))$ 无权 / $O(n(m + n \log n))$ 有权 | $O(k(m + n))$ 无权 |
| 缩放因子 | 无需额外缩放 | 需按 $n/k$ 缩放 |
| 无向图修正 | 归一化时处理 | 采样时需特殊处理 |

### 2.2 反向累加路径的设计原理

Brandes 算法的核心创新在于**反向累加**（backward accumulation），避免了枚举所有最短路径。

#### 核心数据结构

在单源最短路径计算后，获得：
- `S`：按距离从源节点递增排序的节点列表
- `P[v]`：节点 $v$ 的前驱节点列表（最短路径上的前一个节点）
- `sigma[v]`：从源节点 $s$ 到 $v$ 的最短路径数量

#### 反向累加的数学原理

定义 $\delta(v)$ 为经过节点 $v$ 的最短路径对的依赖值：
$$\delta(v) = \sum_{w: v \in P(w)} \frac{\sigma(v)}{\sigma(w)} (1 + \delta(w))$$

节点 $v$ 的介数中心性增量为：
$$\Delta c_B(v) = \sum_{w \neq s, v} \delta(v)$$

#### 实现代码 (betweenness.py:461-470)

```python
def _accumulate_basic(betweenness, S, P, sigma, s):
    delta = dict.fromkeys(S, 0)
    while S:
        w = S.pop()  # 从最远的节点开始（栈操作）
        coeff = (1 + delta[w]) / sigma[w]
        for v in P[w]:
            delta[v] += sigma[v] * coeff
        if w != s:
            betweenness[w] += delta[w]
    return betweenness, delta
```

#### 设计要点

1. **栈结构**：`S.pop()` 从距离最远的节点开始处理（后进先出）
2. **依赖传递**：$\delta(w)$ 传递给其前驱节点
3. **系数计算**：`coeff = (1 + delta[w]) / sigma[w]` 包含：
   - 常数 1：表示路径 $s \to \dots \to w$ 本身
   - $\delta(w)$：表示经过 $w$ 的其他路径贡献
4. **路径计数**：按路径数量比例 `sigma[v] / sigma[w]` 分配贡献

#### 包含端点的版本 (betweenness.py:473-483)

```python
def _accumulate_endpoints(betweenness, S, P, sigma, s):
    betweenness[s] += len(S) - 1  # 源节点到所有其他节点的路径
    delta = dict.fromkeys(S, 0)
    while S:
        w = S.pop()
        coeff = (1 + delta[w]) / sigma[w]
        for v in P[w]:
            delta[v] += sigma[v] * coeff
        if w != s:
            betweenness[w] += delta[w] + 1  # +1 表示 s->w 这条路径
    return betweenness, delta
```

---

## 3. PageRank 三套实现的入口分发机制

### 3.1 三套实现对比

| 实现版本 | 函数名 | 底层技术 | 适用场景 |
|----------|--------|----------|----------|
| 默认版本 | `_pagerank_scipy` | SciPy 稀疏矩阵 + 幂迭代 | 大规模图（推荐） |
| NumPy 版本 | `_pagerank_numpy` | NumPy + LAPACK 特征值求解 | 小规模稠密图 |
| 纯 Python 版本 | `_pagerank_python` | 纯 Python 字典 + 幂迭代 | 无 SciPy/NumPy 环境 |

### 3.2 入口分发机制

**当前实现** (pagerank_alg.py:110-112)：
```python
def pagerank(...):
    return _pagerank_scipy(...)
```

当前默认直接调用 `_pagerank_scipy`，没有显式的回退逻辑。

### 3.3 各版本实现细节

#### SciPy 稀疏矩阵版本 (`_pagerank_scipy`)

**核心实现** (pagerank_alg.py:452-501)：
```python
def _pagerank_scipy(G, alpha=0.85, ...):
    import numpy as np
    import scipy as sp
    
    N = len(G)
    nodelist = list(G)
    
    # 1. 构建稀疏邻接矩阵
    A = nx.to_scipy_sparse_array(G, nodelist=nodelist, weight=weight, dtype=float)
    S = A.sum(axis=1)
    dangling_nodes = S == 0
    S[~dangling_nodes] = 1.0 / S[~dangling_nodes]  # 行归一化
    
    # 2. 构建行随机矩阵
    Q = sp.sparse.dia_array((S, 0), shape=A.shape).tocsr()
    A = Q @ A  # 归一化后的转移矩阵
    
    # 3. 幂迭代
    for _ in range(max_iter):
        xlast = x
        x = (
            alpha * (x @ A + np.sum(x[dangling_nodes]) * dangling_weights)
            + (1 - alpha) * p
        )
        # 收敛检查
        if err < N * tol:
            return dict(zip(nodelist, map(float, x)))
```

**优势**：
- 稀疏矩阵存储，内存效率高
- 矩阵运算向量化，速度快
- 适合大规模图

#### NumPy 稠密矩阵版本 (`_pagerank_numpy`)

**核心实现** (pagerank_alg.py:342-355)：
```python
def _pagerank_numpy(G, alpha=0.85, ...):
    import numpy as np
    
    # 1. 构建 Google 矩阵（稠密）
    M = google_matrix(G, alpha, personalization=personalization, ...)
    
    # 2. 直接求解特征值
    eigenvalues, eigenvectors = np.linalg.eig(M.T)
    ind = np.argmax(eigenvalues)
    largest = np.array(eigenvectors[:, ind]).flatten().real
    
    # 3. 归一化
    norm = largest.sum()
    return dict(zip(G, map(float, largest / norm)))
```

**特点**：
- 使用 `np.linalg.eig` 直接求解主特征向量
- 需要存储稠密矩阵，内存复杂度 $O(n^2)$
- 适合小规模图（$n < 1000$）

#### 纯 Python 版本 (`_pagerank_python`)

**核心实现** (pagerank_alg.py:115-172)：
```python
def _pagerank_python(G, alpha=0.85, ...):
    D = G.to_directed()
    W = nx.stochastic_graph(D, weight=weight)  # 行随机图
    N = W.number_of_nodes()
    
    # 初始化
    x = dict.fromkeys(W, 1.0 / N)
    p = dict.fromkeys(W, 1.0 / N)
    dangling_nodes = [n for n in W if W.out_degree(n, weight=weight) == 0.0]
    
    # 幂迭代（纯 Python 循环）
    for _ in range(max_iter):
        xlast = x
        x = dict.fromkeys(xlast.keys(), 0)
        danglesum = alpha * sum(xlast[n] for n in dangling_nodes)
        
        for n in x:
            # 逐边遍历更新
            for _, nbr, wt in W.edges(n, data=weight):
                x[nbr] += alpha * xlast[n] * wt
            x[n] += danglesum * dangling_weights.get(n, 0) + (1.0 - alpha) * p.get(n, 0)
        
        if err < N * tol:
            return x
```

**特点**：
- 纯 Python 实现，无外部依赖
- 使用字典存储，逐边遍历
- 速度最慢，仅作为备用

### 3.4 Google 矩阵构建

`google_matrix` 函数构建完整的 PageRank 转移矩阵 (pagerank_alg.py:235-268)：

```python
def google_matrix(G, alpha=0.85, personalization=None, ...):
    import numpy as np
    
    # 1. 邻接矩阵
    A = nx.to_numpy_array(G, nodelist=nodelist, weight=weight)
    
    # 2. 处理悬挂节点（出度为 0）
    dangling_nodes = np.where(A.sum(axis=1) == 0)[0]
    A[dangling_nodes] = dangling_weights  # 按个性化向量跳转
    
    # 3. 行归一化
    A /= A.sum(axis=1)[:, np.newaxis]
    
    # 4. 添加随机跳转
    return alpha * A + (1 - alpha) * p
```

数学公式：
$$M = \alpha A + (1 - \alpha) \mathbf{1} p^T$$

其中：
- $A$：行归一化的邻接矩阵
- $p$：个性化向量（默认均匀分布）
- $\alpha$：阻尼因子（默认 0.85）

---

## 3.5 特征向量中心性与 Katz 中心性的多版本实现

与 PageRank 类似，特征向量中心性和 Katz 中心性也提供了两套实现：幂迭代版本和 NumPy 直接求解版本。

### 3.5.1 特征向量中心性的两套实现对比

| 特性 | 幂迭代版 `eigenvector_centrality` | NumPy 版 `eigenvector_centrality_numpy` |
|------|-----------------------------------|----------------------------------------|
| 底层技术 | 纯 Python 幂迭代 | SciPy ARPACK (`sp.sparse.linalg.eigs`) |
| 图连通性要求 | 无（但非强连通图可能收敛慢） | **必须连通**（有向图强连通，无向图连通） |
| 默认迭代次数 | `max_iter=100` | `max_iter=50`（Arnoldi 迭代） |
| 收敛阈值 | `tol=1e-6` | `tol=0`（机器精度） |
| 异常类型 | `PowerIterationFailedConvergence` | `ArpackNoConvergence`, `AmbiguousSolution` |

#### 实现差异分析

**幂迭代版核心实现** (eigenvector.py:162-194)：
```python
def eigenvector_centrality(G, max_iter=100, tol=1.0e-6, nstart=None, weight=None):
    # 初始化：默认全 1 向量
    if nstart is None:
        nstart = {v: 1 for v in G}
    x = {k: v / nstart_sum for k, v in nstart.items()}
    
    # 幂迭代：使用 (A + I) 而非 A 以保证收敛
    for _ in range(max_iter):
        xlast = x
        x = xlast.copy()  # I * xlast
        # 累加 A * xlast
        for n in x:
            for nbr in G[n]:
                w = G[n][nbr].get(weight, 1) if weight else 1
                x[nbr] += xlast[n] * w
        
        # 归一化为单位欧氏范数
        norm = math.hypot(*x.values()) or 1
        x = {k: v / norm for k, v in x.items()}
        
        # L1 范数收敛检查
        if sum(abs(x[n] - xlast[n]) for n in x) < nnodes * tol:
            return x
    raise nx.PowerIterationFailedConvergence(max_iter)
```

**NumPy 版核心实现** (eigenvector.py:339-357)：
```python
def eigenvector_centrality_numpy(G, weight=None, max_iter=50, tol=0):
    # 前置检查：图必须连通
    connected = nx.is_strongly_connected(G) if G.is_directed() else nx.is_connected(G)
    if not connected:
        raise nx.AmbiguousSolution(
            "`eigenvector_centrality_numpy` does not give consistent results for disconnected graphs"
        )
    
    # 构建稀疏邻接矩阵
    M = nx.to_scipy_sparse_array(G, nodelist=list(G), weight=weight, dtype=float)
    
    # 使用 ARPACK 求解最大实特征值对应的特征向量
    # which="LR" 表示 Largest Real part
    _, eigenvector = sp.sparse.linalg.eigs(
        M.T, k=1, which="LR", maxiter=max_iter, tol=tol
    )
    
    # 处理结果：取实部并归一化
    largest = eigenvector.flatten().real
    norm = np.sign(largest.sum()) * sp.linalg.norm(largest)
    return dict(zip(G, (largest / norm).tolist()))
```

#### 关键设计差异

1. **矩阵选择**：
   - 幂迭代版：使用 $(A + I)$ 进行迭代，避免负主导特征值问题
   - NumPy 版：直接使用 $A^T$，通过 `which="LR"` 指定寻找最大实特征值

2. **连通性要求**：
   - 幂迭代版：无强制要求，但非连通图可能收敛到各分量的混合
   - NumPy 版：**强制要求连通**，否则抛出 `AmbiguousSolution`
   - 原因：ARPACK 在非连通图上可能选择不同的特征向量，结果不一致

3. **异常处理**：
   - 幂迭代版：超时时抛出 `PowerIterationFailedConvergence`
   - NumPy 版：ARPACK 不收敛时抛出 `ArpackNoConvergence`，图不连通时抛出 `AmbiguousSolution`

#### 适用场景

| 场景 | 推荐版本 | 原因 |
|------|----------|------|
| 大规模稀疏图 | 幂迭代版 | 内存效率高，无需构建完整矩阵 |
| 小规模连通图 | NumPy 版 | 直接求解，无收敛问题 |
| 非连通图 | 幂迭代版 | NumPy 版会拒绝计算 |
| 需要精确控制收敛 | 幂迭代版 | 可调整 `tol` 和 `max_iter` |
| 快速验证 | NumPy 版 | 直接求解，无需担心迭代次数 |

---

### 3.5.2 Katz 中心性的两套实现对比

| 特性 | 幂迭代版 `katz_centrality` | NumPy 版 `katz_centrality_numpy` |
|------|----------------------------|---------------------------------|
| 底层技术 | 纯 Python 幂迭代 | NumPy 线性求解 (`np.linalg.solve`) |
| α 条件 | 迭代中检查收敛 | **必须满足** $\alpha < 1/\lambda_{\max}$ |
| 稠密矩阵 | 不需要 | **需要**（构建完整稠密矩阵） |
| 异常类型 | `PowerIterationFailedConvergence` | `LinAlgError`（矩阵奇异时） |

#### 实现差异分析

**幂迭代版核心实现** (katz.py:149-194)：
```python
def katz_centrality(G, alpha=0.1, beta=1.0, max_iter=1000, tol=1.0e-6, ...):
    # 初始化：默认全 0 向量
    if nstart is None:
        x = {n: 0 for n in G}
    
    # 构建 beta 向量
    try:
        b = dict.fromkeys(G, float(beta))
    except (TypeError, ValueError, AttributeError):
        b = beta  # beta 可能是字典
    
    # 幂迭代：x = alpha * A^T * x + beta
    for _ in range(max_iter):
        xlast = x
        x = dict.fromkeys(xlast, 0)
        
        # 计算 A^T * xlast
        for n in x:
            for nbr in G[n]:
                x[nbr] += xlast[n] * G[n][nbr].get(weight, 1)
        
        # x = alpha * (A^T * xlast) + beta
        for n in x:
            x[n] = alpha * x[n] + b[n]
        
        # 收敛检查
        error = sum(abs(x[n] - xlast[n]) for n in x)
        if error < nnodes * tol:
            # 可选归一化
            if normalized:
                s = 1.0 / math.hypot(*x.values())
            else:
                s = 1
            for n in x:
                x[n] *= s
            return x
    raise nx.PowerIterationFailedConvergence(max_iter)
```

**NumPy 版核心实现** (katz.py:309-331)：
```python
def katz_centrality_numpy(G, alpha=0.1, beta=1.0, normalized=True, weight=None):
    # 构建 beta 向量
    try:
        nodelist = beta.keys()
        b = np.array(list(beta.values()), dtype=float)
    except AttributeError:
        nodelist = list(G)
        b = np.ones((len(nodelist), 1)) * beta
    
    # 构建邻接矩阵的转置（稠密矩阵）
    A = nx.adjacency_matrix(G, nodelist=nodelist, weight=weight).todense().T
    
    # 直接求解线性方程组：(I - alpha * A^T) * x = beta
    n = A.shape[0]
    centrality = np.linalg.solve(np.eye(n, n) - (alpha * A), b).squeeze()
    
    # 可选归一化
    norm = np.sign(np.sum(centrality)) * np.linalg.norm(centrality) if normalized else 1
    return dict(zip(nodelist, (centrality / norm).tolist()))
```

#### 关键设计差异

1. **数学方法**：
   - 幂迭代版：通过迭代求解不动点方程 $x = \alpha A^T x + \beta$
   - NumPy 版：直接求解线性方程组 $(I - \alpha A^T) x = \beta$

2. **α 参数约束**：
   - 幂迭代版：只要收敛即可，不强制检查 $\alpha < 1/\lambda_{\max}$
   - NumPy 版：**要求矩阵 $(I - \alpha A^T)$ 可逆**，即 $\alpha < 1/\lambda_{\max}$
   - 如果 $\alpha$ 过大，NumPy 版会抛出 `LinAlgError`（矩阵奇异）

3. **内存需求**：
   - 幂迭代版：$O(n)$，只需存储当前迭代向量
   - NumPy 版：$O(n^2)$，需要构建完整稠密邻接矩阵

4. **归一化时机**：
   - 幂迭代版：收敛后才进行归一化
   - NumPy 版：求解后进行归一化

#### 适用场景

| 场景 | 推荐版本 | 原因 |
|------|----------|------|
| 大规模图 ($n > 10^4$) | 幂迭代版 | 无需稠密矩阵，内存可控 |
| 小规模图 ($n < 10^3$) | NumPy 版 | 直接求解，精度高 |
| α 接近 $1/\lambda_{\max}$ | 幂迭代版 | NumPy 版可能因矩阵接近奇异而失败 |
| 需要 β 为字典（节点个性化） | 两者均可 | 两个版本都支持 β 为字典 |
| 需要非归一化结果 | 两者均可 | 通过 `normalized=False` 控制 |

---

### 3.5.3 多版本策略的设计哲学

与 PageRank 类似，特征向量中心性和 Katz 中心性的多版本策略体现了以下设计考量：

1. **性能与精度的权衡**：
   - 幂迭代版：牺牲一定精度（需收敛）换取内存效率和可扩展性
   - NumPy 版：牺牲内存换取精确性和速度（小规模图）

2. **用户群体分层**：
   - 普通用户：使用默认幂迭代版，无需理解特征值求解细节
   - 高级用户：可选择 NumPy 版获得更精确的结果

3. **与 PageRank 的差异**：
   - PageRank：默认使用 SciPy 稀疏矩阵版（性能最优）
   - 特征向量/Katz：默认使用纯 Python 幂迭代版（兼容性更好）
   - 原因：PageRank 常用于大规模 Web 图，性能更关键；特征向量/Katz 更多用于学术研究，兼容性更重要

4. **无自动降级机制**：
   - 与 PageRank 相同，特征向量中心性和 Katz 中心性的两套实现也是**独立的可调用工具**
   - `eigenvector_centrality` 和 `eigenvector_centrality_numpy` 是两个独立的公开函数
   - 不存在一个入口函数自动选择版本的机制
   - 用户需根据需求显式选择调用哪个版本

---

## 4. `normalize` 参数的语义差异总结

### 4.1 各算法归一化方式对比

| 算法 | 参数名 | 数学含义 | 归一化目标 |
|------|--------|----------|------------|
| 介数中心性 | `normalized` | 除以 (s,t) 对数量 | 值范围 [0, 1] |
| Katz 中心性 | `normalized` | 欧氏范数归一化 | $\|x\|_2 = 1$ |
| 特征向量中心性 | （无参数） | 欧氏范数归一化 | $\|x\|_2 = 1$ |
| PageRank | （无参数） | L1 范数归一化 | $\sum x_i = 1$ |
| 接近度中心性 | `wf_improved` | 按可达比例缩放 | 多组件图修正 |

### 4.2 详细数学解释

#### 介数中心性的归一化

**原始定义**：
$$c_B(v) = \sum_{s \neq v \neq t} \frac{\sigma(s, t | v)}{\sigma(s, t)}$$

**归一化后**：
- 有向图：$c_B'(v) = \frac{c_B(v)}{(n-1)(n-2)}$
- 无向图：$c_B'(v) = \frac{2c_B(v)}{(n-1)(n-2)}$

**目的**：使得最大可能介数为 1（完全图中中心节点的介数）

#### Katz 中心性的归一化

**原始定义**：
$$x = \alpha A x + \beta \mathbf{1}$$

**归一化后**：
$$x' = \frac{x}{\|x\|_2}$$

**目的**：使结果向量具有单位长度，便于比较不同参数下的结果

#### 特征向量中心性的归一化

**数学定义**：
$$\lambda x = A x$$

**归一化**：特征向量天然具有任意缩放性，因此归一化为单位范数
$$x' = \frac{x}{\|x\|_2}$$

#### PageRank 的归一化

**数学定义**：PageRank 是随机游走的平稳分布
$$x^T = \alpha x^T A + (1 - \alpha) p^T$$

**归一化**：结果自动满足 $\sum x_i = 1$，是概率分布

### 4.3 使用注意事项

1. **参数名相同但语义不同**：
   - `normalized=True` 在介数中心性中是"除以对数量"
   - 在 Katz 中心性中是"单位欧氏范数"

2. **缺省行为不同**：
   - 介数中心性默认 `normalized=True`
   - 特征向量中心性无参数但始终归一化
   - PageRank 无参数但结果是概率分布

3. **可比性问题**：
   - 不同算法的归一化值不可直接比较
   - 介数中心性的 0.5 与 Katz 中心性的 0.5 含义完全不同

---

## 5. 大规模图场景下的精度与效率权衡

### 5.1 各算法的时间复杂度

| 算法 | 精确版本复杂度 | 近似版本复杂度 | 近似策略 |
|------|----------------|----------------|----------|
| 介数中心性 | $O(n(m + n))$ 无权 / $O(n(m + n \log n))$ 有权 | $O(k(m + n))$ | 采样 $k$ 个源节点 |
| 接近度中心性 | $O(n(m + n))$ | - | 无官方近似 |
| 特征向量中心性 | $O(t m)$，$t$ 为迭代次数 | - | 幂迭代本身是近似 |
| PageRank | $O(t m)$，$t$ 为迭代次数 | - | 幂迭代本身是近似 |
| Katz 中心性 | $O(t m)$，$t$ 为迭代次数 | - | 幂迭代本身是近似 |

### 5.2 介数中心性的采样近似机制

#### 采样原理

精确介数中心性需要对每个节点作为源进行最短路径计算：
$$c_B(v) = \sum_{s \in V} \sum_{t \in V} \frac{\sigma(s, t | v)}{\sigma(s, t)}$$

采样近似只对 $k$ 个采样节点作为源进行计算，然后按比例缩放：
$$\hat{c}_B(v) = \frac{n}{k} \sum_{s \in S_k} \sum_{t \in V} \frac{\sigma(s, t | v)}{\sigma(s, t)}$$

#### 实现细节 (betweenness.py:503-560)

```python
def _rescale(betweenness, n, *, normalized, directed, endpoints=True, sampled_nodes=None):
    k = None if sampled_nodes is None else len(sampled_nodes)
    N = n if endpoints else n - 1
    
    K_source = N if k is None else k
    
    if k is None or endpoints:
        # 精确计算或包含端点的采样
        if normalized:
            scale = 1 / (K_source * (N - 1))
        else:
            # 无向图修正：每条路径被计数两次
            correction = 2 if not directed else 1
            scale = N / (K_source * correction)
    else:
        # 排除端点时的采样，需区分源节点和非源节点
        if normalized:
            scale_source = 1 / ((K_source - 1) * (N - 1)) if K_source > 1 else math.nan
            scale_nonsource = 1 / (K_source * (N - 1))
        else:
            correction = 1 if directed else 2
            scale_source = N / ((K_source - 1) * correction) if K_source > 1 else math.nan
            scale_nonsource = N / (K_source * correction)
        
        # 源节点使用不同的缩放因子
        sampled_nodes = set(sampled_nodes)
        for v in betweenness:
            betweenness[v] *= scale_source if v in sampled_nodes else scale_nonsource
```

#### 采样的无偏性

当 $k = n$ 时，采样结果应与精确结果一致。代码中 (betweenness.py:217-219)：
```python
if k == len(G):
    # This is done for performance; the result is the same regardless.
    k = None
```

#### 精度-效率权衡的理论分析

根据 Brandes 和 Pich (2007) 的理论分析，采样近似的误差界具有以下特性：

**渐近复杂度**：
- 要达到相对误差 $\epsilon$，所需的采样数量为 $k = O\left(\frac{\log n}{\epsilon^2}\right)$
- 这是一个与图结构相关的上界，实际误差取决于具体的图拓扑

**重要注意事项**：
- **不存在与图无关的固定百分比误差**：不同的图结构会导致相同采样比例下的误差差异很大
- 例如，在一个高度偏斜的图（如星形图）中，中心节点的介数可能被低估，而边缘节点可能被高估
- 采样误差还取决于节点的实际介数值：介数大的节点通常需要更多样本才能准确估计

**实际使用建议**：
- $k = 50$：可用于快速探索性分析，识别明显的异常节点
- $k = 100$：通常足以识别前 1% 的重要节点
- $k = 1000$：在大多数图上可获得合理的近似质量
- 如果需要精确排序或精确数值，应使用精确版本（$k = \text{None}$）

**采样的无偏性保证**：
- 当 $k = n$ 时，采样结果与精确结果在数学上等价
- 代码中对此进行了优化 (betweenness.py:217-219)：
  ```python
  if k == len(G):
      k = None
  ```

### 5.3 幂迭代类算法的收敛性与失败边界

特征向量中心性、PageRank、Katz 中心性都使用幂迭代方法，其精度由迭代次数和收敛阈值控制。

#### 收敛条件

以特征向量中心性为例 (eigenvector.py:191-193)：
```python
if sum(abs(x[n] - xlast[n]) for n in x) < nnodes * tol:
    return x
```

**L1 范数收敛条件**：$\|x^{(t)} - x^{(t-1)}\|_1 < n \cdot \text{tol}$

#### 影响收敛速度的因素

1. **谱间隙**：$\lambda_1 - |\lambda_2|$ 越大，收敛越快
   - PageRank 的阻尼因子 $\alpha$ 间接控制谱间隙
   - $\alpha$ 越小，收敛越快，但结果越接近均匀分布

2. **图结构**：
   - 强连通图收敛快
   - 有多个连通分量的图可能收敛慢或不收敛

3. **起始向量**：
   - 好的起始向量（如 `nstart`）可减少迭代次数
   - PageRank 默认使用均匀分布

#### 收敛失败边界场景

当幂迭代达到 `max_iter` 次仍未满足收敛条件时，所有幂迭代版本都会抛出 `PowerIterationFailedConvergence` 异常。

**各算法抛出位置与条件**：

| 算法 | 抛出位置 | 默认 `max_iter` | 收敛检查方式 |
|------|----------|-----------------|--------------|
| 特征向量中心性（幂迭代） | `eigenvector.py:194` | 100 | L1 范数 `< nnodes * tol` |
| Katz 中心性（幂迭代） | `katz.py:194` | 1000 | L1 范数 `< nnodes * tol` |
| PageRank（纯 Python） | `pagerank_alg.py:172` | 100 | L1 范数 `< N * tol` |
| PageRank（SciPy） | `pagerank_alg.py:501` | 100 | L1 范数 `< N * tol` |

**触发收敛失败的常见原因**：

1. **谱间隙过小**：
   - 当次大特征值 $|\lambda_2|$ 非常接近主特征值 $\lambda_1$ 时
   - 收敛速度呈几何级数下降：$O\left(\left|\frac{\lambda_2}{\lambda_1}\right|^t\right)$
   - PageRank 中 $\alpha$ 接近 1 时会出现此问题（$\alpha = 0.99$ 可能导致收敛极慢）

2. **起始向量投影不佳**：
   - 如果 `nstart` 提供的起始向量在主特征向量上的投影很小
   - 幂迭代需要更多次迭代才能"捕捉"到主特征向量
   - 特征向量中心性使用 $(A + I)$ 而非 $A$ 进行迭代，部分缓解了此问题

3. **图结构问题**：
   - 非强连通的有向图可能存在多个主特征向量
   - 幂迭代可能在多个特征向量之间"震荡"
   - 多个连通分量的无向图可能收敛到各分量的混合

4. **数值精度问题**：
   - 浮点数累积误差可能导致收敛检测失效
   - 特别在图规模很大时，`nnodes * tol` 的阈值可能不够精确

5. **Katz 中心性的 α 参数问题**：
   - 如果 $\alpha \geq 1/\lambda_{\max}$，理论上不收敛
   - 幂迭代可能发散或震荡

**调用方应如何处理**：

1. **捕获异常并调整参数**：
   ```python
   try:
       centrality = nx.eigenvector_centrality(G, max_iter=100, tol=1e-6)
   except nx.PowerIterationFailedConvergence:
       # 增加迭代次数或放宽收敛阈值
       centrality = nx.eigenvector_centrality(G, max_iter=500, tol=1e-4)
   ```

2. **使用 numpy 版本作为备选**：
   - `eigenvector_centrality_numpy` 使用 ARPACK 直接求解，不依赖幂迭代
   - `katz_centrality_numpy` 使用线性方程组直接求解
   - **注意**：这些版本有自身的限制：
     - `eigenvector_centrality_numpy` 要求图连通，否则抛出 `AmbiguousSolution`
     - `katz_centrality_numpy` 要求 $\alpha < 1/\lambda_{\max}$，否则可能抛出 `LinAlgError`

3. **调整算法参数**：
   - **PageRank**：减小 $\alpha$（如从 0.85 改为 0.8）可增大谱间隙，加快收敛
   - **Katz 中心性**：确保 $\alpha < 1/\lambda_{\max}$，可使用 `max(nx.adjacency_spectrum(G))` 计算最大特征值
   - **所有算法**：增大 `max_iter` 或放宽 `tol`

4. **预处理图结构**：
   - 对非强连通图，考虑分析其强连通分量
   - 或使用其他中心性度量（如 PageRank 天然处理非强连通图，因为有随机跳转）

5. **切换到其他中心性度量**：
   - 如果幂迭代持续失败，考虑使用更稳定的算法：
     - 介数中心性（无收敛问题，但计算复杂度高）
     - 接近度中心性（无收敛问题）
     - 度中心性（最简单，无收敛问题）

### 5.4 大规模图的实际建议

#### 选择精确算法的场景

- 图规模 $n < 10^4$
- 需要精确结果（如学术研究、关键决策）
- 有向图上的介数中心性（采样偏差较大）

#### 选择近似算法的场景

- 图规模 $n > 10^5$
- 仅需相对排序（如识别前 K 个重要节点）
- 探索性数据分析

#### 采样大小选择建议

根据 Brandes 和 Pich (2007) 的研究：
- 要达到 $\epsilon$ 相对误差，需要 $k \approx O(\frac{\log n}{\epsilon^2})$ 个样本
- 实践中，$k = 100$ 通常足以识别前 1% 的重要节点
- $k = 1000$ 可在大多数图上获得合理的近似

### 5.5 各算法的内存复杂度

| 算法 | 内存复杂度 | 备注 |
|------|------------|------|
| 介数中心性（精确） | $O(n + m)$ | BFS/Dijkstra 辅助空间 |
| 介数中心性（采样） | $O(n + m)$ | 与精确版本相同 |
| 特征向量中心性 | $O(n)$ | 仅需存储当前迭代向量 |
| PageRank (SciPy) | $O(m)$ | 稀疏邻接矩阵 |
| PageRank (NumPy) | $O(n^2)$ | 稠密矩阵，不适合大规模 |
| Katz 中心性 | $O(n)$ | 仅需存储当前迭代向量 |

---

## 6. 总结与设计洞察

### 6.1 统一接口设计哲学

NetworkX 的中心性算法接口设计遵循以下原则：

1. **参数名复用但语义明确**：
   - `weight` 参数在所有算法中都存在，但文档中明确说明其语义（距离 vs 强度）
   - `normalized` 参数仅在语义相似的算法中使用

2. **合理的默认值**：
   - 介数中心性默认 `normalized=True`（用户通常需要归一化结果）
   - PageRank 默认 `alpha=0.85`（经典值）

3. **渐进式暴露复杂度**：
   - 简单用户：使用默认参数即可获得合理结果
   - 高级用户：可调整 `tol`, `max_iter`, `nstart` 等参数

### 6.2 差异化实现的考量

1. **介数中心性的特殊性**：
   - 唯一提供采样近似的精确算法
   - 原因：时间复杂度最高（$O(n(m + n))$）
   - 其他算法的幂迭代本身已是近似方法

2. **PageRank 的多版本策略**：
   - 默认使用 SciPy 稀疏矩阵版本（性能最优）
   - 保留 NumPy 和纯 Python 版本作为独立的可调用工具
   - **注意**：当前实现**不存在自动回退/降级机制**：
     - 公开入口 `pagerank()` 直接硬编码调用 `_pagerank_scipy()` (pagerank_alg.py:110-112)
     - 如果 SciPy 不可用，会直接抛出 `ImportError`，而非自动回退到 `_pagerank_numpy` 或 `_pagerank_python`
     - `_pagerank_numpy` 和 `_pagerank_python` 是内部函数，用户需显式导入调用
   - 三套实现的定位：
     - `_pagerank_scipy`：生产环境首选，性能最优
     - `_pagerank_numpy`：小规模图精确求解，需理解稠密矩阵特性
     - `_pagerank_python`：无依赖环境备用，或用于学习理解算法

3. **特征向量 vs Katz 的关系**：
   - Katz 中心性是特征向量中心性的推广
   - 当 $\alpha \to 1/\lambda_{\max}$ 且 $\beta > 0$ 时，Katz 趋近于特征向量中心性
   - 代码中分别实现，但共享幂迭代框架

### 6.3 有向图处理的一致性

所有算法对有向图的处理保持一致的设计哲学：

1. **默认使用入边**：
   - 接近度中心性：自动反转图
   - 特征向量中心性：计算左特征向量
   - Katz 中心性：使用 $A^T$
   - PageRank：原始设计即为入边重要性

2. **提供显式反转机制**：
   - 如需使用出边重要性，用户可手动调用 `G.reverse()`
   - 文档中明确说明这一点

### 6.4 工程实现的亮点

1. **Brandes 算法的反向累加**：
   - 将 $O(n^3)$ 的朴素算法优化到 $O(n(m + n))$
   - 巧妙的依赖传递设计，避免枚举所有路径

2. **PageRank 的稀疏矩阵优化**：
   - 使用 SciPy 稀疏矩阵，内存效率提升显著
   - 悬挂节点的向量化处理，避免条件分支

3. **幂迭代的收敛检查**：
   - 使用 L1 范数而非 L2 范数，计算更快
   - 按节点数缩放阈值 `nnodes * tol`，与图规模无关

---

## 参考文献

[1] Ulrik Brandes. A Faster Algorithm for Betweenness Centrality. Journal of Mathematical Sociology, 25(2):163-177, 2001.

[2] Ulrik Brandes and Christian Pich. Centrality Estimation in Large Networks. International Journal of Bifurcation and Chaos, 17(7):2303-2318, 2007.

[3] Linton C. Freeman. A set of measures of centrality based on betweenness. Sociometry, 40:35-41, 1977.

[4] Leo Katz. A New Status Index Derived from Sociometric Index. Psychometrika, 18(1):39-43, 1953.

[5] Page, Lawrence; Brin, Sergey; Motwani, Rajeev and Winograd, Terry. The PageRank citation ranking: Bringing order to the Web. 1999.

[6] Mark E. J. Newman. Networks: An Introduction. Oxford University Press, USA, 2010.
