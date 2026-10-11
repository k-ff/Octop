# 10 · 修改指南：改代码前查这一篇

> 一句话：这是"要改某个东西时该去哪儿"的速查表。它汇总了 02–09 各篇的修改指南和坑点清单，还收录了本次梳理时发现的"仓库文档与代码不符"和"疑似问题"。具体步骤仍以各篇对应章节为准。

---

## 1. 开发环境与验证命令

```bash
uv sync                        # 安装依赖（外部库源码在 .venv/lib/python3.12/site-packages/octop_*）
make install-hooks             # 每个 clone 做一次：commit 时自动跑 make all + 前端 build
uv run octop init              # 初始化 ~/.octop（或直接 octop run 走 Web 设置向导）
make dev                       # 前端 Vite(热更新) + 后端 octop run 同时启动；Vite 把 /api 与 WS 代理到 8088
make dev-backend / dev-frontend

make all                       # 交付标准：format-all + lint + typecheck(mypy --strict) + test
make test-fast                 # 跳过 slow 用例
make test-affected             # testmon：只跑受改动影响的测试（本地快速反馈）
uv run pytest tests/unit/xxx -x -q
uv run pytest tests/unit/i18n -q   # 改了 i18n JSON 之后必跑
cd dashboard && npx tsc -b     # 改了前端之后必跑
make build-frontend            # dashboard/ → src/octop/dashboard/（产物，别手改）
```

调试入口：

| 想看什么 | 去哪里 |
|---|---|
| 服务日志 | `~/.octop/logs/octop.log`；root 日志级别由环境变量 `OCTOP_LOG_LEVEL` 决定，不是 `config.log_level`（02 篇 §5 第 4 步） |
| API 是否可读、字段是否正确 | `http://127.0.0.1:8088/api/docs`（Scalar） |
| 一轮对话内部发生了什么 | Agent 开启 trajectory 后在控制台查看轨迹（03 篇、06 篇 §8） |
| 单步跟踪一条消息 | 断点：`api/routers/chat/ws.py` → `infra/gateway/process/processor.py:885 iter_turn_chunks` → `infra/agents/manager.py` 的 `stream` → `octop_harness/agent.py` |
| 数据库现状 | 默认 SQLite 在 `~/.octop/` 下；表结构见 06 篇表清单（schema v18） |

## 2. "我要改 X" → 去哪里

| 我要…… | 看哪篇的哪节 | 主要改动文件 |
|---|---|---|
| 加一个 REST 端点 | 07 §11.1（含完整骨架） | `api/routers/<res>.py` + `api/app.py` 注册 + `infra/…` 领域方法 + 测试 |
| 加一个错误码 | 07 §11.2 | `infra/errors.py`（`ErrorCode` **和** `_DEFAULT_STATUS`）+ `i18n/{en,zh}.json` 的 `errors.*` + `dashboard/src/locales/{en,zh}.json` 的 `apiErrors.*` |
| 加一条用户可见文案 | 07 §11.3、09 §11.3 | 后端 `src/octop/i18n/{en,zh}.json`；前端 `dashboard/src/locales/{en,zh}.json` |
| 新增表 / 加列 | 06 §11.1 | `infra/db/migrations/00N_*.sql` + `00N_*.pg.sql`、`migrate.py` 幂等 helper、`tests/unit/db/test_db_pool.py` 版本断言 |
| 新增 repo | 06 §11.2 | `infra/db/repos/xxx.py` → `RepoBundle`（`infra/db/services.py`）→ 如有长期持有，还要接入 `AppRuntime.replace_services` |
| 给 Agent 加配置项 | 04 §14.1 | `infra/agents/settings/` → `manager.py::_build_harness_config`（`:2978` 起）→ API 模型 → 前端表单 |
| 加一个内置工具 | 04 §14.2 | 在 `_build_harness_config` 中组装进 tools；显示名放到 `i18n` 的 `tools.*` |
| 接入新 LLM provider | 04 §14.3 | `infra/agents/providers/` + harness provider 配置 |
| 改 Agent 工作区种子文件（SOUL.md 等） | 04 §14.4 | `infra/agents/workspace/`，必须走 `BackendWorkspace` |
| 每轮对话前注入上下文 / 参数 | 03 §14.2 | `processor.py::_build_dashboard_request` 与 IM 路径；新参数**只能放进 `configurable`** |
| 加一种流式事件（前端能收到） | 03 §14.1 | 后端投影 + 前端 `chatStore.handleHarnessChunk` 加 `case` |
| 改历史落库 | 03 §14.3、06 §8 | `infra/history/`、`processor.py::_record_turn_history` |
| 加一个斜杠命令 | 05 §11.1（以 `/ping` 为例） | `infra/gateway/slash/handlers/` + `catalog.py` + `i18n` 的 `slash.*` + 测试 |
| 接入新 IM 渠道 | 05 §11.2 | `octop_gateway` 渠道实现 + `infra/gateway/channels/` + `bot_creators/` |
| 改 IM 回复格式 | 05 §11.3 | `gateway/process/stream_project.py`、`response_mode.py` |
| 加一种 cron 任务类型 / 触发语法 | 08 §7 | `cron/task_type.py`、`cron/delivery.py`、`cron/trigger.py` |
| 新增 connector | 08 §7 | `connectors/catalog.py` + adapters/registry |
| 换 embedding / 知识库新格式 | 08 §7 | `knowledge/` |
| 新增 skill 来源或安装目的地 | 08 §7 | `skills/skills_hub.py`、`SkillInstallTarget` |
| 新增前端页面 | 09 §11.1 | `dashboard/src/routes/` + `pages/` + `api/modules/` + `locales/` |
| 新增 CLI 子命令 | 09 §11.2（怎么选传输层） | `src/octop/cli/commands/<name>.py` + `cli/registry.py` |
| 改启动流程 / 加一个运行时单例 | 02 §13.1 | `infra/server.py::_boot_runtime`、`AppRuntime`、`stop()` |
| 加一个配置项（config.json / 环境变量） | 02 §7 | `src/octop/config.py`（`OctopConfig` 是 frozen 的，配置是启动时的快照） |

## 3. 通用红线（每次改动前过一遍）

1. **分层**：`infra/` 不 import `api/`、`cli/`、`launch.py`；`cli/` 不 import `api/`；router 只做校验、调用 infra、映射错误，不写业务规则，也不直接 import repo（01 篇 §2）。
2. **工作区 I/O**：Agent 工作区内容一律通过 `BackendWorkspace`（`agent.workspace` / `harness_workspace_for_agent` / `require_running_workspace`）读写；不要用 `Path.write_text` 直接写 `~/.octop/agents/<id>/`，也不要按 backend 类型分支。
3. **i18n**：用户可见文字不要硬编码英文。en 和 zh 的键树必须一致；`ErrorCode`、后端 `errors.*`、前端 `apiErrors.*` 三者的 key 集合由测试强制相等。
4. **DB 迁移**：SQL 必须成对（`.sql` + `.pg.sql`）；未发版的改动折叠进当前未发布的版本号；`migrate.py` 的 helper 必须保持幂等；记得改版本断言。
5. **异步**：async 函数里不要做阻塞 I/O（文件、SQLite 大查询、子进程），用 `asyncio.to_thread` / `run_in_executor`。
6. **时间显示**：前端不用裸 `toLocaleString()` 显示时间戳，统一走服务器时区（`useServerTimezone` + `formatMessageTime.ts`）。
7. **跨平台测试**：CI 同时跑 Linux 和 Windows。路径用 `tmp_path` 和 `Path` 比较，POSIX 专属逻辑加 `posix_only` 标记，`OCTOP_HOME` 通过 monkeypatch 指向 `tmp_path`。
8. **构建产物**：不要手改 `src/octop/dashboard/`，它会被 `emptyOutDir` 整体清空。
9. **API 文档**：新路由要写 `summary`，用 Pydantic 定义请求和响应模型，完成后去 `/api/docs` 看一眼。
10. **交付**：`make all` 通过；改了前端再加跑 `npx tsc -b`。

## 4. 跨模块高频陷阱（从各篇汇总）

| # | 陷阱 | 出处 |
|---|---|---|
| 1 | **两条对话路径平行但不完全相同**：IM 走 `GlobalProcessor.__call__`，控制台和 CLI 走 `iter_turn_chunks`。修 bug 或加功能时两边都要看 | 03 §8.1、§14.4 |
| 2 | **harness 请求不接受未知顶层键**：`ChatRequest(**request)` 遇到未知字段会直接 TypeError，新参数只能放进 `configurable` | 03 §14.2 |
| 3 | **全新安装时 `server.services` / `app_runtime` 为 `None`**（延迟绑定数据库），向导期间可能执行的代码要判空 | 02 §13.2 |
| 4 | **向导换库会热替换 services**：长期持有 repo 的新单例必须接入 `AppRuntime.replace_services`，否则会继续写旧库 | 02、06 |
| 5 | **启动两阶段装配**：Gateway 先于 Agent 启动，靠 `refresh_media_backends` 补媒体后端，不要删 | 02、05 |
| 6 | **IM 身份不等于授权**：所有 IM 成员都映射到 Agent 属主的 user_id，破坏性斜杠命令要判断 `is_virtual_channel` | 05 §11.4 |
| 7 | **内存态会随重启丢失**：HITL 待审批记录（TTL 30 分钟）、`turn_active`、会话策略缓存都只在内存里 | 03、05 |
| 8 | **两层串行锁**：ChannelManager 会话锁 + `AgentManager._thread_execution_lock`。新增旁路调用 `AgentManager.stream` 时，考虑是否要用 `Gateway.run_in_session`（参考 Cron） | 03 §14.4 |
| 9 | **取消不算失败**：WS `cancel` 或 `/stop` 让 harness 流正常结束，Octop 按成功轮次记账 | 03 |
| 10 | **对话正文的权威来源是 checkpoint**：`thread_messages` 只是读模型，写失败只打 warning | 06 |
| 11 | **PG 与 SQLite 事务语义不同**：多语句写入要用 `transaction()` 才能两边一致 | 06 §11.3 |
| 12 | **配置是启动快照**：`~/.octop/env` 在 `start()` 时才注入，模块顶层读 `os.environ` 拿不到其中的值 | 02 §13.2 |
| 13 | **CLI 的 Offline 写入不影响正在运行的服务**，Embedded 命令会在 CLI 进程里另起一套运行时 | 09 §11.4 |
| 14 | **流式中不要同步重编译 Agent 图**（会段错误）；移除 Agent 前要 quiesce 记忆 GC 线程 | 04 §14.5 |
| 15 | **路由注册顺序影响匹配**：`/api/users/invites`、`/roles` 必须挂在 `/api/users/{user_id:int}` 之前 | 07 |
| 16 | **JWT 中间件拒绝的请求不经过异常处理器**，所以没有本地化；422 与 `HTTPException` 不是 `{"error": …}` 信封格式 | 07 |
| 17 | **locale 默认值**：请求没指定语言时默认 `zh`，文案 key 缺失时回退 `en`；HTTP 路径只看 `Accept-Language` | 07 |
| 18 | **前端 `user_turn` 里 `mcp_servers` / `knowledge_base_ids` 的空数组有语义**（显式关闭默认项），不能省略 | 09 §11.4 |

## 5. 仓库文档与代码不符（以代码为准）

本次梳理时逐条核对过，**修改代码时别被旧文档误导**。如果以后要顺手修文档，可以单独提一个 PR。

| 文档说法 | 实际代码 |
|---|---|
| `AGENTS.md`：CLI 子命令在 `cli/*_cmd.py` | 实际在 `src/octop/cli/commands/<name>.py`，由 `cli/registry.py` 懒加载 |
| `AGENTS.md`：`api/errors.py` 负责 `OctopError` → HTTP 映射 | 该文件不存在，映射在 `src/octop/api/app.py:77-98` |
| `AGENTS.md`：DB 版本断言"currently `v == 7`" | 实际是 `v == 18`（`tests/unit/db/test_db_pool.py:102`） |
| `docs/architecture.md`：每个 Agent 一个 ChannelManager | 全进程只有一个（`infra/gateway/gateway.py:244`），渠道以 `tenant_id=agent_id` 区分 |
| `docs/architecture.md`：Cron 经 `Gateway.push_text_from_session` | Cron 用 `run_in_session` + `AgentManager.stream` + `push_session_text`（`infra/cron/delivery.py:67-173`）；`push_text_from_session` 的调用方是主动关怀 |
| `docs/configuration.md`：JWT 密钥在 `~/.octop/secrets/jwt_secret` | 存在 DB `secrets` 表的 `k="jwt"` 行（`infra/server.py:588-590`） |
| `docs/cli.md:64`：Attach 层要求 `octop user login` | 没有这个命令，也没有 Attach 层；实际是 Offline / Embedded / External，另外 `octop memory` 走本地 TCP 控制通道 |
| `docs/agent-call-agent.md`：`AgentManager.apply_mention_agent_calls` | 当前代码中已不存在；团队功能通过替换 harness TeamManager 方法实现（`infra/agents/teams/team_manager.py:1577`） |

代码自身偏离约定的地方（只记录，不要顺手改）：`UserRoleRepo` 没有挂到 `RepoBundle`，router 直接 import 它（`api/routers/user_roles.py:17`）；写 Agent 工作区的 skill 安装目标定义在 HTTP 层（`api/routers/skills.py:868`）。

## 6. 疑似问题 / 待确认（可作为练手的切入点）

| 问题 | 位置 | 状态 |
|---|---|---|
| cron 接口的 docstring 说支持 `@every`，但 `build_trigger` 不支持 | `api/routers/cron.py:127`、`infra/cron/trigger.py` | 已确认 docstring 与实现不一致 |
| 知识库 `knowledge_context_char_budget` 在 `search_knowledge` 工具路径上可能没有生效 | `infra/knowledge/params.py:15` | 待确认 |
| `octop run --workers/--reload` 实际不生效（把 app 对象交给了 `uvicorn.Server.serve()`） | `src/octop/launch.py:100-152` | 待确认是否有意为之 |
| `config.py` 的 capabilities 解析块写了两遍（结果等价） | `src/octop/config.py:518-540` | 代码冗余 |
| 取消的轮次在 legacy 历史投影里没有"已取消"标记 | `processor.py:1081` | 设计取舍 |
| 平台 `subject_id` 含冒号时，session key 的 `split(":", 3)` 会错位 | `gateway/slash/ctx.py:50-58`、`threads.py:221` | 待确认是否有平台会出现 |

> 第 1 条最适合作为第一个练手 PR：改动很小，但要走完 docstring 或 trigger、测试、`make all` 的完整流程。
