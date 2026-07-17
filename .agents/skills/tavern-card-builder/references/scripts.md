# 配套后台脚本

脚本 = 仅 index.ts 的文件夹，后台 iframe 运行无界面；jQuery 经 `window.parent.$` 直接作用于整个酒馆页面。

## 1. 骨架模板

```ts
$(async () => {                       // 顶层 await 必须包进 $(async)——webpack 不支持裸顶层 await
  await waitGlobalInitialized('Mvu'); // 用到 Mvu 才需要
  // 注册事件监听...
});
$(window).on('pagehide', () => { /* 清理 */ });   // 禁 'unload'
export {};                            // 防与 @types 全局 ambient 冲突（TS2451）
```

脚本设置用脚本变量 + zod；脚本库按钮用 `eventOn(getButtonEvent('按钮名'), fn)`。向酒馆页面挂 UI 两模式：`createScriptIdDiv()` 沿用酒馆样式（禁 tailwind）/ `createScriptIdIframe()` 隔离样式（优先 tailwind），均配 `teleportStyle()`，pagehide 时 unmount+remove。

## 1.1 交付格式：可导入的脚本 JSON（默认）

**默认交付每支脚本一个可导入 .json**（酒馆助手→脚本库→导入），不要只给 js 代码让用户手动新建粘贴（易漏配按钮/启用状态，长代码复制易截断）。完整字段集：

```json
{
  "type": "script",
  "enabled": true,
  "name": "脚本名",
  "id": "生成一个 UUID（每支唯一）",
  "content": "…完整 JS 代码字符串…",
  "info": "给用户看的说明（可空）",
  "button": { "enabled": true, "buttons": [ { "name": "按钮名", "visible": true } ] },
  "data": {},
  "export_with": { "data": true, "button": true }
}
```

无按钮的脚本 `button` 写 `{ "enabled": false, "buttons": [] }`。`content` 里的代码在 JSON 中注意转义（引号/反斜杠/换行 `\n`）——用脚本生成 JSON 而不是手拼。组装进卡时同一对象写进 `extensions.tavern_helper.scripts` 数组。

## 2. MVU 事件钩子（5 个，精确值）

| 事件 | 值 | 签名要点 | 典型用途 |
|---|---|---|---|
| VARIABLE_INITIALIZED | `'mag_variable_initiailized'`（官方拼写就少个 l 的 i） | (variables, swipe_id) | 初始化后处理 |
| VARIABLE_UPDATE_STARTED | `'mag_variable_update_started'` | (variables) | 快照 |
| COMMAND_PARSED | `'mag_command_parsed'` | (variables, commands, message_content) | **修复命令**：去 gemini 连字符、繁转简、路径纠偏 |
| VARIABLE_UPDATE_ENDED | `'mag_variable_update_ended'` | (variables, variables_before_update) | **后处理主战场**：clamp、删归零物品、检测、清理 |
| BEFORE_MESSAGE_UPDATE | `'mag_before_message_update'` | ({variables, message_content}) | 写楼层前干预 |

**只有事件能拿到 old_variables**——限制变动幅度/检测突破/禁改某字段这类"对比前后"的逻辑，schema.ts 做不到，只能在 VARIABLE_UPDATE_ENDED 里做。

自行 `generate()` 不产生新楼层 → MVU 不自动解析 → 必须 `Mvu.parseMessage(content, old)` 后 `Mvu.replaceMvuData(new, ...)` 手动写回（指南附录 A.5 高频坑）。

## 3. iframe 寿命陷阱（额外解析模式必读）

消息楼层 iframe 寿命绑楼层——额外模型解析阶段 MVU 事件 fire 时，新楼 iframe 还没挂载订阅，**结构性抓不到**。解法：**事件捕获放常驻脚本**，经全局变量中继给 UI：

```
常驻脚本：订阅 MVU 三事件（可再包裹 generate/generateRaw 抓提示词）
  → insertOrAssignVariables({ xx_obs: {...} }, { type: 'global' })
楼层 iframe：getVariables({ type: 'global' }) 纯读渲染
```

## 4. 五个成熟脚本模式（直接抄改）

**A. 数值自动检测**：VARIABLE_UPDATE_ENDED 后跑升级/数值健康/词缀/技能四类检测 → 写 `stat_data._检测结果`（`_` 前缀=只读约定）→ 读取变量条目的 _rvHot 把它上桌（"[系统检测·务必据此升级/修复/解锁]"）。价值：替代世界书里的检测类条目，省 token。

**B. 变量观察器**：订阅 STARTED/COMMAND_PARSED/ENDED + 包裹 generate 抓变量提示词 + leaf-diff → 最近 8 轮写全局变量（如 `xx_mvu_obs`）→ 状态栏 modal 纯读展示"发送提示词/AI 原始返回/解析命令/实际落库 diff"四段。**排查"没生效"的真证据链工具**，新卡调试期强烈建议带上。

**C. 自动清理**：VARIABLE_UPDATE_ENDED 机械清理（数量=0 物品 / 过期临时词缀 / 完成日程 24h 归档 / 完成待办 48h 删 / 对话记录滚动保留）；超上限字段**用 `SillyTavern.setExtensionPrompt(ID, text, 1, 0)` 注入提醒**让 LLM 语义合并——绝不写辅助字段进 stat_data（无 catchall 会被 strip；有 catchall 也污染数据）。

**D. 骰子守卫**：AI 输出 `<骰子>d20|DC|描述</骰子>` 标签协议（`d20+`=优势、`|none|`=纯展示）→ 脚本监听 MESSAGE_RECEIVED/EDITED/SWIPED → **Mulberry32 以 `m${messageId}-t${tagIndex}` 种子**（重渲染结果不漂移）→ 状态修正（重伤-15/夜间-10）→ `setChatMessages([{message_id, message: 替换后}], {refresh:'affected'})` 就地替换 → `processing` Set 防再触发。原则：**真随机在代码侧，LLM 不会随机**；世界书检定条目管 AI 认知，脚本管落地，公式双写要留同步注释。

**E. persona 优化**：监听 `CHAT_COMPLETION_PROMPT_READY`（注意 dryRun 跳过），开局已把 persona 写入档案后，把 messages 里的 persona 正文替换为一行引用——省每轮重复 token。仅 chat completion API 有效。

## 5. 常用酒馆事件速查

`tavern_events`: `CHAT_CHANGED`（聊天切换，配 `reloadOnChatChange()`）、`MESSAGE_SENT/RECEIVED/EDITED/DELETED/UPDATED/SWIPED`、`GENERATION_STARTED(type, options, dry_run)`/`GENERATION_ENDED(message_id)`、`CHAT_COMPLETION_PROMPT_READY({chat, dryRun})`、`GENERATE_AFTER_COMBINE_PROMPTS({prompt, dryRun})`、`STREAM_TOKEN_RECEIVED`。
`eventOn` 重复注册同一 listener 被忽略；顺序用 `eventMakeFirst/eventMakeLast`；自定义事件任意字符串。
`SillyTavern.getContext()` 关键件：`setExtensionPrompt(id, content, position, depth)`、`registerMacro`、`executeSlashCommandsWithOptions`、`chat`/`chatMetadata`。ChatMessage 的 `is_system=true` 即隐藏楼层（不发 LLM）。
