# 《用袖珍项目学 Go》中文辅助阅读版：构建说明

## 内容范围

本工程由用户提供的中文排版工程继续扩展而成，收录第 3--12 章：

- 第 3--9 章：沿用既有 revised 工程中的 `part01`--`part09`；
- 第 10 章：`chapters/chapter10_habits_grpc.tex`；
- 第 11 章：`chapters/chapter11_html_grpc_client.tex`；
- 第 12 章：`chapters/chapter12_seatkeeper_api.tex`，为基于 SeatKeeper 课程设计撰写的原创连续项目第一章。

主文件为 `main.tex`。已编译成品为 `main.pdf`：共 598 个物理页，页面尺寸为 170 mm × 240 mm。第 12 章位于物理页 522--598，共 77 页；书内页码为 508--584。

第 12 章配套代码位于：

```text
code/chapter12-seatkeeper/
```

它包含 REST API、应用服务、PostgreSQL repository、显式 SQL 迁移、OpenAPI 3.1、Docker Compose、单元测试、HTTP 测试和带 build tag 的 PostgreSQL 集成测试。

## 构建环境

正文使用 XeLaTeX。工程不包含、也不分发字体文件；系统中需安装下列字体族：

- Noto Serif CJK SC
- Noto Sans CJK SC
- Noto Sans Mono CJK SC
- Noto Serif
- Noto Sans Devanagari
- Noto Sans Arabic
- Noto Sans Ethiopic
- DejaVu Sans

建议使用较完整的 TeX Live 环境，并确保 `fontspec`、`ucharclasses`、`tcolorbox`、`listings`、`tikz`、`caption`、`needspace` 等宏包可用。

## 编译命令

在工程根目录执行三遍 XeLaTeX，以稳定目录、页码和交叉引用：

```bash
xelatex -interaction=nonstopmode -halt-on-error main.tex
xelatex -interaction=nonstopmode -halt-on-error main.tex
xelatex -interaction=nonstopmode -halt-on-error main.tex
```

本次交付环境使用了等价的 XDV 流程：

```bash
xelatex -no-pdf -interaction=batchmode -halt-on-error main.tex
xelatex -no-pdf -interaction=batchmode -halt-on-error main.tex
xelatex -no-pdf -interaction=batchmode -halt-on-error main.tex
xdvipdfmx -E -o main.pdf main.xdv
```

也可以使用：

```bash
latexmk -xelatex -interaction=nonstopmode -halt-on-error main.tex
```

## 第 12 章代码验证

进入代码目录后运行：

```bash
cd code/chapter12-seatkeeper
make fmt
make vet
make test
make test-race
make build
```

本次交付环境中，下列检查已经通过：

```text
gofmt check
CGO_ENABLED=1 go test ./...
CGO_ENABLED=1 go test -race ./...
CGO_ENABLED=1 go vet ./...
CGO_ENABLED=1 go build ./...
```

PostgreSQL 集成测试也完成了编译和启动检查，但交付环境没有可用的 PostgreSQL 服务或 Docker daemon，因此测试按照设计因缺少 `SEATKEEPER_TEST_DATABASE_URL` 而跳过。要执行真实数据库往返测试，请先启动 PostgreSQL，再运行：

```bash
export SEATKEEPER_TEST_DATABASE_URL='postgres://seatkeeper:seatkeeper@localhost:5432/seatkeeper?sslmode=disable'
CGO_ENABLED=1 go test -v -tags=integration ./internal/postgres
```

该测试会清空本章的 `events` 和 `reservations` 表，请使用一次性开发数据库。

## 目录结构

```text
main.tex                                   主文件
main.pdf                                   已编译中文合并版
styles/pocketbook.sty                      版式、字体、代码与提示框环境
chapters/                                  第 3--12 章正文
figures/                                   原书插图与既有项目图
code/chapter12-seatkeeper/                 第 12 章完整代码 checkpoint
review/CH12_SOURCE_STYLE_RESEARCH.md       开工前资料与写作风格研究
review/CH12_TECHNICAL_PEDAGOGY_AUDIT.md    第 12 章技术、教学与版式终审
review/CH12_CODE_VERIFICATION_FINAL.log    Go 工程实际检查输出
review/PDF_PREFLIGHT_CH12_FINAL.json       PDF 预检结果
```

## 内容边界

第 3--11 章是中文辅助阅读内容；第 12 章是沿用原书教学方法写成的原创续写，不冒充英文原书正文。代码标识符、文件名、命令和 API 字段保持英文，中文负责解释概念、动机、风险和取舍。
