---
title: Obsidian DataviewJS 插件使用实例
date: 2023-01-01 00:00:00
tags:
---

## 前言

Obsidian 是一个功能强大的笔记应用程序，越来越多的用户选择使用它来记录自己的思想、保存自己的笔记。它支持最近很火的双链功能，能在任意页面之间建立连接，最终形成知识网络。也可以不使用它的双链功能，只将其作为我们的个人知识管理库。
作为一款开源软件，Obsidian 的社区非常活跃，用户自己同时作为开发者在不断开发新的插件来扩展 Obsidian 的功能。其中一个非常有用的插件是 Dataview，它可以让用户使用 Markdown 文件中的数据来创建自定义视图和列表等。使用 Dataview，我们可以轻松地汇总、分析和可视化笔记中的数据。

Dataview 的另一个优点是可以通过简单的标记语言来过滤数据，使得用户可以快速、准确地找到他们需要的信息。这些标记可以使用类似 SQL 的语言进行编写，甚至支持自定义 JavaScript 函数来进一步扩展功能。

如果你是一个需要从海量笔记中快速获取信息的人，那么 Dataview 将是非常值得尝试的插件。本文中，我们将重点介绍它的使用和功能，同时也会分享一些我自己的使用经验和技巧。

## Dataview Query Language

虽然本文讨论的是 Dataview 中的 JavaScript，但是还是简要介绍一下 Dataview 中的 Query Language。

一个完整的 Dataview 查询包括以下要素

```sql
<QUERY-TYPE> <fields>
FROM <source>
<DATA-COMMAND> <expression>
<DATA-COMMAND> <expression>
        ...
```

- QUERY-TYPE 可以为 LIST、TASK、CALENDAR、TABLE
- fileds 是要查询的字段
- source 是数据来源，类似于 SQL 要查询的表
- DATA-COMMAND 是对数据的限制条件
  Dataview 使用示例：例如要查询当前笔记（Markdown 文件）同级目录下的 Books 文件夹中的所有文件并按照创建时间降序排序，可编写 Dataview 查询如下：

```sql
LIST
FROM "Books"
SORT file.ctime DESC
```

如果我们要在此基础上加一点限制条件，例如文件中含有 `#致用` 标签的文件，则可以使用如下查询语句：

```sql
LIST
FROM "Books"
SORT file.ctime DESC
WHERE contains(tags, "#致用")
```

更多示例可参考官方文档中关于 [Query Language](https://blacksmithgu.github.io/obsidian-dataview/queries/structure/) 的介绍。

## DataviewJS

尽管 Dataview Query Language 能满足绝大部分需求，在部分场景下，我们希望能自定义查询。下面我们从具体实例出发，使用 DataviewJS 完成不同自定义任务。

### 为指定名称文件夹内的文件编制索引

如果我们想查询当前文件夹下与当前文件名相同的文件夹内的所有文件，假设我们的文件目录如下：

```plaintext
├─ 离散数学.md
└─ 离散数学
　　 ├─ 01 命题逻辑.md
　　 ├─ 02 谓词逻辑.md
　　 ├─ 03 集合与关系.md
　　 ├─ 04 函数.md
　　 ├─ 05 代数系统.md
　　 └─ 06 图和树.md
```

我们希望使用 `离散数学.md` 为文件夹内的文件的索引，并按照序号排序，则可编写 DataviewJS 如下：

```jsx
// 获取当前文件名
const fileName = dv.current().file.name;
// 提取文件前缀编号: 例如 01、02、03...
const fileIndex = fileName.slice(0, 2);
// 获取文件所在文件夹
let path = dv.current().file.folder;
// 如果文件不在根目录则在目录后加上路径分隔符
if (path) path += "/";
// 如果文件有编号则子文件夹为编号命名, 否则与该文件相同
if (!isNaN(parseInt(fileIndex))) path += fileIndex;
else path += fileName;
// 使用 list 将结果返回
dv.list(
  dv
    // 根据官方文档, 路径两边的双引号应该保留
    .pages(`"${path}"`)
    // 只查询一级子目录下的文档
    .where((p) => p.file.folder === path)
    // 按照文件名升序排序
    .sort((c) => c.file.name).file.link
);
```

得到的结果与预期相符。

### 为会议记录编制清单

为了实现编程读取文件名中的日期，我们需要将所有会议记录文件按照格式 `年月日_会议主题` 来命名，例如 `20220422_软件组会`，然后我们就可以在会议文件夹所在目录下建立与会议文件夹同名的文件，并编程如下，即可得到预期结果。

```plaintext
├─ 会议.md
└─ 会议
　　 ├─ 20220114_社会调查.md
　　 ├─ 20220514_常规组会.md
　　 └─ 20220626_教育评议.md
```

```jsx
dv.table(
  // 表头为 Date、Category、Title 三个字段
  ["Date", "Category", "Title"],
  dv
    // 在与当前文件同名的文件夹中查找结果
    .pages(`"${dv.current().file.name}"`)
    // 结果按字母序顺序排序
    .sort((page) => page.file.name)
    // 对 page 进行操作来最终获得表头的三个字段
    .map((page) => {
      const fileName = page.file.name;
      // 匹配开头的日期, 格式必须是 20220305
      const matchResult = fileName.match(/(\d{4})(\d{2})(\d{2})/);
      // 自定义链接名, 使用 substring() 去掉文件名的日期部分
      const updatedFileLink = dv.fileLink(
        page.file.path,
        false,
        fileName.substring(9)
      );
      return [
        // Date: 使用 - 分割年月日
        `${matchResult[1]}-${matchResult[2]}-${matchResult[3]}`,
        // Category: 每个文件中的 tag
        page.file.tags,
        // Title: 自定义链接名作为标题
        updatedFileLink,
      ];
    })
);
```

### 为 OJ 刷题记录编制索引

此例中，我们需要记录来自不同平台题目，需要根据 URL 提取平台信息，并显示当前文件的通过状态，例如 A 表示 Accept，N 表示 Not Started，W 表示 Wrong Answer。实现这样的功能需要在子目录的 markdown 头部添加 metadata 如下：

```yaml
---
URL: https://leetcode.cn/problems/two-sum
Status: A
---
```

然后我们就可以使用 `page.status` 和 `page.url` 访问到这两个变量，并根据需要进行映射，得到最终结果。

```jsx
dv.table(
  ["Title", "Source", "Tags", "Status"],
  dv
    .pages(`"${dv.current().file.folder}/${dv.current().file.name}"`)
    .sort((b) => [b.status, b.tags, b.title])
    .map((b) => {
      let result, source;
      // 用于映射 URL 中的关键字和 table 的 Source 字段
      const sourceMap = {
        leetcode: "LeetCode",
        pintia: "PinTiA",
        atcoder: "AtCoder",
        luogu: "洛谷",
        codeforce: "Codeforces",
        nowcoder: "牛客",
      };
      // 用于将文件中的 A、N、W 映射为不同颜色的状态显示在表格中
      const statusMap = {
        A: "<b><font color=green>Accept</font></b>",
        N: "<b><font color=grey>Not Started</font></b>",
        W: "<b><font color=red>Wrong Answer</font></b>",
      };
      if (b.url) {
        for (const k in sourceMap) {
          if (~b.url.indexOf(k)) {
            source = sourceMap[k];
            break;
          }
        }
      } else source = "其他";
      for (const k in statusMap) {
        if (k === b.status) {
          result = statusMap[k];
          break;
        }
      }
      // 和表头的四个字段一一对应
      return [b.file.link, source, b.file.tags, result];
    })
);
```

得到的结果如下：

![](1678841002699.png)

## 参考资料

1. [DataviewJS 文档](https://blacksmithgu.github.io/obsidian-dataview/)
