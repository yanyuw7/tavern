# 全域易错点速查（写完自检用）

横向清单：每条一两行，症状→修法，详细展开在括号指向的分册。**交付任何一层之前，扫对应节。** 状态栏前端的 27 条另见 statusbar-pitfalls.md。

## A. EJS 条目（重复声明是重灾区）

1. **const/let 重复声明炸整条 prompt**：多个 EJS entry 拼进同一 AsyncFunction 作用域，第二个声明同名变量直接 `SyntaxError: Identifier already declared`——整条 prompt 编译失败，所有条目全哑。修：无 @@private 的卡顶层一律 `var`；或 entry 首行加 `@@private`（自动包 `<% { %>...<% } %>` 独立作用域）。**按卡现有派系走，别混**。（ejs.md §1）
2. **重复声明守卫被当冗余删掉**：`if (typeof stat === 'undefined') var stat = getvar('stat_data');` 和"使用 var 避免重复声明"的注释是**关键设计**——整理代码时删了它，多条目同发时立刻炸。
3. **反引号模板串里写 `${}`**：EJS 先吃一遍插值，变量未定义直接抛 `xxx is not defined`。占位符用 `[UID]` 方括号。（ejs.md §2）
4. **JS 字符串里写 `\n` 字面量**：EJS 编译阶段把 `\n` 当真换行，字符串语法裂开。用模板自然换行 + `<% for %>` 输出，或 `String.fromCharCode(10)`。
5. **变量访问器跨卡照搬**：`getvar('stat_data')` 派与 `variables.stat_data` 派不通用——改哪张卡用哪张卡现有写法。
6. **摘要对象塞回变量 JSON**：把 `_总览:true` 之类的压缩摘要放进发给 AI 的变量结构里，AI 会当真实路径去 patch。摘要必须放 JSON **外**的文本区并附"严禁对此 JSONPatch"警告。（ejs.md §3）
7. **楼层正文出现字面 `<% %>` 被渲染执行**：用 `<#escape-ejs>` 范围转义。

## B. zod schema

8. **union+partial 接嵌套 = 静默吞数据**：partial 先剥未知 key 再套默认，AI 明明输出了、存进去全是默认值。宽容输入一律 `z.unknown().transform` 手工拍平。（schema.md §5B）
8a. **设计对象时没决策封闭性**：每个 z.object 必答"这里允许 AI 随意写 key 吗？"——漏答默认封闭时 AI 扩展字段被静默 strip（"写了存不进"头号根因）；乱开 catchall(z.any()) 则垃圾键污染+前端读不到。开口必配两件套：规则写允许字段清单 + 前端动态渲染。（schema.md §3 决策流程）
9. **用 `.min/.max` 做限制**：越界让整条 parse fail → 数据整体回默认。用 `transform(clamp)`——破坏性的更新应转化生效而不是丢弃。
10. **`z.coerce.boolean()`**：把字符串 `"false"` 转成 true。布尔直接 `z.boolean()`。
11. **transform 不幂等**：schema 每轮重 parse，transform 里做累加/追加类操作会让值每轮漂移。`Schema.parse(Schema.parse(x))` 必须等于 `Schema.parse(x)`。
12. **`.default([])` 共享冻结数组**：被 in-place 修改时炸。容器默认用 `prefault(() => [])` 工厂函数。（schema.md §6）
13. **可被 remove 的对象用 `.optional()`**：remove 后无法重建默认结构。用每字段 `.prefault()` + 整体 `.prefault({})`。
14. **用 `.passthrough`/`.strict`**：官方明令禁止——前者放进任意垃圾，后者让整条 fail。
15. **schema 改完没重载**：更新卡内脚本 + 重载/重开对话才生效；世界书重导入**不会**更新 schema。
16. **手写"帮 MVU 初始化"补丁**：`replaceMvuData` 直写 0 楼绕过 `loadInitVarData()` → 初始化事件不发 → schema 默认值从未注入 → 五层连锁静默劣化。初始化只走 [InitVar] 标准协议。（schema.md §8）

## C. 世界书 / 条目

17. **entries 键 ≠ uid**：导入后条目列表完全空白（分页有数量、列表不显示）。构造必须 `{str(e["uid"]): e}`。（worldbook.md §4）
18. **[InitVar] 被启用**：MVU 按条目名扫描读取，启用只会把整坨 YAML 塞进提示词。永远 enabled=false。
19. **读取变量条目挂了 [mvu_plot] 前缀**：额外解析模式下变量 AI 看不到当前变量 → 扫不到已有实体 → 幽灵档案无限增殖（前缀路由体系最贵的一课）。"让 AI 看变量"的条目必须**无前缀**；修复只改 comment 删前缀，content 不动。（worldbook.md §1.2）
20. **plot 条目混入字段名/路径/JSONPatch**：变量 AI 收不到（发错了对象），剧情 AI 看不懂（浪费 token）。全部撤到 [mvu_update]。兜底写入规则（"未标金额按 X 扣"）也属变量层。
20a. **前缀拼写/位置自创变体**：`[MVU_update]`/`[mvu-plot]`/前缀不在 comment 开头——都不会被按预期路由。精确小写 `[mvu_update]`/`[mvu_plot]`、放最开头。（worldbook.md §1.3）
20b. **只在"随 AI 输出"模式下测试**：该模式前缀不路由、一切正常；用户切"额外模型解析"就炸。造卡按额外解析的路由纪律写前缀，两种模式都安全。（worldbook.md §1.1）
21. **短 key 误触发**：纯数字/≤2 字符拉丁 key（"22"/"2B"）命中属性栏数字，一次误注入几十条。外书配"姓名主键+世界名次键 AND_ANY"门控。
22. **字段集跨版本套模板**：不同 ST 版本的 entry 字段集有差异，生成世界书以**同项目已验证可导入的文件**为模板。
23. **独立世界书与卡内嵌双写**：必然分叉。定一个为主，另一个只在发版时同步。
24. **字典 key 含英文句点**：`7.62×39弹` 落库时被点路径劈成 `"7"` 残桩。key 用无点形，真名放 `名称` 字段。（update-rules.md §4）
24a. **条目不配递归双禁**：常驻条目（尤其读取变量——内容里全是角色名）不禁 prevent_outgoing → 每轮连锁拉起一串 keyed 条目，token 静默膨胀且极难察觉。默认全部双禁，lore 链有意设计才白名单开放。（worldbook.md §2.1）

## D. 后台脚本

25. **顶层裸 `await`**：webpack 打包报错。包进 `$(async () => { ... })`。（scripts.md §1）
26. **新文件无 import/export**：被 TS 当全局脚本，与 @types 的 ambient 声明冲突（TS2451）。末尾加 `export {};`。
27. **用 `DOMContentLoaded`/`'unload'`**：远程 `$('body').load()` 场景下前者不触发；后者不可靠。只用 `$(() => {})` 与 `'pagehide'`。
28. **事件捕获放楼层 iframe**：iframe 寿命绑楼层，额外解析阶段的 MVU 事件结构性抓不到。捕获放常驻脚本 → 全局变量中继。（scripts.md §3）
29. **自行 `generate()` 后变量没更新**：不产生新楼层 → MVU 不自动解析。必须 `Mvu.parseMessage(content, old)` + `Mvu.replaceMvuData(new)` 手动落库。
30. **`CHAT_COMPLETION_PROMPT_READY` 不判 dryRun**：世界书激活计算等干跑阶段也会 fire，误触发实际逻辑。回调第一件事 `if (dryRun) return;`。
31. **往 stat_data 写辅助字段**：主 schema 无 catchall 直接被 strip；有 catchall 也污染数据。给 LLM 的提醒用 `setExtensionPrompt` 注入。
32. **依赖 `eventOn` 重复注册叠加**：同一 listener 重复注册会被**忽略**（不叠加）——需要多次处理就写在一个 listener 里。
33. **iframe 里对异步注入全局做存在性体检**：`typeof Mvu` 探测误报率高。要么 `waitGlobalInitialized` 等待，要么删掉体检功能。

## E. MVU / 变量操作

34. **三方言混用**：原生 `_.set` 点路径 / 官方 `replace/delta/insert/remove/move` / RFC `add/replace/remove`——接手卡先认方言，规则、示例、清理文案全套跟着方言走。（SKILL.md §0）
35. **名字做动态 key**：`/角色档案/张三` → 重名重建、无法改名。一律 `前缀_NNN` UID + 计数器闭环。
36. **整对象 replace**：`replace /基本信息 {姓名, 年龄}` 丢掉未列出的所有兄弟字段。改单字段只 patch 单字段。
37. **`_.get(o, p, default)` 的 null 陷阱**：默认值只在 undefined 触发，LLM 写 null 照样透传崩掉数组方法。拉数据后过 asArr/asObj/asString。
38. **AI 改 `_` 前缀只读字段**：规则里声明 `_` 开头字段禁改、不列更新规则；脚本侧才有写权。
39. **变量作用域选错**：客观世界=message 层 stat_data；应用本地=chat 变量；玩家跨卡偏好=global；凭据/UI 位置=localStorage（永不入变量树）。（second-api.md §5）

## F. 部署 / 交付

40. **三套不联动**：世界书 JSON、schema 脚本、dist 前端各自独立生效——改了源文件没粘回卡 / 没重载 schema / 没重 build，都是"改了没生效"。交付时**列全用户操作清单**，"写完代码就报完成"是事故源。
41. **批量改不复验**：改完必须跑断言脚本（计数/keyless/dup/抽样），"改完了"≠"对了"。用户追问"你确定"时，跑脚本再答。
42. **直接字符串替换大 JSON 不做防护**：byte-safe 三件套——锚点命中数必须=1、写前 `.bak`、写后 JSON.parse 自检 + 字节增量核对。临时脚本用完即删。
43. **给 LLM 的文档带 markdown 装饰 / 完整 JSONPatch 模板 / 可套用例句**：装饰浪费 token；模板会被复读硬编码值；例句被逐字照抄。（SKILL.md 铁律 13）
44. **部件交付不给可导入 JSON / 文件名无编号**：让用户在 UI 逐字段手填 = 复制错误高发；正则执行顺序=列表顺序，文件名无编号 → 用户乱序导入 → 消除兜底跑到美化前面（功能静默错乱）。部件一律可导入 .json + 两位序号前缀。（regex.md §4 / worldbook.md §7.1）
