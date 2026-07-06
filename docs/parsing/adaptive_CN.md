# 自适应爬取

!!! success "前置条件"

    1. 你已完成或阅读过[查询元素](../parsing/selection_CN.md)页面，了解如何从 [Selector](../parsing/main_classes_CN.md#selector) 对象中查找/提取元素。
    2. 你已完成或阅读过[核心类](../parsing/main_classes_CN.md)页面，了解 [Selector](../parsing/main_classes_CN.md#selector) 类。

自适应爬取（此前称为 automatch）是 Scrapling 最强大的特性之一。它通过智能跟踪和重新定位元素，让你的爬虫在网站改版后仍能继续工作。

假设你正在爬取结构如下的一页：
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
你想爬取第一个商品，即 ID 为 `p1` 的那个。你可能会写如下选择器
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
选择器将失效，代码需要维护。这正是 Scrapling `adaptive` 特性的用武之地。

使用 Scrapling，你可以在首次选择元素时启用 `adaptive` 特性；下次再选该元素而它不存在时，Scrapling 会记住其属性，在网站上搜索与该元素相似度最高的元素——且无需 AI :)

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
下面展示一个使用示例，然后深入说明用法和细节。注意它与所有选择方法配合，不仅限于 CSS/XPATH。

## 真实场景
以真实网站为例，用某个抓取器获取源码。要演示此特性，需要找一个即将改版/改结构的网站，复制源码，再等待网站变化。这几乎不可能，除非我认识站主，但那就成了 staged 测试，哈哈。

为解决此问题，我使用 [The Web Archive](https://archive.org/) 的 [Wayback Machine](https://web.archive.org/)。这是 [2010 年 StackOverFlow 网站](https://web.archive.org/web/20100102003420/http://stackoverflow.com/) 的存档；够老了吧？</br>看看自适应特性能否用同一选择器，从 2010 年旧设计和当前设计中提取同一按钮 :)

若要从旧设计提取 Questions 按钮，可用选择器：`#hmenus > div:nth-child(1) > ul > li:nth-child(1) > a`。此选择器非常具体，由 Google Chrome 生成。


在两个版本中测试同一选择器
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
    print('Scrapling found the same element in the old and new designs!')  # Spoiler alert: it does!
```
注意我引入了新参数 `adaptive_domain`。对 Scrapling 而言，这是两个不同域名（`archive.org` 和 `stackoverflow.com`），因此会隔离各自的 `adaptive` 数据。要告知 Scrapling 它们是同一网站，保存两处的 `adaptive` 数据时必须传入要使用的自定义域名，避免被隔离。

真实场景中代码相同，只是两次请求用同一 URL，因此不需要 `adaptive_domain`。这是我能给出的最接近真实案例的示例，希望没有造成困惑 :)

因此，上面两个示例分别使用了 `Selector` 和 `Fetcher` 类，说明自适应逻辑一致。

!!! info

    创建 `adaptive_domain` 参数的主要原因是：网站改版/改结构时 URL 也可能变化。此时可用它继续对新 URL 使用先前存储的自适应数据。否则 Scrapling 会视为新网站并丢弃旧数据。

## 自适应爬取特性如何工作
自适应爬取分两个阶段：

1. **保存阶段**：存储元素的唯一属性
2. **匹配阶段**：之后查找具有相似属性的元素

假设你通过任意方法选中了元素，希望下次爬取该站时即使结构/设计变化，库仍能帮你找到它。

尽量少用技术细节，一般逻辑如下：

  1. 你以下面将展示的方式告知 Scrapling 保存该元素的唯一属性。
  2. Scrapling 使用配置的数据库（默认 SQLite）保存每个元素的唯一属性。
  3. 由于网站所有者可能修改或删除元素的一切，无法用元素本身作为数据库唯一标识。为此，存储系统依赖两件事：
     1. 当前网站的域名。使用 `Selector` 类时在初始化传入；使用抓取器时从 URL 自动获取。
     2. 用于从数据库查询该元素属性的 `identifier`。不必总是自己设置，稍后讨论。

     二者稍后用于从数据库检索元素的唯一属性。

  4. 之后网站结构变化，你启用 `adaptive` 让 Scrapling 查找元素。Scrapling 检索唯一属性，将页面上所有元素与已有属性比较，按相似度计分，比较时会考虑方方面面，稍后说明。
  5. 返回与目标元素相似度最高的元素。

### 唯一属性
你可能会问，谈到元素属性被删除或修改时，我们说的唯一属性是什么。

对 Scrapling 而言，依赖的唯一要素包括：

- 元素标签名、文本、属性（名和值）、兄弟元素（仅标签名）、路径（仅标签名）。
- 元素父元素的标签名、属性（名和值）和文本。

需理解元素之间的比较并非精确匹配，而是这些值的相似程度。一切都会被考虑，甚至值的顺序，例如元素 class 名以前的顺序与现在的顺序。

## 如何使用自适应特性
自适应特性可应用于任意已找到的元素，作为 CSS/XPath 选择方法的参数，如上所示，稍后详述。

首先，必须在初始化 [Selector](main_classes_CN.md#selector) 时传入 `adaptive=True`，或在所用抓取器上启用，如下所示。

示例：
```python
from scrapling import Selector, Fetcher
page = Selector(html_doc, adaptive=True)
# OR
Fetcher.adaptive = True
page = Fetcher.get('https://example.com')
```
若使用 [Selector](main_classes_CN.md#selector) 类，需用 `url` 参数传入网站 URL，以便 Scrapling 按域名区分各元素保存的属性。

若未传 URL，保存元素唯一属性时 URL 字段将使用 `default`。仅当你之后对另一网站使用相同 identifier 且初始化时未传 URL 时才有问题。保存会覆盖先前数据，`adaptive` 特性只使用最近保存的属性。

此外还有 `storage` 和 `storage_args`，用于连接数据库；默认使用库提供的 SQLite 类。除非你打算自定义存储系统，否则不必关心，我们将在[开发章节的单独页面](../development/adaptive_storage_system_CN.md)说明。

全局启用 `adaptive` 后，有两种主要用法。

### CSS/XPath 选择方式
如上例，首先对页面上存在的元素使用 `auto_save` 参数选择，例如：
```python
element = page.css('#p1', auto_save=True)
```
当元素不存在时，可用同一选择器加 `adaptive` 参数，库会为你找到它
```python
element = page.css('#p1', adaptive=True)
```
很简单吧？

底层发生了很多事。记得之前说的 identifier 吗？在 `css`/`xpath` 方法中，identifier 自动设为你传入的选择器，以简化使用 :)

此外，这些方法都可传 `identifier` 自行设置，在某些场景有用，也可与 `auto_save` 一起保存属性。

### 手动方式
手动保存和检索元素再重新定位，均在 `adaptive` 特性内完成，如下所示。允许用任意方法或选择重新定位任意元素！

假设通过文本得到元素：
```python
element = page.find_by_text('Tipping the Velvet', first_match=True)
```
可用 `save` 方法保存其唯一属性，但必须自行设置 identifier。本例用 `my_special_element`，实际代码中最好用有意义的 identifier，就像变量命名一样 :)
```python
page.save(element, 'my_special_element')
```
之后要用 `adaptive` 检索并在页面内重新定位时：
```python
>>> element_dict = page.retrieve('my_special_element')
>>> page.relocate(element_dict, selector_type=True)
[<data='<a href="catalogue/tipping-the-velvet_99...' parent='<h3><a href="catalogue/tipping-the-velve...'>]
>>> page.relocate(element_dict, selector_type=True).css('::text').getall()
['Tipping the Velvet']
```
因此使用 `retrieve` 和 `relocate` 方法。

若要保持为 `lxml.etree` 对象，不传 `selector_type` 参数
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
在 `adaptive` 保存过程中，仅保存选择结果中第一个元素的唯一属性。因此若选择器在页面其他位置选中不同元素，重新定位时 `adaptive` 只会返回第一个元素。这不包括组合 CSS 选择器（例如用逗号组合多个选择器），因为此类选择器会被拆分并分别执行。

## 结语
详细说明此特性而不复杂化颇具挑战。若仍有不清楚之处，可前往 [discussions 区](https://github.com/D4Vinci/Scrapling/discussions)，我会尽快回复，或 Discord 服务器，或私下联系交流 :)
