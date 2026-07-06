# 从 BeautifulSoup 迁移到 Scrapling

BeautifulSoup 与 Scrapling 的 API 对照。Scrapling 更快，提供等价的解析能力，并增加了抓取与现代网页处理功能。

部分 BeautifulSoup 快捷写法在 Scrapling 中没有直接对应。Scrapling 有意省略这些快捷方式以保持性能。


| 任务                                                            | BeautifulSoup 代码                                                                                            | Scrapling 代码                                                                    |
|-----------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------|
| 解析器导入                                                      | `from bs4 import BeautifulSoup`                                                                               | `from scrapling.parser import Selector`                                           |
| 从字符串解析 HTML                                               | `soup = BeautifulSoup(html, 'html.parser')`                                                                   | `page = Selector(html)`                                                           |
| 查找单个元素                                                    | `element = soup.find('div', class_='example')`                                                                | `element = page.find('div', class_='example')`                                    |
| 查找多个元素                                                    | `elements = soup.find_all('div', class_='example')`                                                           | `elements = page.find_all('div', class_='example')`                               |
| 查找单个元素（示例 2）                                          | `element = soup.find('div', attrs={"class": "example"})`                                                      | `element = page.find('div', {"class": "example"})`                                |
| 查找单个元素（示例 3）                                          | `element = soup.find(re.compile("^b"))`                                                                       | `element = page.find(re.compile("^b"))`<br/>`element = page.find_by_regex(r"^b")` |
| 查找单个元素（示例 4）                                          | `element = soup.find(lambda e: len(list(e.children)) > 0)`                                                    | `element = page.find(lambda e: len(e.children) > 0)`                              |
| 查找单个元素（示例 5）                                          | `element = soup.find(["a", "b"])`                                                                             | `element = page.find(["a", "b"])`                                                 |
| 按文本内容查找元素                                              | `element = soup.find(text="some text")`                                                                       | `element = page.find_by_text("some text", partial=False)`                         |
| 用 CSS 选择器查找第一个匹配元素                                 | `elements = soup.select_one('div.example')`                                                                   | `elements = page.css('div.example').first`                                        |
| 用 CSS 选择器查找所有匹配元素                                   | `elements = soup.select('div.example')`                                                                       | `elements = page.css('div.example')`                                              |
| 获取页面/元素源码的格式化版本                                   | `prettified = soup.prettify()`                                                                                | `prettified = page.prettify()`                                                    |
| 获取页面/元素源码的非格式化版本                                 | `source = str(soup)`                                                                                          | `source = page.html_content`                                                      |
| 获取元素标签名                                                  | `name = element.name`                                                                                         | `name = element.tag`                                                              |
| 提取元素文本内容                                                | `string = element.string`                                                                                     | `string = element.text`                                                           |
| 提取文档或标签下全部文本                                        | `text = soup.get_text(strip=True)`                                                                            | `text = page.get_all_text(strip=True)`                                            |
| 访问属性字典                                                    | `attrs = element.attrs`                                                                                       | `attrs = element.attrib`                                                          |
| 提取属性                                                        | `attr = element['href']`                                                                                      | `attr = element['href']`                                                          |
| 导航到父元素                                                    | `parent = element.parent`                                                                                     | `parent = element.parent`                                                         |
| 获取元素的所有父级                                              | `parents = list(element.parents)`                                                                             | `parents = list(element.iterancestors())`                                         |
| 在元素的父级中搜索元素                                          | `target_parent = element.find_parent("a")`                                                                    | `target_parent = element.find_ancestor(lambda p: p.tag == 'a')`                   |
| 获取元素的所有兄弟节点                                          | N/A                                                                                                           | `siblings = element.siblings`                                                     |
| 获取元素的下一个兄弟节点                                        | `next_element = element.next_sibling`                                                                         | `next_element = element.next`                                                     |
| 在元素的兄弟节点中搜索元素                                      | `target_sibling = element.find_next_sibling("a")`<br/>`target_sibling = element.find_previous_sibling("a")`   | `target_sibling = element.siblings.search(lambda s: s.tag == 'a')`                |
| 在元素的兄弟节点中搜索多个元素                                  | `target_sibling = element.find_next_siblings("a")`<br/>`target_sibling = element.find_previous_siblings("a")` | `target_sibling = element.siblings.filter(lambda s: s.tag == 'a')`                |
| 在元素的后续元素中搜索元素                                      | `target_parent = element.find_next("a")`                                                                      | `target_parent = element.below_elements.search(lambda p: p.tag == 'a')`           |
| 在元素的后续元素中搜索多个元素                                  | `target_parent = element.find_all_next("a")`                                                                  | `target_parent = element.below_elements.filter(lambda p: p.tag == 'a')`           |
| 在元素的祖先中搜索元素                                          | `target_parent = element.find_previous("a")` ¹                                                                | `target_parent = element.path.search(lambda p: p.tag == 'a')`                     |
| 在元素的祖先中搜索多个元素                                      | `target_parent = element.find_all_previous("a")` ¹                                                            | `target_parent = element.path.filter(lambda p: p.tag == 'a')`                     |
| 获取元素的上一个兄弟节点                                        | `prev_element = element.previous_sibling`                                                                     | `prev_element = element.previous`                                                 |
| 导航到子元素                                                    | `children = list(element.children)`                                                                           | `children = element.children`                                                     |
| 获取元素的所有后代                                              | `children = list(element.descendants)`                                                                        | `children = element.below_elements`                                               |
| 过滤满足条件的一组元素                                          | `group = soup.find('p', 'story').css.filter('a')`                                                             | `group = page.find_all('p', 'story').filter(lambda p: p.tag == 'a')`              |


¹ **说明：** BS4 的 `find_previous`/`find_all_previous` 会按文档顺序搜索所有前置元素，而 Scrapling 的 `path` 仅返回祖先（父级链）。二者并非完全等价，但祖先搜索已覆盖最常见用法。

BeautifulSoup 支持修改/操作已解析的 DOM。Scrapling 不支持——它是只读的，并针对数据提取做了优化。

### 完整示例：提取链接

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

Scrapling 将抓取与解析合并为一步。

**说明：**

- **解析器**：BeautifulSoup 支持多种解析引擎。Scrapling 始终使用 `lxml` 以获得性能。
- **元素类型**：BeautifulSoup 元素为 `Tag` 对象；Scrapling 元素为 `Selector` 对象。二者都提供类似的导航与提取方法。
- **错误处理**：两库在找不到元素时均返回 `None`（如 `soup.find()` 或 `page.find()`）。`page.css()` 在无匹配时返回空的 `Selectors` 列表。使用 `page.css('.foo').first` 可安全获取第一个匹配或 `None`。
- **文本提取**：Scrapling 的 `TextHandler` 提供额外文本处理方法，如 `clean()` 用于去除多余空白、连续空格或 unwanted 字符。
