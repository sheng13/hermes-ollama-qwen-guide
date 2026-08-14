# Hermes Agent + Ollama + Qwen + Docker：从零到可用完整教程

> 适用环境：Ubuntu 22.04 / 24.04（实体机或虚拟机）  
> 架构：Hermes Agent 与 Ollama 各自在一个 Docker 容器，通过 Compose 私有网络通信。  
> 更新基准：2026-08-14。安装前建议再次查看文末官方文档。

## 1. 完成后会得到什么

```text
你（终端、API 或聊天平台）
             │
             ▼
Hermes Agent 容器 :8642
  ├─ 规划任务
  ├─ 调用工具
  ├─ 操作文件／终端
  └─ 维护记忆与会话
             │ OpenAI 兼容 API
             ▼
Ollama 容器 :11434
             │
             ▼
Qwen 本地大语言模型
```

特点：

- 模型在本机运行，普通对话不需要付 API 费用。
- Hermes 负责 Agent 工作流和工具，Ollama 负责模型推理，Qwen 是实际 LLM。
- 模型和 Hermes 数据都写入持久卷，升级容器不会丢失。
- 默认只监听 `127.0.0.1`，不会直接暴露到局域网或互联网。

## 2. 四个名称分别是什么

| 名称 | 作用 |
|---|---|
| Docker | 隔离、启动和管理服务 |
| Docker Compose | 用一份 YAML 同时管理 Hermes 与 Ollama |
| Ollama | 下载、保存并运行本地模型，提供 API |
| Qwen | 阿里 Qwen 系列大语言模型 |
| Hermes Agent | 调用 LLM、规划任务、调用终端与其他工具 |

Hermes 不是模型，Ollama 也不是模型。正确关系是：**Hermes 调用 Ollama，Ollama 加载 Qwen。**

## 3. 硬件需求与模型选择

先查看硬件：

```bash
lscpu | sed -n '1,20p'
free -h
df -h /
lspci | grep -Ei 'VGA|3D|NVIDIA|AMD'
nvidia-smi  # 只有 NVIDIA 驱动正常时才有结果
```

建议：

| 可用 RAM/VRAM | 建议模型 | 说明 |
|---|---|---|
| 8–12 GB RAM | `qwen3:4b` | 能聊天，但 Agent 工具能力较弱 |
| 16 GB RAM | `qwen3:8b` | 学习与测试；长上下文可能很慢或内存不足 |
| 32 GB RAM | `qwen3.5:27b` 或较小量化版 | 推荐起点，工具调用更可靠 |
| 48 GB+ RAM/VRAM | `qwen2.5-coder:32b` / 更大 Qwen | 编程与复杂 Agent 任务 |

重要限制：Hermes Agent 的工具工作流要求至少 **64,000 tokens 上下文**。模型本身、量化格式和可用内存必须能承担这个长度。小模型即使能启动，也不代表能可靠调用工具。CPU 可以运行，但第一条回复可能需要几分钟。

如果你的电脑只有 16 GB RAM，先把本文的 `qwen3.5:27b` 全部替换为 `qwen3:8b` 做功能验证；若 Hermes 拒绝模型上下文或频繁内存不足，需要改用更合适硬件或云端模型。

## 4. 安装 Docker Engine（Ubuntu）

### 4.1 移除冲突套件

这一步不会删除 `/var/lib/docker` 里的既有镜像与卷，但生产机仍应先备份。

```bash
sudo apt remove -y docker.io docker-compose docker-compose-v2 \
  docker-doc podman-docker containerd runc
```

如果提示某些套件没有安装，可以忽略。

### 4.2 加入 Docker 官方 apt 仓库

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

建立来源文件：

```bash
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

安装：

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

验证：

```bash
sudo docker run --rm hello-world
sudo docker version
sudo docker compose version
```

### 4.3 可选：让当前用户不用 sudo 执行 Docker

注意：`docker` 群组几乎等同 root 权限，只给可信用户。

```bash
sudo usermod -aG docker "$USER"
newgrp docker
```

重新登录后验证：

```bash
docker run --rm hello-world
```

如果不加入群组，后续每条 `docker` 命令前加 `sudo`。

## 5. NVIDIA GPU 支持（CPU 用户跳过）

先确保宿主机 `nvidia-smi` 正常。然后安装 NVIDIA Container Toolkit：

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o \
  /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -fsSL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

验证容器能看到 GPU：

```bash
docker run --rm --gpus all ubuntu nvidia-smi
```

虚拟机必须另外启用 GPU passthrough；普通虚拟显卡不能让 Ollama 使用实体 NVIDIA GPU。

## 6. 准备本教程文件

本教程目录已经包含：

```text
hermes-ollama-qwen/
├── compose.yaml              # CPU 与通用配置
├── compose.nvidia.yaml       # NVIDIA GPU 覆盖配置
├── .env.example              # 密钥范例
├── Modelfile                 # 固定 Qwen 64K 上下文
└── hermes-data/
    └── config.yaml           # Hermes → Ollama 配置
```

进入目录：

```bash
cd /home/wsfoo/games/hermes-ollama-qwen
```

建立真正的 `.env`：

```bash
cp .env.example .env
openssl rand -hex 32
```

复制输出，编辑 `.env`：

```bash
nano .env
```

结果类似：

```dotenv
API_SERVER_KEY=4c9c...中间省略...e812
```

不要把 `.env` 上传到 GitHub。

## 7. 理解 Compose 网络

两个容器在 `hermes-net` 网络中。容器名称会变成内部 DNS 名称：

```text
Hermes 访问 Ollama：http://ollama:11434/v1
宿主机访问 Ollama：http://127.0.0.1:11434
宿主机访问 Hermes：http://127.0.0.1:8642
```

**容器里的 `localhost` 只代表该容器本身。** 因此 Hermes 配置不能写 `http://localhost:11434/v1`，必须写 `http://ollama:11434/v1`。

## 8. 启动 Ollama

CPU：

```bash
docker compose up -d ollama
```

NVIDIA GPU：

```bash
docker compose -f compose.yaml -f compose.nvidia.yaml up -d ollama
```

检查：

```bash
docker compose ps
docker compose logs --tail=100 ollama
curl http://127.0.0.1:11434/api/tags
```

看到 `"models": []` 代表服务正常，只是还没有模型。

## 9. 下载并测试 Qwen

推荐配置默认使用：

```bash
docker compose exec ollama ollama pull qwen3.5:27b
```

模型很大，下载时间取决于网络和磁盘。查看进度与现有模型：

```bash
docker compose exec ollama ollama list
```

先直接测试模型：

```bash
docker compose exec ollama ollama run qwen3.5:27b \
  "请用繁体中文解释 Docker 是什么，只回答三句话。"
```

退出交互模式按 `Ctrl+D` 或输入 `/bye`。

测试 OpenAI 兼容 API：

```bash
curl http://127.0.0.1:11434/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model":"qwen3.5:27b",
    "messages":[{"role":"user","content":"只回答 OK"}],
    "max_tokens":20
  }'
```

## 10. 强制使用 64K 上下文

Compose 已设：

```yaml
OLLAMA_CONTEXT_LENGTH: "64000"
```

更明确的方法是建立自定义模型：

```bash
docker compose exec -T ollama sh -c 'cat >/tmp/Modelfile' < Modelfile
docker compose exec ollama ollama create qwen-hermes-64k -f /tmp/Modelfile
```

然后把 `hermes-data/config.yaml` 的模型改成：

```yaml
model:
  default: qwen-hermes-64k
  provider: custom
  base_url: http://ollama:11434/v1
  context_length: 64000
```

让模型加载并检查实际上下文：

```bash
docker compose exec ollama ollama run qwen-hermes-64k "回答 OK"
docker compose exec ollama ollama ps
```

检查 `CONTEXT` 栏，应接近 `64000`。如果启动时内存不足，请换小模型；不要只把数值调大而忽略硬件需求。

## 11. 设置 Hermes Agent

配置文件已经位于：

```text
./hermes-data/config.yaml
```

内容：

```yaml
model:
  default: qwen3.5:27b
  provider: custom
  base_url: http://ollama:11434/v1
  context_length: 64000
```

如果使用刚建立的自定义模型，把 `default` 改成 `qwen-hermes-64k`。

也可以运行 Hermes 官方设置向导：

```bash
docker compose run --rm hermes setup
```

向导中选择：

```text
Provider：Custom endpoint
Base URL：http://ollama:11434/v1
API key：留空或填 no-key
Model：qwen3.5:27b 或 qwen-hermes-64k
Context length：64000
```

向导写入的资料保存在宿主机 `./hermes-data`，删除容器不会丢失。

## 12. 启动完整系统

CPU：

```bash
docker compose up -d
```

NVIDIA：

```bash
docker compose -f compose.yaml -f compose.nvidia.yaml up -d
```

查看状态：

```bash
docker compose ps
docker compose logs --tail=100 hermes
docker compose logs --tail=100 ollama
```

持续看日志：

```bash
docker compose logs -f hermes ollama
```

按 `Ctrl+C` 只会退出日志画面，不会停止容器。

## 13. 使用 Hermes

### 13.1 终端交互

启动临时 Hermes CLI，并连接相同持久资料：

```bash
docker compose run --rm --entrypoint hermes hermes
```

可以尝试：

```text
列出当前目录中的 Python 文件。
读取 README.md 并摘要。
建立 hello.py，内容打印 Hello Hermes，但执行前先让我确认。
```

Hermes 请求执行危险命令时会要求确认。不要在不理解命令时直接同意。

常用指令：

```text
/model                 查看或切换模型
/usage                 查看上下文使用量
/compress              压缩长对话
/new                    新会话
/quit                   离开
```

### 13.2 测试 Hermes API

Hermes Gateway 的 API 默认只绑定宿主机回环地址。读取 `.env` 中的密钥后：

```bash
set -a
source .env
set +a
curl http://127.0.0.1:8642/v1/models \
  -H "Authorization: Bearer $API_SERVER_KEY"
```

如果需要聊天端点，可查看当前 Hermes Gateway 文档确认请求格式。不要把密钥写进命令历史或截图。

### 13.3 停止、启动与重启

```bash
docker compose stop
docker compose start
docker compose restart
docker compose restart hermes
docker compose restart ollama
```

更新镜像：

```bash
docker compose pull
docker compose up -d
```

## 14. 管理模型和资料

列出模型：

```bash
docker compose exec ollama ollama list
```

切换模型：

1. `ollama pull 新模型`
2. 修改 `hermes-data/config.yaml`
3. `docker compose restart hermes`

删除某个模型：

```bash
docker compose exec ollama ollama rm 模型名称
```

查看 Docker 磁盘占用：

```bash
docker system df
docker volume ls
```

重要资料：

- Ollama 模型：Docker named volume `hermes-qwen_ollama-models`
- Hermes 配置、会话、记忆：`./hermes-data`
- 密钥：`./.env`

备份 Hermes：

```bash
tar -czf hermes-data-backup-$(date +%F).tar.gz hermes-data .env
```

备份文件包含秘密，不要公开分享。

## 15. 安全设置

当前 Compose 使用：

```yaml
127.0.0.1:11434:11434
127.0.0.1:8642:8642
```

这表示只有本机能访问。不要为了方便直接改成：

```yaml
0.0.0.0:8642:8642
```

除非同时完成：

- 强随机 `API_SERVER_KEY`
- HTTPS 反向代理
- 防火墙来源限制
- Hermes 用户／平台 allowlist
- 定期更新与日志审计

Hermes 能执行终端和修改文件，权限比普通聊天机器人高。不要把 Docker socket、整个 `/`、SSH 私钥或敏感目录挂进 Hermes 容器。

`.gitignore` 至少包含：

```gitignore
.env
hermes-data/.env
hermes-data/sessions/
hermes-data/memories/
*.tar.gz
```

## 16. 常见故障排除

### 16.1 `Cannot connect to the Docker daemon`

```bash
sudo systemctl status docker
sudo systemctl start docker
```

如果用非 root 用户：

```bash
id
groups
```

确认包含 `docker`，然后重新登录。

### 16.2 Hermes 显示 `Connection refused`

从 Hermes 容器内测试 Ollama：

```bash
docker compose exec hermes curl -s http://ollama:11434/v1/models
```

若失败：

```bash
docker compose ps
docker compose logs ollama
docker network inspect hermes-qwen_hermes-net
```

确认 URL 是 `http://ollama:11434/v1`，不是 `localhost`。

### 16.3 `model not found`

```bash
docker compose exec ollama ollama list
```

`config.yaml` 中的名称必须完全一致，包括冒号后的 tag。

### 16.4 Hermes 报上下文小于 64K

```bash
docker compose exec ollama ollama ps
cat hermes-data/config.yaml
```

确认 Ollama 实际 CONTEXT 和 Hermes `context_length` 都为 64000。必要时使用第 10 节 Modelfile 建立自定义模型。

### 16.5 容器被系统杀掉或出现 exit code 137

通常是内存不足：

```bash
free -h
dmesg -T | grep -Ei 'out of memory|killed process'
docker stats
```

换较小模型，关闭其他程序，或增加 RAM/swap。大上下文 KV cache 也会消耗大量内存。

### 16.6 CPU 很慢

```bash
docker compose exec ollama ollama ps
docker stats
```

CPU 模式是正常可用但较慢。设置的 64K 上下文会让首次 prefill 更久。可以换小模型，但仍要满足 Hermes 的最低上下文要求。

### 16.7 NVIDIA GPU 没被使用

```bash
nvidia-smi
docker run --rm --gpus all ubuntu nvidia-smi
docker compose -f compose.yaml -f compose.nvidia.yaml exec ollama ollama ps
```

若宿主机正常而容器失败，重新执行：

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 16.8 工具调用变成普通文字

原因通常是模型工具能力不足，而不是 Hermes 坏掉。

- 先确认所选 Qwen tag 支持 tools。
- 换更大、更新的工具调用模型。
- 查看 Hermes 日志中的 tool-call parse 错误。
- 小模型适合聊天，不一定适合自主 Agent。

### 16.9 修改配置没有生效

```bash
docker compose restart hermes
docker compose logs --tail=100 hermes
```

确认编辑的是宿主机的 `./hermes-data/config.yaml`。

## 17. 完整验收清单

逐项执行：

```bash
docker version
docker compose version
docker compose ps
curl http://127.0.0.1:11434/api/tags
docker compose exec ollama ollama list
docker compose exec ollama ollama ps
docker compose exec hermes curl -s http://ollama:11434/v1/models
docker compose logs --tail=50 hermes
```

最终应满足：

- [ ] Ollama 容器 healthy
- [ ] Hermes 容器 running
- [ ] Qwen 已下载
- [ ] Ollama API 能回答
- [ ] Hermes 能列出模型
- [ ] 上下文为 64K
- [ ] Hermes 能完成一次安全的文件读取任务
- [ ] 数据在容器重启后仍存在
- [ ] 8642 与 11434 没有暴露到公网

## 18. 卸载与恢复

只停止并删除容器，保留模型：

```bash
docker compose down
```

连 Ollama 模型卷一起删除（模型需要重新下载）：

```bash
docker compose down -v
```

`./hermes-data` 和 `.env` 是 bind mount 文件，不会被 `down -v` 删除。如需删除请先备份，并明确确认目录。

## 19. 官方资料

- Hermes Agent 本地 Ollama 指南：<https://hermes-agent.nousresearch.com/docs/guides/local-ollama-setup>
- Hermes Agent Docker 指南：<https://hermes-agent.nousresearch.com/docs/user-guide/docker/>
- Hermes Agent Provider 配置：<https://hermes-agent.nousresearch.com/docs/integrations/providers>
- Docker Engine Ubuntu 安装：<https://docs.docker.com/engine/install/ubuntu/>
- Docker Compose 安装：<https://docs.docker.com/compose/install/linux/>
- Ollama Docker：<https://docs.ollama.com/docker>
- Ollama 模型库：<https://ollama.com/library>

## 20. 最短执行路线

已经安装 Docker、硬件足够时：

```bash
cd /home/wsfoo/games/hermes-ollama-qwen
cp .env.example .env
# 编辑 .env，换成 openssl rand -hex 32 的结果

docker compose up -d ollama
docker compose exec ollama ollama pull qwen3.5:27b
docker compose up -d
docker compose logs -f hermes
```

另开一个终端：

```bash
cd /home/wsfoo/games/hermes-ollama-qwen
docker compose run --rm --entrypoint hermes hermes
```

到这里，Hermes Agent 就会通过 Docker 网络调用 Ollama 中的 Qwen。

---

# 进阶详细篇

以下章节从“能运行”继续深入到“能安全、稳定、可维护地长期使用”。第一次安装可以先完成第 1～13 章，再按需要阅读。

## 21. 新手名词表

| 名词 | 简单解释 | 在本项目中的例子 |
|---|---|---|
| Image | 只读的软件安装模板 | `ollama/ollama:latest` |
| Container | 由 Image 启动的运行实例 | `hermes-ollama` |
| Volume | Docker 管理的持久资料 | `ollama-models` |
| Bind mount | 把宿主机目录映射进容器 | `./workspace:/workspace` |
| Port | 服务对外接收连接的编号 | Ollama 11434、Hermes 8642 |
| Network | 容器之间通信的虚拟网络 | `hermes-net` |
| Environment variable | 启动时传给程序的设置 | `OLLAMA_CONTEXT_LENGTH` |
| API | 程序与程序通信的接口 | `/v1/chat/completions` |
| Token | 模型处理文字的基本单位 | 64K context 约 64,000 tokens |
| Context window | 一次请求能记住的最大 token 数 | Hermes 要求至少 64K |
| Parameter | 模型规模的一种指标 | 8B、27B、32B |
| Quantization | 降低模型精度换取更少内存 | Q4、Q5、Q8 |
| Tool calling | 模型以结构化方式请求执行工具 | 读取文件、运行命令 |
| Agent | 能规划并多步调用工具的系统 | Hermes Agent |

## 22. 模型内存如何估算

模型参数量不等于实际文件大小。粗略估算：

```text
FP16：参数量 × 2 bytes
INT8：参数量 × 1 byte
Q4：参数量 × 0.5 byte，再加运行开销
```

例如 27B Q4 权重大约需要 13.5 GB，实际还要加：

- KV cache（随上下文长度增加）
- Ollama 运行缓冲区
- Hermes、Docker 与操作系统内存
- GPU 驱动与显示占用

因此“模型文件 16 GB”不代表“16 GB RAM 一定能运行”。64K 上下文的 KV cache 可能额外使用数 GB到十几 GB，依模型结构而异。

选择顺序：

1. 先看可用 RAM，而不是总 RAM。
2. 保留至少 4～8 GB 给操作系统和 Docker。
3. GPU VRAM 不足时，Ollama 可能把部分层放到 CPU/RAM，速度会下降。
4. 先用小模型验证架构，再升级模型。
5. 用 `docker stats` 和 `ollama ps` 实测，不靠猜测。

压力测试：

```bash
docker stats hermes-ollama hermes-agent
docker compose exec ollama ollama ps
watch -n 1 free -h
watch -n 1 nvidia-smi  # NVIDIA
```

## 23. CPU-only 安装路线

CPU 用户只使用 `compose.yaml`：

```bash
docker compose up -d ollama
docker compose exec ollama ollama pull qwen3:8b
docker compose exec ollama ollama run qwen3:8b "你好"
```

然后修改：

```yaml
# hermes-data/config.yaml
model:
  default: qwen3:8b
  provider: custom
  base_url: http://ollama:11434/v1
  context_length: 64000
```

注意：小模型可能不支持有效 64K 工具工作流。这个路线主要用于验证网络、Docker、API 与 Hermes 配置是否正确。复杂任务应换更强模型或硬件。

CPU 优化建议：

- 关闭不必要的虚拟机与浏览器标签页。
- 不要同时加载多个大模型。
- 将 `OLLAMA_KEEP_ALIVE` 设为 `10m` 可较快释放内存。
- 第一轮 prefill 很慢属于正常现象。
- 避免让系统开始大量 swap；swap 会让推理极慢。

## 24. NVIDIA GPU 完整检查链

按层检查，不要一开始就怪 Ollama：

```text
宿主机驱动 → Docker GPU → Ollama 容器 → 模型 GPU offload
```

第一层：

```bash
nvidia-smi
```

第二层：

```bash
docker run --rm --gpus all ubuntu nvidia-smi
```

第三层：

```bash
docker compose -f compose.yaml -f compose.nvidia.yaml up -d ollama
docker compose exec ollama ollama run qwen3.5:27b "回答 OK"
```

第四层：

```bash
docker compose exec ollama ollama ps
nvidia-smi
```

如果 `ollama ps` 显示全部 CPU：

- 检查是否真的使用 `compose.nvidia.yaml`。
- 检查 NVIDIA Container Toolkit。
- 检查虚拟机是否配置 PCIe GPU passthrough。
- 重新启动 Docker 与 Ollama。

## 25. 虚拟机、WSL2 与 Docker Desktop 差异

### Ubuntu 虚拟机

- CPU 模式通常直接可用。
- GPU 需要 hypervisor 与硬件支持 passthrough。
- 虚拟机磁盘必须留足模型空间。
- 桥接/NAT 只影响外部访问，不影响两个 Compose 服务互联。

### Windows + WSL2

- 建议使用 Docker Desktop 的 WSL2 backend。
- NVIDIA GPU 需要 Windows 驱动支持 WSL CUDA。
- Windows 主机与 WSL/容器的 `localhost` 行为要依实际服务位置判断。
- 如果 Hermes 与 Ollama都在同一 Compose，不需要处理 host 地址，仍使用 `http://ollama:11434/v1`。

### macOS

- Docker Desktop 中的 Ollama 容器通常不能使用 Apple Metal GPU 加速。
- 若追求性能，可让 Ollama 原生运行在 macOS，Hermes 容器使用 `host.docker.internal:11434`。
- 本教程的“双容器方案”在 macOS 更适合功能验证，而不是最高性能。

## 26. `compose.yaml` 逐段解释

### Ollama 服务

```yaml
ollama:
  image: ollama/ollama:latest
  restart: unless-stopped
```

`restart` 表示机器重启后自动恢复，除非你明确停止过容器。

```yaml
volumes:
  - ollama-models:/root/.ollama
```

模型储存在 named volume，删除容器不会删除模型。

```yaml
ports:
  - "127.0.0.1:11434:11434"
```

从左到右：宿主机地址、宿主机端口、容器端口。绑定回环地址可避免局域网其他设备直接调用 Ollama。

### Hermes 服务

```yaml
volumes:
  - ./hermes-data:/opt/data
  - ./workspace:/workspace
working_dir: /workspace
```

Hermes 配置与会话在 `/opt/data`；实际允许操作的练习文件放 `/workspace`。这是权限边界，不建议挂载整个主目录。

```yaml
depends_on:
  ollama:
    condition: service_healthy
```

Hermes 会等待 Ollama healthcheck 成功，但这不保证模型已经下载。

## 27. 第一次启动：逐条验收版

不要一次执行所有命令。每一步成功后再继续。

### A. Docker

```bash
docker version
docker compose version
```

### B. 配置密钥

```bash
cd /home/wsfoo/games/hermes-ollama-qwen
cp .env.example .env
SECRET_VALUE=$(openssl rand -hex 32)
printf 'API_SERVER_KEY=%s\n' "$SECRET_VALUE" > .env
```

### C. 检查 Compose 展开结果

```bash
docker compose config > /tmp/hermes-compose-expanded.yaml
sed -n '1,240p' /tmp/hermes-compose-expanded.yaml
```

不要把展开后的文件公开，因为其中包含密钥。

### D. 启动 Ollama

```bash
docker compose up -d ollama
docker compose ps
curl --fail http://127.0.0.1:11434/api/tags
```

### E. 下载模型

```bash
docker compose exec ollama ollama pull qwen3.5:27b
docker compose exec ollama ollama list
```

### F. 测试模型

```bash
docker compose exec ollama ollama run qwen3.5:27b "只回答：模型正常"
```

### G. 启动 Hermes

```bash
docker compose up -d hermes
docker compose ps
docker compose logs --tail=200 hermes
```

### H. 测试容器间网络

```bash
docker compose exec hermes curl --fail http://ollama:11434/v1/models
```

### I. 启动 CLI

```bash
docker compose run --rm --entrypoint hermes hermes
```

第一次任务建议只读：

```text
请列出 /workspace 中的文件，不要修改任何内容。
```

## 28. 工作目录与权限控制

本教程把：

```text
宿主机 ./workspace
```

映射为：

```text
Hermes 容器 /workspace
```

放一个测试项目：

```bash
mkdir -p workspace/demo
printf 'print("hello")\n' > workspace/demo/main.py
```

让 Hermes：

```text
读取 /workspace/demo/main.py，解释代码，不要修改。
```

确认读取正常后，再尝试：

```text
在 /workspace/demo 建立 README.md，写明启动方法。执行前告诉我会修改哪些文件。
```

安全原则：

- 初次使用只挂载专用 workspace。
- 不挂载 `/`、`/etc`、`~/.ssh`、浏览器资料或密码库。
- 不挂载 `/var/run/docker.sock`；那几乎等于把宿主机 root 权限交给 Agent。
- 重要项目先提交 Git，再允许 Agent 修改。
- 仔细阅读 Hermes 的危险命令确认提示。

只读挂载示例：

```yaml
volumes:
  - ./reference-docs:/reference:ro
```

## 29. 用 Python 调用 Ollama API

建立 `workspace/test_ollama.py`：

```python
import json
import urllib.request

payload = {
    "model": "qwen3.5:27b",
    "messages": [
        {"role": "system", "content": "你是简洁的中文助理。"},
        {"role": "user", "content": "解释什么是容器。"},
    ],
    "stream": False,
}

request = urllib.request.Request(
    "http://127.0.0.1:11434/v1/chat/completions",
    data=json.dumps(payload).encode("utf-8"),
    headers={"Content-Type": "application/json"},
)

with urllib.request.urlopen(request, timeout=1800) as response:
    data = json.load(response)
    print(data["choices"][0]["message"]["content"])
```

运行：

```bash
python3 workspace/test_ollama.py
```

这个请求直接调用 Ollama，不经过 Hermes；适合区分“模型问题”与“Agent 问题”。

## 30. Open WebUI 网页界面（可选）

已经提供 `compose.webui.yaml`。启动：

```bash
docker compose -f compose.yaml -f compose.webui.yaml up -d
```

打开：

```text
http://127.0.0.1:3000
```

第一次进入建立管理员账号。Compose 会把 Open WebUI 的 OpenAI 连接指向：

```text
http://hermes:8642/v1
```

并使用 `.env` 的 `API_SERVER_KEY`。

检查：

```bash
docker compose -f compose.yaml -f compose.webui.yaml ps
docker compose -f compose.yaml -f compose.webui.yaml logs --tail=100 open-webui
```

停止 WebUI但保留其他服务：

```bash
docker compose -f compose.yaml -f compose.webui.yaml stop open-webui
```

Open WebUI 镜像更新较快，生产环境建议把 `main` 改成经过测试的固定版本 tag。

## 31. 局域网访问与反向代理

默认只允许本机访问，这是安全选择。如果要让同一 Wi-Fi 的另一台电脑访问 WebUI，可以把：

```yaml
127.0.0.1:3000:8080
```

改成：

```yaml
0.0.0.0:3000:8080
```

然后访问：

```text
http://服务器局域网IP:3000
```

必须同时做到：

- 使用强管理员密码。
- 防火墙只允许可信网段。
- 不把 3000 端口转发到公网。
- 不直接暴露 Ollama 11434。
- 不直接暴露无 TLS 的 Hermes 8642 到公网。

UFW 范例（把网段换成你的真实网段）：

```bash
sudo ufw allow from 192.168.30.0/24 to any port 3000 proto tcp
sudo ufw status numbered
```

注意 Docker 发布端口可能绕过部分 UFW 规则；高安全环境应使用 `DOCKER-USER` 链或反向代理，并由管理员审核网络设计。

## 32. Hermes 记忆、会话与配置

`./hermes-data` 是 Hermes 的持久资料根目录，可能包含：

- `config.yaml`：普通配置
- `.env`：provider 密钥或平台 token
- 会话历史
- 记忆
- Skills
- 平台连接资料

检查权限：

```bash
ls -la hermes-data
find hermes-data -maxdepth 2 -type f -printf '%M %u:%g %p\n' | sed -n '1,120p'
```

不要直接编辑正在写入的会话文件。修改 `config.yaml` 后：

```bash
docker compose restart hermes
docker compose logs --tail=100 hermes
```

## 33. 性能调优

### Keep alive

```yaml
OLLAMA_KEEP_ALIVE: "24h"
```

适合持续使用，但会长期占用 RAM/VRAM。偶尔使用可改为：

```yaml
OLLAMA_KEEP_ALIVE: "10m"
```

### 并发

不要一开始同时让多个 Hermes 会话调用同一个大模型。先确认单会话稳定，再增加并发。内存不足时并发会导致模型反复加载或容器被杀。

### 上下文

64K 是 Hermes 当前最低要求，但长上下文有显著成本：

- prefill 更慢
- KV cache 更大
- 每轮可能处理更多历史

使用 `/compress` 和 `/new` 控制会话长度，不要让无关任务长期混在同一对话。

### 检查实际瓶颈

```bash
docker stats
nvidia-smi dmon  # NVIDIA，可用时
vmstat 1
iostat -xz 1     # 需 sysstat
```

判断：

- CPU 100%、GPU 空闲：CPU 推理或 GPU 未配置。
- RAM 接近满、swap 快速增加：模型/上下文太大。
- GPU VRAM 满但利用率低：可能在等待 CPU 或磁盘。
- 磁盘持续高负载：模型反复加载或系统 swap。

## 34. 更新、固定版本与回滚

`latest` 方便学习，但生产环境有不可预测更新。查看当前镜像：

```bash
docker image inspect nousresearch/hermes-agent:latest \
  --format '{{index .RepoDigests 0}}'
docker image inspect ollama/ollama:latest \
  --format '{{index .RepoDigests 0}}'
```

更新前备份：

```bash
tar -czf hermes-before-update-$(date +%F-%H%M).tar.gz hermes-data .env
```

更新：

```bash
docker compose pull
docker compose up -d
docker compose logs --tail=200
```

如果新版本有问题，可以把 Compose 镜像改回已知 tag 或 digest，再执行：

```bash
docker compose pull
docker compose up -d
```

不要在没有备份时升级重要实例。

## 35. 备份与恢复演练

### Hermes bind mount

```bash
tar -czf hermes-data-$(date +%F).tar.gz hermes-data .env
```

恢复前停止 Hermes：

```bash
docker compose stop hermes
```

先把现有目录改名保留，而不是直接覆盖：

```bash
mv hermes-data hermes-data.before-restore
mkdir hermes-data
# 然后从备份解压
```

### Ollama 模型

通常不必备份模型，可以依据名称重新 pull。网络昂贵或离线环境才考虑备份 Docker volume。

查看卷真实位置：

```bash
docker volume inspect hermes-qwen_ollama-models
```

不要在 Ollama 正在写入模型时复制卷内容。

## 36. 日志与健康监控

常用命令：

```bash
docker compose ps
docker compose top
docker compose logs --since=10m hermes
docker compose logs --since=10m ollama
docker inspect hermes-ollama --format '{{json .State.Health}}'
```

简单定期检查：

```bash
curl --fail --max-time 5 http://127.0.0.1:11434/api/tags >/dev/null \
  && echo 'Ollama OK' || echo 'Ollama FAIL'
```

日志持续变大时可在 Compose 为服务加入：

```yaml
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

## 37. 故障判断树

```text
Docker 命令失败？
├─ 是 → 检查 docker.service、权限、磁盘
└─ 否
   Ollama /api/tags 失败？
   ├─ 是 → 检查 ollama 日志、端口、健康状态
   └─ 否
      Ollama 直接问答失败？
      ├─ 是 → 检查模型名称、RAM/VRAM、上下文
      └─ 否
         Hermes 容器访问 /v1/models 失败？
         ├─ 是 → 检查 hermes-net 与 base_url
         └─ 否
            Hermes 聊天失败？
            ├─ 上下文错误 → 检查实际 64K
            ├─ 工具格式错误 → 换支持 tools 的模型
            ├─ timeout → 增加 timeout 或换小模型/GPU
            └─ 权限错误 → 检查 workspace 与文件权限
```

收集诊断资料：

```bash
docker compose ps
docker compose logs --tail=300 hermes ollama
docker compose exec ollama ollama list
docker compose exec ollama ollama ps
free -h
df -h
docker system df
```

分享日志前删除 API key、token、用户名、私有路径和对话内容。

## 38. 常见问答

### 一定要有 GPU 吗？

不一定。CPU 可以运行，但大模型和 64K 上下文会明显较慢。

### Hermes、Ollama、Qwen 可以放同一个容器吗？

技术上可以，但不建议。分开容器更容易升级、观察和更换模型服务器。

### 为什么不用 `localhost:11434`？

因为 Hermes 在另一个容器中；容器内 localhost 指 Hermes 自己。Compose 服务名 `ollama` 才是正确地址。

### 模型下载一次后重启会消失吗？

不会，模型在 Docker volume 中。只有 `docker compose down -v` 或手动删卷才会消失。

### 可以断网使用吗？

镜像和模型下载完成后，本地推理可以离线；网页搜索等外部工具仍需要网络。

### 可以让 Hermes 操作我的项目吗？

把项目复制到 `workspace`，或只挂载特定项目。不要直接挂载整个主目录。

### 可以公开到互联网吗？

可以，但 Hermes 能执行工具，风险高。需要 TLS、认证、最小权限、网络限制、审计和持续更新；不建议初学者直接公开。

### 为什么模型会回答，但不会执行工具？

聊天能力和可靠 tool calling 是两件事。使用明确标注 tools 的较强模型，并确认服务端工具格式兼容。

## 39. 命令速查表

```bash
# 状态
docker compose ps
docker compose logs -f
docker stats

# 启停
docker compose up -d
docker compose stop
docker compose start
docker compose restart hermes
docker compose down

# 模型
docker compose exec ollama ollama list
docker compose exec ollama ollama pull qwen3.5:27b
docker compose exec ollama ollama run qwen3.5:27b
docker compose exec ollama ollama ps

# API
curl http://127.0.0.1:11434/api/tags
docker compose exec hermes curl http://ollama:11434/v1/models

# 更新
docker compose pull
docker compose up -d

# 磁盘
docker system df
docker volume ls

# CLI
docker compose run --rm --entrypoint hermes hermes
```

## 40. 从零重建检查表

不要直接删除既有资料。先确认备份，再在测试目录演练：

1. 安装 Docker 与 Compose。
2. 验证 `hello-world`。
3. 建立 `.env` 随机密钥。
4. `docker compose config` 检查配置。
5. 只启动 Ollama。
6. 下载并直接测试 Qwen。
7. 确认实际上下文。
8. 设置 Hermes `base_url=http://ollama:11434/v1`。
9. 启动 Hermes。
10. 从 Hermes 容器测试 `/v1/models`。
11. 用 CLI 执行只读任务。
12. 在专用 workspace 测试文件修改。
13. 备份 hermes-data 与 `.env`。
14. 需要时再启用 Open WebUI。
15. 最后才考虑局域网或公网开放。

执行到这里，你不仅“装好了”，也知道每一层如何验证、出错时从哪里开始查。
