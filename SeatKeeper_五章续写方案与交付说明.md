# SeatKeeper：五章连续 Go 大项目续写方案与交付说明

## 解释“5 章”与“4 章扩展”

本方案按 **1 个奠基章 + 4 个扩展章** 处理，共 5 章：

- 第 13 章建立 REST + PostgreSQL 的可运行基础；
- 第 14–17 章分别扩展一致性、可靠事件/微服务、后台任务/实时更新、可观测性/交付；
- 四个详细 AI 提示词对应第 14–17 章。

## 项目

**SeatKeeper**：面向社区活动、工作坊和小型会议的限量席位预订平台。

核心不变量：

```text
reserved_seats <= capacity
```

最终包含：

- Go 1.23 REST API
- PostgreSQL、迁移、约束、索引、事务
- 并发不超卖、幂等键、ETag/If-Match
- 事务型 Outbox
- NATS JetStream
- 独立通知微服务与独立数据库
- 有界并发的过期 worker
- SSE 实时状态
- `slog`、request ID、Prometheus、health/readiness
- 限流、超时、优雅关闭、Docker Compose、CI、load generator

## 五章路线

| 章节 | 情境 | 主要交付 | 仓库代码增量建议 |
|---|---|---|---:|
| 13 构建 SeatKeeper API | 共享表格失控，需要一个网页/CLI 都能调用的后端 | REST、领域模型、PostgreSQL、迁移、OpenAPI、测试 | 650–850 行业务 Go + 150–220 行测试 |
| 14 最后一张票 | 两部手机同时买到最后两张，网络重试又重复下单 | 原子容量、事务、幂等、状态机、ETag、并发测试、loadgen | 450–650 + 200–300 测试 |
| 15 消失的确认信 | DB 已确认但通知丢失，直接双写无法可靠 | Outbox、JetStream、事件契约、通知微服务、幂等消费 | 650–850 + 180–280 测试 |
| 16 被遗忘的座位 | 用户占位后离开，页面假售罄且前端频繁轮询 | 过期 worker、有界并发、context、SSE、背压 | 450–650 + 150–230 测试 |
| 17 发布之夜 | 值班工程师无法回答服务是否可用、慢在哪里、能否安全关闭 | 日志、指标、探针、限流、graceful shutdown、Compose、CI | 300–500 Go + 220–350 配置/文档 |

## 体量控制

- 每章 8–12 个教学小节；
- 每章 7–12 个代码清单；
- 单个清单以 12–45 行为主，极少超过 70 行；
- 正文展示新/改代码 220–380 行；
- 完整实现放代码包，不反复整文件复制；
- 每章至少一个“之前失败、之后通过”的测试或故障实验；
- 每章 3–6 个 Note/Warning/Tip；
- 每章结束有可运行 checkpoint/tag。

## 写作工作流结论

不要完全先写完最终代码再补正文，也不要完全即兴边写边做。

采用：

> **先完成足以验证终局的工程样机，再按章节完成可运行代码，并在每个 checkpoint 刚完成时立即写正文。**

推荐步骤：

1. 固定最终领域模型、API、进程、事件名和验收场景；
2. 用 walking skeleton 验证最危险技术点；
3. 从第 13 章开始建立真实 checkpoint；
4. 每章先写失败测试/复现，再完成代码到绿色；
5. 立即根据真实 diff、命令输出和日志写正文；
6. 从上章 tag 的干净目录按正文重放；
7. 冻结 tag 后再写下一章。

## 代码交付内容

最终代码包内的重要文档：

- `README.md`：运行、演示、测试、API 和边界
- `docs/project-design-zh.md`：原书内容/风格回顾与完整五章蓝图
- `docs/ai-prompts-zh.md`：共享总控提示词与四个扩展章提示词
- `docs/writing-workflow-zh.md`：代码与正文的推荐生产流程
- `docs/chapter-checkpoints-zh.md`：五个 checkpoint 的文件增量与验收
- `docs/architecture-zh.md`：事务、Outbox、worker、SSE 和可观测性说明
- `api/openapi.yaml`：OpenAPI 3.1 契约

## 验证说明

交付代码已执行 `gofmt`，并使用 API 兼容的本地依赖替身完成：

```text
go vet ./...
go test ./...
go test -race ./...
六个 cmd 包的 go build
```

由于交付环境无法访问公共 Go module 代理，也没有可用的 PostgreSQL/NATS/Docker daemon，因此没有在该环境中执行真实依赖下载、Docker Compose 端到端运行或 PostgreSQL 集成测试。项目保留真实依赖的 `go.mod`；在联网环境先运行 `go mod tidy`，再执行 README 中的 `make up`、`make test-race` 和 `make integration`。
