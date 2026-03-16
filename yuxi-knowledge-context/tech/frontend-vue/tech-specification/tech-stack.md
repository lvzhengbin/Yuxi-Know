# 前端技术栈说明

> 领域：frontend-vue | 项目：yuxi-know v0.5.1 | 更新：2026-03-16

## 语言与运行时

| 项目 | 版本 |
|------|------|
| JavaScript (ES Modules) | - |
| Node.js (dev) | - |
| 包管理 | pnpm@10.11.0 |

## 核心框架

| 框架 | 版本 | 用途 |
|------|------|------|
| Vue | ^3.5.29 | 核心 UI 框架（Composition API） |
| Vue Router | ^4.6.4 | SPA 路由管理 |
| Pinia | ^3.0.4 | 状态管理 |
| pinia-plugin-persistedstate | ^4.7.1 | Pinia 状态持久化（localStorage） |

## 构建工具

| 工具 | 版本 | 用途 |
|------|------|------|
| Vite | ^7.3.1 | 构建工具 + Dev Server |
| @vitejs/plugin-vue | ^6.0.4 | Vue SFC 编译插件 |

## UI 组件库

| 库 | 版本 | 用途 |
|----|------|------|
| ant-design-vue | ^4.2.6 | 主 UI 组件库 |
| @ant-design/icons-vue | ^7.0.1 | Ant Design 图标 |
| lucide-vue-next | ^0.542.0 | 推荐图标库（更现代） |
| less | ^4.5.1 | CSS 预处理器 |

## 可视化

| 库 | 版本 | 用途 |
|----|------|------|
| @antv/g6 | ^5.0.51 | 知识图谱可视化（主图库） |
| sigma + graphology | ^3.0.2 / ^0.26.0 | 备用图渲染（Sigma.js） |
| echarts | ^6.0.0 | 数据图表（Dashboard） |
| d3 | ^7.9.0 | 底层数据可视化工具 |
| @sigma/edge-curve | ^3.1.0 | Sigma 边弯曲插件 |
| @sigma/node-border | ^3.0.0 | Sigma 节点边框插件 |

## Markdown / 富文本

| 库 | 版本 | 用途 |
|----|------|------|
| marked | ^16.4.2 | Markdown 解析 |
| marked-highlight | ^2.2.3 | Markdown 代码高亮 |
| highlight.js | ^11.11.1 | 代码语法高亮 |
| md-editor-v3 | ^5.8.5 | Markdown 编辑器组件 |

## 思维导图

| 库 | 版本 | 用途 |
|----|------|------|
| markmap-lib | ^0.18.12 | Markdown → 思维导图数据 |
| markmap-view | ^0.18.12 | 思维导图渲染 |

## 工具库

| 库 | 版本 | 用途 |
|----|------|------|
| @vueuse/core | ^13.9.0 | Vue Composition 工具集 |
| dayjs | ^1.11.19 | 日期处理 |

## 代码质量

| 工具 | 版本 | 用途 |
|------|------|------|
| ESLint | ^9.39.3 | 代码规范检查 |
| Prettier | ^3.8.1 | 代码格式化 |
| eslint-plugin-vue | ^10.8.0 | Vue 专用 lint 规则 |

## 关键配置

### Vite 代理

```js
// web/vite.config.js
proxy: {
  '^/api': {
    target: env.VITE_API_URL || 'http://api:5050',
    changeOrigin: true
  }
}
```

路径别名 `@` → `./src`，所有 API 请求代理到后端服务。

### CSS 设计系统

颜色变量定义于 `web/src/assets/css/base.css`：
- `--main-*`：主品牌色阶（青蓝色系，`--main-color: #016179`）
- `--gray-*`：灰色阶
- `--color-primary/secondary/success/error/warning/info-*`：标准5档语义色阶
- 支持暗黑模式（dark mode）
