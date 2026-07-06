# 性能基准测试

Scrapling 不仅功能强大——而且速度极快。以下基准测试将 Scrapling 解析器与最新版本的其他流行库进行对比。

### 文本提取速度测试（5000 个嵌套元素）

| # |      库      | 时间 (ms) | 相对 Scrapling | 
|---|:-----------------:|:---------:|:------------:|
| 1 |     Scrapling     |   2.02    |     1.0x     |
| 2 |   Parsel/Scrapy   |   2.04    |     1.01     |
| 3 |     Raw Lxml      |   2.54    |    1.257     |
| 4 |      PyQuery      |   24.17   |     ~12x     |
| 5 |    Selectolax     |   82.63   |     ~41x     |
| 6 |  MechanicalSoup   |  1549.71  |   ~767.1x    |
| 7 |   BS4 with Lxml   |  1584.31  |   ~784.3x    |
| 8 | BS4 with html5lib |  3391.91  |   ~1679.1x   |


### 元素相似度与文本搜索性能

Scrapling 的自适应元素查找能力显著优于替代方案：

| 库     | 时间 (ms) | 相对 Scrapling |
|-------------|:---------:|:------------:|
| Scrapling   |   2.39    |     1.0x     |
| AutoScraper |   12.45   |    5.209x    |

> 所有基准均为 100+ 次运行的平均值。方法学见 [benchmarks.py](https://github.com/D4Vinci/Scrapling/blob/main/benchmarks.py)。
