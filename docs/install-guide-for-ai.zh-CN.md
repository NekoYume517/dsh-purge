# dsh-purge 安装指南（AI Agent 操作手册）

> 本文档面向 AI Agent（Claude Code / ZCode / Codex 等），指导如何在一台 Windows 机器上安装 DeepSeek Harness 插件 **dsh-purge**，重点是**官方桌面端（dsh desktop EXE）**的正确安装方式。文末附有实测踩过的全部坑，遇到失败先对照排查。

---

## 1. dsh-purge 是什么

[dsh-purge](https://github.com/YuJunZhiXue/dsh-purge) 是 DeepSeek Harness（下称 dsh）的社区插件。原作已暂停更新（只适配到 dsh 0.2.0-rc.2）。当前维护中的分叉：

- **仓库**：<https://github.com/NekoYume517/dsh-purge>
- **版本**：v1.1.71-fork.3（基于上游 1.1.71，MIT，署名要求见上游 README）
- **兼容**：dsh 0.2.0-rc.2 及 **0.2.1-rc.2**（补注册了 0.2.1 移除的 `cwd` 提示词变量，否则每轮对话报 `unknown prompt variable "{{cwd}}"` 并中断）
- **npm 上没有这个包**（`dsh-purge` 404），**只能从 GitHub tarball 安装**

---

## 2. 安装前检查

### 2.1 确认 dsh 宿主版本

dsh-purge fork.3 只保证适配 **dsh 0.2.x**。查桌面端版本：

```powershell
# 桌面 EXE 版本（0.2.1.x 的壳 + 内部 dsh 0.2.1-rc.2 均可）
(Get-Item 'D:\software\dsh\DeepSeek Harness.exe').VersionInfo.ProductVersion
```

同时确认 EXE 自带的 dsh 核心版本（以宿主日志为准，桌面壳 0.2.1.0 对应 dsh 0.2.1-rc.2）。

**红线**：凡是依赖 `@deepseek-ai/dsh-*` 为 `0.1.x` 或 `0.11.x` 的插件（如 `hermes-dsh-bridge@0.11.4`），装进 0.2.x 的 profile 会遮蔽官方同名核心包，导致宿主启动崩溃（`1 required plugin did not activate`）。装任何插件前先看它的 peerDependencies。

### 2.2 确认 dsh CLI 可用（两条路径，认准 EXE 自带的那条）

| CLI | 路径 | pnpm 版本 | 用途 |
| --- | --- | --- | --- |
| **EXE 自带（必用）** | `<EXE安装目录>\resources\runtime\cli\bin\dsh.cmd` | pnpm 11 | 装/卸/管理 profile 插件 |
| npm 全局 `dsh` | `C:\Users\<user>\AppData\Roaming\npm\dsh.cmd` 或 `D:\npm-global\dsh.cmd` | pnpm 10 或 11 | 查配置/诊断；**别用它管理 desktop profile** |

npm 全局的 `dsh` 对 `--profile desktop` 会报 `profile "desktop" is managed exclusively by the Electron application`——这不是故障，是保护。**desktop profile 的所有插件操作必须用 EXE 自带 CLI。**

### 2.3 环境要求

- GitHub 访问需要代理（中国大陆环境），示例用 Clash：`HTTPS_PROXY=http://127.0.0.1:7890`
- Node.js ≥18（宿主自带，无需单独装）

---

## 3. 安装：官方桌面端（dsh desktop EXE）——重点

### 3.1 前置：完全退出桌面端

托盘图标右键退出（或任务管理器确认没有 `DeepSeek Harness.exe` 进程）。文件锁会导致 pnpm 安装失败或激活异常。

```powershell
Get-Process 'DeepSeek Harness' -ErrorAction SilentlyContinue | Stop-Process -Force
```

### 3.2 找到 EXE 自带 CLI 的实际路径

桌面端可能装在不同目录，先定位：

```bash
# 常见位置二选一
ls "D:/software/dsh/resources/runtime/cli/bin/dsh.cmd"     # 本机实测位置
ls "C:/Program Files/DeepSeek Harness/resources/runtime/cli/bin/dsh.cmd"
```

下文用变量 `<DSH_CLI>` 代指该 dsh.cmd 完整路径。

### 3.3 安装命令（Git Bash 示例）

```bash
HTTPS_PROXY=http://127.0.0.1:7890 HTTP_PROXY=http://127.0.0.1:7890 \
  "<DSH_CLI>" plugin --profile desktop add \
  "https://github.com/NekoYume517/dsh-purge/archive/refs/heads/master.tar.gz"
```

成功标志：输出 `+ dsh-purge <版本号>` 和 `Done in ...s`。

**注意**：

- 优先用 `master.tar.gz` URL（随仓库更新）；固定版本用 release 资产 `https://github.com/NekoYume517/dsh-purge/releases/download/v1.1.71-fork.3/dsh-purge-1.1.71-fork.3.tgz`
- 本地已下载的 tgz 也可装：`add "C:\path\to\dsh-purge-1.1.71-fork.3.tgz"`，但 dep 引用会变成 `file:` 路径——**tgz 别放临时目录**，挪走后要重装
- 若报 `ERR_PNPM_MISSING_TARBALL_INTEGRITY`，见 §6.3 修复

### 3.4 安装后验证（必须逐项做）

```bash
# ① 包结构完整（之前版本的坑：lib/ 缺失导致静默不激活）
ls "C:/Users/<user>/.dsh/profiles/desktop/node_modules/dsh-purge/lib/index.js"

# ② 版本正确
python -c "import json; print(json.load(open(r'C:/Users/<user>/.dsh/profiles/desktop/node_modules/dsh-purge/package.json',encoding='utf-8'))['version'])"
# 期望输出: 1.1.71-fork.3 或更新

# ③ 依赖与 bundle 注册
python -c "import json; d=json.load(open(r'C:/Users/<user>/.dsh/profiles/desktop/package.json',encoding='utf-8')); print(d['dependencies'].get('dsh-purge')); print('dsh-purge' in d['dsh']['profile']['bundles'])"
# 期望: 依赖 URL/file 引用 + True

# ④ 语法与导入 smoke
node --check "C:/Users/<user>/.dsh/profiles/desktop/node_modules/dsh-purge/lib/index.js"
cd "C:/Users/<user>/.dsh/profiles/desktop/node_modules/dsh-purge" && node -e "import('./lib/index.js').then(m=>console.log('OK',m.name))"
```

### 3.5 让插件生效（需要用户手动）

1. **完全退出**桌面端（托盘退出，不是关窗口），重新打开 EXE
2. 打开任意会话，**会话标题旁**会出现 **dsh-purge** 入口
3. 点开 → **清洗** → **应用**（成功会自动重启宿主；补丁没打完不会重启）
4. 重启后**新开一轮对话**，注入才进当前会话

### 3.6 激活状态验证

- 插件页（插件 → dsh-purge）：4 个组件应为"运行中"而非"未运行"
- purge 内部日志 `%TEMP%\dsh-purge-desktop.log`：重启后应出现新的 `[boot] enabled=true surface=desktop` 行
- 补丁状态：`~/.dsh/dsh-purge/applied.json` 记录已应用版本

---

## 4. 安装：官方 dsh web（`dsh web`，可选）

web profile 用 npm 全局 CLI（或 EXE 自带 CLI 均可）：

```bash
HTTPS_PROXY=http://127.0.0.1:7890 HTTP_PROXY=http://127.0.0.1:7890 \
  dsh plugin --profile web add \
  "https://github.com/NekoYume517/dsh-purge/archive/refs/heads/master.tar.gz"
```

装完**完全退出并重启** `dsh web`，同样点会话标题旁 dsh-purge → 清洗 → 应用。web 与 desktop 的额度/开关/profile 状态互相独立。

---

## 5. 卸载 / 更新

```bash
# 卸载（EXE 自带 CLI）
"<DSH_CLI>" plugin --profile desktop remove dsh-purge

# 更新 = 重新 add 同一个 URL（版本变了会自动替换）
```

卸载后如需还原宿主补丁：插件还在时先在 UI 里点「还原」，或用 `dsh-purge --revert`（purge 自带 CLI，在包的 `bin/` 里）。

---

## 6. 实测坑清单（按出现频率排序）

### 6.1 组件全部显示"未运行"

安装成功但插件页 4 个组件都"未运行"、且 `%TEMP%\dsh-purge-desktop.log` 没有新 `[boot]` 行 → 插件没被激活。按顺序排查：

1. **包结构残破**：查 `node_modules/dsh-purge/lib/index.js` 是否存在。pnpm 解某些 tarball（如 git archive 直出的 PAX 无顶层目录格式）可能得到平铺结构丢掉 `lib/`。修复：改用 npm pack 形状的 tarball（release 资产）或重装
2. **package.json 的 `dsh` 键下有未知子键**：宿主对 bundle manifest 严格校验，多余键 → 整包静默拒绝。fork 的 manifest 只能有 `dsh.bundle` / `dsh.client` / `dsh.install`
3. 没重启宿主（必须托盘退出再开）

### 6.2 `unknown prompt variable "{{cwd}}"`

用的还是上游原版（只适配 0.2.0）。换 fork.3。

### 6.3 `ERR_PNPM_MISSING_TARBALL_INTEGRITY`

首次安装失败后 lockfile 残留无 integrity 的条目，之后所有 `add` 都被挡。修法：

```bash
# 删 profile 的 pnpm-lock.yaml 后重装
rm "C:/Users/<user>/.dsh/profiles/desktop/pnpm-lock.yaml"
# 仍报错则手动补 integrity：下载 tarball 算 sha512（base64），写入 lockfile 的 resolution: {integrity: sha512-xxx, tarball: url}
```

### 6.4 pnpm 版本冲突 / store 报错

`dsh plugin` 底层是 pnpm 透传。npm 全局 CLI（可能带 pnpm 10）与 EXE 自带 CLI（pnpm 11）混用同一 profile 会报 store location/版本错误。**始终用 EXE 自带 CLI**；已经混了就删 `node_modules` + `pnpm-lock.yaml` 重装。

### 6.5 启动崩溃 `1 required plugin did not activate`

装了依赖 0.1.x 的插件（如 hermes-dsh-bridge 0.11.4）。卸掉该插件 + pnpm 重解析即可。另注意：崩溃弹窗的 **"Disable third-party plugins" 按钮会把用户补丁层 `cordis.patch.yml` 整个清空**（自动备份为 `cordis.patch.yml.bak-<时间戳>`，内含 OFM 提供商、默认模型等设置），处理完要从 bak 恢复。

### 6.6 启动崩溃 `ENOENT ... office-skills\office\SKILL.md`

自动更新后 `resources\runtime` 载荷缺失。修法：从 `%LOCALAPPDATA%\@deepseek-aidsh-desktop-updater\installer.exe`（NSIS 内嵌 7z，扫描签名 `37 7A BC AF 27 1C` 定位偏移）解出 `resources/runtime` 整树覆盖到 EXE 安装目录。与插件无关。

### 6.7 诊断技巧：绕过 desktop 保留名抓真实报错

`desktop` 这个 profile 名被 EXE 锁定，CLI 无法对其 dump-config，激活失败的 stderr 也看不到。**诊断方法**：把整个 profile 复制到临时 DSH_HOME 下换个名字，用 EXE 自带 CLI 跑一次，"did not activate" 的完整错误就打出来了：

```bash
mkdir -p ~/.dsh-diag/profiles/desktop-diag
cp ~/.dsh/profiles/desktop/{package.json,cordis.patch.yml,pnpm-workspace.yaml} ~/.dsh-diag/profiles/desktop-diag/
cmd //c "mklink /J C:\Users\<user>\.dsh-diag\profiles\desktop-diag\node_modules C:\Users\<user>\.dsh\profiles\desktop\node_modules"
DSH_HOME="C:/Users/<user>/.dsh-diag" "<DSH_CLI>" --profile desktop-diag 2>&1 | head -40
```

---

## 7. 快速参考卡

```bash
# 变量
DSH_CLI="D:/software/dsh/resources/runtime/cli/bin/dsh.cmd"
PROXY="HTTPS_PROXY=http://127.0.0.1:7890 HTTP_PROXY=http://127.0.0.1:7890"
URL="https://github.com/NekoYume517/dsh-purge/archive/refs/heads/master.tar.gz"

# 装（desktop：先退出 EXE）
env $PROXY "$DSH_CLI" plugin --profile desktop add "$URL"
# 装（web）
env $PROXY dsh plugin --profile web add "$URL"

# 验结构
ls ~/.dsh/profiles/desktop/node_modules/dsh-purge/lib/index.js

# 卸
env $PROXY "$DSH_CLI" plugin --profile desktop remove dsh-purge
```

装完后：**托盘退出 → 重开 EXE → 会话标题旁 dsh-purge → 清洗 → 应用 → 自动重启 → 新开对话生效。**
