# AGENT.md

## 项目概览

这是一个 AstrBot 插件，用于监控 Minecraft 服务器状态并向指定 QQ 群推送变化通知。

- 插件入口：`main.py`
- 插件元信息：`metadata.yaml`
- 配置 schema：`_conf_schema.json`
- 使用说明：`README.md`

当前仓库几乎所有逻辑集中在 `main.py` 的 `MyPlugin` 类中，没有测试目录和独立模块拆分。

## 运行模型

### 生命周期

1. AstrBot 加载插件并实例化 `MyPlugin`。
2. `__init__` 读取配置，校验关键字段（`target_group`、`server_ip`、`server_port`）。
3. 若 `enable_auto_monitor=true` 且配置完整，延迟 5 秒创建后台监控任务。
4. 后台任务循环执行：拉取服务器状态 -> 对比缓存 -> 有变化则发群消息。
5. 插件卸载时在 `terminate` 中取消任务。

### 监控主循环

主循环函数：`direct_hello_task`。

每次循环行为：

1. `sleep(check_interval)`
2. 调用 `_fetch_server_data` 同时请求主接口与自定义接口（主接口优先，失败回退自定义）
3. 调用 `check_server_changes` 对比状态缓存
4. 变化时拼接完整通知并调用 `notify_subscribers`

状态缓存字段：

- `last_player_ids`
- `last_player_id_name_map`
- `last_status`
- `last_update_time`

## 命令与行为

以 `main.py` 为准。

- `/start_server_monitor`：启动后台监控
- `/stop_server_monitor`：停止后台监控
- `/查询`：立即查询服务器状态
- `/重置监控`：清空缓存并重置首次检测状态

## 配置项（真实来源）

来源：`_conf_schema.json` + `main.py`。

- `target_group`：目标群号（会被强制校验为纯数字字符串）
- `server_name`：展示名
- `server_ip`：服务器地址
- `server_port`：服务器端口
- `server_type`：`java` 或 `bedrock`（代码未强校验）
- `api_base_url`：自定义状态 API 基础地址（主接口失败时回退，支持基础地址或模板）
- `check_interval`：检测间隔秒数
- `enable_auto_monitor`：是否自动启动监控

代码中“必要配置”判定条件：`target_group && server_ip && server_port`。

## 外部依赖

- AstrBot API（`astrbot.api.*`）
- `aiohttp`
- 主状态 API（固定）：`https://api.mcstatus.io/v2/status/{type}/{ip}:{port}`
- 自定义状态 API（可选回退）：`api_base_url`
- AIOCQHTTP 的 `send_group_msg` 动作

## 开发约定（给后续 Agent）

### 修改原则

- 修改行为前优先阅读 `main.py`，不要只看 `README.md`。
- 涉及命令名、配置项时，要同步更新 `README.md`、`_conf_schema.json`、`metadata.yaml`。
- 新增配置项时必须给默认值、描述、hint，并在 `__init__` 中落地读取和校验。
- 监控逻辑变更必须考虑“首次检测”行为，避免刷屏。

### 推荐变更点

- 状态拉取：`_fetch_server_data`
- 消息格式：`_format_server_info`
- 变化判断：`check_server_changes`
- 群消息推送：`notify_subscribers`
- 定时行为：`direct_hello_task`

### 本地快速检查

仓库当前无自动化测试，至少执行：

```powershell
python -m py_compile main.py
```

如果在 Windows/PowerShell 下读取中文文档出现乱码，优先按 UTF-8 方式读取文件内容。

## 当前行为要点

1. 玩家进出判定以玩家 ID 集合差异为准，不再使用总人数差值作为主判断依据。
2. 状态消息会输出 `🤖 在线假人: N`，`Anonymous Player`（含前缀变体）计为假人。
3. 查询链路为“主接口优先 + 自定义接口回退”，并发发起请求以减少等待。

## 已知不一致与技术债

1. `main.py` 的 `@register` 信息与 `metadata.yaml` 不一致：
- register 名称：`minecraft_monitor`
- metadata 名称：`astrbot_plugin_minecraft_monitor`
- register 版本：`1.0.0`
- metadata 版本：`v2.0`

2. `initialize` 日志提示命令是 `/start_hello`，但实际命令是 `/start_server_monitor`。

3. `_conf_schema.json` 默认端口 `123456` 超过常规端口范围（1-65535）。

4. `server_type` 未做白名单校验，错误值会直接进入 API URL。

## 建议优先改进顺序

1. 统一命令文档与代码实际行为。
2. 统一插件元数据版本与注册信息。
3. 增加配置校验（端口范围、server_type 白名单、check_interval 下限）。
4. 视情况拆分 `main.py`（API、格式化、状态检测、命令处理）。

## 协作提示

当你接手这个仓库时，默认按以下顺序行动：

1. 先确认需求影响的是“拉取数据”“状态判定”还是“消息下发”。
2. 再定位到对应函数修改，尽量避免跨函数重复逻辑。
3. 修改后至少进行语法检查与一次手工命令回归。
4. 最后同步 `README.md`，避免文档继续漂移。
