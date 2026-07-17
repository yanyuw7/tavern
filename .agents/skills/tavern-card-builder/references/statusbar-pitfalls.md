# 状态栏坑大全（27 条实录）

每条：症状 → 根因 → 修法。来源：多个成熟状态栏项目的源码注释、版本考古与实战复盘。写完组件对照扫一遍。

## A. 布局与弹窗

1. **二级弹窗错位/显示不全**：祖先有 `backdrop-filter/transform/filter` → `position:fixed` 退化为相对该祖先定位，叠加 `overflow:hidden` 被裁剪。修：modal 里再弹的层一律 `createPortal(node, document.body)`。
2. **Portal 后文字看不清**：Portal 到 body 脱离了状态栏主题继承，ST body 默认色（常白）与深/浅主题冲突。修：Portal 节点**显式写 color**。
3. **CJK 标题被挤成一字一行竖排**：flex `space-between` 行里长 nowrap badge 把标题挤到 1 字宽。修：标题 `flex:'1 1 auto', minWidth:0, overflowWrap:'break-word'`；badge 容器 `maxWidth:'50%', overflow:'hidden'`；内容侧 tagTrim。
4. **iframe 高度截断 modal**：用 fixed 或依赖视口高。修：覆盖式 modal `position:absolute; inset:0` 相对 `.panel`，内部独立滚动。
5. **禁 vh / min-height 撑高 / absolute 脱流 / 横向滚动条**（官方 iframe 适配四禁，错版主因）。

## B. React 专属

6. **打开 modal 再关，正文滚动回顶**：`{!activeView && <正文/>}` 条件渲染卸载了 DOM。修：always-render + `style={{display:'none'}}` 切换，节点常驻 scrollTop 自然保留。
7. **组件内定义子组件 → remount → 滚动回顶/input 失焦/state 重置**：父 render 每次生成新函数引用，React 视为新类型 unmount+mount。修：提升到模块顶层，依赖走 props。
8. **轮询刷新把抽屉 tab 强制重置**：useEffect 依赖了"每次 render 新建的数组/对象"（如 tabs），轮询型应用 1-2s 一刷 → effect 反复触发。修：`prevOpenRef` 只在 open false→true 时重置；新建对象故意不进依赖。**轮询应用里，依赖每次新建的对象的 useEffect 都是定时炸弹。**
9. **LLM 输出 object 让 React crash**（"Objects are not valid as a React child"）：渲染 LLM 数据前全部过 `safeNode()`。字符串拼接派天然免疫（顶多 [object Object]）。
10. **lodash get 的 null 陷阱**：`_.get(o, p, default)` 默认值只在 undefined 触发，LLM 写 null 照样透传 → 数组方法/Object.entries 崩。修：拉数据后过 `asArr/asObj/asString`。

## C. 数据与渲染

11. **g() 空字符串不回退**：取值函数只判 `undefined/null`，空串 `""` 透传 → 界面空白。修：`v === '' || v == null` 都走 default。
12. **硬编码字段数组渲染半开放区**：schema catchall 长出的新字段永不显示。修：`Object.keys()` 动态遍历。
13. **核心数值数组与 schema 脱同步**：schema 加了字段，前端数组没跟 → 新数值 UI 上不存在。同步链第 5 站，schema 每次加字段都 grep 前端。
14. **schema 大改后的死代码**：旧 render 函数读废弃路径（/职业系统 /财务系统），留着误导维护。修：schema 重构后 grep 前端全部读取路径清一遍（实测一次清掉 ~200 行死代码）。
15. **AI 把装备写进非标准槽位 → 物品隐形**：只按固定槽渲染，AI 误写的键（主武器 vs 武器）不显示。修：**白名单渲染 + 白名单外兜底展示**——先渲固定槽，再把 `Object.keys` 里不在白名单且有值的兜出来。
16. **物品名含英文句点被路径劈断的残桩**：字典里出现纯数字/单字符 key 且无名称。修：显示优先读 `名称` 字段回退 key；检测到残桩渲染 `⚠ 疑似损坏，建议让 AI 重建`——**渲染层要暴露数据层事故，不要默默吞**。
17. **变量值可裸插，模型原文必转义**：MVU 变量信任自家 schema 可直接插值；AI 原始输出（观察器的 prompt/返回/diff、`<sum>` 摘要）插 innerHTML 前必过 `esc()`（`& < >` 三换），否则 AI 文含标签直接炸 DOM。
18. **HTML 坑位无渲染代码**：照抄骨架时 index.html 里的 id 没有对应 render → 永远空白无报错。修：写完对一遍"HTML id ↔ render 代码"清单（实测某卡四个坑位永远空白）。

## D. 事件与刷新

19. **inline onclick 点了没反应且无报错**：webpack/TS 模块作用域函数 inline 字符串找不到。修：`(window as any).toggleFold = toggleFold` 挂载；或改事件委托。
20. **动态列表事件叠加/失效**：每次重渲后散绑。修：**事件委托绑容器一次**（`list.addEventListener('click', e => e.target.closest('.tile'))`）；modal 内控件 innerHTML 覆盖后重绑是安全的（旧节点连 listener 销毁）。
21. **重复刷新竞争**：MESSAGE_UPDATED 与 MESSAGE_RECEIVED 都触发刷新互相踩。修三重防护：`isRefreshing` 并发锁 + UPDATED 300ms 防抖（"等 replaceMvuData 写完"）+ RECEIVED 延迟 1000ms 让位。
22. **patch 收集依赖外层标签 → 换模型丢数据**：解析器把 `allPatches.push` 嵌在"匹配到 `<UpdateVariable>`"里，Claude 只输出 `<Analysis>+<JSONPatch>`（无外层）时 patch 全丢 → 变量永不写入。修：按 `<JSONPatch>` 独立匹配；兜底"有开标签无闭合"（输出截断）；标签缺失时剥已知块用剩余文本兜底——**绝不因格式差异整轮跳过变量更新**。
23. **MVU 存在性体检误报**：iframe + 异步注入下 `typeof Mvu` 探测误报率高，"明明装了却报错"。教训：不可靠的存在性检查直接删功能，靠诊断快照兜底。
24. **用户调过的视图被数据刷新重置**：地图视角/滚动位置每次 render 重算。修：`viewInitialized` 标志只首渲定位；用户手动状态提升进 state，数据驱动渲染只读不写。

## E. 设置与持久化

25. **设置合并的"替换语义"丢新增默认值**：`user.xxx ?? DEFAULT` 用户存过一次后，版本新增的默认项全丢。修：默认 ∪ 用户配置**合并**（`{...DEFAULTS, ...parsed}` 或去重 union），另配 `migrateLegacySettings` 版本迁移。
26. **localStorage 读回不校验**：脏值/旧版本结构直接用 → NaN/崩。修：逐字段类型校验 + 钳制，非法整体回 fallback。
27. **观察/捕获逻辑放楼层 iframe**：iframe 寿命绑楼层，额外模型解析阶段事件 fire 时新楼 iframe 未挂载，**结构性抓不到**。修：捕获放常驻脚本 → 全局变量中继 → iframe 纯读渲染；未装常驻脚本时 UI 给明确提示。

## 附：交互链路两个易错点

- **发消息触发 AI**：`createChatMessages([{role:'user', message}])` 后要 `triggerSlash('/trigger')` 才会生成；前端**绝不直写 stat_data**（真随机/去重在前端做，建档扣费交给 AI 走 MVU 管线）。
- **两步流程防 AI 抢跑**：报价/确认拆两条消息时，提示词里显式声明阶段边界（"这是报价阶段，不要扣 EP，等确认"）。
