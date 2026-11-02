# ShareIt 代码评审报告（2026-11-02）

评审方式：只读评审（research + 实测）。除本报告外未写入/修改仓库任何文件，未执行 git commit/push。
评审基线：HEAD = `faa0755 docs: add git commit message convention`（共 7 个 commit）。

## 1. 仓库定位与规模概览

- 定位：隐私优先的家庭图片/视频分享应用，**当前处于纯文档阶段**（README.md:22、AGENTS.md:11、docs/TECH_STACK.md:5 三处一致声明"没有任何代码"）。
- 规模：`git ls-files` 共 8 个跟踪文件、合计约 405 行（`wc -l $(git ls-files)`）：
  - Markdown 7 个（README、AGENTS、CLAUDE、handoff、MIGRATION_CONVENTION、git-commit-message、docs/TECH_STACK）
  - Bash 1 个：`tools/check-migrations.sh`（145 行，全仓唯一可执行代码，git index 模式 100755）
- 语言分布：Shell ~145 行，其余为文档。无 package.json / pyproject / go.mod，无源码目录、无测试、无 CI 配置（无 `.github/`）。
- 结论：这是一次**文档治理 + 唯一工具脚本**的评审，不存在应用层代码可审。"整体架构与模块边界""数据模型"两维度的对象尚不存在，相关结论以"未开始"如实记录，不做推测性评价。

## 2. 结论摘要

| 级别 | 数量 |
|---|---|
| P0 | 0 |
| P1 | 0 |
| P2 | 3 |
| P3 | 5 |

最关键的 3 条：

1. **P2-1**：`tools/check-migrations.sh` 注释声称动词检查是"non-blocking"警告（第 119 行），实际被计入 FAIL 导致 `exit 1`——实测复现（见 §4 m2 用例），注释与行为矛盾，未来接入 CI 会误拦合法迁移。
2. **P2-2**：仓库无 `.gitignore`（根目录 `ls` 与 `git ls-files` 均无），而 AGENTS.md:21 明确禁止提交密钥与真实照片/视频——缺少最基本的防误提交兜底。
3. **P2-3**：checker 未校验 header 中 `-- Migration NNN:` 的编号格式及其与文件名编号的一致性（第 101 行只 `grep -q -- "-- Migration"`），实测无编号 header 判为通过（§4 m3 用例），与脚本注释宣称的检查项不符。

## 3. 逐条发现

### 维度 1：整体架构与模块边界

- 无代码架构可审。文档间职责划分清晰一致：产品定义归 `handoff.md`、选型待决策项归 `docs/TECH_STACK.md`、agent 约定归 `AGENTS.md`、`CLAUDE.md` 仅做引用保持单一事实源（CLAUDE.md:3-5）。未发现矛盾。
- 迁移规范（MIGRATION_CONVENTION.md）与检查脚本（tools/check-migrations.sh）成对存在，但脚本未被任何自动化引用（无 CI、无 npm scripts、`.git/hooks/` 无激活 hook——实测 `ls .git/hooks | grep -v sample` 为空），规范的执行目前完全靠人工自觉。归入 P3-5。

### 维度 2：正确性与潜在 bug

- **P2-1（blocking 警告 bug）**：`tools/check-migrations.sh:119-135`。非白名单动词被追加进 `issues` 数组（124 行），随后 `if [ ${#issues[@]} -gt 0 ]`（129 行）使 `FAIL` 加一并最终 `exit 1`（143-144 行）；但同行注释写的是 "verb-first warning (non-blocking)"（119 行），且末尾汇总文案区分了 FAIL 与 WARN（139-141 行），说明设计意图是警告不阻塞。实测 m2 用例（合法编号 + 动词 `initial`）：输出 `x 002_initial_seed.sql - warning: ...`，`exit=1`。
  - 影响：脚本一旦接入 CI/本地钩子，任何动词不在白名单内的合规迁移都会被误判失败；同时汇总行 "OK ... (N warning(s))" 在有警告时永远打不出来（死代码路径）。
  - 修复建议：二选一——(a) 动词警告不追加进 `issues`，只 `WARN++` 并单独打印；(b) 维持阻塞语义，删掉 "non-blocking" 注释并把该检查改名为 error。推荐 (a)。
- **P2-3（header 编号未校验）**：`tools/check-migrations.sh:101-103` 仅检查前 12 行中存在字符串 `-- Migration`，不校验 `NNN` 格式，也不与文件名编号比对。实测 m3 用例（header 写 `-- Migration something without number`）：判定通过，`exit=0`。
  - 影响：迁移文件 header 编号与文件名脱节（如文件名 `005_` 而 header 写 `Migration 003`）不会被发现，削弱 MIGRATION_CONVENTION.md 第 3 节的可执行性。
  - 修复建议：改用正则 `^-- Migration ([0-9]{3}):` 提取编号并与文件名 `NNN` 比对，不一致即报 issue。
- **P3-3（Date 校验过松）**：第 105 行正则 `[0-9]{2}:[0-9]{2}` 不限制 24 小时/分钟范围。实测 `-- Date: 2026-09-10 29:99` 判定通过（`exit=0`）。影响极小（格式错误肉眼可见），属加固项。修复建议：在正则匹配后追加范围判断（小时 ≤23、分钟 ≤59）。

### 维度 3：安全

- 全仓 grep 常见密钥模式（`password|secret|api_key|token|PRIVATE KEY|sk-|ghp_|AKIA...`）：**无匹配**，git 历史中亦无（文件总量小，已全量检索）。
- **P2-2（无 .gitignore）**：根目录无 `.gitignore`（`ls -la` 与 `git ls-files` 确认）。AGENTS.md:21 规定"不要提交任何密钥、测试用的真实照片/视频"，但没有 .gitignore 兜底，后续搭建骨架时 `.env`、`node_modules/`、媒体素材存在被误提交的真实路径。
  - 修复建议：立即添加通用 `.gitignore`（`.env*`、`node_modules/`、`dist/`、`.DS_Store`、媒体临时目录等）。
- 无认证/授权、注入、越权面可审（无任何服务端/客户端代码）。标注"未确认（对象不存在）"。

### 维度 4：数据模型与迁移一致性

- 数据模型尚未设计（docs/TECH_STACK.md:16-22 列为待决策项），无迁移目录（`supabase/migrations`、`db/migrations` 均不存在），无已部署 migration 可被覆写的问题。
- **P3-2（规范与脚本动词清单不一致）**：MIGRATION_CONVENTION.md:19 列举动词 `create_/add_/alter_/fix_/drop_`；脚本白名单（123 行）额外放行 `set|update|rename|delete|migrate`。两者谁为准不明确。修复建议：在规范中把动词清单声明为"示例"或与脚本白名单对齐。

### 维度 5：错误处理与失败模式

- 脚本无迁移目录时输出明确错误并 `exit 1`（38-40 行），实测符合预期；目录参数不存在时报错退出（43-46 行）；空目录正常提示并 `exit 0`（55-58 行）。`set -u` 已启用（18 行），路径变量均有引号包裹，glob 用 `nullglob` 防空匹配。未发现会导致误报通过的静默失败路径。
- 一个边界观察（非缺陷）：`FILES` 依赖 shell 字典序排序，因强制三位零填充编号，排序与数值序一致，当前无风险。

### 维度 6：测试覆盖与验证门真实性

- 无测试文件（`git ls-files` 确认），checker 脚本本身无测试夹具。本次评审用 /tmp 临时夹具实测了 6 个用例（见 §4），测后已 `rm -rf` 清理，仓库未受污染（`git status --porcelain` 为空）。
- **P3-5（验证门未接入自动化）**：checker 是全仓唯一验证门，但无 CI（无 `.github/workflows`）、无 pre-commit hook（`.git/hooks/` 仅 sample）、无 npm scripts。当前阶段（尚无迁移文件）影响小，但迁移开始出现前必须接入，否则规范形同虚设。

### 维度 7：依赖与构建健康

- 无任何包管理器清单与锁文件，无构建流程，无第三方依赖。`bash -n tools/check-migrations.sh` 语法检查通过。shellcheck 本机未安装（`command -v shellcheck` 为空），静态 lint 未执行——标注"未确认"。

### 维度 8：明显性能热点

- 不适用。脚本为线性遍历，145 行规模无热点。

### 维度 9：文档与实现一致性

- **P3-1（AGENTS.md 状态描述过时）**：AGENTS.md:11 称"仓库目前只有 `handoff.md` 和 `docs/TECH_STACK.md`，没有任何代码"，但实际已存在 README.md、MIGRATION_CONVENTION.md、git-commit-message.md 和 `tools/check-migrations.sh`（README.md:22 的表述"只有文档"与现状一致，AGENTS.md 未同步）。修复建议：AGENTS.md 状态段改为与 README 一致的表述并列举现有文件。
- **P3-4（文件权限异常）**：`MIGRATION_CONVENTION.md` 工作区权限为 `-rw-------`（600），其余文件均 644（`ls -la` 输出）。git index 中全部为 100644（`git ls-files --stage` 确认），不影响仓库内容，但工作区权限异常提示曾被手动 `chmod`。属观察项，无需处理或恢复为 644 保持观感一致。
- 其余一致性良好：README "下一步" 与 handoff.md "下一步计划" 吻合；handoff.md 勾选状态与 TECH_STACK "尚未选型"一致；落地页指向 `shareit-landing` 在两处一致。

## 4. 验证门实际执行结果

仓库无 lint/test/build 命令（无 package.json 等），可执行的验证门只有 checker 脚本本身。以下为真实执行输出摘要（夹具建于 /tmp，用后已删除）：

| # | 命令 | 结果 |
|---|---|---|
| 1 | `bash -n tools/check-migrations.sh` | 通过（SYNTAX OK） |
| 2 | `./tools/check-migrations.sh`（无参数，仓库现状） | 输出 `x No migration directory found (supabase/migrations or db/migrations).`，`exit=1`——符合设计（仓库尚无迁移目录） |
| 3 | m1：2 个完全合规文件 | `OK All 2 migration file(s) pass (0 warning(s)).`，`exit=0` |
| 4 | m2：合规文件 + 动词 `initial`（非白名单） | 报 `warning: description should start with a verb`，**`exit=1`**——证实 P2-1 |
| 5 | m3：header 为 `-- Migration something without number` | 判定通过，`exit=0`——证实 P2-3 |
| 6 | m4：编号 001→003 跳号 | 报 `numbering: skip detected (1 -> 003), expected 2`，`exit=1` |
| 7 | m5：`-- File:` 与实际文件名不符 | 报 `header: '-- File:' is '001_wrong_name.sql', ...`，`exit=1` |
| 8 | Date 用 `29:99` | 判定通过，`exit=0`——证实 P3-3 |

未执行项及原因：shellcheck（本机未安装）；npm test / build / lint（无 package.json）；CI（无配置）。未启动任何常驻服务，未安装新依赖，未 kill 任何进程。

## 5. 修复优先级建议

1. **搭建骨架前（最先做）**：添加 `.gitignore`（P2-2）——成本极低，防密钥/媒体误提交。
2. **第一批迁移落地前**：修复 checker 的 blocking-warning 矛盾（P2-1）与 header 编号校验缺失（P2-3），并把脚本接入 pre-commit 或 CI（P3-5），同时对齐动词清单口径（P3-2）、按需加固 Date 范围（P3-3）。
3. **随手清理**：同步 AGENTS.md 状态描述（P3-1）；`MIGRATION_CONVENTION.md` 权限恢复 644（P3-4，可选）。
