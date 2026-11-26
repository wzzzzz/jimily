# 参考实现接入说明

`overview.client.vue` 是账本概览页的 Nuxt 4 参考实现，位于 `design/` 下，**不会被 Nuxt 编译**。开发节点按以下步骤接入：

## 1. 放置页面

```bash
cp design/personal-ledger-overview/reference/overview.client.vue \
   app/pages/user/overview.client.vue
```

页面使用现有 `public` 布局与 `auth` 中间件：

```ts
definePageMeta({
  layout: 'public',
  middleware: ['auth'],
});
```

## 2. 新增导航入口

`app/components/layout/AppSidebar.vue`：

- 在普通用户菜单中新增「概览」，指向 `/user/overview`。
- 建议放在「Jimi 助手」之前，图标使用 `HomeIcon` 或 `Squares2X2Icon`。

`app/components/layout/AppBottomNav.vue`：

- 新增「概览」，指向 `/user/overview`。
- 底栏由 4 项调整为 5 项，容器 `grid-cols-4` 改为 `grid-cols-5`。
- 保持现有选中态样式（`bg-primary-500/20`、`text-primary-600`）。

## 3. 复用现有组件与接口

| 能力 | 现有实现 | 用法 |
| --- | --- | --- |
| 汇总卡 | `app/components/flows/FlowsStatistics.vue` | 传入 `flow/page` 返回的 `totalIn`、`totalOut`、`notInOut` |
| 最近流水 | `flow/page` | `pageSize: 5`，按 `day` 倒序 |
| 支出构成 | `ChartsCommonPie` | `group-by="industryType"`、`flow-type="支出"`、月份起止 |
| 趋势 | `MonthBar` | 现有组件；若需严格限制 6 个月，可扩展 props 或改用 `analytics/month` 数据 |
| 快速记账 | `FlowEditDialog` | 通过 `showFlowEditDialog`、`dialogFormTitle`、`selectedFlow` 复用 |
| 预算 | `budget/all` | 按 `month` 匹配当前所选月份 |
| 提示 | `Alert` / `Toast` | 保存成功、失败、取消等 |

## 4. 关键约束

- 不新增数据库表、字段、Prisma migration。
- 不修改登录、JWT、用户隔离、密码哈希。
- 不新增运行时依赖或图表库。
- 所有查询通过现有 `doApi`，服务端从 JWT 取 `userId`。
- 弹窗保持显式取消，不靠点击遮罩关闭。
- 同时验证桌面端与窄屏、加载、空数据、错误、禁用、长文本与深色模式。

## 5. 验证

需求设计节点已验证：

- 静态原型在 4173 端口可访问。
- 桌面/移动/深色/记一笔弹窗截图正常。
- 快速记账后汇总与最近流水即时更新（Mock 数据）。

开发节点需补充：

```bash
pnpm install
pnpm build
# 需要 PostgreSQL 与 .env 时再执行
pnpm dev
```

未执行 `pnpm build` / `pnpm dev` 的原因：当前需求设计环境未提供该仓库的 PostgreSQL 与 `.env`，且仓库 `AGENTS.md` 要求未经明确要求不启动长期进程。开发节点应执行并记录结果。
