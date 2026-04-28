在 Ubuntu 下安装 OpenClaw，优先用**一键脚本**（新手友好），也可手动用 npm 或 Docker 安装。

### 一、系统准备（必做）
```bash
# 更新系统包
sudo apt update && sudo apt upgrade -y

# 安装基础工具
sudo apt install -y curl git
```

### 二、一键安装（官方推荐，最快）
```bash
# 官方脚本
curl -fsSL https://openclaw.ai/install.sh | bash

# 国内镜像（网络慢时用）
# curl -fsSL https://open-claw.org.cn/install-cn.sh | bash
```
- 脚本会自动安装 Node.js 22+ 并引导配置。
- 验证：`openclaw --version` 显示版本即成功。

### 三、npm 手动安装（需已装 Node.js 22+）
```bash
# 安装 Node.js 22（如未装）
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# 全局安装 OpenClaw
sudo npm install -g openclaw@latest

# 验证
openclaw --version
```

### 四、Docker 安装（环境隔离）
```bash
# 安装 Docker（如未装）
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker

# 拉取并运行
docker run -d -p 18789:18789 openclaw/openclaw
```

### 五、初始化与启动
```bash
# 配置向导（模型、API Key 等）
openclaw onboard

# 启动服务
openclaw gateway start

# 打开 Web UI（默认 http://127.0.0.1:18789）
openclaw dashboard
```

### 常见问题
- 权限不足：命令前加 `sudo`。
- 下载慢：换国内镜像脚本或 npm 源。
- 依赖缺失：`openclaw doctor` 自动检测修复。

需要我帮你写一个 Ubuntu 下一键安装+初始化的完整脚本，你直接复制运行即可吗？