# 浏览器半端（client.js）与 UI 约定

> 面向接手本仓库的 AI agent 与开发者。讲 `client.js` 挂到 DSH Web 前端的**插槽扩展点**、
> **图标约定**，以及"哪些入口属于 root 作用域、哪些属于会话"。
> 快照：whale_craft **0.1.7**（含未发版改动：插件详情页「设置」入口），2026-10-04。

## 一句话

`client.js` 是**手写 factory bundle**（无构建步骤，`exports["./client"]` 直接指向它；
改完由 `@deepseek-ai/dsh-client-hmr` 热换，**不需要**重启实例，刷新页面即可）。
它只做三件事：会话状态条 / 强制停止、**「设置」模态框**、以及往宿主插槽注册入口。

## 插槽扩展点清单

| 插槽 | kind | 作用域 | 入口 | 何时出现 |
| --- | --- | --- | --- | --- |
| `conversation.session.header.actions` | list | **session** | 状态条 / 强制停止（order 50）、「设置」按钮（order 45） | 进了游戏的会话 |
| `conversation.input.right` | list | **session** | 新会话页 hero 的「设置」按钮（**驱动器**，本身 return null，按钮靠 DOM 插到模式芯片右边） | 新会话页 + MC 模式 |
| `plugins.detail.actions` | list | **root** | 插件页 → **whale_craft 详情页**头部的「设置」按钮（order 20） | 打开 whale_craft 的 bundle 详情页时 |

- **会话插槽**（前两个）由宿主注入 `sessionId` / `useSessions` / 会话 cwd，能做"按会话/工作区"的判定与请求。
- **root 插槽**（`plugins.detail.actions` 等）**没有会话、没有工作区**。宿主对这些 list 插槽的约定是：
  每个条目都拿到该页的 `subject`（`{kind:'bundle',pkg}` / `{kind:'row',pkg,row}` / `{kind:'item',id}`），
  **对无关的 subject 返回 `null`** 即可（页面按 `order` 排序，条目自绘 chrome）。
  详见 `@deepseek-ai/dsh-client-ui-plugin-manager` README 的 "Detail page extension points"。
- 我们靠 `subject.kind==='bundle' && subject.pkg.name==='whale_craft'` 自过滤，因此按钮**只**出现在
  whale_craft 自己的详情页，不会污染别的插件页。**不要**往 `plugins.item` 注册——那一组卡片会被列在
  「官方」分组下，把我们标成"官方"语义不对；whale_craft 的包卡片本来就在「已安装」里。

## 🔴 图标约定（2026-10-04 用户定）

**所有 UI 图标一律取自 DSH 官方图标集，不要自己画、也不要另引第三方图标库。**

- 图标集随 `@deepseek-ai/dsh-client-ui-primitives` 分发，而该包是宿主 shell 的**平台内置模块**
  （静态模块表里就有它：`react` / `react-dom` / `@deepseek-ai/cordis` / `dsh-client-store` /
  `dsh-client-ui-slots` / `dsh-client-ui-primitives` / `dsh-client-ui-dockkit`）。官方插件（如
  插件管理器）也是直接 `require` 它。**因此我们 `require` 即得"同源同款"，无需在 package.json 声明。**
- 用法：`require('@deepseek-ai/dsh-client-ui-primitives').IconSettingsOutlineRegular`（本文件顶部有
  **兜底 require**：宿主万一没提供就退回无图标，别让整个客户端半端挂掉）。
- **纯 DOM 按钮**（hero 那个靠 `mountHeroChipButton` 注入的）：拿不到 React 组件，所以借
  `require('react-dom/client').createRoot` 把图标渲染进按钮（图标仍是同一套，不另画/不搬路径），
  dispose 时 `unmount()`。React 渲染的按钮（标题条、插件详情页）直接用组件即可。
- 命名规律：`Icon<名字>OutlineRegular` = **1px 描边**（常规）；`…OutlineMedium` = **1.3px**（强调）。
  `size` 是 prop（默认 16）。原生「卸载」按钮用的是 `IconTrashOutlineRegular`（size 13），
  我们的「设置」按钮就用 `IconSettingsOutlineRegular`（size 13）与之对齐。
- 图标集是 DSH **自家设计**（16px 网格、1px 描边），**不是**某个开源图标库 ⇒ "官方同款"只能靠
  这个平台模块；若以后需要它没有的图形，才考虑引第三方开源图标库，并在 `THIRD_PARTY_NOTICES.md` 记一笔。

## 设置的两态：「有工作区 / 无工作区」（2026-10-04）

`McSettingsModal` 按**有没有工作区**呈现两套形态，判据是服务端 `/api/mc/config` 回的 `hasWorkspace`
（`wsCwd` 只是兜底初值）：

- **有工作区**（会话标题条 / 新会话页入口）：标题右边加**小间隔 + 文件夹图标 + 灰色工作区名**
  （`IconFolderOpenRegular` + registry 的 `title`；`default-workspace` 显示为「默认工作区」）。
  每个**工作区相关设置项**的标题旁挂 `WsMark`（灰色文件夹图标，hover「该设置项应用于本工作区」）。
- **无工作区**（插件详情页入口，root 作用域）：提示词三开关 / 版本标记 / 发布区**隐藏**，提示词正文
  **只读**且显示内置默认；账户 / 指令白名单**照常可用**。受影响子页面底部居中出现 `WsNeedHint`
  （文件夹图标 +「在对话中打开设置，以编辑工作区详细设置」）。

请求统一走 `withSid()` 带 `sessionId`/`cwd`；服务端 **`resolveWorkspaceCwd` 可空**（不 400、不建档），
`configView(cwd)` 在无 cwd 时工作区键回 `null`，`agents-md` GET 无 cwd 时只读回 `DEFAULT_AGENTS_MD`。
🔴 无工作区时**绝不能**把 `null` 喂给 `wsCfgValues` / `memoryRootFor`（后者会兜底到全局
`stateDir(/memory)` 目录）。

入口细节：**插件详情页**那个按钮（`plugins.detail.actions`）用**受控** `open`/`onClose` 打开模态框；
会话两入口仍是非受控（`settingsBus`）。**新会话页**按钮只要是 MC/MC+ 就显示，**没选工作区时禁用**
（`mountHeroChipButton` 的 `getDisabled` + `refresh`，不重建 DOM）。详见 [architecture.md](architecture.md)。

提示词页「注入」区的两个文件名（`RULES.md` / `AGENTS.md`）用 `FileName` 渲染：文件不存在时**斜体灰删除线**
+ hover「目前没有这个文件」（`data-wc-missing`），但**不禁用**开关——开关始终用于改配置。

## 🔴 弹窗约定：右上角关闭按钮（2026-10-04 用户定，其余弹窗照此）

关闭按钮 = **叉图标**（`IconCloseOutlineRegular`），`data-wc-xbtn`：**与标题同色**（`--dsw-alias-label-primary`）、
**无边框、无底色**、hover 变**红**（`--dsw-alias-state-error-primary`）。**不要**再用 `data-wc-btn` 那种带边框的
文字按钮（旧版是 `data-wc-btn data-wc-tiny` + 文本 `×`）。

## 快速验证

```bash
node tools/check-core.mjs     # 覆盖 client.js 的语法/动态加载
npm run dev:web               # 隔离调试实例；改 client.js 只需刷新浏览器
```

> 改 `client.js` 的结构约定（新增插槽入口、换图标库）后，**同步更新本页**。
