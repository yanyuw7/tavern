# EJS 条目写法（ST-Prompt-Template）

世界书/预设条目里嵌 JS。两个处理时机：生成时（进 prompt 前）与渲染时（显示前）。造卡最常用三类：读取变量条目、守卫条目、动态注入。

## 1. 作用域两派（按卡走，绝不混）

ST-Prompt-Template 把同一条 prompt 里的多个 EJS entry 拼进**同一个 AsyncFunction 作用域**编译：

- **无 @@private 派**：顶层变量**必须 `var`**（函数作用域允许重复声明）；`const`/`let` 会在第二个 entry 抛 `SyntaxError: Identifier already declared` 整条 prompt 编译失败。配"重复声明守卫"：`if (typeof stat === 'undefined') var stat = getvar('stat_data');`——这类守卫是关键设计不是冗余，别删。
- **@@private 派**：entry 首行 `@@private` → 编译进独立作用域 → const/let 安全。

改哪张卡用哪张卡现有的派系；新增 entry 前先看邻居条目怎么写。变量访问器同理按卡：`getvar('stat_data')`（带 `typeof getvar==='function'` 守卫）vs `variables.stat_data || variables`。

## 2. 语法硬约束（三条都过才算健康）

1. 代码块内 const/let 数量符合该卡派系（无 private 派应为 0）。
2. 反引号模板字符串里**禁 `${}`**——EJS 先吃一遍，未定义变量直接抛错。占位符用 `[UID]`/`[楼层ID]` 方括号写法。
3. JS 字符串里**禁 `\n` 字面量**——EJS 编译阶段把 `\n` 当真换行，字符串语法裂开。要换行用模板自然换行 + `<% for %>` 循环 + `<%- x %>` 输出，或 `String.fromCharCode(10)`。

## 3. 读取变量条目（核心标准件，必须无前缀）

职责：把 stat_data 裁剪后发给 AI。规模小用官方宏 `{{get_message_variable::stat_data}}` 全量直读；实体多了必须 EJS 过滤，否则 token 爆炸 + lost-in-the-middle。

**salience 三层结构**：

```
① _rvHot[]  热事务上桌（置顶）：扫非常态事务——异常设施状态/进行中任务/世界事件/_检测结果，
   输出"⚡当前进行中/非常态事务（优先据此演出，勿当作未发生）"
② 全量区：活跃实体（在场/随行/最近N楼提到的）选进 filtered 对象 → JSON.stringify(filtered, null, 2)
   包在 <status_current_variables> 标签里（配正则从下轮 prompt 剥除）
③ _rvBrief[] 压缩区：不活跃实体移出 JSON，一行一条摘要放
   "==== 已知但不活跃（仅供参考·非变量结构）===="文本区
   ⚠️ 必须附警告："严禁对下列文本中的名字/UID 生成 JSONPatch，更新须用真实路径 /xxx/${UID}/..."
```

绝不把摘要对象（`_总览:true` 之类）塞回变量 JSON——AI 会当真实结构去 patch。

**过滤策略参考**：详情集 = 玩家 ∪ 在场角色（可再加 最近4楼提到名字的）；组织按"玩家所在 + 在场角色所在"过滤，秘密身份组织仅同组织可见；空容器/待生成字段删；全默认值的子档案整块删（某块字段全是初始默认 = 没发生过任何相关剧情 → 不发，省 token）。

**salience 铁律**：任何"AI 必须看到才能动作"的深埋变量（脚本写的 `_检测结果` 等），必须有 _rvHot 上桌通道——变量"可查询"不等于"AI 看见"，提示词里没有 = 不存在。

尾部可挂自检提醒块（`<status_sync_check>`：正文实体 → 变量对应逐项勾）。

## 4. 守卫条目族（EJS 算硬约束输出禁令）

用变量实时算出强制规则文本，比纯文字规则可靠（资源/死亡/召唤/载具守卫同型）：

```
<%_ if (typeof stat === 'undefined') var stat = getvar('stat_data'); _%>
【资源状态】金币: <%- 金币 %> | 燃料: <%- 燃料 %>
<%_ if (金币 <= 0) { _%>
⛔ 金币为0——禁止任何购买行为，不可凭空获得资源
<%_ } _%>
```

配置 `depth=0` 贴生成点注入。变体：数值锁定（体力 0 禁消耗体力行动）/死亡锁定/召唤次数/EP 余额。
公式双写锚：EJS 条目与前端各算一份同一公式时，两处都留"⚠️ 与 xxx.ts 同步（搜 lootMult），改时两处都改"注释。

## 5. 处理时机与注入控制

两个处理时机：**生成时**（提示词构造好后、发给 LLM 前执行 `<% %>`）与**渲染时**（收到完整输出后对楼层 HTML 执行，只改显示不改原文）。

条目标题前缀注入：

| 前缀 | 含义 |
|---|---|
| `[GENERATE:BEFORE]` / `[GENERATE:AFTER]` | 注入提示词开头（仅蓝灯）/ 末尾（蓝灯+绿灯） |
| `[RENDER:BEFORE]` / `[RENDER:AFTER]` | 注入楼层渲染内容开头/末尾 |
| `[GENERATE:{idx}:BEFORE/AFTER]` | 注入第 idx 条 messages |
| `[GENERATE:REGEX:模式]` | 消息匹配正则时注入 |
| `[InitialVariables]` | 条目内容视为变量树写入初始消息变量 |

`@@` 装饰器（条目内容开头独占行）：`@@activate`（视为蓝灯）/ `@@dont_activate` / `@@generate_before/after` / `@@render_before/after` / `@@message_formatting`（RENDER 输出为 HTML）/ `@@initial_variables` / `@@private`（自动包 `<% { %>...<% } %>` 防变量重声明——@@private 派的原理）/ `@@if 表达式`（单行 JS，不过则排除条目——NSFW 指南条件激活的实现）/ `@@iframe [标题]`（RENDER 内容包 iframe 防样式污染）/ `@@dont_preload` / `@@preprocessing`。

`@INJECT` 精细注入（条目设为**未激活**，标题填注入语句，内容为要发送的文本）：
`@INJECT pos=1,role=system`（绝对位置）/ `@INJECT pos=-1,role=user` / `@INJECT target=user,index=1,at=after,role=system` / `@INJECT regex="^问题",role=system`。

## 6. EJS 内置函数（完整参考）

变量（作用域映射：global→extension_settings、local→chat_metadata（聊天）、message→楼层变量、cache→临时、initial→初始）：

```js
setvar(key, value, options?)   getvar(key, options?)
incvar(key, value=1, options?) decvar(key, value=1, options?)   // 可带 min/max
delvar(key, index?, options?)  insvar(key, value, index?, options?)
patchVariables(key, jsonPatch, options?)   // RFC 6902 patch 变量
jsonPatch(dest, change)                    // patch 任意对象
parseJSON(text)                            // 宽松 JSON 解析
// options: { scope: 'global'|'local'|'message'|'cache'|'initial',
//            index, flags: 'nx'|'xx'|'n'|'nxs'|'xxs', results: 'old'|'new'|'fullcache',
//            merge /*用 _.merge 而非覆盖*/, dryRun, noCache, min, max, withMsg }
```

世界书：

```js
await getwi(lorebook, title, data?)              // 直接加载条目内容（绕过激活）
await activewi(lorebook, title, force=false)     // 触发原生激活，需在 [GENERATE:BEFORE] 内
await activateWorldInfoByKeywords(keywords, condition?)
await getEnabledWorldInfoEntries(chara, global, persona, charaExtra, onlyExisting)
await getWorldInfoData(name)   await getWorldInfoActivatedData(name, keyword, condition?)
selectActivatedEntries(entries, keywords, condition?)
```

角色卡/预设/快速回复：`await getchar(name?)`、`await getCharData(name?)`、`await getpreset(name)`、`await getqr(name, label)`。
聊天楼层：`getChatMessage(idx, role?)`、`getChatMessages(count | count,role | start,end | start,end,role)`、`matchChatMessages(pattern)`。
提示词注入：`injectPrompt(key, prompt, order=100, sticky=0)`、`getPromptsInjected(key)`、`hasPromptsInjected(key)`。
正则：`activateRegex(pattern, replace, opts?)`（opts: minDepth/maxDepth/user/assistant/worldinfo/message/generate/basic/order/html/sticky…）。
模板：`await evalTemplate(content, data?)`、`print(...)`（仅 `<% %>` 内）、`define(name, value)`（跨条目定义全局函数）、`await execute(cmd)`（跑 STScript）、`getSyntaxErrorInfo(code)`。

## 7. EJS 内置常量

| 常量 | 含义 |
|---|---|
| `variables` | 合并变量表（message 末→起 → local → global）——读取变量条目的数据源 |
| `SillyTavern` | getContext() 返回值 |
| `_` / `$` / `toastr` / `faker` | lodash / jQuery / toastr / faker 假数据 |
| `runType` | `'generate' \| 'preparation' \| 'render' \| 'render_permanent'` |
| `userName` / `charName` / `chatId` / `characterId` | 身份 |
| `lastUserMessageId` / `lastCharMessageId` / `lastMessageId` | 楼层 id |
| `lastUserMessage` / `lastCharMessage` | 最后消息内容 |
| `model` / `generateType` | 当前模型 / 生成类型（normal/swipe/regenerate/quiet…） |
| `charLoreBook` / `userLoreBook` / `chatLoreBook` | 绑定的世界书名 |

仅 render 时：`message_id`、`swipe_id`、`is_last`、`is_user`。仅 `[GENERATE:*]` 内：`world_info`（当前条目）、`generateBuffer`、`generateData`。

## 8. 防泄漏与调试

- 楼层正文出现字面 `<% %>` 会被渲染时执行——用 `<#escape-ejs>...<#/escape-ejs>` 范围转义，或正则预处理。
- 调试：STScript `/ejs code` 直接跑一段、`/ejs-refresh` 重读全部世界书；`window.EjsTemplate.evaltemplate/getSyntaxErrorInfo` 可在控制台排错。
- EJS 报语法错先查三件套（§2），再用 `getSyntaxErrorInfo` 定位行。
