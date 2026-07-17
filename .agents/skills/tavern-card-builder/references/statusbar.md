# 状态栏 / 游玩界面前端（总册）

前端界面 = 无沙盒 iframe 前台显示在消息楼层（文件夹含 index.ts + index.html 即被 webpack 识别为界面；仅 index.ts 则是脚本）。

本册讲架构与原则。动手写组件时**必读姊妹册**：
- `statusbar-components.md` —— 16 节组件手册：Tab/卡片/进度条/chip/Modal 三流派/折叠/设置页/诊断页/工具函数标准件/渲染惯用法/从零最短路径，全部带可抄代码骨架
- `statusbar-pitfalls.md` —— 27 条坑实录（布局弹窗/React/数据渲染/事件刷新/持久化），写完对照扫一遍

## 1. 架构三档（按规模选）

| 档 | 结构 | 范本 |
|---|---|---|
| 单体 | `init()` 一个大函数 + utils 服务层 | 小规模卡够用 |
| 模块化 TS | `index.ts`(总render+事件) + `state.ts`(共享单例) + `render/*`(每 Tab 一模块) + `actions/*`(交互) + `utils/*` | 中型卡推荐 |
| React | App.tsx + components/(业务全 Modal) + utils/ + `paths.ts` 集中路径表 + types/constants | 大型卡/重交互 |

React 档入口惯例：jQuery ready 包 `ReactDOM.createRoot`。技术栈可用（浏览器环境，禁 nodejs 库）：jquery/lodash/vue/react/pinia/pixi/gsap/toastr/zod/klona 等。

## 2. iframe 适配硬规则（错版主因）

- **禁 `vh`** 等受宿主高度影响单位——用 `width` + `aspect-ratio` 控制尺寸。
- 避免 `min-height`、`overflow: auto` 把父容器撑高；主体不 `position: absolute` 脱离文档流；不得出现横向滚动条。
- index.html 只写静态 body + 空 head（webpack 注入）；**禁** `<link rel="stylesheet">`、`<script src>`、空 `<img src="">`。样式走 `import './index.scss'` 或 vue 组件内。
- 生命周期：加载 `$(() => {})`（**禁 DOMContentLoaded**——远程 load 不触发）；卸载 `$(window).on('pagehide')`（**禁 unload**）；顶层用 `errorCatched(init)()` 包。

## 3. 数据读取（防"读不到变量"）

入口两件套（MVU 卡必写）：

```ts
await waitGlobalInitialized('Mvu');
await waitUntil(() => _.has(getVariables({ type: 'message' }), 'stat_data'));
```

读取降级链（五级，健壮性天花板）：`Mvu.getMvuData({type:'message', message_id: 最新assistant楼})` → 该楼 data 字段 → `Mvu.getMvuData('latest')` → `getVariables({type:'message', message_id:'latest'})` → 空骨架兜底。简版三级：message latest → chat → getVariables()。
负数 message_id 是倒数索引；界面所在楼用 `getCurrentMessageId()`。

刷新策略两派：事件驱动（`eventOn(tavern_events.MESSAGE_UPDATED)` 300ms 防抖 + `MESSAGE_RECEIVED` 1s 兜底 + `isRefreshing` 防重入）vs 一次渲染（打开时读一次，第二 API 任务完成后局部重渲）。有变量频繁变动就选事件驱动。

## 4. 防御模式（LLM 数据不可信，UI 是最后一道）

- **路径集中**：所有变量路径进 `VAR_PATHS` 常量表，组件用 `_.get(state, path, default)`，绝不散写字符串——schema 改字段时只改一处。
- **取值函数**：`g(path, default)` 空值**和空字符串**都回退 default（空串忘了处理是实际踩过的坑）。
- **safeDisplay(value)**：任意值→人类可读串，递归 maxDepth=3 防爆栈，替代裸 JSON.stringify（防玩家看到 raw JSON）。
- **tagTrim 截断**：schema 语义是短标签的字段（X氛围/X状态/X类型），渲染前截到 ~10 字符 + `title=` 悬停全文——LLM 会把整段塞进 tag 字段，规则管不住，UI 必须防。
- **buildUidMap**：汇集全部档案生成 `uid → 名字(uid)`，`resolveUIDsInText` 把自由文本里的 `npc_001` 翻译成名字；新旧路径双挂向下兼容。
- **isPlayerNpc 过滤**：LLM 会把 persona 误建成 NPC 档案，渲染前滤掉。
- **messageParser 剥离**（若界面要读正文）：三类思维链（`<thinking>`、名字含 plan/reasoning/cot 的标签、**孤立 `</thinking>` 假闭合——删到最后一个闭合为止**）+ 结构子标签（`<UpdateVariable>`/`<选择>`）。
- 动态字段渲染用 `Object.keys` 遍历而非硬编码字段数组（schema 半开放区会长新字段）。

## 5. React 三坑（跨项目实录）

1. **内联组件 remount**：组件定义写在另一组件函数体内 → 父组件每次 render 生成新函数引用 → React 视为不同类型 → unmount+mount → 滚动回顶/input 失焦/state 重置。修法：提升到模块顶层，依赖走 props。
2. **CJK Flex 竖排**：`space-between` 行里长 nowrap badge 把中文标题挤到一字宽逐字竖排。修法双层：标题 `flex:'1 1 auto', minWidth:0` + badge 容器 `maxWidth:'50%', overflow:'hidden'`；内容侧 tagTrim。
3. LLM 整段塞 tag 字段——见上 tagTrim，结构防御 + 内容防御都要。

## 6. 占位符投放与动作投递

占位符两种：世界书常驻条目逼 AI 每楼输出 `<StatusPlaceHolderImpl/>`（条目内容就一句"一定要有"）→ 配正则把它替换成 iframe；或只写在开场白 alternate_greetings（仅首楼有）。前者每楼都有状态栏，后者省心。
**占位符→iframe 的正则三支配合（远程/本地二选一 + 对 AI 隐藏）与"正则状态栏"轻量形态**：详见 `regex.md` §2.C——那册还讲了 markdownOnly/promptOnly 三态机制，是理解整个转换链路的前提。

前端向游戏投递动作三哲学（按侵入度选）：
1. **复制提示词**：`navigator.clipboard.writeText(动作提示词)` 玩家手动发——零侵入，适合重要决策。
2. **自动发用户消息**：`createChatMessages([{role:'user', message}])` + `triggerSlash('/trigger')`——前端**不写变量**，建档交给 AI 走 MVU 管线；真随机在前端完成（LLM 不会随机）。
3. **追加输入框不发送**：push 到 `#send_textarea` 光标置尾，玩家补话后自己发。
共同铁律：**前端不直接写 stat_data**（绕过 schema 与事件；唯一例外是第二 API 回收管线，见 second-api.md）。

## 7. 构建与部署

- 仓库根 webpack 自动发现入口：`{示例,src}/**/index.{ts,tsx,js,jsx}`；`pnpm build`（或 watch）。
- 产物 `dist/<项目>/<模块>/index.html`——JS/CSS 全内联单文件，即部署物（酒馆助手前端界面直接引这一个 html；脚本则是 index.js）。
- 分发：粘贴内容，或 jsdelivr 直链 `https://testingcf.jsdelivr.net/gh/用户/仓库/dist/.../index.js`（@hash 锁版本）。
- 实时调试：酒馆扩展设置开「酒馆助手-实时监听」→ 代码改动热重载，无需手动 build/刷新（配 chrome-devtools MCP 验证）。
- 新脚本文件无 import/export 会被当全局脚本与 @types 的 ambient 声明冲突（TS2451）→ 末尾加 `export {};`。
