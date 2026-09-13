---
name: windbot-duel-review
description: Runs one local srvpro bot-vs-bot duel with Debug WindBot, captures bot A's console to log.log, reconstructs the duel, and reports numbered play problems with suggested executor fixes. Use when the user asks to 开一把, 对战检查, 对战复盘, review bot A's plays, or run two specified WindBot decks against each other on local srvpro. Do not use when the user is only editing an executor and did not ask to duel.
---

# WindBot 双 bot 对战复盘

每次调用只做用户**当前这句话**要求的事。普通改 executor 的对话禁止自行开打。

术语：**A** = 被复盘的 bot（打日志）；**B** = 对手。`Deck=` 取 `[Deck("Name", ...)]` 的第一个参数（如 `Elfnote`、`Labrynth`），不是 `.ydk` 文件名。

## 门禁

- 没点名的 P/L 不动代码，包括看起来像铁证的功能性错误。
- 第一次对战、以及未批准 L 的对战：**零代码改动**（含日志）。
- 禁止改 `GameAI` / `GameBehavior` / 其它公共日志。
- 禁止启动或重启 srvpro。禁止改 `BotWrapper/bot.conf`、Dialogs。
- 任何代码改动（修 P、加 L、修编译错误）之后：**只 Debug 编译并汇报，绝对不对战。** 即使同一句话里写了「再打 / 再来一把 / 开一把」，也先改完、汇报、停下。
- 新的一局只在用户**看过改动之后**另发一句确认时才开（如「确认，再来一把」「开一把」）。上一轮对话里的「再打」不算确认。

## 当前这句话在做什么

先看这句话是不是在确认开打。只有**没有夹带改代码要求**、且明确要开新局时，才走对战。

1. **确认开打**（本句不要求改 P/L，且明确要开新局：如「再来一把」「开一把」，或改完代码后的「确认」）：走「对战流程」→ 交 `P` + `L` 列表 → 停。缺 Deck 就先问，不要猜。沿用本会话已指定的 A/B，除非用户改指定。「确认 P2 / 这个问题属实」只表示认可条目，不是开打。
2. **改 P**：只改点名的条目和用户给的方向 → Debug 编译 → 汇报动了什么 → **停，等用户确认再开打。**
3. **加 L**：只追加批准的日志点（零行为变化）→ Debug 编译 → 汇报加了哪些 L → **停，等用户确认再开打。**
4. 一句里同时「改 P/L」和「再打」：只做改代码 + 编译 + 汇报，**不要接着开打。**

`HostInfo` 用户可覆盖；默认 `windbot-review`。

## 对战流程

复制并勾选：

```
- [ ] 7911 在听
- [ ] Debug Any CPU 编过
- [ ] bin/Debug/WindBot.exe 存在
- [ ] 清空并准备仓库根目录 log.log
- [ ] 启动 A（Debug=True，重定向到 log.log）
- [ ] 启动 B（Debug=False）
- [ ] 结束或 10 分钟超时
- [ ] 读 log.log 复盘
- [ ] 只输出 P/L 列表
```

### 1. 检查 srvpro

本机 `127.0.0.1:7911` 必须已在听。不在听就停，告诉用户先起 srvpro。不要改端口，不要起服务器。

```powershell
Get-NetTCPConnection -LocalPort 7911 -State Listen -ErrorAction SilentlyContinue
```

### 2. 编译

读 `AGENTS.local.md` 第 7 节，用那里的 MSBuild **完整路径**（不要 `msbuild`、不要 `dotnet build`、不要 vswhere）。工作目录为仓库根。

失败：贴错误，对战一步不做。成功后确认 `bin/Debug/WindBot.exe` 存在。

### 3. 启动双 bot

工作目录必须是 `bin/Debug`（`cards.cdb`、`Decks`、`Dialogs` 在这里）。`log.log` 写在**仓库根目录**，每次覆盖。

Deck 含空格时给 `Deck=` 加引号。A/B 名称固定 `ReviewA` / `ReviewB`。A 必须 `Debug=True Chat=False`；B 必须 `Debug=False Chat=False`。

在**同一** `cmd` 里先 `chcp 65001`，再启动 A，避免中文卡名写入 `log.log` 后乱码：

```powershell
$repo = (Get-Location).Path  # 仓库根
$bin = Join-Path $repo "bin/Debug"
$exe = Join-Path $bin "WindBot.exe"
$log = Join-Path $repo "log.log"
$hostInfo = "windbot-review"  # 或用户给的值
$deckA = "Elfnote"            # 用户指定
$deckB = "Labrynth"           # 用户指定

if (Test-Path $log) { Remove-Item $log -Force }

$argA = "Name=ReviewA Deck=$deckA Host=127.0.0.1 Port=7911 HostInfo=$hostInfo Debug=True Chat=False"

$pA = Start-Process -FilePath "cmd.exe" -ArgumentList "/c","chcp 65001 >nul & `"$exe`" $argA > `"$log`" 2>&1" -WorkingDirectory $bin -PassThru -WindowStyle Hidden
Start-Sleep -Seconds 2
$pB = Start-Process -FilePath $exe -ArgumentList @(
    "Name=ReviewB",
    "Deck=$deckB",
    "Host=127.0.0.1",
    "Port=7911",
    "HostInfo=$hostInfo",
    "Debug=False",
    "Chat=False"
) -WorkingDirectory $bin -PassThru -WindowStyle Hidden
```

`Deck=` 必须是**一个**参数。名称含空格时写成 `Deck=Level VIII` 这一整个元素，不要拆开。

### 4. 等待结束

- 日志出现 `Duel finished against`（Debug 编译下 `OnWin`），或 A/B 进程都退出 → 正常结束。
- 从启动 A 起 **10 分钟**仍未结束：`Stop-Process` 杀掉 A 和 B，按**不完整对局**复盘，并在报告开头标明超时。
- 不要去读「终端面板」当日志；只认 `log.log` 文件。

等日志时用轮询/Await，不要 `sleep` 死等满 10 分钟。

### 5. 复盘依据

只根据 A 的 `log.log` 加公开规则知识，**不要假装看到了 executor 命中或合法候选**（框架默认不打这些）。

`Debug=True` 可见（`Logger.WriteLine`，玩家 0=A，1=对手）：

- `(0 draw N card)` / `(Go to Phase)`
- 准备阶段：`Bot Hand` / `Bot Spell` / `Bot Monster` 快照
- LP 伤害/回复、移动、appear、overlay/deattach、攻击、become target、Confirm

另：`Logger.DebugWriteLine` / 牌组 `ElfnoteLog` 只在 Debug 编译进日志。胜负行：`Duel finished against …, result: Win|Lose|Draw`。

对照材料（按需读，不要全文件塞进上下文）：

- `Game/AI/Decks/{Deck}.md`（有则读；耀圣是 `Elfnote.md`，日志里是中文全名，简称以该文档为准）
- A 的 `*Executor.cs`：`AddExecutor` 顺序、相关条件、已有日志
- 效果文本/字段：`AGENTS.local.md` 里的 zh-CN `cards.cdb` / `strings.conf`

## 判定规则

三条都看：**功能性**、**路线**、**微调**。没把握标不确定。同型问题合并成一条并举例，不要把一类失误拆成二十条。

| 结论 | 何时 |
| --- | --- |
| 功能性·确定 | 非法选择、卡死、自杀支付 LP、必应没应且公开信息足够、选了日志能证明的错误公开目标 |
| 路线·确定 | 与 `{Deck}.md` 或 executor 写明的轴冲突；或没有文档，但当场公开信息已足够说明把轴打崩 |
| 微调·确定 | 文档/代码意图支持，且日志足够 |
| 不确定 | 缺候选/命中日志、与文档冲突的纯牌理、没有 md 时的一般「更优」猜测 |

文档/代码意图与 agent 牌理冲突：标不确定，写「文档说 X」。不允许用牌理压文档。没有 md 时，路线/微调默认不确定，除非公开信息已足够说明打崩或功能性错误。

## 报告格式

先给一行结果（胜负或超时/崩溃）和 A/B 的 Deck。然后只出列表，不要长篇战报。每条 P **必须**带建议；建议是给用户拍板用的，**不要在报告阶段改代码**。

```markdown
结果：ReviewA（{DeckA}）vs ReviewB（{DeckB}）— Win|Lose|Draw|超时|崩溃

### P1 — 功能性|路线|微调 — 确定|不确定
- 何时：第几回合 / 谁的回合 / 阶段 / 连锁（能定多少写多少）
- 现象：A 做了什么、为什么算问题
- 日志：摘 1–5 行原文
- 涉及：卡名；能定位则到 executor 函数名（不要假装定位到行）
- 文档：有则引用 md 小节或代码意图；与牌理冲突时写「文档说 X」
- 建议：改哪个函数、改什么意图（条件 / 选卡 / AddExecutor 顺序）；不确定时写「先加 Lx」或标「需先加日志」；2–4 句，不要贴整段补丁

### L1
- 文件：Game/AI/Decks/{A}Executor.cs
- 位置：函数名 / 现有日志附近
- 打什么：一条 Debug 日志的内容（用已有 `ElfnoteLog` 或同款 `Logger.DebugWriteLine("[Deck] …")`）
- 为何：缺了它无法把某条 P 从不确定推进到确定
```

建议写法：对准 executor 里已有逻辑，说明「现在会怎样、应改成怎样」；保留原函数其它行为。不要发明新公共 API。文档/代码意图与牌理冲突时，建议跟文档走，并写「文档说 X」。不确定条目仍给最可能的修法，但开头标明依赖哪条 L。

无问题也要明说「这局没看出确定问题」，仍可列 L。停，不加赛。

用户点名时用编号：`改 P2`（按该条建议改）、`改 P2、P5，方向是……`（覆盖建议）或 `加 L1`。改完后用「确认，再来一把」开新局。

## 点名之后

**修 P：** 只动点名条目。用户没另给方向时，按该条报告里的「建议」改；给了「方向是……」则以用户方向为准。保留原函数逻辑，只改指定问题。注释语言跟 `AGENTS.local.md`（本项目用英文注释）。改完用同一套 Debug Any CPU 编译；失败则只修编译错误，不顺手改牌理。汇报：改了哪些 P、动了哪些函数、哪些 P/L 没动。末尾写「等你确认后再开下一局」。**到此结束，不要启动 bot。**

**加 L：** 只追加日志，禁止改条件、返回值、选卡、`AddExecutor` 顺序。有 `ElfnoteLog` 就用它。Diff 若夹带行为变化，停下来问用户。编过并汇报后同样**结束，不要启动 bot。** 末尾同样写「等你确认后再开下一局」。
