# DSH Memory 实施进度

更新：2026-10-26（Asia/Shanghai）。

设计基线：16d7a7839c2cd3bc0cfbeced872e94c7370d0da3。详细依据见 [P0_COMPATIBILITY_AUDIT.md](P0_COMPATIBILITY_AUDIT.md)。

| 阶段 | 状态 | 验收结果 |
| --- | --- | --- |
| P0 实施前兼容审查 | 审计文档完成，有保留 | 安装版本/进程入口、定向源码、离线 SQLite/FTS5、Token 账本、共享权限、inbox 解析通过；真实空会话运行门禁未验证，未标为完整运行验收通过 |
| P1 数据与权限 | 未开始 | 无 Memory schema、归档或生产行为变更 |
| P2～P6 | 未开始 | 未实施召回、候选、预算、后台提炼、反思或自主成长 |

## 已确认

Harness 核心/agent/persona/compaction/DeepSeek 适配器为安装版 0.2.1-rc.2；bridge 0.2.1-r1、HEAD a49877f，工作树有用户既有改动；SnowLuma runtime 2.2.0、SDK/MCP 1.14.20。监听进程入口已对照，激活插件与实际 provider 未验证。

系统 Node v24.15.0 内置 SQLite 3.51.3/FTS5 可用；SnowLuma 自带 Node v24.22.1，各自满足对应运行要求。无安装/升级依赖。

离线验收：Token 账本脚本 exit 0、inbox 12/12、合成共享权限 6 断言、内存 SQLite/FTS/触发器/事务/安全删除语法通过。unicode61 中文单字测试命中 0，需保留设计的中文处理方案。无全量回归或真实账号验收。

## 遗留风险与阻塞

- R1：compaction/summary 的独立 usage 未在所审 Harness tokenUsage 与 bridge 折叠中计入；现有面板不能直接作全部请求账单。阻塞后续预算能力。
- R2：共享历史撤销未清理已注入上下文；会话退役异步且 archive 非物理删除。生产归档/召回接入前需来源依赖、删除屏障及完成回执。
- R3：逐任务 max_tokens、每次 HTTP 尝试预算、完整输入上界、统一模型和 actual usage 语义未运行验收；禁止据此放行后台调用。
- R4：会话令牌/共享组不是完整 Memory ACL；旧工作记忆不具备可信人物和来源链，不可自动导入或扩大许可。
- R5～R7：文件型 WAL/恢复/清理、已加载 profile、真实空会话、支持的 Harness 删除协议未验证。
- 设计第 19 节要求的空会话门禁因本次仅允许无网络合成测试而保留；P0 文档交付完成不等于该门禁已通过。

## 下一步（等待用户指令）

先确认如何补齐设计规定的隔离空会话门禁及 P1 开发目录/分支边界。若用户明确允许先做独立离线 P1 骨架，则最小任务为 schema/migration + store/policy + sources/epoch + RequestContext 的合成测试；不开真实归档、不导入历史、不调用模型。P1 生产接入另需权限、删除、关闭开关回归和运行接口验收。

## 交付与变更范围

本次只交付 docs/implementation/P0_COMPATIBILITY_AUDIT.md 与本文件，通过独立文档工作目录交付到 Fz1025/DSH-Memory 的工作分支；提交/PR 编号由最终交付记录报告。本文件不自写自身 commit SHA。

未修改生产业务代码、角色卡或模型配置；未读取私聊正文/密钥/Cookie；未发送 QQ 或空间内容；未调用付费模型；未重启服务；未暂存或提交原项目既有改动；未修改原 remote。

完成本轮后停止，不自动进入 P1。
