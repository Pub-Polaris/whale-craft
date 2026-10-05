# whale_craft 工具参考（29 个）

> 快照 whale_craft 0.2.0。定义以 `index.js` 的 `apply()` 为准。
> 参数记法：`名:类型`，`*` = 必填，`=值` = 默认值。

## 模式与工具暴露范围

| 模式 | 暴露范围 |
| --- | --- |
| **MC模式** | 本插件：`mc_*` 除 `mc_admin_config` 外的 24 个 + `mc_kit_*` 3 个（共 27 个）<br>宿主：文件工具 `read`/`write`/`edit`/`glob`/`grep`/`read_image` + `present` + `mcMode.allowOtherTools` 配置项<br>**不含** `mc_admin_config` |
| **MC+模式** | 全部 29 个本插件工具（含 `mc_admin_config`）；不套白名单，另见标准模式全量工具 |
| **其他模式** | 仅 `mc_admin_config`（`mc_*` / `mc_kit_*` 全部隐藏，另有 guard 硬拒兜底） |

## 一、连接与会话（10）

### `mc_status`
- **描述**：当前会话状态：在线/坐标/血量/子服/连接/待处理事件/看门狗 + 最近聊天
- **参数**：无
- **超时**：默认

### `mc_lan`
- **描述**：只听局域网多播公告，找"对局域网开放"的房间（不扫端口，恒定 seconds 秒内返回）
- **参数**：`mode:string=broadcast`、`seconds:number=3`（≤15）
- **超时**：默认

### `mc_ping`
- **描述**：对已知地址发一次 STATUS ping（不登录）：通不通/版本/协议号/MOTD/人数/延迟
- **参数**：`address:string*`、`port:number=25565`、`timeoutMs:number=5000`（≤30000）、`subserver:string`
- **超时**：默认

### `mc_connect`
- **描述**：连接服务器（唯一连接入口）；账号来自「MC设置」，成功后等区块并自动挂看门狗
- **参数**：`host:string`、`port:number=25565`、`subserver:string`、`account:string`、`version:string`（缺省自动探测）
- **超时**：默认

### `mc_accounts`
- **描述**：账户 list/search/use/refresh；永不返回密码/token
- **参数**：`action:string=list`、`innerID:string`、`query:string`
- **超时**：默认

### `mc_stop`
- **描述**：停本会话：优雅退游戏 + 清后台任务；不中断当前轮
- **参数**：`reason:string`
- **超时**：默认

### `mc_disconnect`
- **描述**：从 MC 下线；看门狗自动关闭并提醒
- **参数**：`reason:string`
- **超时**：默认

### `mc_sessions`
- **描述**：列出所有活跃 MC 会话实例（诊断）
- **参数**：无
- **超时**：默认

### `mc_capabilities`
- **描述**：能力/限制自述：支持版本、mineflayer 版本、登录方式、各项上限
- **参数**：无
- **超时**：默认

### `mc_diag`
- **描述**：诊断快照：物理/控制位/收包/事件队列 + 提示词注入状态
- **参数**：无
- **超时**：默认

## 二、观察（5）

### `mc_map`
- **描述**：看周围地形：chars 字符图 / image 俯视图 / both
- **参数**：`radius:number=32`（≤96）、`glyphStep:number=2`、`yTop:number=10`、`yBottom:number=-24`、`format:string=chars`、`scale:number=4`、`out:string`
- **超时**：60s

### `mc_scan`
- **描述**：扫描周围方块：给 name 找位置，不给则统计
- **参数**：`radius:number=8`（≤24）、`height:number=4`（≤16）、`name:string`、`limit:number=10`
- **超时**：默认

### `mc_entities`
- **描述**：附近实体（玩家/生物/掉落物）及距离
- **参数**：`radius:number=24`
- **超时**：默认

### `mc_inventory`
- **描述**：背包与手持物品
- **参数**：无
- **超时**：默认

### `mc_events`
- **描述**：读/消费事件队列（聊天/系统/受伤/生死/断线）；peek 只看不清
- **参数**：`limit:number=20`（≤100）、`kind:string`、`peek:boolean`、`waitSec:number=0`（≤120，会被唤醒打断）
- **超时**：130s

## 三、看门狗控制（2）

### `mc_watch`
- **描述**：看门狗控制（唯一唤醒通道）：status/arm/disarm/log
- **参数**：`action:string=status`、`reason:string`
- **超时**：默认

### `mc_config`
- **描述**：读写本会话看门狗配置（唤醒条件/近距半径/叫法等）
- **参数**：`patch:object`（点号键浅合并）、`reset:boolean`
- **超时**：默认

## 四、交互与动作（8）

### `mc_say`
- **描述**：公屏说话（截 220 字）
- **参数**：`message:string*`
- **超时**：默认

### `mc_move`
- **描述**：移动：walk 走 / fly 直飞 / jump 原地跳
- **参数**：`x/y/z:number`、`mode:string`、`budgetMs:number=40000`（≤90000）
- **超时**：120s

### `mc_act`
- **描述**：单动作：look/toward/place/break/use/attack/equip/toss
- **参数**：`mode:string=look`、`who:string`、`x/y/z:number`、`name:string`、`count:number`、`approach:boolean=true`、`budgetMs:number`
- **超时**：120s

### `mc_dig`
- **描述**：挖方块：给 name 挖最近的，或给坐标
- **参数**：`name:string`、`x/y/z:number`、`maxDistance:number=6`、`count:number=1`（≤16）
- **超时**：120s

### `mc_build`
- **描述**：长方体批量搭建（会走近/垫脚）
- **参数**：`x1/y1/z1/x2/y2/z2:number*`、`name:string`、`max:number=64`（≤256）
- **超时**：180s

### `mc_give`
- **描述**：创造模式直接取物品（协议级，无需 OP）
- **参数**：`name:string`、`count:number=1`、`slot:number`（0-44）、`clearAll:boolean`
- **超时**：60s

### `mc_sequence`
- **描述**：按序执行一串世界交互（≤64 步）
- **参数**：`steps:array`、`stopOnError:boolean=true`、`budgetMs:number=300000`（≤570000）
- **超时**：600s

### `mc_command`
- **描述**：执行服务器指令（白名单，最后手段）
- **参数**：`command:string*`
- **超时**：默认

## 五、游戏外辅助（3，`mc_kit_*`）

### `mc_kit_memory`
- **描述**：长期记忆（工作区 `.whale-craft/`）读写：index/read/append/write/put/delete/search
- **参数**：`action:string=index`、`path:string`、`topic:string`、`server:string`、`text:string`、`content:string`、`key:string`、`source:string`、`name:string`、`query:string`、`limit:number=30`
- **超时**：默认

### `mc_kit_image`
- **描述**：图像：info/embed/render（SVG→PNG）/grid/save
- **参数**：`action:string`、`path:string`、`out:string`、`svg:string`、`svgPath:string`、`width:number`、`height:number`、`scale:number`、`paths:array`、`cols:number`、`cell:number=256`、`gap:number=8`、`labels:array`、`title:string`
- **超时**：120s

### `mc_kit_express`
- **描述**：把发布区 `.whale-craft/.express/` 的文件换成给用户的一行（路径或 URL）
- **参数**：`path:string*`
- **超时**：默认

## 六、管理（1，`mc_admin_*`）

### `mc_admin_config`
- **描述**：全局配置读写（get/set/unset/reset/list）；MC 模式不可见也不可调
- **参数**：`action:string=get`、`path:string`、`value:json`
- **超时**：默认
