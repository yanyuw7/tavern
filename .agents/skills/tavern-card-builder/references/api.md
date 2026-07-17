# 接口速查（酒馆助手 TavernHelper + SillyTavern 原生）

写脚本/前端的接口手册。**选用优先级：酒馆助手接口 > SillyTavern 原生（getContext）> STScript（triggerSlash）**——高层接口更稳定，原生接口抽象低，STScript 兜底。

## 1. 可用全局

无需 import 直接用：`$`(jQuery)、`_`(lodash)、`z`(zod)、`toastr`、`SillyTavern`、`TavernHelper`、`Mvu`（先 `await waitGlobalInitialized('Mvu')`）、`triggerSlash`、`errorCatched`、`eventOn` 族、`waitUntil`。TavernHelper 的函数同时挂在全局（`getChatMessages` 直接调即可）。

仅 iframe 内：`reloadIframe()`、`getIframeName()`（界面=`TH-message--楼层号--序号`，脚本=`TH-script--脚本名--脚本id`）、`getCurrentMessageId()`（仅楼层 iframe）、`getScriptId()`（仅脚本）、`getAllVariables()`（合并变量表）。

生命周期：加载 `$(() => {})`（禁 DOMContentLoaded——远程 load 不触发）；卸载 `$(window).on('pagehide')`（禁 unload）；顶层 `errorCatched(init)()`；聊天切换重载用 `eventOn(tavern_events.CHAT_CHANGED, () => window.location.reload())`。

## 2. 变量接口（七作用域）

```ts
type VariableOption = { type: 'global' } | { type: 'preset' } | { type: 'character' } | { type: 'chat' }
  | { type: 'message'; message_id?: number | 'latest' }   // 负数=倒数索引
  | { type: 'script'; script_id?: string } | { type: 'extension'; extension_id: string };

getVariables(option); replaceVariables(variables, option);
updateVariablesWith(updater, option);                  // 读改写一体
insertOrAssignVariables(variables, option);            // 深合并（存在则覆盖）
insertVariables(variables, option);                    // 深合并（存在则跳过）
deleteVariable(variable_path, option) => { variables, delete_occurred };
registerVariableSchema(schema, option);                // 仅 UI 校验
```

选用口诀：世界书/预设条目内用 EJS `getvar/setvar`（见 ejs.md）；前端/脚本用上面这套；MVU 数据用 `Mvu.getMvuData/replaceMvuData` 或 `defineMvuDataStore`。vue 响应式数据入库前必须 `klona()` 去 proxy。

## 3. 消息楼层

```ts
getChatMessages(range, { include_swipes? });   // range: 0 / -1 / '0-{{lastMessageId}}'
setChatMessages([{ message_id, message?, data?, is_hidden? }], { refresh: 'none'|'affected'|'all' });
createChatMessages([{ role, message, data?, name?, is_hidden? }], { insert_before?, refresh? });
deleteChatMessages(message_ids, { refresh? });
rotateChatMessages(begin, middle, end);

type ChatMessage = { message_id: number; name: string; role: 'system'|'assistant'|'user';
                     is_hidden: boolean; message: string; data: Record<string,any>/*楼层变量*/; extra: {} };
```

要点：`createChatMessages` 发用户消息后需 `triggerSlash('/trigger')` 才触发生成；`data` 字段可随消息携带变量快照；`is_hidden=true` 不发给 LLM。
显示层：`formatAsDisplayedMessage(text)`（宏+正则+markdown → 楼层 HTML）、`retrieveDisplayedMessage(id)`（JQuery 实例）、`refreshOneMessage(id)`。

## 4. 生成接口

```ts
generate(config)      // 用当前预设（含世界书/角色定义/聊天历史）
generateRaw(config)   // 不用预设，自定义顺序（仍发世界书）

type GenerateConfig = {
  generation_id?: string;            // 配 stopGenerationById
  user_input?: string;
  image?: File | string | (File|string)[];
  should_stream?: boolean;
  overrides?: Overrides;             // 覆盖 persona/char_description/scenario/chat_history 等
  injects?: Omit<InjectionPrompt,'id'>[];
  max_chat_history?: 'all' | number; // 第二 API 惯用 0
  custom_api?: { apiurl?, key?, model?, source?, max_tokens?, temperature?, ... };
  tools?: ToolDefinition[]; tool_choice?: 'auto'|'required'|'none'|...;
  json_schema?: { name, value /*JSON Schema*/, strict? };   // 强制结构化输出
};
// generateRaw 额外: ordered_prompts?: ('world_info_before'|'persona_description'|'char_description'
//   |'char_personality'|'scenario'|'world_info_after'|'dialogue_examples'|'chat_history'|'user_input'
//   | { role, content, image? })[]
```

流式：`should_stream:true` + `eventOn(iframe_events.STREAM_TOKEN_RECEIVED_FULLY, text => ...)`（还有 `_INCREMENTALLY`）。停止：`stopGenerationById(id)` / `stopAllGeneration()`。
⚠️ **自行 generate 不产生新楼层 → MVU 不自动解析**——需 `Mvu.parseMessage(content, old)` + `Mvu.replaceMvuData(new)` 手动落库（高频坑）。

## 5. 世界书新 API（造卡脚本主力）

```ts
getWorldbookNames(); getCharWorldbookNames(name); getChatWorldbookName('current');
rebindCharWorldbooks('current', { primary, additional }); getOrCreateChatWorldbook('current');
getWorldbook(name);                              // → WorldbookEntry[]
replaceWorldbook(name, entries); updateWorldbookWith(name, updater);
createWorldbook(name, entries?); createOrReplaceWorldbook(name, entries?); deleteWorldbook(name);
createWorldbookEntries(name, new_entries); deleteWorldbookEntries(name, predicate);

type WorldbookEntry = {
  uid: number; name: string; enabled: boolean;
  strategy: { type: 'constant'|'selective'|'vectorized';    // 蓝灯/绿灯/向量化
              keys: (string|RegExp)[];
              keys_secondary: { logic: 'and_any'|'and_all'|'not_all'|'not_any'; keys: [] };
              scan_depth: 'same_as_global'|number };
  position: { type: 'before_character_definition'|'after_character_definition'
                  |'before_example_messages'|'after_example_messages'
                  |'before_author_note'|'after_author_note'|'at_depth'|'outlet';
              role: 'system'|'assistant'|'user'; depth: number; order: number };
  content: string; probability: number;
  recursion: { prevent_incoming: boolean; prevent_outgoing: boolean; delay_until: null|number };
  effect: { sticky: null|number; cooldown: null|number; delay: null|number };
};
```

旧 API（getLorebookEntries 族）已废弃，一律用新的。程序化读禁用条目拼第二 API 上下文就靠 `getWorldbook` + 按 `name` 前缀过滤（见 second-api.md）。

## 6. 提示词注入（两套，别混）

```ts
// 酒馆助手（持久化到聊天文件，跨轮生效）
injectPrompts([{ id, position: 'in_chat'|'none', depth, role, content, should_scan? }], { once? });
uninjectPrompts(ids);
// 'none' 用于仅触发世界书绿灯扫描而不实际插入

// SillyTavern 原生（内存态，脚本生命周期内）
SillyTavern.getContext().setExtensionPrompt(id, content, position, depth, scan?, role?, filter?);
```

自动清理脚本给 LLM 的"超限提醒"用 setExtensionPrompt（不落存档）；需要跨会话持久的注入用 injectPrompts。

## 7. 酒馆正则接口（程序化管理 regex_scripts）

```ts
getTavernRegexes({ type: 'global'|'character'|'preset', name? });
replaceTavernRegexes(regexes, option); updateTavernRegexesWith(updater, option);
formatAsTavernRegexedString(text, source, destination);  // source: 'user_input'|'ai_output'|...; destination: 'display'|'prompt'
isCharacterTavernRegexesEnabled();
```

写正则内容规范见 regex.md；这里是"从脚本增删改正则"的接口。

## 8. 其他常用（简表）

| 类 | 关键函数 |
|---|---|
| 角色卡 | `getCharData(name)` / `getCharacter` / `createOrReplaceCharacter`；Character.extensions 里有 `regex_scripts` 与 `tavern_helper.scripts` |
| 预设 | `getPresetNames` / `loadPreset` / `getPreset` / `replacePreset` / `setPreset(name, partial)` |
| 音频 | `playAudio` / `pauseAudio` / `replaceAudioList` / `setAudioSettings`（BGM/环境音） |
| 宏 | 原生 `SillyTavern.getContext().registerMacro(key, value)`（`{{key}}`）；酒馆助手 `registerMacroLike(regex, replaceFn)`（正则宏） |
| 脚本按钮 | `eventOn(getButtonEvent('按钮名'), fn)`；`replaceScriptButtons` / `getScriptButtons` |
| 全局共享 | `initializeGlobal(name, value)` / `await waitGlobalInitialized(name)`——跨 iframe 共享接口（MVU 就这么注册的） |
| Slash | `triggerSlash('/trigger')`——STScript 命令，管道返回结果 |
| 工具 | `substitudeMacros(text)`、`getLastMessageId()`、`errorCatched(fn)` |
| 导入 | `importRawCharacter/Chat/Preset/Worldbook/TavernRegex(filename, blob)` |

## 9. SillyTavern.getContext()（原生，第二优先级）

| 成员 | 用途 |
|---|---|
| `chat: ChatMessage[]` | 全部楼层（原生结构，见下） |
| `characters` / `characterId` / `chatId` / `getCurrentChatId()` | 当前卡与聊天 |
| `chatMetadata` | 聊天元数据（含 variables） |
| `eventSource` / `eventTypes` | 原生事件系统 |
| `Popup` / `POPUP_TYPE` / `callGenericPopup` | 弹窗 |
| `registerMacro(key, value, desc?)` | 注册 `{{key}}` 宏 |
| `registerFunctionTool(tool)` | 注册 LLM 函数工具 |
| `executeSlashCommandsWithOptions(text)` | 跑 STScript |
| `setExtensionPrompt(...)` | 见 §6 |
| `generate` / `sendStreamingRequest` / `stopGeneration` | 原生生成 |
| `loadWorldInfo` / `saveWorldInfo` / `getWorldInfoPrompt` | 原生世界书（优先用 §5 新 API） |
| `messageFormatting(mes, name, isSys, isUser, id)` | 文本→楼层 HTML |
| `substituteParams(content)` | 替换宏 |

原生 ChatMessage：`{ name, is_user, is_system /*=隐藏不发LLM*/, mes, swipe_id?, swipes?, extra?, variables? }`；role 推导：narrator→system、is_user→user、其余→assistant。

## 10. 事件系统

```ts
const { stop } = eventOn(event, listener);   // 重复注册同一 listener 被忽略
eventOnce / eventMakeFirst / eventMakeLast   // 顺序控制
await eventEmit(event, ...data);             // 自定义事件任意字符串
eventRemoveListener / eventClearEvent / eventClearListener / eventClearAll
```

高频 `tavern_events`：`APP_READY`、`CHAT_CHANGED(file)`、`MESSAGE_SENT/RECEIVED/EDITED/DELETED/UPDATED/SWIPED(id)`、`GENERATION_STARTED(type, options, dry_run)`、`GENERATION_ENDED(id)`、`STREAM_TOKEN_RECEIVED(text)`、`CHAT_COMPLETION_PROMPT_READY({chat, dryRun})`（**注意判 dryRun 跳过**）、`GENERATE_AFTER_COMBINE_PROMPTS({prompt, dryRun})`、`WORLDINFO_UPDATED(name, data)`、`WORLD_INFO_ACTIVATED(entries)`、`PRESET_CHANGED`。
`iframe_events`：`MESSAGE_IFRAME_RENDER_STARTED/ENDED`、`GENERATION_STARTED/ENDED(text, id)`、`STREAM_TOKEN_RECEIVED_FULLY/INCREMENTALLY`。
MVU 事件 5 个见 scripts.md §2。
