# Harness 要素之沙箱隔离：让 Agent 只能在围栏里干活

> Agent 能执行命令、能写文件、能调接口——这些能力在没有隔离的情况下，意味着它可以直接操作你的生产数据库、删掉你的本地文件、往外发网络请求。沙箱隔离要解决的，就是给 Agent 的执行环境画一个物理边界：你可以在里面随便折腾，但出不去。

## 核心判断

一句话分清：

- **没有沙箱 = 让实习生直接操作生产环境。** 它可能很谨慎，但你不敢赌。
- **有沙箱 = 给实习生一台独立的测试机。** 它怎么折腾都不影响真实环境，出了问题重置就行。

prompt 里写"不要删除重要文件"和把 Agent 关进一个只读的容器里，是两码事。前者靠自觉，后者靠物理隔离。

## 没有沙箱的 Agent 会出什么事

这些场景都发生过：

| 风险类型 | 具体场景 |
|---------|---------|
| 文件系统破坏 | Agent 为了"清理项目"执行了 `rm -rf node_modules`，但路径拼错了 |
| 环境变量泄露 | Agent 把 `.env` 里的数据库密码打印到了调试日志里 |
| 网络外泄 | Agent 调用了外部 API 把项目代码片段发了出去 |
| 进程干扰 | Agent 启动的子进程占满了 CPU，宿主机卡死 |
| 交叉污染 | 多个 Agent 任务共用同一个工作目录，互相覆盖文件 |

这些问题的共同点是：Agent 的执行环境和你的真实环境之间没有隔离边界。不是 Agent 想搞破坏，而是它的"工作台"就是你的"手术台"——一个失手就是事故。

## 沙箱在 Harness 架构中的位置

五个 Harness 要素里，约束定义"不能做什么"，沙箱隔离保证"做了也影响不到外面"。两者是互补的两层防御：

```text
┌─────────────────────────────────────────────┐
│   沙箱隔离（OS 级物理边界，包围一切执行）      │
│  文件系统隔离 │ 网络隔离 │ 进程隔离 │ 资源限制  │
│  ┌─────────────────────────────────────┐    │
│  │         Harness 约束层（逻辑门禁）    │    │
│  │  ┌─────────────────────────────┐    │    │
│  │  │ 上下文管理 │ 反馈回路 │ 可观测性│    │    │
│  │  │  ┌─────────────────────┐   │    │    │
│  │  │  │  Agent（执行层）     │   │    │    │
│  │  │  └─────────────────────┘   │    │    │
│  │  └─────────────────────────────┘    │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

沙箱是最外层的物理围栏——即使约束层的某条规则被绕过（比如 prompt 注入导致约束失效），Agent 仍然出不了沙箱的边界，因为它根本没有权限触碰沙箱外面的东西。约束层是逻辑防线（代码级拦截），沙箱是物理防线（OS 级隔离），两层配合互为兜底。

## 沙箱隔离的四个维度

### 1. 文件系统隔离：Agent 只能看到自己的工作目录

最基本的隔离。Agent 的所有文件操作都限制在一个独立目录里，看不到也碰不到宿主机的其他文件。

用 Docker 实现最直接：

```dockerfile
# Dockerfile.agent-sandbox
FROM python:3.12-slim

# 创建隔离的工作目录
RUN useradd -m -s /bin/bash agent && \
    mkdir -p /workspace && \
    chown agent:agent /workspace

# 只安装必要的工具
RUN pip install --no-cache-dir openai requests

USER agent
WORKDIR /workspace

# Agent 只能操作 /workspace 下的文件
```

构建并运行：

```bash
# 构建沙箱镜像
docker build -f Dockerfile.agent-sandbox -t agent-sandbox .

# 把项目代码挂载进去，但只读
docker run --rm \
  -v $(pwd)/src:/workspace/src:ro \
  -v $(pwd)/output:/workspace/output:rw \
  agent-sandbox \
  python run_agent.py
```

关键点：
- 项目源码以只读模式挂载（`:ro`），Agent 只能读不能改原始文件
- Agent 的输出写到独立的 `output` 目录
- 宿主机的 `.env`、`secrets`、`.git` 根本没有挂进去——Agent 连看都看不到

### 2. 网络隔离：控制 Agent 能访问什么

Agent 如果能随意发网络请求，就可能把敏感信息外泄，或者调用未授权的外部服务。

```bash
# 完全禁止网络访问（最安全，适合纯本地分析任务）
docker run --rm --network none \
  -v $(pwd)/src:/workspace/src:ro \
  agent-sandbox \
  python run_agent.py
```

完全禁网是最简单也最安全的策略。但如果 Agent 需要调用 LLM API 或访问内部 Git 仓库，就需要更精细的控制。

**推荐做法：用网络代理控制出站流量。** 在容器外运行一个正向代理（如 squid 或 mitmproxy），只放行白名单域名：

```bash
# 启动白名单代理（示例，仅允许 api.openai.com 和 gitlab.internal）
docker run -d --name agent-proxy \
  -v $(pwd)/proxy-allowlist.conf:/etc/squid/squid.conf:ro \
  -p 3128:3128 \
  ubuntu/squid

# Agent 容器通过代理上网
docker run --rm \
  --network agent-net \
  -e HTTP_PROXY=http://agent-proxy:3128 \
  -e HTTPS_PROXY=http://agent-proxy:3128 \
  -v $(pwd)/src:/workspace/src:ro \
  agent-sandbox \
  python run_agent.py
```

> **注意**：仅靠 `--add-host` 或 `--dns` 参数无法真正实现网络白名单。它们只控制了 DNS 解析层，容器仍然可以用 IP 直接访问其他地址。可靠的网络隔离需要在网络层拦截——要么用代理，要么用 iptables 规则。Claude Code 的做法是在沙箱外运行一个域名白名单代理，所有出站连接必须经过代理验证。

### 3. 进程和资源隔离：防止 Agent 吃光系统资源

Agent 可能会启动大量子进程、消耗大量内存，甚至陷入死循环。

```bash
docker run --rm \
  --cpus="1.0" \            # 最多用 1 个 CPU 核心
  --memory="512m" \          # 最多用 512MB 内存
  --pids-limit=50 \          # 最多 50 个进程
  --ulimit nofile=256:256 \  # 限制打开的文件数
  --read-only \              # 根文件系统只读
  --tmpfs /tmp:size=100m \   # 临时目录限制 100MB
  agent-sandbox \
  python run_agent.py
```

这些限制确保即使 Agent 写了一个无限循环或者 fork 炸弹，也不会影响宿主机。最坏情况就是容器被 OOM 杀掉，宿主机毫发无损。

### 4. 会话隔离：多任务互不干扰

当多个 Agent 任务并行运行时，每个任务必须有独立的执行环境，互相看不到对方的文件和状态。

```python
# sandbox_manager.py
import subprocess
import uuid

class SandboxManager:
    """每个任务分配一个独立的沙箱容器"""

    def create(self, task_id: str = None) -> str:
        sandbox_id = task_id or str(uuid.uuid4())[:8]
        container_name = f"agent-sandbox-{sandbox_id}"

        subprocess.run([
            "docker", "run", "-d",
            "--name", container_name,
            "--cpus=1.0",
            "--memory=512m",
            "--network", "none",
            "--read-only",
            "--tmpfs", "/tmp:size=100m",
            "agent-sandbox",
            "sleep", "infinity",  # 保持容器运行
        ], check=True)

        return container_name

    def execute(self, container: str, command: str) -> str:
        result = subprocess.run(
            ["docker", "exec", container, "bash", "-c", command],
            capture_output=True, text=True, timeout=60,
        )
        return result.stdout

    def destroy(self, container: str):
        subprocess.run(["docker", "rm", "-f", container], check=True)
```

每个任务一个容器，任务完成后销毁。两个 Agent 任务之间不存在共享状态，从根源上消除交叉污染。

## 实战操作：用 Docker 搭一个最小沙箱

下面用一个完整的例子把上面的内容串起来。场景是：让 Agent 在沙箱里分析一个 Python 项目的代码质量。

### 第一步：创建沙箱 Dockerfile

```dockerfile
# Dockerfile.sandbox
FROM python:3.12-slim

RUN useradd -m agent && \
    mkdir -p /workspace /output && \
    chown agent:agent /workspace /output

# 安装代码分析工具
RUN pip install --no-cache-dir flake8 pylint

USER agent
WORKDIR /workspace
```

### 第二步：写一个启动脚本

```bash
#!/bin/bash
# run-in-sandbox.sh
# 用法: ./run-in-sandbox.sh <项目目录> <分析命令>

PROJECT_DIR="${1:-.}"
COMMAND="${2:-flake8 /workspace/src}"

docker build -f Dockerfile.sandbox -t code-sandbox . 2>/dev/null

echo "启动沙箱..."
docker run --rm \
  --name agent-sandbox-$(date +%s) \
  --cpus="1.0" \
  --memory="512m" \
  --network none \
  --read-only \
  --tmpfs /tmp:size=50m \
  -v "${PROJECT_DIR}/src":/workspace/src:ro \
  -v "${PROJECT_DIR}/output":/output:rw \
  code-sandbox \
  bash -c "${COMMAND} > /output/report.txt 2>&1; echo \"Exit: \$?\" >> /output/report.txt"

echo "分析完成，报告在 ${PROJECT_DIR}/output/report.txt"
```

### 第三步：运行

```bash
chmod +x run-in-sandbox.sh
mkdir -p output

# 运行沙箱分析
./run-in-sandbox.sh . "flake8 /workspace/src --max-line-length=120"
```

跑完之后，可以逐项验证隔离效果——容器内访问宿主机 home 目录、curl 外网、吃光内存，分别对应文件系统、网络、资源三个维度的拦截。结果汇总：

### 运行效果对比

| 维度 | 无沙箱 | 有沙箱 |
|------|-------|--------|
| Agent 能看到 `.env` | 能 | 不能（没有挂载进去） |
| Agent 能改源码 | 能 | 不能（只读挂载） |
| Agent 能访问外网 | 能 | 不能（`--network none`） |
| Agent 进程失控 | 影响宿主机 | 容器被杀，宿主机无影响 |
| 多任务互相干扰 | 共享文件系统 | 每个任务独立容器 |

## 什么时候需要沙箱，什么时候不需要

| 场景 | 是否需要沙箱 | 原因 |
|------|------------|------|
| 纯文本生成（不执行代码） | 不需要 | Agent 没有执行能力，不存在隔离需求 |
| 读取文件做分析（只读） | 通常不需要 | 但如果文件包含敏感信息，建议隔离 |
| 执行代码、运行命令 | 需要 | 任何代码执行都可能产生副作用 |
| 访问数据库或外部 API | 需要 | 必须控制访问范围和权限 |
| 多 Agent 并行任务 | 必须有 | 防止交叉污染 |
| 输出进入生产环境 | 必须有 | 隔离是最后一道防线 |

判断标准：**Agent 有没有"动手能力"？** 只要它能执行代码、写文件或发网络请求，就应该考虑沙箱。

## Docker 不是唯一选项：沙箱方案选型

前面的实战部分全部基于 Docker，因为它门槛最低、上手最快。但 Docker 容器共享宿主机内核，存在内核漏洞逃逸风险（历史上出现过 CVE-2019-5736 等案例）。对于多租户、公网暴露或运行完全不可信代码的场景，Docker 可能不够。

2026 年的 Agent 沙箱生态已经分化为多个层级：

| 方案 | 隔离原理 | 安全强度 | 启动延迟 | 适用场景 |
|------|---------|---------|---------|---------|
| **Docker 容器** | Linux Namespace + Cgroup | 中等（共享内核） | ~500ms | 本地开发、内部工具、可信用户 |
| **gVisor** | 用户态 Linux 内核，拦截系统调用 | 中高 | ~100ms | 需要比容器更强但不需要完全 VM 隔离 |
| **Firecracker 微虚拟机** | 基于 KVM 的独立内核 | 极强（内核级隔离） | ~150ms | 多租户、公网暴露、不可信代码（AWS Lambda、E2B 在用） |
| **Wasm** | 语言层面线性内存隔离 | 强 | 毫秒级 | 仅 JS/TS 场景，资源敏感的轻量 Agent |
| **OS 原生沙箱** | macOS Seatbelt / Linux bubblewrap | 强 | 极快 | 本地 Agent 工具（Claude Code 在用） |

**选型判断标准**：问一个问题——"如果这个沙箱被攻破，最坏结果是什么？"

- 最坏结果是自己的开发环境重置 → Docker 容器够用
- 最坏结果是其他用户的数据泄露 → 至少 gVisor，建议 Firecracker
- 最坏结果是公网被攻击者利用 → 必须 Firecracker 或同等级的内核隔离

## Claude Code 的沙箱：一个工业级参考

同项目的文章已经分析过 Claude Code 的 Harness 架构。这里单独看它的沙箱实现，因为它的做法与前面的 Docker 方案有本质区别。

Claude Code **没有用 Docker 容器**，而是直接使用操作系统原生的沙箱机制：

- **Linux**：使用 **bubblewrap**（轻量级沙箱工具，不需要 root 权限）
- **macOS**：使用 **Seatbelt**（macOS 内置的沙箱框架）

具体隔离策略：

1. **文件系统**：限制 Claude 只能读写当前工作目录，阻止访问 SSH 密钥、系统配置等敏感文件
2. **网络**：通过沙箱外运行的代理服务器控制出站连接。Agent 的所有网络请求先经过代理，代理强制执行域名白名单——只有预批准的域名才能连通
3. **子进程覆盖**：隔离不仅覆盖 Claude Code 自身，还覆盖它生成的任何脚本、程序或子进程

效果数据：启用沙箱后，权限确认请求减少了 **84%**。也就是说，沙箱让 Agent 在预设边界内自主运行，大幅减少了需要人工干预的中断。

这个设计体现了一个关键思路：**沙箱不只是安全防线，还是效率工具。** 隔离做得好，Agent 的自主性反而更高——因为你敢放手让它干活了。

## 沙箱与其他四个 Harness 要素的关系

- **沙箱 + 约束**：约束是逻辑门禁（代码层拦截），沙箱是物理围栏（OS 层隔离）。两层防御，任何一层失效另一层还能兜住。
- **沙箱 + 上下文管理**：沙箱隔离了执行环境，上下文管理隔离了信息环境。一个管 Agent 能碰什么，一个管 Agent 能看到什么。
- **沙箱 + 反馈回路**：Agent 在沙箱里执行的结果（成功/失败/超时）就是天然的反馈信号。容器退出码、执行日志、资源使用情况都可以作为反馈输入。
- **沙箱 + 可观测性**：容器级别的监控（CPU、内存、网络流量、进程数）提供了细粒度的可观测数据。Agent 在沙箱里的每一步操作都可以被记录和审计。

## 延伸阅读

本文是 Harness Engineering 五要素系列的一部分。其他四个要素：

- [Harness 要素之约束：不是写在 prompt 里的建议，而是编进流程的硬卡控](Harness要素之约束：不是写在prompt里的建议，而是编进流程的硬卡控.md)
- [Harness 要素之上下文管理：Agent 跑偏的根源是喂错了信息](Harness要素之上下文管理：Agent跑偏的根源是喂错了信息.md)
- [Harness 要素之反馈回路：不是跑完就算，而是跑歪了能自动拉回来](Harness要素之反馈回路：不是跑完就算，而是跑歪了能自动拉回来.md)
- [Harness 要素之可观测性：Agent 在想什么，你必须看得见](Harness要素之可观测性：Agent在想什么，你必须看得见.md)

Claude Code 的 Harness 架构全景分析：

- [Claude Code 就是一个 Harness：从泄露的 51 万行源码看 Agent 的运行底座](Claude-Code就是一个Harness：从泄露源码看Agent的运行底座.md)

---

*本文由本人构思并把控，借助 AI 辅助整理成文，仅代表个人观点，欢迎交流。*
