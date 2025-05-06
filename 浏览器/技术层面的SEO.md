### 1. 多语言实施 - 如果目标受众是多语言用户，这一步非常重要
（1）使用正确的URL结构

子域名：例如，使用 ar.example.com 为阿语版本。

子目录：例如，使用 example.com/ar/ 为阿语版本。

顶级域名：对于完全不同的地区，使用不同的顶级域名，如 example.ar。

根据实际需要进行选择，一般example.ar有利于本地排名，ar.example.com能获取主站的权重，但这两者在运营成本和管理方面均有明显权限；example.com/ar/ 管理方便，但对本地排名无特殊加分。

注意：避免使用带有语言参数的动态链接，例如 lang=ar，来创建多语言网站。它可能导致搜索引擎索引和排名效率降低、内容重复风险增加，以及网站追踪变得更加复杂。


Reference: https://developers.google.com/search/docs/specialty/international/managing-multi-regional-sites?hl=zh-cn



（2）实施hreflang标签

hreflang标签是一个HTML标记，用于告诉搜索引擎您的网页有不同语言的版本。这对于多语言网站来说非常重要，因为它帮助搜索引擎了解哪个版本的内容适用于特定的语言或地区用户，从而提供更准确的搜索结果。正确使用hreflang标签可以提高网站在不同语言搜索中的可见性和相关性，避免内容重复的问题。

假设网站提供英语、阿拉伯语和土耳其语版本，您可以在每个页面的<head>部分添加如下hreflang标签：

```html
<link rel="alternate" hreflang="en" href="https://www.example.com/en/" />

<link rel="alternate" hreflang="ar" href="https://www.example.com/ar/" />

<link rel="alternate" hreflang="tr" href="https://www.example.com/tr/" />
```

### 2. 优化页面URL

使用简短、准确、包含关键词的URL。

使用关键词：URL应包含您希望页面针对的关键词。

使用连字符分隔单词：在URL中使用连字符（-）作为“单词分隔符”。避免使用下划线或空格。

使用小写字母：大多数现代服务器将URL中的大写和小写字母视为相同，但为了安全起见，建议使用小写字母。

避免使用特殊字符或符号：例如 & 和 %

避免使用日期：在URL中包含日期会使URL变长，并且在更新内容时可能会产生混淆。

避免动态URL：从纯SEO的角度来看，动态URL参数（如UTM跟踪）可能会导致问题。

帮助导航：使用有组织的URL子文件夹，使用户和搜索引擎容易理解网站的不同部分。

Reference：https://developers.google.com/search/docs/crawling-indexing/url-structure?hl=zh-cn

### 3. 站点地图（Sitemap）- 让搜索引擎更容易找到和索引网站的所有页面

创建一个sitemap.xml文件，列出网站上的所有页面，并通过Google Search Console提交（及其他搜索引擎站长平台），以加速Google的索引过程。

备注：如果网站只有低于10个页面，可暂时不创建，直接向GSC提交单个页面收录。

Reference: https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap?hl=zh-cn

### 4. robots.txt文件 - 指导搜索引擎哪些页面应该或不应该被爬取

robots.txt文件是一个在网站根目录下的文本文件，它可以指定哪些内容可以被搜索引擎爬虫访问，哪些不可以。这有助于防止搜索引擎索引敏感或不重要的页面。

robots.txt文件可以阻止爬虫访问页面，但如果页面被其他网站链接，它仍然可能被索引。要完全阻止页面被索引，应使用noindex标签。

记得将站点地图添加到 robots.txt 文件

Reference: https://developers.google.com/search/docs/crawling-indexing/robots/create-robots-txt?hl=zh-cn

### 5. 结构化数据

Google 搜索会尽力了解网页内容。您可以在网页上添加结构化数据，向 Google 提供有关该网页含义的明确线索，从而帮助我们理解该网页。结构化数据是一种提供网页相关信息并对网页内容进行分类的标准化格式；例如，食谱网页上会有食材、烹饪时长和温度、卡路里等各类信息。

Google 会利用在网络上找到的结构化数据来了解网页内容并收集有关网络和世界的一般信息，例如标记中关于人物、图书或公司的信息。例如，当食谱网页包含 JSON-LD 结构化数据（描述食谱的标题、作者和其他详情）时，Google 搜索可以使用这些信息显示食谱的富媒体搜索结果：

![食谱网页的结构化数据如何影响 Google 搜索中的富媒体搜索结果](https://developers.google.com/static/search/docs/images/structured-data-explainer.png?hl=zh-cn)

了解结构化数据的工作方式： https://developers.google.com/search/docs/advanced/structured-data/intro-structured-data?hl=zh-cn

结构化数据常规指南： https://developers.google.com/search/docs/advanced/structured-data/sd-policies?hl=zh-cn

结构化数据如何提升搜索结果：https://seoasoorm.com/zh/rich-results/

如何添加及测试结构化数据：https://seoasoorm.com/zh/structured-data/

### 6. Canonical 标签 – 整合重复网址

使用规范标记整合重复网址： https://developers.google.com/search/docs/advanced/crawling/consolidate-duplicate-urls?hl=zh-cn

使用规范标记的最佳实践：https://seoasoorm.com/zh/canonical-tags/



### 7. 设置Google Analytics - 用于跟踪网站流量和用户行为

在网站上安装Google Analytics，以跟踪访问者行为和网站性能。你可以通过Google Tag Manager统一安装和管理Google（及其他常见的）的各类工具。

### 8.  将网站提交到Google搜索控制台

你可以使用 Google Search Console 执行许多操作，例如发现技术错误、监控排名、提交站点地图等等。要将网站添加到 Google Search Console，请使用Google帐户登录Google Search Console。

Reference: https://developers.google.com/search/docs/monitor-debug/search-console-start?hl=zh-cn


### 9. 移动端优化
保持URL一致性：无论用户是通过桌面还是移动设备访问，都应使用相同的URL。此外，尽量使用简洁、描述性强的静态URL，避免使用过长或包含大量参数的动态URL。

响应式设计：使用响应式设计确保网站在各种设备上都能正确显示。这意味着网站的布局和内容会根据用户设备的屏幕尺寸自动调整。

快速加载：移动设备用户通常对加载速度更为敏感。优化图片大小、使用缓存和减少不必要的脚本，以加快网站在移动设备上的加载速度。

Reference: https://developers.google.com/search/docs/crawling-indexing/mobile/mobile-sites-mobile-first-indexing?hl=zh-cn


### 10.  优先考虑直观导航

确保菜单项易于访问，所有重要分类页面都可以从主菜单访问。

### 11.  页面标题 & 元描述 - 这些是用户在搜索结果中看到的第一印象

标题应简洁明了，包含最重要的关键词，并尽可能包含品牌名称。标题长度最好在45到60个字符之间。

元描述是对网页内容的简短总结，每个页面应有独特的描述（即不应该在不同的页面使用相同的元描述），合理在表达中嵌入关键词，长度约150个字符。

Meta Keywords完全不需要写，它在10年前就对Google排名没有任何帮助了。

### 12.  图标（Favicon） - 小细节，但有助于提升品牌识别度
Favicon是显示在浏览器标签、书签列表、历史记录、搜索结果等地方的小图标。它通常是网站标志的简化版本。

常见的Favicon格式是.ico，但现在也支持.png和.gif格式。标准尺寸是16x16或32x32像素。

你需要将Favicon文件放在网站的根目录下，并在HTML的<head>部分正确地链接到它。

Reference: https://developers.google.com/search/docs/appearance/favicon-in-search?hl=zh-cn