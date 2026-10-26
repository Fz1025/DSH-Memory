# DSH Memory V1.0 — P0 兼容性审查

日期：2026-10-26（Asia/Shanghai）。范围：实施前只读审计；未实施 P1～P6。

设计基线：[DSH_MEMORY_V1.0.md @ 16d7a7839c2cd3bc0cfbeced872e94c7370d0da3](https://github.com/Fz1025/DSH-Memory/blob/16d7a7839c2cd3bc0cfbeced872e94c7370d0da3/docs/DSH_MEMORY_V1.0.md)。已读取全文，重点核对第 2、3、12、14、18、19 节；沿用原有审查，仅补查安装包、调用路径、撤销路径和离线基础能力。

## 1. 结论与验收状态

**P0 审计交付完成，有保留；设计第 19 节的完整运行门禁尚未通过。** 在本次“测试仅限无网络、无真实账号数据”限制下，没有创建真实 Harness 空会话、读生产投影或操作生产 inbox。不得将静态兼容与合成测试通过写成真实链路验收通过。

现有组件具备接入独立 Memory 模块的基础：会话 RPC 包装、会话级令牌、提示构造、隔离 compaction、幂等 Token 账本、Node 内置 SQLite/FTS5。它们不提供完整的 Memory 权限、来源图、撤销传播、持久任务队列或硬预算。

P1 只可在另行下达阶段指令后启动。建议从独立数据/权限骨架和合成数据开始；生产归档、召回、模型调用均保持关闭。当前结论不是生产接入放行。

证据级别：**静态已验证**＝安装包/源码；**离线已验证**＝本次合成运行；**运行元数据已验证**＝监听端口、进程入口；**未验证**＝真实接口、账号或 provider 行为。源码行号对应审计时工作树，末尾 SHA-256 可识别漂移。

## 2. 环境与版本锁定

为避免上传个人机器目录，本文使用逻辑根：
- B：本地 DeepSeek-QQ-Harness/qq-bridge。
- H：由 QQ 链路启动脚本指向的 Harness 安装目录下 node_modules/@deepseek-ai。
- S：本地 SnowLuma 发行目录。
- 启动脚本：QQ 链路根目录 scripts/start-qq-chain.ps1。

| 组件 | 本次确认 | 证据及边界 |
| --- | --- | --- |
| qq-bridge | package 0.2.1-r1；HEAD a49877f4e58d04e47532d44b803360d0f001160d | B/package.json、git rev-parse；运行进程命令入口匹配 bridge.js。工作树含大量已修改/未跟踪文件，HEAD 不代表实际文件内容 |
| Harness CLI | 0.2.1-rc.2 | H/dsh/package.json；3080 监听 PID 的命令入口匹配启动脚本所指 dsh/lib/bin.js，执行文件为系统 Node。未调用版本 RPC；已加载内存代码与当前磁盘字节完全一致仍未验证 |
| agent/persona | 均为 0.2.1-rc.2 | H/dsh-agent、dsh-persona/package.json |
| compaction 家族 | compaction、basic、tool-result-pruner、image-offload 均为 0.2.1-rc.2 | 对应 package.json；不代表每个插件均已启用 |
| LLM、持久化、查询 | llm-deepseek、session-persistence、session-persistence-jsonl、session-query、session-query-sqlite 均为 0.2.1-rc.2 | 对应 package.json |
| bridge Harness proxy | 0.1.1-rc.2 | B/node_modules/@deepseek-ai/dsh-host-apiproxy/package.json；版本号不同不能单独判为不兼容 |
| SnowLuma | runtime 2.2.0 / win32-x64 | S/package.json、release.json；3001 监听进程使用发行包 node.exe、入口 index.mjs。未请求真实 get_version_info |
| SnowLuma SDK/MCP | 均为 1.14.20 | B/node_modules/@snowluma/{sdk,mcp}/package.json；这是客户端版本，不能当作 runtime 版本 |
| Node | Harness/bridge 系统 Node v24.15.0；SnowLuma 自带 v24.22.1 | 两个 node.exe --version。bridge 要求 >=22.13.0；SnowLuma 要求 ^24.22.1 或 >=26.12.1。两个运行时各自满足，不能让 SnowLuma 改用系统 Node |
| profile 声明 | web/package.json 声明 qq-mode-console、qq-agent-presets | 只读包依赖，不读含凭据配置。激活 preset、有效模型、压缩配置和工具表均未从生产端核实 |

没有发现仅凭上述包版本就必须升级的证据；也不能据此宣称 SDK 对 SnowLuma 2.2.0 所有动作兼容。实际模型 ID、provider 路由和有效 maxTokens 未验证，未读取角色卡或模型凭据配置。

审计开始先检查 Git：QQ 链路外层目录和 SnowLuma 发行目录不是 Git 仓库；B 是独立 Git 仓库，origin 保持 Derpyu520/qq-bridge，原有脏工作树未暂存、提交、清理或回滚。审查文档放在独立 DSH-Memory-audit 目录，仅文档通过 GitHub 连接提交到用户自己的文档仓库。

## 3. 接口、会话生命周期与上下文

| 接入点 | 已验证证据 | 兼容性判断 |
| --- | --- | --- |
| Harness 通信 | B/src/dsh-client.js：wrapArgs、ensureAuth、callUnary、_remoteMuxGenerator；session/create、prompt、projections、page、search、follow | 启动授权换 Cookie、slash RPC、remote.mux 包装存在；prompt 自动补 requestId。仅审源码，未读取 Cookie、未握手 |
| 聊天请求 | B/src/bridge.js:2177、2240、9330、11293 的 api.sessions.prompt | 多条入口，不能只在一个唤醒函数加预算/依赖登记；重试与压缩在 Harness 内另有路径 |
| 会话映射/退役 | B/src/bridge.js:2754–2840，sessionPolicy、retireSession、stopRetiredSessionWork；B/src/dsh-client.js stopSessionWork | 本地撤销映射/轮换 token，清 inbox→cancel→archive；archive 只隐藏，不是删除 |
| 退役失败边界 | stopRetiredSessionWork 使用异步任务；错误只记录，不向调用方提供完成屏障 | 普通权限切换已有保护，但 Memory 撤销不能把调用 retireSession 当作远端清理完成。需可等待的结果及失败闭锁 |
| 退役早退 | retireSession 无 sessionId 时直接 return，位于 token 轮换之前 | 未映射会话的旧 token 失效语义不能只依赖此函数；Memory policy 需要独立撤销版本/能力校验，不在 P0 修改 |
| 路由模块 | B/src/core/session-router.mjs；bridge 没有该模块导入 | 不将纯路由模块能力当作主流程已全量接入；应以 bridge 当前真实入口为准 |
| 工作记忆注入 | B/src/bridge.js:8965 formatMemoryV2、10461–10510 提示组装，初始提示只在 bootstrap 注入 | 未来检索应单独标识块并记录 session/source/policy 依赖；不可改旧 memory 语义或覆盖 roleLine |
| 核心人格机制 | H/dsh-persona/lib/index.js:21–46 向 systemPrompt 注册 scoped prefix/suffix | 可复用会话隔离。核心角色仍归管理员管理，不把长期记忆写入固定人格 |
| 压缩 | B/plugins/qq-agent-presets/presets/qq-chat-v2.patch.yml 末尾 compaction group / isolate.compaction、toolResultPruner；thresholdChars=50000、headChars=45000、tailChars=2000 | basic、command-compact、pruner 的声明存在。字符裁剪不是 Token 硬预算；摘要不等于获授权长期事实 |
| 压缩存储 | H/dsh-compaction-basic/lib/index.js:629–648，summary/usage 和 shadowedRange 入事件日志 | 原始日志与压缩摘要可能同时留存；来源撤销必须处理整个派生上下文，不能只删未来 Memory 表 |
| 工具限制 | B/plugins/qq-agent-presets/qzone-tool-restrict.mjs:4–33 | tools/change 后恢复专用 allowlist，发现前 fail-closed；未来 Memory preset 要采用同类机制，但不扩充空间工具面 |

跨会话/模式的当前会话策略不包含未来 Memory policy_version 或 persona 核心版本。新增系统必须有明确依赖与撤销协议，不能认为现有模式切换会自动处理记忆授权改变。

## 4. DeepSeek 请求、输出限制与计量

### 4.1 可用接口与限制

H/dsh-llm/lib/index.js:2393–2397 提供 **llm/stream waterfall**，是比 bridge prompt 更靠近实际请求的潜在拦截点。安装包支持 prepared call 的配置一致性检查；具体插件作用域、执行顺序及是否覆盖所有请求尚未运行验证。

H/dsh-llm-deepseek/lib/index.js:1719 将 options.maxTokens ?? model.maxTokens ?? connection.maxTokens 映射为 max_tokens；:1821–1834 解析 Messages input/output/cache-read/cache-creation；:2217 发出 /messages 请求。**这是安装包的 Messages 路径证据，不是当前真实账号的有效协议证明。** 普通聊天的桥接 prompt 调用未传逐任务 maxTokens；不能把“适配器支持”写成“Memory 已能逐请求强制”。

H/dsh-compaction-basic/lib/index.js:282–336 的 summarizeWithLlm 使用 ctx.llm.stream，并有独立 maxTokens/provider/model 选择。未来“同一管理员所选模型”需同时约束压缩、提炼和反思，不能只检查主聊天 selectModel。H/dsh-llm-retry/lib/index.js:137–173 记录重试；DeepSeek 适配器 :2169–2241 还可能在文件引用失效后内部重发 /messages。**一次 llm/stream 不必等于一次 HTTP 尝试**，预算必须定义覆盖边界；文本专用后台可以缩小路径，但仍需验收。

### 4.2 现有账本可复用，但不是全量账单

B/src/token-ledger.js:80–90、393 起：assistant/message、assistant/attempt 提取 usage；session/turn/step/retry + seq 幂等；快照基线覆盖、重连不双计。B/src/bridge.js:2491–2526 接入事件与 tokenUsage 投影。离线账本测试通过。

**新增确认的计量缺口（R1，高）：**
- H/dsh-compaction-basic/lib/index.js:633–648 将独立摘要调用 usage 写入 compaction/summary。
- H/dsh-token-meter/lib/index.js:396–407 的 tokenUsage 投影仅接受 assistant/message、assistant/attempt。
- B/src/token-ledger.js 的 usageOfEvent 同样不接受 compaction/summary。
- 因此当前这两个折叠路径不会直接计入摘要事件的独立 usage。不能把投影“权威”扩大为 provider 全部调用总量；是否有其他实际运行层补计尚未验证。需独立补齐并防重复，之后才能宣称含压缩的总量。

B/src/model-prices.js normalizeUsage 对缺失/非正数返回 0；适合既有展示但不满足 Memory 的 unknown/uncertain 结算要求。Memory 应保留 wire_protocol、normalization_version、request_id 和分类；缺 usage 不可按免费或自动退款处理。reasoning 不再次加入四桶总量，Messages 互斥缓存桶与其他协议须分别归一化。

空间使用独立 QzoneDshRunner 客户端（B/src/bridge.js:2386、B/src/qzone-dsh.js:65–101）；不能假设主 bridge 账本覆盖它、黑话学习或其他客户端。价格测试只说明本地固定表计算自洽，本次未查实时价格/费率，也未核对真实账单。

## 5. 工作记忆、共享历史与权限

当前工作记忆是每会话 social-v2 状态中的 activeTopics、pendingThoughts、memberImpressions；B/src/bridge.js:8997–9062 使用字符截断、数量限制、时间清理，印象 target 为调用参数字符串，不是可信全局 person_id。可保留现状；不能自动将昵称印象变成长期人物事实，不能无来源双写。

共享历史 B/src/shared-history.js:9–38 有独立共享组、重叠组拒绝、来源与目标双端准入；B/src/bridge.js:4873–4906 绑定调用者 token，限制 GET、count 1～50，映射固定 OneBot 历史动作。它从 SnowLuma 查询，不是本地完整归档。共享权限不自动授权 archive/extract/recall/publish。

权限可复用基础：B/src/bridge.js:3196 agentTokenOk、v2SessionAllowed；:3351 起 x-agent-call 集中关卡；:3482–3485 管理端拒绝 agent 头。Memory 管理 API 必须延续管理/agent 分离，而不是把会话令牌升级成管理员。closed-agent 无会话令牌的既有特例只适用于当前工具；未来 Memory 无 RequestContext 应拒绝，不继承为通用豁免。

**新增确认的撤销缺口（R2，高）：** B/src/bridge.js:3497–3513 保存共享设置，做版本比较、备份、写配置、更新 cfg 与审计，然后返回；该路径没有退役已读取历史的会话、删除已注入上下文或失效派生物。新读取会被 historyTargets 拒绝，但旧 Harness 上下文可能继续持有已共享内容。不能将开关关闭声称为撤销已知内容。Memory 必须补依赖图、policy_version、任务失效、撤销屏障和可确认的会话重建。

威胁模型补充：

| 威胁 | 现有边界 | Memory 所需边界 |
| --- | --- | --- |
| 模型伪造 key/来源、把群 A 内容带到 B | 会话 token + 准入；共享组只读例外 | 服务端建立 RequestContext；bot/persona/scope/purpose 分离；叶子来源权限取交集 |
| 撤销后旧摘要/缓存继续泄露 | 新请求检查权限；旧上下文未追溯 | context_dependencies + epoch；完成屏障前禁止受影响记忆注入 |
| 私聊成长洗成公共人格/空间素材 | 空间工具独立 allowlist | 私聊派生仍属私聊域；仅已审核 public_material；空间不挂 Memory 通用检索 |
| 群友借 agent 工具管理权限/删除别人数据 | x-agent-call、agent-token、管理路由隔离 | 新 capability 不默认授予；可信本人身份来自事件；模型参数不能构成授权 |
| 同名/跨账号误合并 | 现有印象是字符串键 | QQ ID 文本 + account_namespace；同人不等于可跨域披露 |
| 旧 worker/恢复备份复活删除资料 | 现有流程无统一 Memory epoch | 来源图、删除 epoch、CAS/租约所有者校验、恢复先重放 tombstone |

本机管理员/closed-agent 全本地工具权限不是这个逻辑 ACL 的防御对象；Memory 不能对持有磁盘与管理权限的操作者承诺隔离。普通 SQLite 也不提供文件静态加密。

## 6. SQLite、FTS5、队列与清理

| 能力 | 本次结果 | 边界 |
| --- | --- | --- |
| 内置驱动 | 系统 Node v24.15.0 的 node:sqlite / DatabaseSync 可用，SQLite 3.51.3 | 不安装依赖；未来 Memory 若换 Node 必须重验 |
| FTS5 | 内存库建表、MATCH、JOIN scope 过滤通过 | FTS 不是 ACL；这只是合成查询，不是已完成 Memory 检索服务 |
| 一致性 | external-content + insert/update/delete 触发器、更新旧词消失、删除词消失、integrity-check 通过 | 生产表及迁移未创建 |
| 中文 | 合成“学习 Java”用 unicode61 搜单字“学”命中 0 | 支持设计的中文二元 terms + 单字/短昵称实体回退，不能直接使用默认分词承诺召回 |
| 事务/安全删除语法 | BEGIN IMMEDIATE、PRAGMA secure_delete=ON、FTS secure-delete=1 可执行 | 内存库没有 WAL/空闲页/备份；不证明物理擦除 |
| Harness 查询库 | H/dsh-session-query-sqlite/lib/index.js:50、85–136、760–788 已有 persisted/live FTS 与索引维护 | 只借鉴实现，不直接打开/改写其生产库；内部 _deleteSession 删除查询索引不等于删除持久会话 |
| Harness 持久化 | H/dsh-session-persistence-jsonl/lib/index.js 提供写锁、flush、JSONL/压缩日志 | 所审接口未确认受支持的细粒度物理删除协议；不要把临时文件清理或 Map.delete 当删除 API |
| 后台复用 | B/src/slang-learner.js、qzone-service.js、qzone-dsh.js 有候选/定时/持久状态及任务收尾 | 未发现可直接替代设计 jobs 表的统一持久租约队列；这些业务状态机不可直接共用私人来源或视为统一 exactly-once 保证 |

尚缺：Memory schema/migration、sources 依赖图、consent/grants、bot/persona 域、删除传播、context_dependencies、FTS 受限查询构造、持久 jobs/lease/CAS/epoch、budget reservation/uncertain、受控备份恢复与脱敏面板。这些是后续交付，不在 P0 补实现。

## 7. 本次验证记录与未验证清单

所有测试无网络调用，无真实账号数据读取；没有运行 bridge 主入口、setup、在线 E2E 或历史控制台测试。

| 执行 | 结果 | 证明范围 |
| --- | --- | --- |
| B 中 node scripts/test-token-usage.mjs | exit 0，全部脚本断言通过 | 合成账本覆盖、重试、快照、基线、截断日志、重启载入、图表汇总；费率断言只验证现有常量 |
| B 中 node --test scripts/test-dsh-inbox-projection.mjs | 12/12，exit 0 | 合成 inbox/control 帧解析、itemId 和结构错误；不证明生产队列已清空 |
| stdin 合成共享权限探针 | 6 个断言通过 | group:1001/1002、private:2001；配对、双端准入、未授权私聊、关闭、重叠组、超大分页；仅导入 shared-history.js |
| stdin 内存 SQLite 探针 | exit 0 | DatabaseSync(':memory:')；建主表/FTS/触发器，事务插两域合成记录，MATCH+scope、更新删除、integrity-check、安全删除语法；中文单字命中 0 |
| 运行元数据 | 3080、3001 监听，入口匹配 | 无 HTTP/RPC，无 Cookie/令牌读取；仅定位实际进程和安装 |

stdin 探针未落盘到生产目录，也未新增功能或测试文件。本次不是完整回归，不宣称现有所有功能全绿。

明确**未验证**：
1. 真实空会话的 create/follow/tokenUsage/inbox 清空/cancel/archive、preset 工具表及 MCP 延迟发现。
2. 当前 profile 实际启用查询/持久化/压缩服务，实际模型/协议/输出上限，provider reasoning 和缓存计费语义。
3. llm/stream 拦截覆盖工具循环、retry、compaction、内部 HTTP 重发；同模型选择与完整输入 Token 上界。
4. SnowLuma 2.2.0 历史动作真实返回、撤回/删除行为、SDK 补丁的全部兼容性。
5. WAL、文件锁竞争、满盘、损坏、备份恢复、FTS/WAL/SSD 残留清理与机器规模性能。
6. 新 Memory 权限矩阵、删除竞态、队列恢复、硬预算、召回质量和开关关闭回归——系统尚未实现。
7. 真实群/私聊归档同意、许可范围与数据量——未读取任何真实聊天正文或生产数据文件。

第 1 项原属设计 P0 运行门禁，本次因用户明确限制测试不得联网而保留；不能因文档完成自动勾选。

## 8. 风险分级与 P1 前置条件

| 编号 | 风险/阻塞 | 门禁 |
| --- | --- | --- |
| R1 高 | compaction 独立 usage 不在所审 Token 折叠中，后台分类也不完整 | 阻塞 P3/P4 预算与“全量”宣称；不阻塞无 LLM 的离线 P1 |
| R2 高 | 共享撤销只阻新读取，旧上下文仍可能持有来源；退役异步无完成屏障 | 生产 P1 接入前必须明确撤销契约；在 P1 构建来源/epoch/依赖，接入完成前禁止真实归档/注入 |
| R3 高 | 输出限制有适配器能力，但逐 HTTP 尝试、同模型和可靠输入上界未验收 | 阻塞任何后台付费任务及 global 硬预算，不要求为了离线 P1 修改模型配置 |
| R4 高 | 凭据/管理特例不能直接作为 Memory ACL；旧 impression 无可信来源 | P1 schema 和权限契约先明确；旧数据不导入、不扩权 |
| R5 中 | SQLite/FTS 基础通过，但中文、WAL 和恢复非等价于开箱可用 | P1 离线迁移/故障测试、P2 中文检索门禁；无新增驱动必要 |
| R6 中 | 工作树脏、进程已加载版本/插件与磁盘声明存在验证空白 | 保留快照身份；P1 开始前重新核对目标工作树；生产接入另补受限空会话验收 |
| R7 高 | 会话 archive 不删除 JSONL/压缩摘要/索引/备份 | P1 前约定分层删除回执和残留披露；未确定支持接口时不得承诺彻底删除，不直接手改运行日志 |

**启动离线 P1 开发前须落实（尚未执行）：**
- 用户下达 P1 指令，并确定安全开发分支/目录与原有未提交改动边界；不得携带其业务改动提交。
- 确定 Memory 为独立存储，native node:sqlite 使用系统 Node；不开生产归档、不导入旧数据、不运行 LLM。
- 固定 RequestContext、capability、来源权限交集、policy_version、删除 epoch 与失败闭锁的契约；新字段默认拒绝授权。
- 明确删除可控范围和 Harness 残留回执，避免 archive=delete；确定工作记忆/长期记忆主数据边界。
- 设计第 19 节空会话门禁仍未满足：若要求它作为 P1 的硬前置，应先补完后才能进入 P1；本审计不擅自豁免。即使用户允许先开发独立离线骨架，也不视为运行门禁通过。

**生产接入前另需**：在获准的隔离测试环境补空会话探测、preset 工具面和 profile 激活验证；完成权限/删除/故障合成矩阵及关闭开关回归。任何后台 LLM 的放行还需 R1/R3 关闭，与 P1 的无 LLM 开发分开验收。

## 9. 建议下一阶段最小任务（仅建议，未开始）

在收到指令并处理上述门禁后，P1 首个小任务限定为独立 store/policy 的 schema 与迁移、来源边、删除 epoch 和 RequestContext/capability 纯规则，全部使用临时合成库；默认关闭，先不接 QQ 收发、不导入旧工作记忆、不加模型请求。验收：跨域拒绝、来源交集、迁移幂等/失败回滚、删除屏障与旧结果拒绝。观察面板与实际归档接入作为 P1 后续小任务，不在本轮展开。

## 附录：关键文件内容指纹

SHA-256 取审计时磁盘文件原始字节，用于区分“同 HEAD、不同工作树”；不含角色卡、配置、日志、数据库或聊天正文。

| 文件 | SHA-256 |
| --- | --- |
| B/src/bridge.js | a7553932900bbdb029d6c49979e61663fb8a7134c9a15713a2efd9947954afbc |
| B/src/dsh-client.js | a1e368815fba46c382f3a6d8c2548d6da0ad31710f5c480d8ef2e0413604b425 |
| B/src/token-ledger.js | bccc003c778ced6c0a48a1cc6987e27560c4c71a7db53533147d76640efb4d2c |
| B/src/shared-history.js | 48bd75c0595a9c09b0a73ce7a35468974ab31b6abb72f34ce1b930ef460eac5e |
| H/dsh-llm/lib/index.js | d18d8ae4c8d961e628708c1e4698e09716a0ebb80e55a672e0bde1029bec1155 |
| H/dsh-llm-deepseek/lib/index.js | eba35025ed1eedda89c51f2da46a3a968f7be1d59719f2ceeadfc4518dff37d6 |
| H/dsh-token-meter/lib/index.js | b76a27f84d1f733c4a760e36fcc0491f807b32d51055703feb0fed1643cc2e05 |
| H/dsh-compaction-basic/lib/index.js | b7ac52f20a36430cf01a66b3a36f23a48ffdb22708d714dbe1fff541b5df4ec5 |

本文件与 PROGRESS.md 是本次唯一拟提交变更。原设计文档不修改；无生产代码、角色卡、模型配置、密钥、Cookie、账号消息或旧未提交改动进入本次提交。
