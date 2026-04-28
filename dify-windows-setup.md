---
name: dify-windows-setup
description: 在 Windows 系统上从零开始搭建 Dify 工作流 demo 的完整指南。当用户提到想在 Windows 上部署 Dify、搭建本地 AI 工作流、用 Docker 跑 Dify、或者想做项目管理 AI 助手 demo 时，使用此 Skill。涵盖环境准备、Hyper-V 配置、Docker 安装、Dify 部署、模型接入到创建第一个应用的全流程。
---

# Windows 系统从零搭建 Dify 工作流 Demo

## 概览

本 Skill 记录了在 Windows 10/11 物理机上完整部署 Dify 的全流程，包含所有常见问题的排查方法。

**总耗时**：约 30-60 分钟（含镜像下载时间）  
**前提条件**：Windows 10/11 Pro/Enterprise（Home 版 Hyper-V 支持有限）

---

## 第一阶段：启用 Hyper-V

### 1.1 检查 Hyper-V 状态

以**管理员身份**打开 PowerShell：

```powershell
Get-Service vmcompute, vmms | Select Name, Status, StartType
```

### 1.2 启用 Hyper-V 功能

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Hyper-V-All /all /norestart
```

执行完成后**重启电脑**。

### 1.3 验证 Hyper-V 正常

重启后用 WSL 命令验证（比直接启动服务更可靠）：

```powershell
wsl echo "hello"
Get-Service vmcompute  # 应显示 Running
```

> **重要说明**：vmcompute 是按需启动的服务，直接用 `Start-Service vmcompute` 会因为没有工作负载而超时报错（错误 1053），这是正常现象。只要 WSL 能运行，Hyper-V 就工作正常。

### 1.4 常见问题排查

**问题：vmcompute 启动失败，错误 1053**
```powershell
# 先启动依赖服务
Start-Service vmms
Start-Service vmcompute
```

**问题：系统文件损坏导致服务异常**
```powershell
sfc /scannow
DISM /Online /Cleanup-Image /RestoreHealth
```

**问题：vmms 报错 0x8007000E（内存不足）**
- 关闭占用内存的进程后重试
- 检查是否有崩溃的第三方进程干扰（查看应用程序事件日志）

---

## 第二阶段：安装 WSL2

```powershell
# 安装 WSL2（如果尚未安装）
wsl --install

# 如果已有 WSL1，升级到 WSL2
wsl --set-version Ubuntu 2

# 验证
wsl -l -v  # VERSION 列应显示 2
```

---

## 第三阶段：安装 Docker Desktop

### 3.1 下载安装

从官网下载：https://www.docker.com/products/docker-desktop/

安装时勾选 **"Use WSL 2 based engine"**。

### 3.2 将 Docker 数据迁移到 D 盘（推荐，避免 C 盘空间不足）

> C 盘空间充足（>50GB 可用）可跳过此步骤。

**完全退出 Docker Desktop 后**，在 PowerShell 执行：

```powershell
# 查看当前 WSL 发行版
wsl -l -v

# 创建目标目录
mkdir D:\Docker\distro

# 迁移 docker-desktop（如果存在）
wsl --export docker-desktop D:\Docker\docker-desktop.tar
wsl --unregister docker-desktop
wsl --import docker-desktop D:\Docker\distro\docker-desktop D:\Docker\docker-desktop.tar --version 2

# 迁移 docker-desktop-data（如果存在）
wsl --export docker-desktop-data D:\Docker\docker-desktop-data.tar
wsl --unregister docker-desktop-data
wsl --import docker-desktop-data D:\Docker\distro\docker-desktop-data D:\Docker\docker-desktop-data.tar --version 2
```

然后在 Docker Desktop 中：**Settings → Resources → Advanced → Disk image location** 改为 `D:\Docker\images`，点 Apply & Restart。

### 3.3 验证 Docker 正常

```powershell
docker --version
docker compose version
docker info | findstr "Docker Root Dir"
```

---

## 第四阶段：部署 Dify

### 4.1 获取 Dify 源码

从 GitHub 下载最新版本：https://github.com/langgenius/dify/releases

解压到目标目录，例如 `D:\AI\dify\`。

### 4.2 配置环境变量

```powershell
cd D:\AI\dify\docker   # 注意：.env.example 在 docker 子目录，不在根目录
copy .env.example .env
```

默认配置即可运行 demo，无需修改。

### 4.3 启动 Dify

```powershell
docker compose up -d
```

第一次启动会拉取约 12 个容器镜像，需要 5-15 分钟。

**验证启动成功**：
```powershell
docker compose ps  # 所有容器应为 Running 或 Healthy 状态
```

正常输出示例：
```
docker-redis-1          Running
docker-web-1            Running
docker-db_postgres-1    Healthy
docker-api-1            Running
docker-worker-1         Running
docker-nginx-1          Running
docker-weaviate-1       Running
...（共 12 个容器）
```

### 4.4 访问 Dify

浏览器打开：**http://localhost**

首次访问设置管理员邮箱和密码。

### 4.5 常用管理命令

```powershell
cd D:\AI\dify\docker

# 停止
docker compose down

# 启动
docker compose up -d

# 查看日志
docker compose logs -f

# 查看状态
docker compose ps
```

---

## 第五阶段：接入 LLM 模型

### 方案一：SiliconFlow（推荐新手，有免费额度）

1. 注册：https://cloud.siliconflow.cn（注册赠约 14 元代金券）
2. 左侧菜单 → API 密钥 → 新建密钥
3. Dify 中：右上角头像 → 设置 → 模型供应商 → 搜索"硅基流动" → 安装 → 填入 API Key
4. 默认模型设置选 `deepseek-ai/DeepSeek-V3`

### 方案二：DeepSeek 官方

1. 注册：https://platform.deepseek.com
2. 获取 API Key（注意：新账号可能无免费额度，需充值）
3. Dify 中：模型供应商 → DeepSeek → 填入 API Key

### 方案三：Ollama（完全本地，完全免费）

```powershell
# 确认已安装并有模型
ollama list

# 如需下载模型
ollama pull qwen2.5:7b
```

Dify 中：模型供应商 → Ollama → 配置地址为 `http://host.docker.internal:11434`

> **注意**：Dify 运行在 Docker 容器内，访问宿主机 Ollama 必须用 `host.docker.internal` 而不是 `localhost`。

---

## 第六阶段：创建第一个应用（项目管理助手示例）

### 6.1 创建应用

左侧菜单 → 工作室 → 创建应用 → **Chatflow** → 填写名称和描述 → 创建

### 6.2 配置 System Prompt

在编排页面系统提示词输入框填入：

```
你是一位经验丰富的项目管理助手，专门协助项目经理处理日常项目管理工作。

你具备以下能力：

1. **PRD需求拆分**：当用户提供PRD文档或需求描述时，你能将其拆分为结构化的任务列表，每个任务包含：任务名称、任务描述、优先级（高/中/低）、预估工时、负责角色建议、验收标准。

2. **任务进度跟进**：当用户描述任务进展时，你能帮助整理进度状态，识别风险点，提出跟进建议。

3. **提醒话术生成**：当用户需要催进度或提醒相关人员时，你能生成专业、得体的提醒消息，语气礼貌但明确。

4. **项目日报编写**：根据用户提供的当日进展信息，生成规范的项目日报，包含：今日完成事项、明日计划、当前风险与阻塞。

回复要求：
- 输出结构清晰，使用表格或列表呈现任务信息
- 语言简洁专业，适合在工作场景直接使用
- 遇到信息不足时，主动追问关键信息
```

### 6.3 发布和测试

点右上角**发布** → **运行**，在对话框输入需求进行测试。

**测试示例**：
```
需求描述：开发一个预测性维护系统的告警管理模块。
主要功能：
1. 设备异常时自动触发告警，支持短信和企业微信通知
2. 告警可以手动确认和关闭，需要填写处理备注
3. 告警历史记录可查询，支持按时间、设备、告警级别筛选

请帮我拆分成开发任务。
```

---

## 关键注意事项

| 事项 | 说明 |
|------|------|
| C 盘空间 | Docker 镜像较大，建议提前将数据目录迁移到 D 盘 |
| API Key 安全 | 永远不要将 API Key 粘贴到聊天窗口，只在配置框中填写 |
| vmcompute 服务 | 按需启动，不要用 Start-Service 测试，用 wsl 命令验证 |
| Ollama 地址 | Docker 内访问宿主机用 host.docker.internal，不用 localhost |
| 数据隐私 | 对话记录本地存储，只有发给 LLM 的内容会经过云端 |
| 重启后启动 | 重启电脑后需手动运行 docker compose up -d 启动 Dify |
