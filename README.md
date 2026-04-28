# Windows 本地部署 Dify 工作流 Demo 指南

> 本指南记录了在 Windows 10/11 物理机上从零开始部署 Dify，并搭建项目管理 AI 助手 Demo 的完整流程。

---

## 目录

- [环境要求](#环境要求)
- [快速开始](#快速开始)
- [详细步骤](#详细步骤)
  - [第一阶段：启用 Hyper-V](#第一阶段启用-hyper-v)
  - [第二阶段：安装 WSL2](#第二阶段安装-wsl2)
  - [第三阶段：安装 Docker Desktop](#第三阶段安装-docker-desktop)
  - [第四阶段：部署 Dify](#第四阶段部署-dify)
  - [第五阶段：接入 LLM 模型](#第五阶段接入-llm-模型)
  - [第六阶段：创建第一个应用](#第六阶段创建第一个应用)
- [常见问题](#常见问题)
- [注意事项](#注意事项)

---

## 环境要求

| 项目 | 要求 |
|------|------|
| 操作系统 | Windows 10/11 Pro 或 Enterprise |
| C 盘可用空间 | 建议 ≥ 50GB，否则需迁移 Docker 数据到其他盘 |
| 内存 | 建议 ≥ 16GB |
| 网络 | 需要访问 Docker Hub 和 GitHub（建议全局代理） |

---

## 快速开始

```powershell
# 1. 启用 Hyper-V
dism.exe /online /enable-feature /featurename:Microsoft-Hyper-V-All /all /norestart
# 重启电脑

# 2. 验证环境
wsl echo "hello"

# 3. 启动 Dify（安装 Docker Desktop 后）
cd <dify解压目录>\docker
copy .env.example .env
docker compose up -d

# 4. 打开浏览器
start http://localhost
```

---

## 详细步骤

### 第一阶段：启用 Hyper-V

以**管理员身份**打开 PowerShell，执行：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Hyper-V-All /all /norestart
```

执行成功后**重启电脑**，然后用以下命令验证：

```powershell
wsl echo "hello"
Get-Service vmcompute  # Status 应为 Running
```

> ⚠️ **注意**：不要用 `Start-Service vmcompute` 来验证 Hyper-V，vmcompute 是按需启动的服务，没有工作负载时会超时报错（错误 1053），这是正常现象。

### 第二阶段：安装 WSL2

```powershell
# 全新安装
wsl --install

# 已有 WSL1 则升级
wsl --set-version Ubuntu 2

# 验证（VERSION 列应显示 2）
wsl -l -v
```

### 第三阶段：安装 Docker Desktop

1. 从 [Docker 官网](https://www.docker.com/products/docker-desktop/) 下载安装包
2. 安装时勾选 **"Use WSL 2 based engine"**
3. 安装完成后启动 Docker Desktop，等待托盘图标静止

**C 盘空间不足时，将数据迁移到 D 盘：**

完全退出 Docker Desktop 后执行：

```powershell
mkdir D:\Docker\distro

wsl --export docker-desktop D:\Docker\docker-desktop.tar
wsl --unregister docker-desktop
wsl --import docker-desktop D:\Docker\distro\docker-desktop D:\Docker\docker-desktop.tar --version 2
```

然后在 Docker Desktop 中：**Settings → Resources → Advanced → Disk image location** 改为 `D:\Docker\images`，点 Apply & Restart。

### 第四阶段：部署 Dify

**下载源码**：从 [Dify GitHub](https://github.com/langgenius/dify/releases) 下载最新版本并解压。

```powershell
# 进入 docker 子目录（注意不是根目录）
cd <dify解压目录>\docker

# 生成配置文件
copy .env.example .env

# 启动（首次约需 5-15 分钟拉取镜像）
docker compose up -d

# 验证（所有容器应为 Running 或 Healthy）
docker compose ps
```

启动成功后访问 **http://localhost**，完成管理员账号注册。

**常用管理命令：**

```powershell
docker compose up -d    # 启动
docker compose down     # 停止
docker compose logs -f  # 查看日志
docker compose ps       # 查看状态
```

> ⚠️ 电脑重启后需手动执行 `docker compose up -d` 重新启动 Dify。

### 第五阶段：接入 LLM 模型

在 Dify 中：**右上角头像 → 设置 → 模型供应商**，选择以下任一方案：

**方案一：SiliconFlow（推荐新手）**
- 注册：https://cloud.siliconflow.cn（新用户赠约 14 元代金券）
- 获取 API Key 后，在模型供应商中搜索"硅基流动"安装并填入 Key
- 默认推理模型选 `deepseek-ai/DeepSeek-V3`

**方案二：DeepSeek 官方**
- 注册：https://platform.deepseek.com
- 注意新账号可能无免费额度，需充值后使用

**方案三：Ollama（完全本地，完全免费）**
- 确认本地已安装 Ollama 并有可用模型：`ollama list`
- Dify 中配置地址为 `http://host.docker.internal:11434`（不能用 localhost）

### 第六阶段：创建第一个应用

1. 左侧菜单 → **工作室** → **创建应用** → 选择 **Chatflow**
2. 填写应用名称，点击创建
3. 在编排页面填写系统提示词
4. 点击右上角**发布** → **运行**，开始对话测试

---

## 常见问题

**Q：vmcompute 服务启动失败，报错 1053**  
A：先依次启动依赖服务：`Start-Service vmms`，再启 `Start-Service vmcompute`。或者直接用 `wsl echo "hello"` 验证，只要 WSL 正常，Hyper-V 就没问题。

**Q：系统文件损坏导致 Hyper-V 异常**  
A：以管理员运行 `sfc /scannow` 修复，修复后重试。

**Q：docker compose up -d 报错无法连接 Docker**  
A：检查 Docker Desktop 是否已启动，等待托盘图标静止后再执行。

**Q：文件上传被禁用**  
A：在应用编排页面开启文件上传功能，PRD 文档可直接以文字粘贴方式输入，效果相同。

**Q：Ollama 在 Docker 中无法访问**  
A：Docker 容器内访问宿主机服务必须使用 `host.docker.internal`，不能用 `localhost`。

---

## 注意事项

- **API Key 安全**：永远不要将 API Key 粘贴到聊天窗口，只在 Dify 配置框中填写
- **数据隐私**：对话记录和知识库存储在本地，只有发送给 LLM 的内容会经过云端服务器
- **C 盘空间**：12 个 Docker 容器镜像占用空间较大，建议提前迁移到其他盘
- **Windows 版本**：Home 版 Hyper-V 支持有限，推荐使用 Pro 或 Enterprise 版本

---

## 相关链接

- [Dify 官网](https://dify.ai)
- [Dify GitHub](https://github.com/langgenius/dify)
- [Docker Desktop 下载](https://www.docker.com/products/docker-desktop/)
- [SiliconFlow 注册](https://cloud.siliconflow.cn)
- [Ollama 官网](https://ollama.com)
