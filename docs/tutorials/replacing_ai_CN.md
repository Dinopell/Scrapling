# Scrapling：稳健 Web 抓取的免费 AI 替代方案

Web 抓取长期以来是数据提取、索引与数据集准备等用途的重要工具。但经验丰富的用户常遇到阻碍效果的问题。近来，基于 AI 的 Web 抓取明显增多，因其有潜力解决这些挑战。

本文将讨论这些常见问题、企业转向该方案的原因、该方案的问题，以及 Scrapling 如何在不使用 AI 成本的情况下为你解决它们。

## 常见问题与具有挑战性的目标

若你从事 Web 抓取已久，可能注意到一些反复出现的问题，例如：

1. **网站结构快速变化** - 网站频繁更新 DOM 结构，导致静态 XPath/CSS 选择器失效。
2. **不稳定的选择器** - 类名与 ID 常变化或使用随机生成值，使爬虫失效或增加抓取难度。
3. **日益复杂的反机器人措施** - CAPTCHA、浏览器指纹识别与行为分析使传统抓取变难
等等

但这仅当你针对已知网站做定向 Web 抓取时如此，此时可为每个网站编写特定代码。

若你开始考虑更大目标，如广域抓取或通用 Web 抓取（随你怎么称呼），上述问题会加剧，并面临新问题，例如：

1. **极端网站多样性** - 通用抓取必须处理 HTML 结构、CSS 用法、JavaScript 框架与后端技术的无数变体。
2. **识别相关数据** - 爬虫如何知道从未见过的页面上哪些数据重要？
3. **分页变体** - 无限滚动、传统分页、「加载更多」按钮，均需不同方法
等等

你如何手动解决？我指的是抓取各种不共享任何共同技术的网站的通用 Web 抓取。

## AI 来救场，但代价高昂

当然，AI 能轻松解决大多数问题，因为它能理解页面源码并识别你想要的字段或为其创建选择器。当然，前提是你已通过其他工具解决反机器人措施 :)

这种方法当然很美。我热爱 AI，尤其着迷于生成式 AI。你可能会在提示工程与调优上花很多时间，但若你能接受，很快会碰到在此使用 AI 的真正问题。

多数网站每页有大量内容，你需要以某种方式传给 AI 才能让它发挥作用。这会像干草堆里的火一样消耗 Token，迅速累积高昂成本。

除非钱对你无关紧要，否则你会寻找更便宜的方案，而 Scrapling 就派上用场了 :smile:

## Scrapling 为你兜底

Scrapling 能处理 Web 抓取中你将面临的几乎所有问题，后续更新将谨慎覆盖其余部分。

### 解决问题 T1：网站结构快速变化
这就是 [adaptive](https://scrapling.readthedocs.io/en/latest/parsing/adaptive.html) 功能的由来。我就知道会谈到它，果然 :)

在 Web 抓取时，若启用 `adaptive` 功能，可保存任意元素的唯一属性，以便网站结构变化后再次找到它。改版最令人沮丧的是元素的任何方面都可能变化，因此没有可依赖的东西。

自适应功能的工作原理是：存储元素的一切独特之处。当网站结构变化时，返回与先前元素相似度得分最高的元素。

我已更详细解释并附大量示例。更多内容见[此处](https://scrapling.readthedocs.io/en/latest/parsing/adaptive.html#how-the-adaptive-feature-works)。

### 解决问题 T2：不稳定的选择器
若你从事 Web 抓取足够久，可能遇到过这种情况。我指的是采用糟糕设计模式的网站：基于原始 HTML 构建、没有任何 ID/类，或使用随机类名且无其他可依赖特征等...

在这些情况下，标准 CSS/XPath 选择器方法并非最优，因此 Scrapling 提供三种额外选择方法：

1. [按元素内容选择](https://scrapling.readthedocs.io/en/latest/parsing/selection.html#text-content-selection)：通过文本内容（`find_by_text`）或匹配文本内容的正则（`find_by_regex`）
2. [选择与另一元素相似的元素](https://scrapling.readthedocs.io/en/latest/parsing/selection.html#finding-similar-elements)：你找到一个元素，其余交给我们！
3. [按过滤器选择元素](https://scrapling.readthedocs.io/en/latest/parsing/selection.html#filters-based-searching)：你指定元素必须满足的条件/过滤器，我们帮你找到！

无需解释这些；点击链接即可清楚 Scrapling 如何解决。

### 解决问题 T3：日益复杂的反机器人措施
众所周知，创建难以检测的爬虫不仅需要住宅/移动代理与类人行为，还需要难以检测的浏览器，Scrapling 提供两种主要方案：

1. [DynamicFetcher](https://scrapling.readthedocs.io/en/latest/fetching/dynamic.html) - 该 Fetcher 提供灵活的浏览器自动化，多种配置选项及少量底层隐蔽改进。
2. [StealthyFetcher](https://scrapling.readthedocs.io/en/latest/fetching/stealthy.html) - 因为我们生活在严酷世界，你需要[全力以赴而非半途而废](https://www.youtube.com/watch?v=7BE4QcwX4dU)，`StealthyFetcher` 应运而生。该 Fetcher 使用我们的隐蔽浏览器——[DynamicFetcher](https://scrapling.readthedocs.io/en/latest/fetching/dynamic.html) 的隐蔽版本，几乎绕过所有烦人的反防护，提供工具处理其余部分，并自动绕过所有类型的 Cloudflare Turnstile/Interstitial！

我们每次更新都会持续改进这两者，敬请期待 :)

### 解决问题 B1 与 B2：极端网站多样性 / 识别相关数据

这个问题较难处理，但 Scrapling 的灵活性使之成为可能。

我曾与一位用 AI 从不同网站提取价格的人交谈。他只关心价格与标题，因此用 AI 帮他找价格。

我告诉他这里不需要 AI，并给出如下示例代码
```python
price_element = page.find_by_regex(r'£[\d\.,]+', first_match=True)  # 获取文本匹配价格正则的第一个元素，如 £10.50
# 若需要包含价格元素的容器/元素
price_element_container = price_element.parent or price_element.find_ancestor(lambda ancestor: ancestor.has_class('product'))  # 或其他方法...
target_element_selector = price_element_container.generate_css_selector or price_element_container.generate_full_css_selector # 或 xpath
```
然后他说像这样的情况怎么办：
```html
<span class='currency'> $ </span> <span class='a-price'> 45,000 </span>
```
于是我将代码更新为
```python
price_element_container = page.find_by_regex(r'[\d,]+', first_match=True).parent # 为本示例调整了正则
full_price_data = price_element_container.get_all_text(strip=True)  # 本例返回 '$45,000'
```
这已满足他的用例。你可以先用第一个正则，找不到再用下一个，依此类推。先覆盖最常见模式，再覆盖较少见的，等等。
会有点枯燥，但肯定比 AI 便宜。

此示例说明我想表达的观点。并非每个挑战都需要 AI 解决，有时需要发挥创意，这可能为你省下很多钱。

### 解决问题 B3：分页变体
这个问题，Scrapling 目前没有直接自动提取分页 URL 的方法，但将在即将到来的更新中添加 :)

但你可以通过搜索最常见模式处理大多数网站，如 `page.find_by_text('Next')['href']` 或 `page.find_by_text('load more')['href']`，或选择器如 `'a[href*="?page="]'` 或 `'a[href*="/page/"]'`——你懂的。

## 成本对比与节省
快速对比。

| 方面         | Scrapling                                                                  | 基于 AI 的工具（如 Browse AI、Oxylabs）                                  |
|----------------|----------------------------------------------------------------------------|----------------------------------------------------------------------------|
| 成本结构 | 可能免费或低成本，无按次费用                                   | 起价 $19/月（Browse AI）至 $49/月（Oxylabs），随用量扩展  |
| 搭建工作量   | 需要少量技术知识，手动搭建                          | 常为零代码，对非技术用户更易                              |
| 使用方式  | 通过代码、终端或 MCP 服务器。                                     | 常通过 GUI 或 API，取决于公司提供的选项 |
| 可扩展性    | 取决于用户实现                                             | 内置大规模托管服务支持                         |
| 适应性   | 通过 `adaptive` 与非选择器选择方法实现高适应性 | 通过 AI 自动实现高适应性，但频繁变化时成本高                   |

本表基于 [Browse AI 定价](https://www.browse.ai/pricing) 与 [Oxylabs Web Scraper API 定价](https://oxylabs.io/products/scraper-api/web/pricing)

## 结论
虽然 AI 提供强大能力，但其成本对许多 Web 抓取任务可能令人望而却步。Scrapling 提供稳健、灵活且高性价比的工具包，应对定向与广域抓取的真实挑战，常可消除对昂贵 AI 方案的需求。通过利用 `adaptive`、多样选择方法与高级 Fetcher 等功能，你可以更高效地构建弹性爬虫。

进一步探索文档，看看 Scrapling 如何简化你未来的 Web 抓取项目！
