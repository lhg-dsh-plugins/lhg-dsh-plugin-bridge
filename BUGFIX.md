# BUGFIX · 0.5.0 → 0.5.1

## 标题

取材渲染超过总字符硬预算：`buildBridgeSource` 在「只剩最新一条、省略提示行装不下」时返回比 `sourceCharBudget` 还长的文本（`src/compression.ts` 的 `renderSection`）。

## 现象与根因

`renderSection` 对 `keep: 'newest'` 分区从最新一条往前装；有条目被丢弃时要在正文前加一行提示（如「（较早的 5 条用户消息因预算省略，保留的是最近的）」），提示行本身也占额度，装不下就逐条让出较早条目——但让位循环守着 `picked.length > 1`：只剩最新一条、提示行仍装不下时，循环停手，函数把「条目 + 提示行」的超额文本原样返回。这违背该函数自述契约「在给定字符预算内渲染一个分区」与模块设计原则「总字符有硬预算」；超出的量恰好是提示行长度加换行。

## 证据链

1. **最小复现（离线，纯函数）**：脚本构造 5 条短用户消息 + 1 条 200 字最新消息，对预算 150–1200 逐点调用 `buildBridgeSource` 检查 `text.length <= budget`（脚本存于集成工作区 `.fixwork/plugin-bridge/repro-budget.mjs`，未入库）。修复前输出节选：

   ```
   budget=218 text.length=245 overflow=27 dropped=["users"]
   budget=219 text.length=245 overflow=26 dropped=["users"]
   ...
   VERDICT=BUG 在 27 个预算点上输出超过硬预算   # exit 1
   ```

   `git stash` 回基线复跑同一脚本仍为 exit 1、同样 27 个违规点；修复后 exit 0、零违规。
2. **issue 面**：上游 open issue 为 0 条（`gh api` 实测），本 bug 由静态审查（让位循环的边界条件）定位、复现坐实。
3. **回归测试**：`test/compression.test.mjs` 新增「取材总长在任何预算点都不超过硬预算」一条（同一构造扫 150–1200 预算）。该测试在未修复代码上失败（26 过 1 红，红的正是新增条），在修复后 27/27 全过。

## 改动文件

- `src/compression.ts`（唯一逻辑改动，+5 行）：让位后只剩一条且提示行仍装不下时，舍提示行、只交最新一条正文，仍标记 `clipped`——省略事实继续经 `BridgeSource.dropped` 通道由调用方展示（该字段的文档用途正是「把被丢弃的分区显示出来」），不静默。正文内容与预算优先级不变。
- `lib/compression.js`：按上游 `npm run build` 重新生成的入库产物（上游 lib/ 随仓提交），与源码改动一致。
- `test/compression.test.mjs`：新增上述回归测试一条，上游既有测试一字未改。
- `package.json` / `package-lock.json`：version 0.5.0 → 0.5.1（lock 两处版本字段同步）、`repository.url` 改落点仓。
- `README.md`：顶部追加中文 fork 头，上游原文一字未删；`UPSTREAM.md`、`BUGFIX.md` 新增。`LICENSE` 与上游逐字一致。

## 复验结果

- 复现脚本：修复前 exit 1（27 个预算点超限，最大超 27 字符）；修复后 exit 0。
- `npm test`（build + typecheck + 全量测试）：
  - 修复前基线（root 沙箱）：408 条 / 406 过 / 1 红 / 1 跳过。
  - 修复后、bump 前：409 条 / 407 过 / 1 红 / 1 跳过，唯一红与基线相同（见下）。
  - 修复后、bump 后（最终形态）：409 条 / 406 过 / 2 红 / 1 跳过。两条红均非产品回归：
    1. `test/summary-file.test.mjs:126`「暂时没有权限时不会忘掉登记」——基线即红，与本改动无关：沙箱以 root 运行，`chmod 0` 拦不住 root 删除，测试的 finally 对已被删目录 `chmod` 报 ENOENT；以非 root 用户 `nobody` 单跑该文件为 19 过 / 0 红 / 1 跳过，证明是 root 环境伪影。
    2. `test/upstream-contract.test.mjs:90`——上游发版一致性断言要求 README 的置顶回退安装行 `github:Totoro-qaq/dsh-plugin-bridge#v<pkg.version>` 与包版本一致。本 fork 按契约不改上游 README 正文（只在顶部加 fork 头），版本号 bump 后该断言必然触发，属 fork 身份守卫的预期后果，与 usage-stats 等仓的同类处理一致，未改测试、未改 README 正文去迁就它。

## 验证边界

- 只验证了离线纯函数路径（取材构建）与上游自带测试套件；未在真实 DSH 宿主里跑端到端迁移（沙箱无宿主环境），端到端行为除取材总长外与上游 0.5.0 一致、未作其他改动。
- 修复只处理 `keep: 'newest'` 分区剩一条时的提示行溢出；`keep: 'head'` 分区原有逻辑（放不下截断标记即整段让位）本身不超预算，未动。
