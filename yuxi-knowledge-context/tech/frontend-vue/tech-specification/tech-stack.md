# 技术栈清单 — frontend-vue

**项目类型**: Vue 3 前端 SPA
**语言**: JavaScript (ES2022+)
**包管理工具**: pnpm
**构建工具**: Vite

---

## 核心框架

| 技术 | 版本 | 用途 |
|------|------|------|
| Vue 3 | ^3.x | 渐进式前端框架，Composition API |
| Vite | ^6.x | 构建工具，开发热重载 |
| Vue Router | ^4.x | SPA 路由管理 |
| Pinia | ^3.x | 状态管理（替代 Vuex） |
| pinia-plugin-persistedstate | ^4.x | Pinia 持久化（localStorage） |

## UI 组件库

| 技术 | 版本 | 用途 |
|------|------|------|
| Ant Design Vue | ^4.2.6 | 主要 UI 组件库（全量引入） |
| @ant-design/icons-vue | ^7.0.1 | Ant Design 图标 |
| lucide-vue-next | - | 补充图标库（推荐，需注意尺寸） |

## 数据可视化

| 技术 | 版本 | 用途 |
|------|------|------|
| @antv/g6 | ^5.0.51 | 知识图谱可视化（主要方案） |
| @sigma/edge-curve | ^3.1.0 | Sigma.js 曲线边渲染 |
| @sigma/node-border | ^3.0.0 | Sigma.js 节点边框渲染 |

## 工具库

| 技术 | 版本 | 用途 |
|------|------|------|
| @vueuse/core | ^13.9.0 | Vue 组合式工具函数集 |
| axios / fetch | - | HTTP 请求（封装于 `src/apis/base.js`） |

## 样式

| 技术 | 用途 |
|------|------|
| Less | 组件样式（CLAUDE.md 规定，必须优先使用 base.css 颜色变量） |
| `src/assets/css/base.css` | 全局颜色变量定义，所有颜色值从此引用 |

## 开发工具

| 技术 | 版本 | 用途 |
|------|------|------|
| ESLint | - | 代码检查（`npm run lint`） |
| Prettier | - | 代码格式化（`npm run format`，`--experimental-cli`） |
| Vitest | (推测) | 单元测试（`src/utils/__tests__/` 存在测试文件） |

---

## 关键技术决策

- **为什么选 Ant Design Vue？** 提供完整的企业级组件，与后台管理系统风格匹配。
- **为什么选 G6 做图谱可视化？** @antv/g6 v5 支持大规模节点渲染，原生支持知识图谱的节点/边类型定制。
- **为什么选 Pinia？** 比 Vuex 更轻量，原生 TypeScript 支持，持久化插件简单易用。
- **CSS 策略**：使用 Less 变量而非硬编码颜色，支持暗黑模式切换（v0.4.0 新增）。

*由 Agent Knowledge Kit v3.0 生成 · 2026-03-19*
