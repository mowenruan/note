# Terminal-Bench 内容与测试方法介绍

> 本文档用于快速理解 Terminal-Bench 是什么、它测试哪些能力，以及如何在本地或评测环境中运行测试。资料整理时间：2026-05-22。Terminal-Bench 和 Harbor 仍在快速迭代，实际命令请以官方文档和 registry 为准。

## 1. Terminal-Bench 是什么

Terminal-Bench 是一个面向 AI Agent 的终端任务基准测试。它不只考察模型能否写出一段代码，而是考察 Agent 能否在真实或接近真实的命令行环境中，自主完成端到端任务。

一个典型任务会要求 Agent 使用 shell、文件系统、编译器、解释器、服务进程、数据文件、调试工具等资源，在受控的 sandbox 中完成目标。完成后，系统会运行隐藏或独立的测试脚本，检查最终状态是否满足要求。

官方仓库对 Terminal-Bench 的定位是：用真实终端环境评估 AI Agent，从编译代码、训练模型到配置服务器，测试 Agent 是否能够自主处理复杂任务。

## 2. Terminal-Bench 主要包含什么

Terminal-Bench 可以理解为两部分：

1. **任务数据集**

   每个任务都是一个独立的小型工作场景，通常包含自然语言说明、运行环境、参考解法和验证测试。Terminal-Bench 2.0 论文中提到的版本包含 89 个精心筛选的任务；官方仓库和后续 registry 中的任务数量会随版本变化。

2. **执行框架**

   执行框架负责创建隔离环境，将任务说明交给 Agent，允许 Agent 与终端交互，并在 Agent 完成后运行测试。早期主要使用 `terminal-bench` / `tb` CLI；Terminal-Bench 2.0 的官方推荐运行方式逐渐转向 Harbor harness。

## 3. 它测试哪些能力

Terminal-Bench 更关注“能不能把事情做完”，而不是单步问答。常见能力包括：

- **终端操作能力**：浏览目录、读取文件、执行命令、管理进程、处理权限和环境变量。
- **代码理解与修改能力**：定位 bug、修改源码、补齐脚本、调整配置、运行测试。
- **构建与依赖处理能力**：安装依赖、编译项目、解决版本或系统库问题。
- **数据处理能力**：清洗数据、转换格式、抽取信息、生成指定输出文件。
- **系统与服务配置能力**：配置 Web 服务、数据库、证书、日志、网络端口等。
- **调试与恢复能力**：分析错误日志、修复损坏文件、恢复 Git 仓库或数据库状态。
- **长程规划能力**：在多步任务中决定先做什么、如何验证、失败后如何调整。

这些能力组合起来，比传统的代码补全或单元测试 benchmark 更接近真实工程工作流。

## 4. 一个任务的基本结构

传统 Terminal-Bench 任务通常包含以下文件：

```text
task-id/
  Dockerfile
  docker-compose.yaml
  task.yaml
  solution.sh 或 solution.yaml
  run-tests.sh
  tests/
    test_outputs.py
  其他任务依赖文件
```

各文件作用如下：

- `task.yaml`：任务说明和元信息，例如自然语言描述、难度、标签、超时时间、测试脚本配置等。
- `Dockerfile` / `docker-compose.yaml`：定义 Agent 所处的隔离终端环境。简单任务可能只有一个容器，复杂任务可以有多个服务容器。
- `solution.sh` / `solution.yaml`：人工编写的参考解法，也常被称为 oracle solution，用于确认任务本身可解。
- `run-tests.sh`：测试入口脚本，负责安装测试依赖并执行验证逻辑。
- `tests/test_outputs.py`：常见的 pytest 测试文件，用于检查最终文件、命令输出、服务状态或程序行为。

在 Harbor 格式中，任务结构有变化，例如：

```text
task-id/
  instruction.md
  task.toml
  environment/
    Dockerfile
  solution/
    solve.sh
  tests/
    test.sh
```

两种结构的核心思想一致：任务说明、隔离环境、参考解法、最终验证。

## 5. 评测流程

Terminal-Bench 的一次评测通常按下面流程进行：

1. **准备环境**

   执行框架读取任务配置，构建或拉取 Docker 镜像，启动隔离 sandbox。

2. **下发任务**

   Agent 只看到任务说明和工作环境。它需要自己探索文件、运行命令并完成任务。

3. **Agent 操作终端**

   Agent 可以执行 shell 命令、编辑文件、启动服务、运行调试脚本。框架会记录交互日志、命令输出和耗时。

4. **运行验证测试**

   Agent 结束后，框架运行测试脚本。传统 Terminal-Bench 中，测试资源通常会在 Agent 运行后复制到测试目录，降低 Agent 直接针对测试文件硬编码的可能性。

5. **生成结果**

   每个任务通常会得到 pass / fail、测试日志、执行轨迹、超时信息等结果。汇总后可以计算 Agent 在某个数据集版本上的通过率。

## 6. 测试方法一：使用 Harbor 运行 Terminal-Bench 2.0

Harbor 是 Terminal-Bench 团队推出的新执行框架。官方 Harbor 文档说明，Harbor 是运行 Terminal-Bench 2.0 的官方 harness。

### 6.1 前置条件

本地需要：

- Docker 已安装并正在运行。
- Python / `uv` 可用。
- 如果测试真实模型 Agent，需要配置对应模型供应商的 API key。

### 6.1.1 在 Linux 服务器安装 Docker

Linux 服务器上通常安装的是 **Docker Engine**，不是 Docker Desktop。先确认服务器发行版：

```bash
cat /etc/os-release
```

如果是 Ubuntu，推荐使用 Docker 官方 apt 仓库安装：

```bash
# 1. 安装基础依赖
sudo apt update
sudo apt install -y ca-certificates curl

# 2. 添加 Docker 官方 GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 3. 添加 Docker 官方 apt 源
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

# 4. 安装 Docker Engine、CLI、Buildx 和 Compose 插件
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5. 启动并设置开机自启
sudo systemctl enable --now docker

# 6. 验证安装
sudo docker run hello-world
docker compose version
```

如果是 Debian，步骤基本相同，但第 2、3 步里的仓库地址要把 `ubuntu` 改成 `debian`：

```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
```

```text
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
```

如果是 CentOS Stream、Rocky Linux、AlmaLinux 或其他 RHEL 系发行版，通常使用 dnf 安装：

```bash
# 1. 安装 dnf 仓库管理工具
sudo dnf -y install dnf-plugins-core

# 2. 添加 Docker 官方仓库
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 3. 安装 Docker Engine、CLI、Buildx 和 Compose 插件
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 4. 启动并设置开机自启
sudo systemctl enable --now docker

# 5. 验证安装
sudo docker run hello-world
docker compose version
```

如果服务器是官方 RHEL，可以把仓库地址换成：

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

默认情况下，普通用户可能需要 `sudo` 才能执行 `docker` 命令。如果希望当前用户直接运行 Docker，可以把用户加入 `docker` 组：

```bash
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

注意：`docker` 组等价于给该用户很高的主机权限。单人实验服务器通常可以这样做；共享服务器上建议先确认安全策略。

常见问题：

- `Cannot connect to the Docker daemon`：Docker 服务没启动，执行 `sudo systemctl status docker` 或 `sudo systemctl start docker`。
- `permission denied while trying to connect to the Docker daemon socket`：当前用户没有 Docker 权限，使用 `sudo docker ...`，或加入 `docker` 组后重新登录。
- `docker compose: command not found`：没有安装 Compose 插件，确认已安装 `docker-compose-plugin`，并使用新命令 `docker compose`，不是旧命令 `docker-compose`。
- 镜像拉取很慢或失败：检查服务器网络、代理、防火墙，以及是否能访问 Docker Hub。

安装 Harbor：

```bash
uv tool install harbor
```

检查命令：

```bash
harbor --help
harbor dataset list
```

### 6.2 使用 oracle 解法做冒烟测试

先跑官方参考解法，确认本机 Docker、数据集下载和 harness 都正常：

```bash
harbor run -d terminal-bench/terminal-bench-2 -a oracle
```

如果 oracle 都无法通过，通常说明问题在环境、依赖、镜像下载或数据集配置，而不是 Agent 本身。

### 6.3 测试一个真实 Agent

下面是 Harbor 官方示例风格的命令，实际模型名、Agent 名和并发数应按自己的环境调整：

```bash
export ANTHROPIC_API_KEY="<your-anthropic-api-key>"

harbor run \
  -d terminal-bench/terminal-bench-2 \
  -m anthropic/claude-haiku-4-5 \
  -a claude-code \
  -n 4
```

常见参数含义：

- `-d`：指定注册数据集。
- `-m`：指定模型。
- `-a`：指定 Agent 适配器。
- `-n`：并发运行的任务数量。
- `-t`：如果当前 Harbor 版本支持，可以指定单个任务 id 运行。

在 Harbor registry 中，也能看到类似下面的单任务运行方式：

```bash
uvx harbor run -d terminal-bench@2.0 -t <task-id>
```

不同版本中数据集 id 可能是 `terminal-bench/terminal-bench-2` 或 `terminal-bench@2.0`。实际使用时建议先运行：

```bash
harbor dataset list
harbor run --help
```

## 7. 测试方法二：使用传统 `tb` CLI

Terminal-Bench 仍提供 `terminal-bench` Python 包和 `tb` CLI。它适合查看旧版任务结构、运行 `terminal-bench-core` 数据集、开发自定义 Agent 或自定义任务。

### 7.1 安装

```bash
uv tool install terminal-bench
```

也可以使用 pip：

```bash
pip install terminal-bench
```

检查：

```bash
tb --help
tb datasets list
```

### 7.2 运行一个任务

官方文档中的示例命令：

```bash
tb run \
  --dataset terminal-bench-core==head \
  --agent terminus \
  --model anthropic/claude-sonnet-4-20250514 \
  --task-id hello-world
```

常见参数含义：

- `--dataset`：指定数据集及版本，例如 `terminal-bench-core==head`。
- `--agent`：使用内置 Agent。
- `--model`：指定模型。
- `--task-id`：只运行某个任务，适合调试。
- `--n-concurrent`：并发任务数，适合批量评测。

### 7.3 测试自定义 Agent

如果要测试自己的 Agent，需要实现 Terminal-Bench 的 Agent 接口。传统 CLI 文档中提到可以实现 `BaseAgent`，核心方法接收任务描述和 tmux session，然后由 Agent 自己与终端交互。

简化结构如下：

```python
from terminal_bench.agents import BaseAgent, AgentResult
from terminal_bench.terminal.tmux_session import TmuxSession


class YourCustomAgent(BaseAgent):
    @staticmethod
    def name() -> str:
        return "your-agent-name"

    def perform_task(
        self,
        task_description: str,
        session: TmuxSession,
        logging_dir=None,
    ) -> AgentResult:
        # 在这里读取任务说明、向 tmux session 发送命令、收集结果。
        ...
```

运行时通过 `--agent-import-path` 指定自定义类：

```bash
tb run \
  --dataset terminal-bench-core==head \
  --agent-import-path path.to.your.agent:YourCustomAgent \
  --task-id hello-world
```

## 8. 如何判断测试是否有效

为了让评测结果有可信度，建议按下面顺序检查：

1. **先跑 oracle**

   参考解法应该稳定通过。如果 oracle 不通过，优先修复任务环境或测试脚本。

2. **单任务调试**

   先用 `--task-id` 或 `-t` 跑一个小任务，确认 Agent 能正常进入环境、执行命令、产生日志。

3. **查看日志**

   重点看：

   - Agent 是否理解任务目标。
   - 是否卡在依赖安装、权限、路径或网络问题。
   - 是否在测试前完成了要求的文件或服务状态。
   - 失败是断言失败、命令失败还是超时。

4. **控制并发**

   本地 Docker 资源有限时，不要一开始就高并发。先从 `-n 1` 或小并发开始，确认稳定后再扩大。

5. **固定数据集版本**

   对比不同 Agent 时，应固定 Terminal-Bench 数据集版本、模型版本、Agent 版本和运行参数。否则通过率不具备可比性。

6. **多次运行高波动任务**

   有些 Agent 或模型存在随机性。对关键任务可以多次运行，记录平均通过率和失败类型。

## 9. 常见结果指标

Terminal-Bench 最核心的指标是：

- **Pass rate**：通过任务数 / 总任务数。
- **Per-task result**：每个任务的 pass / fail / timeout。
- **Timeout rate**：超时比例，反映 Agent 是否能在限制内完成长程任务。
- **Failure category**：失败原因，例如理解错误、环境错误、代码错误、测试未通过、依赖失败。
- **Trajectory / logs**：Agent 的终端操作轨迹，用于分析行为质量。

排行榜通常更关注固定数据集版本上的整体通过率；研究或开发 Agent 时，日志和失败类型同样重要。

## 10. 使用建议

- 如果目标是复现 Terminal-Bench 2.0 或对接新生态，优先使用 Harbor。
- 如果目标是理解旧任务格式、开发传统 Terminal-Bench 任务或使用 `tb` CLI 生态，可以使用 `terminal-bench` 包。
- 如果只是验证环境，先跑 oracle，不要直接跑昂贵模型。
- 如果要比较多个 Agent，务必固定数据集版本和运行参数。
- 如果要写新任务，测试脚本应检查最终行为，而不是只检查表面文件名；任务说明也要明确所有会被测试依赖的输出格式、路径和边界条件。

## 11. 参考资料

- Terminal-Bench GitHub 仓库：https://github.com/harbor-framework/terminal-bench
- Terminal-Bench 任务结构文档：https://www.tbench.ai/docs/task-overview
- Terminal-Bench 安装文档：https://www.tbench.ai/docs/installation
- Terminal-Bench First Steps：https://www.tbench.ai/docs/first-steps
- Harbor 运行 Terminal-Bench 文档：https://www.harborframework.com/docs/tutorials/running-terminal-bench
- Harbor Getting Started：https://www.harborframework.com/docs/getting-started
- Terminal-Bench 2.0 论文：https://arxiv.org/abs/2601.11868
