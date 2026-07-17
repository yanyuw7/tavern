# 酒馆正则系统（Tavern Regex）

卡的"显示层管道"：AI 输出的结构化标签（变量块/思维链/选择栏/状态栏占位符）全靠正则转换成玩家看到的样子，同时控制 AI 下一轮能看到什么。一张成熟 MVU 卡标配 10~12 支。

## 1. 字段结构与三态机制（先懂这个再写）

正则脚本存在卡 JSON 的 `extensions.regex_scripts` 数组，每支：

```json
{
  "scriptName": "隐藏获取变量",
  "findRegex": "/<status_current_variables>[\\s\\S]*?<\\/status_current_variables>/gm",
  "replaceString": "",
  "placement": [2],
  "markdownOnly": true,
  "promptOnly": true,
  "disabled": false,
  "runOnEdit": true
}
```

**三态机制（最核心概念）**——`markdownOnly` 与 `promptOnly` 的组合决定"谁看不到"：

| markdownOnly | promptOnly | 效果 | 典型用途 |
|---|---|---|---|
| true | false | 只影响**显示**——玩家看不到，AI 下轮照样看到，存档原文保留 | 美化类（原文还要给 AI 参考时） |
| false | true | 只影响 **prompt**——AI 下轮看不到，玩家照样看到 | 防复读（AI 别看到自己上轮的标签）、防 token 浪费 |
| true | true | 玩家和 AI 都看不到，但存档原文保留（**非破坏**） | 隐藏类首选（变量注入块/选择栏兜底） |
| false | false | **直接改写存档消息文本（破坏性）** | 慎用——除非真要永久改写（如清洗旧格式） |

`placement`：1=用户输入、2=AI 输出、3=slash 命令、5=世界书。绝大多数用 `[2]`；要连用户消息里的标签一起处理用 `[1,2]`。
`runOnEdit`：玩家编辑消息后是否重跑（美化/隐藏类一般 true）。
**执行顺序 = 数组顺序**：美化在前、消除兜底在后（"选择框消除（放美化后面）"——美化没吃掉的残留由兜底剥掉）。

## 2. 标准件清单（一套 12 支的分工）

### A. 隐藏类（replaceString 为空串）

| 支 | findRegex 要点 | 三态 |
|---|---|---|
| 隐藏获取变量 | `/<status_current_variables>[\s\S]*?<\/status_current_variables>/gm` | markdown+prompt 双 true（玩家 AI 都不看，注入块只在本轮有用） |
| 去除变量更新（prompt 侧） | 见 §3 未闭合兼容写法 | **仅 promptOnly**——显示侧留给美化正则接管 |
| 对 AI 隐藏状态栏占位符 | `<StatusPlaceHolderImpl/>` 字面量 | 仅 promptOnly——AI 永远不知道占位符被替换成了什么 |
| 隐藏思维链 | `/<Analysis>[\s\S]+?<\/Analysis>/gm` | 按需：只 markdownOnly=玩家不看但 AI 记得；双 true=都不看 |

### B. 美化类（replaceString 是 HTML 模板，捕获组填值）

**thinking 折叠**：`/<thinking>([\s\S]*?)<\/thinking>/gi` → `<details style="..."><summary>💭 思考过程</summary>$1</details>`——原文塞进可折叠块。

**流式双态美化**（一对配合，处理"AI 正在输出变量块"的过程感）：

```
支 1「变量同步中」——匹配"有开标签、尚无闭标签"（流式进行中）：
  /<UpdateVariable>(?![\s\S]*<\/UpdateVariable>)\s*([\s\S]*)$/gi
  → 替换成"⏳ 同步中"动画卡片
支 2「变量同步完成」——匹配完整闭合块：
  /<UpdateVariable>\s*([\s\S]*?)\s*<\/UpdateVariable>/gi
  → 替换成"✅ 已同步"折叠卡片（内容进 details）
```

流式渲染时每个 token 都重跑正则：未闭合阶段命中支 1 显示动画，闭合瞬间支 2 接管——**negative lookahead 是流式美化的关键技术**。

**选择框美化**：把 plot 层约定的 `<选择>\n1|类型|文本\n2|...` 逐行捕获（每行两个捕获组），替换成完整 HTML 按钮组（含 `<style>`）。行数固定的卡直接写死 N 行捕获；行数可变要写多支或用宽松匹配。配一支"选择框消除"兜底（`/<选择>[\s\S]*?<\/选择>/gs`，双 true）放在美化**后面**。

### C. 状态栏转换（占位符 → 界面，两种形态）

**形态 1：iframe 加载型**（配酒馆助手前端，功能强可交互）

三支配合：

```
支 1「状态栏（远程）」 enabled：
  find: <StatusPlaceHolderImpl/>   （字面量）
  replace（markdownOnly, placement[2]）:
  ```
  <body>
  <script>
  $('body').load('https://你的用户名.github.io/仓库/dist/你的卡/状态栏/index.html');
  </script>
  </body>
  ```
支 2「状态栏（本地测试）」 disabled 备用：
  同上，URL 换 http://localhost:5500/dist/.../index.html —— 调试时手动二选一切换
支 3「对 AI 隐藏占位符」 promptOnly：见 A 类
```

机制：replaceString 里的 **``` 代码块包 `<body><script>`** 会被酒馆助手渲染成楼层内 iframe，script 里 jQuery `.load()` 拉远程构建产物。配套世界书常驻条目逼 AI 每楼输出 `<StatusPlaceHolderImpl/>`（内容就一句"输出末尾必须带 `<StatusPlaceHolderImpl/>`"）。占位符标签名可自定，全卡一致即可。

**形态 2：内联 HTML 型（正则状态栏，轻量无依赖）**

replaceString 直接内联**完整 HTML+CSS**（可几十 KB）；数据来源两种：
- 捕获 AI 输出的结构化状态块（`<状态>HP:80|MP:50|...</状态>` 逐字段捕获组填进模板）
- 或模板里用酒馆宏 `{{getvar::xx}}` / `{{get_message_variable::stat_data.路径}}` 直读变量

对比：

| | iframe 型 | 内联 HTML 型 |
|---|---|---|
| 依赖 | 酒馆助手 + 托管的 dist | 无（随卡走） |
| 交互 | 完整 JS（Tab/modal/发消息） | 无 JS，最多 CSS 交互（details/hover） |
| 更新 | 改 dist 重新部署，卡不动 | 改卡里的 replaceString |
| 适合 | 重度玩法卡 | 轻量卡/分发简单优先 |

### D. 第二 API 中间标签双重隐藏

同一个 findRegex 配**两支**：`显示隐藏`（markdownOnly, placement[2]，玩家不看中间标签）+ `对AI隐藏`（promptOnly, placement[1,2]，主 AI 下轮不看到自己上轮写的标签——**防复读**）。见 second-api.md §3。

## 3. findRegex 写法要点与坑

- **跨行必须 `[\s\S]` 或 `s` flag**——`.` 默认不匹配换行，`<tag>.*</tag>` 只能吃单行。
- **懒惰匹配 `*?`**：同标签一楼多次出现时，贪婪 `[\s\S]*` 会从第一个开标签吃到最后一个闭标签。
- **未闭合兼容**（AI 输出被截断/漏闭合时也要剥干净）：
  `/<update(?:variable)?>(?:(?!.*<\/update(?:variable)?>).*$|.*<\/update(?:variable)?>)/gsi`
  ——"要么后面根本没有闭标签（吃到结尾），要么吃到闭标签为止"。变量块类的 prompt 隐藏必须这么写，否则截断楼会把半个 JSON 喂给下轮 AI。
- **拼写变体宽容**：`<update(?:variable)?>`、`/gi` 忽略大小写——笨模型会拼错标签。
- **反向引用保证开闭一致**：动态标签名用 `/<敌方(fac_\d+)>([\s\S]*?)<\/敌方\1>/g`。
- **中文标签名完全可用**（`<选择>`），但避免 `<a>`/`<b>` 这类会被 markdown/HTML 吞的通用名。
- **replaceString 里的 CSS 要自带作用域**：内联 style 或类名加卡前缀（`.xxcard-choice-btn`），裸 `button{...}` 会污染整个楼层。
- **$1..$n 引用捕获组**；模板里的字面 `$` 注意转义。
- 正则存进 JSON 时 `\` 双重转义（`[\\s\\S]`）——用脚本生成时留意。

## 4. 交付与部署

**默认交付可直接导入的 JSON 文件**（每支一个 .json，用户在 扩展→正则→导入 一键进；不要只给"内容+手动配置说明"让用户在 UI 里逐字段填——手动填 placement/三态勾选框极易错）。完整字段集（**11 个字段一个不少**，缺字段可能导入异常）：

```json
{
  "id": "生成一个 UUID（每支唯一）",
  "scriptName": "隐藏获取变量",
  "findRegex": "/<status_current_variables>[\\s\\S]*?<\\/status_current_variables>/gm",
  "replaceString": "",
  "trimStrings": [],
  "placement": [2],
  "disabled": false,
  "markdownOnly": true,
  "promptOnly": true,
  "runOnEdit": true,
  "substituteRegex": 0
}
```

注意：`findRegex` 是 `/pattern/flags` 字符串形式，JSON 里 `\` 要双写（`[\\s\\S]`）、`</` 写 `<\\/`；`id` 每支唯一 UUID；`substituteRegex` 数字（0=关）。多支可以合并成一个数组文件，但逐支独立文件更便于用户选择性导入。

**命名必须带序号（防导入顺序错误）**：正则执行顺序 = 它在列表里的物理顺序，用户逐支导入的先后就是执行顺序——文件名和 `scriptName` 都加两位序号前缀，序号即导入顺序：

```
01-隐藏获取变量.json      → scriptName: "01-隐藏获取变量"
02-去除变量更新.json      → scriptName: "02-去除变量更新"
05-选择框美化.json        → scriptName: "05-选择框美化"
06-选择框消除.json        → scriptName: "06-选择框消除（必须在美化后）"
```

顺序敏感对（美化→消除兜底、同态双支）中间留号，方便日后插新支不用全体改号；顺序敏感的支在名字里再加一句人话提醒（"必须在美化后"）。用户导入完在 UI 列表里按编号一眼核对顺序对不对。

- 随卡走的形态：写进卡 JSON `extensions.regex_scripts` 数组（组装模式，见 worldbook.md §7.2），生成时按 id/scriptName 幂等去重。
- 验证顺序：开一楼让 AI 输出全部标签 → 逐支检查显示效果 → 看下一轮 prompt（酒馆的 prompt 查看器）确认 promptOnly 生效 → 编辑消息确认 runOnEdit。
- 常见事故：忘了"消除兜底"导致美化漏网的原始标签直接裸露；promptOnly 忘开导致 AI 复读自己的标签；两支状态栏（本地/远程）同时 enabled 双重加载。
