# P1.1 — 离线 Memory 数据与权限基础开发报告

日期：2026-10-26（Asia/Shanghai）。本阶段仅 P1.1，未进入 P1.2。
设计依据：DSH_MEMORY_V1.0.md 第 3、5、7、16、19 节。

## 1. 前置 P0 PR 审查与基线

[P0 PR #1](https://github.com/Fz1025/DSH-Memory/pull/1) 已先完成审查，再从 Draft 转 Ready，并通过常规 Squash and Merge 合并。

审查时 head 为 acdc8ed7646055dd2668d39d1b231d9c192089de，只有一个提交、两份新增文件：
- docs/implementation/P0_COMPATIBILITY_AUDIT.md
- docs/implementation/PROGRESS.md

已读取完整变更、提交、PR 状态、审查线程、评论、check-runs 和 combined status；未发现正文凭据、Cookie、私人聊天记录、真实配置或个人机器目录。文档与总设计一致，明确保留 P0 真实空会话、有效模型/输出限制、请求预算和删除接口的运行门禁，不需要修改结论。

当时 mergeable=true、mergeable_state=clean；reviews、review_threads、讨论评论为空；check-runs=0，statuses=[]。combined status 的 pending 是没有状态条目，并非失败任务；main protected=false，必需状态检查列表为空。rulesets 查询返回 403，原因是该私有仓库套餐不支持该功能，没有将其伪称为“已读取空规则”。合并使用预期 head SHA 和常规 squash，没有强制绕过。

**P0 合并后的 main commit：39a664bc05ab49bca0201bf72c59813c3b90a597。**
合并后重新读取 main 元数据，并以此建立文档分支 docs/p1.1-offline-foundation。
P0 原设计与审计正文保持不变。合并只接受文档，不代表所有运行门禁通过。

## 2. 开发隔离与代码交付

实际代码属于 qq-bridge，在独立本地 Git worktree 开发：
- 代码基线：a49877f4e58d04e47532d44b803360d0f001160d。
- 本地分支：memory/p1.1-offline。
- 本地提交：3035d279107f2399233a37f600d5e3b6df435aa3。
- 变更范围：仅新增 src/memory/ 下 10 个文件，共 1055 行。
- 本地独立目录名称：qq-bridge-memory-p1.1；完整机器路径在用户交付消息中提供，不写入远程文档。
- 原生产工作树的已修改/未跟踪状态在开发前后比较一致，未暂存、覆盖或提交其任何改动。
- 原 origin 未改，也未向第三方 qq-bridge 推送；未创建新的用户代码仓库。代码提交仅保留本地。

独立 worktree 从已提交基线创建，不含生产目录的既有未提交功能；因为本次没有接主流程，这一隔离不会替用户丢弃或迁移那些功能。未来集成须在重新核对当前工作树后移植本次单独提交。

## 3. 模块与文件

| 新增文件（相对 qq-bridge 根） | 已完成能力 |
| --- | --- |
| src/memory/schema.mjs | 18 个核心业务/引用表、STRICT、FK/复合租户约束、唯一键与索引；FTS external-content 与增改删触发器；依赖边拒绝自环/循环 |
| src/memory/migrations.mjs | migration 1 核心结构、migration 2 FTS；版本/校验和/user_version 一致；所有待执行迁移和账本同事务；拒绝较新库/校验漂移/异步事务 |
| src/memory/policy.mjs | 严格 RequestContext；可信进程内签发标记；bot/persona/QQ namespace/域；capability 与 purpose；默认拒绝、deny 优先、当前版本 grant、来源交集和禁止私域来源洗成 public/global |
| src/memory/store.mjs | SQLite 初始化、WAL、FULL synchronous、FK、1000ms busy timeout、完整性检查、脱敏错误、合成任务快照/结果、授权单条读取、事务式逻辑删除屏障 |
| src/memory/sandbox.mjs | 只创建 OS 临时目录中的品牌化合成库，关闭后只清固定 DB/WAL/SHM 文件；不支持生产 filename |
| src/memory/index.mjs | 默认关闭 facade；不开库、不注册路由；显式开启仍只允许合成 sandbox 或内存库 |
| src/memory/errors.mjs | 稳定 MemoryError.code；不向调用者暴露 SQLite 错误正文和路径 |
| src/memory/test/foundation.test.mjs | 28 项无网络、合成人物/虚构 QQ 号与临时/内存库测试 |
| src/memory/.gitignore | 排除模块内 *.sqlite*、*.db* 与 sidecar |
| src/memory/README.md | 调用方式、可信边界、偏差和未完成能力说明 |

没有修改 package.json、依赖、主 bridge.js、角色卡、提示词、模型配置、QQ 空间发布功能或任何生产状态文件。

核心业务表：bots、personas、persons、identities、conversations、scopes、grants、subject_consents、raw_messages、sources、memories、memory_sources、memory_edges、jobs、job_sources、deletion_requests、deletion_targets、context_dependencies。另有 schema_migrations 和 memory_fts；SQLite 自动 FTS shadow tables 不算新增业务能力。

## 4. 权限与来源/删除语义

RequestContext 包含 bot_id、persona_id、actor_id、conversation_id、session_id、scope_id、audience、capabilities、denied_capabilities、policy_version、purpose、request_id、qq_identity。严格拒绝未知字段、无效枚举/数字类型/通配能力和重复能力；QQ ID 是文本，不用昵称合并身份。

createContextAuthority 的 resolver 只可放在可信服务端组合入口，接受由服务端认证的 opaque handle。只有由它签发、冻结并通过 WeakSet 标记的对象能使用数据服务；模型提交相同形状 JSON 或复制对象也会被拒绝。数据服务再核对数据库内 actor/QQ namespace/QQ ID/会话 audience。**本阶段只提供合成 resolver，未实现 QQ 真实认证；若未来服务端直接信任请求 JSON，品牌机制不能替代认证。** 有能力替换 resolver 或执行 maintenance SQL 的本机可信代码不在 ACL 防御边界。

private/group 默认只在同域、同会话、当前许可能力内使用；shared 无隐式权限；public_material/self_global 也需要明确 grant，且仍绑定 bot/persona。跨域必须有服务端加载的精确 capability/purpose grant。源域和目标域策略版本必须匹配，过期/撤销 grant 不生效，deny 优先于 allow。旧共享历史开关没有迁移成任何新权限。

合成任务仅持久化引用、源 epoch/策略版本及目标 scope epoch。提交结果在 BEGIN IMMEDIATE 事务内重读身份、权限、来源状态与快照；删除 epoch、策略或来源 grant 变化即拒绝。多来源必须全部获准；派生记忆保留全部叶子来源与 parent→child 边，不能只保留被允许的那一部分。

逻辑撤销在一个事务内：增加 scope epoch、标记来源 revoked、沿依赖图使合成记忆不可用、取消受影响合成任务、写 barrier_applied 回执与对象清单。中途失败整体回滚。读取时仍重查叶子来源和祖先状态，未用旧缓存放宽。完整性检查同时核对 FK、活跃来源依赖、删除目标和 FTS 内容一致性。

**该回执只代表逻辑不可用。** 完整正文、FTS 内容、WAL、备份可能仍留存；physicalDeletion=false、Harness cleanup=not_implemented。没有执行清理生产文件或改动 Harness 会话机制。公开/全局素材的人工审核流程未实现，测试只手工设置合成 scope/grant，不代表系统能自动审批公开化。

## 5. 设计偏差与分阶段实现

1. 第 7 节全量表按 P1.1 必需子集落地。aliases、候选、关系、成长、预算/usage/归因、tombstone 和审计业务表留到对应阶段；subject_consents/context_dependencies 目前是结构，尚未绑定告知同意、真实注入或删除服务。没有宣称完整 V1.0 schema 已完成。
2. scopes.persona_id 从可空收紧为必填。新增冗余 bot/persona 及复合 FK，raw 必须与其 scope 的 conversation 相符；阻止跨租户/角色引用。数据库 TEXT ID 允许合成可读 ID，新建结果/任务使用 UUID。
3. scope/source 增加 deletion_epoch；job_sources 增加源/域 epoch 与 policy 快照；grants 增加 effect、source_policy_version、target_policy_version，避免把 grant 自身 CAS version 当策略版本。
4. FTS 单独版本迁移；无模块时可保留核心库并禁用 FTS，不安装扩展。缺失能力分支用注入 ftsAvailable=false 的离线测试验证；本机实际 FTS5 存在。中文分词、检索、排序、MATCH 查询接口均未实现，search_terms 只在测试 SQL 中手动填充。
5. jobs 表是合成结果一致性快照，不是后台调度器/租约 worker。没有提炼或模型调用；合成 memory 文本由测试手工提供，confidence=0、epistemic_status=uncertain。
6. 删除按 scope epoch 保守失效同域在途任务；牺牲部分无关任务可用性来避免旧结果提交。后续细化不得削弱来源检查。
7. migration 版本/校验和与待执行 DDL 使用一个事务，不实现自动降级或坏库重建；P1.1 只使用临时合成库，生产迁移备份/回滚运维尚未实现。

## 6. 离线测试与需求对应

执行环境：既有系统 Node v24.15.0，node:sqlite；未安装/升级依赖。
命令（在独立代码目录）：

```powershell
node --test src/memory/test/foundation.test.mjs
```

最终结果：**28 tests / 28 pass / 0 fail / 0 skipped，exit 0**。另完成 staged diff --check、提交文件范围检查、数据库忽略规则检查和原生产 Git 状态比较。测试创建 OS 临时库并在连接关闭后清理，没有读取生产 DB/配置/聊天。

| 用户要求 | 验证结果 |
| --- | --- |
| 1 初始化 | 文件库首次初始化，WAL、FK、FTS、integrity 检查通过 |
| 2 迁移幂等 | 文件库关闭/重开重复 migrate，保留合成记忆；版本/校验和不重复执行 |
| 3 失败回滚 | 追加迁移先建表再注入非法 SQL，DDL/版本账本回滚，旧行保留 |
| 4 FK | 无父项、跨 bot/persona 来源、错误 namespace、raw/scope 会话不符均拒绝；身份唯一约束通过 |
| 5 bot/persona 隔离 | 同群号也不能跨 bot 或 persona 读取 |
| 6 私→群拒绝 | 同私聊可读，群聊上下文拒绝 |
| 7 A→B 默认拒绝 | 没有新增精确 grant 不开放 |
| 8 来源交集 | 两来源只授权一个时拒绝；全部获准后可用；任一 deny 再拒绝；跨层依赖也检查 |
| 9 policy_version | 旧上下文、旧目标任务、源域策略更新或 grant 撤销时拒绝 |
| 10 epoch | 删除后旧任务拒绝，派生记忆不可读；跨域快照也失效 |
| 11 DB 不可用 | 关闭、坏文件、WAL 写锁竞争返回 DB_UNAVAILABLE；无权限降级，坏文件不被空库替换 |
| 12 未启用 | facade 默认关闭、无开库/回调/新文件；主入口未导入 Memory且与基线零 diff |

附加通过：严格 RequestContext/JSON 伪造、purpose 区分、deny 优先、共享域默认拒绝、私域不能洗成全局、迁移 checksum/较新库拒绝、FTS 缺失降级、循环依赖拒绝、重复提交/去重事务、删除中途故障整体回滚、FTS 增改删同步及删除回执一致性。

**未完成的完整集成测试：** 实际 bridge 开关关闭回归、真实 QQ 认证/归档/撤回、Harness 会话退役、context_dependencies 真实追踪、文件物理清理/备份恢复、线上磁盘满/进程强退恢复、真实模型与预算。第 12 项只验收离线 facade 和“未接入口”的静态事实，不冒充完整 QQ 运行验收。

## 7. 当前风险、后续建议与停止点

P0 R1（压缩独立 usage）、R2（真实共享上下文撤销）、R3（请求预算/输出/模型覆盖）、R6（激活 profile/真实空会话）、R7（Harness/备份物理删除）继续保留。R4 仅新增离线规则基础，未建立真实认证/同意/管理边界。R5 仅补齐临时文件库 WAL、锁竞争、坏库拒绝、FTS 同步；不代表真实规模性能、备份恢复和物理擦除完成。

P1.2 建议限制为无模型的 consent/grants/来源登记服务契约、策略版本生命周期、撤销传播的合成适配与观察数据；不要立刻接实时消息或开启归档。若要接生产，必须先重新核对原脏工作树、移植本地提交并补 P0 运行门禁。后续可由用户建立自有代码仓库，再单独制定迁移/敏感历史审查与 remote 方案；本次没有擅自创建、改 remote 或推代码。

P1.1 新文档 PR 保持 Draft，等待用户验收；不自动合并，不进入 P1.2。
