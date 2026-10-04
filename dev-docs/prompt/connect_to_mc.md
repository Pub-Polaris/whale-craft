# 连接到MC 提示词

用户在本插件「连接到MC」弹窗里点「连接」（或点探测到的局域网服务器行）时，插件给**当前会话**
注入这段提示词并让它跑一轮——真正的连接由 LLM 去调 `mc_connect` 完成。

**实现以 `src/connect-prompt.mjs` 为准**，本文件是它的可读版本（改两处要一起改）。

```text
Connect to the Minecraft server at ${address} using the account ${account}.
Call mc_connect with that address and account, then use mc_events to see what happened.
```

- `${address}`：服务器地址——手动连接用**用户在输入框里填的原文**；局域网按钮用探测到的 `host:port`。
- `${account}`：形如 `DeepSeek (id: acc-1a2b3c4d)`。名字给人看，`id:` 后面是 `mc_connect` 的 `account`
  参数要的内部 id（**只传 id，不碰密码**）。
- **仅局域网按钮**触发时，多一行：

  ```text
  The address points to a server on the local network and may be temporary.
  ```

- **追加插槽**：`CONNECT_PROMPT_APPENDERS`（`src/connect-prompt.mjs`）——"因属性而追加提示词"的扩展点，
  每项 `(ctx) => string`，非空即追加。默认空。

> 🔴 铁律：注入走**插件提示行**（`source.kind='plugin:whale_craft'`, `form:'notice'`），
> **绝不冒充用户发言**。
