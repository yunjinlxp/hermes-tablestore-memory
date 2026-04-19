# Hermes TableStore 记忆插件

基于 TableStore 的 Hermes Agent 外接记忆提供器。

这个插件使用官方 `tablestore` Python SDK，并接入 Hermes 的 memory
provider 接口，提供：

- 基于 TableStore Memory API 的长期语义记忆
- 每轮对话完成后的自动记忆写入
- 下一轮对话前的相关记忆预取
- 显式的记忆查询、写入、删除工具

Hermes 内建的 `MEMORY.md` 和 `USER.md` 仍然保留并继续工作；这个插件是在
内建记忆之上增加一个外部记忆后端。

## 功能

- `tablestore_profile`
  查看当前 scope 下已有的记忆。
- `tablestore_search`
  对记忆做语义检索。
- `tablestore_remember`
  写入一条长期记忆。
- `tablestore_forget`
  按 id 删除一条记忆。
- 自动 `sync_turn()`
  在一轮对话完成后，把用户消息和助手回复写入 TableStore。
- 自动 `queue_prefetch()`
  在下一轮前预取相关记忆。
- Hermes 内建 `memory` 工具的写入会自动镜像到 TableStore。

## 依赖条件

- 已安装 Hermes Agent
- Hermes 所用的 Python 环境能够安装 `tablestore==6.4.5`
- 可访问的 TableStore OTS endpoint，并启用了 Memory API
- 有权限访问目标实例的 AK/SK
- 已存在的 memory store，或者具有创建它的权限

## 安装

### 推荐方式：从 GitHub 安装

```bash
hermes plugins install yourname/hermes-tablestore-memory
```

或者：

```bash
hermes plugins install https://github.com/yourname/hermes-tablestore-memory.git
```

安装后，Hermes 会把插件放到：

```text
~/.hermes/plugins/tablestore-mem/
```

### 手动安装

把这个仓库整体复制到：

```text
~/.hermes/plugins/tablestore-mem/
```

然后在 Hermes 使用的 Python 环境里安装依赖：

```bash
uv pip install --python "$(which python)" tablestore==6.4.5
```

如果 Hermes 运行在单独 venv 里，请把 `python` 换成对应解释器。

## 启用

交互式启用：

```bash
hermes memory setup
```

然后在列表里选择 `tablestore-mem`。

或者手动启用：

```bash
hermes config set memory.provider tablestore-mem
```

## 配置

密钥类配置建议放到：

```text
~/.hermes/.env
```

非敏感配置建议放到：

```text
$HERMES_HOME/tablestore_memory.json
```

### 必填环境变量

```bash
TABLESTORE_MEMORY_AK=your_access_key_id
TABLESTORE_MEMORY_SK=your_access_key_secret
TABLESTORE_MEMORY_ENDPOINT=https://your-instance.region.ots.aliyuncs.com
TABLESTORE_MEMORY_INSTANCE=your-instance-name
TABLESTORE_MEMORY_STORE=hermes_mem
```

### 可选环境变量

```bash
TABLESTORE_MEMORY_APP_ID=hermes
TABLESTORE_MEMORY_TENANT_ID=
TABLESTORE_MEMORY_DESCRIPTION=
TABLESTORE_MEMORY_ENABLE_RERANK=true
TABLESTORE_MEMORY_AUTO_CREATE_STORE=true
TABLESTORE_MEMORY_TIMEOUT=30
```

### `tablestore_memory.json` 示例

```json
{
  "endpoint": "https://your-instance.region.ots.aliyuncs.com",
  "instance_name": "your-instance-name",
  "memory_store_name": "hermes_mem",
  "app_id": "hermes",
  "enable_rerank": true,
  "auto_create_store": true,
  "timeout": 30.0
}
```

如果用户没有指定记忆库名，插件会默认使用 `hermes_mem`，并在缺失时自动创建。

## Scope 设计

插件使用的 TableStore scope 为：

```text
appId / tenantId / agentId / runId
```

当前四个字段的获取方式如下：

- `appId`
  来源：优先取用户配置 `TABLESTORE_MEMORY_APP_ID` 或 `app_id`
  默认值：`hermes`
- `tenantId`
  来源：优先取 Hermes 会话中的 `user_id`
  回退：用户配置 `TABLESTORE_MEMORY_TENANT_ID` 或 `tenant_id`
  默认值：`__default__`
- `agentId`
  来源：Hermes 当前会话身份，当前实现里主要是 `agent_identity`
  默认值：`hermes`
- `runId`
  来源优先级：
  1. `gateway_session_key`
  2. `session_title`
  3. 当前 `session_id`
  默认值：仅在以上都为空时回退到 `__default__`

也就是说，真正需要用户配置的主要是 `appId` 和 `tenantId`；`agentId` 与
`runId` 默认由 Hermes 会话上下文提供，不建议用户手工管理。

另外，写入和检索使用不同的 scope 策略：

- 写入时：使用当前会话的精确 scope
- 检索时：在当前 `tenantId` 下，将 `agentId=*`、`runId=*`

这样可以实现：

- 记忆写入时保留精确归属
- 检索时按租户维度跨 agent、跨会话召回

## 验证安装

先确认 Hermes 已识别插件：

```bash
hermes memory status
```

你应该能看到：

```text
Provider:  tablestore-mem
Plugin:    installed
Status:    available
```

之后可以在 Hermes 对话里直接使用，也可以用 CLI 命令验证。

如果插件已经安装，但 Hermes 所使用的 Python 环境里还没有 `tablestore`
依赖，需要先安装：

```bash
uv pip install --python /path/to/hermes/venv/bin/python tablestore==6.4.5
```

## CLI 命令

当 `memory.provider` 设置为 `tablestore-mem` 后，可以直接使用：

```bash
hermes tablestore-mem add "用户偏好简洁回答"
hermes tablestore-mem add "用户喜欢 Rust" --metadata source=manual --metadata topic=preferences
hermes tablestore-mem add "同步写入这条记忆" --sync
hermes tablestore-mem search "简洁回答"
hermes tablestore-mem search "Rust" --top-k 10
```

说明：

- `hermes tablestore-mem add` 通过 provider 写入一条记忆
- `hermes tablestore-mem add` 默认异步写入；传入 `--sync` 才会等待写入完成
- `hermes tablestore-mem search` 返回 JSON 检索结果
- `--metadata KEY=VALUE` 可以重复使用
- 这些 CLI 命令只有在 `tablestore-mem` 是当前激活的 memory provider 时才会注册

## 运行说明

- 插件直接调用 `tablestore.OTSClient` 的 memory 方法
- OTS SDK 负责请求签名和鉴权
- `is_available()` 只检查本地配置是否存在，不做网络请求，符合 Hermes 插件约定
- `sync_turn()` 使用后台线程，避免阻塞主代理循环
- 显式 `tablestore_remember` 写入默认也是异步，除非传入 `sync=true`
- 搜索和预取默认启用 rerank，可通过配置或调用参数关闭
- `tablestore_remember` 的一次写入，后端可能会抽取成多条结构化记忆单元

## 常见问题

### 插件显示 “not available”

通常是以下配置缺失：

- `TABLESTORE_MEMORY_ENDPOINT`
- `TABLESTORE_MEMORY_INSTANCE`
- `TABLESTORE_MEMORY_AK`
- `TABLESTORE_MEMORY_SK`

运行：

```bash
hermes memory status
```

### 鉴权失败

请检查：

- endpoint 是否真的是 OTS endpoint，而不是别的业务网关
- instance name 是否正确
- AK/SK 是否对该实例有权限
- SDK 版本是否为 `tablestore==6.4.5`

### 搜索结果不是原始输入文本

这是正常现象。`AddMemories` 可能会把原始输入抽取、归一化成多条结构化记忆，
所以搜索返回的可能是抽取后的事实，而不是完全逐字一致的原文。

### 首次写入较慢

如果默认库 `hermes_mem` 尚不存在，且开启了自动创建，那么第一次写入通常会比
普通请求更慢，因为插件会先创建记忆库，再执行写入。当前默认超时是 `30` 秒。

## 仓库结构

这个仓库已经按 Hermes 可直接安装的格式组织好了：

```text
.
├── __init__.py
├── cli.py
├── CHANGELOG.md
├── plugin.yaml
├── README.md
├── README.zh-CN.md
├── after-install.md
└── LICENSE
```

仓库根目录就是插件根目录。

## 许可证

本项目采用 MIT License，详见 [LICENSE](./LICENSE)。
