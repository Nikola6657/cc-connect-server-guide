# 如何将cc connect部署到服务器上运行

这是一份面向新手的完整部署教程，用于在新的 Linux 云服务器上运行 Codex，并通过手机微信发送任务、接收执行结果。已有部署的服务器不需要重复执行。

目标：手机微信 → cc-connect → 云端 Codex → 调用中转站、执行程序 → 微信回复。

本文沿用本次实际部署的版本：Codex CLI 0.163.0、cc-connect 1.5.0、Node.js 22.22.0。它们是本次部署的固定版本，不代表最新版本。系统按阿里云 Alibaba Cloud Linux 4、x86_64 编写；Ubuntu、ARM 服务器不要直接照抄安装命令。

## 先认识操作位置

- 云控制台：阿里云网页，用于查看实例、启动和远程连接。
- 服务器终端：远程连接打开的黑色命令窗口。本文 Linux 命令都在这里输入，不是在电脑 PowerShell 中输入。
- root：服务器管理员；安装软件和管理服务用它。
- codexbot：专门运行 AI 的普通用户。
- 命令框内的内容可以复制；不要复制说明文字、代码框边缘或终端提示符。
- 一步完成并检查结果后再继续；出现报错先停在该步骤。

## 1. 进入服务器终端

打开阿里云控制台，找到服务器实例，确认状态为“运行中”，使用实例的远程连接功能进入终端。不同实例产品可能显示 Workbench 或其他连接入口。

本教程后续管理员步骤要求 root 终端。先输入：

```bash
whoami
cat /etc/os-release
uname -m
```

成功标志：第一条输出 root；系统是 Alibaba Cloud Linux 4；架构输出 x86_64。

如果不是 root，但当前账号有 sudo 权限，输入 sudo -i 切换；如果不具备权限，需要使用管理员连接方式。

云控制台内的连接入口与从自己电脑 SSH 登录不同。若要从电脑使用 SSH，需要配置密钥，并确保安全组允许自己电脑的公网 IP 访问 TCP 22。微信长轮询方案本身无需开放网站端口。

## 2. 安装基础工具和 Node.js（root）

先安装工具：

```bash
dnf install -y curl tar xz git python3 nano
```

再检查 Node：

```bash
node -v
npm -v
```

如果已经是 Node.js 22 或更高，并且 npm 正常，可以跳过下面的 Node 安装。部分服务器镜像已经预装 Node.js，先检查版本再决定是否安装。

如果提示找不到命令，下面是为全新 x86_64 服务器安装固定版本的方法：

```bash
cd /tmp
curl -fLO https://nodejs.org/dist/v22.22.0/node-v22.22.0-linux-x64.tar.xz
curl -fLO https://nodejs.org/dist/v22.22.0/SHASUMS256.txt
python3 -c 'import hashlib,pathlib; n="node-v22.22.0-linux-x64.tar.xz"; lines=pathlib.Path("SHASUMS256.txt").read_text().splitlines(); expected=next(x.split()[0] for x in lines if x.split()[-1]==n); assert hashlib.sha256(pathlib.Path(n).read_bytes()).hexdigest()==expected, "校验失败"; print("Node 安装包校验通过")'
tar -xJf node-v22.22.0-linux-x64.tar.xz -C /opt
ln -s /opt/node-v22.22.0-linux-x64/bin/node /usr/local/bin/node
ln -s /opt/node-v22.22.0-linux-x64/bin/npm /usr/local/bin/npm
ln -s /opt/node-v22.22.0-linux-x64/bin/npx /usr/local/bin/npx
node -v
npm -v
```

此安装方法仅适用于尚无这些命令的新服务器。如果提示目标文件已存在，先检查现有安装，不要强行覆盖。

如果下载超时，先解决下载问题；不要继续解压不存在或未校验的安装包。国内服务器访问 nodejs.org、GitHub 或 npm 可能不稳定，这与你的中转站能否访问是两件事。

## 3. 安装 Codex 和 cc-connect（root）

安装固定版本 Codex：

```bash
npm install -g @openai/codex@0.163.0
codex --version
command -v codex
```

如果 Node 是按上一步安装到 /opt 的，npm 全局命令可能也在该目录。若 codex 提示找不到命令，可先检查：

```bash
npm prefix -g
ls /opt/node-v22.22.0-linux-x64/bin/codex
```

确认该文件存在后，再建立链接：

```bash
ln -s /opt/node-v22.22.0-linux-x64/bin/codex /usr/local/bin/codex
codex --version
```

安装 cc-connect 官方 Linux 二进制：

```bash
cd /tmp
curl -fL --retry 3 -o cc-connect-v1.5.0-linux-amd64.tar.gz https://github.com/chenhg5/cc-connect/releases/download/v1.5.0/cc-connect-v1.5.0-linux-amd64.tar.gz
printf '%s  %s\n' '72859035a1ee011b710204fc508de711838f919eb2ae6f104f1ddb3e5cd8ca87' 'cc-connect-v1.5.0-linux-amd64.tar.gz' | sha256sum -c -
```

必须看到校验 OK，再继续：

```bash
mkdir -p /tmp/cc-connect-install-v1.5.0
tar -xzf /tmp/cc-connect-v1.5.0-linux-amd64.tar.gz -C /tmp/cc-connect-install-v1.5.0
install -m 0755 /tmp/cc-connect-install-v1.5.0/cc-connect-v1.5.0-linux-amd64 /usr/local/bin/cc-connect
cc-connect --version
```

如果服务器下载 GitHub 文件失败：在电脑浏览器中打开上面的下载地址，下载后用阿里云远程终端的上传功能，或 SFTP 工具上传到 /tmp，再做校验和安装。电脑下载后上传也是可用的安装方式。

## 4. 创建运行账号（root）

仅第一次创建时执行：

```bash
useradd --create-home --shell /bin/bash codexbot
install -d -o codexbot -g codexbot -m 0755 /home/codexbot/workspace
su - codexbot
```

检查：

```bash
whoami
echo "$HOME"
command -v codex
command -v cc-connect
```

应该看到 codexbot、/home/codexbot，以及两个程序的路径。

后面第 5～8 步都用 codexbot 操作。

## 5. 配置中转站（codexbot）

创建目录和配置文件：

```bash
umask 077
mkdir -p ~/.codex
nano ~/.codex/config.toml
```

在 nano 中粘贴下面内容：

```toml
model_provider = "relay"
model = "gpt-6.1-sol"
model_reasoning_effort = "high"
approval_policy = "never"
sandbox_mode = "workspace-write"

[model_providers.relay]
name = "relay"
base_url = "https://api.example.com/v1"
wire_api = "responses"
requires_openai_auth = true

[sandbox_workspace_write]
network_access = true
```

nano 保存方法：按 Ctrl+O，按回车确认文件名，再按 Ctrl+X 退出。此后编辑其他文件也用这套方法。

上面的 https://api.example.com/v1 是占位地址，必须替换为自己的服务商提供的 API 基础地址；gpt-6.1-sol 也必须是该服务商实际支持的模型 ID，否则按其说明修改。此模板要求服务商兼容 Responses API。不要填写电脑上 CC Switch 的 127.0.0.1 代理地址；服务器上的 127.0.0.1 指服务器自己。

接着输入下面整行命令：

```bash
python3 -c 'import getpass,json,os,pathlib; p=pathlib.Path.home()/".codex"/"auth.json"; k=getpass.getpass("请输入中转站 API Key（不显示输入）: ").strip(); assert k, "API Key 不能为空"; p.write_text(json.dumps({"OPENAI_API_KEY":k})); os.chmod(p,0o600); print("API Key 已保存")'
chmod 600 ~/.codex/config.toml
```

看到提示后粘贴真实 API Key 并回车。输入时没有星号、没有字符显示，属于正常现象。Key 不需要发给聊天机器人。

此方案里 CC Switch 不是服务器必需组件，API 配置直接写入云端 Codex。

## 6. 先单独测试 Codex（codexbot）

```bash
cd ~/workspace
codex exec --skip-git-repo-check '请用 Python 计算 1 到 100 的和，并告诉我实际运行结果。'
```

首次调用需要消耗中转站额度。成功标志：完成 Python 调用，并回复 5050。

出现鉴权失败、模型不存在或网络错误时，先检查第 5 步，不要急着接微信。网络或协议问题也可能来自中转站，需要结合具体报错判断。

## 7. 配置 cc-connect 并扫码连接微信（codexbot）

创建配置：

```bash
mkdir -p ~/.cc-connect
nano ~/.cc-connect/config.toml
```

粘贴：

```toml
data_dir = "/home/codexbot/.cc-connect"
language = "zh"

[display]
mode = "quiet"
thinking_messages = false
tool_messages = false
reply_footer = false
show_context_indicator = false

[[projects]]
name = "codex-weixin"
reset_on_idle_mins = 0

[projects.agent]
type = "codex"

[projects.agent.options]
work_dir = "/home/codexbot/workspace"
model = "gpt-6.1-sol"
mode = "full-auto"

[[projects.platforms]]
type = "weixin"

[projects.platforms.options]
token = ""
allow_from = ""
burst_limit = 60
burst_window_secs = 60
```

这里 token 和 allow_from 先留空，下一步扫码工具会填写。

```bash
cc-connect weixin setup --config /home/codexbot/.cc-connect/config.toml --project codex-weixin --set-allow-from-empty
```

用手机微信扫描终端显示的二维码，并确认登录。若二维码显示不完整，可以放大终端，或使用命令提供的二维码链接，按终端提示处理。

成功后，连接令牌及账号信息会写回配置。使用 nano 检查 allow_from 已填入你扫码关联的微信用户 ID，随后退出；不要把完整配置截图公开。若仍为空，先根据扫码结果和项目说明补齐允许的用户 ID。

```bash
chmod 600 ~/.cc-connect/config.toml
```

如果你的电脑上已有同一微信机器人的 cc-connect，应先停止本机进程及自动启动任务，避免两端同时轮询。从零安装使用上面的扫码流程；迁移已有连接则需要另外迁移登录状态。

发送预算及 quiet 设置用于规避 cc-connect 1.5.0 过程消息消耗预算、最终回复发不出来的问题。这只是本地发送预算设置，不保证绕过微信网关的实际限流；不要短时间密集发送大量任务。

## 8. 前台测试微信（codexbot）

启动：

```bash
cc-connect --config /home/codexbot/.cc-connect/config.toml
```

此时终端持续显示日志、不返回命令提示符，是正常运行状态。等待 platform ready 等就绪日志。

在手机微信对应的机器人聊天中发送：

> 请运行 hostname 和 pwd，并用 Python 计算 1 到 100 的和，只回复最终结果。

成功标志：微信收到服务器名、/home/codexbot/workspace 和 5050。

收到后回服务器终端，按 Ctrl+C 停止前台进程，再执行：

```bash
exit
whoami
```

应该回到 root。前台测试进程必须停掉，再启动下一步后台服务。

## 9. 设置后台服务（root）

先确认程序路径：

```bash
command -v cc-connect
command -v codex
```

下面假定两者都能通过 /usr/local/bin 找到。如果 Codex 实际位于其他目录，需要把其目录加入服务的 PATH。

```bash
nano /etc/systemd/system/cc-connect-cloud.service
```

粘贴：

```ini
[Unit]
Description=Weixin to Codex cloud bridge
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=120
StartLimitBurst=10

[Service]
Type=simple
User=codexbot
Group=codexbot
WorkingDirectory=/home/codexbot/workspace
Environment=HOME=/home/codexbot
Environment=PATH=/usr/local/bin:/usr/bin:/bin
ExecStart=/usr/local/bin/cc-connect --config /home/codexbot/.cc-connect/config.toml
Restart=on-failure
RestartSec=8
TimeoutStopSec=30
UMask=0077
NoNewPrivileges=true
StandardOutput=journal
StandardError=journal
SyslogIdentifier=cc-connect-cloud

[Install]
WantedBy=multi-user.target
```

保存后执行：

```bash
systemctl daemon-reload
systemctl enable --now cc-connect-cloud
systemctl is-active cc-connect-cloud
systemctl is-enabled cc-connect-cloud
journalctl -u cc-connect-cloud -n 40 --no-pager
```

成功标志：active、enabled；稍等后日志出现微信平台 ready。若暂未 ready，等约 20～30 秒，再查一次日志。

后台服务运行后，不再需要保持这个终端窗口打开。

## 10. 验证电脑关机后仍能使用

先关闭电脑上的 cc-connect，以及相关自动启动任务，再关闭电脑，用手机微信发送：

> 请用 Python 计算 123 × 456，只回复实际运行结果。

收到 56088，说明不依赖电脑的目标已经实现。

如果是专门部署的全新服务器，且没有其他任务，可以在方便时从阿里云控制台重启服务器。等启动完成后再发一条微信指令，验证整机重启恢复。关掉 SSH 窗口、重启机器人服务、重启服务器是三种不同的验证。

## 日常维护（root 终端）

```bash
# 查看服务状态；按 q 退出状态页
systemctl status cc-connect-cloud

# 查看最近日志
journalctl -u cc-connect-cloud -n 80 --no-pager

# 修改配置后重启
systemctl restart cc-connect-cloud

# 暂停机器人
systemctl stop cc-connect-cloud

# 恢复机器人
systemctl start cc-connect-cloud

# 看内存、磁盘
free -h
df -h
```

修改运行用户配置的方法：

```bash
su - codexbot
nano ~/.cc-connect/config.toml
exit
systemctl restart cc-connect-cloud
```

微信登录过期需要重新扫码时，先在 root 终端停止服务，再切换 codexbot 执行第 7 步扫码命令；成功后回 root 启动服务。

## 常见问题

| 现象 | 先检查哪里 |
|---|---|
| command not found | 程序是否安装成功，命令目录是否在 PATH |
| 401 或鉴权失败 | API Key、API 基础地址、中转站账户状态 |
| 模型不存在 | 中转站实际支持的模型名是否为 gpt-6.1-sol |
| Codex 正常，微信不回复 | 微信平台 ready、allow_from、本机是否仍在同时轮询、发送失败日志 |
| 只收到过程，没收到答案 | quiet 显示设置及 cc-connect 1.5.0 的发送预算 |
| 终端里正常，关掉终端就不行 | 是否只做了前台测试，systemd 服务是否 active |
| 电脑关机后不行 | 消息是否实际仍由电脑处理，云端服务是否 ready |
| 程序被杀死或卡住 | 内存、磁盘、程序是否超出 2GB 服务器能力 |

## 参考

- Codex CLI 官方文档：https://developers.openai.com/codex/cli/
- cc-connect 项目：https://github.com/chenhg5/cc-connect
- 本次固定版本配置示例：https://github.com/chenhg5/cc-connect/blob/v1.5.0/config.example.toml
- 微信接入说明（主分支文档可能随版本变化）：https://github.com/chenhg5/cc-connect/blob/main/docs/weixin.md

本文根据一套实际部署结果以及已检查的 cc-connect 配置、源码整理。安装、扫码步骤不代表已在另一台空白服务器重新完整执行；换系统、版本或中转站时需要重新检查。
