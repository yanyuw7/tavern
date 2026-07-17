# 第二 API / 多 AI 架构

主线 AI 之外再挂独立模型：分担变量解析、扮演敌方阵营、离线 NPC、论坛内容、世界推进。三种成熟形态：任务流水线型、敌方 AI 标签路由型、平板应用型（多流水线+分档反馈）。

## 0. 总原则

- **主线 AI 是唯一剧情权威**。副 AI 只产 JSONPatch 或 XML 包裹的纯文本，绝不产叙事正文（防 split-brain / 双倍 token / 数据不一致）。
- 玩家操作优先翻译成主输入走主线，只有后台任务才走第二 API。
- API 凭据存 **localStorage**（可 base64 混淆），**永不入变量树、不入卡**。iframe 的 localStorage 可能被清——读写优先 host window 并做旧数据迁移（host window 双写 + 旧数据迁移）。
- 按任务分配模型：高频轻量任务（润色/意图分析）配便宜模型，一次性重任务配高质量模型——`FunctionId → endpoint` 映射表。

## 1. 调用规范

```ts
generateRaw({
  ordered_prompts: [ { role:'system', content: 严格输出契约 }, { role:'user', content: 情境 } ],
  max_chat_history: 0,          // 不带聊天历史，上下文全由你拼
  custom_api: { apiurl, key, model, source: 'openai', max_tokens },
});
```

配套：`testConnection`（Promise.race 30s 超时 + 错误分类给中文提示）、`fetchModels`（官方 getModelList 优先、原生 fetch /models 兜底）。

## 2. 规则供给：任务前缀命名空间

变量规则按任务拆多份世界书条目：`[mvu_general/combat/relation/world/npc/common]`，**全部 disabled + order 10050+**——永不进主提示词；前端 `getEntriesForTask(prefixes, ..., includeDisabled=true)` 按前缀读**禁用条目**拼给第二 API（**前缀分层直接成为程序化索引**）。拼前 `cleanTavernMacros` 剥 `{{get_message_variable::…}}` 宏。每份规则文件头写职责声明（由 X 任务读取 / 包含 / 不含）。

任务流水线（stage 升序串行，前面任务的 patch 先应用，后面任务看到新状态）：

| stage | 任务 | 触发 | 读的前缀 |
|---|---|---|---|
| 1 | variable_update | 每楼 | [mvu_general]+[mvu_common] |
| 2 | combat_resolution | 事件: battle_ended（状态 diff 检测：prev.是否战斗中===true && curr!==true） | [mvu_combat]+[mvu_common] |
| 3 | relationship_update | 每 5 楼 | [mvu_relation]+[mvu_common] |
| 4 | world_advance | 每 10 楼 | [mvu_world]+[mvu_common] |
| 5 | offscreen_npc_advance | 每 10 楼 | [mvu_npc]+[mvu_common] |

TaskContext 同时带 maintext 和 **userInput 玩家原话**（防主 AI 改写吞掉玩家明确指令）。key 触发的补充条目只对 maintext `includes()` 命中才注入。

## 3. 输出回收三件套（防越权防复读）

1. **标签路由**：主 AI 回复里写 `<敌方fac_001>给该阵营的情境私信</敌方fac_001>` → 脚本 MESSAGE_RECEIVED 后正则 `/<敌方(fac_\d+)>([\s\S]*?)<\/敌方\1>/g`（**反向引用保证开闭一致**）提取 → 内容直接当 user prompt 并行发各阵营 API。信息不对称由主 AI 控制（它决定告诉敌方什么）。
2. **越权拒绝双保险**：副 AI system prompt 明令"只输出一个 `<UpdateVariable><JSONPatch>` 块、禁止叙事、仅修改 /敌方阵营/<你的facId>/*"；**代码侧再硬校验** `path.startsWith('/敌方阵营/')` 否则丢弃。prompt 约束会被无视，白名单才是底线。
3. **正则双重隐藏**：同一正则两条配置——`显示隐藏`（markdownOnly）玩家看不到中间标签；`对AI隐藏`（promptOnly）主 AI 下轮看不到自己上轮写的标签，**防复读**。

## 4. 变量写入接缝（USE_MVU_PARSE 开关模式）

第二 API 返回的 patch 怎么落库，留一个布尔开关：

- **fallback 自实现**（schema 宽松期）：正则抽 `<UpdateVariable>` → 自写 applyJsonPatch → `replaceVariables`。绕过 schema 校验、不发 MVU 事件——检测类脚本联动不了。
- **官方管线**（schema 严格化后切换）：`Mvu.getMvuData` → `await Mvu.parseMessage(rawResponse, oldData)` → `await Mvu.replaceMvuData(newData)` → `eventEmit(Mvu.events.VARIABLE_UPDATE_ENDED, newData, oldData)`——带 schema 校验 + 事件钩子。
- 前置条件：schema 完成 prefault 迁移（否则冻结数组 in-place 修改会炸）；切换需重导卡使新 schema 生效。

## 5. 玩家操作反馈四档

UI 操作翻译成 `《指令名:参数》 主体` 格式，按重要度分档处理：

| 档 | 处理 |
|---|---|
| 完全静默 | 仅 patch 变量（设闹钟/屏蔽/取关） |
| Toast | patch + UI 弹提示（点赞/关注/购买） |
| 轻剧情 | patch + 入队 `pending_actions[]`，下次 GENERATION_STARTED 合并成**一条** `[平板批量操作]` inject（私信/评论/打赏） |
| 交互 | push 到 `#send_textarea` 追加不覆盖、不自动发送，玩家补话后自己发 |

数据分四层：客观世界=stat_data（message 变量）；应用本地=chat 变量（tablet_data）；玩家跨卡偏好=global 变量；凭据+UI 位置=localStorage。
业务规则住卡的 `[mvu_update]` 条目专章，应用本体只管 UI+调度+默认 prompt——卡作者按协议补一段即可接入（§9/§11 解耦）。
卡级定制三层覆盖：`window.XXX_OVERRIDE > window.XXX_PRESETS > 内置默认`，卡作者写 30-50 行 override 接卡。

## 6. LLM 输出铁律（三处独立验证过）

- 只准输出 schema 已存在的真实路径；禁自由设计 JSON key（AI 会把 view-shaped JSON 的"玩家/在场"当新路径污染 stat_data）。
- 变量更新 → `<JSONPatch>` 真实路径；非变量数据 → **XML 业务标签包纯文本**（`<EventLog>`、`<post idx="N">`），简单值用分隔符（`UID / 倾向`）；标签命名避开 `<a>`/`<b>` 通用名。
- 副 AI 的输出契约写进 system prompt 并给一个正例 + 明确"除此之外不输出任何东西"。
