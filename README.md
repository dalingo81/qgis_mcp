# QGISMCP —— 让大模型直接操作 QGIS

通过 [Model Context Protocol (MCP)](https://modelcontextprotocol.io/docs/getting-started/intro) 把 [QGIS](https://qgis.org/) 接到大模型上，让 AI 能直接读写工程、加载图层、跑 Processing 算法、执行 PyQGIS 代码并出图。

原项目由 [Juan Santos Ochoa](https://github.com/jjsantos01) 开发，设计上参考了 [BlenderMCP](https://github.com/ahujasid/blender-mcp/tree/main)。

> **关于本仓库**
> 这是 [`jjsantos01/qgis_mcp`](https://github.com/jjsantos01/qgis_mcp) 的 fork。上游有 1084 star，但**自 2025-10-01 起已 11 个月未更新，18 个 issue 未处理**。
> 本 fork 在 **QGIS 3.44.12 "Solothurn"** 上把 15 个工具全部实测了一遍，**修复了 5 个 bug 并做了 4 项改进**，详见 [本 fork 的修改](#本-fork-的修改)。
> 英文原版保留在 [README_EN.md](./README_EN.md)。

---

## 目录

- [它能做什么](#它能做什么)
- [架构](#架构)
- [本 fork 的修改](#本-fork-的修改)
- [安装](#安装)
- [WorkBuddy 一键安装](#workbuddy-一键安装)
- [15 个工具](#15-个工具)
- [使用示例](#使用示例)
- [已知限制与坑](#已知限制与坑)
- [许可证](#许可证)

---

## 它能做什么

- **双向通信**：大模型通过 socket 与 QGIS 建立连接，可读取状态、下发指令
- **工程管理**：新建、打开、保存 `.qgz` / `.qgs` 工程
- **图层操作**：加载矢量（shp / gpkg / gdb / geojson…）与栅格（tif / img…）图层，列表、移除、缩放定位
- **执行 Processing 算法**：调用 QGIS Processing 工具箱里的任意算法（缓冲区、裁剪、重投影、融合…）
- **执行任意 PyQGIS 代码**：`execute_code` 直接 `exec()` 代码，并把 `stdout` 和 `traceback` 原样回传给模型 —— **整套设计里最强大的一环**，模型能看着报错自我纠错
- **渲染出图**：把当前地图视图导出成 PNG

## 架构

QGIS 是 GUI 程序，没法从外部往里塞一个 Python 解释器。所以这套方案分两层：让插件在 QGIS 内部监听端口，由独立的 MCP Server 进程做协议转换。

```mermaid
flowchart LR
    A["大模型<br/>(Claude / WorkBuddy 等 MCP 客户端)"]
    B["qgis_mcp_server.py<br/>独立进程，MCP 协议"]
    C["QGIS MCP 插件<br/>运行在 QGIS 内部"]
    D["PyQGIS / QGIS 内核"]

    A -- "stdio (JSON-RPC)" --> B
    B -- "TCP 127.0.0.1:9876" --> C
    C --> D
```

| 组件 | 位置 | 作用 |
|---|---|---|
| [QGIS 插件](./qgis_mcp_plugin/) | QGIS 进程内 | 起 socket server，接收并执行命令 |
| [MCP Server](./src/qgis_mcp/qgis_mcp_server.py) | 独立 Python 进程 | 实现 MCP 协议（stdio），转发到插件 |

---

## 本 fork 的修改

15 个工具逐一实测，下面 5 个 bug 均已**复现 → 修复 → 通过 MCP 通道复验**。

### 修复的 bug

| # | 问题 | 后果 | 修复 |
|---|---|---|---|
| 1 | `get_project_info` 写死只返回 10 个图层 | 23 个图层的工程只报 10 个，模型会误判工程内容 | 返回全部图层 |
| 2 | `get_layers` / `get_project_info` 调用 `findLayer(...).isVisible()` 未判空 | 图层在工程里但不在图层树时**直接崩溃** | 判空处理，这类图层返回 `visible: false` |
| 3 | `create_new_project` 仅在 `fileName()` 非空时才 `clear()` | 工程从未保存过时，旧图层会赖在"新工程"里 | 无条件先 `clear()` |
| 4 | `execute_processing` 把输出图层 `str()` 成 `"<QgsVectorLayer: 'output' (memory)>"` | 结果不可用，图层也没进工程 | 返回结构化 `{id, name, type, feature_count}`，并自动加入工程 |
| 5 | `render_map` 直接用 `mapLayers().values()` | **完全无视图层树可见性和绘制顺序**，隐藏的图层照样画 | 遍历图层树，只渲染可见图层且按树序绘制 |

顺带把图层类型从不可读的枚举值（`vector_0` / `vector_1`）改成 `vector (Point)` / `vector (Line)` / `vector (Polygon)` / `raster`。

### 其他改进

- **插件自动启动服务**：`initGui()` 里直接拉起 socket server，QGIS 一开就在监听 9876，不用每次手动点 "Start Server"
- **关闭面板不再停服务**：面板关闭只是隐藏，只有 "Stop Server" 才真停
- **显式用 IPv4 连接**：server 端连 `127.0.0.1` 而非 `localhost`。部分机器上 `localhost` 优先解析成 IPv6 `::1`，而插件只监听 IPv4，会报一句很不明确的 *"Could not connect to Qgis"*
- **修复心跳探活**：原代码引用了不存在的属性，且 `sendall(b'')` 在 TCP 上永远成功，根本检测不到对端已死

---

## 安装

### 环境要求

| 项目 | 说明 |
|---|---|
| QGIS | 3.x（本 fork 实测 **3.44.12 LTR**，Qt5 构建） |
| Python | 3.10 或更高 |
| 依赖管理 | [uv](https://docs.astral.sh/uv/getting-started/installation/)（推荐）或普通 venv |

> 上游只测过 QGIS 3.22。本 fork 在 3.44.12 上全量验证通过。

### 1. 获取代码

```bash
git clone https://github.com/dalingo81/qgis_mcp.git
cd qgis_mcp
```

### 2. 安装 QGIS 插件

把 [qgis_mcp_plugin](./qgis_mcp_plugin/) 整个文件夹复制到 QGIS 当前 profile 的插件目录。

profile 目录可在 QGIS 里通过 `设置 → 用户配置 → 打开当前配置文件夹` 定位，然后进入 `python/plugins`：

- **Windows**：`C:\Users\用户名\AppData\Roaming\QGIS\QGIS3\profiles\default\python\plugins`
- **macOS**：`~/Library/Application Support/QGIS/QGIS3/profiles/default/python/plugins`

复制后**重启 QGIS**，到 `插件 → 管理和安装插件`，在 `全部` 里搜索 "QGIS MCP" 并勾选启用。

### 3. 配置 MCP 客户端

#### 方案 A：uv（上游原版写法）

```json
{
  "mcpServers": {
    "qgis": {
      "command": "uv",
      "args": [
        "--directory",
        "/绝对路径/qgis_mcp/src/qgis_mcp",
        "run",
        "qgis_mcp_server.py"
      ]
    }
  }
}
```

#### 方案 B：venv 直跑（推荐，本 fork 实测）

先建环境并装依赖：

```bash
cd qgis_mcp
uv venv
uv pip install -e .
```

然后配置里直接指向 venv 的解释器：

```json
{
  "mcpServers": {
    "qgis": {
      "command": "C:\\绝对路径\\qgis_mcp\\.venv\\Scripts\\python.exe",
      "args": [
        "C:\\绝对路径\\qgis_mcp\\src\\qgis_mcp\\qgis_mcp_server.py"
      ]
    }
  }
}
```

比 `uv run` 启动更快，也不受 `uv` 是否在 PATH 里的影响（Windows 上 uv 常装在 `~/.local/bin`，MCP 客户端未必找得到）。

#### WorkBuddy

编辑 `~/.workbuddy/mcp.json`，加入上面的 `qgis` 条目。嫌手动配麻烦，直接看下一章的 [一键安装提示词](#workbuddy-一键安装)。

### 4. 启动并验证

1. 打开 QGIS —— 插件会自动拉起服务（右下角无报错即成功）
2. 在客户端里调用 `ping`，返回 `{"pong": true}` 即链路通

---

## WorkBuddy 一键安装

如果你是 [WorkBuddy](https://www.workbuddy.cn/) 用户，上面 4 步可以全部交给 AI 做：把下面这段提示词整段复制进对话框即可（`<安装目录>` 留着也行，WorkBuddy 会自己挑位置并告诉你；想指定就提前替换成绝对路径）。

```plain
帮我在这台机器上安装并跑通 qgis-mcp（QGIS 的 MCP 服务器），装完我要能在对话里直接操作 QGIS。

仓库用这个 fork：https://github.com/dalingo81/qgis_mcp
（上游 jjsantos01/qgis_mcp 已停更 11 个月，这个 fork 修了 5 个 bug，并在 QGIS 3.44.12 LTR 上把 15 个工具全量实测过。）

按顺序执行，每步做完用一行汇报结果；失败了说清原因再问我的意见，不要静默跳过，也不要自作主张换方案：

0. 先确认本机已安装 QGIS（Windows 通常在 C:\Program Files\QGIS 3.44.12，其他系统用 which/qgis 找）。
   没装就停下来告诉我，等我装好再继续。

1. 克隆仓库到 <安装目录>/qgis_mcp，克隆前先告诉我你打算放在哪。

2. 建虚拟环境并装依赖：
   cd <安装目录>/qgis_mcp
   uv venv
   uv pip install -e .
   uv 不在 PATH 的话，Windows 一般在 %USERPROFILE%\.local\bin\uv.exe，用绝对路径调用。

3. 把仓库里的 qgis_mcp_plugin 整个文件夹，复制到 QGIS 当前 profile 的插件目录：
   - Windows: %APPDATA%\QGIS\QGIS3\profiles\default\python\plugins
   - macOS: ~/Library/Application Support/QGIS/QGIS3/profiles/default/python/plugins
   复制完提醒我重启 QGIS，并在「插件 → 管理和安装插件」里启用 "QGIS MCP"。

4. 在 ~/.workbuddy/mcp.json 的 mcpServers 里加一条 qgis，不要动其他已有条目：
   {
     "qgis": {
       "command": "<安装目录>/qgis_mcp/.venv/Scripts/python.exe",
       "args": ["<安装目录>/qgis_mcp/src/qgis_mcp/qgis_mcp_server.py"]
     }
   }
   注意：必须是绝对路径；用 venv 里的解释器直跑，别用 "uv run" —— Windows 上 uv 常常不在 MCP 客户端的 PATH 里。
   macOS/Linux 把 .venv/Scripts/python.exe 换成 .venv/bin/python。

5. 做完后告诉我两件事：
   - 去「连接器管理」页面给新出现的 qgis 服务器点「信任」
   - 重启 WorkBuddy，不重启工具列表不会加载新服务器

6. 我重启 WorkBuddy 并打开 QGIS 后，调用 qgis 的 ping 工具验证，返回 {"pong": true} 就算装好了。
   如果报 "Could not connect to Qgis"，按这个顺序排查：QGIS 是否真的开着 → 插件是否已启用
   → 9876 端口是否被占用 → server 端是不是连的 127.0.0.1（localhost 在部分机器上会先解析到
   IPv6 ::1，而插件只监听 IPv4，会报这句含义模糊的错误）。
```

> **两个容易踩的点**
>
> 1. **必须重启 WorkBuddy**：工具列表在会话启动时固化，信任了不重启照样看不到 `mcp__qgis__*` 工具。
> 2. **别拿连接器状态页判断成败**：那里只列内置/市场连接器，自定义 MCP 不在其中。可靠判据是两条 —— 进程里有没有 `qgis_mcp_server.py` 的命令行、模型侧能否搜到 `mcp__qgis__*` 工具。

---

## 15 个工具

| 工具 | 说明 |
|---|---|
| `ping` | 测试与 QGIS 插件的连接 |
| `get_qgis_info` | 获取 QGIS 版本、profile 路径、已装插件数 |
| `load_project` | 打开指定路径的工程文件 |
| `create_new_project` | 新建工程并保存 |
| `get_project_info` | 获取当前工程信息（CRS、图层清单） |
| `add_vector_layer` | 加载矢量图层（shp / gpkg / gdb / geojson…） |
| `add_raster_layer` | 加载栅格图层（tif / img…） |
| `get_layers` | 列出工程中所有图层 |
| `remove_layer` | 按图层 ID 移除 |
| `zoom_to_layer` | 缩放到指定图层范围 |
| `get_layer_features` | 读取图层要素，可限制条数 |
| `execute_processing` | 执行 Processing 算法 |
| `save_project` | 保存工程到指定路径 |
| `render_map` | 把当前地图视图渲染成图片 |
| `execute_code` | 在 QGIS 内执行任意 PyQGIS 代码 |

---

## 使用示例

实测跑通的几个中文场景（QGIS 3.44.12 + WorkBuddy）：

```plain
1. 加载 D:\Downloads\shp\中国地图.gdb 里的全部数据
   → 18 个图层、308 万条要素，OpenFileGDB 驱动直读，约 0.2 秒加载完成

2. 找出 name 为「测绘大厦」的要素
   → 命中 buildings_a 图层，返回要素 ID、坐标 113.368223°E, 23.12603°N

3. 画出从翠湖山庄到测绘大厦的开车路径
   → 截取局部路网 → 建图 → Dijkstra 最短路 → 生成红色路线图层，2.60 公里

4. 把当前视图渲染成 2000×1600 的 PNG
   → render_map 输出，图层可见性正确
```

> **注意**：`Shape_Area` / `Shape_Length` 这类字段在 EPSG:4326 下是**平方度和度**，不是平方米。要算真实面积/长度，先重投影到 CGCS2000 高斯投影带（如 EPSG:4547），或用椭球面积计算。

---

## 已知限制与坑

这些问题上游尚未解决，用之前心里有数：

1. **命令跑在 Qt 主线程**：执行耗时的 `processing.run()` 或渲染大图时，QGIS 界面会卡住，且**没有超时也没有取消机制**。处理百万级要素前建议先关掉画布渲染（`iface.mapCanvas().setRenderFlag(False)`）。
2. **消息分帧没有长度前缀**：靠"能否 `json.loads` 成功"来判断一条消息是否收完。一旦两条命令粘包，缓冲区会永久累积再也解不出来，**只能重启插件**。
3. **QGIS 必须开着**：没启动 QGIS 时调用工具只会得到 *"Could not connect to Qgis"*，MCP server 不会帮你拉起 QGIS。
4. **端口固定 9876**：插件界面能改端口，但 server 端是常量，改了就连不上。
5. **`execute_code` 是 `exec()` 任意代码**：功能强，但务必清楚你在执行什么。
6. **上游无 LICENSE**：见下方。

**数据合规提醒**：如果拿 OSM 等公开派生数据做底图，注意它**不含国界、行政界线、九段线**，海岸线与岛屿画法也不是国家标准画法。内部分析可以，**对外发布或出正式图件必须走自然资源部标准地图服务（带审图号）**。

---

## 许可证

上游仓库 **没有 LICENSE 文件**（GitHub 返回 license 为 null），意味着默认保留所有权利。本 fork 继承了这一状态。

- 自己研究、内部使用：没问题
- 二次开发后对外分发、或集成进商业产品：**建议先联系上游作者确认授权**

---

## 相关链接

- 上游仓库：<https://github.com/jjsantos01/qgis_mcp>
- MCP 协议：<https://modelcontextprotocol.io>
- QGIS 官方文档：<https://docs.qgis.org>
- 自然资源部标准地图服务：<http://bzdt.ch.mnr.gov.cn>
