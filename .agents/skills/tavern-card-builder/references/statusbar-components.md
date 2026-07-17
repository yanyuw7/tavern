# 状态栏组件手册（照着写）

三种范式的组件级写法。范式 A = 模块化 TS+jQuery（成熟形态，新卡默认）；范式 B = 同套写法单文件版；范式 C = React 重型。文末有从零最短路径。

目录：1 Tab 栏 / 2 卡片列表 / 3 进度条 / 4 徽章 chip / 5 Modal 三流派 / 6 折叠面板 / 7 设置页 / 8 诊断页 / 9 按钮组 / 10 空态加载 / 11 字体主题 / 12 缩放手柄 / 13 高级组件 / 14 工具函数标准件 / 15 渲染惯用法 / 16 最短路径

## 1. Tab 栏

范式 A `actions/tabs.ts`（14 行全文，纯 class 切换）：

```ts
document.querySelectorAll('.tab').forEach(tab => {
  tab.addEventListener('click', function () {
    document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
    document.querySelectorAll('.pnl').forEach(p => p.classList.remove('active'));
    (tab as HTMLElement).classList.add('active');
    document.getElementById('p-' + (tab as HTMLElement).dataset.t)?.classList.add('active');
  });
});
```

HTML 约定 `<button class="tab" data-t="ov">` ↔ `<div class="pnl" id="p-ov">`；CSS `.pnl{display:none} .pnl.active{display:block}`；**窄屏就一招 `overflow-x:auto`**（曾有项目做独立 mobile 底栏，后删——横向滚动足够，别做两套）。React 版 TabBar：icon 上 label 下、label `slice(0,2)` 截两字全名进 title、点击语义 toggle（再点关闭）。

## 2. 卡片 / 列表

**tile 网格 → 点开详情 modal**（两级模式）：

```ts
html += `<div class="mecha-tile" data-uid="${uid}">
  <span class="mecha-tile-rarity">${rarity}</span>
  <span class="mecha-tile-lv">Lv.${lv}</span>
  <div class="mecha-tile-name">${name}</div>
  <div class="mecha-tile-state">${stateTxt}</div>
</div>`;
// 容器: display:grid;grid-template-columns:repeat(auto-fill,minmax(128px,1fr));gap:10px
// 四角信息用 position:absolute 钉在 tile 上
```

变体：HP 分档 class（`hpPct>=60?'':hpPct>=30?'low':'crit'`）、死亡遮罩（`.tile-status-dead{position:absolute;inset:0;background:rgba(0,0,0,.65)}`）。

**头部点击展开卡**（NPC/势力）：`.npc-header onclick="toggleNPC('${id}')"` + `.npc-detail{display:none}` + `.npc-card.expanded .npc-detail{display:block}`；免注册版直接 `onclick="this.parentElement.classList.toggle('open')"`。

**万用信息格**：`<div class="item"><div class="lbl">标签</div><div class="val">值</div></div>` 配 `.grid.g2/.g3/.g4` 网格。

## 3. 进度条

静态坑位 + JS 只改宽：`<div class="bar"><div class="bar-fill bar-hp" id="cmd-hp-bar"></div></div>`，`b(id, cur, max)` 设 `width:${pct}%`；色种走 class（`.bar-hp/.bar-san/.bar-exp/.bar-aff` 渐变）；`.bar-fill{transition:width .5s}`。
动态生成版 `bar(label, cur, max, cls)` 返回 `md-bar-row` 字符串（标签+数值行 / 条体两段）。
颜色三档惯用法：`durPct > 60 ? 'var(--olive)' : durPct > 30 ? 'var(--warn)' : 'var(--danger)'`。
React 侧另有 SVG 环形条 StatRing（双 circle + strokeDasharray/offset + rotate(-90) 顶部起画）。

## 4. 徽章 / chip

- 按值染色徽章：`<div class="relation-badge relation-${relation}">` + CSS 枚举中文 class（`.relation-敌对{...}`）。**保底色必须写在 `.relation-badge` 本体**——AI 写了表外值就没颜色。
- React Chip 三变体核心手法——**hex 色 + alpha 后缀**：`soft: { background: \`${c}22\`, color: c, border: \`1px solid ${c}44\` }`。
- 状态徽章带脉冲 + 点击跳 modal：战斗中=红+pulse，`onClick={() => onBadgeClick?.(ViewState.EXPLORE)}`。

## 5. Modal 三流派

**流派 1：覆盖式（iframe 内最稳）**——不用 fixed，绝对定位铺满 `.panel`，覆盖即遮罩：

```scss
.mm-modal{display:none;position:absolute;inset:0;background:var(--bg);z-index:300;flex-direction:column}
.mm-modal.active{display:flex}
.mm-head{display:flex;justify-content:space-between;padding:12px 16px;flex-shrink:0}
.mm-body{flex:1;overflow-y:auto;padding:16px}
```

开关 = `classList.add/remove('active')`；嵌套 = 开子先关父。

**流派 2：React 遮罩 modal**——单一 `activeView` 状态驱动，`.modal-backdrop` 独立一层承接点击关闭，`.modal-container{max-width:900px;max-height:90vh}`。

**流派 3：Portal 二级弹窗**——modal 里再弹的层（抽屉/确认框）**必须 `createPortal(node, document.body)`**：祖先有 backdrop-filter/transform 时 fixed 退化成相对定位 + overflow:hidden 裁剪 = 弹窗跑正文中间/显示不全。配套：ESC 监听只在 open 时挂、body 滚动锁存 prev 值还原、Portal 节点显式写 color（脱离主题继承）。

## 6. 折叠面板

```ts
`<div class="fold-card ${count > 0 ? 'open' : ''}" id="foldEquip">
  <div class="fold-header" onclick="toggleFold('foldEquip')">
    <div class="fold-title"><span>🛡️</span><span class="fold-name">装备</span>
    ${count > 0 ? `<span class="fold-sub">${count}件</span>` : ''}</div>
    <span class="fold-arrow">▼</span>
  </div>
  <div class="fold-body">...`
// CSS: .fold-body{display:none} .fold-card.open .fold-body{display:block}
//      .fold-card.open .fold-arrow{transform:rotate(180deg)}
// 有内容默认展开 = 模板里 ${count>0?'open':''}
```

**toggle 函数必须挂 window**（inline onclick 找不到 webpack 模块作用域函数，点了没反应且无报错）：`(window as any).toggleFold = toggleFold;`。调试页直接用原生 `<details open><summary>` 更省事。

## 7. 设置页（localStorage）

设置页五要素（完整范本结构）：

```ts
const KEY = 'xx_ui_settings';   // 换成你的卡前缀
function load(): Settings {
  try { const raw = localStorage.getItem(KEY);
    return raw ? { ...DEFAULTS, ...JSON.parse(raw) } : { ...DEFAULTS };  // 展开兜底新增字段
  } catch { return { ...DEFAULTS }; }
}
function apply(): void {                       // 唯一真理：改任何设置都走它
  const p = document.querySelector('.panel');  // 变量设在 .panel 不是 :root——iframe 里只影响自己
  for (const v of THEME_VARS) p.style.setProperty(v, theme.vars[v]);   // 主题全套
  for (const [v, val] of Object.entries(cur.colors)) p.style.setProperty(v, val);  // 自定义覆盖
  p.classList.remove('font-xs','font-sm','font-md','font-lg','font-xl'); p.classList.add(cur.font);
  p.style.setProperty('--panel-w', cur.width + '%');
  p.classList.toggle('collapsed', cur.collapsed);
}
// input/change 事件里连调 apply(); save(); ——实时生效实时持久化
```

细节：THEMES 数组每套 10 个 CSS 变量（语义色不随主题变）；主题选择器带色板预览（4 个变量渲成小方块）；切主题清自定义色。localStorage 读回**永远不信任**：逐字段类型校验+钳制（`Math.min(3, Math.max(0.5, scale))`），非法整体回 fallback。版本迁移：`migrateLegacySettings(parsed)`。

## 8. 诊断页

MVU 观察器 modal：**捕获在常驻脚本（iframe 抓不到额外解析阶段），modal 只读全局变量渲染**。打开时 `setInterval(1500ms)` 轮询、关了自动 `clearInterval`；左列表右详情 + `<details>` 四段（发送提示词/AI 原始返回/解析命令/落库 diff）；**AI 原文必过 esc() 转义**；复制按钮 try/catch（iframe 剪贴板权限可能被拒）。
React 版：订阅式日志源 + 导出双保险——clipboard 失败自动退 Blob 下载（`URL.createObjectURL` + a.click + revoke）。

## 9. 按钮组

- 单选组：全清 `.active` 再点亮 + 切换时清关联状态（`pickedSpec.clear()`）。
- 多选网格：`Set` 存选中，toggle 后整格重渲，配"全选/清空"。
- **主按钮禁用+文案合一**：`btn.disabled = !!reason; btn.textContent = reason || '💎 启动制造 ×N（消耗 X EP）'`——不可用原因直接写按钮上。
- 点击外部关闭下拉：`document.addEventListener('click', e => !e.target.closest('.dropdown') && close())`。

## 10. 空态 / 加载

空态：渲染函数空数据返回 `''`，调用侧 `el.innerHTML = render(...) || '<div class="empty">空空如也</div>'`。
启动 gating（防白屏渲染空数据，三家一致）：

```ts
await waitGlobalInitialized('Mvu');   // try/catch 包住不强依赖
await waitUntil(async () => {
  const sd = await readGameData();
  return Object.keys(sd).length > 0;
}, { timeout: 10000, intervalBetweenAttempts: 100 });
```

生成中四态：`'idle'|'sending'|'generating'|'processing'` + spinner + 流式预览区，busy 时选项隐藏。

## 11. 字体缩放 / 主题

字体缩放通用公式：一个 `--font-scale` + 所有字号 `calc(Npx * var(--font-scale))`。预设 class（`.panel.font-xs{--font-scale:.8}`）或连续滑块（`p.style.setProperty('--font-scale', v)`，先解除预设 class 防叠加）或全局 rem（`html{font-size:calc(16px * var(--font-scale,1))}`）。
主题两派：JS 对象数组逐变量 setProperty vs CSS class 制（`theme-xx` 内做"品牌变量→语义变量"映射、组件只消费语义层）。改版防破图：留旧变量别名映射层（`--txt: var(--bone);`）。

## 12. 缩放手柄 ResizeHandle

核心：mousedown 后 listener 挂 **window**（拖出手柄不断）；`onResize`（拖动中实时改 state）与 `onCommit`（mouseup 才写 localStorage）分离；`document.body.style.cursor/userSelect` 设置与还原；尺寸钳制 `Math.max(400, Math.min(3000, v))`。要点如上，完整实现约 114 行。

## 13. 高级组件

- **两栏详情 scroll-spy**（左目录右内容）：`secs: {id,icon,label,html}[]` + `add()` 空段守卫；点击 `content.scrollTo({top: tgt.offsetTop - 4, behavior:'smooth'})`；滚动高亮用 rAF 节流（`ticking` 标志）；窄屏 media query 左目录变顶部横条。
- **详情段落工具族**：`row(k,v)` → `dl(rows)`（auto-fill minmax(208px,1fr) 网格）→ `field(label,val)` 长文本 → `subtitle(t)` → `statCells(obj, fields)`（过滤 `undefined/''/'无'`）。
- **九宫格地图**：视野中心存 state，双层循环生成 3×3，方向键只改 center 局部重渲；`mapViewInitialized` 保证只首渲按当前位置定位（用户调过的视角，数据刷新只读不写）。
- **SVG 雷达图**：`angles = data.map((_,i) => -Math.PI/2 + i*2*Math.PI/n)`，背景三层缩放多边形，数据面 `fill=${color}33`；中文评级映射数字表（`{极弱:1,...,超凡:14}`）。
- **在场人数小红点**：主 render 时按"位置字符串前缀"汇总 `state.baseOccupancy[loc].push(name)`，渲染设施时查表出 `<div class="room-dot">${n}</div>`——位置命名 `基地/楼层/设施[/子区域]` 强制格式是前提（更新规则里锁）。
- **调料快捷 chips**：预设短语按钮点击追加 textarea（`ta.value = ta.value ? ta.value + '；' + v : v`）。
- **世界名归一 + datalist**：自由输入吸附已有值（精确→忽略大小写→原样），防库碎片化。
- **两步报价确认**（two_step.ts，121 行通用件）：配置驱动（id 字符串 + buildQuoteMsg/buildConfirmMsg）；待确认文本存 localStorage 跨刷新；**提示词里显式声明阶段边界**（"这是报价阶段，不要扣 EP，等确认"）防 AI 抢跑。

## 14. 工具函数标准件

范式 A `utils/helpers.ts`（7 件，直接抄）：

| 函数 | 作用 | 要点 |
|---|---|---|
| `g(o, path, d)` | 路径取值 | `path.replace(/\[(\d+)\]/g,'.$1').split('.')` 逐级下钻，中途 null 回 d |
| `v(x)` | 解析 JSON 字符串值 | `{...}` 形字符串 try parse，失败原样 |
| `vn(x, d)` | 取值带回退 | `v(x) != null ? : d` |
| `t(id, x)` | 写 textContent | `String(x ?? '')` |
| `h(id, html)` | 写 innerHTML | |
| `b(id, cur, max)` | 写进度条宽 | |
| `fm(o)` | 滤 `_` 前缀字段 | 隐藏 `_检测结果` 等内部字段 |

范式 B 同名不同义（按自家脏数据形态选）：`v(x)` 解 MVU 数组 `[值,"描述"]` 取 `x[0]` + `{数值:...}` 递归剥壳；`vn` 把 `'待定'/'待生成'/'--'/'未命名'/''` 中文占位符也视为空；`str(x)` 任意值转串（数组 join('、')、对象 stringify）。

范式 C React 防崩件：`safeNode(v)`（对象→`v.名称||v.名字||v.描述||stringify.slice(0,80)`，防 "Objects are not valid as a React child"）；`asArr/asObj/asString`（**治 lodash get 的 null 陷阱**——默认值只在 undefined 触发，LLM 写 null 照样崩）；`dictToStr`（catchall 动态键渲染，兼容整个 dict 被写成 JSON 字符串）；`safeDisplay`（递归 maxDepth=3）；`buildUidMap`/`resolveUIDsInText`/`displayName`；`esc()`（`& < >` 三换，AI 原文插 innerHTML 前必过）。

领域件：好感分档 `getAff`（≥80 挚友 / ≥50 亲近 / ≥20 熟识 / ≥0 陌生 / ≥-20 冷淡 / 仇敌）；双格式兼容 `getHP`（先探嵌套 `HP.当前` 再退平铺 `HP`）；`sampleDistinct`（Fisher-Yates 抽样）；`RANK_ORDER` 排序表。

## 15. 渲染惯用法（jQuery 派）

- 顶层 `render(d)` 按 Tab 顺序调模块；模块签名 `renderXxx(sd): void`（收整棵 stat_data，自己取子树自己找容器）；返回 string 的是小写片段工具。
- 全量 innerHTML 重建为主（简单可靠），**纯 UI 操作走局部**（地图方向键/fold 开合）；重渲会丢的 UI 状态提升进 `state.ts` 单例（`export const state = { cachedData, mapViewCenter, expandedFloors, ... }`），模板里读 state 还原。
- 事件绑定四手法：静态骨架 init 时绑一次 / 动态列表**事件委托**（绑容器 `closest('.tile')`）/ modal 内控件 innerHTML 覆盖后重绑（旧节点连 listener 一起销毁不会叠）/ inline onclick 挂 window。
- modal 打开读 `state.cachedData` 而非重新异步读取。
- 数字防抖动 `font-variant-numeric:tabular-nums`；`select option` 每主题显式指定背景色（否则系统白底）。

## 16. 从零最短路径（范式 A）

1. `index.html`：`.panel > .header + .tabs(.tab[data-t]) + .content(.pnl#p-xx…) + 若干 .mm-modal`，动态区只留 `<div id="xxx-list">` 坑位。
2. `index.scss`：`:root` 变量块 + tabs/pnl/item/grid/bar/fold-card/mm-modal 全套 + 字体 calc 块 + 滚动条。
3. `utils/helpers.ts`（7 件原样）+ `utils/variableReader.ts`（五级 fallback 原样）+ `state.ts`。
4. `actions/`：tabs / folding（window 挂载）/ settings（改 KEY 和 THEMES）/ two_step（原样）。
5. `render/` 每 Tab 一模块：tile 网格 + fold-card + item 格三板斧；详情 modal 用 secs+add+scroll-spy 套路。
6. `index.ts`：render(d) 调度 → refreshGameData（isRefreshing 锁）→ 事件（MESSAGE_UPDATED 300ms 防抖 + MESSAGE_RECEIVED 1s 兜底）→ init 一次 → `$(() => init())` + waitUntil gating。
7. 对照 `statusbar-pitfalls.md` 27 条逐一自查。
