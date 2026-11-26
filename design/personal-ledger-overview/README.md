# 账本概览 — 设计原型与参考实现

本目录是「个人记账项目雏形」在 Jimily / 记米粒中的需求设计产物，包含：

- `index.html`：可运行的静态原型（Mock 数据，无后端依赖）。
- `assets/logo.webp`：记米粒 Logo。
- `screenshots/`：桌面端、移动端、深色模式与记一笔弹窗截图。
- `reference/overview.client.vue`：Nuxt 4 页面参考实现（不会被 Nuxt 编译）。
- `reference/README.md`：参考实现接入说明。

## 预览

静态原型只依赖一个静态文件服务器：

```bash
cd design/personal-ledger-overview
python3 -m http.server 4173
```

打开 `http://127.0.0.1:4173/`。

原型覆盖：

- 桌面端侧栏 + 顶栏 + 概览内容。
- 移动端顶栏 + 底栏 5 项（概览、预算、分析、日历、流水）。
- 月份切换、回到本月。
- 汇总卡、预算进度、最近流水、支出构成、近 6 个月趋势。
- 快速记账弹窗（新增/编辑、金额校验、保存后概览刷新）。
- 深色模式切换。
- 空态、无预算、超支提示等状态。

> 原型中的金额、流水、预算、趋势均为 Mock 数据，用于评审视觉与交互，不代表真实账本。

## 设计系统对齐

- 主题：沿用 `app/utils/theme-presets.ts` 的默认白蓝主题；主色 `primary-500/600`，辅助色 slate，强调色 indigo。
- 语义色：`bg-surface`、`bg-surface-muted`、`text-foreground`、`text-muted`、`border-border`、`text-primary-*`。
- 卡片：白底、`border-border`、`rounded-lg`、`shadow-soft`，与流水页、分析页一致。
- 图标：Heroicons 线性图标风格。
- 图表：ECharts / `nuxt-echarts`；原型使用轻量 SVG 模拟环形图与柱状图。
- 深色模式：原型使用 CSS 变量模拟现有 `dark:` 语义色；实际实现使用现有主题变量。

## 参考实现

`reference/overview.client.vue` 是页面级参考实现，目标位置：

```text
app/pages/user/overview.client.vue
```

接入时还需要：

1. 在 `app/components/layout/AppSidebar.vue` 的用户菜单新增「概览」。
2. 在 `app/components/layout/AppBottomNav.vue` 新增「概览」，底栏由 4 项调整为 5 项（`grid-cols-5`）。
3. 确认 `FlowsStatistics`、`ChartsCommonPie`、`MonthBar`、`FlowEditDialog` 的导入与 props 与当前仓库一致。
4. 按仓库规范运行 `pnpm install`、`pnpm build`；本地预览需 PostgreSQL 与 `.env`，不在需求设计节点执行。
5. 不新增数据库表、字段、Prisma migration 或运行时依赖。

详见 `reference/README.md` 与 `docs/参考代码与变更说明.md`。
