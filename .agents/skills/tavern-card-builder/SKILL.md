---
name: tavern-card-builder
description: SillyTavern 酒馆助手角色卡的全套制作规范——从多个成熟实战项目提炼的通用方法论。凡是涉及"做一张新卡（MVU 变量卡或纯文字卡）/ 把现有纯文字卡改造成 MVU 变量卡 / 给卡加系统 / 写 zod schema / 写变量更新规则 / 组织世界书条目 / 写卡的 description 或开场白 / 写状态栏前端 / 写配套脚本 / 接第二 API / 写酒馆正则 / 修别人的卡 / 排查变量没生效"的任务，动手前必读本 skill。哪怕用户只说"加个数值""做个状态栏""写个世界书""帮我改造这张卡""帮我看看为什么变量丢了"，只要上下文是 SillyTavern/酒馆助手/MVU/角色卡，就先用本 skill 找到对应 reference 再动手。
---

# 酒馆助手 MVU 角色卡制作规范

**Version:** 2.2.2

通用造卡方法论。SKILL.md 是总控：生态地图、工作流、全局铁律、分册导航。动手写某一层之前，**先读对应 reference 分册**。

## 工作区协作边界

本 skill 是"制卡规划与部件设计"入口，不替代项目工程流。新卡设计、纯文字卡改造、变量系统规划、世界书/状态栏/正则/脚本方案，优先用本 skill 组织需求、拆部件、查对应分册。

在 `C:\Users\Administrator\OneDrive\ST-\角色卡工作区` 内落到既有同源项目源码、组件库、验证、组装、打包、PNG 嵌入、发布时，继续叠加并服从 `card-workflow`、`card-components`、`rolecard-release-workflow`。遇到精确 API、Tavern Helper / MVU / EJS / STScript 签名，以 `ST开发指南DB` 与 `sillytavern-helper-dev` 的权威引用为准；本 skill 负责把它们接成可执行制卡流程。

## 0. 生态四层与"命令方言"识别（最先搞清楚）

```
SillyTavern（宿主）
 └─ 酒馆助手 Tavern Helper（脚本/前端 iframe 运行时，window.TavernHelper）
     ├─ ST-Prompt-Template（世界书/预设里的 EJS，window.EjsTemplate）
     └─ MVU / MagVarUpdate（消息楼层变量框架，window.Mvu）
          └─ mvu_zod（registerMvuSchema，用 zod 校验 stat_data）
```

**MVU 变量更新有三种方言，绝不可混用**。接手任何卡先识别它用哪种（看 `[mvu_update]变量更新规则` 或 `变量输出格式` 条目）：

| 方言 | 输出形态 | 说明 |
|---|---|---|
| MVU 原生命令 | `_.set('a.b', 旧值, 新值)` 点路径 | 官方默认 |
| 官方 JSONPatch 变体 | `<UpdateVariable><JSONPatch>`，op = `replace/delta/insert/remove/move` | 官方模板的固定条目「变量输出格式」 |
| RFC 6902 标准 | 同容器，op = `add/replace/remove`，数组尾 `/-` | 实战项目常用；本 skill 的规则/清理/自检文案按它写 |

给现有卡写规则时沿用它的方言；造新卡推荐 RFC 标准。

## 1. 造卡标准工作流（10 步，三入口）

先认入口，再按顺序做（每步先读对应分册）：

- **入口 A · 新 MVU 卡**：完整走 1→10。
- **入口 B · 改造现有纯文字卡为 MVU 卡**：第 1 步换成"读卡分析"（读 `references/retrofit.md`——变量化信号扫描 + 原味保持铁律 + 最小可跑四件套），第 2~10 步照走。
- **入口 C · 纯文字卡（不要变量系统，允许正则与文本状态栏）**：读 `references/text-card.md`——状态块协议（文本自延续链替代变量）、长期记忆四层、状态栏按卡气质设计、与 MVU 卡**相反**的正则纪律、裁剪版工作流。

1. **需求访谈（硬门槛——没全部问清之前，禁止动手写任何文件）**

   造卡是大工程，自作主张设计变量体系 = 大概率返工。访谈按"大项 → 小项 → 确认"三轮走，每轮问题少而准（一次 2~4 个问题，别把问卷 dump 给用户）：

   **第一轮·大项清单**：这张卡有哪些系统？逐项过——题材世界观 / 角色档案 / 物品背包 / 任务日程 / 组织势力 / 战斗 / 经济货币 / 时间天气 / 色色系统(SFW/NSFW) / 状态栏 / 第二 API / 其他玩法系统。让用户勾选+补充，而不是自己拍脑袋。
   **第二轮·逐个大项问小项**：每个勾选的大项里包含什么？例：角色档案要哪些块（基本信息/性格/外貌/关系/能力/记录）？数值有哪些、范围多少、默认多少？物品要不要耐久/堆叠？任务要不要优先级/过期？状态栏要哪些 Tab？有没有想参考/照抄的现成卡？

   **第二轮还有一组"输出格式与内容"必问项**（不管勾了哪些大项都要问，这些最容易被替作者做主）：
   - **选择栏**：要不要每轮末尾出选项？要的话——几个选项、类型标签有哪些（推进/互动/日常/NSFW…）、**文案规范选哪派**（强制完整主谓宾且主语必须 {{user}} 的"可直发派"，还是允许短语/开放式的"氛围派"——两派设计见 plot-style.md §4）、要不要正则美化成按钮。
   - **摘要/总结机制**：要不要？（MVU 卡 = 备忘录历史摘要+个人总结；纯文字卡 = 状态块摘要/总结区）保留多少条、多久归档合并。
   - **世界书自定义内容**：作者有没有**现成的设定文本**想收进世界书（世界观/NPC/地点/组织/事件/私设规则）？有的话逐份问：哪些常驻、哪些 key 触发、哪些做成默认禁用的可选 DLC。
   **第三轮·复述确认**：把理解到的完整结构复述成变量树草图 + 条目规划 + 组件清单，让用户确认或纠正。**用户点头后才进入第 2 步。**

   访谈中用户没提到但按经验必要的（UID 计数器/清理规则/读取变量条目这类基建），不用问，直接按本 skill 规范带上；但凡涉及**玩法取舍**的（数值范围、建档门槛宽严、NSFW 开不开、方言选哪种），必须问，不要替用户决定。

2. **设计 stat_data 结构 → 写 schema**——先画变量树再写 zod。读 `references/schema.md`。
3. **写 [InitVar]**——空结构初始变量（record 全 `{}`，让 LLM 剧情中逐步填充；只有真正的全局默认值写实值）。**条目永远 enabled=false**，MVU 主动扫描它。
4. **写 [mvu_update]变量更新规则**——主条目常驻，详则拆独立条目按需触发。读 `references/update-rules.md`。
5. **写开局协议与卡本体**——叙事开局（[mvu_plot]，key=开始游戏）与变量开局（建档模板）分开写；卡片字段与开场白按 `references/card-writing.md`（系统卡流派：经典五字段留空、内容条目化、开场白=README 页+起手页）。
6. **写"读取变量"条目**——EJS 过滤 stat_data 发给 LLM。**必须无前缀**（额外解析模式下带 [mvu_plot] 前缀 = 变量 AI 看不到当前变量 → 幽灵实体增殖，有真实事故实录）。读 `references/ejs.md`。
7. **写 plot 层**——演绎指南 / 文风指引 / 选择栏。读 `references/plot-style.md`。
8. **状态栏前端**（可选）。读 `references/statusbar.md`。
9. **配套脚本**（可选）——数值检测 / 自动清理 / 观察器 / 骰子守卫 / persona 优化。读 `references/scripts.md`。第二 API 另读 `references/second-api.md`。
10. **部件交付（默认）与验证**——**默认不生成完整角色卡 JSON**，逐部件交付；**部件一律给可直接导入的 JSON**（手动逐字段填 UI 极易复制错误）：
    - 世界书 → **可导入的独立世界书 JSON**（过 worldbook.md §4 硬规则）；条目极少或用户要逐条审查时才给 txt+配置表
    - 脚本 → **每支一个可导入 .json**（完整字段集 scripts.md §1.1）："酒馆助手→脚本库→导入"
    - 正则 → **每支一个可导入 .json**（11 字段模板 regex.md §4）："扩展→正则→导入"
    - 状态栏 → 构建产物 + 部署说明
    **所有部件文件名加两位序号前缀**（`01-xxx.json`…）——正则执行顺序=列表顺序，乱序导入=功能错乱；其余部件编号防漏项。每个部件末尾附"导入到哪里、改完要重载什么"。**仅当用户明确说"组装角色卡/打包成卡"才生成完整卡 JSON**（读 worldbook.md §7）。交付前把 `references/pitfalls.md` 对应节扫一遍（状态栏另扫 statusbar-pitfalls.md）。

## 2. 全局铁律（跨层最高优先级）

0. **先问清再动工**：新卡/新系统必须先过需求访谈（大项→小项→复述确认三轮），用户确认结构后才写文件。改现有卡时若需求含糊（"加个数值"没说范围/联动/谁用），同样先问。宁可多一轮确认，不做要返工的大方案。
1. **schema 是唯一事实源**。改任何字段走同步链：schema → InitVar → 更新规则的 schema 树 → 读取变量 EJS → 状态栏 → 脚本 → plot 指南。每一站问"这字段谁写、谁读、谁展示、谁清理？"
2. **前缀即路由**：额外模型解析模式下 `[mvu_plot]` 只发剧情 AI、`[mvu_update]` 只发变量 AI、无前缀双发（"随 AI 输出"模式下前缀仅是标注——但造卡一律按路由纪律写，两种模式都安全）。"让 AI 看当前变量"的条目必须无前缀；变量规则绝不挂 [mvu_plot]。改任何条目前自问："这条变量 AI 需不需要看见？"详见 worldbook.md §1。
3. **[InitVar] 永远禁用**（enabled=false）。MVU 按条目名扫描读取，启用反而把 YAML 塞进提示词。
4. **变量层与剧情层不可混**：plot 条目里出现字段名/路径/JSONPatch = 立刻撤走归 update 层；update 层不写演绎细节。
5. **无 catchall 的对象会静默 strip 未定义字段**——"AI 写了但存进去没有"十有八九是这个。反过来：任何 catchall 开口都要在规则里写明允许的字段清单。
6. **绝不用 union+partial 接嵌套输入**（partial 先剥未知 key 再套默认 = 静默吞数据）。宽容输入用 `z.unknown().transform` 手工拍平。
7. **动态 key 一律 `前缀_NNN` UID**，配 `/UID计数器/${前缀}` 闭环（建档前扫重名→命中 replace 不 add→新建则计数器+1）。禁名字做 key；**key 禁含英文句点**（JSONPatch 路径转内部点路径会被劈断）。
8. **变量只存状态不存事件**（反流水账）："这条明天还成立吗？"成立=状态可存；过时=事件，归历史摘要或正文。
9. **部署三套互不联动**：世界书 JSON 重导入 / schema 脚本重载+重开对话 / dist 前端重 build 重绑。改任一套都要告诉用户重导哪个——"写完代码就报完成"是事故源。
10. **世界书 JSON 的 entries 键必须 == uid**（字符串化），否则导入后列表空白。字段集对齐同项目已验证可导入的文件，别跨版本套模板。
11. **改卡必须复验**：批量动作后跑脚本断言（entry 计数/keyless 扫描/dup 扫/关键字段抽样），不靠肉眼。byte-safe 编辑：改 JSON 内容用 `raw.split(JSON.stringify(old)).join(JSON.stringify(new))`，锚点命中数必须=1，写前 .bak。
12. **排查"没生效"看真证据**：(a) AI 真实输出 → (b) 变量真实存储 → (c) 比对差异。顺序不可跳。可装一个"变量观察器"脚本做证据链（见 scripts.md）。
13. **文档给 LLM 看的都剥 markdown 装饰**（`#`/`**`/`---`/`>` 全去，yaml 缩进天然有结构）；禁完整 JSONPatch 模板（LLM 会复读硬编码值）；禁可套用例句（会逐字照抄）。
14. **token 经济**：常驻条目只留每轮必需，详则拆条目按 key/@@if/事件触发；"LLM 是不是每轮都需要看它？"不是就拆。
15. **默认部件交付，不主动组卡，部件给可导入 JSON**：产出物 = 世界书 / 脚本 / 正则各自**可直接导入的 .json**（+ 状态栏产物）和"导入到哪里"说明——不要给"内容+手动配置说明"让用户逐字段填 UI（复制错误高发）。完整角色卡 JSON 只在用户明确说"组装/打包成卡"时才生成——组卡格式敏感又容易覆盖用户已有内容，主动组卡是事故源。
16. **产物同样版本化**：交付的每个部件都带版本号，任何修改必 bump——卡整体用 `character_version`；schema 用文件头注释版本块（累积记录每版改了什么）；世界书条目版本写 **content 首行**（⚠️ 绝不写进条目名——comment 是前缀路由与程序化读取的 key，必须稳定）；脚本/状态栏写代码头注释。迭代交付时说明里必列"部件 vX→vY 改了什么"——版本号是用户核对"哪些部件要重新导入"的唯一凭据，没有它改一百次也无从对账。

## 3. 分册导航

| 分册 | 什么时候读 |
|---|---|
| `references/retrofit.md` | 把现有纯文字卡改造成 MVU 卡（读卡分析/原味保持/最小改动清单） |
| `references/text-card.md` | 做纯文字卡——状态块协议/长期记忆四层/状态栏按卡设计/反向正则纪律 |
| `references/card-writing.md` | 写卡片字段（description 等五字段）与开场白编排（README页/起手页/向导式开局/占位符纪律） |
| `references/schema.md` | 写/改 zod schema、字段被吞、数据类型设计、schema 迁移 |
| `references/update-rules.md` | 写/改 [mvu_update] 变量更新规则、UID/建档/联动/清理/自检 |
| `references/worldbook.md` | 组织条目、触发方式、生成/修世界书 JSON、外书处理、组卡部署 |
| `references/api.md` | 写任何脚本/前端代码前查接口——变量七作用域/楼层/生成/世界书/注入/正则/事件全签名 |
| `references/ejs.md` | 写读取变量/守卫类 EJS 条目、EJS 报错——含内置函数/常量/装饰器/注入前缀完整参考 |
| `references/statusbar.md` | 写状态栏/游玩界面前端（架构与原则总册） |
| `references/statusbar-components.md` | 动手写状态栏组件时——16 节组件骨架手册 + 工具函数标准件 + 从零最短路径 |
| `references/statusbar-pitfalls.md` | 状态栏写完自查 / 排查前端 bug——27 条坑实录 |
| `references/regex.md` | 写酒馆正则——隐藏/美化/流式双态/状态栏占位符转换/防复读，三态机制与 12 支标准件 |
| `references/scripts.md` | 写后台脚本（检测/清理/观察/骰子/persona） |
| `references/second-api.md` | 接第二 API、多 AI 任务、论坛/敌方 AI/离线 NPC |
| `references/plot-style.md` | 写演绎指南/文风指引/选择栏等 plot 条目 |
| `references/pitfalls.md` | **交付前必扫**——全域 43 条易错点速查（EJS 重复声明/schema 吞数据/条目路由/脚本生命周期/变量方言/部署联动） |

## 4. 参考资源

- 若当前工作区内已有成熟的 MVU 卡项目（含 schema/世界书/状态栏源码），**优先参考其现成实现与既有约定**——本 skill 的规范可能与该项目的家规有出入时，尊重项目家规。
- MVU 框架源码与文档：GitHub `MagicalAstrogy/MagVarUpdate`；mvu_zod：GitHub `StageDog/tavern_resource`。
- 酒馆助手模板项目（含 `@types` 接口定义、示例角色卡、webpack 构建）：搜索 tavern_helper_template；`@types/iframe/exported.mvu.d.ts` 是 MVU 接口的权威签名。
- MVU 事件 5 个精确值：`VARIABLE_INITIALIZED='mag_variable_initiailized'`（官方源码就这么拼错，照抄）、`VARIABLE_UPDATE_STARTED/ENDED`、`COMMAND_PARSED`、`BEFORE_MESSAGE_UPDATE`；listener 第一参数是 variables（以 d.ts 为准，部分文档示例写错）。
- 脚本可用全局（无需 import）：`$`、`_`、`z`、`toastr`、`SillyTavern`、`TavernHelper`、`Mvu`（先 `await waitGlobalInitialized('Mvu')`）、`eventOn` 族、`waitUntil`。禁 nodejs 库。

## 5. 维护规范（修改本 skill 前必读，对 Claude 同样生效）

**任何内容变更——改 SKILL.md 或任意 references/*.md、新增/删除分册——必须同时做两件事，缺一不算完成修改**：

1. **bump SKILL.md 正文顶部的 `Version` 标记**（semver）：内容修订/补充 = patch（2.0.0→2.0.1）；新分册/新入口/新机制 = minor（2.0→2.1）；结构性重构/推翻旧规范 = major。Codex skill frontmatter 只保留 `name` 与 `description`，不要把版本号写回 YAML frontmatter。
2. **在 `CHANGELOG.md` 追加一行**：`版本 | 日期 | 一句话改了什么`。

改完自检："version 涨了吗？CHANGELOG 写了吗？"任何一个没做，等于修改没交付——版本号不动的文档改一百次也无从对账。
