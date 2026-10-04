# 架构

> 快照：0.1.7（2026-10-02）。行号会漂，指路以"文件 + 符号名"为准。

## 1. DSH 插件形态与挂载

whale_craft 是**标准 DSH 插件**（两半端）：

| 半端 | 文件 | 挂载方式 |
| --- | --- | --- |
| host（服务端） | `index.js` + `src/` | `package.json` → `dsh.bundle.patch: ./cordis.patch.yml`；profile 的 `dsh.profile.bundles` 里列包名即生效 |
| client（浏览器） | `client.js` | `package.json` → `exports["./client"]`；`dsh.client.platform: "web"` |

- `cordis.patch.yml` 内容就一件事：`insert: - id: whale_craft / name: whale_craft`（2026-10-02 起连 config 块也去掉了——`autoConnect` 是 0.1 时代的死配置）。
- `index.js` 导出 `name='whale_craft'`、`inject=['webServer','tools']`、`Config`（schemastery schema；**宿主拿不到 schemastery 时不导出**，`apply` 自己兜默认值——见 [src-modules.md §16](src-modules.md)）、`apply(ctx, config)`。
  - 两个宿主包（`@deepseek-ai/dsh-tools` / `schemastery`）都是 **optional 且可缺省**：顶层不做静态 import（那会让干净安装/官方 desktop 在链接期就 `failed to import`，issue #5）；`defineTool` 走 `src/tool-def.mjs` 的宿主优先/内置兜底。
- **host 半端不热重载**（装/改后要重启 DSH）；**client bundle 是 HMR 热更**的。
- 浏览器端入口协议：`window.__ModuleLoader__.load({ id:'whale_craft', factory(require){...} })`，`apply(ctx)` 往 `slots` 注册三个插槽。

## 2. 运行时组件图

```
DSH host 进程
└─ index.js apply(ctx, config)
   ├─ PluginConfig          $DSH_HOME/whale_craft/config.json（全局，热生效）
   ├─ wsconfig（src）        <记忆根>/config.json 按工作区的配置（提示词三开关 + 版本标记；迁移/建档）
   ├─ protected（src）       RULES.md / AGENTS.md / config.json 的"可读不可写"统一判定
   ├─ AccountStore          $DSH_HOME/whale_craft/accounts.json + 宿主凭据服务（密码/token）
   ├─ McRegistry            agentId → McSession
   │   └─ McSession         McBot + events[]（≤200 条）+ Watchdog + selectedAccount
   │       └─ McBot         src/core.mjs：mineflayer 实例（一个游戏角色）
   ├─ 工具注册（29 个）      mc_* 25 / mc_kit_* 3 / mc_admin_* 1，全部经 asTool()
   ├─ HTTP（webServer）
   │   ├─ /api/mc/*                     状态/停止/设置（账户/配置/提示词/分享/preset 名单）
   │   └─ /api/whale-craft/express/*    发布区文件（仅 online 模式注册）
   ├─ agent/pre-step 监听   提示词注入（改写 decision.messages）+ MC 模式策略对账
   ├─ ensureMcPreset        启动自举/自检「MC模式」preset
   ├─ installArchiveGuard   包装 workspaceRegistry.archiveSession
   └─ extensions/*.mjs      自动加载（apply(api)）

浏览器
└─ client.js
   ├─ McStatusBar           会话标题条状态 + 「强制停止」
   ├─ McSettingsModal       账户 / 指令白名单 / 提示词 / 文件分享 四个标签页
   └─ McSettingsDockEntry   新会话页（hero 区）的入口按钮
```

## 3. 会话与实例模型（index.js）

- `McSession`：`agentId / config / bot(McBot, instanceId=agentId) / mode(standby|active|sleep) / events / maxEvents=200 / watchdog / selectedAccount / waitInterruptedAt / waitInterruptReason`。
- `McRegistry`：`getOrCreate(agent)` / `peek(id)`（**只读不建**，HTTP 用）/ `destroy(id)` / `destroyAll()`（插件卸载时 `ctx.effect` 里调）。`destroy()` 顺序：disarm 看门狗 → 优雅退游戏 → 清事件。
- **事件队列唯一写入方**是 `McSession.ensureWired`（懒绑 bot 事件 → `#pushEvent`）：`spawn / death / reconnect / offline / chat / system / damage`。超 200 条裁旧。`drainEvents()` 供 `mc_events` 读。
- `interruptWait(reason)`：置 `waitInterruptedAt` 标记，`src/wait.mjs` 的等待循环（250ms 轮询）看到就立刻收工 —— 这是"唤醒不被等待堵住"的一半（另一半在 Watchdog `#inject`）。
- **状态量语义**（`modeView()` / `mc_status` / `GET /api/mc/status`）：

| 量 | 含义 |
| --- | --- |
| `online` | `bot.entity` 存在 **且** 底层连接没结束（`_client.ended !== true`）——防"幽灵在线" |
| `ghost` | 连接已结束但 `bot.entity` 残留 —— 明确告知"要回游戏请重新 mc_connect" |
| `reconnecting` | 正在两次自动重连尝试之间的窗口 |
| `reconnectPending` | 掉线置位；**真重连成功或手动 mc_connect 成功才清** —— 覆盖"正在尝试连接"那几十秒，免得状态条退回"未上线" |
| `active`（HTTP） | `online || reconnecting || reconnectPending || watchdog.armed` —— 决定浏览器状态条显不显示 |

- **单实例锁**：连服期间 `$DSH_HOME/whale_craft/.instance.<会话>.json`（`lockDir = stateDir`，不写包目录）。

## 4. 游戏侧核心：`McBot`（src/core.mjs）

详见 [src-modules.md §1](src-modules.md)。要点：

- `connect({host, port, subserver, version, auth, onAuth})`：`auth` 是唯一凭据入口（`{mode:'offline'|'yggdrasil',...}`，绝不出现在返回值）；版本默认自动探测；**顶号**（"already connected"且建连 <9s）最多重试 4 次；失败**保留原连接**。
- 自动重连：延迟 5s 起、失败翻倍上限 60s、成功复位；`offline` 事件在 `b.on('end')` 里 emit，payload `{sub, reason, willReconnect, at}`。
- 事件：`spawn / offline / reconnect / death / damage / chat / system` + 观察器 `playerJoin / playerLeave / teleport / pushed / pickup`（后三类默认只留档，见看门狗矩阵）。`chat` 的识别覆盖 signed（`player_chat` 包）、unsigned、**以及被服务端塞进 system 位置的玩家聊天**（`#playerChatFrom` 兜底正则 `/^\s*<who>\s*text$/`）。
- **按键上报层已移除（2026-10-02）**：原先为 26.2 加的 `player_input` 兼容层（含"先查后发"护栏）整体撤掉——上游还连不了 26.2（mineflayer 4.39.0 只到 26.1；minecraft-data 3.117.0 只有元数据、无数据目录）。将来重建的注意事项见 [history.md](history.md)（D1 / F10）。
- 不变量：所有可能永不 settle 的 await 经 `#t()`（`withTimeout`+`raceAbort` 包裹）——超时/中断会松掉全部控制位；`placeBlock`/`breakBlock` **必须复验 `blockAt` 才报成功**（报假成功是历史教训）。

## 5. 事件 → 唤醒链路（"该不该醒" vs "发生过什么"）

两条通道，分工明确：

| 通道 | 提供者 | 内容 |
| --- | --- | --- |
| **拉**（发生过什么） | `mc_events` 读 `sess.events` | 聊天、系统消息、受伤、上线/死亡/重连/断线。⚠️ 被传送/捡物/上下线只进**看门狗留档**（`mc_watch {action:"log"}`），不在这个队列 |
| **推**（该不该醒） | Watchdog → 插件提示行注入 | 唤醒矩阵命中 → 合并 → 限流 → 注入 |

完整注入流程（0.1.7 定格的顺序，每一步都有事故背书）：

1. 事件命中 `wakeOn` 某一项（或心跳到点 / 45s 补提醒）→ 进 `pending`；
2. 攒 `observeWindowMs`（2s）把连珠炮合并，`#flush` 时过 `maxWakePerMinute`（6 次/分）限流；
3. 注入前**先 `sess.interruptWait(kind)`** —— 让堵在 `mc_events{waitSec}` 里的等待立刻返回（工具一返回 → step 结束 → 唤醒当场投递）；
4. 优先 `agent.steer(userMessage(...))`（运行中 = 下一步插话；空闲 = 起一轮 = 唤醒）；`steer` 不可用时兜底 `sessionController.prompt({mode:'steer'}, promptSignal)` —— **promptSignal 必传**（`sessionController.prompt` 是 @Remote 签名，不传会 `throwIfAborted` 炸）；
5. 注入被 `gate` 检查：**不在 MC 模式只记账不注入**（drop）。

- 看门狗挂宿主 job（`jobs.start({kind:'mc-watch', label:'MC 看门狗（整局存活）', owner:agent, run})`）：job 存活期=整局游戏；`disarm` 幂等结算（`#settleJob`），宿主 kill job 时 `cancel → disarm(fromJob:true)`。
- 断线矩阵：`offline` 且 `willReconnect` → 唤醒"正在自动重连"（不关看门狗）；`willReconnect=false` → **自动关看门狗** + 关闭通知（"你已经不在 MC 里了"）。
- 看门狗配置（`WATCH_DEFAULTS`，`mc_config` 可改）见 [src-modules.md §3](src-modules.md)。

## 6. 提示词注入体系（唯一通道 = 插件提示行）

**本插件不往系统提示词里塞任何东西**。注入 = 往会话投"插件提示行"，共 4 条内容：

| 顺序 | 内容 | 开关（默认；前两个**按工作区**，存 `<工作区>/.whale-craft/config.json`） |
| --- | --- | --- |
| 1 | 工作区根 `AGENTS.md`（宿主原生文件，插件再补一份） | `injectWorkspaceAgentsMd`（关） |
| 2 | `.whale-craft/RULES.md`：行事准则（称呼/记忆/看门狗/登服/聊天/建筑/硬规矩） | `injectWhaleCraftAgentsMd`（开） |
| 3 | **版本硬提示词**（硬编码随版本发布：哪些工具不成熟、怎么把文件给用户看） | 无开关 |
| 4 | 记忆总索引：`.whale-craft/README.md` + 自动目录树 | 无开关 |

- **挂载点**：`agent/pre-step` waterfall（放行前**必须 `next()`**），把提示行改写进 `decision.messages` 的**本步最前**（不能塞 `inbox.nextStep` —— 宿主 `preStep()` 先 `inbox.claim()` 再跑瀑布，塞队列会晚一步）。
- **去重三层**：① 台账 `noticeLedger`（指纹 `preset|PLUGIN_VERSION`）；② 会话日志回读 `deliveredRelsFor`（找 `source.plugin==='whale_craft' && form==='notice'` 的历史消息）；③ `inbox` 队列同名检查。
- **切出 MC 模式**时 `withdrawAgentsMdNotices`：清队列 + 清台账 + 划 `since` 时间线。
- **为什么叫 RULES.md 不叫 AGENTS.md**：DSH 会把 `AGENTS.md`/`CLAUDE.md` 当工作区指令自动注入任何碰过该目录的会话（不受插件开关控制）。改名的效果 = 注入只剩插件这一条通道、且只对 MC 模式生效。老文件自动迁移进 `RULES.md`，原文件改名 `AGENTS.md.bak-<时间>`（内容不丢、宿主不再认）。
- **受保护文件对 AI 只读**（RULES.md / AGENTS.md / config.json；文件工具 guard + `MemoryStore` 写方法两条路，判定统一在 `src/protected.mjs`）。`rulesFollowVersion`（默认开，**按工作区**）靠 config.json 的 `rulesVersion` 字段在新版本时整体替换；首次见到无记录只记版本不覆盖。旧工作区单独的 `.rules-version` 标记会在"备好记忆目录"时机迁入 config.json 并删除（迁移见 `src/wsconfig.mjs`）。
- **消息构造**走 `src/user-message.mjs`：宿主 `@deepseek-ai/dsh-llm` 的 `createUserMessage` 优先、拿不到用自带等价实现 —— 因为该包曾漏进依赖声明，导致"工具都在、提示词全无"（详见 [history.md](history.md)）。

## 7. MC 模式与权限隔离（双保险）

判据 `isMcModeAgent`：`agentPresets.composedPreset` ∈ `mcModePresets`（默认 `['minecraft','whale_craft']`）。

1. **`tools.restrict({allow})`**（无条件白名单）：`mc_*`（按 `hideAdminTools` 去掉 `mc_admin_*`）+ `mc_kit_*` + 文件工具（`read/write/edit/glob/grep/read_image`）+ `present` + `mcMode.allowOtherTools`。宿主的 `pwsh/subagent/workflow/serve_*` 一个都看不见。⚠️ 只能**收窄** —— 不能凭空添加 preset 没挂的工具（宿主报错里 `known global tools:` 可直接解析）；`restrict` 是**黏性**的，切换靠 disposer。
2. **`guard`**（服务端硬拒，`index.js` apply 内注册）：① `mc_admin_*` 硬拒；② 文件类工具参数命中凭据路径（`.credentials`/`credentials.yaml`/`/.dsh/`）、`/secrets/` → 硬拒（读写都不行）；③ **受保护文件**（`src/protected.mjs`：RULES.md / AGENTS.md / config.json）**可读不可写** —— `write|edit` 与记忆工具写动作命中即拒，读类工具放行（判定 = 路径写法命中 或 解析到记忆根后正好是该文件）；④ 文件工具路径必须落在 `<工作区>/.whale-craft/` 内，**空路径也算越界**；⑤ `present` 的文件路径必须在会话工作区内。
3. **无工作区 → 整体拒绝**：会话没选工作区时，**不套隔离、不注入、不建记忆目录**；「MC设置」API 400 并说明原因（`noWorkspaceRefused`）。
4. 模式切换触发点：`agent/created`、`agent/session-start`、`agent-preset/selected` 三个钩子里双向对账（进套用/出撤销）；`liftMcModePolicy` 撤销时要**撤回提示行**（漏撤回曾是"标准模式没 pwsh"的事故）。

## 8. MC 模式 preset 自举（ensureMcPreset）

约束与事实：

- 宿主 preset authoring **只允许整目录复制**（`agentPresets.copy(源, 新id, 显示名)`），调用方不得提供 composition 文本；
- preset id 必须是目录名规则 `^[a-z0-9][a-z0-9-]*$` —— 默认名单里的 `whale_craft` **带下划线、永远不是合法 preset id**，所以自动建的目标 id 取 `mcModePresets` 里第一个合法的（`pickPresetTarget`，默认 = `minecraft`）；源优先 `minimal` → `standard` → `ptc`（`pickPresetSource`）。
- 启动时（`ctx.inject(['agentPresets'])` 后 `void ensureMcPreset().catch(仅记日志)`——**绝不让 rejection 触发宿主 fail-loud exit**）：
  - 若 `mcModePresets` 里**一个都不存在** → 复制官方源建「MC模式」，然后打三个补丁：persona 换一句话（去掉官方 `complete: true` / `includeRuntimeContext: false`）、关掉 persistent-shell、补工具组（`tool-fs` / `tool-jobs` / `present` / `compaction`，加之前用**同步 `svc.roots` 扫目录**探测包是否存在——用 async `list()` 会静默失效，GitHub issue #1 的根因）；
  - 若存在 → `planPresetAction`（src/config.mjs）决策 `leave / meta / rebuild`：只看显示名/简介不对 → 只修显示文本；**组成**是"我们当初复制的那份"而官方源变了（或 `MC_PRESET_SPEC` 规格升版）→ 备份 `<id>.bak-<时间>` 后重新复制；**只要你动过组成就绝不碰**。
- 判定"这份是不是我建的、有没有被改过"靠 preset 目录里的自建标记 `.whale-craft.json`（`createdBy/spec/source/compositionHash`）。
- 每次启动还自检自建 preset 的 persona 键名（老 DSH `text` / 新 DSH `prefix`，`PERSONA_TEXT_KEYS`）。
- 隔离实例用 `WHALE_CRAFT_NO_PRESET_WRITE=1` 禁止写 preset。

## 9. HTTP 面与安全

**信任栅栏**（`isTrustedRequest`，照抄 dsh-serve 的同款）：Host 必须回环或 ∈ `webRuntime.trustedHosts`；`Sec-Fetch-Site: cross-site` 拒；带 Origin 时 host 必须一致。非信任一律 403。两条顶层前缀路由各自独立过栅栏：`/api/mc/*` 与 `/api/whale-craft/express/*`（后者**只在 online 分享模式注册**）。

| 方法 + 路径 | 用途 |
| --- | --- |
| GET `/api/mc/status?sessionId=` | 状态条轮询（`active/online/reconnecting/busy/connection/timeouts/watch`…） |
| GET `/api/mc/sessions` | 会话列表（调试） |
| GET `/api/mc/mode?sessionId=` | preset 判据 + hasWorkspace + 注入诊断（**可重试兜底**用，前端主判据在本地） |
| GET `/api/mc/presets` | preset 名单（**必须在 settingsGate 之前**，否则前端静默兜底） |
| POST `/api/mc/stop` | 强制停止（`cancelTurn: true`） |
| `/api/mc/accounts`（GET/POST/PATCH/DELETE）+ `/accounts/refresh` | 账户 CRUD / 改名 / 探测刷新（`probeBot.authOnly`） |
| `/api/mc/authservers`（POST/DELETE） | 认证服务器增删（`parseAuthlibCard` 解析卡片） |
| `/api/mc/config`（GET/PATCH） | 配置读写（**分流**：全局键 → PluginConfig；提示词三开关 → 该工作区的 config.json） |
| `/api/mc/agents-md`（GET/PUT/DELETE） | RULES.md 读/写/恢复默认 |
| `/api/mc/express`（GET/DELETE） | 分享状态 / 「清除分享数据」 |
| GET/HEAD `/api/whale-craft/express/<工作区uuid>/<相对路径>` | 发布区文件（仅 online 模式） |

- **设置类 API 的错误形态统一 200 + `{ok:false, error, needUserAction?, hint?}`**（前端 `apiFetch` 要求 `payload.ok===true`）。
- **settingsGate**：sessionId → 工作区（或 client cwd）；查不到**不猜**、400 拒绝；顺带 `ensureMemoryRootForCwd`（"点开 MC设置"是仅有的两个建记忆目录时机之一）。
- **express 路由安全**：uuid 是 DSH 工作区注册表的**稳定 id**（查不到就 404，**不退回目录名**）；路径**逐段**白名单拼接（`..`/`.`/空段/段内分隔符/盘符/`~`/控制字符一律拒）→ 拼完 `realpath` 复查仍在发布区内（**符号链接也出不去**）；不列目录；单文件 ≤32MB；svg/html 加 `Content-Security-Policy: sandbox`。
- **base 推导链**（online 模式"获取当前"）：浏览器 `location.origin` → `Origin` 头 → 同源 `Referer` → `X-Forwarded-Proto`+`Host` → `Host`。

## 10. 发布区与文件分享（.out vs .express）

| 目录 | 谁能拿 | 用途 |
| --- | --- | --- |
| `<工作区>/.whale-craft/.out/` | 谁都拿不到 | 默认输出（`mc_map image`、`mc_kit_image` 落盘） |
| `<工作区>/.whale-craft/.express/` | 取决于分享模式 | 发布区（**目录即白名单**，支持子目录） |

- `expressMode: off`（默认）：`mc_kit_express` 恒回 `EXPRESS_OFF_TEXT`（让 AI 把绝对路径给用户），路由不存在（访问即 404）。
- `expressMode: online`：回 `base + /api/whale-craft/express/<uuid>/<rel>` 完整 URL；缺 base 回 `EXPRESS_NEED_BASE_TEXT`（报错不抛异常）。老配置 `local` 一律当 `off`。
- 前端渲染只认**绝对 http(s)** 图片地址 ⇒ 只有 online 的 URL 能内联成图。

## 11. 停止、归档与保护

- **`stopSession` 四步顺序**（强制停止/归档共用）：停 LLM → 优雅退游戏（看门狗 disarm(notify)）→ **只杀 `ownerSession` / `owner===sessionId` 的 job**（无主的不设防，**误杀宿主任务的坑**）→ 再停一次 LLM。HTTP 的 `/api/mc/stop` 默认 `cancelTurn: true`；工具 `mc_stop` 必须 `cancelTurn: false`（否则自我 abort 卡死）。
- **归档保护**：包装 `ctx.workspaceRegistry.archiveSession`——归档一个正在玩 MC 的会话时先踢下线 + 关看门狗 + 清后台任务再放行。⚠️ 这是**接替宿主内部方法**（不是公开扩展点），DSH 升级后可能需跟进；安装必须在 `workspaceRegistry` 就绪之后（`ctx.inject(['workspaceRegistry'], ...)`）。

## 12. 落盘地图

| 东西 | 位置 |
| --- | --- |
| 全局配置 / 账户元数据 / 日志 / 会话锁 | `$DSH_HOME/whale_craft/`（`config.json` / `accounts.json` / `logs/whale-craft.log` / `.instance.<会话>.json`） |
| 凭据（password/token） | 宿主凭据服务 `$DSH_HOME/.credentials.yaml`（key=`whale-craft/<innerID>`；**绝不降级写明文**） |
| 记忆 / 行事准则 / 输出 / 发布区 | `<会话工作区>/.whale-craft/`（`README.md` / `RULES.md` / `config.json` / `.out/` / `.express/`） |
| 按工作区的配置（提示词三开关 + 版本标记） | `<会话工作区>/.whale-craft/config.json`（旧版单独的 `.rules-version` 标记会迁入并删除） |
| 调试实例的一切 | `.dev/home/`（隔离 DSH_HOME，gitignored） |

环境变量阀门：`MC_LOG`（日志路径）、`DSH_HOME`、`WHALE_CRAFT_DIR` / `WHALE_CRAFT_STATE_DIR`（自检/隔离用状态目录）、`WHALE_CRAFT_MEMORY_DIR`（记忆目录，供自检/调试隔离）、`WHALE_CRAFT_NO_PRESET_WRITE`（禁止写 preset）。

## 13. 宿主耦合点 / DSH 升级敏感清单

改 DSH 版本或调试"升级后坏了"时，逐一核对：

1. `agent/pre-step` waterfall **必须 `next()`**（不交棒 = 全崩，0.1.4 P0）。
2. `tools.restrict` 的**黏性**语义与 disposer；`agent-preset/locked`。
3. `workspaceRegistry.archiveSession` 内部方法（归档保护包装点）。
4. `agentPresets.copy` 的 authoring 契约（只允许整目录复制）；`svc.roots` 同步 vs `list()` async（issue #1）。
5. preset persona 键名（`text` → `prefix`）；`complete` / `includeRuntimeContext` 的压制关系。
6. `@deepseek-ai/dsh-llm` 的 `createUserMessage` 可用性（缺失时走自带兜底，但**依赖声明不能少**）。
7. `sessionController.prompt` 是 @Remote（必传 signal）；`agent.steer` 在 step 边界的消费语义。
8. `hosts`/`trustedHosts`、`webServer` 最长前缀路由。
9. jobs 服务的 `owner`/`caller` 语义：**只认会话 id 字符串**（`resolveOwner()` 拿它查 agents 注册表、`assertAccess()` 按 `job.owner.id === caller` 比对）；`stopSession` 只杀自己名下的（无主 job 对所有会话可见，别碰）。
10. 会话消息 `source.kind`：**v4 格式拒绝裸露的 `'plugin'`**，规范值是 `plugin:<插件名>`（= 宿主 v3→v4 迁移的产出，`plugin` 字段随之去掉）；宿主 `createUserMessage` 对传进来的 source **原样透传**、不会替我们修正。

## 14. DSH 插件页（0.2.0+）：名称/描述 与 设置入口

两张面孔：侧栏「插件」管理面板（`dsh-client-ui-plugin-manager`，装/停/配置，渲染图标）与 设置→「内置插件」库存页（只读列表）。名称/描述两张共用同一份数据。

**显示元数据**（宿主 `readPluginMeta`，读自包内文件）：

- 标题/描述 = `locale/<语言 id>.json` 的 `{ "meta": { "title", "description" } }`；回退 `package.json.name` / `description`。`en.json` 是**触发条件**（先解析到它，才会读整个 locale 目录）。
- 语言 id 内置就是 `zh` / `en`（`resolveText` 按小写键查）⇒ 中文文件必须叫 `zh.json`，`zh-CN.json` 永不命中。
- 图标 = `package.json.icon`（包内相对路径；SVG/PNG/JPEG/WebP，≤256 KiB，转 data URL）。
- 🔴 locale 文件必须**逐文件**写进 `exports` 才能被解析到（模式写法 `"./locale/*": "./locale/*.json"` 会把 `en.json` 吞成 `en.json.json`，静默失效——2026-10-02 实测踩过）；漏了 = 静默回退成包名。
- 本仓库现状：`locale/en.json` + `locale/zh.json`（Whale Craft / 鲸鱼工艺），selfcheck 逐文件对账。

**设置入口契约**（浏览器端 client 插槽；完整契约表内嵌在运行时 `dsh-cordis-client-runner`，可搜索查任意插槽）：

- `plugins.bundle.config`（keyed，key = 包名）：bundle 详情页的配置区——契约明确"插件自己的配置"放这里，最贴合。
- `settings.plugins.tab`（list）：在 设置→内置插件 页加一个标签页（库存页 id `'all'` 是样例）。
- `plugins.detail.actions` / `.badge` / `.section`（list）：详情页按钮/徽章/区块；组件收到 `subject`，不是本包时渲染 null。
- `pluginNavigation.openBundle('whale_craft')`：从任何地方跳到插件页并打开指定包（反向入口）。
- 槽组件可用标准 hooks：`useSessions` / `useWorkspaces` / `usePanelInfo` / `useResource` 等。
- ⚠️ 全局设置/插件页**没有会话上下文**——MC 设置（settingsGate 要 sessionId/cwd）接入时先解决工作区来源（待做）。
