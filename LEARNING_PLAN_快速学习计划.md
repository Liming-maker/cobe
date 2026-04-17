# COBE 项目学习计划

## 📋 项目概览
- **项目名称**: COBE - 高性能 WebGL 地球仪可视化库
- **关键特性**: 
  - 高性能，零依赖，~5KB
  - 支持标记点(markers)和弧线(arcs)
  - 支持 CSS 锚点定位，DOM 可交互
  
- **技术栈**: 
  - WebGL 渲染引擎
  - GLSL 着色器
  - TypeScript
  - Next.js(演示网站)
  - esbuild(构建工具)

## 🎯 学习路线图（七步学习法）

### ✅ 第一步：全局认知（10-15分钟）完成
**核心问题**：项目分成什么模块？入口在哪？
**理解内容**：
- **主要模块**：
  1. 核心库（src/）- WebGL 渲染引擎
  2. 演示网站（website/） - Next.js 应用
  3. 构建系统（scripts/） - esbuild 编译
  
- **目录职责**：
  - `src/index.js` 💎 主入口，导出 createGlobe 函数
  - `src/webgl.js` 💎 WebGL 基础工具（创建着色器、程序）
  - `src/*.glslx` 💎 GLSL 着色器（由 glslx 编译）
  - `website/` 演示和验证
  
- **架构风格**：单一库文件 + WebGL 渲染 + 支持动态更新

### 📖 第二步：理解核心业务路径（30分钟）
**典型场景**：用户看到一个旋转的地球，上面有标记点和连接线
**调用链路**：
```
createGlobe(canvas, options)
  ↓ 1. 初始化：创建 WebGL 上下文(webgl2 优先降级到 webgl)
  ↓ 2. 编译：创建顶点/片段着色器 for 地球/标记/弧线
  ↓ 3. 纹理：读取地球纹理图像
  ↓ 4. 缓冲：设置顶点缓冲区对象(VBO)
  ↓ 5. 循环：requestAnimationFrame 连续渲染
  ↓ 6. 更新：通过 onRender 回调更新 phi/theta
  ↓ 7. 销毁：destroy() 清理资源
```

**三类渲染对象**：
- 🌍 **地球**：大球体 + 贴图
- 📍 **标记点**：可选颜色、大小，支持 CSS 锚定
- 🔗 **弧线**：从A地到B地的曲线

### 🧠 第三步：搞懂设计思路（需要深入）
待完成...

### 🔧 第四步：分析工程细节（需要深入）
待完成...

### 🚀 第五步：实践验证（动手操作）
- [ ] 构建项目：`pnpm install && pnpm build`
- [ ] 运行演示：`cd website && pnpm dev`
- [ ] 修改参数观察效果

### 📝 第六步：从 Good First Issues 入手
GitHub 上查看是否有新人友好的任务

### 💾 第七步：知识沉淀
- [ ] 绘制架构图
- [ ] 编写模块总结

## 📊 核心文件清单
| 文件 | 用途 | 优先级 |
|------|------|--------|
| src/index.js | 主入口文件 | 高 |
| src/webgl.js | WebGL 实现 | 高 |
| src/globe.vert.glslx | 顶点着色器 | 高 |
| src/globe.frag.glslx | 片段着色器 | 高 |
| src/arc.glslx | 弧线着色器 | 中 |
| src/marker.glslx | 标记着色器 | 中 |
| src/anchor.js | CSS 锚点定位 | 中 |
| src/texture.js | 纹理管理 | 低 |
| scripts/build.js | 构建脚本 | 低 |
1
2
3
4