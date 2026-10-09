（AI generated）

对 lxl66566.github.io（vuepress-theme-hope 2.0.0-rc.79 + VuePress 2.0.0-rc.20 + Vite + pnpm）的全量扫描结论，目标：从 0 用 SolidJS 写博客框架，markdown 内容与 URL 尽量不动。

## 一、现状盘点

- 技术栈：VuePress 2.0.0-rc.20 + vuepress-theme-hope 2.0.0-rc.79，bundler 为 Vite，包管理 pnpm；纯中文单语言站点（`lang: "zh-CN"`），无 i18n。
- 部署：GitHub Actions（`.github/workflows/deploy-docs.yml`）push `code` 分支后构建，`JamesIves/github-pages-deploy-action@v4` 把 dist 发布到 `main` 分支（写 `.nojekyll`）；站点同时挂在 GitHub Pages / Cloudflare / Netlify / VPS 反代，域名为 `https://absx.pages.dev`。
- 内容规模：170 个 md 文件、3707 个链接（677 个站内 `.md` 相对链接、318 个带锚点且大量为中文锚点、170 个 `.html` 链接、3 处 `#:~:text=` Text Fragment）、33 个手写 .vue 组件约 2900 行、`data/` 纯数据文件约 6700 行（galgame_list 2815、comic 1223、book_list 945、av 597、anime_list 366、job_list 136、article 93、gossip 52、learning 45）。
- md 目录结构与数量：articles 54（技术文章）、coding 35（编程笔记）、learning 22（课程笔记，公式重镇）、gossip 20（闲聊/作者）、hobbies 18（galgame.md 为 2000+ 行巨型文件）、essay 7（按年随笔）、hide 5（半隐藏页，不在导航）、farraginous 4（杂项）、blog 4（博客元页面）、index.md（站点主页）。
- 关键好消息：33 个手写组件完全不依赖 vuepress 内部 API（无 usePageData、无 @theme 别名导入），只 import `vue` + 自有 `data/`、`definition/`、`utils/` 模块 + fuse.js。Vue 到 Solid 是纯语法翻译，不牵扯框架数据层。

## 二、Markdown 渲染管线（最大头，"不改 md"的关键）

新框架需要一个 markdown-it 管线。主题开了约二十个特性，实际用到的按用量决定实现范围。

必须实现（有真实用量）：

| 特性 | 实际用量 | 实现方式 |
| --- | --- | --- |
| `:::` 容器 | tabs 76、details 69、right(align) 22、tip 20、code-tabs 16、warning 13、danger 8、caution/error/info 各 1，共约 228 个开启行，嵌套用 `::::` | markdown-it-container 逐类型注册 |
| `@tab` | 235 处、32 个文件；含 `::: tabs#card` 带 id 的持久化变体（uno.md 单文件 29 个） | 自写或参考现有 tabs 插件，id 持久化选中状态 |
| 数学公式 | 行内 `$...$` 约 743 处、35 个文件；`$$` 块级为 0 | markdown-it-katex（或 texmath），KaTeX 沿用 |
| 脚注 | 129 个定义 + 146 个引用、52 个文件，存在中文 id（如 `[^胆结石]`） | markdown-it-footnote，需验证 Unicode label |
| tasklist | 210 项、3 个文件（blog/todo.md、hide/inspiration.md、gossip/schedule.md） | markdown-it-task-lists |
| Shiki 高亮 | 全站代码块；`:collapsed-lines` / `:no-collapsed-lines` 代码块属性 8 处、`{2}` 行高亮 | shiki + 代码块 attrs 支持 + collapsedLines transformer |
| 图片尺寸 | `=300x` 语法 12 处 | markdown-it-imsize 或 attrs |
| mark `==x==` | 约 5 处 | markdown-it-mark |
| figure | 29 张独立成行图片转 `<figure>` | 自写小规则 |
| HTML 透传 | `<span>` 321（含 class="heimu" 92）、`<br>` 197、`<div>` 40、`<img>` 19、`<iframe>` 17（嵌 public/charts 静态图表页）、`<details>` 12、`<text style="color:red">` 84 处（galgame.md，非标准标签） | markdown-it 开 `html: true` 原样通过，浏览器按未知元素渲染 |

可以砍掉（开启但 0 使用）：`include`(@src)、`spoiler`、`imgMark`、`stylize`(Recommended→Badge)、`<!-- more -->` 摘要分隔符（全站 0 个，`excerptLength: 1` 本来等于没摘要）。

两个隐蔽的硬骨头：

1. 中文锚点 slug：318 个站内链接带锚点，锚点 id 必须与 VuePress 的 heading id 生成规则完全一致（保留中文、空格转连字符），否则全站深链断裂。这是"不改 md"的硬约束。
2. `.md` 相对链接解析：677 个 `./xxx.md` / `../xxx.md` 链接与 170 个 `.html` 链接，意味着新框架应沿用 `.html` 后缀的 URL 结构，构建期把 md 相对链接重写到目标路由。另有 3 处 `#:~:text=`，theme.ts 专门为此关掉了 activeHeaderLinks，新框架不要做滚动时改写 URL hash 的功能。

## 三、md 内嵌组件与 `<template #插槽>`（最特殊的部分）

组件注册分两层：`client.ts` 全局注册 9 个小组件（TelegramLink、RSSLink、OrderBadge、dtls、dtlslong、heimu、ZoomedImg、dated、furigana）；`config.ts` 用 vite alias 注册 12 个列表组件供 md 直接使用（AvTable、ComicTable、GalExhibitionGrid、Av、reciter、ArticleCell、GalList、BookList、AnimeList、JobList、SpeedupList、CryptocurrencyExchangeList）。

md 中的实际用量：

- 全局小组件：`<heimu>` 黑幕 160 个标签（40+ 文件）、`<Badge>`（主题）43、`<furigana>` 注音 44、`<dtlslong>` 38、`<ZoomedImg>` 26、`<dated>` 15、`<dtls>` 10、`<OrderBadge>` 5、`<Av>` 14、`<VPIcon>` 2。
- 大型列表组件 + 具名插槽：`<GalList>` 配 128 个 `<template #作品名>`、`<BookList>` 50、`<SpeedupList>` 25、`<AnimeList>` 22、`<JobList>` 17、另有 `<ArticleCell>`、`<GalExhibitionGrid>` 等，全站约 250 个 `<template #xxx>`。

这是 SolidJS 迁移最别扭的点：Solid 没有具名插槽语法，而 md 里有 250 个。推荐路线：在 markdown 渲染管线加一个构建期转换规则，把 `<GalList>...</GalList>` 内部的 `<template #xxx>内容</template>` 解析出来转成 props 传给 Solid 组件（如编译成 island 调用 `<GalList items={...} slots={{...}}>` 或按 name 分桶的 children），md 一字不改。备选路线（改组件 API + 批量改 md）违背"不改 md"前提，不取。

架构建议：走"构建期把 md 渲染成静态 HTML + 交互组件做成 Solid islands"的路线（VuePress 本质也是静态内容）。正文、容器、表格全是静态 HTML，只有 GalList 这类列表组件与主题 chrome 是 Solid。这也是把 250 个插槽转换做在构建期的前提。

**作者说：这部分 markdown 需要修改，而不是去兼容。因为目的是抛弃陈旧的 vue 语法，改用强大的 tsx。这些 markdown 部分要搞成 mdx 还是就正常 md + 编译期替换，你自己看。**

## 四、Vue 到 Solid 组件迁移

33 个组件（components/ 目录，约 2900 行）迁移为纯语法翻译。映射关系：

- `ref` -> `createSignal`；`computed` -> `createMemo`；`watch` -> `createEffect`；`onMounted`（9 处）-> `onMount`。
- `v-model`（17 处）-> `value` prop + `onChange`/`onInput` 回调。
- `defineProps`（21 处）-> Solid props 解构或直接访问；`defineEmits`（2 处）-> 回调 props。
- 默认插槽 -> `props.children`；`useSlots()` 动态具名插槽（GalList 按条目名取插槽再转发的模式）-> 传 render 函数或键值对。
- 组件依赖：仅 `vue`、自有 `data/*.ts`（纯数据，原样复用）、`definition/*.ts`（纯类型，原样复用）、`utils/*.ts`（debounce/FormatDate/NumberToChinese/PartitionArray，纯函数，原样复用）、fuse.js（沿用）。

代表组件规模：GalList 263 行 + GalListItem 173 行（搜索/排序/展开，fuse.js + 动态插槽，最复杂）、CryptocurrencyExchangeList 212 行、SpeedupList 268 行、ZoomedImg 164 行（photoSwipe 类缩放交互）、heimu 55 行（最简单，可作迁移模板验证管线）。

## 五、布局系统：navbar / sidebar / TOC

省略

## 六、博客系统

- frontmatter 约定（全站实际只用这几个键）：`date`(170)、`category`(169，单数键名 + YAML 列表)、`tag`(140，同样单数)、`icon`(159，FontAwesome 短名)、`feed: false`(3，均在 hide/)、`externalLinkIcon: false`(3，NSFW 三页)、首页的 `home`/`layout: BlogHome`/`hero: false`。没有任何文件用 `title`，标题全靠正文 `# 一级标题`，新框架要从首个 h1 提取标题。没有 star/order/sticky/cover/excerpt 等。
- 首页（src/index.md）：`layout: BlogHome` + `hero: false`，渲染博主信息卡（name 绝对值_x、intro /gossip/author.html、medias GitHub+Telegram+Bilibili）+ 文章流（articlePerPage: 3 分页）。首页样式可重写，但文章流 + 每页 3 篇分页的行为要有。
- 标签/分类归档页：没有手写 tags.md/categories.md，归档页完全由主题 blog 插件自动生成（`/category/xxx`、`/tag/xxx`，从文章信息里的链接进入）。新框架要实现从 frontmatter 聚合 category/tag 生成归档页（带分页），URL 路径建议保持一致。
- 文章信息行：`Date, Category, PageView, Tag, ReadingTime`。PageView 是死的（Giscus 不提供浏览量，站内无任何统计后端），可砍。ReadingTime 需自实现（中文按字数/阅读速度）。
- RSS：`feed: { rss: true, count: 20 }` 生成 `/rss.xml`，`feed: false` 可排除（hide/ 三页在用）。构建期生成即可。

## 七、功能集成

Algolia DocSearch

Giscus 评论区（GitHub Discussions）

文章加密

FontAwesome 图标

图片体验与其他

photoSwipe（点图放大）+ imgLazyload 为全自动插件，新框架要自带等价物（photoSwipe 库可直接沿用 + `loading="lazy"`）。Google Analytics（G-MKRDBH1ZP1，一个 script 标签）。SEO 插件（hostname https://absx.pages.dev，生成 og meta）；当前反而未装 sitemap 插件，新框架可补上。暗色模式为主题默认开关（`html[data-theme="dark"]` + localStorage），giscus 与样式都挂在该属性上。`remove-pwa` 插件是一次性清理历史 PWA 缓存，迁移后不需要。
