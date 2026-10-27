# P1.1.1 — 两项缺陷修复与离线复验报告

日期：2026-10-27（Asia/Shanghai）。停止点：P1.1.1。

**结论：PASS，仅限 P1.1 离线数据与权限基础及本次两项修复。** 本轮范围未发现仍阻止此范围验收的缺陷。生产接入门禁没有解除，没有进入 P1.2。

## 1. 保留原始 FAIL 记录

P1.1 原始本地提交：3035d279107f2399233a37f600d5e3b6df435aa3。

2026-10-27 独立验收：**FAIL**。原有测试 28/28；独立边界测试 11/15，A09/A10/A11/A12 共四项失败。发现：
- F01 / 高：中间祖先 scope 撤销或策略变化后，后代仍可读取、旧任务仍可写入，完整性检查漏检。
- F02 / 中：FTS 补迁移没有回填历史记忆，MATCH 漏数据，索引一致性检查失败。

上述历史结论仍然成立，没有覆盖原始开发报告或删除本地 FAIL 报告/日志。本报告记录后续修复后的状态。

## 2. 本地交付与范围

分支：memory/p1.1-offline。新本地提交：**a02b28d3f0b3f454511d7b59b3842b698ae379e1**。

仅修改/新增 src/memory/ 下 6 个文件，439 additions / 3 deletions：
- store.mjs：逐层依赖验证、快照、读取/提交重查、完整性与历史可用性分离。
- policy.mjs：重查原派生路径的精确 extract 授权，保持 deny 优先和租户隔离。
- migrations.mjs：在原事务中应用独立校验和补丁账本。
- repairs.mjs：新增依赖快照表及一次性 FTS 重建/一致性检查补丁。
- test/p111-regression.test.mjs：22 项新增回归测试。
- README.md：安全升级语义、历史数据限制和测试方法。

原 schema.mjs、迁移 1/2 SQL、原 28 项测试、原独立 15 项测试均未更改。未修改 bridge、package.json、角色卡、模型或 QQ 空间功能。原生产工作树的 Git 状态、差异摘要、未跟踪文件哈希及 origin 前后相同；代码仅本地提交，没有向第三方仓库推送。

## 3. 缺陷 A 的修复

原因是原实现只递归汇总叶子来源、检查祖先 memory 自身状态，没有保存并核对中间域的完整依赖约束。

新生成的 job/memory 保存独立 provenance 快照，包含域 policy/epoch、来源 epoch 与域快照、精确来源集合及 parent ID/version。读取和任务提交检查所有祖先和来源，重查每一级域的当前状态、策略与 epoch，逐条重放原来的来源→派生域、父记忆域→子记忆域授权。撤销、到期或 deny 均拒绝。

这里区分两个权限：原派生路径必须持续保有 extract 授权；当前调用者仍须对请求记忆及全部叶子来源持有 recall 授权。重放原派生路径不会给 recall 调用者签发 extract 能力，也不会用 extract 代替当前 recall。此语义允许原独立测试的合法链路继续工作，同时中间授权失效会阻断后代。

读取、任务重查与写入均在 BEGIN IMMEDIATE 事务内，另一个写入者不能插入校验和提交之间。锁竞争安全拒绝，释放后再次查当前撤销状态。保留原循环插入保护，并加入迭代遍历和完整性图检查；超过 1000 节点安全拒绝。删除来源链接、祖先边或 job 来源快照会与证据集合不符而被拒绝，不能缩小交集绕过检查。

完整性输出的 ok 表示结构一致，dependenciesAvailable/unavailable 单独报告历史策略/epoch/grant 撤销和缺少旧快照。合法逻辑删除保留的 revoked 历史行不被误称为数据库损坏；缺失证据、循环和未传播的来源撤销导致活跃记忆违反约束仍是 INTEGRITY_FAILED。

**旧 memory/job 缺少新增快照时，保留原行但返回 PROVENANCE_UNVERIFIED。** 无法从当前状态可靠推算历史 epoch，因此没有自动补造可信快照或恢复旧记忆。这是安全拒绝的兼容策略，需要后续单独决策，不是数据丢失或本次引入新业务。

修复位置：store.mjs:50（图检查）、68（快照）、77（快照核对）、103（完整祖先验证）、156（完整性）、213/238/247（任务及结果快照）；policy.mjs:61（原派生路径规则）。

## 4. 缺陷 B 的修复

原 external-content FTS 迁移只有表和未来写入触发器，没有为旧行建立索引。

保持原迁移 1/2 的 SQL、校验和、schema_migrations 历史记录及 core user_version 不变。新增独立 schema_repairs 账本：
- p1.1.1-provenance-v1：创建 memory_provenance、job_provenance。
- p1.1.1-fts-rebuild-v1：执行 FTS rebuild，随后 integrity-check rank=1，全部成功后才记录补丁完成。

旧 core v1、新空库和已执行旧迁移的 v2 库均有明确路径。FTS 不可用时延后 FTS 补丁，原生能力可用后再执行。原迁移、补丁 DDL、索引重建/验证及账本在同一事务，失败整体回滚；再次执行不重复修复或写入记忆。已完成补丁后若索引另外损坏，不会通过重写补丁账本掩盖，打开时仍会做一致性检查并安全拒绝。

位置：repairs.mjs:4-18；migrations.mjs:59-76。业务 schemaVersion 仍为 1/2；是否包含 P1.1.1 修复须同时核对 schema_repairs，不能只看 user_version。

## 5. 执行环境、命令与结果

环境：Windows win32 x64，PowerShell，Node v24.15.0，node:sqlite / SQLite 3.51.3。本机原生 FTS5 可用。没有安装/升级依赖。

命令（独立代码目录；边界测试路径指原独立验收文件）：

~~~powershell
node --test --test-reporter=tap src/memory/test/foundation.test.mjs
node --test --test-reporter=tap <原独立验收目录>/boundary.test.mjs
node --test --test-reporter=tap src/memory/test/p111-regression.test.mjs
~~~

| 测试集 | 通过 | 失败 | 跳过 | exit |
| --- | ---: | ---: | ---: | ---: |
| 原有 foundation | 28/28 | 0 | 0 | 0 |
| 原独立边界（未修改） | 15/15 | 0 | 0 | 0 |
| 新增回归 | 22/22 | 0 | 0 | 0 |
| 合计 | **65/65** | **0** | **0** | — |

原独立 A09/A10/A11/A12 已从 FAIL 变为 PASS，其他 11 项继续通过。未删除、跳过或弱化原测试。

新增覆盖：
- R01-R05：三级祖先域撤销、policy bump、grant 撤销、deny 优先、祖先 epoch。
- R06-R09：移除证据、边、job 来源快照及循环损坏拒绝。
- R10-R12：合法历史逻辑撤销、双连接写锁撤销竞态、DAG 菱形和 reopen/recall-only 调用。
- R13-R15：旧 core/v2 历史数据补索引、幂等和校验和保持、文件 reopen 与旧 provenance 拒绝。
- R16-R17：core v1/v2 修复失败，DDL/索引/账本/原始记忆整体回滚，随后可重试。
- R18-R20：补丁 checksum 漂移拒绝、模拟无 FTS 后补迁移、新空库。
- R21-R22：无旧快照也不掩盖结构循环、祖先 grant 到期与来源 epoch。

开发中首轮新增回归为 19/20：策略失效先被相邻旧 grant 判成 FORBIDDEN，断言期望 POLICY_CHANGED。调整实现为先遍历核对全部快照，再重放授权路径，未放宽断言；最终扩展为 22/22。原始独立验收 FAIL 的四项结果仍保留。

修复后重新审查完整差异，核对跨 bot/persona、私→群/跨群、public 来源限制、deny、缺失依赖、事务原子性和默认关闭边界；本轮未发现新的范围内阻断项。git diff --check 通过。

## 6. 证据与验收界限

完整 TAP 日志、代码 patch、状态/哈希比较证据和本地报告均留在用户本地验收目录。远程只追加本脱敏报告，不上传业务源码、patch、数据库、真实配置、聊天数据或含机器路径的原始日志。

本报告的 PASS 是实际运行的本地 SQLite/原生 FTS 操作，加上全部合成身份和合成数据的离线权限测试。无 FTS 分支和撤销变化是故障/状态注入模拟。

以下没有变成 PASS：
- 真实 QQ 身份认证、真实同意/授权管理、消息归档/召回和生产主流程运行回归。
- Harness 旧上下文清理、物理删除、备份恢复/物理擦除。
- 真实模型调用、usage/硬预算、空会话及其他 P0 运行门禁。
- 生产升级、生产性能、磁盘满或断电恢复。

模块仍默认关闭，入口未接 Memory。所有测试仅临时或内存合成库。未读取真实聊天或生产库，未调用 QQ/模型 API，未修改运行服务，未启用生产 Memory。

## 7. 决策建议与停止点

建议将 **P1.1 离线基础的当前验收状态更新为 PASS（经 P1.1.1 修复）**，同时保留原始 FAIL 历史。

建议用户审阅此次追加报告后，将仅含文档的 PR #2 转 Ready 并合并；合并只接受文档记录，不代表业务代码已推送或生产集成获准。本次 PR 继续保持 Draft，没有自动改 Ready、批准或合并。

不自动进入 P1.2。P0 及生产接入门禁仍保留。
