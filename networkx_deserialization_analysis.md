# NetworkX 反序列化阶段属性恢复机制深度分析

## 一、反序列化整体架构

NetworkX 的反序列化遵循**统一的两阶段模式**：

```
持久化数据 → 解析器 (Parser) → 中间表示 → 图构建器 (Builder) → NetworkX Graph
```

### 1.1 反序列化入口函数模式

所有格式都提供相似的 API：

| 函数 | 功能 | 输入 |
|------|------|------|
| `read_*()` | 从文件读取 | 文件路径或文件对象 |
| `parse_*()` | 从字符串/迭代器解析 | 字符串或行迭代器 |
| `generate_*()` | 序列化（生成字符串） | 图对象 |

### 1.2 反序列化核心流程

```
1. 输入处理
   ├── 文件打开（@open_file 装饰器）
   ├── 编码处理
   └── 压缩文件自动解压（.gz, .bz2）

2. 语法解析
   ├── XML 解析（GraphML, GEXF）
   ├── 自定义词法分析器（GML）
   ├── 直接字典访问（JSON）
   └── 文本分割（EdgeList, AdjacencyList）

3. 属性恢复
   ├── 图级属性 → G.graph
   ├── 节点属性 → G.nodes[node]
   └── 边属性 → G.edges[u, v]

4. 图类型决策
   ├── 有向/无向检测
   ├── MultiGraph 检测（重复边）
   └── 最终图类型确定

5. 后处理
   ├── 节点重命名（GML label 映射）
   ├── 边 ID 处理
   └── 默认值存储
```

---

## 二、各格式属性恢复机制详解

### 2.1 GraphML 格式（`graphml.py:839-1053`）

#### 核心类：`GraphMLReader`

**初始化参数**：
- `node_type`：节点 ID 转换类型（默认 `str`）
- `edge_key_type`：边 key 转换类型（默认 `int`）
- `force_multigraph`：强制返回 MultiGraph

#### 反序列化执行流程

```
__call__(path/string)
    │
    ├── 1. XML 解析 → ElementTree
    │
    ├── 2. find_graphml_keys() → (keys, defaults)
    │       ├── 提取所有 <key> 元素
    │       ├── 构建类型映射：key_id → {name, type, for}
    │       └── 提取默认值到 defaults 字典
    │
    └── 3. 遍历 <graph> 元素
            └── make_graph(graph_xml, keys, defaults)
```

#### 属性恢复的三个层次

##### 层次 1：图级属性恢复

```python
# graphml.py:862-902
def make_graph(self, graph_xml, graphml_keys, defaults, G=None):
    # ... 图类型初始化 ...
    
    # 默认值存储（注意：不自动应用到节点/边）
    G.graph["node_default"] = {}
    G.graph["edge_default"] = {}
    for key_id, value in defaults.items():
        key_for = graphml_keys[key_id]["for"]
        name = graphml_keys[key_id]["name"]
        python_type = graphml_keys[key_id]["type"]
        if key_for == "node":
            G.graph["node_default"].update({name: python_type(value)})
        if key_for == "edge":
            G.graph["edge_default"].update({name: python_type(value)})
    
    # ... 添加节点和边 ...
    
    # 图级属性恢复
    data = self.decode_data_elements(graphml_keys, graph_xml)
    G.graph.update(data)  # 合并到 G.graph
```

**关键点**：
- GraphML 的默认值**不会自动应用**到节点或边
- 默认值存储在 `G.graph["node_default"]` 和 `G.graph["edge_default"]`
- 用户需要手动应用（文档中有示例代码）

##### 层次 2：节点属性恢复

```python
# graphml.py:904-918
def add_node(self, G, node_xml, graphml_keys, defaults):
    # 1. 提取节点 ID 并转换类型
    node_id = self.node_type(node_xml.get("id"))
    
    # 2. 解码所有 <data> 元素
    data = self.decode_data_elements(graphml_keys, node_xml)
    
    # 3. 添加节点到图
    G.add_node(node_id, **data)  # 所有属性通过关键字参数传入
```

##### 层次 3：边属性恢复

```python
# graphml.py:920-959
def add_edge(self, G, edge_element, graphml_keys):
    # 1. 提取 source/target 并转换类型
    source = self.node_type(edge_element.get("source"))
    target = self.node_type(edge_element.get("target"))
    
    # 2. 解码边属性
    data = self.decode_data_elements(graphml_keys, edge_element)
    
    # 3. 处理 edge id（MultiGraph 的 key）
    edge_id = edge_element.get("id")
    if edge_id:
        self.edge_ids[source, target] = edge_id
        try:
            edge_id = self.edge_key_type(edge_id)
        except ValueError:
            pass  # 转换失败则保持字符串
    
    # 4. MultiGraph 检测
    if G.has_edge(source, target):
        self.multigraph = True  # 标记为 MultiGraph
    
    # 5. 添加边
    G.add_edges_from([(source, target, edge_id, data)])
```

#### 核心属性解码函数：`decode_data_elements()`

```python
# graphml.py:961-1018
def decode_data_elements(self, graphml_keys, obj_xml):
    data = {}
    for data_element in obj_xml.findall(f"{{{self.NS_GRAPHML}}}data"):
        # 1. 获取 key 引用
        key = data_element.get("key")
        
        # 2. 查找预先声明的类型信息
        try:
            data_name = graphml_keys[key]["name"]
            data_type = graphml_keys[key]["type"]
        except KeyError as err:
            raise nx.NetworkXError(f"Bad GraphML data: no key {key}") from err
        
        # 3. 类型转换
        text = data_element.text
        if text is not None and len(list(data_element)) == 0:
            if data_type is bool:
                # 布尔值特殊处理：支持 true/false, True/False, 1/0
                data[data_name] = self.convert_bool[text.lower()]
            else:
                # 使用构造函数转换：int(), float(), str() 等
                data[data_name] = data_type(text)
        elif len(list(data_element)) > 0:
            # 处理 yFiles 扩展（有子元素）
            self._extract_yfiles_data(data_element, data)
        elif text is None:
            # 空值处理
            data[data_name] = ""
    
    return data
```

#### 图类型决策机制

```python
# graphml.py:862-902
def make_graph(self, graph_xml, graphml_keys, defaults, G=None):
    # 阶段 1：根据 edgedefault 初始化图类型
    edgedefault = graph_xml.get("edgedefault", None)
    if G is None:
        if edgedefault == "directed":
            G = nx.MultiDiGraph()  # 总是先创建 Multi* 图
        else:
            G = nx.MultiGraph()
    
    # 阶段 2：添加边时检测重复边
    # 在 add_edge() 中：
    # if G.has_edge(source, target):
    #     self.multigraph = True
    
    # 阶段 3：最终决策
    if self.multigraph:
        return G  # 保持 MultiGraph
    
    # 转换为普通图
    G = nx.DiGraph(G) if G.is_directed() else nx.Graph(G)
    
    # 非 MultiGraph 时，将 edge id 作为属性添加
    nx.set_edge_attributes(G, values=self.edge_ids, name="id")
    return G
```

**决策流程**：
```
开始
  │
  ├── 读取 edgedefault 属性
  │       ├── "directed" → MultiDiGraph
  │       └── 其他 → MultiGraph
  │
  ├── 添加所有节点和边
  │       └── 发现重复边 → self.multigraph = True
  │
  └── 最终决策
          ├── self.multigraph == True → 返回 Multi* 图
          │
          └── self.multigraph == False
                  ├── 有向 → DiGraph
                  └── 无向 → Graph
```

---

### 2.2 GML 格式（`gml.py:298-525`）

#### 核心函数：`parse_gml_lines()`

GML 使用**自定义词法分析器 + 递归下降解析器**的组合策略。

#### 阶段 1：词法分析（Tokenization）

```python
# gml.py:301-363
def tokenize():
    patterns = [
        r"[A-Za-z][0-9A-Za-z_]*\b",  # 0: KEYS (标识符)
        r"[+-]?(?:[0-9]*\.[0-9]+|[0-9]+\.[0-9]*|INF)(?:[Ee][+-]?[0-9]+)?",  # 1: REALS
        r"[+-]?[0-9]+",  # 2: INTS
        r'".*?"',  # 3: STRINGS
        r"\[",  # 4: DICT_START
        r"\]",  # 5: DICT_END
        r"#.*$|\s+",  # 6: COMMENT_WHITESPACE (忽略)
    ]
    
    for line in lines:
        # 处理多行字符串
        if multilines:
            # ... 拼接多行 ...
        
        # 逐字符匹配
        while pos < length:
            match = tokens.match(line, pos)
            if match is None:
                raise nx.NetworkXError(f"cannot tokenize ...")
            
            # 根据匹配模式分类
            if i == 0:    # KEYS
                value = group.rstrip()
            elif i == 1:  # REALS → 直接转为 float
                value = float(group)
            elif i == 2:  # INTS → 直接转为 int
                value = int(group)
            else:  # STRINGS
                value = group
            
            if i != 6:  # 非注释/空白
                yield Token(Pattern(i), value, lineno + 1, pos + 1)
```

**类型推断在词法阶段完成**：
- 匹配到浮点数模式 → `float(value)`
- 匹配到整数模式 → `int(value)`
- 匹配到字符串模式 → 保持字符串，后续用 `destringizer` 处理

#### 阶段 2：语法解析（Parsing）

```python
# gml.py:377-442
def parse_kv(curr_token):
    dct = defaultdict(list)  # 支持重复键
    while curr_token.category == Pattern.KEYS:
        key = curr_token.value
        curr_token = next(tokens)
        category = curr_token.category
        
        # 根据值类型处理
        if category == Pattern.REALS or category == Pattern.INTS:
            # 数值类型：直接使用词法分析的结果
            value = curr_token.value
            curr_token = next(tokens)
        
        elif category == Pattern.STRINGS:
            # 字符串类型
            value = unescape(curr_token.value[1:-1])  # 去掉引号并转义
            
            # destringizer 机制：解析复杂类型字符串
            if destringizer:
                try:
                    value = destringizer(value)
                except ValueError:
                    pass  # 解析失败则保持字符串
            
            # 特殊处理：空列表和元组
            if value == "()":
                value = ()
            if value == "[]":
                value = []
            
            curr_token = next(tokens)
        
        elif category == Pattern.DICT_START:
            # 嵌套字典：递归解析
            curr_token, value = parse_dict(curr_token)
        
        else:
            # 特殊情况：id/label/source/target 允许非标准格式
            if key in ("id", "label", "source", "target"):
                value = unescape(str(curr_token.value))
                # ...
            elif curr_token.value in {"NAN", "INF"}:
                value = float(curr_token.value)
            else:
                unexpected(curr_token, "an int, float, string or '['")
        
        dct[key].append(value)  # 收集到列表（支持重复键）
    
    # 清理：处理重复键
    def clean_dict_value(value):
        if not isinstance(value, list):
            return value
        if len(value) == 1:
            return value[0]  # 单值：从列表中取出
        if value[0] == LIST_START_VALUE:
            return value[1:]  # 列表标记
        return value  # 多值：保持列表
    
    return curr_token, {k: clean_dict_value(v) for k, v in dct.items()}
```

**`destringizer` 机制详解**：

```python
# 使用示例
>>> from networkx.readwrite.gml import literal_destringizer, literal_stringizer
>>> G = nx.Graph()
>>> G.add_node(1, data=[1, 2, {"a": "b"}])
>>> list(nx.generate_gml(G, stringizer=literal_stringizer))
# 输出中 data 被序列化为字符串
>>> H = nx.parse_gml(lines, destringizer=literal_destringizer)
# 恢复为实际的 list/dict
```

```python
# gml.py:85-110
def literal_destringizer(rep):
    """Convert a Python literal to the value it represents."""
    if isinstance(rep, str):
        orig_rep = rep
        try:
            return literal_eval(rep)  # 使用 ast.literal_eval
        except SyntaxError as err:
            raise ValueError(f"{orig_rep!r} is not a valid Python literal") from err
    else:
        raise ValueError(f"{rep!r} is not a string")
```

**支持的类型**：
- 基础类型：`int`, `float`, `str`, `bool`, `None`
- 容器类型：`list`, `tuple`, `dict`, `set`
- 嵌套结构：任意深度的嵌套

#### 阶段 3：图构建

```python
# gml.py:464-525
# 1. 图类型决策
directed = graph.pop("directed", False)
multigraph = graph.pop("multigraph", False)

if not multigraph:
    G = nx.DiGraph() if directed else nx.Graph()
else:
    G = nx.MultiDiGraph() if directed else nx.MultiGraph()

# 2. 图级属性恢复
graph_attr = {k: v for k, v in graph.items() if k not in ("node", "edge")}
G.graph.update(graph_attr)

# 3. 节点恢复
nodes = graph.get("node", [])
for i, node in enumerate(nodes if isinstance(nodes, list) else [nodes]):
    id = pop_attr(node, "node", "id", i)  # 必需属性
    
    # label 重命名映射
    if label is not None and label != "id":
        node_label = pop_attr(node, "node", label, i)
        mapping[id] = node_label
    
    G.add_node(id, **node)  # 剩余属性作为节点属性

# 4. 边恢复
edges = graph.get("edge", [])
for i, edge in enumerate(edges if isinstance(edges, list) else [edges]):
    source = pop_attr(edge, "edge", "source", i)  # 必需
    target = pop_attr(edge, "edge", "target", i)  # 必需
    
    # 引用检查
    if source not in G:
        raise nx.NetworkXError(f"edge #{i} has undefined source {source!r}")
    if target not in G:
        raise nx.NetworkXError(f"edge #{i} has undefined target {target!r}")
    
    if not multigraph:
        # 普通图：检查重复边
        if not G.has_edge(source, target):
            G.add_edge(source, target, **edge)
        else:
            raise nx.NetworkXError(f"edge #{i} ... is duplicated")
    else:
        # MultiGraph：处理 key
        key = edge.pop("key", None)
        G.add_edge(source, target, key, **edge)

# 5. 后处理：节点重命名
if label is not None and label != "id":
    G = nx.relabel_nodes(G, mapping)
```

#### GML 图类型决策 vs GraphML

| 维度 | GML | GraphML |
|------|-----|---------|
| 决策依据 | `directed` 和 `multigraph` 字段 | `edgedefault` 属性 + 重复边检测 |
| 初始类型 | 根据 `directed` 和 `multigraph` 直接创建 | 总是先创建 Multi* 图 |
| MultiGraph 检测 | 依赖 `multigraph` 字段 | 自动检测重复边 |
| 灵活性 | 较低（需要显式声明） | 较高（自动推断） |

---

### 2.3 GEXF 格式（`gexf.py:749-1087`）

#### 核心类：`GEXFReader`

**特点**：支持**动态图**和**可视化属性**，版本自动探测。

#### 阶段 1：版本自动探测

```python
# gexf.py:759-772
def __call__(self, stream):
    self.xml = ElementTree(file=stream)
    
    # 先尝试默认版本
    meta = self.xml.find(f"{{{self.NS_GEXF}}}meta")
    g = self.xml.find(f"{{{self.NS_GEXF}}}graph")
    if g is not None:
        return self.make_graph(g, meta_xml=meta)
    
    # 失败后遍历所有支持版本
    for version in self.versions:
        self.set_version(version)
        meta = self.xml.find(f"{{{self.NS_GEXF}}}meta")
        g = self.xml.find(f"{{{self.NS_GEXF}}}graph")
        if g is not None:
            return self.make_graph(g, meta_xml=meta)
    
    raise nx.NetworkXError("No <graph> element in GEXF file.")
```

**支持的版本**：
- `1.1draft`
- `1.2draft`（默认）
- `1.3`

#### 阶段 2：图级属性恢复

```python
# gexf.py:774-860
def make_graph(self, graph_xml, meta_xml=None):
    # 1. 初始图类型
    edgedefault = graph_xml.get("defaultedgetype", None)
    if edgedefault == "directed":
        G = nx.MultiDiGraph()
    else:
        G = nx.MultiGraph()
    
    # 2. meta 元数据恢复
    if meta_xml is not None:
        description = meta_xml.find(f"{{{self.NS_GEXF}}}description")
        if description is not None:
            G.graph["description"] = description.text
        keywords = meta_xml.find(f"{{{self.NS_GEXF}}}keywords")
        if keywords is not None:
            G.graph["keywords"] = keywords.text
    
    # 3. graph 元素属性恢复
    graph_name = graph_xml.get("name", "")
    if graph_name != "":
        G.graph["name"] = graph_name
    graph_start = graph_xml.get("start")
    if graph_start is not None:
        G.graph["start"] = graph_start
    graph_end = graph_xml.get("end")
    if graph_end is not None:
        G.graph["end"] = graph_end
    graph_mode = graph_xml.get("mode", "")
    if graph_mode == "dynamic":
        G.graph["mode"] = "dynamic"
    else:
        G.graph["mode"] = "static"
    
    # 4. 时间格式
    self.timeformat = graph_xml.get("timeformat")
    if self.timeformat == "date":
        self.timeformat = "string"
    
    # 5. 属性声明解析
    attributes_elements = graph_xml.findall(f"{{{self.NS_GEXF}}}attributes")
    for a in attributes_elements:
        attr_class = a.get("class")
        if attr_class == "node":
            na, nd = self.find_gexf_attributes(a)
            G.graph["node_default"] = nd
        elif attr_class == "edge":
            ea, ed = self.find_gexf_attributes(a)
            G.graph["edge_default"] = ed
    
    # 6. Gephi 0.7beta 兼容性 hack
    ea = {"weight": {"type": "double", "mode": "static", "title": "weight"}}
    edge_attr.update(ea)
```

#### 阶段 3：节点属性恢复（多层叠加）

```python
# gexf.py:862-951
def add_node(self, G, node_xml, node_attr, node_pid=None):
    # 多层属性叠加
    data = {}
    
    # 层 1：普通属性（<attvalues>）
    data = self.decode_attr_elements(node_attr, node_xml)
    
    # 层 2：父子关系（<parents>）
    data = self.add_parents(data, node_xml)
    
    # 层 3：时间切片（版本相关）
    if self.VERSION == "1.1":
        data = self.add_slices(data, node_xml)  # <slices>/<slice>
    else:
        data = self.add_spells(data, node_xml)  # <spells>/<spell>
    
    # 层 4：可视化属性（<viz:*>）
    data = self.add_viz(data, node_xml)
    
    # 层 5：开始/结束时间（属性）
    data = self.add_start_end(data, node_xml)
    
    # 节点 ID 和标签
    node_id = node_xml.get("id")
    if self.node_type is not None:
        node_id = self.node_type(node_id)
    
    node_label = node_xml.get("label")
    data["label"] = node_label
    
    # 父节点 ID
    node_pid = node_xml.get("pid", node_pid)
    if node_pid is not None:
        data["pid"] = node_pid
    
    # 递归处理嵌套节点
    subnodes = node_xml.find(f"{{{self.NS_GEXF}}}nodes")
    if subnodes is not None:
        for node_xml in subnodes.findall(f"{{{self.NS_GEXF}}}node"):
            self.add_node(G, node_xml, node_attr, node_pid=node_id)
    
    G.add_node(node_id, **data)
```

#### 可视化属性解析

```python
# gexf.py:908-951
def add_viz(self, data, node_xml):
    viz = {}
    
    # 颜色
    color = node_xml.find(f"{{{self.NS_VIZ}}}color")
    if color is not None:
        if self.VERSION == "1.1":
            viz["color"] = {
                "r": int(color.get("r")),
                "g": int(color.get("g")),
                "b": int(color.get("b")),
            }
        else:
            viz["color"] = {
                "r": int(color.get("r")),
                "g": int(color.get("g")),
                "b": int(color.get("b")),
                "a": float(color.get("a", 1)),  # 1.2+ 支持透明度
            }
    
    # 大小
    size = node_xml.find(f"{{{self.NS_VIZ}}}size")
    if size is not None:
        viz["size"] = float(size.get("value"))
    
    # 粗细（边）
    thickness = node_xml.find(f"{{{self.NS_VIZ}}}thickness")
    if thickness is not None:
        viz["thickness"] = float(thickness.get("value"))
    
    # 形状
    shape = node_xml.find(f"{{{self.NS_VIZ}}}shape")
    if shape is not None:
        viz["shape"] = shape.get("shape")
        if viz["shape"] == "image":
            viz["shape"] = shape.get("uri")  # 图片形状
    
    # 位置
    position = node_xml.find(f"{{{self.NS_VIZ}}}position")
    if position is not None:
        viz["position"] = {
            "x": float(position.get("x", 0)),
            "y": float(position.get("y", 0)),
            "z": float(position.get("z", 0)),
        }
    
    if len(viz) > 0:
        data["viz"] = viz
    return data
```

#### 动态属性解析

```python
# gexf.py:1035-1067
def decode_attr_elements(self, gexf_keys, obj_xml):
    attr = {}
    attr_element = obj_xml.find(f"{{{self.NS_GEXF}}}attvalues")
    if attr_element is not None:
        for a in attr_element.findall(f"{{{self.NS_GEXF}}}attvalue"):
            key = a.get("for")
            try:
                title = gexf_keys[key]["title"]
            except KeyError as err:
                raise nx.NetworkXError(f"No attribute defined for={key}.") from err
            
            atype = gexf_keys[key]["type"]
            value = a.get("value")
            
            # 类型转换
            if atype == "boolean":
                value = self.convert_bool[value]
            else:
                value = self.python_type[atype](value)
            
            # 动态 vs 静态
            if gexf_keys[key]["mode"] == "dynamic":
                # 动态属性：(value, start, end) 三元组列表
                ttype = self.timeformat
                start = self.python_type[ttype](a.get("start"))
                end = self.python_type[ttype](a.get("end"))
                
                if title in attr:
                    attr[title].append((value, start, end))
                else:
                    attr[title] = [(value, start, end)]
            else:
                # 静态属性：直接赋值
                attr[title] = value
    
    return attr
```

**动态属性数据结构**：
```python
# 静态属性
G.nodes[node]["price"] = 100.0

# 动态属性
G.nodes[node]["price"] = [
    (90.0,  "2024-01-01", "2024-06-30"),
    (100.0, "2024-07-01", "2024-12-31"),
    (110.0, "2025-01-01", None),
]
```

#### 边属性恢复

```python
# gexf.py:983-1033
def add_edge(self, G, edge_element, edge_attr):
    # 1. 混合边检查
    edge_direction = edge_element.get("type")
    if G.is_directed() and edge_direction == "undirected":
        raise nx.NetworkXError("Undirected edge found in directed graph.")
    if (not G.is_directed()) and edge_direction == "directed":
        raise nx.NetworkXError("Directed edge found in undirected graph.")
    
    # 2. 节点引用
    source = edge_element.get("source")
    target = edge_element.get("target")
    if self.node_type is not None:
        source = self.node_type(source)
        target = self.node_type(target)
    
    # 3. 属性叠加
    data = self.decode_attr_elements(edge_attr, edge_element)
    data = self.add_start_end(data, edge_element)
    if self.VERSION == "1.1":
        data = self.add_slices(data, edge_element)
    else:
        data = self.add_spells(data, edge_element)
    
    # 4. 边 ID 和特殊属性
    edge_id = edge_element.get("id")
    if edge_id is not None:
        data["id"] = edge_id
    
    multigraph_key = data.pop("networkx_key", None)
    if multigraph_key is not None:
        edge_id = multigraph_key
    
    weight = edge_element.get("weight")
    if weight is not None:
        data["weight"] = float(weight)
    
    edge_label = edge_element.get("label")
    if edge_label is not None:
        data["label"] = edge_label
    
    # 5. MultiGraph 检测
    if G.has_edge(source, target):
        self.simple_graph = False
    
    # 6. 添加边
    G.add_edge(source, target, key=edge_id, **data)
    
    # 7. mutual 边处理（双向边）
    if edge_direction == "mutual":
        G.add_edge(target, source, key=edge_id, **data)
```

---

### 2.4 JSON 系列格式

#### Node-Link 格式（最常用）

```python
# node_link.py:142-260
def node_link_graph(
    data,
    directed=False,
    multigraph=True,
    *,
    source="source",
    target="target",
    name="id",
    key="key",
    edges="edges",
    nodes="nodes",
):
    # 1. 图类型决策（参数覆盖）
    multigraph = data.get("multigraph", multigraph)
    directed = data.get("directed", directed)
    
    if multigraph:
        graph = nx.MultiGraph()
    else:
        graph = nx.Graph()
    if directed:
        graph = graph.to_directed()
    
    # 2. 图级属性恢复（直接赋值）
    graph.graph = data.get("graph", {})
    
    # 3. 节点属性恢复
    # 列表推导式：除 'id' 外的所有键值对
    for d in data[nodes]:
        node = _to_tuple(d.get(name, next(c)))  # 处理元组节点
        nodedata = {str(k): v for k, v in d.items() if k != name}
        graph.add_node(node, **nodedata)
    
    # 4. 边属性恢复
    for d in data[edges]:
        src = tuple(d[source]) if isinstance(d[source], list) else d[source]
        tgt = tuple(d[target]) if isinstance(d[target], list) else d[target]
        
        if not multigraph:
            edgedata = {str(k): v for k, v in d.items() 
                       if k != source and k != target}
            graph.add_edge(src, tgt, **edgedata)
        else:
            ky = d.get(key, None)
            edgedata = {
                str(k): v
                for k, v in d.items()
                if k != source and k != target and k != key
            }
            graph.add_edge(src, tgt, ky, **edgedata)
    
    return graph
```

**Node-Link 数据结构**：
```python
{
    "directed": bool,
    "multigraph": bool,
    "graph": {...},           // 图级属性
    "nodes": [
        {"id": node1, "attr1": v1, ...},
        {"id": node2, "attr2": v2, ...},
    ],
    "edges": [
        {"source": u, "target": v, "attr3": v3, ...},           // 普通图
        {"source": u, "target": v, "key": k, "attr3": v3, ...}, // MultiGraph
    ]
}
```

#### Adjacency 格式

```python
# adjacency.py:85-156
def adjacency_graph(data, directed=False, multigraph=True, attrs=_attrs):
    # 类似的图类型决策
    multigraph = data.get("multigraph", multigraph)
    directed = data.get("directed", directed)
    # ...
    
    # 图级属性：注意！存储为列表，需要转换
    graph.graph = dict(data.get("graph", []))
    
    # 节点恢复：建立索引映射
    mapping = []
    for d in data["nodes"]:
        node_data = d.copy()
        node = node_data.pop(id_)
        mapping.append(node)
        graph.add_node(node)
        graph.nodes[node].update(node_data)
    
    # 边恢复：通过索引查找
    for i, d in enumerate(data["adjacency"]):
        source = mapping[i]  # 第 i 个节点
        for tdata in d:
            target_data = tdata.copy()
            target = target_data.pop(id_)
            
            if not multigraph:
                graph.add_edge(source, target)
                graph[source][target].update(target_data)
            else:
                ky = target_data.pop(key, None)
                graph.add_edge(source, target, key=ky)
                graph[source][target][ky].update(target_data)
```

**注意**：Adjacency 格式的图级属性存储方式不同：
```python
# 序列化时
data["graph"] = list(G.graph.items())  # 转为列表

# 反序列化时
graph.graph = dict(data.get("graph", []))  # 转回字典
```

#### Tree 格式（限制最多）

```python
# tree.py:84-135
def tree_graph(data, ident="id", children="children"):
    graph = nx.DiGraph()  # 强制 DiGraph
    
    def add_children(parent, children_):
        for data in children_:
            child = data[ident]
            graph.add_edge(parent, child)  # 边没有属性！
            
            grandchildren = data.get(children, [])
            if grandchildren:
                add_children(child, grandchildren)
            
            # 节点属性：除 ident 和 children 外
            nodedata = {
                str(k): v for k, v in data.items() 
                if k != ident and k != children
            }
            graph.add_node(child, **nodedata)
    
    # 根节点
    root = data[ident]
    children_ = data.get(children, [])
    nodedata = {str(k): v for k, v in data.items() 
               if k != ident and k != children}
    graph.add_node(root, **nodedata)
    add_children(root, children_)
    
    return graph
```

**Tree 格式的严重限制**：
1. 只能表示**有向树**（强制 `DiGraph`）
2. **边属性完全丢失**
3. **图级属性完全丢失**
4. 只能表示单根树

---

### 2.5 EdgeList 格式

```python
# edgelist.py:177-297
def parse_edgelist(
    lines, comments="#", delimiter=None, create_using=None, nodetype=None, data=True
):
    G = nx.empty_graph(0, create_using)
    
    for line in lines:
        # 1. 注释处理
        if comments is not None:
            p = line.find(comments)
            if p >= 0:
                line = line[:p]
            if not line:
                continue
        
        # 2. 分割
        s = line.rstrip("\n").split(delimiter)
        if len(s) < 2:
            continue  # 跳过无效行
        
        u = s.pop(0)
        v = s.pop(0)
        d = s  # 剩余部分作为数据
        
        # 3. 节点类型转换
        if nodetype is not None:
            try:
                u = nodetype(u)
                v = nodetype(v)
            except Exception as err:
                raise TypeError(
                    f"Failed to convert nodes {u},{v} to type {nodetype}."
                ) from err
        
        # 4. 边数据解析（三种模式）
        if len(d) == 0 or data is False:
            # 模式 1：无数据
            edgedata = {}
        
        elif data is True:
            # 模式 2：字典格式（使用 literal_eval）
            try:
                if delimiter == ",":
                    edgedata_str = ",".join(d)
                else:
                    edgedata_str = " ".join(d)
                edgedata = dict(literal_eval(edgedata_str.strip()))
            except Exception as err:
                raise TypeError(
                    f"Failed to convert edge data ({d}) to dictionary."
                ) from err
        
        else:
            # 模式 3：指定列名和类型
            if len(d) != len(data):
                raise IndexError(
                    f"Edge data {d} and data_keys {data} are not the same length"
                )
            edgedata = {}
            for (edge_key, edge_type), edge_value in zip(data, d):
                try:
                    edge_value = edge_type(edge_value)
                except Exception as err:
                    raise TypeError(
                        f"Failed to convert {edge_key} data {edge_value} "
                        f"to type {edge_type}."
                    ) from err
                edgedata.update({edge_key: edge_value})
        
        G.add_edge(u, v, **edgedata)
    
    return G
```

**EdgeList 的三种数据格式**：

| 格式 | 示例 | `data` 参数 |
|------|------|-------------|
| 无数据 | `"1 2"` | `False` 或 `len(d)==0` |
| 字典格式 | `"1 2 {'weight': 3, 'color': 'red'}"` | `True` |
| 列格式 | `"1 2 3 red"` | `[("weight", float), ("color", str)]` |

**EdgeList 的严重限制**：
1. **图级属性完全丢失**
2. **节点属性完全丢失**
3. **孤立节点无法表示**（除非有自环）
4. **MultiGraph 的 edge key 无法表示**

---

### 2.6 AdjacencyList 格式

```python
# adjlist.py:174-243
def parse_adjlist(
    lines, comments="#", delimiter=None, create_using=None, nodetype=None
):
    G = nx.empty_graph(0, create_using)
    
    for line in lines:
        # 注释处理
        p = line.find(comments)
        if p >= 0:
            line = line[:p]
        if not len(line):
            continue
        
        # 分割
        vlist = line.rstrip("\n").split(delimiter)
        u = vlist.pop(0)  # 第一个是源节点
        
        # 节点类型转换
        if nodetype is not None:
            try:
                u = nodetype(u)
            except BaseException as err:
                raise TypeError(
                    f"Failed to convert node ({u}) to type {nodetype}"
                ) from err
        
        G.add_node(u)  # 添加源节点
        
        # 邻居节点类型转换
        if nodetype is not None:
            try:
                vlist = list(map(nodetype, vlist))
            except BaseException as err:
                raise TypeError(
                    f"Failed to convert nodes ({','.join(vlist)}) to type {nodetype}"
                ) from err
        
        # 添加边（无任何属性）
        G.add_edges_from([(u, v) for v in vlist])
    
    return G
```

**AdjacencyList 格式示例**：
```
# 这是注释
A B C   # 节点 A 连接到 B 和 C
D E     # 节点 D 连接到 E
F       # 孤立节点 F（只有源节点，没有邻居）
```

**AdjacencyList 的限制**：
1. **所有属性完全丢失**（图级、节点、边）
2. 孤立节点只能通过"只有源节点没有邻居"的行表示
3. 边的顺序取决于节点的遍历顺序

---

## 三、属性恢复能力对比矩阵

### 3.1 三级属性恢复能力

| 格式 | 图级属性 | 节点属性 | 边属性 | 默认值 | 动态属性 |
|------|:--------:|:--------:|:------:|:------:|:--------:|
| **GraphML** | ✅ | ✅ | ✅ | ✅（存储不自动应用） | ❌ |
| **GML** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **GEXF** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **JSON Node-Link** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **JSON Adjacency** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **JSON Tree** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **EdgeList** | ❌ | ❌ | ⚠️（仅边） | ❌ | ❌ |
| **AdjacencyList** | ❌ | ❌ | ❌ | ❌ | ❌ |

### 3.2 类型保真度对比

| 格式 | 类型系统 | 类型恢复方式 | 复杂类型支持 |
|------|---------|-------------|-------------|
| **GraphML** | 强类型（XML Schema） | 预先声明 + 构造函数转换 | 基础类型 |
| **GML** | 弱类型 + 词法推断 | 词法阶段推断 + destringizer | ✅ `literal_eval` 支持 |
| **GEXF** | 强类型（XML Schema） | 预先声明 + 构造函数转换 | 基础类型 |
| **JSON** | 动态类型 | JSON 原生类型 | `list`, `dict`, `str`, `number` |
| **EdgeList** | 无类型 / 显式指定 | `literal_eval` 或类型构造器 | 列格式支持指定类型 |
| **AdjacencyList** | 无类型 | 无（只存储拓扑） | 不支持 |

### 3.3 图类型决策机制对比

| 格式 | 决策依据 | 是否自动检测 MultiGraph | 初始图类型 |
|------|---------|------------------------|-----------|
| **GraphML** | `edgedefault` + 重复边检测 | ✅ 自动 | 总是 `Multi*` |
| **GML** | `directed` + `multigraph` 字段 | ❌ 依赖声明 | 根据声明直接创建 |
| **GEXF** | `defaultedgetype` + 重复边检测 | ✅ 自动 | 总是 `Multi*` |
| **JSON** | `directed` + `multigraph` 字段 | ❌ 依赖数据中的值 | 根据数据值 |
| **EdgeList** | `create_using` 参数 | ❌ 不检测 | 完全由参数决定 |
| **AdjacencyList** | `create_using` 参数 | ❌ 不检测 | 完全由参数决定 |

---

## 四、异常处理和边界策略深度分析

### 4.1 GraphML 异常处理

#### 异常类型：`nx.NetworkXError`

| 错误场景 | 错误消息 | 代码位置 |
|---------|---------|---------|
| 不支持超边 | `"GraphML reader doesn't support hyperedges"` | `graphml.py:883-884` |
| 不支持 ports | `warnings.warn("GraphML port tag not supported.")` | `graphml.py:907-909` |
| 混合有向/无向边 | `"directed=false edge found in directed graph."` | `graphml.py:927-934` |
| 未定义的 key 引用 | `"Bad GraphML data: no key {key}"` | `graphml.py:969-970` |
| 缺少 `attr.name` | `"Unknown key for id {attr_id}."` | `graphml.py:1035-1036` |
| 缺少 `attr.type` | `warnings.warn("No key type for id ... Using string")` | `graphml.py:1032-1034` |

#### 边界情况处理

1. **不完整的 GraphML 头**：
```python
# graphml.py:294-307
glist = list(reader(path=path))
if len(glist) == 0:
    # 尝试修复：添加默认命名空间
    header = b'<graphml xmlns="http://graphml.graphdrawing.org/xmlns">'
    path.seek(0)
    old_bytes = path.read()
    new_bytes = old_bytes.replace(b"<graphml>", header)
    glist = list(reader(string=new_bytes))
```

2. **节点和边类型转换失败**：
```python
# graphml.py:946-949
try:
    edge_id = self.edge_key_type(edge_id)
except ValueError:  # 转换失败则保持原值
    pass
```

3. **yFiles 扩展处理**：
```python
# graphml.py:981-1016
elif len(list(data_element)) > 0:
    # 有子元素 → 假设是 yFiles 扩展
    # 尝试提取 node_label, shape_type, x, y, label 等
```

### 4.2 GML 异常处理

#### 词法错误

```python
# gml.py:344-346
match = tokens.match(line, pos)
if match is None:
    m = f"cannot tokenize {line[pos:]} at ({lineno + 1}, {pos + 1})"
    raise nx.NetworkXError(m)
```

#### 语法错误

```python
# gml.py:365-370
def unexpected(curr_token, expected):
    category, value, lineno, pos = curr_token
    value = repr(value) if value is not None else "EOF"
    raise nx.NetworkXError(
        f"expected {expected}, found {value} at ({lineno}, {pos})"
    )
```

#### 结构错误

| 错误场景 | 错误消息 |
|---------|---------|
| 无 graph 元素 | `"input contains no graph"` |
| 多图 | `"input contains more than one graph"` |
| 节点 id 重复 | `"node id {id!r} is duplicated"` |
| 节点 label 重复 | `"node label {node_label!r} is duplicated"` |
| 边引用不存在的节点 | `"edge #{i} has undefined source/target {source!r}"` |
| 非 multigraph 重复边 | `"edge #{i} ... is duplicated"` |
| multigraph 重复 key | 提示添加 `"multigraph 1"` |
| 缺少必需属性 | `"{category} #{i} has no {attr!r} attribute"` |

#### 边界情况处理

1. **多行字符串**：
```python
# gml.py:318-338
if multilines:
    multilines.append(line.strip())
    if line[-1] == '"':  # 结束
        line = " ".join(multilines)
        multilines = []
    else:
        continue  # 继续
else:
    if line.count('"') == 1:  # 开始
        multilines = [line.rstrip()]
        continue
```

2. **特殊数值处理**：
```python
# gml.py:419-427
elif curr_token.value in {"NAN", "INF"}:
    # 特殊处理：NAN 和 INF 作为数值
    value = float(curr_token.value)
    curr_token = next(tokens)
```

3. **id/label/source/target 的宽松处理**：
```python
# gml.py:402-418
if key in ("id", "label", "source", "target"):
    # 这些关键字允许非标准格式
    try:
        value = unescape(str(curr_token.value))
        # ...
    except Exception:
        unexpected(...)
```

### 4.3 GEXF 异常处理

| 错误场景 | 错误消息 |
|---------|---------|
| 无 graph 元素 | `"No <graph> element in GEXF file."` |
| 混合有向/无向边 | `"Undirected edge found in directed graph."` |
| 未定义的 attribute | `"No attribute defined for={key}."` |

#### 边界情况处理

1. **版本自动回退**：见 `__call__()` 方法
2. **Gephi 0.7beta 兼容性**：
```python
# gexf.py:834-840
# Hack to handle Gephi0.7beta bug
# 自动添加 weight 属性定义
ea = {"weight": {"type": "double", "mode": "static", "title": "weight"}}
edge_attr.update(ea)
```

3. **嵌套节点递归**：
```python
# gexf.py:890-893
subnodes = node_xml.find(f"{{{self.NS_GEXF}}}nodes")
if subnodes is not None:
    for node_xml in subnodes.findall(f"{{{self.NS_GEXF}}}node"):
        self.add_node(G, node_xml, node_attr, node_pid=node_id)
```

### 4.4 JSON 系列异常处理

JSON 格式的异常处理**相对简单**，主要依赖 Python 内置异常：

| 错误场景 | 异常类型 |
|---------|---------|
| 缺少 `nodes` 键 | `KeyError` |
| 缺少 `edges` 键 | `KeyError` |
| 属性名重复 | `nx.NetworkXError("Attribute names are not unique.")` |
| 非树结构（Tree 格式） | `TypeError("G is not a tree.")` |
| 非有向图（Tree 格式） | `TypeError("G is not directed.")` |
| 非连通图（Tree 格式） | `TypeError("G is not weakly connected.")` |

### 4.5 EdgeList 异常处理

| 错误场景 | 异常类型 | 错误消息 |
|---------|---------|---------|
| 节点类型转换失败 | `TypeError` | `"Failed to convert nodes {u},{v} to type {nodetype}."` |
| 字典解析失败 | `TypeError` | `"Failed to convert edge data ({d}) to dictionary."` |
| 数据列数不匹配 | `IndexError` | `"Edge data {d} and data_keys {data} are not the same length"` |
| 字段类型转换失败 | `TypeError` | `"Failed to convert {edge_key} data {edge_value} to type {edge_type}."` |

### 4.6 AdjacencyList 异常处理

| 错误场景 | 异常类型 | 错误消息 |
|---------|---------|---------|
| 源节点类型转换失败 | `TypeError` | `"Failed to convert node ({u}) to type {nodetype}"` |
| 邻居节点类型转换失败 | `TypeError` | `"Failed to convert nodes ({','.join(vlist)}) to type {nodetype}"` |

---

## 五、关键边界情况汇总

### 5.1 孤立节点处理

| 格式 | 能否表示孤立节点 | 方式 |
|------|:--------------:|------|
| **GraphML** | ✅ | `<node>` 元素 |
| **GML** | ✅ | `node` 列表 |
| **GEXF** | ✅ | `<node>` 元素 |
| **JSON** | ✅ | `nodes` 列表 |
| **EdgeList** | ⚠️ 有限 | 只有自环的边 |
| **AdjacencyList** | ✅ | 只有源节点的行 |

### 5.2 默认值处理策略

| 格式 | 默认值存储位置 | 是否自动应用 |
|------|--------------|:------------:|
| **GraphML** | `G.graph["node_default"]` / `G.graph["edge_default"]` | ❌ 需手动应用 |
| **GEXF** | `G.graph["node_default"]` / `G.graph["edge_default"]` | ❌ 需手动应用 |
| **GML** | 不支持默认值 | - |
| **JSON** | 不支持默认值 | - |

**GraphML 默认值手动应用示例**（来自文档注释）：
```python
# 读取后手动应用默认值
default_color = G.graph["node_default"]["color"]
for node, data in G.nodes(data=True):
    if "color" not in data:
        data["color"] = default_color
```

### 5.3 类型转换失败处理

| 格式 | 转换失败策略 | 示例场景 |
|------|-------------|---------|
| **GraphML** | `node_type`/`edge_key_type` 参数控制 | edge_id 转 int 失败则保持字符串 |
| **GML** | `destringizer` 异常静默忽略 | 字符串解析失败则保持字符串 |
| **GEXF** | `node_type` 参数控制 | 类似 GraphML |
| **EdgeList** | 抛出 `TypeError` | 严格模式 |
| **AdjacencyList** | 抛出 `TypeError` | 严格模式 |

### 5.4 重复边（MultiGraph 检测）

| 格式 | 检测方式 | 处理策略 |
|------|---------|---------|
| **GraphML** | `G.has_edge(source, target)` | 设置 `self.multigraph = True` |
| **GEXF** | `G.has_edge(source, target)` | 设置 `self.simple_graph = False` |
| **GML** | 依赖 `multigraph` 字段 | 声明为普通图时重复边抛异常 |
| **JSON** | 依赖 `multigraph` 字段 | 不检测，由数据决定 |
| **EdgeList** | 不检测 | 由 `create_using` 决定 |

---

## 六、代码位置速查

| 功能 | 文件 | 关键函数/类 |
|------|------|------------|
| GraphML 反序列化 | `graphml.py:839-1053` | `GraphMLReader` |
| GML 反序列化 | `gml.py:298-525` | `parse_gml_lines()` |
| GML 词法分析 | `gml.py:301-363` | `tokenize()` |
| GML 语法解析 | `gml.py:377-442` | `parse_kv()` |
| GEXF 反序列化 | `gexf.py:749-1087` | `GEXFReader` |
| JSON Node-Link | `node_link.py:142-260` | `node_link_graph()` |
| JSON Adjacency | `adjacency.py:85-156` | `adjacency_graph()` |
| JSON Tree | `tree.py:84-135` | `tree_graph()` |
| EdgeList 解析 | `edgelist.py:177-297` | `parse_edgelist()` |
| AdjacencyList 解析 | `adjlist.py:174-243` | `parse_adjlist()` |

---

## 七、最佳实践建议

### 7.1 格式选择建议

| 需求场景 | 推荐格式 | 理由 |
|---------|---------|------|
| 完整属性保真 | **GraphML** 或 **GEXF** | 强类型，完整属性支持 |
| Python 复杂类型 | **GML** + `literal_stringizer` | 支持 `list`, `dict`, `tuple` 等 |
| 动态图数据 | **GEXF** | 原生支持动态属性 |
| Web 前端交互 | **JSON Node-Link** | d3.js 原生支持 |
| 简单边数据 | **EdgeList** | 简单高效 |
| 仅拓扑结构 | **AdjacencyList** 或 **Graph6** | 最紧凑 |

### 7.2 错误处理建议

1. **使用 try-except 包装反序列化**：
```python
import networkx as nx

try:
    G = nx.read_graphml("data.graphml")
except nx.NetworkXError as e:
    print(f"GraphML 解析错误: {e}")
except KeyError as e:
    print(f"JSON 格式错误，缺少键: {e}")
except TypeError as e:
    print(f"类型转换错误: {e}")
```

2. **验证反序列化结果**：
```python
# 检查节点数
assert len(G.nodes) == expected_node_count

# 检查关键属性
assert "name" in G.graph
assert all("weight" in G.edges[u, v] for u, v in G.edges)
```

3. **GML destringizer 使用**：
```python
from networkx.readwrite.gml import literal_destringizer, literal_stringizer

# 写入时序列化复杂类型
nx.write_gml(G, "data.gml", stringizer=literal_stringizer)

# 读取时恢复复杂类型
G = nx.read_gml("data.gml", destringizer=literal_destringizer)
```

### 7.3 性能优化建议

1. **大文件选择**：
   - 超大图（100万+节点）：避免 XML 格式（GraphML, GEXF）
   - 考虑 EdgeList 或 AdjacencyList

2. **GraphML 性能**：
   - `write_graphml_lxml()` 比 `write_graphml_xml()` 快
   - 需要安装 `lxml` 库

3. **增量解析**：
   - 某些格式支持生成器模式，但反序列化通常需要完整读取

---

## 八、总结

NetworkX 的反序列化机制展现了**高度的设计一致性**和**格式特定的优化**：

### 核心模式

1. **两阶段解析**：语法解析 → 属性恢复
2. **图类型决策**：初始类型 + MultiGraph 检测
3. **属性分层恢复**：图级 → 节点 → 边
4. **异常处理**：格式越复杂，错误检查越严格

### 格式特性对比

| 维度 | 强类型格式（GraphML, GEXF） | 弱类型格式（GML, JSON） | 文本格式（EdgeList, AdjacencyList） |
|------|:--------------------------:|:----------------------:|:----------------------------------:|
| 类型安全 | ✅ 高 | ⚠️ 中 | ❌ 低 |
| 属性完整性 | ✅ 完整 | ✅ 完整 | ⚠️ 部分或无 |
| 错误检测 | ✅ 严格 | ⚠️ 中等 | ⚠️ 宽松 |
| 性能 | 🐢 较慢 | ⚡ 中等 | ⚡ 最快 |
| 复杂度 | 📚 高 | 📗 中 | 📕 低 |

### 关键设计权衡

1. **GraphML/GEXF 的设计哲学**：
   - 先声明类型，后使用数据
   - 默认值存储但不自动应用（给用户控制权）
   - 严格验证，早报错

2. **GML 的设计哲学**：
   - 词法阶段推断类型
   - `destringizer` 扩展机制（灵活性）
   - 详细的错误位置信息（行号、列号）

3. **JSON 的设计哲学**：
   - 直接字典映射（简单）
   - 依赖 JSON 原生类型（兼容性）
   - 最少的验证（性能）

4. **文本格式的设计哲学**：
   - 仅必要信息（紧凑）
   - 灵活的解析选项（`data` 参数）
   - 宽松的错误处理（跳过无效行）

理解这些设计权衡，可以帮助开发者在不同场景下选择最合适的序列化格式，并正确处理反序列化过程中的各种边界情况。
