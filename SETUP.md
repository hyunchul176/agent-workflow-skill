# 环境搭建指南（Windows 从零开始）

> 在一台新的 **Windows 11** 电脑上，从零搭好这套终端工作流环境：
> **Starship（提示符）+ WSL/Ubuntu + Zellij（多窗格）+ Claude Code + agent-workflow Skill**。
> 估计耗时：30~60 分钟（取决于网速）。全程纯本地，**不需要任何 SSH / 远程服务器**。

> 本指南配套文件都在本交付包里（`dotfiles/`、`.claude/skills/agent-workflow/`），不依赖任何云盘账号。

---

## 你将得到什么

- 漂亮的 **Pastel Powerline** 终端提示符（Starship + Nerd Font 图标）
- **WSL/Ubuntu** Linux 环境跑 Claude Code
- **Zellij** 一键多窗格布局（一个会话里同时开多个面板）
- **Claude Code** + 预装好的 **多 Agent 工作流 Skill**

---

## 阶段 1 — 装 Windows 端工具（15 分钟）

> 全部用 **PowerShell**（普通权限即可，除非特别说明）。

### 1.1 检查 winget

```powershell
winget --version
```
Win 11 自带。没有就去 Microsoft Store 装 "App Installer"。

### 1.2 安装 Starship

```powershell
winget install --id Starship.Starship -e --accept-source-agreements --accept-package-agreements
```

### 1.3 安装 JetBrainsMono Nerd Font

```powershell
winget install --id DEVCOM.JetBrainsMonoNerdFont -e --accept-source-agreements --accept-package-agreements
```

### 1.4 让 Windows Terminal 用这个字体

打开 Windows Terminal → `Ctrl + ,` → 选 PowerShell 配置文件 → **外观 → 字体** → 选 `JetBrainsMono Nerd Font` → 保存。
（不设字体的话，提示符里的图标会变成方块/乱码。）

### 1.5 配置 PowerShell profile

```powershell
New-Item -ItemType File -Path $PROFILE -Force | Out-Null
Set-Content -Path $PROFILE -Value @"
Invoke-Expression (&starship init powershell)
Import-Module Terminal-Icons
"@
```

### 1.6 放置 Starship 配置（用本交付包里的 `dotfiles/starship.toml`）

```powershell
# 把 <交付包路径> 换成你解压本交付包的位置
New-Item -ItemType Directory -Path "$env:USERPROFILE\.config" -Force | Out-Null
Copy-Item "<交付包路径>\dotfiles\starship.toml" "$env:USERPROFILE\.config\starship.toml"
```
或者不用本包配置、直接生成官方预设：
```powershell
starship preset pastel-powerline -o "$env:USERPROFILE\.config\starship.toml"
```

### 1.7 安装 Terminal-Icons 模块

```powershell
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force -Scope CurrentUser
Set-PSRepository -Name PSGallery -InstallationPolicy Trusted
Install-Module -Name Terminal-Icons -Repository PSGallery -Scope CurrentUser -Force
```

### 1.8 重启 PowerShell 验证

关掉所有 PowerShell 窗口，开新窗口。应看到 Pastel Powerline 提示符，`ls` 时文件名前有彩色图标。

---

## 阶段 2 — 装 WSL + Ubuntu（10~30 分钟，看网速）

### 2.1 以**管理员身份**打开 PowerShell
`Win + X` → "终端（管理员）"。

### 2.2 安装 WSL + Ubuntu 24.04 LTS
```powershell
wsl --install -d Ubuntu-24.04 --web-download
```
> 卡 0% 时：`Ctrl+C` → `wsl --update --web-download` → 重试。

### 2.3 设置 UNIX 用户名 / 密码
装完会弹 Ubuntu 窗口要求设置用户名密码。记住密码（后面 `sudo` 要用）。

### 2.4 验证
```bash
whoami && pwd && cat /etc/os-release | head -2
```

---

## 阶段 3 — 配置 WSL 内部环境（10 分钟）

> 在 **Ubuntu 窗口里**操作。

### 3.1 安装 nvm + Node.js LTS
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install --lts && nvm use --lts && nvm alias default 'lts/*'
node --version && npm --version
```

### 3.2 安装 Claude Code
```bash
npm install -g @anthropic-ai/claude-code
claude --version
```
首次运行 `claude` 需要**登录 Anthropic 账号**（浏览器授权）。

### 3.3 安装 Zellij
```bash
sudo snap install zellij --classic
echo 'export PATH="$PATH:/snap/bin"' >> ~/.bashrc
exec bash -l
zellij --version
```

### 3.4 放置 Zellij 配置（用本交付包里的 `dotfiles/zellij/`）

WSL 里访问 Windows 文件是 `/mnt/c/...`。把 `<交付包路径>` 换成本包在 Windows 下的路径（如 `/mnt/c/Users/<你的用户名>/Desktop/agent-workflow-skill`）：
```bash
mkdir -p ~/.config/zellij/layouts
cp "<交付包路径>/dotfiles/zellij/config.kdl" ~/.config/zellij/
cp "<交付包路径>/dotfiles/zellij/layouts/my0.kdl" ~/.config/zellij/layouts/
```

### 3.5 配置 bashrc 快捷别名
```bash
cat >> ~/.bashrc << 'EOF'

# === workflow shortcuts ===
# 把 <项目路径> 换成你的实际项目目录
alias new='cd <项目路径> && zellij -s work -n my0'
alias old='cd <项目路径> && zellij attach work'
EOF
source ~/.bashrc
```

---

## 阶段 4 — 安装多 Agent 工作流 Skill（5 分钟）

> 这是本交付包的核心功能。详见 `README.md`。

1. 把本交付包里的 `.claude/skills/agent-workflow/` 整个文件夹，复制到你的**目标项目根目录**下：
   ```bash
   mkdir -p <项目路径>/.claude/skills
   cp -r "<交付包路径>/.claude/skills/agent-workflow" <项目路径>/.claude/skills/
   ```
2. 在该项目里启动 Claude Code：
   ```bash
   cd <项目路径> && claude
   ```
3. 在 Claude Code 里运行：
   ```
   /agent-workflow init
   ```
   它会自动搭好 `.workflow/` 状态目录、安装工作流协议、让你填项目目标。

之后每条消息以 "你是 executor / planner / …" 开头即可触发工作流。完整用法见 `README.md`。

---

## 阶段 5 — 验收清单

PowerShell：
- [ ] `starship --version` 有版本号
- [ ] `ls` 有彩色文件图标
- [ ] 提示符是 Pastel Powerline 样式

Ubuntu：
- [ ] `node --version` >= 18
- [ ] `claude --version` 有版本号
- [ ] `zellij --version` 有版本号
- [ ] `new` 能启动 Zellij 3-pane 布局
- [ ] Zellij 里跑 `claude` 能进入交互界面（首次需登录）
- [ ] `/agent-workflow init` 成功搭出 `.workflow/`

全部 ✅ → 搭建完成 🎉

---

## 附录：常见问题

**Q1: `new` 报 "There is no active session!"**
→ alias 必须用 `-n my0`，不要用 `-l my0`。见 3.5。

**Q2: WSL 里 `claude` 命令找不到**
→ nvm 装的全局包需重启 shell：`exec bash -l` 或新开 Ubuntu 窗口。

**Q3: Zellij 启动后所有 pane 卡在 `/mnt/c/...`**
→ 布局里有硬编码 cwd，删掉：
```bash
sed -i '/cwd[ =]/d' ~/.config/zellij/layouts/my0.kdl
```
（本包自带的 `my0.kdl` 已无此问题，无需处理。）

**Q4: 提示符图标是方块/乱码**
→ Windows Terminal 没设 Nerd Font，回到 1.4。

**Q5: PowerShell profile 不生效**
→ 确认 `Get-Content $PROFILE` 内容正确，重开窗口。
