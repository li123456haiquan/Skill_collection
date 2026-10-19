# Virtuoso-Bridge-Lite 完整架构图

## 整体架构

```mermaid
graph TB
    subgraph "用户层"
        CLI["virtuoso-bridge CLI<br/>(start/status/eval/load)"]
        PYAPI["Python API<br/>(VirtuosoClient, SpectreSimulator)"]
        AGENT["AI Agent<br/>(Claude Code, Cursor)"]
    end

    subgraph "核心客户端层"
        VC["VirtuosoClient<br/>- TCP SKILL 客户端<br/>- execute_skill()<br/>- 组合 Ops 类"]
        SS["SpectreSimulator<br/>- Spectre 仿真运行器<br/>- PSF 结果解析"]
        SC["SanitizingClient<br/>- 清理敏感信息"]
    end

    subgraph "操作层 (领域特定 API)"
        LO["LayoutOps<br/>- 版图编辑<br/>- GDS 导出"]
        SO["SchematicOps<br/>- 原理图编辑<br/>- 网表导入导出"]
        SYMO["SymbolOps<br/>- 符号编辑<br/>- 从原理图生成"]
        LIBO["LibraryOps<br/>- 库管理<br/>- 分类管理"]
        MO["MaestroOps<br/>- ADE/Maestro<br/>- 仿真设置"]
    end

    subgraph "传输层"
        SSHC["SSHClient<br/>- SSH 隧道管理<br/>- 远程命令执行<br/>- 文件传输"]
        LOCAL["本地模式<br/>- 无 SSH<br/>- 直接 TCP"]
    end

    subgraph "远程/本地服务"
        VIRTUOSO["Cadence Virtuoso<br/>- SKILL 进程<br/>- RAMIC Bridge Daemon"]
        SPECTRE["Spectre 仿真器<br/>- 独立运行<br/>- PSF 输出"]
    end

    subgraph "底层通信"
        TCP["TCP 连接<br/>host:port"]
        SSH["SSH 隧道<br/>-L local:remote:port"]
        IPC["IPC<br/>ipcBeginProcess<br/>evalstring"]
    end

    CLI --> VC
    CLI --> SS
    PYAPI --> VC
    PYAPI --> SS
    AGENT --> CLI
    AGENT --> PYAPI

    VC --> LO
    VC --> SO
    VC --> SYMO
    VC --> LIBO
    VC --> MO

    VC --> SSHC
    VC --> LOCAL
    SS --> SSHC
    SS --> LOCAL

    SSHC --> SSH
    LOCAL --> TCP

    SSH --> VIRTUOSO
    TCP --> VIRTUOSO

    VIRTUOSO --> IPC
    SSHC --> SPECTRE

    style CLI fill:#e1f5ff
    style PYAPI fill:#e1f5ff
    style AGENT fill:#fff4e1
    style VC fill:#ffe1e1
    style SS fill:#ffe1e1
    style SC fill:#ffe1e1
    style LO fill:#e1ffe1
    style SO fill:#e1ffe1
    style SYMO fill:#e1ffe1
    style LIBO fill:#e1ffe1
    style MO fill:#e1ffe1
    style SSHC fill:#f5e1ff
    style LOCAL fill:#f5e1ff
    style VIRTUOSO fill:#ffffe1
    style SPECTRE fill:#ffffe1
```

## 数据流详解

### SKILL 执行流程

```mermaid
sequenceDiagram
    participant User as 用户/Agent
    participant VC as VirtuosoClient
    participant Ops as Layout/Schematic/...Ops
    participant SSH as SSHClient (可选)
    participant TCP as TCP Socket
    participant Daemon as RAMIC Bridge Daemon
    participant Virtuoso as Virtuoso SKILL

    User->>VC: client.layout.create(lib, cell)
    VC->>Ops: LayoutEditor(owner, lib, cell)
    User->>Ops: editor.draw_rect(layer, coords)
    Ops->>VC: execute_skill(skill_code)
    VC->>SSH: warm() (if remote)
    SSH->>TCP: 建立 TCP 连接
    VC->>TCP: 发送 SKILL JSON
    TCP->>Daemon: TCP 数据
    Daemon->>Virtuoso: evalstring(skill_code)
    Virtuoso->>Virtuoso: 执行 SKILL
    Virtuoso->>Daemon: 返回结果
    Daemon->>TCP: JSON 响应
    TCP->>VC: 接收结果
    VC->>Ops: VirtuosoResult
    Ops->>User: 返回编辑器状态
```

### Spectre 仿真流程

```mermaid
sequenceDiagram
    participant User as 用户/Agent
    participant SS as SpectreSimulator
    participant SSH as SSHClient
    participant Remote as 远程 Shell
    participant Spectre as Spectre 仿真器
    participant PSF as PSF 解析器

    User->>SS: SpectreSimulator.from_env()
    SS->>SSH: 检查 spectre 可用性
    SSH->>Remote: which spectre
    Remote->>SSH: spectre 路径
    SSH->>SS: 返回路径

    User->>SS: run(netlist, corner)
    SS->>SSH: 上传 netlist 到远程
    SSH->>Remote: 文件传输

    SS->>SSH: 执行 spectre 命令
    SSH->>Remote: spectre -E ...
    Remote->>Spectre: 启动仿真
    Spectre->>Spectre: 运行仿真
    Spectre->>Remote: 生成 PSF 文件
    Remote->>SSH: 完成

    SS->>SSH: 下载 PSF 文件
    SSH->>SS: 文件传输

    SS->>PSF: 解析 PSF
    PSF->>SS: SimulationResult
    SS->>User: 返回结果
```

## 模块依赖关系

```mermaid
graph LR
    subgraph "核心模块"
        INIT["virtuoso_bridge/__init__.py<br/>包入口"]
        MODELS["models.py<br/>数据模型"]
        ENV["env.py<br/>环境变量"]
        PROFILE["profile.py<br/>配置管理"]
    end

    subgraph "Virtuoso 客户端"
        BRIDGE["virtuoso/basic/bridge.py<br/>VirtuosoClient"]
        OPS["virtuoso/ops.py<br/>通用操作"]
        COMPOSITION["virtuoso/basic/composition.py<br/>SKILL 组合"]
    end

    subgraph "领域操作"
        LAYOUT["virtuoso/layout/__init__.py<br/>LayoutOps"]
        SCHEMATIC["virtuoso/schematic/__init__.py<br/>SchematicOps"]
        SYMBOL["virtuoso/symbol/__init__.py<br/>SymbolOps"]
        LIBRARY["virtuoso/library/__init__.py<br/>LibraryOps"]
        MAESTRO["virtuoso/maestro/__init__.py<br/>MaestroOps"]
    end

    subgraph "传输层"
        TUNNEL["transport/tunnel.py<br/>SSHClient"]
        REMOTE["transport/remote.py<br/>远程运行器"]
        PARAMIKO["transport/paramiko.py<br/>Paramiko 后端"]
    end

    subgraph "Spectre"
        RUNNER["spectre/runner.py<br/>SpectreSimulator"]
        PSF["spectre/psf.py<br/>PSF 解析"]
        PARSER["spectre/parser.py<br/>解析器"]
    end

    subgraph "CLI"
        CLI_MAIN["cli.py<br/>命令行接口"]
    end

    INIT --> BRIDGE
    INIT --> TUNNEL
    INIT --> RUNNER
    INIT --> MODELS

    MODELS --> BRIDGE
    MODELS --> RUNNER

    ENV --> BRIDGE
    ENV --> TUNNEL
    ENV --> RUNNER

    PROFILE --> BRIDGE
    PROFILE --> TUNNEL
    PROFILE --> RUNNER

    BRIDGE --> OPS
    BRIDGE --> COMPOSITION
    BRIDGE --> LAYOUT
    BRIDGE --> SCHEMATIC
    BRIDGE --> SYMBOL
    BRIDGE --> LIBRARY
    BRIDGE --> MAESTRO

    LAYOUT --> OPS
    SCHEMATIC --> OPS
    SYMBOL --> OPS
    LIBRARY --> OPS
    MAESTRO --> OPS

    BRIDGE --> TUNNEL
    RUNNER --> REMOTE

    TUNNEL --> REMOTE
    TUNNEL --> PARAMIKO

    RUNNER --> PSF
    RUNNER --> PARSER

    CLI_MAIN --> BRIDGE
    CLI_MAIN --> TUNNEL
    CLI_MAIN --> RUNNER

    style INIT fill:#e1f5ff
    style MODELS fill:#e1f5ff
    style ENV fill:#e1f5ff
    style PROFILE fill:#e1f5ff
    style BRIDGE fill:#ffe1e1
    style OPS fill:#ffe1e1
    style COMPOSITION fill:#ffe1e1
    style LAYOUT fill:#e1ffe1
    style SCHEMATIC fill:#e1ffe1
    style SYMBOL fill:#e1ffe1
    style LIBRARY fill:#e1ffe1
    style MAESTRO fill:#e1ffe1
    style TUNNEL fill:#f5e1ff
    style REMOTE fill:#f5e1ff
    style PARAMIKO fill:#f5e1ff
    style RUNNER fill:#ffffe1
    style PSF fill:#ffffe1
    style PARSER fill:#ffffe1
    style CLI_MAIN fill:#ffe1e1
```

## 核心类关系图

```mermaid
classDiagram
    class VirtuosoInterface {
        <<interface>>
        +execute_skill(code) VirtuosoResult
        +ensure_ready() VirtuosoResult
        +warm_remote_session() VirtuosoResult
    }

    class VirtuosoClient {
        -_host: str
        -_port: int
        -_timeout: int
        -_tunnel: SSHClient
        +layout: LayoutOps
        +schematic: SchematicOps
        +symbol: SymbolOps
        +library: LibraryOps
        +maestro: MaestroOps
        +execute_skill(code) VirtuosoResult
        +from_env() VirtuosoClient
        +local() VirtuosoClient
        +from_tunnel() VirtuosoClient
    }

    class LayoutOps {
        -_owner: VirtuosoClient
        +create(lib, cell) LayoutEditor
        +modify(lib, cell) LayoutEditor
        +export_gds(...) GdsExportResult
    }

    class SchematicOps {
        -_owner: VirtuosoClient
        +create(lib, cell) SchematicEditor
        +modify(lib, cell) SchematicEditor
        +read(...) dict
        +export_netlist(...) Result
        +import_netlist(...) Result
    }

    class SymbolOps {
        -_owner: VirtuosoClient
        +create(lib, cell) SymbolEditor
        +modify(lib, cell) SymbolEditor
        +generate_from_schematic(...) Result
        +read_ports(...) dict
    }

    class LibraryOps {
        -_owner: VirtuosoClient
        +list() list~str~
        +get(name) LibraryInfo
        +create(name, path) LibraryInfo
        +delete(name) void
        +rename(name, new_name) LibraryInfo
    }

    class MaestroOps {
        -_owner: VirtuosoClient
        +read() dict
        +create_testbench(...) Result
        +run(...) Result
        +read_results(...) dict
    }

    class SSHClient {
        -_remote_host: str
        -_remote_user: str
        -_remote_port: int
        -_local_port: int
        -_jump_host: str
        +is_running() bool
        +warm() void
        +run_command(cmd) str
        +upload_file(src, dst) void
        +download_file(src, dst) str
    }

    class SpectreSimulator {
        -_ssh_runner: SSHRunner
        +from_env() SpectreSimulator
        +run(netlist, corner) SimulationResult
        +check_license() bool
    }

    class VirtuosoResult {
        +status: ExecutionStatus
        +output: str
        +errors: list~str~
        +metadata: dict
        +execution_time: float
    }

    class ExecutionStatus {
        <<enumeration>>
        SUCCESS
        ERROR
        TIMEOUT
    }

    VirtuosoInterface <|.. VirtuosoClient
    VirtuosoClient *-- LayoutOps
    VirtuosoClient *-- SchematicOps
    VirtuosoClient *-- SymbolOps
    VirtuosoClient *-- LibraryOps
    VirtuosoClient *-- MaestroOps
    VirtuosoClient o-- SSHClient
    VirtuosoClient --> VirtuosoResult
    SpectreSimulator o-- SSHClient
    VirtuosoResult --> ExecutionStatus
```

## 远程 vs 本地模式对比

```mermaid
graph TB
    subgraph "远程模式"
        R1["本地电脑"]
        R2["SSH 隧道<br/>-L local:remote:port"]
        R3["跳板主机 (可选)"]
        R4["远程服务器<br/>Virtuoso + Spectre"]
        R5["RAMIC Daemon<br/>在 Virtuoso CIW 中"]

        R1 -->|"SSH 连接"| R2
        R2 -->|"端口转发"| R3
        R3 -->|"SSH 跳转"| R4
        R4 --> R5
    end

    subgraph "本地模式"
        L1["同一台机器"]
        L2["Virtuoso + Bridge"]
        L3["RAMIC Daemon<br/>在 Virtuoso CIW 中"]

        L1 --> L2
        L2 --> L3
    end

    style R1 fill:#e1f5ff
    style R4 fill:#ffe1e1
    style L1 fill:#e1ffe1
    style L2 fill:#e1ffe1
```

## 编辑器上下文管理器流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant Ops as Layout/SchematicOps
    participant Editor as Layout/SchematicEditor
    participant VC as VirtuosoClient
    participant Virtuoso as Virtuoso

    User->>Ops: client.layout.create(lib, cell)
    Ops->>Editor: LayoutEditor(owner, lib, cell, mode="w")
    Editor->>VC: dbOpenCellViewByType(..., mode="w")
    VC->>Virtuoso: execute_skill()
    Virtuoso->>Virtuoso: 打开/创建 cellview
    Virtuoso->>VC: 返回 cellview ID
    VC->>Editor: 返回编辑器实例

    User->>Editor: with editor as ctx:
    User->>Editor: ctx.draw_rect(...)
    Editor->>VC: leCreateRect(...)
    VC->>Virtuoso: execute_skill()
    Virtuoso->>Virtuoso: 绘制矩形
    Virtuoso->>VC: 返回结果
    VC->>Editor: 返回结果

    User->>Editor: exit context
    Editor->>VC: geSaveCellView()
    VC->>Virtuoso: execute_skill()
    Virtuoso->>Virtuoso: 保存 cellview
    Virtuoso->>VC: 返回结果
    Editor->>VC: geCloseCellView()
    VC->>Virtuoso: execute_skill()
    Virtuoso->>Virtuoso: 关闭 cellview
    VC->>Editor: 完成
```

## 文件系统映射

```mermaid
graph LR
    subgraph "本地文件系统"
        LOCAL_SCRIPTS["Python 脚本<br/>examples/"]
        LOCAL_NETLIST["网表文件<br/>*.scs"]
        LOCAL_ENV[".env 配置"]
    end

    subgraph "远程文件系统 (远程模式)"
        REMOTE_DAEMON["RAMIC Daemon<br/>/tmp/virtuoso_bridge_*/"]
        REMOTE_NETLIST["上传的网表<br/>运行目录"]
        REMOTE_PSF["PSF 输出<br/>psf/"]
    end

    subgraph "Virtuoso 工作区"
        VIRTUOSO_LIB["设计库<br/>lib/"]
        VIRTUOSO_CELL["单元<br/>lib/cell/view"]
    end

    LOCAL_SCRIPTS -->|"upload_file()"| REMOTE_DAEMON
    LOCAL_NETLIST -->|"upload_file()"| REMOTE_NETLIST
    LOCAL_ENV -->|"环境变量读取"| SSH_CONFIG

    REMOTE_DAEMON -->|"Virtuoso load()"| VIRTUOSO_CELL
    REMOTE_NETLIST -->|"Spectre 读取"| VIRTUOSO_CELL
    VIRTUOSO_CELL -->|"生成"| REMOTE_PSF
    REMOTE_PSF -->|"download_file()"| LOCAL_NETLIST

    style LOCAL_SCRIPTS fill:#e1f5ff
    style REMOTE_DAEMON fill:#ffe1e1
    style VIRTUOSO_CELL fill:#e1ffe1
```

## 关键设计模式

### 1. 组合模式 (Composition Pattern)
- `VirtuosoClient` 组合了多个 Ops 类
- 每个 Ops 类持有 `VirtuosoClient` 的引用
- 实现了职责分离和模块化

### 2. 工厂模式 (Factory Pattern)
- `VirtuosoClient.from_env()` - 从环境变量创建
- `VirtuosoClient.local()` - 本地模式创建
- `VirtuosoClient.from_tunnel()` - 从隧道创建

### 3. 上下文管理器 (Context Manager)
- `LayoutEditor` / `SchematicEditor` / `SymbolEditor`
- 自动管理 cellview 的打开、保存、关闭

### 4. 策略模式 (Strategy Pattern)
- SSH 后端选择 (OpenSSH vs Paramiko)
- 传输策略（远程 vs 本地）

## 核心文件说明

| 文件 | 职责 | 关键类/函数 |
|------|------|------------|
| `virtuoso_bridge/__init__.py` | 包入口 | 导出核心类 |
| `virtuoso_bridge/models.py` | 数据模型 | VirtuosoResult, ExecutionStatus |
| `virtuoso_bridge/env.py` | 环境变量 | load_vb_env() |
| `virtuoso_bridge/profile.py` | 配置管理 | resolve_profile() |
| `virtuoso/basic/bridge.py` | 核心客户端 | VirtuosoClient |
| `virtuoso/ops.py` | 通用操作 | open_cell_view, save_cell_view |
| `virtuoso/layout/__init__.py` | 版图操作 | LayoutOps, LayoutEditor |
| `virtuoso/schematic/__init__.py` | 原理图操作 | SchematicOps, SchematicEditor |
| `virtuoso/symbol/__init__.py` | 符号操作 | SymbolOps, SymbolEditor |
| `virtuoso/library/__init__.py` | 库管理 | LibraryOps |
| `virtuoso/maestro/__init__.py` | ADE/Maestro | MaestroOps |
| `transport/tunnel.py` | SSH 隧道 | SSHClient |
| `transport/remote.py` | 远程运行 | SSHRunner |
| `spectre/runner.py` | Spectre 仿真 | SpectreSimulator |
| `spectre/psf.py` | PSF 解析 | parse_psf() |
| `cli.py` | 命令行接口 | main() |

---

**注意**：这些 Mermaid 图表可以使用支持 Mermaid 的 Markdown 查看器（如 GitHub、VS Code with Mermaid 插件、Typora 等）来渲染。
