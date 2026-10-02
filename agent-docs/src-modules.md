# src/ 模块参考

> 快照：0.1.7。所有模块都是纯 ESM，**只有 config.mjs 引了 express.mjs 一处模块间依赖**，其余全部由 `index.js` 组装。
> 通用规矩：任何可能永不 settle 的 await 套 `withTimeout`/`raceAbort`；日志走 `logLine()`。

---

## 1. `src/core.mjs` —— McBot（mineflayer 封装，不依赖 DSH）

**导出**：`McBot`（default 也是它）、`Vec3`、`DEFAULTS`、`TIMEOUTS`、`DEFAULT_COMMAND_WHITELIST`、`withTimeout`、`raceAbort`、`jsonSafe`、`lossless`、`glyphOf`、`logLine`、`libraryInfo`、`sessionFlags`、`takeAuthJoinError`、`wrapYggdrasilServer`、`friendlyAuthError`。

### 工具函数

- `withTimeout(promise, ms, label)`：race 守卫定时器**故意不 unref**（否则只剩它跑时进程提前退出）；超时错误带 `mcTimeout:true`。
- `raceAbort(promise, signal, label)`：给 await 接 abort（宿主 cancel 不杀同进程 promise）；错误带 `mcAborted:true`。
- `lossless(value)`：宿主工具返回值校验要求纯 JSON（原型必须是 `Object.prototype`/null，不允许 `-0`）。类实例只留自有可枚举属性、Vec3→`{x,y,z}`、Date→ISO、Map/Set 展开、循环→`'[circular]'`、NaN/±Inf→null。
- `logLine(...)`：独立落盘（无实例也能用），前缀 `[whale_craft HH:MM:SS]`，写 `DEFAULTS.logFile`（默认 `$DSH_HOME/whale_craft/logs/whale-craft.log`，`MC_LOG` 可覆盖；**不写包目录**）。
- `libraryInfo()`：报 mineflayer 版本 / testedVersions / minecraft-data 版本，不连服。
- `sessionFlags(mode)`：离线账户 `haveCredentials/useAccessToken=false`（假 token 做 session join 会 `ForbiddenOperationException`，真炸过）；yggdrasil=true。
- `wrapYggdrasilServer` / `takeAuthJoinError` / `friendlyAuthError`：yggdrasil 库新旧回调兼容层 + 认证错误转"可执行的人话"（`needUserAction:true`）。

### McBot 类

- 构造：`{...config, instanceId}`；字段含 `bot/sub/connecting/connectedAt/lastError/autoReconnect/reconnectDelay/reconnecting/reconnectPending/chat/inputTimer/stopped/lastTimeout/abortSignal/stats{connects,deaths,chats,lastEventAt,timeouts}`。
- **`online` getter** = `bot.entity 存在 && _client.ended !== true` —— 幽灵在线的判据（见下）。
- `connect(opts)`：参数 `{host, port, subserver, version, auth, onAuth}`；`auth` 是**唯一凭据入口**（`{mode:'offline'|'yggdrasil', username, password?, server?...}`），绝不出现在任何返回值。细节：
  - 已在线且同 sub+host → 直接返回；并发 connecting → 等同一个 promise；
  - 顶号（`already connected|already logged` 且建连 <9s）最多重试 4 次 × 3.5s；失败**保留原连接**；
  - yggdrasil 才设 `sessionServer`；`fakeHost: sub` 过 HAProxy 子服路由；等 spawn 超时 `connectTimeoutMs`（45s）。
- **重连**：延迟 5s 起、失败翻倍上限 60s、成功复位；`reconnecting` 只覆盖两次尝试间窗口，`reconnectPending` 覆盖**整个断线期**（直到真重连成功/手动 connect 成功）。
- **`offline` 事件**（`b.on('end')` 里 emit）：`{sub, reason: lastError ?? '连接结束', willReconnect, at}`；`willReconnect = autoReconnect && bot===b && !stopped`。**断线必须进上层队列**（0.1.7 修）。
- `disconnect(reason)`：停重连/输入/观察器 → `quit()` 宽限 3s（计时器故意不 unref）→ 没走掉才强断；返回 `{graceful,forced,ms}`。`authOnly()`：只认证不连接（账户 refresh 用）。
- **事件清单**：`spawn {sub,position,gamemode}`、`offline`、`reconnect {sub}`、`death {position}`、`damage {health,position}`（health≤6）、`chat {who,text}`（不含自己；1.5s 同人同文去重）、`system {text}`、观察器 `playerJoin/playerLeave {who}`、`teleport`（>24 格/秒）、`pushed`（≥2 格且非自移）、`pickup {items}`、`log`。
  - ⚠️ **core 没有 `heartbeat` 事件** —— 心跳是看门狗配置项，靠 `stats.lastEventAt` 判。
- **聊天识别**（`wireChatEvents`/`#playerChatFrom`）：`player_chat` 包（签名）直接处理；`message` 事件里 `position==='chat'` 跳过防重复；`game_info` 必 system；其余先试玩家聊天判据（`chat.type.text` / `chat.type.team.*` / `commands.message.display.incoming` / 兜底渲染形状正则 `<who> text`）——覆盖"被服务端塞进 system 位置的玩家聊天"。
- **协议护栏**：`#supportsPacket(version, name)` 查 `minecraft-data(version).protocol.play.toServer.types['packet_'+name]`（Map 缓存）。`player_input`（20Hz 按键上报）只在该版本真有时才发，每 tick 复查（重连可能换版本）。**红线**：protodef 对未知包名**不报错**、写出 `02 00 00`（id=0x00 空 body），会被服务端当 `accept_teleportation` 解 → 抛 DecoderException 秒踢（1.21/1.21.1 事故）。
- **动作方法**（全部经 `#t()` 超时包装，超时/中断自动松 7 个控制位）：`walkTo`（arrive 1.6、|dy|≤1.5、一直按 forward、1.2s 无进展跳）、`flyTo`（仅创造，finally 必 `stopFlying` 恢复重力）、`dig`（自动换收割工具）、`placeBlock`（判据链：选块→缺货自动给→距离>5.5 直接抛"够不着"→目标必须 `canPlaceInto`（boundingBox 'empty'，水/岩浆/草花可放）→ 找 6 邻域参照→**复验 `blockAt` 才报 placed**）、`breakBlock`（复验 `now!==name` 才报 broken）、`build`、`giveItem`（协议级 `creative.setInventorySlot`）、`clearInventory`、`useBlock`、`attack`、`tossItem`、`runSequence`（≤64 步，见工具表）、`command`（必须 `/` 开头，带 allow 回调）、`chatSay`。
- **观察**：`scan` / `heightmap`（R≤96）/ `mapImage`（RGBA，白框标自己）/ `entities` / `inventory` / `waitForChunks`（默认 20s；`blockAt` 脚下出现即算到）/ `status()`（含 ghost 特判）/ `connectionView()`（只出 host/port/subserver，不带账号）。
- **错误纪律**：`#reportError` 是唯一错误上报口 —— 记 lastError+日志，**仅在有 listener 时才 emit('error')**（EventEmitter 无监听者 emit('error') 会 throw，曾把整个 DSH 带走）。
- 不变量：`setInterval` 回调必须自兜异常（宿主只对 `unhandledRejection` fail-loud，`uncaughtException` **直接杀进程**）；`_selfMovingAt` 用于把自走/自飞从 teleport/pushed 误判中排除（3s 窗口）。

## 2. `src/wait.mjs` —— waitForEvents

- 参数 `{sess, from, kind=null, waitSec=0, signal=null, pollMs=250}`（`now`/`sleep` 可注入，供测试）。
- 返回 `{waitedMs, interrupted, reason}`。收工条件按序检查：① `signal.aborted` → `reason:'用户停止'`（interrupted:false）；② `sess.waitInterruptedAt > startedAt` → `interrupted:true, reason: waitInterruptReason`；③ 新事件出现 → `reason:'有事件'`；④ 超时 → `reason:null`。
- **存在的理由**：宿主 `agent.steer` 在**下一步 step 边界**消费，而 step 边界要等当前工具调用返回 —— 没有打断机制时，`mc_events{waitSec:120}` 会把看门狗的唤醒文案**压到等待结束**才投递（"等待堵住了它正在等的那件事"，0.1.7 事故）。

## 3. `src/watchdog.mjs` —— 单脑看门狗

**导出**：`Watchdog`、`WATCH_DEFAULTS`。

- 构造 `{ctx, sess, agent, promptSignal}`（promptSignal **必传**，`sessionController.prompt` 是 @Remote）。
- `arm()` 幂等：`learnName(sess.bot.username)`（**从登录档案现学游戏名**，不写进源码——防私人名字随开源副本泄露）→ 绑 bot 事件 → `#startJob()`（宿主 jobs，`kind:'mc-watch'`，`owner:agent`；无 jobs 服务降级"无 job 模式"）→ 1s `setInterval(#tick)`。
- `disarm(reason, {notify, fromJob})`：先 teardown，再把 job 结算（**幂等**，只结算一次；`readOutput` = 留档末 30 条）；`notify` 时注入"你已经不在 MC 里了…"。自动关闭时机：offline 且不重连 / `mc_disconnect{notify}` / stopSession / 会话 destroy / 宿主 kill job（`fromJob:true`）。
- **唤醒矩阵**（`wakeOn`，`mc_config` 可改）：

| 项 | 默认 | 判定 | 文案要点 |
| --- | --- | --- | --- |
| `mention` | ✅ | `calledBy()` 正则命中 `mentionPatterns` | `[有人喊我] who text ← 命中叫法` |
| `nearbySpeech` | ✅ | hypot ≤ `nearRadius`(16) | `就在我旁边` |
| `damage` | ✅ | damage 事件 | `血量降到 N` |
| `death` | ✅ | death 事件 | `我死了` |
| `teleport` | ✅ | teleport 事件 | `位置瞬移 N 格` |
| `pushed` | ❌ | pushed 事件 | `被动移动` |
| `itemPickup` | ❌ | pickup 事件 | `捡到 …` |
| `playerJoin` / `playerLeave` | ❌ | 对应事件 | `who 上线/下线了` |
| `disconnect` | ✅ | offline 且 willReconnect | `连接断了（reason）——正在自动重连…` |
| `heartbeat` | ❌ | 距上次唤醒 ≥ `heartbeatSec`(300) 且在线 | `【心跳｜已挂机 Ns】` |

- 同话题延续：唤醒后 `topicWindowSec`(120s) 内的发言**直接算 mention**。命中进 `pending` 攒 `observeWindowMs`(2s) 合并 → `#flush` 过 `maxWakePerMinute`(6) 限流 → 一条 `【MC 看门狗｜标签】` 正文注入。另有 `followUpAfterSec`(45)：唤醒后 45s 无下文补提醒一次。
- **`#inject`（唯一注入口）**：① gate（非 MC 模式只记账 drop）；② **先 `sess.interruptWait(kind)`**；③ 首选 `agent.steer(userMessage(...))`（`source:{kind:'plugin',form:'notice'}`，宿主渲染成折叠一行；空闲时起一轮、运行中下一步插话）；④ 兜底 `sessionController.prompt({mode:'steer'}, promptSignal)`。**绝不用** `followup`/queue 类"冒充用户发言"的通道。
- 留档 `this.log`（内存，上限 200，`#record` 是唯一写入点）—— 与 `sess.events` 分开，双写曾是 bug。

## 4. `src/memory.mjs` —— MemoryStore（记忆树）

- 构造 `(root, {create=true})`；**生产用 `create:false`**：目录只在两个时机建（首次 MC 模式会话 / 点开 MC设置，`ensureMemoryRootCwd` + `seededRoots` 去重）。
- 常量：单文件文本 256KB、图片 16MB、最多 2000 文件、深度 ≤5、段名 `^[\w.\-一-龥 ]+$`。
- **`safePath(rel)` 是唯一安全关口**：拒空/绝对路径/盘符；`RULES.md`、`AGENTS.md` 与根级插件文件（README/accounts/config/.rules-version）**对 AI 封锁**；拒 `.`/`..`/深层/超长/非法段；`resolve` 后必须仍在 root 内。
- 方法：`ensureRoot/ensureReadme`（骨架只建一次）· `pathFor({topic,server})`（server 缺省 `_global/`；无扩展名补 `.md`）· `list()`（根级插件文件不算记忆；解析标题/条目数/摘要）· `renderTree()`/`indexText()`（5s 缓存；注入用 = README 正文 + 目录树）· `read()`（文本→content；**图片→附件**（工具层 `attachments.saveImage`）；二进制→元信息）· `put()`（**把工作区任意文件复制进记忆**，16MB，name 清洗）· `append({text,key})`（单条 ≤4000 字；同 key 正则替换旧 bullet）· `write()`（整文件覆盖，拒图片）· `delete()` · `search()`（跨文本文件逐行，limit ≤100）· `overview()`。
- 写后清 `_textCache`（投递的索引恒新）。⚠️ `list()` 只豁免**根级**插件文件，`.out/`/`.express/` 会被遍历进去（未专门跳过）。

## 5. `src/agentsmd.mjs` —— RULES.md 行事准则

- 路径：`agentsMdPath(dir)=<记忆根>/RULES.md`；`legacyAgentsMdPath`（老 AGENTS.md，只用于迁移/守卫）。
- `DEFAULT_AGENTS_MD`（默认准则）小节：宗旨 / 称呼 / 记忆 / 边界信息 / 登录游戏 / 看门狗 / 聊天 / 建筑须知 / 较长思考 / 硬规矩。
- `migrateLegacyAgentsMd`：老 `AGENTS.md` → 内容搬进 RULES.md，原文件**改名**为 `AGENTS.md.bak-<时间戳>`（不删：内容不丢、宿主不再认）；幂等。
- `readAgentsMd`（`source` 按内容是否逐字等于默认判 `default/custom`）；`writeAgentsMd`（空拒；≤128KB）；`resetAgentsMd`（**把默认写回文件**而非删除）。
- `isAgentsMdPath(text)`：先把 JSON 转义还原再匹配 —— 供 guard 在文件工具参数里**硬拒**对 RULES/AGENTS 的读写（AI 不可读写，只有 Master 在「MC设置→提示词」编辑）。
- `syncRulesVersion(dir, version, {follow})` + `.rules-version` 标记：无文件→建默认+`created`；有文件无标记→只记版本 `marked`（**不覆盖**）；版本变+follow 开→替换为默认 `replaced`；follow 关→只更新标记 `kept`（以后打开**不翻旧账**）。

## 6. `src/version-prompt.mjs` —— 版本硬提示词

- 导出 `versionPromptText/versionPromptHash/versionPromptTitle/versionPromptSource`。哈希 = 正文 sha256 前 8 位（只标正文；正文**不含版本号**，跨版本稳定）。
- 正文两条：① 本版本 `mc_move/mc_act/mc_build` 不成熟 → 优先 `mc_command`（`/tp` `/setblock` `/fill` `/clone`），被拒再回退；② 发文件流程（先放 `.express/` 或 `mc_kit_memory put`，再 `mc_kit_express`；按分享模式给路径或 URL）。
- **不可编辑、无开关**（硬编码随版本发布）——与 RULES.md（Master 维护）、记忆（玩出来的经验）三分工。

## 7. `src/user-message.mjs` —— 插件提示行构造

- `userMessage(input)`：宿主 `@deepseek-ai/dsh-llm` 的 `createUserMessage` 优先；解析不到/异常 → `builtinUserMessage`（自带等价实现：`{role:'user', content, source:{kind:'plugin'}, id}`）。
- `messageFactoryKind()`：诊断 `'host'|'builtin'`。
- ⚠️ 存在理由：0.1.4 时该包漏进依赖声明 → 别人 npm 装出来"工具都在、提示词全无"。**依赖声明不能少**，兜底只是保险。

## 8. `src/config.mjs` —— 配置 + preset 规划

### DEFAULT_CONFIG（键 → 默认 → 含义）

| 键 | 默认 | 含义 |
| --- | --- | --- |
| `commandWhitelist` | 22 个指令名 | `mc_command` 放行；支持精确名 / `/正则/flags` / `"*"` |
| `mcModePresets` | `['minecraft','whale_craft']` | 权限隔离判据（注意 `whale_craft` 带下划线，**不是合法 preset id**） |
| `mcMode.allowOtherTools` | `[]` | MC 模式白名单**额外**放行（只能收窄，不能凭空加） |
| `mcMode.hideAdminTools` | `true` | 隐藏 `mc_admin_*`（另有 guard 硬拒） |
| `memoryDir` | `null` | null = `<工作区>/.whale-craft` |
| `allowAllCommands` | `false` | 指令白名单总开关 |
| `injectWhaleCraftAgentsMd` | `true` | 注入 RULES.md |
| `injectWorkspaceAgentsMd` | `false` | 额外注入工作区根 AGENTS.md |
| `rulesFollowVersion` | `true` | 版本变 → 替换 RULES.md（靠 `.rules-version`；首见只记不覆盖） |
| `ensureMcPreset` | `true` | 启动自举「MC模式」preset |
| `expressMode` | `'off'` | 文件分享：`off`/`online`（老值 `local` 一律当 off） |
| `expressBase` | `''` | online 模式的访问 base |

- `PluginConfig`：`load`（坏配置不崩、记 `lastError` 按默认跑）、`set` 只认 `TOP_KEYS`（= DEFAULT_CONFIG 键）且过 `validate`、`values()` 深合并（数组整体覆盖）；语义 getter（`mcModePresets/memoryDir/expressMode/expressBase/commandAllowed/isMcModePreset`…）每次现读 ⇒ **改完热生效**。
- `resolveStateDir`：`WHALE_CRAFT_STATE_DIR` → `WHALE_CRAFT_DIR`+whaleDir → `$DSH_HOME/whale_craft` → `~/.dsh/whale_craft`。

### preset 规划（纯函数，供 index.js 的自举）

- `PRESET_ID_RE=/^[a-z0-9][a-z0-9-]*$/`（与宿主同名规则；下划线非法 ⇒ 目标 id 不能叫 whale_craft）。
- `MC_PRESET_SPEC=7`：规格版本，**改动 = 下次启动重建自建 preset**；`planPresetAction` 判定序：不存在→`create`；非本插件建的→只在简介是复制残留时 `meta` 否则 `leave`；无 compositionHash→只敢 `meta/leave`；spec 变→`rebuild`；组成被用户改过（hash 不符）→`leave`；官方源变了（源 hash 变）→`rebuild`；显示文本不对→`meta`。
- `pickPresetTarget`（取 `mcModePresets` 里第一个合法的）/ `PREFERRED_PRESET_SOURCES=['minimal','standard','ptc']` / `pickPresetSource` / `isCopiedPresetDescription`。
- `PERSONA_TEXT_KEYS=['prefix','text']` + `personaTextKeyOf`：跨 DSH 版本探测 persona 键名。
- `patchPersonaInComposition`（换 persona 正文；`complete:true→false`、`includeRuntimeContext:false→true`，后者会压掉其它提示段）、`disableShellInComposition`（persistent-shell `disabled:true`）、`patchToolGroupsIntoComposition`（`MC_PRESET_TOOL_GROUPS`：`tool-fs`/`tool-jobs`/`present`/`compaction`）。

## 9. `src/accounts.mjs` —— 账户与凭据

- **存储分离**：元数据 `$DSH_HOME/whale_craft/accounts.json`；凭据（password/token/clientToken）**只进宿主凭据服务**（`ctx.credentials`，key=`whale-craft/<innerID>` → `$DSH_HOME/.credentials.yaml` owner-only）。凭据服务不可用 → **直接抛错拒绝，绝不降级写明文**（MC agent 手里有 read 工具）。
- 类型：`offline` / `yggdrasil`（第三方皮肤站，authlib-injector 风格）/ microsoft（`add` 抛"暂不支持"）。
- 工具函数：`offlineUuid`（复刻 Java `UUID.nameUUIDFromBytes("OfflinePlayer:<name>")`：md5+v3+variant）、`dashUuid`、`normalizeServerUrl`、`parseAuthlibCard`（解析 `authlib-injector:yggdrasil-server:<urlencoded>` 卡片/裸网址，raw 与 decode 各试一次）。
- `AccountStore`：`load`（坏库降级空库记 `loadError`）/`save`/`ensureDefaults`（`seeded` 标记**只种一次**：预置 LittleSkin 认证服 + 默认离线账户 `DeepSeek`，可删且不长回）/`view`（**绝不含密码/token**）/`add/update/remove`（remove 连带删凭据并让渡默认）/`search`/`resolve`。
- 凭据方法：`setCredential`（`kind:'grant'` payload）/`getCredential`（**仅内部登录流程**）/`deleteCredential`/`refreshCredentialIndex`（宿主 credentials `listRecords` 是同步的）。
- 认证服务器：`listAuthServers/addAuthServer/renameAuthServer`（id 永不变）/`removeAuthServer`（被账户占用拒删）。网络登录本身在 core.mjs（yggdrasil 流程），本模块只承载元数据/凭据脚手架。

## 10. `src/express.mjs` —— 发布区（纯函数，不碰 fs）

- 常量：`EXPRESS_DIR='.express'`、`OUT_DIR='.out'`、`EXPRESS_URL_PREFIX=/api/whale-craft/express`、`WORKSPACE_ID_RE`。
- `parseExpressPath`：要求**已解码**的 pathname（`%2e%2e` 先还原成 `..` 才可判）；首段 workspaceId 过正则，其后 ≥1 段。
- `safeExpressTarget(root, segments)` 逐段拒：空段/`.`/`..`/含 `/` 或 `\`/盘符/`~` 开头/控制字符/段 >255；拼完复查仍在 root 内（纵深防御，真正兜底是路由层 `realpath`）。
- `mimeOf`（全扩展名放行，未知 octet-stream）、`SANDBOX_TYPES`（svg/html/xml/js → CSP sandbox 头）。
- `resolveExpressMode`（仅 `online` 算开）、`normalizeExpressBase`（必须 http(s)、去尾斜杠、非法 null）、`onlineUrlOf`、`expressRefFor`（abs 必须落在 `.express/` 下，`.out` 文件返回 null；产出 `{rel, url, markdown}`）、文案常量 `EXPRESS_OFF_TEXT` / `EXPRESS_NEED_BASE_TEXT`（逐字：自检断言原文）。

## 11. `src/image.mjs` + `src/png.mjs` —— 图像

- `image.mjs`：`sharp` 是**可选依赖**（`createRequire` 懒加载）；`imageEngineAvailable()/imageEngineError()`；缺失时 `need()` 抛"图像引擎不可用（sharp 解析失败：…）"，**不静默假装成功**。`ImageEngine`：`info` / `embed`（图片→data URI + `<image>` tag，引图进 SVG）/ `render`（SVG→PNG，density×scale，scale 夹 0.05-16）/ `grid`（≤64 张拼网格，**返回 SVG 文本**可继续编辑）/ `save`。
- `png.mjs`：零依赖 RGBA8 PNG 编码器（color type 6、8bit、filter 0、deflate 9）；`encodePng(width,height,rgba)`。用途：`mc_map` 出真地形图时避免给宿主多加原生依赖。

## 12. `src/lan.mjs` —— 局域网公告监听

- `LAN_BROADCAST={group:'224.0.2.60', port:4445}`；`parseLanBroadcast`（`[AD]…[/AD]` 取 1-65535 端口，`[MOTD]` 可选）。
- `listenLanBroadcast({seconds=3})`：udp4 + reuseAddr 绑 4445 → `addMembership`（禁多播环境**静默降级**为只听本机广播）；按 `host+port` 去重；时长夹 [0.5, 15]s；**错误/超时全 resolve 空数组、绝不 reject**；`timer.unref`。**只被动听，不扫端口**（2026-09-18 砍掉扫网段）。

## 13. `src/ping.mjs` —— STATUS ping

- 协议栈锚点：`requireFromMineflayer` 从 mineflayer 自己的依赖树 `require('minecraft-protocol')`（与 mc_connect 同栈同版本表）。
- `parseAddress`（默认 25565；认 `host`、`host:port`、`[::1]:25565`；裸 IPv6 抛错）；`flattenMotd`（拍平 + 去 `§` 色码）；`friendlyNetError`（ECONNREFUSED/ETIMEDOUT/ENOTFOUND/… → 人话；`unsupported protocol` → 建议手填 version）。
- `statusPing({host, port, timeoutMs, fakeHost, version})`：**永不抛异常**（P0 教训：一切 reject/超时收敛成 `{ok:false, error, hint}`）。流程：不用上游 `mc.ping`（不暴露 client、超时 120s），用 `minecraft-protocol` 原语自建：握手（nextState=1）→ STATUS → `ping_start` → 收 `server_info` → 写 `ping` 量往返延迟 → 无论成败 `client.end()+socket.destroy()` 防挂 socket。硬超时夹 [1s,30s] 默认 5s；成功返回 `{ok:true, elapsedMs, handshakeMs, statusMs, latencyMs, version, protocol, players{online,max,sample≤12}, motd, motdRaw, hasFavicon}`。

## 14. 模块依赖与不变式

- 唯一模块间 import：`config.mjs → express.mjs`（`EXPRESS_MODES/normalizeExpressBase/resolveExpressMode`）。
- 记忆根定位（index.js `memoryRootFor`）：`WHALE_CRAFT_MEMORY_DIR` env → `pluginConfig.memoryDir` → `<会话 cwd>/.whale-craft` → `stateDir/memory` 兜底。
- 跨模块不变式：① 记忆路径全过 `safePath`，RULES/AGENTS 双向封锁；② 凭据只进宿主凭据服务，`view()`/工具返回/HTTP 永不见；③ 发布区只服务 `.express/`，`.out/` 永不对外；④ LAN 只被动听；⑤ ping 永不 reject；⑥ 一切写给模型的注入都是"提示行"。
