---
name: legado-source-generator
description: 自动为 Legado（阅读 App）生成书源 JSON 的专项技能。当用户提到"生成书源"、"制作书源"、"帮我写书源"、"阅读书源"、"legado书源"、"给某网站生成书源"，或提供一个小说/漫画/有声书网站的 URL 并要求生成对应的书源 JSON 时，必须立即调用本技能。即使用户只说"帮我做个书源"或"这个网站能加书源吗"，也应立即使用本技能。
---

# Legado 书源自动生成技能

## 概述

本技能通过分析目标网站的 HTML 结构，自动生成符合 Legado（开源阅读 App）规范的书源 JSON 文件。

---

## 工作流程

### 第一步：收集基本信息

向用户确认或收集以下信息（如果 URL 已提供则直接进入下一步）：

1. **目标网站 URL**（必须）
2. **书源分组**（可选，例如：`小说`、`漫画`、`有声书`）
3. **书源类型**（可选：0=文字, 1=有声，默认 0）

---

### 第二步：分析网站结构

使用 `fetch_webpage` 或浏览器工具依次抓取以下页面并分析 HTML 结构：

#### 2.1 抓取首页
- URL：网站根域名
- 目标：获取网站名称、整体框架结构

#### 2.2 抓取搜索页
- 尝试常见搜索 URL 格式：
  - `{baseUrl}/search?q={{key}}`
  - `{baseUrl}/search?keyword={{key}}`
  - `{baseUrl}/search.php?key={{key}}`
  - `{baseUrl}/ar.php?keyWord={{key}}`（部分站点）
  - `{baseUrl}/search/{{key}}`
- 分析搜索结果列表的 HTML 结构，识别：
  - 书籍列表容器（class/id）
  - 书名、作者、封面、分类、最新章节元素

#### 2.3 抓取书籍详情页
- 从搜索结果中取第一个书籍 URL 访问
- 分析：书名、作者、封面、分类、简介、最新章节、目录链接

#### 2.4 抓取目录页
- 从详情页获取目录 URL 访问
- 分析：章节列表、章节名称、章节链接、下一页链接

#### 2.5 抓取正文页
- 从目录页获取第一章 URL 访问
- 分析：正文内容容器、下一页链接

---

### 第三步：编写规则

根据分析结果，按照以下规则语法编写各部分规则。

#### 规则语法参考

**Default（JSOUP）语法：**
```
class.类名@子标签@属性
id.元素ID@属性
tag.标签名.位置@属性
```
- 常用属性：`text`、`textNodes`、`href`、`src`、`all`、`html`、`ownText`
- 位置：正整数从 0 开始，`-1` 为倒数第一，`!0` 排除第一个

**CSS 语法（必须以 `@css:` 开头）：**
```
@css:.class-name > a@href
@css:#id span.sub@text
```

**XPath 语法（必须以 `@XPath:` 或 `//` 开头）：**
```
//div[@class='content']//p/text()
//*[@property="og:novel:author"]/@content
```

**JSONPath 语法（必须以 `@json:` 或 `$.` 开头）：**
```
$.data.list[*]
@json:$.items.*.name
```

**JavaScript（`<js>...</js>` 或 `@js:` 结尾）：**
```
tag.a@href@js: result + "?page=1"
```

**正则净化（跟在规则后）：**
```
tag.p@text##广告内容##
```

#### 常用模式库

| 场景 | 推荐规则写法 |
|------|-------------|
| OG 标签取封面 | `//*[@property="og:image"]/@content` 或 `@css:meta[property="og:image"]@content` |
| OG 标签取作者 | `//*[@property="og:novel:author"]/@content` |
| OG 标签取分类 | `//*[@property="og:novel:category"]/@content` |
| OG 标签取简介 | `//*[@property="og:description"]/@content` |
| 取正文文本节点 | `id.content@textNodes` 或 `class.chapter-content@textNodes` |
| 搜索结果列表 | `class.search-list@tag.li` 或 `class.result-list@tag.li!0`（排除第一个非数据行） |
| 章节列表 | `class.chapter-list@tag.li` 或 `class.catalog-list@tag.a` |
| 翻页链接 | `class.pager@tag.a.-1@href`（最后一个 a 标签） |

---

### 第四步：构建 JSON

按以下完整模板生成书源 JSON：

```json
[
  {
    "bookSourceComment": "",
    "bookSourceGroup": "分组名",
    "bookSourceName": "网站名称",
    "bookSourceType": 0,
    "bookSourceUrl": "https://www.example.com",
    "bookUrlPattern": "",
    "customOrder": 0,
    "enabled": true,
    "enabledCookieJar": false,
    "enabledExplore": false,
    "exploreUrl": "",
    "header": "",
    "lastUpdateTime": 0,
    "loginUrl": "",
    "respondTime": 0,
    "searchUrl": "https://www.example.com/search?q={{key}}",
    "weight": 0,
    "ruleBookInfo": {
      "author": "",
      "coverUrl": "",
      "intro": "",
      "kind": "",
      "lastChapter": "",
      "name": "",
      "tocUrl": ""
    },
    "ruleContent": {
      "content": "",
      "nextContentUrl": "",
      "sourceRegex": "",
      "webJs": ""
    },
    "ruleExplore": [],
    "ruleSearch": {
      "author": "",
      "bookList": "",
      "bookUrl": "",
      "coverUrl": "",
      "intro": "",
      "kind": "",
      "lastChapter": "",
      "name": ""
    },
    "ruleToc": {
      "chapterList": "",
      "chapterName": "",
      "chapterUrl": "",
      "isVip": "",
      "nextTocUrl": ""
    }
  }
]
```

#### 字段说明

**基本信息：**
- `bookSourceUrl`：唯一标识，通常为网站根域名（如 `https://www.bxwx.la/`）
- `bookSourceName`：显示名称
- `bookSourceType`：0=文字书，1=有声书
- `enabledExplore`：发现功能是否启用（有 exploreUrl 时为 true）
- `enabledCookieJar`：需要 Cookie 时为 true
- `header`：需要自定义请求头时填写（JSON 字符串格式）

**搜索规则（ruleSearch）：**
- `bookList`：书籍列表容器规则，返回元素列表
- `name`：在每个书籍元素内取书名
- `author`：取作者
- `coverUrl`：取封面图片 URL
- `bookUrl`：取详情页 URL（支持相对 URL，会自动拼接 bookSourceUrl）
- `kind`：取分类/标签
- `lastChapter`：取最新章节
- `intro`：取简介

**详情规则（ruleBookInfo）：**
- `tocUrl`：目录页 URL（不填则使用 bookUrl 作为目录页）

**目录规则（ruleToc）：**
- `chapterList`：章节列表规则
- `chapterName`：章节名称
- `chapterUrl`：章节 URL
- `nextTocUrl`：目录下一页 URL（多页目录时填写）
- `isVip`：VIP 章节标识，结果为 null/false/0/"" 时为非 VIP

**正文规则（ruleContent）：**
- `content`：正文内容，优先使用 `textNodes` 保留段落格式
- `nextContentUrl`：正文下一页 URL

---

### 第五步：验证与输出

1. 检查所有必填字段（bookSourceUrl、bookSourceName、searchUrl）是否已填写
2. 检查 URL 是否使用了相对路径（如果是，需要用 baseUrl 拼接）
3. 确认 `searchUrl` 中含有 `{{key}}`
4. 如果网站有编码问题（如 GBK），在 URL 选项中添加 `{"charset":"gbk"}`
5. 输出格式为 JSON 数组 `[{...}]`，可直接导入 Legado App

---

## 参考示例

### 示例一：基于 CSS class 的 HTML 小说站（参考 1779125027.json）

```json
[{
  "bookSourceGroup": "小说",
  "bookSourceName": "笔下文学",
  "bookSourceType": 0,
  "bookSourceUrl": "https://www.bxwx.la/",
  "enabled": true,
  "enabledCookieJar": true,
  "enabledExplore": true,
  "searchUrl": "https://www.bxwx.la/ar.php?keyWord={{key}}",
  "ruleSearch": {
    "bookList": "class.txt-list txt-list-row5@tag.li!0",
    "name": "class.s2@text",
    "author": "class.s4@text",
    "coverUrl": "class.imgbox@img@src",
    "bookUrl": "class.s2@tag.a@href",
    "kind": "class.s1@text",
    "lastChapter": "class.s3@text"
  },
  "ruleBookInfo": {
    "coverUrl": "class.imgbox@img@src",
    "intro": "class.desc xs-hidden@text",
    "lastChapter": "class.section-list fix.0@.0@text"
  },
  "ruleToc": {
    "chapterList": "class.section-list fix.1@tag.li",
    "chapterName": "tag.a@text",
    "chapterUrl": "tag.a@href",
    "nextTocUrl": "class.right@tag.a@href"
  },
  "ruleContent": {
    "content": "id.content@textNodes",
    "nextContentUrl": "class.section-opt m-bottom-opt@tag.a.2@href"
  }
}]
```

### 示例二：基于 XPath 的小说站

```json
[{
  "bookSourceGroup": "小说",
  "bookSourceName": "采墨阁手机版",
  "bookSourceType": 0,
  "bookSourceUrl": "https://m.caimoge.com",
  "ruleBookInfo": {
    "author": "//*[@property=\"og:novel:author\"]/@content",
    "coverUrl": "//*[@property=\"og:image\"]/@content",
    "intro": "//*[@property=\"og:description\"]/@content",
    "kind": "//*[@property=\"og:novel:category\"]/@content"
  }
}]
```

### 示例三：基于 JSONPath 的 API 站

```json
[{
  "bookSourceGroup": "JSON",
  "bookSourceName": "猎鹰小说网",
  "bookSourceUrl": "http://api.book.lieying.cn",
  "searchUrl": "/Book/search?query={{key}}&start={{(page-1)*10}}",
  "ruleSearch": {
    "bookList": "$..books[*]",
    "name": "$.title",
    "author": "$.author",
    "coverUrl": "$.cover",
    "bookUrl": "/Book/getChapterListByBookId?bookId={$._id}"
  },
  "ruleToc": {
    "chapterList": "$.chapterInfo.chapters.[*]",
    "chapterName": "$.title",
    "chapterUrl": "$.link"
  },
  "ruleContent": {
    "content": "$.chapter.body"
  }
}]
```

---

## 特殊情况处理

### 网站需要 Cookie / 登录
- 设置 `"enabledCookieJar": true`
- 如果需要登录，设置 `"loginUrl"` 字段

### GBK 编码网站
搜索 URL 后追加编码参数：
```
https://www.example.com/search?key={{key}},{"charset":"gbk"}
```

### 需要 WebView 加载（反爬虫）
章节 URL 规则后追加：
```
tag.a@href##$##{"webView":true}
```
或在 URL 后加 `,{"webView":true}`

### 多页目录
在 `ruleToc.nextTocUrl` 中填写下一页链接规则。

### AJAX/动态内容
若页面内容通过 JS 动态加载，需在 URL 中使用 `{"webView":true}` 或分析对应的 API 接口。

### 有声书
设置 `"bookSourceType": 1`，正文规则返回音频 URL，配合 `sourceRegex` 使用嗅探功能。

---

## 注意事项

1. **bookSourceUrl 唯一性**：同一网站只能有一个书源，重复导入会覆盖
2. **相对 URL**：搜索结果和目录中的链接如果是相对路径，Legado 会自动拼接 bookSourceUrl
3. **规则调试**：输出的书源可在 Legado App 的「书源编辑」→「调试」功能中逐步验证
4. **字符集**：中文网站默认 UTF-8，遇到乱码时加 `{"charset":"gbk"}`
5. **位置索引**：Default 规则中位置从 0 开始，`-1` 为最后一个，`!0` 排除第一项
