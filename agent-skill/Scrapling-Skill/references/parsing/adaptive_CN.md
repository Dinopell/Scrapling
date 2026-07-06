# 自适应抓取

自适应抓取（此前称为 automatch）是 Scrapling 最强大的功能之一。它通过智能跟踪和重新定位元素，使抓取器在网站变更后仍能继续工作。

考虑如下结构的页面：
```html
<div class="container">
    <section class="products">
        <article class="product" id="p1">
            <h3>Product 1</h3>
            <p class="description">Description 1</p>
        </article>
        <article class="product" id="p2">
            <h3>Product 2</h3>
            <p class="description">Description 2</p>
        </article>
    </section>
</div>
```
要抓取第一个商品（ID 为 `p1` 的那个），会使用如下选择器：
```python
page.css('#p1')
```
当网站所有者实施如下结构变更时
```html
<div class="new-container">
    <div class="product-wrapper">
        <section class="products">
            <article class="product new-class" data-id="p1">
                <div class="product-info">
                    <h3>Product 1</h3>
                    <p class="new-description">Description 1</p>
                </div>
            </article>
            <article class="product new-class" data-id="p2">
                <div class="product-info">
                    <h3>Product 2</h3>
                    <p class="new-description">Description 2</p>
                </div>
            </article>
        </section>
    </div>
</div>
```
该选择器将不再有效，你的代码需要维护。这正是 Scrapling 的 `adaptive` 功能发挥作用的地方。

使用 Scrapling 时，你可以在首次选择元素时启用 `adaptive` 功能；下次用相同方式选择该元素而元素不存在时，Scrapling 会记住其属性，并在网站上搜索与该元素相似度最高的元素。

```python
from scrapling import Selector, Fetcher
# Before the change
page = Selector(page_source, adaptive=True, url='example.com')
# or
Fetcher.adaptive = True
page = Fetcher.get('https://example.com')
# then
element = page.css('#p1', auto_save=True)
if not element:  # One day website changes?
    element = page.css('#p1', adaptive=True)  # Scrapling still finds it!
# the rest of your code...
```
它适用于所有选择方法，不仅限于 CSS/XPath 选择。

## 真实场景示例
本示例使用 [The Web Archive](https://archive.org/) 的 [Wayback Machine](https://web.archive.org/) 演示跨不同网站版本的自适应抓取。[2010 年的 StackOverflow 网站快照](https://web.archive.org/web/20100102003420/http://stackoverflow.com/) 与当前设计对比，说明自适应功能可以用相同选择器提取相同按钮。

要从旧设计中提取 Questions 按钮，可使用类似 `#hmenus > div:nth-child(1) > ul > li:nth-child(1) > a` 的选择器（此选择器由 Chrome 生成）。

在两个版本中测试相同选择器：
```python
from scrapling import Fetcher
selector = '#hmenus > div:nth-child(1) > ul > li:nth-child(1) > a'
old_url = "https://web.archive.org/web/20100102003420/http://stackoverflow.com/"
new_url = "https://stackoverflow.com/"
Fetcher.configure(adaptive = True, adaptive_domain='stackoverflow.com')

page = Fetcher.get(old_url, timeout=30)
element1 = page.css(selector, auto_save=True)[0]

# Same selector but used in the updated website
page = Fetcher.get(new_url)
element2 = page.css(selector, adaptive=True)[0]

if element1.text == element2.text:
...    print('Scrapling found the same element in the old and new designs!')
```
此处使用 `adaptive_domain` 参数，因为 Scrapling 将 `archive.org` 和 `stackoverflow.com` 视为两个不同域名，会隔离各自的 `adaptive` 数据。传入 `adaptive_domain` 可告诉 Scrapling 在自适应数据存储时将它们视为同一网站。

在两次请求使用相同 URL 的典型场景中，不需要 `adaptive_domain` 参数。自适应逻辑在 `Selector` 和 `Fetcher` 类中工作方式相同。

**注意：** 创建 `adaptive_domain` 参数的主要原因是：网站在变更设计/结构的同时更改了 URL。此时可用它继续在新 URL 上使用先前存储的自适应数据。否则 Scrapling 会将其视为新网站并丢弃旧数据。

## 自适应抓取功能的工作原理
自适应抓取分两个阶段工作：

1. **保存阶段**：存储元素的唯一属性
2. **匹配阶段**：之后查找具有相似属性的元素

通过任意方法选择元素后，库可以在下次抓取网站时找到它，即使网站经历了结构/设计变更。

总体逻辑如下：

  1. Scrapling 保存该元素的唯一属性（方法见下文）。
  2. Scrapling 使用配置的数据库（默认 SQLite）保存每个元素的唯一属性。
  3. 由于网站所有者可能更改或移除元素的任何内容，元素的任何部分都不能用作数据库的唯一标识符。存储系统依赖两件事：
     1. 当前网站的域名。使用 `Selector` 类时在初始化时传入；使用 fetcher 时从 URL 自动获取。
     2. 用于从数据库查询该元素属性的 `identifier`。identifier 并非总是需要手动设置（见下文）。

     两者将用于后续从数据库检索元素的唯一属性。

  4. 之后，当网站结构变更时，启用 `adaptive` 会使 Scrapling 检索元素的唯一属性，并将页面上所有元素与之匹配。根据与目标元素的相似度计算得分。比较时会考虑一切因素。
  5. 返回与目标元素相似度得分最高的元素。

### 唯一属性
Scrapling 依赖的唯一属性包括：

- 元素标签名、文本、属性（名称和值）、兄弟元素（仅标签名）和路径（仅标签名）。
- 元素父级的标签名、属性（名称和值）和文本。

元素之间的比较并非精确匹配，而是基于这些值的相似程度。一切都会被考虑，包括值的顺序（例如 class 名称的书写顺序）。

## 如何使用 adaptive 功能
adaptive 功能可应用于任意已找到的元素，并作为 CSS/XPath 选择方法的参数添加。

首先，在初始化 [Selector](main_classes_CN.md#selector) 类时传入 `adaptive=True` 启用 `adaptive` 功能，或在使用的 fetcher 上启用。

示例：
```python
from scrapling import Selector, Fetcher
page = Selector(html_doc, adaptive=True)
# OR
Fetcher.adaptive = True
page = Fetcher.get('https://example.com')
```
使用 [Selector](main_classes_CN.md#selector) 类时，通过 `url` 参数传入网站 URL，以便 Scrapling 按域名区分各元素保存的属性。

若未传入 URL，保存元素唯一属性时 URL 字段将使用 `default`。仅在使用相同 identifier 抓取不同网站且未传入 URL 参数时会有问题。保存过程会覆盖先前数据，`adaptive` 功能仅使用最新保存的属性。

`storage` 和 `storage_args` 参数控制数据库连接；默认使用库提供的 SQLite 类。

使用 `adaptive` 功能有两种主要方式：

### CSS/XPath 选择方式
首先，在页面上存在目标元素时，选择时使用 `auto_save` 参数：
```python
element = page.css('#p1', auto_save=True)
```
当元素不再存在时，使用相同选择器并传入 `adaptive` 参数，让库查找它：
```python
element = page.css('#p1', adaptive=True)
```
使用 `css`/`xpath` 方法时，identifier 会自动设为传入方法的选择器字符串。

此外，对于所有这些方法，你可以传入 `identifier` 参数自行设置。这在某些情况下很有用，也可配合 `auto_save` 参数保存属性。

### 手动方式
可以在 `adaptive` 功能中手动保存、检索和重新定位元素。这允许重新定位通过任意方法找到的任何元素。

通过文本获取元素的示例：
```python
element = page.find_by_text('Tipping the Velvet', first_match=True)
```
使用 `save` 方法保存其唯一属性。必须手动设置 identifier（使用有意义的 identifier）：
```python
page.save(element, 'my_special_element')
```
之后，使用 `adaptive` 在页面内检索并重新定位元素：
```python
>>> element_dict = page.retrieve('my_special_element')
>>> page.relocate(element_dict, selector_type=True)
[<data='<a href="catalogue/tipping-the-velvet_99...' parent='<h3><a href="catalogue/tipping-the-velve...'>]
>>> page.relocate(element_dict, selector_type=True).css('::text').getall()
['Tipping the Velvet']
```
此处使用了 `retrieve` 和 `relocate` 方法。

若要保留为 `lxml.etree` 对象，省略 `selector_type` 参数：
```python
>>> page.relocate(element_dict)
[<Element a at 0x105a2a7b0>]
```

## 故障排除

### 未找到匹配
```python
# 1. Check if data was saved
element_data = page.retrieve('identifier')
if not element_data:
    print("No data saved for this identifier")

# 2. Try with different identifier
products = page.css('.product', adaptive=True, identifier='old_selector')

# 3. Save again with new identifier
products = page.css('.new-product', auto_save=True, identifier='new_identifier')
```

### 匹配到错误元素
```python
# Use more specific selectors
products = page.css('.product-list .product', auto_save=True)

# Or save with more context
product = page.find_by_text('Product Name').parent
page.save(product, 'specific_product')
```

## 已知问题
在 `adaptive` 保存过程中，仅保存选择结果中第一个元素的唯一属性。因此，若你使用的选择器在页面其他位置选中了不同元素，后续重新定位时 `adaptive` 只会返回第一个元素。这不包括组合 CSS 选择器（使用逗号组合多个选择器的情况），因为此类选择器会被拆分并分别单独执行。
