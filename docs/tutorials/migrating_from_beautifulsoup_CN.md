# 从 BeautifulSoup 迁移到 Scrapling

若你已熟悉 BeautifulSoup，你会惊喜地发现：Scrapling 更快，提供与 BS 相同的解析能力，并增加 BS 中没有的解析功能，还为获取与处理现代网页引入强大新特性。本指南帮助你快速将现有 BeautifulSoup 代码适配为 Scrapling。

下表涵盖抓取网页时最常见的操作。每行展示如何用 BeautifulSoup 完成特定任务，以及 Scrapling 中的对应方法。

你会注意到 BeautifulSoup 中的部分快捷方式在 Scrapling 中缺失，这也是 BeautifulSoup 比 Scrapling 慢的原因之一。要点是：若同一功能已能用简短一行实现，就没必要牺牲性能去缩短那行代码 :)


| 任务                                                            | BeautifulSoup 代码                                                                                            | Scrapling 代码                                                                    |
|-----------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------|
| 解析器导入                                                   | `from bs4 import BeautifulSoup`                                                                               | `from scrapling.parser import Selector`                                           |
| 从字符串解析 HTML                                        | `soup = BeautifulSoup(html, 'html.parser')`                                                                   | `page = Selector(html)`                                                           |
| 查找单个元素                                        | `element = soup.find('div', class_='example')`                                                                | `element = page.find('div', class_='example')`                                    |
| 查找多个元素                                       | `elements = soup.find_all('div', class_='example')`                                                           | `elements = page.find_all('div', class_='example')`                               |
| 查找单个元素（示例 2）                            | `element = soup.find('div', attrs={"class": "example"})`                                                      | `element = page.find('div', {"class": "example"})`                                |
| 查找单个元素（示例 3）                            | `element = soup.find(re.compile("^b"))`                                                                       | `element = page.find(re.compile("^b"))`<br/>`element = page.find_by_regex(r"^b")` |
| 查找单个元素（示例 4）                            | `element = soup.find(lambda e: len(list(e.children)) > 0)`                                                    | `element = page.find(lambda e: len(e.children) > 0)`                              |
| 查找单个元素（示例 5）                            | `element = soup.find(["a", "b"])`                                                                             | `element = page.find(["a", "b"])`                                                 |
| 按文本内容查找元素                                | `element = soup.find(text="some text")`                                                                       | `element = page.find_by_text("some text", partial=False)`                         |
| 用 CSS 选择器查找第一个匹配元素          | `elements = soup.select_one('div.example')`                                                                   | `elements = page.css('div.example').first`                                        |
| 用 CSS 选择器查找所有匹配元素                | `elements = soup.select('div.example')`                                                                       | `elements = page.css('div.example')`                                              |
| 获取页面/元素源码的美化版本             | `prettified = soup.prettify()`                                                                                | `prettified = page.prettify()`                                                    |
| 获取页面/元素源码的非美化版本             | `source = str(soup)`                                                                                          | `source = page.html_content`                                                      |
| 获取元素标签名                                      | `name = element.name`                                                                                         | `name = element.tag`                                                              |
| 提取元素文本内容                           | `string = element.string`                                                                                     | `string = element.text`                                                           |
| 提取文档或标签下所有文本          | `text = soup.get_text(strip=True)`                                                                            | `text = page.get_all_text(strip=True)`                                            |
| 访问属性字典                             | `attrs = element.attrs`                                                                                       | `attrs = element.attrib`                                                          |
| 提取属性                                           | `attr = element['href']`                                                                                      | `attr = element['href']`                                                          |
| 导航到父元素                                            | `parent = element.parent`                                                                                     | `parent = element.parent`                                                         |
| 获取元素所有父级                                   | `parents = list(element.parents)`                                                                             | `parents = list(element.iterancestors())`                                         |
| 在元素父级中搜索元素           | `target_parent = element.find_parent("a")`                                                                    | `target_parent = element.find_ancestor(lambda p: p.tag == 'a')`                   |
| 获取元素所有兄弟节点                                  | N/A                                                                                                           | `siblings = element.siblings`                                                     |
| 获取下一个兄弟节点                                  | `next_element = element.next_sibling`                                                                         | `next_element = element.next`                                                     |
| 在元素兄弟节点中搜索元素          | `target_sibling = element.find_next_sibling("a")`<br/>`target_sibling = element.find_previous_sibling("a")`   | `target_sibling = element.siblings.search(lambda s: s.tag == 'a')`                |
| 在元素兄弟节点中搜索多个元素            | `target_sibling = element.find_next_siblings("a")`<br/>`target_sibling = element.find_previous_siblings("a")` | `target_sibling = element.siblings.filter(lambda s: s.tag == 'a')`                |
| 在元素后续元素中搜索元素     | `target_parent = element.find_next("a")`                                                                      | `target_parent = element.below_elements.search(lambda p: p.tag == 'a')`           |
| 在元素后续元素中搜索多个元素       | `target_parent = element.find_all_next("a")`                                                                  | `target_parent = element.below_elements.filter(lambda p: p.tag == 'a')`           |
| 在元素祖先中搜索元素         | `target_parent = element.find_previous("a")` ¹                                                                | `target_parent = element.path.search(lambda p: p.tag == 'a')`                     |
| 在元素祖先中搜索多个元素           | `target_parent = element.find_all_previous("a")` ¹                                                            | `target_parent = element.path.filter(lambda p: p.tag == 'a')`                     |
| 获取上一个兄弟节点                              | `prev_element = element.previous_sibling`                                                                     | `prev_element = element.previous`                                                 |
| 导航到子元素                                          | `children = list(element.children)`                                                                           | `children = element.children`                                                     |
| 获取元素所有后代                               | `children = list(element.descendants)`                                                                        | `children = element.below_elements`                                               |
| 过滤满足条件的元素组        | `group = soup.find('p', 'story').css.filter('a')`                                                             | `group = page.find_all('p', 'story').filter(lambda p: p.tag == 'a')`              |


¹ **注意：** BS4 的 `find_previous`/`find_all_previous` 按文档顺序搜索所有前置元素，而 Scrapling 的 `path` 仅返回祖先（父级链）。二者并非完全等价，但祖先搜索覆盖了最常见用例。

**请记住的关键点**：BeautifulSoup 提供解析后修改与操作页面的功能。Scrapling 更专注于更快地为你的抓取解析页面，之后你可对提取的信息做任何处理。因此，两种工具都可用于 Web 抓取，但其中之一专精于 Web 抓取 :)

### 综合示例

以下是用 BeautifulSoup 与 Scrapling 抓取网页并提取所有链接的简单示例。

**使用 BeautifulSoup：**

```python
import requests
from bs4 import BeautifulSoup

url = 'https://example.com'
response = requests.get(url)
soup = BeautifulSoup(response.text, 'html.parser')

links = soup.find_all('a')
for link in links:
    print(link['href'])
```

**使用 Scrapling：**

```python
from scrapling import Fetcher

url = 'https://example.com'
page = Fetcher.get(url)

links = page.css('a::attr(href)')
for link in links:
    print(link)
```

如你所见，Scrapling 将获取与解析合并为一步，使代码更简洁高效。

!!! abstract "**补充说明：**"

    - **不同解析器**：BeautifulSoup 允许设置解析引擎，其中之一是 `lxml`。Scrapling 不这样做，为性能原因默认使用 `lxml` 库。
    - **元素类型**：BeautifulSoup 中元素为 `Tag` 对象；Scrapling 中为 `Selector` 对象。但二者提供类似的导航与数据提取方法与属性。
    - **错误处理**：两库在未找到元素时均返回 `None`（如 `soup.find()` 或 `page.find()`）。Scrapling 中 `page.css()` 无匹配时返回空 `Selectors` 列表，可用 `page.css('.foo').first` 安全获取第一个匹配或 `None`。访问属性前请检查 `None` 或空结果以避免错误。
    - **文本提取**：Scrapling 通过 `TextHandler` 提供额外文本处理方法，如 `clean()`，可帮助去除多余空白、连续空格或不需要的字符。完整列表请查阅文档。

文档提供更多 Scrapling 功能细节，以及可传入所有方法的完整参数列表。

本指南应能让你从 BeautifulSoup 到 Scrapling 的迁移顺畅直接。祝抓取愉快！
