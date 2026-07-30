> **说明**：本文件为英文原文（`3d-object.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 3D 物体

构建一个用户可以从各个角度查看并可下载的 3D 物体，
使用 three.js 构建，并在 three_d_stage 起始模板中呈现。

首先调用 copy_starter_component，参数为 kind: "three_d_stage.js" —
它就是整个查看器：工作室照明、地面阴影、轨道控制、
自动取景相机，以及一个可将物体导出为
OBJ + MTL 或 GLB 的工具栏。阅读复制文件的使用说明块，
并严格按照其页面骨架操作。你只需编写模型构建模块脚本。

将页面构建为纯 HTML — 一个带有普通 <script>
标签的 .html 文件 — 即使项目的其他设计是 .dc.html 设计
组件：DC 将脚本限制在 <helmet> 中，这与舞台的
挂载存在竞争，无法承载此骨架。

仅在 <head> 中通过此确切固定版本的 import map 加载 three.js，
在任何模块脚本之前。不要更改版本、URL 或哈希值，不要
添加 three.js 的其他副本，也不要导入除列出的三个之外的
附加组件 — 该映射是有意设计的封闭集合，因此其他任何内容
都会解析失败，而不是加载未经验证的代码：

<script type="importmap">
{
  "imports": {
    "three": "https://unpkg.com/three@0.184.0/build/three.module.js",
    "three/addons/controls/OrbitControls.js": "https://unpkg.com/three@0.184.0/examples/jsm/controls/OrbitControls.js",
    "three/addons/exporters/OBJExporter.js": "https://unpkg.com/three@0.184.0/examples/jsm/exporters/OBJExporter.js",
    "three/addons/exporters/GLTFExporter.js": "https://unpkg.com/three@0.184.0/examples/jsm/exporters/GLTFExporter.js"
  },
  "integrity": {
    "https://unpkg.com/three@0.184.0/build/three.module.js": "sha384-8FCZ1eVO6it4+pbec2aDtnTrwjWXZLJRC+MAGCIPDgsYnUrl/E0A2YlF8ioMKI/J",
    "https://unpkg.com/three@0.184.0/build/three.core.js": "sha384-dw2ooPewaEIrAgl6oFDBmmBWCE9oW9LxRGcfwZ0hLvEprzo202wXl7vCYHRlSnOT",
    "https://unpkg.com/three@0.184.0/examples/jsm/controls/OrbitControls.js": "sha384-4rziNxOBZKQ69i+w+f89KJ55TCYquwchVbByQwmaOeIOXdOU2PLDn3kOfXHwIJC9",
    "https://unpkg.com/three@0.184.0/examples/jsm/exporters/OBJExporter.js": "sha384-nbwtoZENJD3Vq+ACK0CuGQdPMuDWHkamC2KJD70EV5nfg6jQjfppKOea07YJN+N3",
    "https://unpkg.com/three@0.184.0/examples/jsm/exporters/GLTFExporter.js": "sha384-VofkvpG6HERhFCYbsUOHeNXBCqID2nfqkQqnVzE1jc/oPcz+qJ13ADdXH08hE+cQ"
  }
}
</script>

以编程方式构建模型，作为由命名部件组成的 THREE.Group：
- 在使用原始 BufferGeometry 之前，先用基本几何体组合（BoxGeometry、CylinderGeometry、SphereGeometry、
  TorusGeometry、LatheGeometry、带 Shape 的 ExtrudeGeometry）；真实物体可分解为比你想象中
  多得多的基本几何体。
- 为每个网格和每个材质命名（"hull"、"walnut"、"brass"）— 这些
  名称成为导出 OBJ 中的 o / usemtl 条目和 GLB 中的节点
  名称，这正是让下载文件可在 Blender 中使用的关键。
- 使用 MeshStandardMaterial 和少量精选调色板（3-5 种材质，
  在部件间共享）；有意设置 roughness/metalness。纹理
  无法在 OBJ 导出中保留 — 优先使用几何体和材质颜色而非
  纹理细节。
- 以真实世界米为单位建模，y 轴向上，原点居中，底部
  停靠在最低 y 处。有意偏移共面多边形约 0.001，以免
  发生 z-fighting。
- 曲面需要足够的分段以在全屏时看起来光滑
  （特征曲面 32+ 径向分段），但不要对没有人
  会看到的部分进行细分。

舞台的工具栏为用户提供 OBJ + MTL（通用，几何体 +
每材质颜色）和 GLB（现代交换格式 — 保留
部件层次结构和 PBR 材质；可干净地导入 Blender、Maya、
Cinema 4D、Unity、Unreal）。这两种是提供的导出格式 —
当用户要求其他格式（FBX、USDZ、STEP）时，直接说明
舞台导出 OBJ + MTL 和 GLB。

用截图迭代：舞台保持其最后一帧可读，因此
普通截图工具可以捕获实时画布 — 无需额外步骤。
编辑模块文件后，在截图前用 show_html 重新加载
（iframe 会缓存已加载的模块）。从默认取景角度
观察物体，优化轮廓、比例和材质
分离 — 轮廓承载了物体本身。
