# 发布流程

> 快照：0.1.7（2026-10-02）。仓库根还有一份面向"人"的 `RELEASING.md`（更新、更细，含实测记录）；本文是给 agent 的摘要 + CI 行为细节。
> ⚠️ 当前状态：main 比 `v0.1.7` tag **多 2 个未发版 commit**（GitHub issue #1 修复、`tools/dev.mjs` 新增）——下次发版需要先在 `CHANGELOG.md` 补节并升 `package.json` 版本。

## 0. 两条渠道，互不依赖

| 渠道 | 触发 | 产物 |
| --- | --- | --- |
| **GitHub Release** | 推 `v*` tag → CI 自动 | `whale_craft-<版本>.zip` + 正文取 CHANGELOG 本节 |
| **npm** | ① CI（仓库配了 `NPM_TOKEN` 就自动发）② 或本机手动 `npm run publish:npm` | npm 包 `whale_craft@<版本>` |

## 1. 版本纪律（发版前必做）

1. `package.json` 的 `version` 升到目标版本；
2. `CHANGELOG.md` 加一节 `## [<版本>] - YYYY-MM-DD`（**Release 正文就是它**，写得像 0.1.7 那样：现象/根因/修法/验收）；
3. 提交、推 main；
4. 打 tag 并推：
   ```bash
   git tag -a v0.1.7 -m "whale_craft 0.1.7"
   git push origin v0.1.7
   ```
   **tag 必须与 `package.json` 版本逐字一致**（release.yml 会校验并直接失败）。

## 2. `release.yml` 行为（推 tag 后）

1. 跑 `check-core` + `selfcheck`（全绿才继续）；
2. 校验 tag == `package.json.version`；
3. `npm pack` → 解包 → 打 **zip**（不是 .tgz）；
4. 用 `.notes.cjs` 从 `CHANGELOG.md` 提取 `## [<版本>]` 那一节当 Release 正文（**不用 `--generate-notes`**，不用 diff）；
5. `gh release create` 挂 zip；
6. **npm "先探再发"**：探测仓库 secret `NPM_TOKEN` —— 有才发（发布前再查 `npm view <pkg>@<版本>` 防重复，已存在则跳过）；**没有就打一条 notice 跳过，工作流照样绿**（2026-09-17 的教训：token 失效让"整条红叉、而发布其实早就成功"）。

## 3. `ci.yml` 行为（push main / PR）

- **check 矩阵**：ubuntu（Node 22 / 24）+ windows（Node 22）→ `npm ci` → `check-core` → `selfcheck`；
- **package job**（全绿后）：`npm pack` 核对 tarball —— 必含 `package.json/index.js/client.js/selfcheck.mjs/cordis.patch.yml/LICENSE/README.md/tools/check-core.mjs`；**不得**混进 `node_modules/`、`logs/`、`accounts.json`、`config.json`、`.whale-craft`；上传 artifact。

## 4. 改工作流文件本身的坑（实测）

`.github/workflows/*` 的推送可能需要 token 有 **`workflow` scope**，而经代理通道推可能**明明有 scope 也被拒**（是通道问题不是 token 问题）。实测 **Contents API 可以**。所以：

```bash
# 修好的模板在 scripts/release.workflow.yml（普通文件随代码分发）
node scripts/land-workflow-fix.mjs --dry    # 先看会改什么
GITHUB_TOKEN=<带 workflow scope 的 token> node scripts/land-workflow-fix.mjs
```

`land-workflow-fix.mjs`：预检 token scopes → 优先 Contents API（GET 拿 sha → PUT base64）→ 失败回退 git push（报 workflow scope 错时提示改用 API）→ 回读校验（剥注释后确认远端没有 `npm publish` 之类预期外内容）。不想折腾 token：把模板内容**粘到网页上**的 `.github/workflows/release.yml`（网页编辑不需要 scope）。

## 5. 本机手动发 npm（不依赖任何 CI secret）

```bash
npm login                                   # 或 NODE_AUTH_TOKEN / 本仓库 .npmrc（已 gitignore）
node scripts/publish-npm.mjs --dry          # 先演练：全检查走一遍不真发
node scripts/publish-npm.mjs                # 真发（要求输入 yes 确认）
node scripts/publish-npm.mjs --yes --otp 123456 --tag next
```

`publish-npm.mjs` **前置检查（任一不过即停，不会发出半成品）**：

1. git 工作树干净（`git status --porcelain` 为空）；
2. `package.json` 版本在 `CHANGELOG.md` 里有对应节；
3. `check-core` + `selfcheck` 全绿（`--skip-checks` 可跳）；
4. `npm whoami` 拿得到身份（token 不可用当场停，不会等 publish 才 404）；
5. 该版本 npm 上**还没发过**（并报当前 latest）；
6. `npm pack --dry-run` 清单里没有 `logs/`、`accounts.json`、`config.json`、`.whale-craft`；
7. 提醒 GitHub 有没有对应 tag（只提醒，不拦）。

- `publishConfig` 钉死 `registry: https://registry.npmjs.org/` + `access: public`（避免本机镜像 registry 把包发错地方）。
- `prepublishOnly` = `npm run check`（**坏树发不出去**，即使不经脚本直接 `npm publish` 也拦得住）。
- Windows 上脚本显式走 `cmd.exe /c`（不用 `shell:true`，防 DEP0190 与参数拆分）。

## 6. 常见问题（速查）

| 现象 | 原因 / 处理 |
| --- | --- |
| 推送被拒 `… without 'workflow' scope` | token 缺 workflow（或**推送通道**问题）→ §4 Contents API / 网页编辑 |
| `E404 Not Found - PUT https://registry.npmjs.org/…` | token 不能发布（过期/只读/非 Automation）→ 重新 `npm login` |
| `npm whoami` 401 | token 没配好 |
| tag 工作流红叉、报 `npm …` 失败 | 远端还是旧工作流 → §4 |
| Release 正文是 `Full Changelog: …` / 附件是 `.tgz` | 同上，旧工作流 → §4 |
| tag 校验失败 | tag 与 package.json 版本不一致（改 tag 或改版本重发） |
