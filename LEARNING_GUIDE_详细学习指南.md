# COBE 代码仓库完整学习指南

## 📚 项目基本信息

**项目名称**: COBE - WebGL 地球仪可视化库  
**GitHub**: shuding/cobe  
**特点**: 高性能 (~5KB)、零依赖、支持交互  
**许可**: MIT  

---

## 🎯 快速导览 - 30秒了解项目

```
COBE 是什么？
  → 一个在浏览器中渲染 3D 地球的库
  → 用户可以放置标记点和连接线
  → 支持旋转、缩放、交互

它有什么独特之处？
  → 文件大小只有 5KB（极致压缩）
  → 完全用 WebGL 从头实现
  → 支持 CSS 锚点定位（可以绑定 DOM 元素）

构成部分？
  → 核心库（src/）：WebGL 渲染引擎
  → 演示网站（website/）：Next.js powered
  → build 脚本：编译着色器
```

---

## 🏗️ 第一步：全局架构 (15分钟) ✓ 已完成

### 目录结构分析

```
cobe/
├── src/                          # 💎 核心库代码
│   ├── index.js                  # 🌟 主入口 - createGlobe() 导出在这
│   ├── webgl.js                  # WebGL 辅助函数（创建着色器、程序）
│   ├── globe.vert.glslx          # 地球顶点着色器
│   ├── globe.frag.glslx          # 地球片段着色器  
│   ├── marker.glslx              # 标记点着色器
│   ├── arc.glslx                 # 弧线着色器
│   ├── anchor.js                 # CSS 锚点定位实现
│   ├── texture.js                # 纹理处理
│   ├── index.d.ts                # TypeScript 类型定义
│   └── texture.png               # 地球地表纹理图
│
├── website/                       # 演示网站
│   ├── app/
│   │   ├── page.tsx              # 主页
│   │   ├── showcases/            # 展示案例
│   │   ├── ip/                   # IP 地球演示
│   │   └── components/           # 交互组件
│   └── package.json
│
├── scripts/
│   └── build.js                  # 🔨 构建脚本 - 编译着色器
│
├── package.json                  # 项目配置
└── README.md                      # 文档
```

### 模块职责清单

| 模块 | 文件 | 职责 | 优先级 |
|------|------|------|--------|
| 入口 | `index.js` | 导出 `createGlobe()` / 协调所有子系统 | 🔴 高 |
| WebGL | `webgl.js` | 着色器编译、程序链接、辅助函数 | 🔴 高 |
| 着色器 | `*.glslx` | GPU 计算 - 顶点变换、颜色计算 | 🔴 高 |
| CSS 锚定 | `anchor.js` | 将标记点绑定到 DOM 元素 | 🟡 中 |
| 纹理 | `texture.js` | 地球表面贴图管理 | 🟡 中 |
| 构建 | `build.js` | 编译项目、内联着色器代码 | 🟡 中 |
| 网站 | `website/` | 演示、测试、文档交互 | 🟢 低 |

---

## 🔗 第二步：核心业务流程 (30分钟)

### 2.1 初始化流程（从用户代码开始）

```
import createGlobe from 'cobe'

let canvas = document.getElementById('cobe')
const globe = createGlobe(canvas, {
  devicePixelRatio: 2,
  width: 1000,
  height: 1000,
  phi: 0,
  theta: 0,
  markers: [...],
  arcs: [...],
  onRender: (state) => { state.phi += 0.01 }
})
```

**内部初始化** (位置: `src/index.js` 行 35-120):
1. 创建 WebGL 上下文 (webgl2 优先降级 → webgl)
2. 设置 Canvas 分辨率（支持 DPR）
3. 编译 3 对着色器程序（地球/标记/弧线）
4. 加载地球纹理图片
5. 设置顶点缓冲区对象(VBO)
6. 启动渲染循环 (requestAnimationFrame)
7. 返回 API: `{ destroy(), update() }`

### 2.2 坐标系统：GPS → 3D

**关键函数** (位置: `src/index.js` 行 20-28):

```javascript
function latLonTo3D([lat, lon]) {
  const latRad = (lat * PI) / 180      // 纬度 → 弧度
  const lonRad = (lon * PI) / 180 - PI // 经度 → 弧度
  const cosLat = cos(latRad)
  
  return [
    -cosLat * cos(lonRad),  // x 坐标
    sin(latRad),            // y 坐标(高度轴)
    cosLat * sin(lonRad)    // z 坐标
  ]
}
```

**使用场景**：
- 标记点 (37.7595°N, 122.4367°W) 旧金山
  → 转换为 3D 球面坐标
- 弧线中间点也通过此转换

### 2.3 每帧渲染循环

```javascript
function render() {
  requestAnimationFrame(render)  // 持续循环（60fps）
  
  // 1. 调用用户回调，获取新角度
  const state = {}
  opts.onRender(state)  // 用户可以修改 state.phi/theta
  
  // 2. 更新旋转角度
  if (state.phi !== undefined) phi = state.phi
  if (state.theta !== undefined) theta = state.theta
  
  // 3. 清除画布
  gl.clearColor(0, 0, 0, opts.opacity)
  gl.clear(gl.COLOR_BUFFER_BIT)
  
  // 4. 绘制三个对象（顺序很重要）
  drawGlobe(phi, theta, globeProgram)     // 地球在底层
  drawMarkers(markers, markerProgram)     // 标记在中层
  drawArcs(arcs, arcProgram)              // 弧线在顶层
}
```

### 2.4 三类对象详解

#### 🌍 地球 (Globe)
- **着色器**: `globe.vert.glslx` + `globe.frag.glslx`
- **输入**: 旋转矩阵(phi,theta) + 地表纹理
- **输出**: 旋转后的球体
- **优化**: 高效纹理采样(mapSamples 参数可调)

#### 📍 标记点 (Marker)  
- **着色器**: `marker.glslx`
- **输入**: 位置[lat,lon] + 大小 + 颜色
- **输出**: 小四边形 + 发光效果
- **特殊**: 支持 CSS 锚定 + 自定义颜色

#### 🔗 弧线 (Arc)
- **着色器**: `arc.glslx`
- **输入**: 起点[lat1,lon1] + 终点[lat2,lon2] + 颜色
- **输出**: 大圆弧（最短路径）
- **特殊**: 支持高度和宽度配置

---

## 🧠 第三步：设计思路 (1小时)

### 3.1 为什么这样设计？

#### ❓ 为什么用 GLSLX 而不是原生 GLSL？

| 特性 | GLSL | GLSLX |
|------|------|-------|
| 变量重命名 | ❌ | ✅ (自动压缩变量名) |
| 文件大小 | 30KB+ | ~5KB ✨ |
| 多着色器一致性 | ❌ | ✅ (保证 varying 同步) |

**实处理**：GLSLX → 压缩后 GLSL → 内联到 JS

#### ❓ 为什么支持 webgl 和 webgl2？

```
现代浏览器      → webgl2 (更快、功能多、绘制效率高)
老旧浏览器(IE)  → webgl  (向后兼容)
恐龙浏览器      → 返回空 API(不崩溃)(graceful fail)
```

**代码**:
```javascript
let gl = canvas.getContext('webgl2')
if (!gl) gl = canvas.getContext('webgl')
if (!gl) return { destroy: () => {}, update: () => {} }
```

#### ❓ 为什么支持 CSS 锚定？

```javascript
// 创建带 ID 的标记
markers: [
  { location: [37, -122], id: 'sf-info' }
]

// HTML 中可以这样绑定
<div id="sf-info" style="anchor-name: --marker-sf-info;">
  San Francisco Info
</div>

// 结果：div 会自动定位到标记点下方
```

### 3.2 性能优化

**为什么只有 5KB？**

1. ✅ **着色器压缩** - GLSLX 自动重命名 (减 50%)
2. ✅ **代码最小化** - terser JS 压缩 (减 40%)
3. ✅ **零依赖** - 不使用任何库 (减少大小)
4. ✅ **着色器内联** - 不需要额外网络请求
5. ✅ **删除死代码** - esbuild tree-shaking

**GPU 性能**：
- 实例化渲染支持（绘制数千标记不降速）
- 高效纹理采样
- 最小化 WebGL 状态切换

### 3.3 扩展点分析

**现有可扩展接口**：
```javascript
// 1. 每帧回调（用于动画/交互）
onRender: (state) => {
  state.phi = newValue
}

// 2. 自定义颜色
markers: [
  { location: [...], color: [1, 0, 0] }  // 红色
]

// 3. 动态更新
globe.update({
  theta: 0.5,
  markers: [...]
})

// 4. 销毁资源
globe.destroy()
```

**添加新功能的方式**：
- 新着色器？ → 创建 `*.glslx` 并编译
- 新光照？ → 修改 `globe.frag.glslx`
- 新交互？ → 通过 `onRender` 和 `update()`

---

## 🔧 第四步：工程细节 (1小时)

### 4.1 构建流程

**命令**:
```bash
pnpm build
# 等价于：node scripts/build.js && cp src/index.d.ts dist/
```

**构建步骤** (位置: `scripts/build.js`):

1. 编译 GLSL 着色器
   ```javascript
   glslx.compile(source, { renaming: 'all' })
   // → 压缩变量名 + 提取代码
   ```

2. 内联着色器到 JavaScript
   ```javascript
   // 将编译后的着色器代码替换 __GLOBE_VERT__ 等占位符
   ```

3. 打包 JavaScript
   ```bash
   esbuild src/index.js --bundle --outfile=dist/index.esm.js
   ```

4. 压缩代码
   ```javascript
   minify(code)  // 使用 terser
   ```

**输出产物**:
```
dist/
├── index.esm.js      # ES Module (5KB) ← 主要输出
└── index.d.ts        # TypeScript 定义
```

### 4.2 关键代码块

#### 错误处理

```javascript
// WebGL 上下文获取失败 → 优雅降级
if (!gl) {
  return { destroy: () => {}, update: () => {} }
}

// 着色器编译失败 → 清理资源并返回 null
if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
  gl.deleteShader(shader)
  return null
}
```

#### 资源管理

```javascript
// 创建时分配
const program = gl.createProgram()
const buffer = gl.createBuffer()
const texture = gl.createTexture()

// 销毁时释放（防止内存泄漏）
return {
  destroy: () => {
    gl.deleteProgram(program)
    gl.deleteBuffer(buffer)
    gl.deleteTexture(texture)
    // ... 清理事件监听器
  }
}
```

### 4.3 依赖关系图

```
index.js (主入口)
  ├─ webgl.js
  │   └─ 提供: createProgram, getUniformLocations, getAttribLocations
  ├─ anchor.js
  │   └─ 提供: createAnchorManager (CSS 锚定)
  ├─ texture.js
  │   └─ 提供: 纹理加载和管理
  └─ *.glslx (着色器)
      └─ 由 build 流程编译并内联

构建依赖:
  glslx        → 编译 GLSL 着色器
  esbuild      → 打包 JavaScript
  terser       → 压缩代码
  TypeScript   → 类型检查(可选)
```

---

## 🚀 第五步：实践验证 (2-3小时)

### 5.1 构建项目

```bash
# 进入项目目录
cd e:\githubAll\Clone\cobe

# 安装依赖 (使用 pnpm)
pnpm install

# 构建库
pnpm build

# 验证输出
ls -la dist/
# 应该看到 index.esm.js (~5KB)
```

### 5.2 运行演示网站

```bash
# 进入网站目录
cd website

# 安装依赖 (首次需要)
pnpm install

# 开发模式启动
pnpm dev

# 打开浏览器查看
http://localhost:3000
```

### 5.3 自由实验 - 修改代码观察效果

**实验 1: 修改标记点颜色**
- 找到: `website/app/components/MarkersArcs.tsx`
- 修改: `markerColor: [1, 0.5, 1]` → `[1, 0, 0]` (红色)
- 结果: 页面中的标记应该变红

**实验 2: 加快地球旋转**
- 找到: `onRender` 回调
- 修改: `state.phi += 0.01` → `state.phi += 0.05`
- 结果: 地球旋转速度变快

**实验 3: 添加新标记**
```javascript
markers: [
  { location: [0, 0], size: 0.05 },              // 赤道和本初子午线
  { location: [51.5074, -0.1278], size: 0.03 }, // 伦敦
  { location: [35.6762, 139.6503], size: 0.03 }, // 东京
]
```

**实验 4: 修改着色器效果 (高端)**
- 编辑: `src/globe.frag.glslx`
- 改变: `diffuse` 值或 `baseColor`
- 构建: `pnpm build`
- 网站: 重新加载看新效果

---

## ✅ 学习进度检查表

### 初级阶段 (现在)
- [x] 了解项目的总体结构
- [x] 知道入口文件在哪里
- [x] 理解主要的三个模块(地球/标记/弧线)
- [ ] 在本地成功构建项目
- [ ] 运行演示网站

### 中级阶段 (下周)
- [ ] 修改参数并看到效果
- [ ] 理解坐标系统变换
- [ ] 分析 GLSL 着色器的作用
- [ ] 理解 CSS 锚定机制
- [ ] 阅读完整的 `index.js`

### 高级阶段 (2周后)
- [ ] 能独立添加新的着色器
- [ ] 能优化性能
- [ ] 能为项目贡献代码
- [ ] 理解 WebGL 的 GPU 管道
- [ ] 能创建基于 COBE 的应用

---

## 📖 推荐学习资源

| 资源 | 链接 | 用途 |
|------|------|------|
| 官方 Demo | https://cobe.vercel.app | 看效果 |
| GitHub 源码 | https://github.com/shuding/cobe | 研究代码 |
| WebGL 基础 | https://webglfundamentals.org/ | 学 WebGL |
| GLSL 教程 | https://learnopengl.com/ | 学着色器 |
| MDN WebGL | https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API | 参考文档 |

---

## 🎓 关键概念速查

| 概念 | 解释 | 在项目中的位置 |
|------|------|----------------|
| **Phi (φ)** | 纵向旋转角度 | index.js |
| **Theta (θ)** | 横向旋转角度 | index.js |
| **Lat/Lon** | 纬度/经度 (GPS坐标) | latLonTo3D() |
| **3D 坐标** | 球面上的 [x, y, z] | latLonTo3D() 输出 |
| **Shader** | 在 GPU 运行的代码 | *.glslx 文件 |
| **VBO** | 顶点缓冲对象 | webgl.js |
| **GLSLX** | GLSL 扩展语言 | 编译步骤 |
| **DPR** | 设备像素比 | index.js 第 60 行 |

---

## ❓ 常见问题解答

**Q: COBE vs Cesium 的区别？**  
A: COBE 极致精简 (5KB)、高性能、专注美观；Cesium 功能丰富、支持 GIS、文件大

**Q: 能在标记上显示文字吗？**  
A: 可以，使用 CSS 锚定 + DOM 元素实现（见 website/app/components/CustomLabels.tsx）

**Q: 能支持 3D 模型吗？**  
A: 不支持，设计理念就是极致简化；但可以创建新着色器绘制其他 3D 对象

**Q: 最多支持多少个标记？**  
A: 数千个标记都能保持 60fps（具体取决于设备）

**Q: 可以商业使用吗？**  
A: 可以，MIT 许可证允许商业使用

---

## 📝 下一步行动

### 今天 (第一天)
```
⏱️ 30 分钟
1. ✅ 阅读本指南
2. ⏳ pnpm install && pnpm build
3. ⏳ 访问 http://localhost:3000
4. 修改一个参数看效果
```

### 本周 (第 2-5 天)
```
⏱️ 2-3 小时
1. 深入阅读 index.js 代码
2. 理解着色器的基本概念
3. 做 3-4 个修改实验
4. 绘制项目架构图
```

### 本月 (第 1-4 周)
```
⏱️ 按需
1. 学习 WebGL 和 GLSL
2. 能独立修改和扩展代码
3. 为项目提交 PR
4. 基于 COBE 创建自己的应用
```

---

**🎉 恭喜！** 你现在对 COBE 项目有了全面的了解。
立即开始实践验证部分吧！
