# legado-source-generator

**一句话**：给 AI 一个小说/漫画/有声书网站 URL，自动生成可直接导入 Legado 阅读 App 的书源 JSON 文件。

---

## 有 vs 没有这个 Skill

|  | 没有 Skill（裸模型） | 有 Skill |
|---|---|---|
| 你要做的事 | 手动看网页源码，查 CSS 选择器，逐字段猜规则，对照文档手写 JSON | 说一句"帮我给这个网址做书源"，AI 自动爬页面、识别结构、输出 JSON |
| 耗时 | 约 30–120 分钟（熟手）；新手可能完全无从下手 | 约 2–5 分钟 |
| 出错风险 | 选择器写错导致搜索/目录/正文全部空白，需反复调试 | AI 同时验证搜索页、详情页、目录页、正文页，不会遗漏关键字段 |
| 特殊站点处理 | GBK 编码、SPA 动态渲染、JSON API 等需手动分析 | 自动识别编码、检测 API 接口、生成对应 JS 规则 |

---

## 限制与边界

| 类型 | 说明 |
|------|------|
| ✅ 能处理 | 普通 HTML 小说站（CSS class / XPath / OG 标签） |
| ✅ 能处理 | 纯 JSON API 类网站（如 `$.data[*]` JSONPath 规则） |
| ✅ 能处理 | 单本书专题站（如 SPA 结构，利用 JS 规则解析 data.js） |
| ✅ 能处理 | 需要 exploreUrl 分类浏览的站点（按文件夹/标签生成分类） |
| ❌ 不能处理 | 需要账号登录才能访问内容的网站（无法自动获取 Cookie） |
| ❌ 不能处理 | 强混淆/加密 JS 渲染、内容完全动态生成且无 API 的站点 |
| ⚠️ 已知问题 | data.js hash 参数随网站更新变化，需手动更新 searchUrl 中的 hash 值 |
| ⚠️ 已知问题 | Legado JS 引擎为 Rhino，不支持 ES6+ 箭头函数/async，生成的 JS 规则使用 ES5 语法 |

---

## 使用方法

**前置条件**：
- [ ] 安装了 Legado（开源阅读 App）并能正常导入书源
- [ ] 目标网站可公开访问（无需登录）

**安装**：无需额外安装，Skill 已内置。将 `SKILL.md` 放入 `~/.copilot/skills/legado-source-generator/` 目录即可。

**触发方式**：直接在对话中说出网站 URL + 意图即可触发：
- `"帮我给 https://xxx.com 做书源"`
- `"这个网站能加书源吗？https://xxx.com"`
- `"生成书源：https://xxx.com，分组是小说"`

**典型对话**：

> **你**：https://wt.tepis.me/ 帮我给这个网址做书源，在章节选择下的所有章节是所有书籍内容，需要制作一个搜索来展示所有的书籍。
>
> **AI**：（自动访问网站 → 发现 SPA 结构 → 找到 `data.js` 数据文件 → 识别文件夹即书籍结构 → 生成搜索/详情/目录/正文规则）
> 输出可直接导入的 `wt_tepis_me.json` 文件，搜索关键词过滤书系，目录递归收集章节，正文直接请求静态接口，无需 WebView。

---

## 真实案例

### 案例 1：标准 HTML 小说站（CSS class 结构）

**输入**：`https://www.bxwx.la/ 帮我做书源`

**输出**（核心规则片段）：
```json
{
  "searchUrl": "https://www.bxwx.la/ar.php?keyWord={{key}}",
  "ruleSearch": {
    "bookList": "class.txt-list txt-list-row5@tag.li!0",
    "name": "class.s2@text",
    "author": "class.s4@text",
    "bookUrl": "class.s2@tag.a@href"
  },
  "ruleToc": {
    "chapterList": "class.section-list fix.1@tag.li",
    "chapterUrl": "tag.a@href",
    "nextTocUrl": "class.right@tag.a@href"
  },
  "ruleContent": { "content": "id.content@textNodes" }
}
```
验证环境：Legado 3.x Android，2026-05

---

### 案例 2：SPA 单本书专题站（data.js JS 解析）

**输入**：`https://wt.tepis.me/ 帮我做书源，章节选择下所有章节就是书籍内容`

**输出要点**：
- `searchUrl` → `data.js?q={{key}}`，用 JS `eval()` 解析 `window.DATA` 树
- 每个顶层文件夹（主线/道具集等）作为一本"书"
- `ruleToc.chapterList` 递归收集子文件夹中的章节
- `ruleContent.content` 直接请求 `https://wt.tepis.me/chapters/{path}`（静态接口，无需 WebView）
- `exploreUrl` 将 53 个书系全部列为分类标签（用户追加请求后生成）

验证环境：Legado 3.x Android，2026-05

---

### 案例 3：追加发现分类（exploreUrl）

**输入**：`将每一个章节的名字作为一个 tagname，更新到 exploreUrl 展示出来`（在已有书源基础上追加）

**输出要点**：
- 通过 Playwright 访问网站，从 `data.js` 动态提取全部 53 个文件夹名称
- 为每个文件夹生成一条 exploreUrl 条目：`主线::https://wt.tepis.me/data.js?folder=%E4%B8%BB%E7%BA%BF`
- `ruleExplore` 规则与 `ruleSearch` 共用同一套 JS 逻辑，通过 URL 参数 `folder=` 识别当前分类
- 同步将 `enabledExplore` 改为 `true`

验证环境：Legado 3.x Android，2026-05

---

### 案例 4（边界）：网站需要进一步分析才能确定规则

**输入**：`https://xxx.com 做书源`（fetch_webpage 返回内容不完整）

**AI 行为**：自动切换至 Playwright 浏览器工具渲染页面 → 分析网络请求找真实数据接口 → 若发现 JSON API 则改用 JSONPath 规则，若是纯动态渲染则在章节 URL 追加 `{"webView":true}` 标注。

---

## Changelog

| 版本 | 日期 | 变化 |
|------|------|------|
| v1.0 | 2026-05-19 | 初版：支持 HTML/JSON API/SPA 三类站点，含 exploreUrl 生成指南 |
