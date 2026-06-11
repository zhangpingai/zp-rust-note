# AGENTS.md

本文件为本仓库内 AI 编码助手的项目级工作说明。适用于整个仓库。

## 项目概览

- 项目名：`ai-check-images`
- 类型：Rust 二进制 HTTP 服务
- 目标：用 Rust + `axum` 提供HTTP服务
- 默认监听地址：`0.0.0.0:8767`
- 主要接口：
  - `GET /v1/api/check_images/hello`
  - `POST /v1/api/check_images/check_more`

## 常用命令

```powershell
cargo run
cargo build
cargo build --release
cargo fmt -- --check
cargo check --all
cargo test
cargo clippy --all-targets --all-features --tests --benches -- -D warnings
cargo deny check -d
typos
```

启动后可用下面命令验证服务：

```powershell
curl.exe http://127.0.0.1:8767/v1/api/check_images/hello
```

期望返回：

```json
{"message":"service is ok!!!"}
```

## 配置文件

运行时配置位于根目录 `config/`：

- `config/gemini.toml`：Gemini API 地址和 Authorization
- `config/apikey.txt`：调用方 API key，一行一个
- `config/prompt.txt`：运行时可修改的 prompt

本项目是公司私有项目，当前约定是提交真实配置文件到 Git。即便如此，修改代码时仍然不要在日志、测试输出或错误信息中打印 API key、Authorization 或完整图片 base64。

## 端口配置

默认端口是 `8767`。如需临时覆盖：

```powershell
$env:AI_CHECK_IMAGES_HOST="127.0.0.1"
$env:AI_CHECK_IMAGES_PORT="8767"
cargo run
```

## 兼容性要求

重写后的 Rust 服务必须保持原 Python FastAPI 调用兼容：

- 成功响应直接返回模型 JSON，不额外包裹 `{ "data": ... }`
- 错误响应格式保持：

```json
{"detail":"错误信息"}
```

- 鉴权方式保持：

```http
Authorization: Bearer <APIKey>
```

- `images` 字段保持为完整 Data URL 字符串数组
- Data URL 只按原 Python 逻辑做格式校验和拆分，不做真实 base64 解码校验

## 代码结构

- `src/main.rs`：服务启动入口
- `src/app.rs`：`axum` 路由和共享状态
- `src/config.rs`：运行时配置读取
- `src/data_url.rs`：Data URL 解析
- `src/errors.rs`：错误响应
- `src/gemini.rs`：Gemini 请求和响应解析
- `src/handlers.rs`：HTTP handler
- `src/logging.rs`：控制台和文件日志初始化

## 日志

- 日志目录：`logs/`
- 日志文件按天滚动
- `logs/` 应保持在 `.gitignore` 中
- 不要记录 API key、Authorization 或完整图片 base64

## 提交前检查

提交前建议至少运行：

```powershell
cargo fmt -- --check
cargo check --all
cargo test
cargo clippy --all-targets --all-features --tests --benches -- -D warnings
cargo deny check -d
typos
```

如果 `pre-commit` hook 自动修改文件，例如修复换行符或文件末尾换行，需要重新执行：

```powershell
git add .
git commit -m "..."
```

## 文档查询规则

当问题涉及库、框架、SDK、API、CLI 工具或云服务的用法时，优先使用 `ctx7` CLI 获取当前文档，不要只依赖记忆。

流程：

```powershell
npx ctx7@latest library <name> "<question>"
npx ctx7@latest docs <libraryId> "<question>"
```

如果默认查询结果不充分，可使用 `--research`。

## 注意事项

- 不要无关重构。
- 不要随意改变接口路径、请求体、错误格式或成功响应结构。
- 不要删除或重写 `config/` 中的真实配置，除非用户明确要求。
- 不要提交运行时产物，例如 `target/`、`logs/`。
- 修改 `deny.toml`、`_typos.toml`、`.pre-commit-config.yaml` 时，要重新跑对应工具验证。
