# CentOS 7 部署 OpenClaw 完整指南
## 一、部署方案对比
|方案|适用场景|优势|劣势|
| ---- | ---- | ---- | ---- |
|Docker 官方镜像|追求快速部署、环境隔离、稳定运行|绕过所有系统依赖限制，一键启动|需要安装 Docker 环境|
|Node.js 原生安装|需要深度定制、开发调试|完全控制源码，可二次开发|依赖复杂，需手动配置 Node.js 22、GCC 9 等|

## 二、方案一：Docker 官方镜像部署（强烈推荐）
此方式为 CentOS 7 上最快速、最稳定的部署方式，可完全绕过 Node.js 版本和系统依赖的限制。
### （一）前置准备
1. **安装 Docker**
```bash
# 一键安装 Docker
curl -fsSL https://get.docker.com | bash
# 启动 Docker 并设置开机自启
systemctl start docker
systemctl enable docker
# 验证安装
docker --version
```
2. **放行防火墙端口**
```bash
# 如果使用 firewalld
firewall-cmd --permanent --add-port=18789/tcp
firewall-cmd --reload
# 或者直接关闭防火墙（测试环境）
systemctl stop firewalld
systemctl disable firewalld
```

### （二）部署步骤
1. **创建持久化目录**
```bash
# 创建配置目录
mkdir -p ~/openclaw
# 修改目录所有者为容器内 node 用户（UID 1000）
sudo chown -R 1000:1000 ~/openclaw
```
2. **拉取并启动容器**
```bash
# 拉取官方镜像
docker pull ghcr.io/openclaw/openclaw:latest
# 启动容器（端口映射为 8700:18789，规避默认端口攻击）
docker run -d \
  --name openclaw \
  --restart unless-stopped \
  -p 8700:18789 \
  -v ~/openclaw:/home/node/.openclaw \
  ghcr.io/openclaw/openclaw:latest
```
3. **执行初始化配置**
```bash
# 进入容器执行初始化向导
docker exec -it openclaw openclaw onboard
```
**关键配置选择**：
- 安全警告：选择 Yes（你已知晓风险）
- 安装模式：选择 QuickStart（快速开始）
- 模型选择：选择拥有的 API Key 对应的模型（如阿里云百炼、智谱等），或 Skip for now 稍后配置
- 消息渠道：建议先 Skip for now，后续可配置飞书、钉钉等
- Skills 安装：可选择 Yes 安装常用技能

4. **获取访问令牌**
```bash
# 获取登录 Token
grep token ~/openclaw/openclaw.json
```
5. **配置远程访问（关键）**
```bash
# 编辑配置文件，添加绑定到 LAN 的配置
nano ~/openclaw/openclaw.json
```
在 gateway 字段下添加：
```json
{
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "lan",
    "controlUi": {
      "allowedOrigins": ["*"]
    }
  }
}
```
保存后重启容器：
```bash
docker restart openclaw
```
6. **访问 Web 界面**
浏览器访问：
```plaintext
http://你的服务器IP:8700/#token=你获取的Token
```

### （三）常用管理命令
```bash
# 查看实时日志
docker logs -f openclaw
# 停止/启动容器
docker stop openclaw
docker start openclaw
# 进入容器
docker exec -it openclaw bash
# 更新镜像
docker pull ghcr.io/openclaw/openclaw:latest && docker restart openclaw
# 列出待批准设备（如飞书）
docker exec -it openclaw openclaw devices list
# 批准设备
docker exec -it openclaw openclaw devices approve 设备ID
```

## 三、方案二：Node.js 原生安装（需手动配置依赖）
适合需要深度定制或开发调试的场景。
### （一）前置依赖安装
1. **升级 GCC 到 9.x 版本**（CentOS 7 默认 GCC 4.8 太低）
```bash
# 配置 SCL 源
cat > /etc/yum.repos.d/CentOS-SCLo-scl.repo << EOF
[centos-sclo-sclo]
name=CentOS-7 - SCLo sclo
baseurl=http://vault.centos.org/centos/7/sclo/x86_64/sclo/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo
enabled=1
EOF

cat > /etc/yum.repos.d/CentOS-SCLo-rh.repo << EOF
[centos-sclo-rh]
name=CentOS-7 - SCLo rh
baseurl=http://vault.centos.org/centos/7/sclo/x86_64/rh/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo
enabled=1
EOF

# 导入 GPG 密钥并刷新缓存
curl -o /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo https://vault.centos.org/centos/7/os/x86_64/RPM-GPG-KEY-CentOS-7
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo
yum clean all && yum makecache

# 安装 GCC 9 及依赖
yum install -y devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils scl-utils

# 启用 GCC 9（临时）
scl enable devtoolset-9 bash

# 永久启用（所有会话生效）
echo "source /opt/rh/devtoolset-9/enable" >> /etc/profile
source /etc/profile

# 验证
gcc --version  # 应显示 GCC 9.x.x
```
2. **安装 Node.js 22**（适配 glibc 2.17）
```bash
# 下载适配 CentOS 7 的 Node.js 包
mkdir -p /usr/local/src/nodejs && cd /usr/local/src/nodejs
wget https://unofficial-builds.nodejs.org/download/release/v22.16.0/node-v22.16.0-linux-x64-glibc-217.tar.gz

# 解压并配置环境变量
tar -zxf node-v22.16.0-linux-x64-glibc-217.tar.gz -C /opt/local/
ln -s /opt/local/node-v22.16.0-linux-x64-glibc-217 /usr/local/nodejs

# 永久配置环境变量
echo 'export PATH=/usr/local/nodejs/bin:$PATH' >> /etc/profile
source /etc/profile

# 验证
node -v  # 应显示 v22.16.0
npm -v
```
3. **配置 npm 镜像源**
```bash
npm config set registry https://registry.npmmirror.com
```

### （二）安装 OpenClaw
```bash
# 执行官方安装脚本
curl -fsSL https://openclaw.ai/install.sh | bash
# 或使用 npm 全局安装
npm install -g openclaw@latest
# 验证安装
openclaw --version
```

### （三）初始化配置
```bash
# 执行初始化向导
openclaw onboard --install-daemon
```
按照提示完成配置（与 Docker 方案相同）。

### （四）启动服务
```bash
# 启动网关服务
openclaw gateway start
# 查看状态
openclaw status
# 访问 Web 界面（默认端口 18789）
openclaw dashboard
```

## 四、配置国内大模型（推荐）
OpenClaw 默认使用海外模型，国内网络访问不稳定，推荐配置国内大模型。
### 阿里云百炼配置
```bash
# 进入容器（Docker 方案）
docker exec -it openclaw bash
# 或直接在终端执行（原生安装）
# 设置百炼 API Key
openclaw config set model.provider aliyun_bailian
openclaw config set model.aliyun_bailian.api_key "你的百炼API Key"
# 重启服务
openclaw restart
```
### 其他国内模型
支持智谱 GLM、通义千问、DeepSeek 等国内模型，根据拥有的 API Key 在配置向导中选择或手动配置。

## 五、常见问题排查
|问题|解决方案|
| ---- | ---- |
|无法访问 Web 界面|检查防火墙端口是否放行，确认容器/服务是否正常运行|
|Token 认证失败|重新获取 Token：docker exec -it openclaw openclaw token generate|
|模型调用失败|验证 API Key 是否正确，网络是否能访问模型接口|
|容器启动失败|检查日志 docker logs openclaw，确认是否有资源不足问题|
|内存不足|建议服务器配置至少 2 核 4GB，或创建 Swap 分区|

## 六、OpenClaw 核心介绍
### （一）核心定位
不是聊天机器人，是**执行代理**，区别于传统 AI（ChatGPT、Claude）的核心能力如下：
|特性|传统 AI（ChatGPT）|OpenClaw|
| ---- | ---- | ---- |
|能力|只能回答问题|能直接执行操作|
|交互方式|在聊天框内对话|通过微信/钉钉/浏览器等多渠道|
|执行任务|手动复制结果|自动完成整个流程|
|系统集成|无|深度集成浏览器、文件系统、命令行|
|记忆能力|有限对话上下文|长期记忆，记住偏好和历史|
|自动化|需要手动操作|7x24 小时自动运行|
|技能扩展|固定能力|技能市场，可安装扩展包|

### （二）实际应用场景
1. **办公自动化**：文件批量处理、报表数据处理、邮件与日程管理
2. **网页自动化**：信息采集、自动化操作、浏览器自动化
3. **系统运维**：服务器监控、自动化运维、故障排查
4. **日常助手**：信息查询、任务提醒、跨平台操作
5. **开发辅助**：代码生成、项目管理、自动化测试

### （三）技术架构优势
1. **Gateway 网关架构**：轻量级网关驻留系统，实时感知状态，直接调用内核指令；
2. **Skills 技能系统**：预置大量技能包，支持社区扩展和自定义开发；
3. **多渠道接入**：支持20+聊天平台，原生API接入+Webhook回调；
4. **记忆系统**：长期记忆偏好，保存历史上下文，支持会话管理；
5. **定时任务**：支持Cron表达式，定时执行并主动推送结果。

### （四）与传统自动化工具对比
|对比项|RPA 工具|Shell 脚本|OpenClaw|
| ---- | ---- | ---- | ---- |
|学习成本|需要专业培训|需要编程能力|自然语言即可|
|灵活性|固定流程|需要改代码|一句话改变需求|
|异常处理|固定逻辑|需要写判断|AI 自动推理处理|
|维护成本|需专业维护|需要调试代码|自然语言优化|
|适用人群|专业人员|程序员|普通用户|

### （五）核心价值：降本增效
1. 节省时间：压缩手动工作时长，7x24小时自动运行，批量处理重复任务；
2. 降低成本：无需专业编程技能、昂贵RPA工具和额外人力；
3. 提升效率：一次配置长期使用，自动化复杂流程，减少人为错误；
4. 解放人力：让人从重复性工作中脱离，专注创造性工作。

### （六）适合人群
办公人员、运营人员、运维人员、开发人员、创业者、有繁琐电脑任务的普通用户。

### （七）与扣子编程的关系
OpenClaw是AI自动化入门优选；若需要更强大的开发能力、可视化编排工作流、一键部署生产环境、更丰富集成生态，推荐前往**扣子编程（https://code.coze.cn/）**。

## 七、面试回答完整性技巧
核心原则：**完整性 = 结构 + 细节 + 验证**
### （一）结构化框架：确保信息层次完整
1. **STAR 法则**（行为面试必用）
    - 背景（Situation）：1-2句简短说明任务背景；
    - 任务（Task）：明确目标和责任；
    - 行动（Action）：详细描述具体行动，说明方法、困难及解决方式（核心）；
    - 结果（Result）：量化成果，强调个人贡献。
2. **问题-原因-解决方案**（技术/业务问题必用）
    - 问题：清晰定义，说明影响和重要性；
    - 原因：多角度分析根源（技术/流程/资源）；
    - 解决方案：给出2-3个可行方案并对比优劣；
    - 实施：详细说明执行步骤；
    - 验证：说明效果验证方式。

### （二）细节填充：确保信息密度完整
1. **量化数据（必填）**：所有结果需数据支撑，从时间、成本、效率、质量、业务等维度量化；
2. **技术细节（技术岗必填）**：说明技术选型原因、使用方式、遇到的问题及解决办法；
3. **业务背景（所有岗位必填）**：说明做事情的原因、解决的业务问题、带来的价值。

### （三）风险控制：确保逻辑闭环完整
1. 主动暴露弱点并说明应对策略；
2. 给出多方案并说明选择理由；
3. 明确方案的假设条件和适用约束。

### （四）完整性自检清单
回答后检查：结构完整性（开头/主体/结尾、逻辑层次）、内容完整性（背景/目标/行动/数据/成果）、技术完整性（技术岗）、业务完整性（所有岗）、风险控制完整性（局限性/替代方案/改进计划）。

### （五）高阶技巧：预判面试官的追问
提前在回答中埋伏面试官可能的追问点（如问题原因、方案选择、技术挑战、反思改进等），覆盖行为类、技术类、团队类、失败类等常见问题类型的追问。

### （六）不同岗位的完整性侧重点
1. **技术岗位**：结构（问题-分析-方案-实施-验证）、细节（技术选型/架构/性能数据）、风险（技术局限性/扩展性）；
2. **产品岗位**：结构（需求-分析-设计-上线-反馈）、细节（调研数据/A/B测试/业务指标）、风险（用户反馈/竞品/迭代）；
3. **运营岗位**：结构（目标-策略-监控-分析-优化）、细节（活动数据/ROI/用户反馈）、风险（成本/留存/可复制性）；
4. **管理岗位**：结构（战略-规划-执行-结果-复盘）、细节（团队管理/决策/激励）、风险（人员/资源/文化）。

### （七）通用终极框架：PREP
- P（Point）：核心观点；
- R（Reason）：支撑理由；
- E（Example）：具体案例；
- P（Point）：重申观点。

**核心原则**：面试回答的完整性并非“说得越多越好”，而是在有限时间内给出**信息密度最高、逻辑最严谨、可验证**的答案。