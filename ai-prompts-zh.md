# 用 AI 生成后四章正文的提示词

本文件按“第 13 章由作者先建立基础系统，第 14–17 章由 AI 在固定 checkpoint 上逐章扩展”的方式设计。不要把四个提示词一次性投给模型；每完成一章，先运行代码和测试、冻结 checkpoint，再把更新后的仓库与下一章提示词交给模型。

## 0. 四章共用的总控提示词

把下面内容作为 system prompt、project instruction，或放在每个章节提示词前。`<...>` 中的内容由编辑替换。

```text
你是一名资深 Go 工程师、技术图书作者和严格的代码审稿人。你正在续写一本以项目驱动方式教授 Go 的图书。前 12 章采用直接、对话式、hands-on 的风格：每章先给真实情境，再列“本章涵盖”、需求和限制；随后从可运行的最小竖切面开始，边测试边实现；穿插 Note、Warning、Tip；最后给 Summary 和可选 Side quests。

你必须基于我提供的 SeatKeeper 仓库 checkpoint 写作，仓库是事实来源。先阅读 README、docs、go.mod、相关 cmd/internal/migrations/api 文件和测试，再开始正文。不得虚构不存在的文件、函数、命令输出、第三方 API 或测试结果。发现代码与章节目标不一致时，先列出最小修复补丁，再以修复后的代码为正文依据。

写作语言为简体中文；Go 标识符、文件名、HTTP 字段、SQL、命令和代码注释保持英文。语气务实、亲切，但不要营销化。解释“为什么”和失败路径，不只说明代码做了什么。

Go 风格约束：
1. 优先标准库；第三方依赖必须解决明确基础设施问题。
2. 接口保持小，并定义在使用方；不要创建 BaseService、万能 Repository、utils/common 大杂烩。
3. 依赖通过构造函数显式注入；错误通过 errors.Is/As 或类型判断分类。
4. context.Context 只作为第一个参数传递，不存入结构体；所有阻塞操作必须考虑取消或超时。
5. 每个 goroutine 都必须说明所有者、停止条件、错误去向和并发上限。
6. 不声称 exactly-once；明确至少一次、幂等和故障窗口。
7. 数据库不变量应尽量由约束、条件更新和事务共同保护，而不是只依赖应用层检查。
8. 所有代码应 gofmt，并能在 Go 1.23 下编译。正文中省略代码时使用明确说明，不使用会破坏可复制性的“伪代码省略号”。

章节结构必须包含：
- 情景化开场（500–900 中文字，先呈现问题，不先报技术名词）
- “本章涵盖”5–7 项
- 功能需求、限制与非目标
- 上一章系统状态图和本章目标图（Mermaid）
- 一个先失败的测试、复现脚本或故障实验
- 8–12 个循序渐进的小节
- 7–12 个关键代码清单；单个清单通常 12–45 行，极少超过 70 行
- 每个清单后的逐段解释、失败分支和 Go 取舍
- 至少 4 组可复制命令及可信的预期输出形态
- 至少 3 个 Note/Warning/Tip
- 测试策略、运行验证、故障注入
- Summary：只总结本章真正完成的能力
- 3–5 个 Side quests，明确它们不属于主线实现

体量约束：正文展示的新/改代码控制在 220–380 行；完整仓库增量按章节提示词中的预算执行。不要为了增加篇幅重复打印未变化的完整文件。长 SQL、OpenAPI、Compose 和夹具只展示关键差异，并指向完整代码包。

代码与正文同步规则：
- 先给本章验收条件，再实现。
- 每个主要阶段后给出 `go test`、`curl`、SQL 或日志验证。
- 若一个命令的具体 UUID、时间或端口会变化，用占位符并解释，不伪造固定值。
- 同一概念第一次出现时详细解释，后续只说明差异。
- 章节结尾给出“本章 checkpoint 文件清单”和建议 git tag。

输出只包含本章成稿与必要的最小补丁说明，不讨论你的内部推理过程。
```

---

## 1. 第 14 章提示词：最后一张票

```text
请基于我提供的 SeatKeeper 第 13 章 checkpoint，撰写第 14 章《最后一张票：用事务、幂等和乐观锁保护预订》。

情境：活动在中午开放，两个用户同时预订最后的座位，旧实现发生超卖；移动网络超时又触发自动重试，产生重复预订；客服和用户还可能同时修改同一个 hold。章节必须从可复现的失败开始，而不是先解释 ACID 定义。

本章的最终验收条件：
1. 30 个并发请求竞争一个容量为 5 的活动时，恰好最多 5 个成功，数据库中 `reserved_seats` 永远不大于 `capacity`。
2. 相同 `X-User-ID`、相同 `Idempotency-Key`、相同请求体重复提交，返回原预订且不再次扣减容量；相同键配不同请求体返回 409。
3. 确认与取消必须遵守 held -> confirmed/cancelled/expired 的状态规则。
4. GET 预订返回数字版本 ETag；确认和取消要求 `If-Match`；旧版本返回 412。
5. 所有容量变更、预订写入和相关约束在 PostgreSQL 中保持原子性。
6. 单元测试、HTTP 测试和带 build tag 的 PostgreSQL 并发集成测试都存在。
7. 增加 Go 编写的 `cmd/loadgen`，使用有界 worker pool，不为每个请求无限创建 goroutine。

必须覆盖的技术内容：
- 先读后写为什么有 TOCTOU 竞争；
- 比较 `SELECT ... FOR UPDATE` 与条件原子 `UPDATE`，说明本项目为什么选择：
  `UPDATE events SET reserved_seats = reserved_seats + $quantity
   WHERE id = $id AND capacity - reserved_seats >= $quantity`；
- PostgreSQL 事务边界和约束；
- `(user_id, idempotency_key)` 唯一约束；
- 对规范化输入做 SHA-256 请求指纹；
- 并发的相同幂等键竞争如何处理唯一约束；
- `version` 字段、ETag 与 `If-Match`；
- `errors.Is` 将存储错误翻译成领域/API 错误；
- goroutine 驱动的集成测试与 `go test -race` 的职责区别；
- 取消 confirmed 预订时释放座位，确认时不重复扣减；
- 过期处理本章只保留领域能力，自动 worker 留到第 16 章。

应重点使用或形成这些文件（以实际仓库为准，不存在时创建）：
- `internal/booking/domain.go`
- `internal/booking/errors.go`
- `internal/booking/service.go`
- `internal/booking/postgres/reservations.go`
- `internal/booking/httpapi/reservations.go`
- `migrations/booking/*.sql`
- `internal/booking/postgres/integration_test.go`
- `cmd/loadgen/main.go`
- `api/openapi.yaml`

建议章节小节顺序：
1. 复现最后一张票的失败
2. 把容量规则写成数据库不变量
3. 用一个条件 UPDATE 争夺容量
4. 把容量和预订放进同一事务
5. 让网络重试安全：幂等键与请求指纹
6. 处理两个相同幂等请求的数据库竞争
7. 把预订状态写成领域状态机
8. 用 ETag 防止丢失更新
9. 映射 HTTP 冲突语义
10. 运行并发集成测试和 loadgen
11. 取舍、限制、总结

代码增量预算：业务 Go 450–650 行，测试 200–300 行，SQL 15–30 行。正文最多展示约 340 行代码；优先展示并发测试、关键 SQL、事务函数、幂等分支和 ETag handler，不重复打印基础 JSON helper。

必须包含的 Warning：
- 幂等键必须有作用域、保留期和请求一致性检查；仅添加唯一键不够。
- `go test -race` 检测的是进程内数据竞争，不会自动发现数据库超卖。
- 不应在事务中做慢速网络调用。

章节结束建议 git tag：`chapter-14-consistency`。
```

---

## 2. 第 15 章提示词：消失的确认信

```text
请基于完成第 14 章后的 SeatKeeper checkpoint，撰写第 15 章《消失的确认信：事务型 Outbox、NATS JetStream 与通知微服务》。

情境：确认预订的数据库事务成功了，但确认邮件没有发送。直接在事务提交后调用邮件或 broker 会丢通知；在事务提交前调用则可能发送一封对应失败事务的邮件。请先用时间线或失败矩阵展示双写困境，再引出 Outbox，不要把 Outbox描述成流行架构名词的堆砌。

本章最终验收条件：
1. 创建、确认、取消或过期预订时，业务状态与事件 envelope 在同一个 PostgreSQL 事务中提交。
2. NATS 暂停时业务写入仍可成功；Outbox 行保持未发布；NATS 恢复后 relay 最终发布。
3. 两个 relay 实例可并发运行，使用 `FOR UPDATE SKIP LOCKED` 和有时限的 lease，不长期重复处理同一行。
4. 发布使用事件 ID 作为 NATS message ID，JetStream 提供时间窗内去重。
5. 事件包含 `id`、`type`、`version`、`aggregate_id`、`occurred_at`、`request_id` 和版本化 data。
6. notifications 是独立进程，拥有独立 PostgreSQL 数据库，不直接读取 booking 表。
7. durable consumer 使用显式 ack/nak；已处理事件再次投递时不会重复记录 delivery。
8. 模拟邮件由 `LogSender` 输出，但 Sender 接口保留替换真实供应商的边界。
9. 测试覆盖 relay 成功/失败、契约编解码、幂等消费者和故障恢复说明。

必须覆盖的技术内容：
- 双写失败矩阵；
- Outbox 表结构、索引和与业务事务同提交；
- relay 的 claim/lease/mark-published/mark-failed 生命周期；
- 为什么 `SKIP LOCKED` 适合多个 relay；
- 至少一次投递而不是 exactly-once；
- NATS JetStream stream、subject、durable、queue group、manual ack；
- 事件契约版本与兼容性；
- 微服务的两个判据：独立部署和数据所有权；
- inbox/`processed_events` 幂等；
- 外部邮件副作用仍存在“发送成功但本地记录失败”的窗口，真实供应商应使用事件 ID 作为幂等键；不要隐藏这一限制；
- context timeout、连接 drain 和重试边界。

应重点使用或形成这些文件：
- `internal/contracts/events.go`
- `internal/booking/postgres/outbox.go`
- `internal/outbox/relay.go`
- `internal/outbox/postgres.go`
- `internal/messaging/nats.go`
- `cmd/outbox-relay/main.go`
- `internal/notification/service.go`
- `internal/notification/consumer.go`
- `internal/notification/postgres/store.go`
- `cmd/notifications/main.go`
- `migrations/booking/*.sql`
- `migrations/notifications/*.sql`
- `deployments/postgres/init.sql`
- `docker-compose.yml`

建议章节小节顺序：
1. 用两个崩溃时刻证明普通双写不可靠
2. 定义跨进程事件契约
3. 在业务事务内追加 Outbox
4. 实现单次 relay 迭代
5. 用 lease 和 SKIP LOCKED 支持多个实例
6. 配置 JetStream 和发布去重
7. 启动独立 notifications 服务
8. 幂等消费与显式 ack
9. 暂停/恢复 NATS 的故障演练
10. 至少一次语义与剩余窗口
11. 总结与 Side quests

代码增量预算：业务 Go 650–850 行，测试 180–280 行，SQL/Compose 30–60 行。正文展示不超过 360 行代码。不要完整打印所有启动函数；重点展示事件 envelope、同事务插入 Outbox、claim SQL、relay loop、consumer handler 和幂等存储。

必须包含的 Warning：
- Outbox 不等于 exactly-once。
- 事件不能直接暴露内部数据库行；契约一旦发布就要考虑兼容性。
- 微服务不应共享同一个业务 schema 或跨库 JOIN。

章节结束建议 git tag：`chapter-15-outbox-microservice`。
```

---

## 3. 第 16 章提示词：被遗忘的座位

```text
请基于完成第 15 章后的 SeatKeeper checkpoint，撰写第 16 章《被遗忘的座位：有界并发后台任务与 SSE 实时更新》。

情境：大量用户创建 hold 后关闭页面，活动显示售罄却没有真实确认；运营只能手工清理。与此同时，前端每两秒轮询一次状态，流量和数据库读负载不断上升。请用“座位何时、由谁、以什么并发度归还”作为章节主线。

本章最终验收条件：
1. held 预订包含 `expires_at`，数据库有只覆盖 held 行的过期索引。
2. expiration worker 按批次读取已到期 ID，并以可配置并发上限处理。
3. 两个 worker 同时看到同一个 ID 时，版本检查/状态机保证最多释放一次容量。
4. 每个任务有 context timeout；进程取消时停止接收新工作并等待现有 goroutine 返回。
5. 过期更新、释放座位和 `reservation.expired` Outbox 事件在同一事务中。
6. Booking API 暴露 SSE endpoint；连接先收到 snapshot，再收到该 reservation 的后续事件。
7. SSE 使用有界 channel。慢客户端不会阻塞 NATS callback；采用明确的丢弃与日志策略。
8. 客户端断开、服务关闭或订阅 drain 后，相关资源被释放。
9. 测试覆盖并发上限、可忽略的版本冲突、取消、SSE snapshot 和事件转发。
10. `go test -race ./...` 是本章完成条件之一。

必须覆盖的技术内容：
- ticker 的生命周期；
- 批处理与并发上限为什么是两个独立旋钮；
- `errgroup` 与 semaphore channel；
- 不要“每条到期记录一个无限制 goroutine”；
- worker 的 at-least-once 扫描如何通过幂等状态转换变得安全；
- `context.WithTimeout` 的取消传播；
- SSE 与 WebSocket 的取舍：本项目只需服务端单向推送；
- `http.Flusher`、`text/event-stream`、heartbeat；
- Core NATS subscription 只接收订阅后的事件，因此先发送当前 snapshot；
- 有界 channel、非阻塞 `select` 和慢消费者背压；
- 写超时为何不能粗暴应用于长寿命 SSE。

应重点使用或形成这些文件：
- `internal/booking/domain.go`
- `internal/booking/service.go`
- `internal/booking/postgres/reservations.go`
- `internal/expiry/worker.go`
- `cmd/expiration-worker/main.go`
- `internal/booking/httpapi/sse.go`
- `internal/booking/httpapi/server.go`
- `internal/messaging/nats.go`
- `docker-compose.yml`
- 相应 `_test.go` 文件

建议章节小节顺序：
1. 复现“假售罄”并定义 hold 生命周期
2. 用部分索引找到到期记录
3. 先实现单条 Expire 状态转换
4. 批量扫描但串行执行
5. 加入有界并发和任务 timeout
6. 证明多个 worker 不会重复释放
7. 为什么轮询不是本场景的最佳反馈方式
8. 建立 SSE snapshot
9. 订阅单个 reservation 的事件
10. 慢客户端、heartbeat 与资源清理
11. race 测试、故障演练和总结

代码增量预算：业务 Go 450–650 行，测试 150–230 行，SQL 10–20 行。正文最多展示约 330 行代码。重点展示 worker 的 RunOnce、状态机/事务、SSE handler 和测试；不要展示大量样板启动代码。

必须包含的 Warning：
- goroutine 不是免费的任务队列；必须有并发上限和取消路径。
- 向慢客户端无限缓存会把背压变成内存泄漏。
- SSE 端点不能沿用普通短请求的统一 WriteTimeout。

章节结束建议 git tag：`chapter-16-workers-sse`。
```

---

## 4. 第 17 章提示词：发布之夜

```text
请基于完成第 16 章后的 SeatKeeper checkpoint，撰写第 17 章《发布之夜：可观测性、韧性和可重复交付》。

情境：活动即将开放，值班工程师面对 booking-api、outbox-relay、expiration-worker、notifications、PostgreSQL 和 NATS，却无法判断哪个进程只是“活着”、哪个真正“准备好”，也不知道请求慢在哪里、Outbox 是否失败、SSE 是否积压。章节必须从一份上线前无法回答的问题清单开始。

本章最终验收条件：
1. 所有进程输出结构化 JSON 日志，并支持 `LOG_LEVEL`。
2. Booking API 接受或生成 `X-Request-ID`，响应返回它；业务事件 envelope 携带该 ID。
3. panic 被恢复并记为 500；请求日志和 HTTP 指标仍能观察到这次请求。
4. `/health/live` 只回答进程存活；`/health/ready` 对必要依赖做有时限检查。
5. Prometheus 指标使用低基数标签，包含 HTTP 数量/耗时、Outbox 成功/失败、过期结果、通知结果和当前 SSE 连接数。
6. API 有请求体上限、超时、按客户端 token-bucket 限流和合理的 server timeouts；SSE 被明确排除于短请求 timeout。
7. SIGTERM 触发 HTTP graceful shutdown、worker 退出和 NATS drain，不用 `os.Exit` 跳过 defer 的清理路径（主函数最终退出除外）。
8. 多阶段 Dockerfile 以非 root 用户运行；Compose 能启动完整拓扑并通过 health/depends_on 连接依赖。
9. CI 执行 gofmt 检查、单元测试、race detector 和 build。
10. `cmd/loadgen`、`scripts/demo.sh` 和发布检查表可以复现主要能力。
11. OpenAPI 与实际状态码、headers、SSE endpoint 一致。

必须覆盖的技术内容：
- `slog` handler 与稳定字段；
- request ID 中间件及中间件顺序；
- liveness 与 readiness 的语义差异；
- Prometheus label cardinality，不能用 URL 中的 UUID、email 或 request ID 做标签；
- response recorder 需要支持 `http.Flusher`，否则 SSE 会被中间件破坏；
- token bucket 与 per-client limiter 的清理；
- `signal.NotifyContext`、`http.Server.Shutdown`、NATS `Drain`；
- `ReadHeaderTimeout`、`ReadTimeout`、`IdleTimeout` 和 SSE 对 `WriteTimeout` 的影响；
- Docker build cache、非 root runtime、数据库初始化只发生在新 volume；
- 故障演练：暂停 NATS、停止通知服务、发送旧 ETag、触发限流、终止容器；
- 本项目仍不是完整生产平台：认证、密钥管理、TLS、真实邮件幂等、备份恢复和 tracing 留作扩展。

应重点使用或形成这些文件：
- `internal/platform/config/*`
- `internal/platform/requestid/*`
- `internal/platform/run/*`
- `internal/booking/httpapi/middleware.go`
- `internal/booking/httpapi/health.go`
- `internal/observability/metrics.go`
- `cmd/*/main.go`
- `Dockerfile`
- `.dockerignore`
- `docker-compose.yml`
- `.github/workflows/ci.yml`
- `scripts/demo.sh`
- `api/openapi.yaml`
- `README.md`

建议章节小节顺序：
1. 列出发布前无法回答的问题
2. 用结构化日志和请求 ID 建立关联
3. 正确组合 panic recovery、access log 和 metrics
4. 区分 live 与 ready
5. 设计低基数指标
6. 给 HTTP 边界增加大小、时间与速率保护
7. 用信号和 context 有序关闭
8. 构建最小、非 root 的容器镜像
9. 用 Compose 启动完整系统
10. CI、loadgen 和故障演练
11. 安全复盘、发布清单和总结

代码增量预算：Go 300–500 行，配置/文档/脚本 220–350 行，测试 100–170 行。正文最多展示约 320 行代码。不要把完整 Compose、OpenAPI 或 CI 全部复制到正文；展示关键段落并解释完整文件位置。

必须包含的 Warning：
- readiness 失败不等于进程应该重启；liveness 不应依赖每个远程服务。
- 指标标签中放用户 ID、URL 参数或 request ID 会造成高基数灾难。
- graceful shutdown 有时间上限；它不是永远等待。

本章最后必须给出：
- 从空环境启动到完成一次预订确认的命令序列；
- 10 项发布检查表；
- 系统最终架构图；
- 五章累计学到的 Go 能力回顾；
- 建议 git tag：`chapter-17-production-ready`。
```

## 5. 使用这些提示词时的编辑流程

1. 把当前章的真实仓库压缩包交给模型，而不是只贴目录树。
2. 要求模型先输出“代码审计与章节差距”，编辑确认后再生成正文。
3. 代码修改进入真实仓库，运行格式化、测试、集成测试和演示命令。
4. 将真实命令输出、错误信息、截图或日志片段回填正文。
5. 冻结 checkpoint 和 tag，再进入下一章。
6. 最后统一做术语、状态码、文件路径和跨章引用检查。

这比让模型在没有仓库的情况下“凭空写一章”可靠得多，也能阻止前后章节出现不同函数签名、不同端口、不同事件名或无法编译的代码。
