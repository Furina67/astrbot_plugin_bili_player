# 更新日志

## 0.2.11 - 2026-11-11

- **修复“回复序号后不被插件接管、被 LLM 接话”**。两个独立原因：
  1. **选歌等待只有 90 秒**。用户看完 10 条候选再回复，等待早已超时结束，消息正常流向 LLM。窗口改为 **300 秒**。
  2. **序号正则不容忍引号/书名号**。从目录里复制 `“3”` 或粘贴带引号的序号时匹配失败，`_is_selection_reply()` 返回 `False`，filter 不认领，同样落到 LLM。现在首尾允许引号、书名号、括号、空白与中文标点。
- `_SELECTION_EDGE_CHARS` 集中定义可容忍的边缘字符，正则首尾各放宽一次，`取消` 仍单独判定。
- 保持 0.2.10 的全部修复：只有 `取消` 才触发丢弃 handler、解析失败不续期、`keep` 用 `reset_timeout=False` 叠加而非重置、每个终止分支都 `controller.stop()`。
## 0.2.10 - 2026-11-11

- **修复“无限重复发送选歌提示”死循环**。`_DiscardSelectionWaitFilter.filter()` 原先对所有非序号消息都返回 `True`（只要有活跃的选歌 filter 就认领）。`custom_filter` 的返回值决定 handler 是否执行，等于把群内所有闲聊都判成选歌回复：消息被交给 waiter → 解析失败 → 发送帮助文本 → 续期 → 超时被无限重置，会话永不结束。现在只有用户显式回复 `取消` 才会触发该 handler。
- `_deliver_selection` 中解析失败的分支**不再续期**等待（能走到这里只说明有 filter 错误认领，续期会把错误放大成死循环）；时长超限提示分支保留会话但**不重置超时**（`reset_timeout=False`，在剩余时间上叠加）。
- `_renew_selection_wait` 新增 keyword-only 参数 `reset_timeout`，默认 `False`，与官方文档中 `keep()` “叠加而非重置”的语义一致。
- `stop_wait` 语义改为 `keep_open`，`finally` 中 `if not keep_open: controller.stop()`，保证每个终止分支都会注销 waiter，避免 filter 残留导致后续消息再次进入 handler。
## 0.2.9 - 2026-11-11

- **紧急修复 `search_song` 崩溃**：0.2.8 的注册轮询误用了未导入的 `USER_SESSIONS`，报 `name is not defined`。
- 轮询条件改为检查模块级 `FILTERS` 中是否已出现本轮的 `_SelectionSessionFilter` 实例（由 `register_wait` 写入），语义等价且不再依赖 AstrBot 内部全局变量，跨版本更稳。

## 0.2.8 - 2026-11-11

- **真正根因修复：自定义 `SessionFilter` 从未被注册**。`@session_waiter` 未传入 `session_filter`，装饰器于是回退到内置 `DefaultSessionFilter`，插件里写好的 `_SelectionSessionFilter`（按会话 ID + 序号语法校验）从头到尾没有被实例化，`_DiscardSelectionWaitFilter` 也就永远找不到目标而沦为死代码。现在把 `selection_filter = _SelectionSessionFilter(session_id)` 显式传入并为 `_run_selection_wait` 增加对应形参，`await waiter(source_event, selection_filter)` 按位置传参（关键字会让装饰器回退到默认 filter）。
- `_NullSelectionController.stop()` 由空操作改为真实实现（置 `stopped` 标志），使 `_cancel_selection_wait()` 能真正取消旧 waiter，避免新旧 waiter 互相覆盖、旧 `_cleanup()` 误清新 waiter。
- 候选列表发出前的 `await asyncio.sleep(0)` 改为最多 0.5 秒的注册轮询（50 × 10ms）：确认 `USER_SESSIONS` 已写入或 waiter 已结束后再返回，消除“注册尚未完成就发列表”的竞态。
- `metadata.yaml` 的 `repo` 指向更新为 `Furina67/astrbot_plugin_bili_player`，与插件市场实际发布仓库一致。

## 0.2.7 - 2026-11-11

- **彻底修复：搜索歌曲后回复裸序号（如「2」）被 LLM 抢答**。上一版只对命令事件调用 `should_call_llm(False)`，但「2」是一条独立的新消息，不受影响；本次改用 AstrBot 官方的会话控制机制接管选择轮次。
- `_start_selection_wait()` 改为用 `@session_waiter(timeout=…, record_history_chains=False)` 装饰内部 `select_song`，并由 `asyncio.ensure_future` 驱动，不再手写 `FILTERS.append` + `asyncio.create_task` + `asyncio.sleep(0)`——旧写法无法保证 waiter 在候选列表发出之前完成注册，导致序号消息到达时无人接管。
- `_run_selection_wait()` 的 `finally` 新增 `source_event.stop_event()`，`_deliver_selection()` 的 `finally` 新增 `reply.stop_event()`：**消息即使被 waiter 接住，也要显式拦下**，否则事件仍会继续下发到 LLM。
- 新增 `_NullSelectionController`（`keep()` / `stop()` 均为空操作），用于 waiter 真正运行前 `_SelectionWait.controller` 的合法占位；`_SelectionWait` 语义不变。
- 导入改为兼容写法：优先 `from astrbot.api.util import session_waiter`（AstrBot v4.30.0+），回退 `astrbot.core.utils.session_waiter`，老版本仍可用。

## 0.2.6 - 2026-11-11

- **修复命令事件被误交给 LLM**：搜索歌曲 / 搜索视频 / 点听 / 点看 执行后不再调用 should_call_llm(True)，搜索回合由插件独占，候选列表之后的裸序号回复（如 3）改由会话控制路由到选歌流程，不再被模型当成普通聊天。
- 候选列表发出后补 stop_event()，避免同一条消息继续流入下游处理链。
- 新增 _renew_selection_wait()：非终态回复（序号不合法、音频超长提示）时调用 SessionController.keep() 续期，不再固定 90 秒后静默失效。

## 0.2.5 - 2026-08-24

- 交付前改由插件主动发送“正在准备音频/视频，请稍候。”提示，再下载、转码与发送媒体；媒体送达后保持静默，彻底移除不可靠的 LLM 开场白与后置收尾时序。
- 移除 `delivery_reply` 配置项，避免出现无效或延迟的 LLM 反馈设置。

## 0.2.4 - 2026-08-24
- 视频交付优先选择最低码率档（AVC 优先、同档取最低带宽），缩短下载与发送耗时。
- 音频文件交付（“序号 音频下载”或配置为直接发文件）自动选择最高可用码率档，语音播放仍保持 192kbps 封顶。
- 扫码登录兼容 B 站更新后的凭证下发：确认后从跳转 URL 的查询参数提取 `SESSDATA` 等凭证，并保留 Set-Cookie 兼容路径。
- WebUI 新增“手动粘贴 Cookie 登录”：凭证即时校验，无效会被拒绝；README 文档化 `accounts.json` 格式。
- aiocqhttp 部署的语音改为直接向 OneBot 传本地 `file://` URI，由 NapCat 自行读取，避免完整歌曲转 Base64 后超出 WebSocket 消息上限导致断连；失败仍降级为文件。
- LLM 交付失败后保留搜索租约，模型可用同一 `search_id` 重试；仅在媒体发送成功后消耗。
- 搜索结果列表改为 `序号. [时长] 标题(截断)`；新增 `show_uploader` 配置（默认关闭），开启后列表附带 up 主名（最多 20 字截断），便于识别翻唱；LLM 候选证据仍不含上传者。

## 0.2.3 - 2026-08-24

- 视频与音频文件默认大小调整为 50 MiB。
- `limits` 配置项扁平化，原生配置面板中的恢复默认值按钮与普通字段对齐，布局更美观。

## 0.2.2 - 2026-08-24

- 交付限制加入原生配置面板：语音/视频/音频文件的时长与体积可在安全范围内调整。
- 时长填 0 表示不限；网络超时、并发等底层预算仍保持固定。

## 0.2.1 - 2026-08-24

- 新增 `delivery_reply` 配置：`none` 只发媒体（默认），`llm` 在媒体发送后让 LLM 补一句自然收尾。
- 交付工具描述随配置生成，`llm` 模式下要求模型只回一句、不复述歌名或发送过程。

## 0.2.0 - 2026-08-23

- 接入 AstrBot 原生插件配置面板：未明确媒体类型时的默认值，以及“仅视频/仅音频”能力门。
- 音频未指定形式时可配置“优先语音，超限或平台不支持时发文件”或“直接发音频文件”。
- `find_in_bilibili` 的 delivery 改为 `auto/video/audio/download`；`deliver_media` 不再接收 delivery，避免模型在交付阶段改写媒体意图。
- 交付成功后工具返回 `None` 终止 Agent Loop，不再出现“已发送”第三条重复消息。
- 直接交付不再发送预告文字，只发送视频/语音/文件本身，彻底消除“预告 + 媒体 + 复述”三连。
- 防抖合并产生的多请求消息不再猜测任务边界：多个 AV/BV 时把整段消息作为模糊关键词搜索，并强制用户自选结果；同轮重复 `find_in_bilibili` 不会覆盖当前搜索租约。
- 直听搜索保留长音频候选，超 15 分钟自动以文件交付；候选列表语法随配置动态生成。
- 插件 Page 增加中文标题与描述，便于从插件详情页进入“音乐账号”。

## 0.1.5 - 2026-08-23

- 点播交互收敛为三个交付动作：默认发送视频（视频文件可自行保存），只有“我要听/播放”才优先发送可直接播放的音频，“音频下载”单独由用户确认。
- 选择语法统一为：回复“序号”发视频、“序号 音频”播放音频、“序号 音频下载”下载音频文件；移除“下载视频”，超长候选标记改为“音频仅可下载”。
- 平台不适配时，语音和视频都会自动回退为文件交付；“我要听”的超长音轨直接交付音频文件兜底。
- 音频下载永不自动：即使模型未设置 `let_user_choose`，download 意图也强制进入用户选择路径。
- 工具参数不再从 note 文本猜测交付形式，行为可预测；同步更新 README 与架构、开发文档及测试。
- 清理交付形式的模糊文本匹配层：工具契约已是结构化枚举，改为显式映射；精简预告文案黑名单，不再误伤含“播放/发送”的正常祝语。
- 精确 AV/BV 多分 P 强制进入用户选择；download 前置搜索不再按语音时长过滤长音频，长音频兜底与视频超限提示更准确。

## 0.1.3 - 2026-07-15

- 补充仓库元数据，支持 AstrBot 记录 GitHub 安装来源并检查更新。
- 新增本更新日志及仓库安装说明。
- 语音保持 15 分钟、25 MiB 的发送边界；显式下载不再设音频时长上限，单文件最多 100 MiB。
- 搜索结果中的长音频会标记“仅可下载”，发送后仍会自动清理临时文件。
- 备用 CDN 与主线路共用一次交付的等待时间，避免下载在多个线路间无限延长。
- 完整条件搜索结果不足时，自动以歌名补充候选并保留首轮结果顺序。
- 支持 AV/BV 号和普通 Bilibili 视频链接精确定位视频分 P 后下载。
- 保留现场、翻唱、AI 创作等候选标题，按用户请求选择版本。

## 0.1.2 - 2026-07-15

- 不支持语音消息的适配器会自动改为发送同一首歌的音频文件。

## 0.1.1 - 2026-07-15

- 首次发布“我想听歌！”。
