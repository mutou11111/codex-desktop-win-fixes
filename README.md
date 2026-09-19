# Codex / ChatGPT 桌面版（Windows）反复故障的排查与修复记录

> 记录一次在 Windows 上长期使用 ChatGPT / Codex 桌面版（Microsoft Store 包 `OpenAI.Codex`）
> 时反复遇到的几类故障、定位过程与最终修复方案。
>
> 适用版本区间：`26.908.x` ~ `26.915.x`（CLI `0.153.x` ~ `0.155.x`）。
> 文中所有用户名、机器名、路径、账号 ID 均已替换为占位符。

---

## 目录

- [0. 环境与关键路径速查](#0-环境与关键路径速查)
- [1. 故障一：启动报 `Unable to locate the Codex CLI binary`](#1-故障一启动报-unable-to-locate-the-codex-cli-binary)
- [2. 故障二：能启动，但发消息卡死（按钮变灰 / 转圈）](#2-故障二能启动但发消息卡死按钮变灰--转圈)
- [3. 故障三：所有"旧对话"发送按钮变灰，新对话正常](#3-故障三所有旧对话发送按钮变灰新对话正常)
- [4. 故障四：发送按钮变成灰色方块（停止键），点不动](#4-故障四发送按钮变成灰色方块停止键点不动)
- [5. 故障五：`stream disconnected ... os error 10054`](#5-故障五stream-disconnected--os-error-10054)
- [6. 数据安全：会话存在哪、会不会丢](#6-数据安全会话存在哪会不会丢)
- [7. 桌面版崩了怎么临时继续干活](#7-桌面版崩了怎么临时继续干活)
- [8. 排查工具箱：日志位置与关键词](#8-排查工具箱日志位置与关键词)
- [9. 经验教训（最重要的一节）](#9-经验教训最重要的一节)
- [10. 补充案例：如何判定"这是客户端缺陷，不是我环境的问题"](#10-补充案例如何判定这是客户端缺陷不是我环境的问题)
- [11. 补充：一次未能定位的反复故障 —— 方法论与排除记录](#11-补充一次未能定位的反复故障--方法论与排除记录)

---

## 0. 环境与关键路径速查

| 用途 | 路径 |
| --- | --- |
| 应用包（MSIX） | `C:\Program Files\WindowsApps\OpenAI.Codex_<版本>_x64__<发布者哈希>\app` |
| 应用自带运行时 | `...\app\resources\codex.exe` 等 |
| 应用运行时部署目标 | `%LOCALAPPDATA%\OpenAI\Codex\bin\<内容哈希>\` |
| node / node_repl 运行时 | `%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\<哈希>\bin\` |
| **应用（Electron）日志** | `%LOCALAPPDATA%\Codex\Logs\<年>\<月>\<日>\codex-desktop-*-t0-*.log` |
| **CLI 日志数据库** | `%USERPROFILE%\.codex\logs_2.sqlite`（SQLite，表 `logs`） |
| 会话记录 | `%USERPROFILE%\.codex\sessions\<年>\<月>\<日>\rollout-*.jsonl` |
| 归档会话 | `%USERPROFILE%\.codex\archived_sessions\` |
| 会话名索引 | `%USERPROFILE%\.codex\session_index.jsonl` |
| 线程目录索引 | `%USERPROFILE%\.codex\sqlite\codex-dev.db` |
| 用户配置 | `%USERPROFILE%\.codex\config.toml` |
| 登录态 | `%USERPROFILE%\.codex\auth.json` |
| 会话写锁 | `%USERPROFILE%\.codex\thread-writer-locks\<线程ID>.lock` |

> **要点**：`%USERPROFILE%\.codex` **不在应用包内**，卸载应用**不会**删除它。
> 你的全部会话、配置、登录态都在这里。

---

## 1. 故障一：启动报 `Unable to locate the Codex CLI binary`

### 症状

启动时弹窗：

```
ChatGPT failed to start.
Unable to locate the Codex CLI binary. Set CODEX_CLI_PATH
or ensure the Electron resources include bin/codex.
```

### 根因

应用每次更新都会带一个**新的、约 300MB 的 `codex.exe`**。启动时它要：

1. 把包内的 `codex.exe` 复制到暂存目录
   `%LOCALAPPDATA%\OpenAI\Codex\bin\.staging-<哈希>-<随机>`
2. 再把这个**目录重命名**成
   `%LOCALAPPDATA%\OpenAI\Codex\bin\<内容哈希>`

**第 2 步会失败**，错误形如：

```
bundled_executable_relocation_failed
  destinationPath = ...\bin\<哈希>
  operation       = rename_staging
  errorCode       = EPERM
  originalError   = {"errno":-4048,"code":"EPERM","syscall":"rename",
                     "path":"...\\bin\\.staging-<哈希>-XXXXXX",
                     "dest":"...\\bin\\<哈希>"}
```

于是运行时目录始终不存在 → 应用找不到 CLI → 启动失败。

### 为什么会 EPERM

这不是权限配置问题（实测目录 ACL 正常，用户有 FullControl）。
最典型的原因是**杀毒软件的实时防护**：它会对刚写入的 300MB 大文件持有句柄，
导致紧随其后的**目录重命名被拒绝**，且重试也会失败。

**症状特征：每次应用更新必然复发**（因为哈希每次都变，重命名每次都要做）。

### 如何确认缺哪个组件

解析 Electron 日志（见 §8）：

```powershell
$log = Get-ChildItem "$env:LOCALAPPDATA\Codex\Logs\$(Get-Date -Format yyyy)\$(Get-Date -Format MM)\$(Get-Date -Format dd)" `
       -Filter "*t0*.log" | Sort-Object LastWriteTime -Desc | Select-Object -First 1

# 哪些组件没就位
Select-String -Path $log.FullName -Pattern 'source=missing'

# 需要补哪些哈希 / 文件
Select-String -Path $log.FullName -Pattern 'bundled_executable_relocation_failed destinationPath=' |
  ForEach-Object { $_.Line -replace '.*bin\\','' -replace ' .*','' } | Sort-Object -Unique
```

`source=missing` 的行里带 `component=<组件名>`，能直接看出缺的是 `codex` 还是 `ripgrep`。

### 修复：手动把运行时目录准备好

应用一发现目录已存在就会跳过搬迁（日志里出现 `source=copied`）。

```powershell
$ver = (Get-AppxPackage *OpenAI.Codex*).Version
$src = Join-Path (Get-AppxPackage *OpenAI.Codex*).InstallLocation 'app\resources'
$dst = "$env:LOCALAPPDATA\OpenAI\Codex\bin\<日志里看到的哈希>"

New-Item -ItemType Directory -Path $dst -Force | Out-Null
# codex 组件
'codex.exe','codex-command-runner.exe','codex-windows-sandbox-setup.exe',
'codex-code-mode-host.exe','codex-windows-sandbox-service.exe' |
  ForEach-Object {
    if (Test-Path (Join-Path $src $_)) { Copy-Item (Join-Path $src $_) $dst -Force }
  }
# ripgrep 组件（单独一个哈希目录）
'rg.exe' | ForEach-Object { if (Test-Path (Join-Path $src $_)) { Copy-Item (Join-Path $src $_) $dst -Force } }
```

> `WindowsApps` 内的文件**不能直接执行**（会 Permission denied），但**可以读取/复制**。
> 复制出来就能运行。

### 修好的判定标准

重启应用后，Electron 日志中：

- `bundled_executable_relocation_failed` 计数 = **0**
- `source=missing` 计数 = **0**
- 且出现 `component=codex ... source=copied`
- 进程列表里 `codex.exe` 的路径指向你补的那个目录

### 根治方向

在杀毒软件里把 `%LOCALAPPDATA%\OpenAI\Codex` 加入**信任区 / 排除目录**，
让实时防护不碰这个目录。如果仍复发，可再尝试按**进程**加信任
（`ChatGPT.exe`、`codex.exe`），或在应用更新前后临时关闭文件实时监控。

### ⚠️ 反面教训：不要用 `CODEX_CLI_PATH`

网上很多建议让你设置环境变量 `CODEX_CLI_PATH` 指向某个 `codex.exe`。
**实测这条路是错的**：

- 即使指向的是**同一个二进制**，应用也会走"外部 CLI 覆盖"这条代码路径
- 结果：**启动能过，但发消息依然卡死**，反而增加了一个新的故障维度
- 而且一旦写死某个版本路径，应用下次更新就会失效，形成"每次更新都要改一次"的恶性循环

**正确做法是：让应用用自己部署的运行时。** 保持 `CODEX_CLI_PATH` **不存在**。

---

## 2. 故障二：能启动，但发消息卡死（按钮变灰 / 转圈）

### 症状

应用能开，登录正常、模型列表正常，但输入文字后**发送按钮变灰**，
点击后变成转圈，**消息永远发不出去**。

### 定位方法（决定性）

查 CLI 日志数据库，看**发送请求到底有没有到达后端**：

```python
import sqlite3, datetime
p = r"C:\Users\<USER>\.codex\logs_2.sqlite"
con = sqlite3.connect(f"file:{p}?mode=ro", uri=True)
cur = con.cursor()
for ts, lv, b in cur.execute(
    "SELECT ts, level, substr(feedback_log_body,1,200) FROM logs "
    "WHERE feedback_log_body LIKE '%TurnInput%' ORDER BY id DESC LIMIT 5"):
    print(datetime.datetime.fromtimestamp(ts), lv, b[:150])
```

- **有新的 `TurnInput`** → 请求发出去了，问题在服务端/网络
- **没有新的 `TurnInput`** → **请求根本没发出去**，卡点在客户端（Electron/CLI 这一侧）

### 这个故障的两个真实成因

#### (a) 运行时缺失（见故障一）

运行时没就位时，界面拿不到后端状态，就会一直"不可发送"。
修好运行时后自动恢复。

#### (b) 保留的旧 `config.toml` 与当前安装不一致 ← 最隐蔽的一个

**背景**：`config.toml` 在 `~/.codex` 下，**卸载应用不会删它**。
如果你重装了应用，却把旧 `config.toml` 保留下来，问题就来了——

旧配置里**写死了上一套安装的运行时路径**：

```toml
[mcp_servers.node_repl]
command = '...\runtimes\cua_node\<旧哈希>\bin\node_repl.exe'

notify = [ "...\runtimes\cua_node\<旧哈希>\...\codex-computer-use.exe", "turn-ended" ]

[mcp_servers.node_repl.env]
CODEX_CLI_PATH = '...\bin\<旧哈希>\codex.exe'
```

这些路径**在新安装里可能已不存在**，或与新版应用的期望不符 →
MCP 服务起不来 / 会话初始化不完整 → **任何对话的发送按钮都变灰**。

**症状特征**：
- 与对话长短**无关**（短对话一样灰）
- 与网络**无关**（日志里没有连接错误）
- 重启应用**无效**

**修复**：让应用重新生成一份干净的配置。

```powershell
# 1) 先备份（务必！）
Rename-Item "$env:USERPROFILE\.codex\config.toml" "config.toml.bak-$(Get-Date -Format yyyyMMdd-HHmmss)"

# 2) 关闭应用后重新启动，它会自己生成新的 config.toml
```

生成后日志里 `relocation 失败数 = 0`、`source=missing = 0`，发送即恢复正常。

> **代价**：`config.toml` 里由应用管理的**个性化设置会重置**
>（界面语言、字号、主题色、模型选择、头像等），需要重新设置一遍。
> 建议先记下这些值再动手。

### 修复判定

```python
# 最近若干分钟内出现了新的 TurnInput，即真正发出去了
n = cur.execute("SELECT COUNT(*) FROM logs WHERE ts>=? AND feedback_log_body LIKE '%TurnInput%'",
                ((datetime.datetime.now()-datetime.timedelta(minutes=10)).timestamp(),)).fetchone()[0]
print("最近10分钟 TurnInput 数:", n)   # > 0 即正常
```

---

## 3. 故障三：所有"旧对话"发送按钮变灰，新对话正常

### 症状

新建对话正常可用；但打开**任何已有的对话**，发送按钮都是灰的。

### 排查清单（按顺序排除）

| 假设 | 检查方式 | 实测结论 |
| --- | --- | --- |
| 上下文超限 | 读会话文件尾部的 `token_count` 记录 | 未超限时也照样灰 |
| 被归档（归档=只读） | 看会话在 `sessions\` 还是 `archived_sessions\` | 归档**不是**原因 |
| 工作目录丢失 | 检查 `cwd` / `thread-writable-roots` 是否存在 | 不是 |
| 线程索引损坏 | 检查 `sqlite\codex-dev.db` 的 `local_thread_catalog` 行数 | 索引只剩十几条时侧边栏会空 |
| **旧 config.toml 不一致** | 见 §2(b) | ✅ **本题正解** |

### 修复

同 §2(b)：把 `config.toml` 改名为备份，重启应用让它重新生成。

### 附带发现：重装应用可以修复"线程索引损坏"

如果 `local_thread_catalog` 行数明显少于磁盘上的会话数（例如只有 16 条而磁盘有 182 条），
说明应用的索引没建全。**卸载重装应用**可以让它重新扫描并重建索引
（`~/.codex` 不在包内，卸载不会动你的数据）。

> ⚠️ 重装前**务必先备份 `~/.codex`**，并确保你能重新安装该应用
>（Store 应用通常**无法用命令行重装**，需要从 Microsoft Store 手动安装）。

---

## 4. 故障四：发送按钮变成灰色方块（停止键），点不动

### 症状

- 发送后按钮变成 **方块 ■**（即"停止生成"），**变灰、点不动**
- 该对话卡住；其它对话正常
- **但回复其实能正常收到**

### 解读

方块 = **界面认为"本轮仍在生成中"**。所以这不是"发不出去"，
而是**界面状态没有复位**——服务端那一轮实际上已经正常结束了。

### 日志特征（用于区分）

- 对应的回合 `task_started → task_complete` 完整结束
- **没有**任何连接断开/重试事件
- 没有 `turn_aborted`

### 处置

1. **先点那个方块（停止）**，尝试让界面复位
2. 不行就**重启应用**

---

## 5. 故障五：`stream disconnected ... os error 10054`

### 症状

回答过程中断：

```
stream disconnected before completion: failed to send websocket request:
IO error: 远程主机强迫关闭了一个现有的连接。 (os error 10054)
```

### 解读

`10054` = `WSAECONNRESET`，**WebSocket 长连接被对端强制重置**。属于网络层。

CLI 自带重试（默认最多 5 次）：

```
stream disconnected - retrying sampling request (1/5 in 182ms)...
```

统计方法：

```python
rows = cur.execute(
    "SELECT feedback_log_body FROM logs WHERE target LIKE '%responses_retry%'").fetchall()
```

多数情况能自愈；若重试耗尽（出现 `5/5`）才会真正失败，重发即可。

### 常见成因

- **代理节点不稳**（长连接被节点重置）——模型流量若走代理，这是首要嫌疑
- 本地网络/运营商链路抖动
- 短时间内高频断连通常集中在某些时段，可据此判断

### 处置

- 换个代理节点（最有效）
- 频繁发生时，做**隔离测试**：临时去掉代理配置直连，即可判断是代理还是本地链路

---

## 6. 数据安全：会话存在哪、会不会丢

### 会话存储结构

```
%USERPROFILE%\.codex\
├─ sessions\<年>\<月>\<日>\rollout-<时间>-<线程ID>.jsonl   ← 会话正文（追加式）
├─ archived_sessions\rollout-*.jsonl                       ← 已归档会话
├─ session_index.jsonl                                     ← 线程名索引
├─ sqlite\codex-dev.db                                     ← 应用的线程目录索引
├─ config.toml / auth.json / .env                          ← 配置与登录态
└─ thread-writer-locks\<线程ID>.lock                       ← 写入锁
```

**关键结论**：

- **一条会话 = 一个 `.jsonl` 文件**，而且是**只追加、不新建**的。
  一条会话跨越几个月，也始终是同一个文件。
- **桌面版与终端 CLI 共用同一份存储**。终端里继续过的对话，
  之后桌面版打开同一条会话时会**完整继承**。
- `%USERPROFILE%\.codex` **不在应用包内** → 卸载/重装应用**不会**动它。

### 安全操作原则

| 操作 | 是否安全 |
| --- | --- |
| 卸载/重装应用 | ✅ 安全（`.codex` 不受影响） |
| 备份整个 `.codex` 目录 | ✅ 推荐，重装前必做 |
| 删除/移动 `config.toml` | ⚠️ 会重置个性化设置（设置可再生） |
| 删除 `sqlite\codex-dev.db` | ⚠️ 会重建索引（索引可再生） |
| **删除 `sessions\`** | ❌ **绝不要做，那是会话正文** |

### 备份命令

```powershell
$bk = "$env:USERPROFILE\Documents\codex-backup-$(Get-Date -Format yyyy-MM-dd)"
robocopy "$env:USERPROFILE\.codex" "$bk\.codex" /E /R:1 /W:1
```

---

## 7. 桌面版崩了怎么临时继续干活

桌面版打不开时，**终端 CLI 通常仍然可用**，而且**共用同一份会话记录**：

```powershell
# 列出所有目录下的会话
codex resume --all

# 直接继续最近一条
codex resume --last

# 按 ID 或名字打开
codex resume <线程ID或名字>
```

> ⚠️ `codex resume` **默认只列当前目录下的会话**，
> 不加 `--all` 会显示 `no sessions yet`——这是最容易踩的坑。

其他相关命令：

| 命令 | 用途 |
| --- | --- |
| `codex fork` | 从某个会话分叉出新分支（保留历史） |
| `codex archive <会话>` / `unarchive` | 归档 / 取消归档 |
| `codex migrate-rollouts` | 检查（加 `--apply` 执行）旧会话迁移到分页历史 |
| `codex doctor` | 诊断本地安装、配置、认证与运行时健康 |

> ⚠️ 桌面版打开某条会话时会持有写锁
> (`~/.codex/thread-writer-locks/<线程ID>.lock`)。
> **不要终端和桌面版同时写同一条会话。**

---

## 8. 排查工具箱：日志位置与关键词

### 应用（Electron）日志

`%LOCALAPPDATA%\Codex\Logs\<年>\<月>\<日>\codex-desktop-*-t0-*.log`

| 关键词 | 含义 |
| --- | --- |
| `bundled_executable_relocation_failed` | 运行时搬迁失败（见 §1） |
| `source=missing` | 某组件未就位（含 `component=`） |
| `source=copied` | 该组件已就位 ✅ |
| `windows_core_runtime_component_resolved` | 组件解析结果 |
| `mcp_server_startup_status_updated` | MCP 服务状态（`starting` / `ready`） |

### CLI 日志数据库

`%USERPROFILE%\.codex\logs_2.sqlite`，表 `logs`
（列：`ts, level, target, feedback_log_body, ...`）

| 关键词 | 含义 |
| --- | --- |
| `TurnInput` | **发送请求到达后端**（判断"到底发出去没有"的关键） |
| `task_started` / `task_complete` | 回合开始 / 正常结束 |
| `turn_aborted` | 回合被中止 |
| `responses_retry` | 连接断开重试（§5） |
| `stream disconnected` | 流中断 |
| `source=missing` / `relocation` | 运行时问题 |

> 该库常达数百 MB，**只读方式打开**并加 `LIMIT`，不要全表扫描。

```python
con = sqlite3.connect(f"file:{p}?mode=ro", uri=True)   # 只读，避免干扰应用
```

---

## 9. 经验教训（最重要的一节）

### 1. 不要用"外挂式"绕过手段

`CODEX_CLI_PATH`、手工 junction、一键修复脚本……这类手段看着能救急，
实际上都是**给应用增加一条它没设计过的代码路径**，会产生新的、更难诊断的故障。
**优先让应用用它自己部署的东西。**

### 2. 改动前先备份，并且用"移动"而不是"删除"

本记录里所有修复都用 `Rename-Item` 改名，而不是删除。
出问题时能**一条命令还原**。

### 3. 不要用强制结束（Kill）来关应用

应用把线程索引写在 SQLite 里。**在它写入过程中被硬杀，索引可能损坏**
（表现：侧边栏会话列表变空 / 只剩几条）。
先试**正常关闭**（发窗口关闭信号），实在不行再强杀。

### 4. 重装应用前，先确认你能装回来

Store/MSIX 应用**不能用 `winget` 装回来**（`winget` 里的 `OpenAI.Codex`
是**同名的终端 CLI，不是桌面应用**）。
必须能从 Microsoft Store 手动安装。**装不回来就是灾难。**

### 5. 先看"请求有没有发出去"，再查网络

判断发送类问题，**第一步永远是查后端有没有收到请求**（§2 的方法）。
这一条能立刻把问题切成"客户端"还是"服务端"两半，
避免在错误的层面浪费时间。

### 6. 区分"看起来坏了"和"真的坏了"

发送按钮"变灰"有两种完全不同的含义：

- 输入框**为空**时变灰 → 正常（禁用态）
- 输入框**有内容**时仍变灰 → 异常

先确认这一点再往下查。

---

## 附：版本对照（记录时）

| 组件 | 版本 |
| --- | --- |
| 应用包 | `26.915.3509.0` → `26.915.4065.0` |
| 应用内部标识 | `26.915.31029` → `26.915.31945` |
| 随附 CLI | `codex-cli 0.155.0-alpha.9` → `0.155.0-alpha.9.2` |

> 应用自带 CLI 的版本会随每次更新而变。
> 验证方式：对比 `%LOCALAPPDATA%\OpenAI\Codex\bin\<哈希>\codex.exe`
> 与应用包内 `app\resources\codex.exe` 的**文件大小是否一致**——
> 一致即为同一个二进制。

---

## 10. 补充案例：如何判定"这是客户端缺陷，不是我环境的问题"

本节记录一次**最终没能靠本地操作修复**的故障，以及**如何尽早判断"该找官方修"**，
避免在无望的方向上继续消耗时间。

### 症状（本次见到的最终形态）

| 场景 | 表现 |
| --- | --- |
| **全新对话** | **第 1 条能发，第 2 条发不出**（按钮变灰） |
| **任何已有对话** | **一律发不出**（按钮灰） |
| 服务端日志 | 回合 `task_complete` **正常结束** |
| 应用日志 | **零错误** |
| 关闭应用 | 卡死，弹出大量"无法终止"，需反复确认 |

> 注意"新对话能发第一条"这个细节很容易被忽略：
> 它说明**收发通路本身是通的**，坏的是"一轮结束后状态复位"这一环。

### 逐项排除清单（全部实测）

| 怀疑项 | 排除依据 |
| --- | --- |
| 上下文超限 | 短对话同样失败 |
| 运行时缺失 | 重装后应用**自己**完成部署：`relocation 失败 0`、`source=missing 0` |
| CLI 版本不匹配 | CLI 二进制与应用包内**字节一致** |
| `config.toml` 陈旧 | 删掉让它重建，无效 |
| 模型缓存 | 7 个模型齐全，缓存刚刷新过 |
| 写锁残留 | 锁文件未被占用 |
| 线程迁移卡住 | 修正迁移标记后，无效 |
| 标签页恢复卡住 | 关掉全部标签页，无效 |
| 归档状态 | 受影响的对话在"活动"目录 |
| 工作目录 / 可写根 | 全部存在 |
| 界面缓存过期 | `Ctrl+R` 强制重载，无效 |
| **杀毒软件** | **整体退出杀软后问题依旧 → 彻底排除** |
| 内存不足 | 64 GB 总量、可用 38 GB |

### 系统级证据（判定"应用自己坏了"的关键）

这类证据**不在应用自己的日志里**，需要查系统事件日志：

**① 应用是否在挂起（无响应）**

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Application'; ProviderName='Windows Error Reporting'
    StartTime=(Get-Date).AddDays(-3)
} -ErrorAction SilentlyContinue |
  Where-Object { $_.Message -match 'OpenAI.Codex' } |
  Select-Object TimeCreated, @{n='Type';e={
      if ($_.Message -match 'MoAppHang')      { '挂起 Hang' }
      elseif ($_.Message -match 'MoAppCrash') { '崩溃 Crash' }
      else                                    { '其它' }
  }}
```

实测结果：**3 天内 6 次 `MoAppHang`（应用无响应）**。

**② 后端进程是否异常退出**

```powershell
$logs = Get-ChildItem "$env:LOCALAPPDATA\Codex\Logs" -Recurse -Filter "*.log"
$logs | Select-String -Pattern 'Codex CLI process exited' |
  ForEach-Object { $_.Line } | Sort-Object -Unique
```

实测结果：

- `Codex CLI process exited classifiedAsExpected=false code=1 signal=null`
- 而且**该进程退出前的最后一条日志是一条无害的插件警告**
  → **没有任何崩溃原因**，这是"被异常终止"而非"自己崩溃"的典型特征
  （自身崩溃通常会留下 panic / backtrace）

**③ 会话状态是否对不上**

```powershell
$logs | Select-String -Pattern 'Conversation state not found|for unknown conversation' |
  Group-Object Pattern | Select-Object Count, Name
```

实测结果：`Conversation state not found` 8 次、`Received turn/started|turn/completed for unknown conversation` 30 次。

### 三条判据：什么时候该停止折腾自己的机器

**三条同时成立**时，继续改配置、重装、清缓存都**不会有结果**：

1. **服务端有完成记录，客户端却没有反应** → 坏在客户端接收侧
2. **排除清单全绿**（尤其：整体退出杀软后依旧）→ 环境无责
3. **有系统级证据**（应用挂起 `MoAppHang` / 后端进程异常退出且无错误信息）→ 应用自身缺陷

此时正确的动作是：**提 issue + 切换到替代通路**，而不是继续本地排查。

### 替代通路（本次采用的方案）

终端 CLI 与桌面版**共用同一份会话记录**（见 §7），所以桌面版坏了也能继续干活：

```powershell
codex resume --all
```

已实测可正常收发消息；且**终端期间产生的对话，桌面版恢复后能完整继承**。

### 已知无效的尝试（列出以免重复浪费时间）

| 尝试 | 结果 |
| --- | --- |
| 重启应用 | 无效 |
| 卸载 + 重装应用 | 无效 |
| 删除 `config.toml` 让它重新生成 | 无效 |
| 更新到最新版本 | 无效 |
| 修正卡住的线程迁移状态 | 无效 |
| 关闭所有标签页 | 无效 |
| `Ctrl+R` 强制重载界面 | 无效 |
| 整体退出杀毒软件 | 无效（**但这条很有价值：一次性排除了杀软**） |

### 一个仍未定性的观察

Electron 日志中反复出现：

```
warning [IpcClient] Received broadcast but no handler is configured
        method=thread-stream-following-changed
```

累计出现 **994 次**。从命名推测它与"某条会话的流是否正在被跟随"有关
（即可能承载"这一轮已结束"的信号），若界面未处理该广播，
就会表现为"永远认为还在生成中"。但 `Ctrl+R` 强制重载界面后问题依旧，
**机制未能确认**，仅作为线索记录。

---

## 11. 补充：一次未能定位的反复故障 —— 方法论与排除记录

本节记录一次**最终没能定位根因**的反复故障。写出来的价值在于：**记录方法与排除项**，
以及一条很关键的教训——它比这次的具体故障更通用。

### 症状

同一台机器上，"发送按钮变灰、消息发不出去"**反复出现**，但：

- **无规律**：有时重启后立刻恢复正常，有时连续重启多次也不好
- **无痕迹**：配置文本、应用日志、后端日志、运行时目录**全部查不出异常**
- **表现会变**：有时"新对话能发、旧对话不能"，有时反过来，有时全都不能

### 这次用的方法：抓状态快照，然后 diff

既然单看某一时刻查不出问题，就**分别在"正常"和"故障"两个状态下抓同一套快照，再做差异对比**。

关键抓取项：

| 项目 | 为什么要抓 |
| --- | --- |
| 版本（应用 / CLI / 二进制大小） | 排除版本错配 |
| 进程列表，**含启动时间** | 能看出是否有**上一次的残留进程** |
| 运行时目录清单 | 排除部署不全 |
| `config.toml` **完整副本** | 供逐行 diff |
| `.codex` 关键文件的大小与修改时间 | 看状态是否在更新 |
| 应用状态摘要（打开标签页数、迁移标记） | 排除界面状态问题 |
| 线程目录行数 | 看索引是否重建 |
| 后端最近 10 分钟：`TurnInput` 次数 / ERROR 数 / app-server 请求分布 | 判断请求有没有发出去、后端是否在忙 |
| 应用日志累计事件计数 | 一次性看清历史异常总量 |

> 抓取脚本本身很简单（读文件 + 查日志库），要点是**每次抓同一套字段**，否则没法对比。

### ⚠️ 最重要的一条教训：不要"重启之后再查"

这次绕了非常多弯，根本原因是：

> **几乎每一次排查，应用都已经被重启过了。**

故障现场（进程、连接、内存中的状态）**已经消失**，看到的永远是"新起的干净进程"，
所以怎么查都是"一切正常"。

**正确顺序是：故障发生 → 保持原状 → 先抓快照 → 再考虑重启。**

这也是前面几节反复强调"先看请求有没有发出去"的同一道理：
**证据是有时效的，现场一旦破坏就再也取不到了。**

### 已排除的怀疑项（逐项实测）

| 怀疑项 | 排除依据 |
| --- | --- |
| 运行时部署不全 | `relocation 失败 = 0`、`source=missing = 0` |
| CLI 版本错配 | 运行中的二进制与应用包内**字节一致** |
| 配置文件内容 | 两份配置**逐行 diff 只差"项目列表"与随机管道名**，却一份能用一份不能用 |
| 界面语言设为中文 | 曾疑似主因（加上→坏、去掉→好），**复测被推翻**（去掉后未恢复） |
| 模型缓存 | 模型列表齐全且刚刷新过 |
| 写锁残留 | 锁文件存在但**未被任何进程持有** |
| 会话被归档 | 受影响的会话在"活动"目录 |
| 线程目录索引行数少 | 只有 14 行、且数据库一天未更新时，应用**依然工作正常** |
| 线程迁移标记未完成 | 在**正常工作**的状态下该标记同样是 `未完成` |
| 打开标签页过多 | 全部关闭后问题依旧 |
| 工作目录 / 可写根 | 全部存在 |
| 界面缓存过期 | 清空缓存并重建后，问题依旧 |
| `Ctrl+R` 强制重载界面 | 无效 |
| **杀毒软件** | **整体退出后问题依旧** |
| 内存不足 | 64 GB 总量、可用 38 GB |

### 观察到的现象（可能相关，但未定性）

**1. 启动后的"线程恢复风暴"**

每次启动，应用都会对已知会话发起 `thread/resume`。实测一次启动约产生
**22~24 次 `thread/resume` + 16~36 次 `model/list`**，持续 **1~2 分钟**。

会话数量越多，这段时间越长（本次环境：**75 条活动会话 + 112 条归档会话**）。

**在这段时间内，发送按钮可能处于不可用状态。**
所以"刚启动就测"和"启动几分钟后测"，结果可能完全不同——
这很可能是"有时好、有时坏"的部分来源。

> 建议：会话积累很多时，把不再需要的旧会话归档，可缩短启动恢复时间。

**2. 可能出现两个实例重叠**

快照里见过"同一批进程存在**两组不同的启动时间**"，说明上一次关闭**没有退干净**。
此时新旧实例可能同时操作同一份会话与运行时。

> 建议：关闭后**确认进程已全部退出**，再重新启动。

**3. 系统级证据：`MoAppHang`**

Windows 错误报告会记录应用"无响应"（本次环境 3 天内 **6 次**）。
查法见 §10。这既是"应用卡死"的客观证据，也是提 issue 的好材料。

**4. 后端进程可能无声退出**

见过 `Codex CLI process exited ... code=1`，且**退出前最后一条日志是无害的插件警告**
（没有任何崩溃原因），同时伴随"会话状态丢失"。

### 一个被证伪的假设（记录以免他人重走）

一度高度怀疑"**把界面语言设为中文会导致发送失效**"，因为当时有一次看起来很干净的对照：

| 操作 | 结果 |
| --- | --- |
| 加入 `[desktop] localeOverride = "zh-CN"` | 发送失效 ❌ |
| 移除该行 | 恢复正常 ✅ |

**但复测推翻了它**：移除该行后问题**并未恢复**；而两份配置逐行 diff 后
**只差"项目列表条数"和每次启动随机生成的管道名**，一份能用、一份不能用。

**结论：触发条件不在配置文件里。** 记录此条，是因为它是本次排查中**最大的弯路**——
基于一次对照就下结论，是非常不可靠的。

### 结论

这次**未能定位根因**。诚实记录如下：

- 触发条件**不留在配置、日志或文件系统的痕迹里**
- 可靠的恢复手段：**重启应用**（有时需要多次，或配合让配置重新生成）
- **替代通路**：终端 CLI 与桌面版**共用同一份会话记录**，桌面版异常时可照常工作（见 §7）
- 若要继续追查，**必须抓故障现场快照**（见本节的快照方法）

---

## 许可

本记录为个人排查经验整理，可自由参考、转载。
文中不包含任何个人身份信息。
