# DSH Memory 实施进度

更新：2026-10-26（Asia/Shanghai）。

设计基线：16d7a7839c2cd3bc0cfbeced872e94c7370d0da3。
P0 文档已通过 [PR #1](https://github.com/Fz1025/DSH-Memory/pull/1) Squash 合并，main 基线为 39a664bc05ab49bca0201bf72c59813c3b90a597。

| 阶段 | 状态 | 验收结果 |
| --- | --- | --- |
| P0 兼容审查 | 文档已合并，有保留 | 真实空会话、有效模型/预算覆盖、删除接口等运行门禁未通过，合并不代表完整运行验收 |
| P1.1 离线数据与权限基础 | 开发完成，待用户验收；文档 PR 保持 Draft | 28/28 离线合成测试通过，代码只在独立本地分支，不接生产 |
| P1.2～P1.4 | 未开始 | 未接真实认证/消息归档/观察面板/完整删除链路 |
| P2～P6 | 未开始 | 未实施检索召回、候选、预算、提炼、反思或自主成长 |

详细依据：[P0_COMPATIBILITY_AUDIT.md](P0_COMPATIBILITY_AUDIT.md)、[P1_1_DEVELOPMENT_REPORT.md](P1_1_DEVELOPMENT_REPORT.md)。

## P1.1 交付

- src/memory/ 新增 10 个文件：SQLite/迁移、权限、合成库边界、来源/任务快照、逻辑删除屏障、测试和说明。
- 使用现有 Node 24 的 node:sqlite，无依赖变更；文件库 WAL/FK/FULL synchronous，FTS5 external-content 触发器。
- RequestContext 可信进程签发与严格校验，bot/persona/QQ namespace 隔离，默认拒绝、deny 优先、capability/purpose、policy_version、来源交集。
- 删除 epoch 在事务内重查，来源撤销后合成记忆及派生物不可读，旧任务不能写回；不宣称物理删除。
- 默认关闭，显式启用也只能打开内存库或 OS 临时合成 sandbox；主聊天入口没有导入 Memory。

代码基线 a49877f4e58d04e47532d44b803360d0f001160d；
本地分支 memory/p1.1-offline；
本地 commit 3035d279107f2399233a37f600d5e3b6df435aa3。
未向第三方 origin 推送；未创建用户代码仓库。文档分支 docs/p1.1-offline-foundation 基于 P0 合并后的最新 main，只提交本报告和进度文件。

## 验证与边界

node --test src/memory/test/foundation.test.mjs：28 tests、28 pass、0 fail、0 skipped。
覆盖初始化/幂等迁移/失败回滚/FK、身份与跨域拒绝、权限交集、policy/epoch、DB 关闭/坏库/WAL 竞争、FTS 同步、删除事务回滚和默认关闭。

完整 bridge 禁用回归、QQ 真实认证与消息、Harness 清理、物理删除/备份恢复和真实性能均未验收。FTS 缺失降级是注入能力标志的规则测试，本机实际 FTS5 可用。未实现全量设计表或持久 worker；详细偏差见开发报告。

## 仍存阻塞

- P0 R1：压缩 usage 计量缺口未修改，后续硬预算仍阻塞。
- R2：本阶段只验证合成库逻辑屏障，未清理真实 Harness 旧共享上下文。
- R3/R6：真实空会话、有效模型/输出限额/逐 HTTP 尝试预算覆盖及实际插件仍未验证。
- R4：离线规则不等于真实服务端身份认证、同意或管理权限实现。
- R5/R7：临时 WAL/FTS 验证不等于生产备份恢复、物理擦除或受支持的 Harness 删除接口。
- 代码 worktree 不含原生产目录未提交功能；未来移植必须保留其改动并重新核对。

## 下一步与停止点

等待用户验收 P1.1；新文档 PR 不自动合并，不进入 P1.2。建议下一阶段限定无模型的 consent/grants/来源服务、策略版本/撤销契约与合成适配；生产接入仍需单独通过运行门禁。

未读取/导入真实聊天、配置凭据或生产库；未改角色卡、提示词、模型、空间发布功能或生产入口；未重启服务、未调用付费模型、未安装依赖。原工作树 Git 状态比较一致，既有改动未提交。
