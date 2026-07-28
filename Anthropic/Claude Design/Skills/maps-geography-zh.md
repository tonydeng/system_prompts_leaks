> **说明**：本文件为英文原文（`maps-geography.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以英文原文为准。

# 地图与地理

地理地图是数据问题，不是绘画：永远不要徒手绘制国家轮廓、海岸线或街道布局——手绘地理内容总是不准确的，用户也会注意到。加载真实几何数据并进行渲染。

将每个地图页面构建为纯 HTML——一个带有普通 <script> 标签的 .html 文件，永远不要使用 .dc.html Design Component，即使项目中其他所有设计都是这种格式：DC 将脚本限制在 <helmet> 中，其挂载时机会与地图容器产生竞态条件——这正是数据可视化和 3D 技能所面临的同样问题。

对于演示文稿、文档、图形和动画——任何静态或导出的内容——使用 d3-geo 渲染 TopoJSON 几何数据：获取 https://cdn.jsdelivr.net/npm/world-atlas@2.0.2/countries-110m.json（Natural Earth 数据，公共领域；URL 已锁定版本——请原样使用），用 topojson.feature(topology, topology.objects.countries) 转换，并使用 d3.geoPath() 在为任务选择的投影下绘制（全球视图用 d3.geoNaturalEarth1；区域缩放用 d3.geoMercator().fitSize(...)）。d3-geo 包含在下面的 d3 包中。仅在 <head> 中通过这些精确的、锁定版本并经过哈希验证的标签加载库。这些标签在被篡改时会拒绝加载；你添加的任何其他脚本都会在未验证的情况下加载——因此不要更改版本、URL 或哈希，也不要从 CDN 添加任何其他内容：

<script src="https://unpkg.com/d3@7.9.0/dist/d3.min.js" integrity="sha384-CjloA8y00+1SDAUkjs099PVfnY2KmDC2BZnws9kh8D/lX1s46w6EPhpXdqMfjK6i" crossorigin="anonymous"></script>
<script src="https://unpkg.com/topojson-client@3.1.0/dist/topojson-client.min.js" integrity="sha384-Ukv1p/xTma6P4/2bY5KzWBw+ydSpXmhCMtyciIQVDJ1RmOxtCYNMF1uXT9T63H67" crossorigin="anonymous"></script>

来自 d3 的内联 SVG 也可以干净地导出为 PNG 和 PDF，而实时地图瓦片则不能——因此导出的交付物始终使用 d3 几何数据，绝不使用嵌入的瓦片地图。

对于街道级交互式地图——原型、网站、任何用户会平移和缩放的内容——使用 Leaflet 配合 OpenStreetMap 瓦片，仅通过这些精确的标签加载（样式表是必需的：没有 leaflet.css，瓦片会渲染错乱）：

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" integrity="sha384-sHL9NAb7lN7rfvG5lfHpm643Xkcjzp4jFvuavGOndn6pjVqS6ny56CAt3nsEVT4H" crossorigin="anonymous">
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js" integrity="sha384-cxOPjt7s7Iz04uaHJceBmS+qpjv2JkIHNVcuOrM+YHwZOmJGBXI00mdUXEq65HTH" crossorigin="anonymous"></script>

使用 L.map(...) 和 L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', { attribution: '© OpenStreetMap contributors' }) 创建地图。归属字符串是 OpenStreetMap 的许可证要求——永远不要省略。
