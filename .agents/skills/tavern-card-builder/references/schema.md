# zod Schema 规范

stat_data 的唯一事实源。MVU 每轮 apply patch 后用它 parse：clamp 数值、enum 回退、strip 未定义字段、transform 自动计算。**schema 怎么写，决定了"AI 写错时数据是被修正还是被丢弃"。**

## 1. 两条路线（先选）

| | 官方模板流 | 实战流（多个成熟卡验证） |
|---|---|---|
| 文件 | `schema.ts`（zod 4，`export const Schema` 纯导出） | `schema.js` 或 `schema.ts`（zod 3 风格组合子） |
| 校验 initvar | `pnpm build/watch` 生成 schema.json，initvar.yaml 头部 `# yaml-language-server: $schema=路径` | 无（靠人工对齐） |
| 默认值 | `.prefault()` 优先 | `.default(d).catch(d)` 双兜底 |
| 前端取数 | `defineMvuDataStore(Schema, {type:'message', ...})`（pinia 双向同步） | `Mvu.getMvuData` / `getVariables` 手动读 |
| 注册 | 同下 | 同下 |

两条路线注册方式相同——卡内一支"变量结构"脚本：

```js
import { registerMvuSchema } from 'https://testingcf.jsdelivr.net/gh/StageDog/tavern_resource/dist/util/mvu_zod.js';
// ...Schema 定义...
$(() => { registerMvuSchema(Schema); });
```

另需一支加载器脚本启用 MVU 本体：`import 'https://testingcf.jsdelivr.net/gh/MagicalAstrogy/MagVarUpdate/artifact/bundle.js';`（惯例做"国内/国外"两支二选一启用）。

官方硬规则（两条路线都适用）：`z`/`_` 是全局不要 import；**永远不用 `.passthrough`/`.strict`**；**幂等**——`Schema.parse(Schema.parse(x))` 必须等于 `Schema.parse(x)`（schema 用于增量解析，每轮都会重 parse）；schema 只定义在 `export const Schema` 内。

## 2. 工具函数组合子（标准件，直接抄）

```js
const clamp = (v, lo, hi) => Math.min(Math.max(v, lo), hi);
const num   = z.coerce.number().transform(v => Math.max(0, v)).default(0).catch(0);   // >=0
const num1  = z.coerce.number().transform(v => Math.max(1, v)).default(1).catch(1);   // >=1
const pct   = z.coerce.number().transform(v => clamp(v, 0, 100)).default(0).catch(0); // 0~100
const attr  = z.coerce.number().transform(v => clamp(v, -3, 3)).default(0).catch(0);  // 性格六维
const favor = z.coerce.number().transform(v => clamp(v, -1000, 1000)).default(0).catch(0); // 好感/忠诚
const str   = (d = '')    => z.string().default(d).catch(d);
const bool  = (d = false) => z.boolean().default(d).catch(d);
const arr    = z.array(z.any()).default([]).catch([]);
const tagArr = z.array(z.string()).default([]).catch([]);
const uidArr = z.array(z.string()).default([]).catch([]);
```

要点：数字一律 `z.coerce.number()`（AI 常写字符串数字）；布尔**直接 `z.boolean()` 不 coerce**（coerce 会把 "false" 转 true）；限制值用 `transform` clamp 而非 `.min/.max`——**破坏 schema 的更新应转化生效而不是整体丢弃**（.min 会让整条 parse fail → 数据回默认）。只在用户要求时加限制，不要擅自 optional/擅自收紧。

## 3. 封闭性：每个对象设计时必答的决策题

**写每一个 z.object 之前先回答："这里允许 AI 随意写入新 key 吗，还是路径固定死？"**——这是 schema 设计最容易遗漏的决策，漏答的后果是两个方向的事故：
- 该开没开（无 catchall）→ AI 写的扩展字段被**静默 strip**——"AI 明明输出了，存进去没有"的头号根因；
- 不该开乱开（catchall(z.any()) 逃生舱）→ 垃圾键污染数据、前端按固定字段渲染读不到、schema 失去校验意义。实战教训：某卡重构时一次性删掉 15 处滥开的 catchall 逃生舱。

**决策流程**（对每个对象过一遍）：

1. 这个位置的 key 集合**能不能枚举全**？能（基本信息/数值面板/固定槽位）→ **封闭**，AI 自创键活该被 strip。
2. key 是动态实体标识（UID/物品名/平台名）、value 结构固定？→ **字典容器**：`catchall(具名Schema)` 或严格流 `z.record(z.string(), 具名Schema)`。
3. AI 确实需要自由发挥的描述区（外貌细节/设施特性/自定义属性）？→ **受限开放**：`catchall(extProp)` 只放基础类型、**禁嵌套对象**（嵌套 = 结构失控的起点）。
4. 纯统计/计数字典？→ `catchall(num)` / `catchall(str)` 类型化。
5. 都不确定？→ **默认封闭**。被剧情证明需要开放再开（开比收容易——收紧要迁移存档，见 §6）。

**开了就要配套两件事（缺一就是白开/坑）**：
- 规则条目里写明这个开口**允许的字段清单**（镜像原则，见下）；
- 前端对这个开口用 `Object.keys` 动态渲染或"白名单+兜底展示"（固定字段渲染 = AI 写进去了 UI 也看不见，等于没开）。

三档代码形态：

```js
// 1. 实体/子对象主体：完全闭合（无 catchall）——字段固定，AI 自创键被 strip
// 2. 创意开放区（设施/区域等）：catchall(extProp) 限定基础类型，禁嵌套对象
const extProp = z.union([z.coerce.number(), z.string(), z.boolean(), z.array(z.any())]);
// 3. 字典容器：catchall(具名Schema) 强类型 value
角色档案库: z.object({}).catchall(CharacterSchema).default({}).catch({})
// 4. 类型化字典：catchall(num) / catchall(str)（统计、审计类）
```

更严格的做法（迁移后 0 catchall）：字典容器改 `z.record(z.string(), X).prefault(() => ({}))`。

**镜像原则**：schema 端每个 catchall/record 开口，规则条目里必须写明允许的字段清单（"装备槽位锚定——仅 6 固定槽，catchall 会静默收下乱键但前端按固定字段渲染读不到"）。封闭处规则里写"禁自造键"。

**三档标注法**（写进更新规则 schema 树的注释）：`❌ 封闭`（未定义被 strip）/ `✅ 半开放`（可加 key 但类型受限）/ `🔥 严格封闭`（AI 最常写错的位置——典型是"必须经过中间分组层"的结构：正确 `/档案/身体部位/腰部`，AI 常漏层写成 `/档案/腰部` 被 strip）。

## 4. 结构选型

- **对象优先于数组**：`物品栏: z.record(z.string().describe('物品名'), z.object({...}))` 而非 z.array——JSONPatch 按 key 定位比按 index 稳。
- 键型选择：固定必选同型键 `z.record(z.enum([...]), 值)`；固定可选 `z.partialRecord`；动态键 `z.record(z.string(), 值)`；固定异型 `z.object`；动态+部分必选 `z.intersection(z.object({...}), z.record(...))`。
- 可被 `{op:"remove"}` 清空再重建的对象：每字段 `.prefault()` + 整体 `.prefault({})`，**不要 `.optional()`**。
- 需要插入序/排序：加 `$time: z.coerce.number().prefault(() => Date.now())`，读取端 `_(data).entries()`。
- `_` 开头字段 = 只读约定（脚本写、AI 禁改），更新规则里不列。
- `z.describe` 只用于字段名不自解释处（如 record 的 key 含义）。

## 5. transform 的两类用法

**A. 自动计算字段**（阶段/状态映射）——必须与规则条目"禁止手写"镜像闭环：

```js
const CharacterSchema = z.object({...}).transform(c => {
    c.好感阶段 = calculateAffectionStage(c.好感度);   // 阈值分档函数
    c.体力.状态 = calculateStaminaStatus(c.体力.当前值, c.体力.上限);
    return c;
}).default({}).catch({});
```

三处呼应：schema transform 算 → 更新规则「自动计算字段（禁止手写）」章 → 建档清单再提醒。数值主、状态辅（"数值状态同步协议"：`生命值>=70% → 生命状态="轻伤"` 阈值表，禁止单独改状态不改数值）。

**B. 宽容输入拍平**（治 schema 静默吞数据）：

```js
// ❌ 绝对禁止：union + partial 接嵌套——partial 先剥掉所有未知 key 再套默认，数据全丢
// ✅ z.unknown() 接任意输入，transform 里手工拍平，再 parse 扁平 schema
const PersonalityTagsSchema = z.unknown().transform(val => {
    // 字符串"A/B/C"→拆分；数组→按位；嵌套对象→{...val, ...(val.底色||{}), ...(val.关系||{})}
    return PersonaCoreSchema.parse(flattened);
}).default(DEFAULT).catch(DEFAULT);
// union 收多形态也可以：EquipSchema = z.union([z.object({...}), z.string(), z.null()]).transform(归一为对象)
```

顶层 transform 还可做引用清洁：剔除 已认识角色/在附近角色 里指向不存在档案的 UID。

## 6. 迁移套路（实录：宽松 schema → 严格 schema）

| 改什么 | 旧 | 新 |
|---|---|---|
| 字典容器 | `z.object({...}).catchall(X).default({}).catch({})` | `z.record(z.string(), X).prefault(() => ({}))` |
| 对象默认 | `.default({}).catch({})` | `.prefault(() => ({}))` |
| 数组 | `.default([]).catch([])` | `.prefault(() => [])` ★工厂函数防数组被冻结后 in-place 修改炸 |
| 标量 | `.default(d).catch(d)` | **不动** |

迁移后遗症管理：旧存档已被剥成默认值的救不回来，只能等 AI 重写；收紧后 AI 高频错配要固化成「schema 数据契约速查」条目（❌/✅ 对照 + 高频错配五类：同义词字段名/自创字段/类型替代/结构照抄/手写自动算字段；通则"不确定就写 `{}` 或 `[]` 让 prefault 兜底，比错填导致整段 fail 安全"）。第三方写入口（第二 API 等）迁移前用自实现 fallback，迁移完切官方 `Mvu.parseMessage` 管线（USE_MVU_PARSE 接缝模式，见 second-api.md）。

## 7. 部署与生效

- **文件头版本块（必写，累积式）**：每次 schema 变更在头注释追加一行"本版改了什么"——这是 schema 的 changelog，排查"哪版引入的问题"全靠它：

```js
// ═══ XX卡 Schema v1.2 ═══
// v1.2: 物品加 耐久 字段；关系阶段 enum 加"挚友"档
// v1.1: 修复 装备槽 catchall 滥开 → 收紧为 6 固定槽
// v1.0: 初版
```

- schema 改动生效条件：更新卡内脚本内容 + **重载脚本/重开对话**。世界书重导入不会更新 schema。
- schema 文件 ↔ 卡内脚本同步用 byte-safe 工具（按 SCRIPT_ID 定位 pull/push，只动那一支脚本的 content，其余字节不动）。
- 官方模板流：改 schema.ts 时保证 `pnpm watch` 在跑（initvar.yaml 校验依赖生成的 schema.json）。

## 8. 反面教材：绕过初始化协议（真实 bug 诊断链）

嵌入脚本用 `Mvu.replaceMvuData` 直写 0 楼"手动初始化" → 绕过 `loadInitVarData()` → `mag_variable_initialized` 事件不发 → mvu_zod 监听不到 → **schema 默认值从未注入** → 后续 `safeParse` 失败 → mvu_zod 不消费命令 → 静默落回原生 commander。表象只是"额外模型解析没启用"，根因藏了五层。

教训：**初始化永远走标准协议**——[InitVar] 条目（enabled=false）由 MVU 自己扫描加载；绝不手写"帮 MVU 初始化"的补丁。修复三处联动：脚本只留 bundle import + [InitVar] 走标准 + 开场白清除 `<UpdateVariable>` 残块。
