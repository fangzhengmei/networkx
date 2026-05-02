# NetworkX 多格式图序列化与反序列化架构分析

## 一、整体架构概览

NetworkX 的图序列化与反序列化功能集中在 `networkx/readwrite/` 模块中，采用**模块化设计**，每种格式对应独立的实现文件。这种架构使得添加新格式或修改现有格式的实现不会影响其他格式。

### 1.1 模块组织结构

```
networkx/readwrite/
├── __init__.py           # 统一导出所有读写函数
├── graphml.py            # GraphML 格式（XML，功能最完整）
├── gml.py                # GML 格式（键值对层次结构）
├── gexf.py               # GEXF 格式（支持动态图的 XML 格式）
├── edgelist.py           # Edge List 格式（简单文本格式）
├── adjlist.py            # Adjacency List 格式
├── multiline_adjlist.py  # 多行邻接表格式
├── json_graph/           # JSON 系列格式
│   ├── __init__.py
│   ├── node_link.py      # Node-Link 格式（适合 d3.js）
│   ├── adjacency.py      # Adjacency 格式
│   ├── tree.py           # Tree 格式
│   └── cytoscape.py      # Cytoscape JSON 格式
├── pajek.py              # Pajek 格式
├── leda.py               # LEDA 格式
├── sparse6.py            # Sparse6 紧凑编码格式
├── graph6.py             # Graph6 紧凑编码格式
├── text.py               # 文本格式
└── p2g.py                # 其他格式
```

### 1.2 统一的 API 设计模式

所有格式模块都遵循相似的 API 设计模式：

| 函数名模式 | 功能 | 示例 |
|-----------|------|------|
| `read_*()` | 从文件读取图 | `read_graphml()`, `read_gml()` |
| `write_*()` | 将图写入文件 | `write_graphml()`, `write_gml()` |
| `parse_*()` | 从字符串/迭代器解析图 | `parse_graphml()`, `parse_gml()` |
| `generate_*()` | 生成字符串表示（迭代器） | `generate_graphml()`, `generate_gml()` |

---

## 二、各格式实现模块详解

### 2.1 GraphML 格式 (`graphml.py`)

**特点**：功能最完整的 XML 格式，支持所有属性类型，工业标准格式。

#### 核心类设计

- **`GraphML` (基类)**：定义命名空间、类型映射系统
- **`GraphMLWriter`**：负责序列化
- **`GraphMLWriterLxml`**：使用 lxml 的高性能版本
- **`GraphMLReader`**：负责反序列化

#### 属性传递机制

GraphML 使用**先声明、后引用**的属性机制：

1. **Key 声明阶段**：在 XML 头部声明所有属性的元数据
   - `id`：属性标识符（如 `d0`, `d1`）
   - `for`：作用域（`graph`/`node`/`edge`/`all`）
   - `attr.name`：属性名
   - `attr.type`：数据类型（`int`/`float`/`string`/`boolean`等）
   - `default`：默认值（可选）

2. **数据引用阶段**：在具体元素中引用已声明的 key

```xml
<!-- Key 声明 -->
<key id="d0" for="node" attr.name="color" attr.type="string">
  <default>blue</default>
</key>
<key id="d1" for="edge" attr.name="weight" attr.type="float"/>

<!-- 数据引用 -->
<node id="A">
  <data key="d0">red</data>  <!-- 覆盖默认值 -->
</node>
<edge source="A" target="B">
  <data key="d1">3.5</data>
</edge>
```

#### 类型系统

```python
# graphml.py:396-434
types = [
    (int, "integer"),
    (str, "yfiles"),
    (str, "string"),
    (int, "int"),
    (int, "long"),
    (float, "float"),
    (float, "double"),
    (bool, "boolean"),
]
# 还支持 numpy 类型（如果可用）
```

#### 属性保真能力

| 属性级别 | 支持情况 | 实现位置 |
|---------|---------|---------|
| 图级属性 | ✅ 完整支持 | `add_graph_element()` 中处理 `G.graph` |
| 节点属性 | ✅ 完整支持 | `add_nodes()` → `add_attributes()` |
| 边属性 | ✅ 完整支持 | `add_edges()` → `add_attributes()` |
| 默认值 | ✅ 支持 | 通过 `<default>` 子元素 |

**特殊处理**：
- yEd 扩展支持（`yfiles` 命名空间）
- 类型推断：`infer_numeric_types` 参数可自动统一数值类型
- MultiGraph 支持：将 edge id 用作 edge key

---

### 2.2 GML 格式 (`gml.py`)

**特点**：简洁的键值对层次结构，可读性好，适合数据交换。

#### GML 语法结构

```
graph [
  directed 0
  node [
    id 0
    label "A"
    color "red"
  ]
  node [
    id 1
    label "B"
  ]
  edge [
    source 0
    target 1
    weight 3.5
  ]
]
```

#### 核心函数

- **`generate_gml()`**：生成 GML 格式行
- **`write_gml()`**：写入文件
- **`parse_gml()`**：解析字符串/迭代器
- **`read_gml()`**：从文件读取
- **`literal_stringizer()` / `literal_destringizer()`**：复杂类型的序列化辅助

#### Token 解析系统

GML 使用自定义的词法分析器：

```python
# gml.py:302-312
patterns = [
    r"[A-Za-z][0-9A-Za-z_]*\b",  # keys (标识符)
    r"[+-]?(?:[0-9]*\.[0-9]+|[0-9]+\.[0-9]*|INF)(?:[Ee][+-]?[0-9]+)?",  # reals
    r"[+-]?[0-9]+",  # ints
    r'".*?"',  # strings
    r"\[",  # dict start
    r"\]",  # dict end
    r"#.*$|\s+",  # comments and whitespaces
]
```

#### 属性传递机制

GML 使用**嵌套字典直接映射**的方式：

| GML 结构 | NetworkX 对应 |
|---------|--------------|
| `graph [...]` | `G.graph` 字典 |
| `node [id X label Y ...]` | 节点属性字典 |
| `edge [source X target Y ...]` | 边属性字典 |

**特殊处理**：
- 节点 id 重映射：GML 要求整数 id，NetworkX 允许任意 hashable 节点
- `stringizer`/`destringizer` 机制：支持复杂 Python 类型（如 list, dict, tuple）
- 布尔值用 `1`/`0` 表示

#### 属性保真能力

| 属性级别 | 支持情况 | 说明 |
|---------|---------|------|
| 图级属性 | ✅ 完整支持 | 除保留关键字 `directed`, `multigraph`, `node`, `edge` |
| 节点属性 | ✅ 完整支持 | 除保留关键字 `id`, `label` |
| 边属性 | ✅ 完整支持 | 除保留关键字 `source`, `target`, `key`（MultiGraph）|

**保留关键字**（这些属性名会被忽略，因为用于结构编码）：
- 图级：`directed`, `multigraph`, `node`, `edge`
- 节点：`id`, `label`
- 边：`source`, `target`, `key`（MultiGraph）

---

### 2.3 GEXF 格式 (`gexf.py`)

**特点**：支持动态图（时间属性）、可视化属性、多图，Gephi 首选格式。

#### 核心类设计

- **`GEXF` (基类)**：版本管理、类型系统
- **`GEXFWriter`**：序列化
- **`GEXFReader`**：反序列化

#### 支持的版本

- `1.1draft`
- `1.2draft`（默认）
- `1.3`

#### 属性传递机制

GEXF 使用** attributes 声明 + attvalues 赋值**的两层机制：

```xml
<!-- 属性声明（类型元数据） -->
<attributes class="node" mode="static">
  <attribute id="0" title="color" type="string">
    <default>blue</default>
  </attribute>
</attributes>

<!-- 属性赋值（具体数据） -->
<node id="A" label="Node A">
  <attvalues>
    <attvalue for="0" value="red"/>
  </attvalues>
</node>
```

#### 动态图支持

GEXF 独有的特性：

```python
# 静态属性：直接值
attr[title] = value

# 动态属性：(value, start, end) 三元组列表
attr[title] = [(value1, start1, end1), (value2, start2, end2), ...]
```

#### 可视化属性（viz 命名空间）

- **color**：`{r, g, b, a}`
- **size**：节点大小
- **position**：`{x, y, z}` 坐标
- **shape**：节点形状
- **thickness**：边粗细

#### 属性保真能力

| 属性级别 | 支持情况 | 特殊特性 |
|---------|---------|---------|
| 图级属性 | ✅ 完整支持 | `name`, `mode`, `start`, `end`, `description`, `keywords` |
| 节点属性 | ✅ 完整支持 | 动态属性、viz 属性、`id`, `label`, `pid`, `parents`, `spells` |
| 边属性 | ✅ 完整支持 | 动态属性、viz 属性、`id`, `label`, `weight`, `type`, `spells` |

**保留关键字**：
- 节点：`id`, `label`, `pid`, `start`, `end`, `parents`, `slices`, `spells`, `viz`
- 边：`id`, `label`, `weight`, `type`, `start`, `end`, `slices`, `spells`, `viz`

---

### 2.4 JSON 系列格式 (`json_graph/`)

**特点**：与 Web 前端无缝集成，适合 d3.js、Cytoscape 等可视化库。

#### 四种 JSON 格式

| 格式 | 模块 | 适用场景 |
|------|------|---------|
| Node-Link | `node_link.py` | 通用图结构，d3.js force layout |
| Adjacency | `adjacency.py` | 邻接矩阵视角 |
| Tree | `tree.py` | 树状结构，d3.js hierarchy |
| Cytoscape | `cytoscape.py` | Cytoscape.js 可视化 |

#### Node-Link 格式详解（最常用）

**数据结构**：
```python
{
    "directed": bool,
    "multigraph": bool,
    "graph": {...},           // 图级属性
    "nodes": [
        {"id": node1, ...},   // 节点属性
        {"id": node2, ...}
    ],
    "edges": [
        {"source": u, "target": v, ...},  // 边属性
        {"source": u, "target": v, "key": k, ...}  // MultiGraph
    ]
}
```

**序列化核心代码** (`node_link.py:126-139`)：
```python
data = {
    "directed": G.is_directed(),
    "multigraph": multigraph,
    "graph": G.graph,  # 直接复制图属性
    nodes: [{**G.nodes[n], name: n} for n in G],  # 节点属性展开
}
# 边属性展开
if multigraph:
    data[edges] = [
        {**d, source: u, target: v, key: k}
        for u, v, k, d in G.edges(keys=True, data=True)
    ]
else:
    data[edges] = [{**d, source: u, target: v} for u, v, d in G.edges(data=True)]
```

#### 属性保真能力

| 属性级别 | Node-Link | Adjacency | Tree | Cytoscape |
|---------|-----------|-----------|------|-----------|
| 图级属性 | ✅ | ✅ | ✅ | ✅ |
| 节点属性 | ✅ | ✅（仅源节点） | ✅ | ✅ |
| 边属性 | ✅ | ✅ | ❌ | ✅ |

**注意**：JSON 格式的**类型信息会丢失**，因为 JSON 只有 `string`, `number`, `boolean`, `null`, `array`, `object` 六种类型。例如 Python 的 `tuple` 会变成 `list`，`int` 和 `float` 的边界可能模糊。

---

### 2.5 Edge List 格式 (`edgelist.py`)

**特点**：最简单的文本格式，适合快速导出/导入边数据。

#### 三种数据格式

1. **无数据**：`node1 node2`
2. **字典格式**：`node1 node2 {'weight': 3, 'color': 'red'}`
3. **任意数据列**：`node1 node2 3 red`

#### 属性保真能力

| 属性级别 | 支持情况 | 说明 |
|---------|---------|------|
| 图级属性 | ❌ 不支持 | 完全丢失 |
| 节点属性 | ❌ 不支持 | 无法表示孤立节点 |
| 边属性 | ⚠️ 部分支持 | 需要显式指定 `data` 参数 |

**限制**：
- 孤立节点无法表示（除非有自环）
- 节点和图属性完全丢失
- MultiGraph 的 edge key 无法表示

---

### 2.6 Adjacency List 格式 (`adjlist.py`)

**特点**：适合无属性的大图，存储效率高。

#### 格式示例

```
# 注释行
A B C    # 节点 A 连接到 B 和 C
D E      # 节点 D 连接到 E
```

#### 属性保真能力

| 属性级别 | 支持情况 |
|---------|---------|
| 图级属性 | ❌ 不支持 |
| 节点属性 | ❌ 不支持 |
| 边属性 | ❌ 不支持 |

**注意**：只保存拓扑结构，所有属性全部丢失。适合仅需要连通性信息的场景。

---

## 三、属性传递机制对比总结

### 3.1 三级属性支持矩阵

| 格式 | 图级属性 | 节点属性 | 边属性 | 默认值 | 动态属性 |
|------|:--------:|:--------:|:------:|:------:|:--------:|
| **GraphML** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **GML** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **GEXF** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **JSON (Node-Link)** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Edge List** | ❌ | ❌ | ⚠️ | ❌ | ❌ |
| **Adjacency List** | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Pajek** | ⚠️ | ⚠️ | ⚠️ | ❌ | ❌ |

### 3.2 类型保真度对比

| 格式 | 类型系统 | 类型保真 | 复杂类型支持 |
|------|---------|---------|-------------|
| **GraphML** | 强类型（XML Schema） | ✅ 高 | 基础类型 |
| **GML** | 弱类型 + stringizer | ⚠️ 中 | 需要 stringizer |
| **GEXF** | 强类型（XML Schema） | ✅ 高 | 基础类型 |
| **JSON** | 动态类型 | ❌ 低 | dict/list 原生支持 |
| **Edge List** | 无类型/字典字面量 | ❌ 低 | 需要 literal_eval |

### 3.3 格式选择决策树

```
需要完整保真（所有属性+类型）？
├── 是 → 支持可视化？
│       ├── 是 → GEXF（支持 viz 属性）
│       └── 否 → GraphML 或 GML
│           ├── 需要跨语言标准 → GraphML
│           └── 需要 Python 复杂类型 → GML（用 stringizer）
└── 否 → 用于 Web 可视化？
        ├── 是 → JSON 系列
        │   ├── 通用图 → Node-Link
        │   ├── 树结构 → Tree
        │   └── Cytoscape → Cytoscape JSON
        └── 否 → 仅拓扑结构？
            ├── 是 → Adjacency List（最紧凑）
            └── 否 → 需要边数据？
                ├── 是 → Edge List
                └── 否 → 重新考虑需求
```

---

## 四、关键实现模式

### 4.1 Writer/Reader 类模式

复杂格式（GraphML, GEXF）采用面向对象的设计：

```
┌─────────────────┐     ┌─────────────────┐
│  FormatWriter   │     │  FormatReader   │
├─────────────────┤     ├─────────────────┤
│ + add_graph()   │     │ + __call__()    │
│ + add_nodes()   │     │ + make_graph()  │
│ + add_edges()   │     │ + add_node()    │
│ + add_attributes()│    │ + add_edge()    │
│ + write()       │     │ + decode_*()    │
└─────────────────┘     └─────────────────┘
```

### 4.2 文件处理装饰器

所有 `read_*` 和 `write_*` 函数都使用 `@open_file` 装饰器：

```python
@open_file(1, mode="wb")  # 第 1 个参数是文件路径
def write_graphml(G, path, ...):
    # path 已经是打开的文件对象
    ...
```

**特性**：
- 自动处理压缩文件（`.gz`, `.bz2`）
- 统一编码处理
- 支持文件路径字符串或文件对象

### 4.3 类型转换模式

#### 数值类型映射

```python
# 通用模式：Python 类型 ↔ 格式类型
self.xml_type = {
    int: "int",
    float: "float",
    bool: "boolean",
    str: "string",
    # numpy 类型扩展
    np.int32: "int",
    np.float64: "float",
}
self.python_type = dict(reversed(a) for a in types)  # 反向映射
```

#### 布尔值特殊处理

不同格式对布尔值的表示不同：
- GraphML/GEXF：`"true"` / `"false"` 字符串
- GML：`1` / `0` 整数
- JSON：原生 `true` / `false`

### 4.4 MultiGraph 处理

| 格式 | Edge Key 处理 |
|------|--------------|
| GraphML | 使用 `<edge id="...">` 属性 |
| GML | 使用 `key` 字段 |
| GEXF | 使用 `<edge id="...">` 属性 |
| JSON Node-Link | 使用 `"key"` 字段 |
| Edge List | ❌ 不支持 |

---

## 五、性能与适用场景

### 5.1 性能特征

| 格式 | 序列化速度 | 反序列化速度 | 存储体积 |
|------|:----------:|:------------:|:--------:|
| **Graph6/Sparse6** | ⚡ 极快 | ⚡ 极快 | 💾 极小 |
| **Adjacency List** | ⚡ 极快 | ⚡ 极快 | 💾 小 |
| **Edge List** | ⚡ 快 | ⚡ 快 | 💾 小 |
| **JSON** | ⚡ 快 | ⚡ 快 | 💾 中 |
| **GML** | 🐢 中 | 🐢 中 | 💾 中 |
| **GraphML (lxml)** | 🐢 中 | 🐢 中 | 💾 大 |
| **GraphML (xml)** | 🐌 慢 | 🐌 慢 | 💾 大 |
| **GEXF** | 🐢 中 | 🐢 中 | 💾 大 |

### 5.2 推荐使用场景

| 场景 | 推荐格式 | 理由 |
|------|---------|------|
| 数据持久化（完整保真） | GraphML / GEXF | 标准格式，类型安全 |
| 与 Gephi 交互 | GEXF | 原生支持，可视化属性 |
| Web 前端可视化 | JSON Node-Link | d3.js 原生支持 |
| 大规模无属性图 | Graph6 / Sparse6 | 极紧凑，极快 |
| 快速边数据交换 | Edge List | 简单，人类可读 |
| Python 内部交换 | GML + literal_stringizer | 支持任意 Python 类型 |
| 跨语言交换 | GraphML | 工业标准，多语言支持 |

---

## 六、代码位置速查

| 功能 | 文件位置 | 关键函数/类 |
|------|---------|------------|
| GraphML 读写 | `networkx/readwrite/graphml.py` | `GraphMLWriter`, `GraphMLReader` |
| GML 读写 | `networkx/readwrite/gml.py` | `generate_gml()`, `parse_gml_lines()` |
| GEXF 读写 | `networkx/readwrite/gexf.py` | `GEXFWriter`, `GEXFReader` |
| JSON Node-Link | `networkx/readwrite/json_graph/node_link.py` | `node_link_data()`, `node_link_graph()` |
| Edge List | `networkx/readwrite/edgelist.py` | `generate_edgelist()`, `parse_edgelist()` |
| Adjacency List | `networkx/readwrite/adjlist.py` | `generate_adjlist()`, `parse_adjlist()` |
| 统一导出 | `networkx/readwrite/__init__.py` | 所有 `read_*`/`write_*` |

---

## 七、注意事项与局限性

### 7.1 常见陷阱

1. **GraphML 安全性警告**：
   > "This parser uses the standard xml library present in Python, which is insecure - see xml module for additional information. Only parse GraphML files you trust."

2. **保留关键字冲突**：
   - 不同格式有不同的保留关键字（如 GML 的 `id`, `label`）
   - 这些属性名在序列化时会被**忽略**

3. **类型信息丢失**：
   - JSON 格式会丢失 Python 特有类型（如 tuple, set）
   - Edge List 的字典格式需要 `literal_eval` 才能恢复

4. **孤立节点问题**：
   - Edge List 无法表示没有边的孤立节点
   - 除非节点有自环

### 7.2 不支持的特性

| 格式 | 不支持的特性 |
|------|-------------|
| GraphML | 混合图（有向+无向边混合）、超边、嵌套图、ports |
| GEXF | 混合图、多图（单个文件多个图） |
| GML | 无本质限制，但非标准扩展可能不兼容 |
| JSON | 循环引用、非 JSON 原生类型 |

### 7.3 版本兼容性

- **GraphML**：遵循 http://graphml.graphdrawing.org/ 规范
- **GEXF**：支持 1.1draft, 1.2draft, 1.3 版本
- **GML**：无正式版本号，实现较为宽松

---

## 八、总结

NetworkX 的序列化架构展现了优秀的**模块化设计**和**统一 API 模式**：

1. **模块隔离**：每种格式独立实现，互不干扰
2. **API 一致性**：`read_*`/`write_*`/`parse_*`/`generate_*` 模式
3. **功能分层**：
   - **功能完整层**：GraphML, GML, GEXF, JSON（全属性支持）
   - **功能受限层**：Edge List（仅边属性）
   - **拓扑仅层层**：Adjacency List, Graph6, Sparse6（无属性）

**选择建议**：
- 生产环境数据持久化：**GraphML** 或 **GEXF**
- 与 Python 生态交互：**GML**（配合 `literal_stringizer`）
- Web 应用：**JSON Node-Link**
- 大规模图数据：**Graph6** / **Sparse6**（仅拓扑）
- 简单数据交换：**Edge List**
