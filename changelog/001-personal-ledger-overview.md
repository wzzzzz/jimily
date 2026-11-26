# 001 账本概览页与快速记账入口

- 日期：2026-11-26
- 分支：`feat/personal-ledger-overview`
- 关联文档：`docs/需求规格书（Spec）/产品 Spec.md`、`docs/需求规格书（Spec）/技术 Spec.md`（交付仓库 Gitea `test-public-repo`）、`docs/账本概览.md`、`docs/e2e/账本概览-E2E用例.md`

## 目的

在现有 Jimily 应用内新增「账本概览」页面与快速记账入口，让登录用户一屏看到所选月份的收入、支出、净收入、预算进度、最近流水、支出构成与近 6 个月趋势。

## 改动内容

| 文件 | 类型 | 说明 |
| --- | --- | --- |
| `app/pages/user/overview.client.vue` | 新增 | 账本概览页：月份切换、汇总、预算进度、最近流水、支出构成、近 6 个月趋势、快速记账/编辑弹窗 |
| `app/components/layout/AppSidebar.vue` | 修改 | 普通用户菜单首位新增「概览」（`HomeIcon`，`/user/overview`） |
| `app/components/layout/AppBottomNav.vue` | 修改 | 新增「概览」为第 1 项，底栏 4 → 5 列；新增路径归一化，修复 `currentPath`（`user/xxx`）与 `item.path`（`/user/xxx`）不一致导致选中态失效的问题 |
| `app/components/charts/CommonPie.vue` | 修改 | 可选 `maxItems`（前 N 类 + 「其他」）、`emptyText`；失败态 + 重试；请求序号防竞态；「其他」不下钻；空态恢复后 `resize` |
| `app/components/charts/MonthBar.vue` | 修改 | 可选 `startDay` / `endDay` / `months` / `emptyText`；指定月份补 0；失败态 + 重试；请求序号防竞态；指定月份时隐藏年份筛选；空态恢复后 `resize` |
| `docs/账本概览.md` | 新增 | 页面使用说明与数据口径 |
| `docs/e2e/账本概览-E2E用例.md` | 新增 | E2E 用例（核心路径、异常路径、回归风险） |

## 数据与契约

- 不新增/修改 `server/api/**`、`server/utils/db/**`、`server/middleware/**`。
- 不新增/修改数据库表、字段、索引，不新增或编辑 `prisma/migrations/**`。
- 不新增/修改 `public/openapi.json`（构建生成内容应与改动前一致）。
- 不新增运行时依赖；继续使用 ECharts、Heroicons、Tailwind、vue-toastification。

## 是否需要开发者生成数据库迁移

不需要。本次无 schema 变更。

## 验证结果

| 检查项 | 命令 | 结果 |
| --- | --- | --- |
| 依赖安装 | `pnpm install --frozen-lockfile`（pnpm 10.30.3 / Node v22.23.2） | 通过 |
| 生产构建 | `pnpm build` | **未通过（环境内存限制）**：7223 个模块转换完成后，在 `rendering chunks` 阶段被沙箱 cgroup OOM kill（exit 137 / `Killed process (node)`，anon-rss 约 2.0–2.1 GB），非代码编译错误；已尝试 `--max-old-space-size` 1536/1792/1920/2048/2304、关闭 minify 与 `reportCompressedSize`、释放沙箱内 MySQL/浏览器进程，仍受同一内存上限限制 |
| SFC 编译校验 | `@vue/compiler-sfc`：`parse` + `compileScript` + `compileTemplate`（5 个改动文件） | 通过：`ALL_SFC_COMPILE_OK` |
| 类型检查 | 仓库未配置 `typecheck` 脚本 | 未执行（不新增 `vue-tsc` 依赖） |
| 单元测试 | 仓库无测试脚本/框架 | 未执行（本次不新增测试框架依赖） |
| E2E / 手工矩阵 | 见 `docs/e2e/账本概览-E2E用例.md` | 未执行：需 PostgreSQL + `.env` 运行环境，且仓库规范要求未经明确要求不启动长期进程 |
| OpenAPI | 构建钩子生成 | 无变化（`public/openapi.json` 无 diff） |

### 实际结果补充

- 构建日志显示代码已完成模块转换（`7223 modules transformed`），失败点统一在 `rendering chunks`，属于打包内存峰值超出沙箱上限；建议在可用内存 ≥ 4 GB（推荐 8 GB）的 CI/开发机执行 `pnpm build` 复核。
- 本次改动为纯前端页面与共享组件的可选 props，不涉及服务端、数据库与接口契约；SFC 编译校验已覆盖模板与脚本语法。

## 未做事项与风险

- 未执行 `pnpm dev` / 真实接口联调：需要 PostgreSQL 与 `.env`。
- 预算「已用」取所选月份实时 `totalOut`，不使用可能滞后的 `Budget.used` 快照；与预算页「刷新额度」后的数字可能不同，属预期口径。
- 同一用户同一月份存在多条预算时按数组顺序取第一条，与现有预算页 `res[0]` 行为一致；本迭代不新增唯一约束。
- 支出构成前 6 类之外的分类合并为「其他」，「其他」不支持下钻（无法用单一分类条件表达）。
- 根路由仍为 `/` → `/user/jimi`；移动端底栏为 5 项。
