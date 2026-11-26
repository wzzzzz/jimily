<template>
  <div class="p-2 md:p-4 pb-20 lg:pb-4 min-h-full space-y-3">
    <!-- 顶部：标题 + 月份控件 + 记一笔 -->
    <div class="flex flex-col gap-3 md:flex-row md:items-end md:justify-between">
      <div>
        <h1 class="text-2xl font-bold text-foreground md:text-3xl">账本概览</h1>
        <p class="mt-1 text-sm text-muted">
          一眼看清本月收支、预算与消费方向
        </p>
      </div>

      <div class="flex flex-wrap items-center gap-2">
        <div
          class="flex items-center gap-1 rounded-full border border-border bg-surface px-2 py-1 shadow-sm"
        >
          <button
            type="button"
            class="rounded-full p-1 text-foreground/60 transition-colors hover:bg-surface-muted focus:outline-none focus:ring-2 focus:ring-primary-500"
            aria-label="上个月"
            @click="changeMonth(-1)"
          >
            <ChevronLeftIcon class="h-4 w-4" />
          </button>
          <span class="min-w-20 px-2 text-center text-sm font-medium tabular-nums">
            {{ monthLabel }}
          </span>
          <button
            type="button"
            class="rounded-full p-1 text-foreground/60 transition-colors hover:bg-surface-muted focus:outline-none focus:ring-2 focus:ring-primary-500 disabled:cursor-not-allowed disabled:opacity-30"
            aria-label="下个月"
            :disabled="isCurrentMonth"
            @click="changeMonth(1)"
          >
            <ChevronRightIcon class="h-4 w-4" />
          </button>
        </div>
        <button
          v-if="!isCurrentMonth"
          type="button"
          class="rounded-md px-2 py-1 text-xs font-medium text-primary-600 transition-colors hover:text-primary-500 focus:outline-none focus:ring-2 focus:ring-primary-500"
          @click="goCurrentMonth"
        >
          回到本月
        </button>
        <button
          type="button"
          class="inline-flex items-center gap-1 rounded-md bg-primary-600 px-3 py-2 text-sm font-medium text-white shadow-sm transition-colors hover:bg-primary-500 focus:outline-none focus:ring-2 focus:ring-primary-500 focus:ring-offset-2"
          @click="openCreateDialog"
        >
          <PlusIcon class="h-4 w-4" />
          记一笔
        </button>
      </div>
    </div>

    <!-- 汇总 -->
    <div class="relative">
      <FlowsStatistics :statistics="statistics" />
      <div
        v-if="loading"
        class="absolute inset-0 animate-pulse rounded-lg bg-surface/70"
        role="status"
        aria-label="汇总加载中"
      ></div>
    </div>

    <div class="grid grid-cols-1 gap-3 lg:grid-cols-2">
      <!-- 预算进度 -->
      <section class="rounded-lg border border-border bg-surface p-4 shadow-soft">
        <div class="mb-3 flex items-start justify-between gap-3">
          <div>
            <h2 class="font-semibold text-foreground">本月预算</h2>
            <p class="mt-1 text-xs text-muted">
              {{ monthLabel }} · 已用 {{ formatMoney(budgetUsed) }}
            </p>
          </div>
          <NuxtLink
            to="/user/budget"
            class="text-xs font-medium text-primary-600 hover:text-primary-500"
          >
            {{ currentBudget ? "调整" : "去设置" }}
          </NuxtLink>
        </div>

        <div v-if="loading" class="space-y-2 py-2" role="status" aria-label="预算加载中">
          <div class="h-8 w-32 animate-pulse rounded bg-surface-muted"></div>
          <div class="h-2 w-full animate-pulse rounded-full bg-surface-muted"></div>
        </div>
        <div v-else-if="budgetError" class="py-6 text-center">
          <p class="text-sm text-red-500">预算加载失败</p>
          <button
            type="button"
            class="mt-2 rounded-md px-3 py-1 text-xs font-medium text-primary-600 hover:bg-surface-muted focus:outline-none focus:ring-2 focus:ring-primary-500"
            @click="loadOverview"
          >
            重试
          </button>
        </div>
        <template v-else-if="currentBudget">
          <div class="flex items-end justify-between gap-3">
            <p class="text-2xl font-bold tabular-nums text-primary-600">
              {{ formatMoney(Math.abs(budgetRemaining)) }}
              <span class="text-xs font-medium text-muted">
                {{ budgetRemaining < 0 ? "已超支" : "剩余" }}
              </span>
            </p>
            <span
              class="rounded-full px-2 py-1 text-xs font-medium"
              :class="
                budgetRemaining < 0
                  ? 'bg-red-50 text-red-600 dark:bg-red-900/30 dark:text-red-400'
                  : 'bg-primary-50 text-primary-600'
              "
            >
              {{ budgetPercent }}%
            </span>
          </div>
          <div
            class="mt-3 h-2 overflow-hidden rounded-full bg-surface-muted"
            role="progressbar"
            aria-label="预算使用进度"
            aria-valuemin="0"
            aria-valuemax="100"
            :aria-valuenow="Math.min(budgetPercent, 100)"
          >
            <div
              class="h-full rounded-full transition-all duration-300"
              :class="budgetRemaining < 0 ? 'bg-red-500' : 'bg-primary-500'"
              :style="{ width: `${Math.min(budgetPercent, 100)}%` }"
            ></div>
          </div>
          <div class="mt-2 flex justify-between text-xs text-muted">
            <span>
              预算
              <strong class="tabular-nums text-foreground">{{
                formatMoney(currentBudget.budget)
              }}</strong>
            </span>
            <span>
              已用
              <strong class="tabular-nums text-foreground">{{
                formatMoney(budgetUsed)
              }}</strong>
            </span>
          </div>
        </template>
        <div v-else class="py-6 text-center">
          <p class="text-sm text-muted">未设置预算</p>
          <NuxtLink
            to="/user/budget"
            class="mt-2 inline-block text-xs font-medium text-primary-600 hover:text-primary-500"
          >
            去设置预算
          </NuxtLink>
        </div>
      </section>

      <!-- 最近流水 -->
      <section class="rounded-lg border border-border bg-surface p-4 shadow-soft">
        <div class="mb-3 flex items-start justify-between gap-3">
          <div>
            <h2 class="font-semibold text-foreground">最近流水</h2>
            <p class="mt-1 text-xs text-muted">
              {{ monthLabel }} · 最近 {{ recentFlows.length }} 笔
            </p>
          </div>
          <NuxtLink
            to="/user/flows"
            class="text-xs font-medium text-primary-600 hover:text-primary-500"
          >
            查看全部
          </NuxtLink>
        </div>

        <div v-if="loading" class="space-y-2" role="status" aria-label="最近流水加载中">
          <div
            v-for="i in 3"
            :key="i"
            class="h-10 animate-pulse rounded bg-surface-muted"
          ></div>
        </div>
        <div v-else-if="flowError" class="py-6 text-center">
          <p class="text-sm text-red-500">流水加载失败</p>
          <button
            type="button"
            class="mt-2 rounded-md px-3 py-1 text-xs font-medium text-primary-600 hover:bg-surface-muted focus:outline-none focus:ring-2 focus:ring-primary-500"
            @click="loadOverview"
          >
            重试
          </button>
        </div>
        <div v-else-if="recentFlows.length > 0" class="divide-y divide-border">
          <button
            v-for="flow in recentFlows"
            :key="flow.id"
            type="button"
            class="flex w-full items-center gap-3 py-2 text-left transition-colors hover:bg-surface-muted focus:outline-none focus:ring-2 focus:ring-primary-500"
            @click="openEditDialog(flow)"
          >
            <span
              class="grid h-8 w-8 flex-shrink-0 place-items-center rounded-md bg-primary-50 text-xs font-semibold text-primary-600"
            >
              {{ (flow.industryType || "其").slice(0, 1) }}
            </span>
            <span class="min-w-0 flex-1">
              <span class="block truncate text-sm font-medium text-foreground">
                {{ flow.name || flow.industryType || "未命名流水" }}
              </span>
              <span class="block truncate text-xs text-muted">
                {{ flow.industryType || "未分类" }} ·
                {{ flow.account?.name || "未关联账户" }} ·
                {{ formatDay(flow.day) }}
              </span>
            </span>
            <span
              class="flex-shrink-0 text-sm font-semibold tabular-nums"
              :class="
                flow.flowType === '收入' ? 'text-primary-600' : 'text-foreground'
              "
            >
              {{ flowTypePrefix(flow.flowType) }}{{ formatMoney(Math.abs(Number(flow.money || 0))) }}
            </span>
            <ChevronRightIcon
              class="h-4 w-4 flex-shrink-0 text-foreground/30"
            />
          </button>
        </div>
        <div v-else class="py-6 text-center">
          <p class="text-sm text-muted">本月还没有流水</p>
          <button
            type="button"
            class="mt-2 rounded-md px-3 py-1 text-xs font-medium text-primary-600 hover:bg-surface-muted focus:outline-none focus:ring-2 focus:ring-primary-500"
            @click="openCreateDialog"
          >
            记一笔
          </button>
        </div>
      </section>

      <!-- 支出构成 -->
      <section class="rounded-lg border border-border bg-surface p-4 shadow-soft">
        <ChartsCommonPie
          title="支出构成"
          width="100%"
          height="260px"
          group-by="industryType"
          flow-type="支出"
          series-name="支出"
          :start-day="monthRange.start"
          :end-day="monthRange.end"
          :max-items="6"
          empty-text="本月暂无支出"
        />
      </section>

      <!-- 近 6 个月趋势 -->
      <section class="rounded-lg border border-border bg-surface p-4 shadow-soft">
        <MonthBar
          title="近 6 个月趋势"
          width="100%"
          height="260px"
          :start-day="trendRange.start"
          :end-day="trendRange.end"
          :months="trendMonths"
          empty-text="暂无趋势数据"
        />
      </section>
    </div>

    <!-- 快速记账 / 编辑流水弹窗 -->
    <FlowEditDialog
      v-if="showFlowEditDialog"
      :title="dialogFormTitle"
      :flow="selectedFlow"
      :success-callback="loadOverview"
    />
  </div>
</template>

<script setup lang="ts">
import {
  ChevronLeftIcon,
  ChevronRightIcon,
  PlusIcon,
} from "@heroicons/vue/24/outline";
import FlowsStatistics from "~/components/flows/FlowsStatistics.vue";
import FlowEditDialog from "~/components/dialog/FlowEditDialog.vue";
import ChartsCommonPie from "~/components/charts/CommonPie.vue";
import MonthBar from "~/components/charts/MonthBar.vue";
import { doApi } from "~/utils/api";
import { showFlowEditDialog } from "~/utils/flag";
import { formatDay } from "~/utils/common";
import type { Page } from "~/utils/model";
import type { Flow } from "~/utils/table";

definePageMeta({
  layout: "public",
  middleware: ["auth"],
});

/** 本地时区当前月，避免 toISOString() 的 UTC 跨月问题 */
const currentMonth = (() => {
  const now = new Date();
  return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, "0")}`;
})();

const selectedMonth = ref(currentMonth);
const loading = ref(false);
const flowError = ref(false);
const budgetError = ref(false);
const requestSeq = ref(0);

const emptyPage = (): Page<Flow> => ({
  pageNum: 1,
  pageSize: 5,
  pages: 0,
  total: 0,
  totalIn: 0,
  totalOut: 0,
  notInOut: 0,
  data: [],
});

const flowPage = ref<Page<Flow>>(emptyPage());
const budgets = ref<any[]>([]);
const dialogFormTitle = ref("新增流水");
const selectedFlow = ref<Record<string, any>>({});

const isCurrentMonth = computed(() => selectedMonth.value === currentMonth);

const monthRange = computed(() => {
  const [year, month] = selectedMonth.value.split("-").map(Number);
  const lastDay = new Date(year, month, 0).getDate();
  return {
    start: `${selectedMonth.value}-01`,
    end: `${selectedMonth.value}-${String(lastDay).padStart(2, "0")}`,
  };
});

/** 以所选月份为终点的近 6 个月（升序） */
const trendMonths = computed(() => {
  const [year, month] = selectedMonth.value.split("-").map(Number);
  return Array.from({ length: 6 }, (_, i) => {
    const d = new Date(year, month - 1 - (5 - i), 1);
    return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}`;
  });
});

const trendRange = computed(() => ({
  start: `${trendMonths.value[0]}-01`,
  end: monthRange.value.end,
}));

const monthLabel = computed(() => {
  const [year, month] = selectedMonth.value.split("-");
  return `${year}年${Number(month)}月`;
});

const statistics = computed(() => ({
  totalIn: Number(flowPage.value?.totalIn || 0),
  totalOut: Number(flowPage.value?.totalOut || 0),
  notInOut: Number(flowPage.value?.notInOut || 0),
}));

const recentFlows = computed<Flow[]>(() =>
  Array.isArray(flowPage.value?.data) ? flowPage.value.data : [],
);

const budgetUsed = computed(() => Number(flowPage.value?.totalOut || 0));

const currentBudget = computed(() =>
  budgets.value.find((item) => item.month === selectedMonth.value),
);

const budgetPercent = computed(() => {
  const limit = Number(currentBudget.value?.budget || 0);
  if (!limit) return 0;
  return Math.round((budgetUsed.value / limit) * 100);
});

const budgetRemaining = computed(() => {
  const limit = Number(currentBudget.value?.budget || 0);
  return Number((limit - budgetUsed.value).toFixed(2));
});

const loadOverview = async () => {
  const seq = ++requestSeq.value;
  loading.value = true;
  flowError.value = false;
  budgetError.value = false;

  const [flowRes, budgetRes] = await Promise.allSettled([
    doApi.post<Page<Flow>>("api/entry/flow/page", {
      startDay: monthRange.value.start,
      endDay: monthRange.value.end,
      pageNum: 1,
      pageSize: 5,
    }),
    doApi.post<any[]>("api/entry/budget/all", {}),
  ]);

  // 月份切换竞态：只应用最新一次请求结果
  if (seq !== requestSeq.value) return;

  if (flowRes.status === "fulfilled" && flowRes.value) {
    flowPage.value = flowRes.value;
  } else {
    flowPage.value = emptyPage();
    flowError.value = true;
  }

  if (budgetRes.status === "fulfilled") {
    budgets.value = Array.isArray(budgetRes.value) ? budgetRes.value : [];
  } else {
    budgets.value = [];
    budgetError.value = true;
  }

  loading.value = false;
};

const changeMonth = (delta: number) => {
  const [year, month] = selectedMonth.value.split("-").map(Number);
  const next = new Date(year, month - 1 + delta, 1);
  const nextKey = `${next.getFullYear()}-${String(next.getMonth() + 1).padStart(2, "0")}`;
  // 不可进入未来月份
  if (nextKey > currentMonth) return;
  selectedMonth.value = nextKey;
};

const goCurrentMonth = () => {
  selectedMonth.value = currentMonth;
};

/** 本地今天，避免 FlowEditDialog 默认日期使用 UTC 造成跨日 */
const localToday = () => {
  const now = new Date();
  const pad = (n: number) => String(n).padStart(2, "0");
  return `${now.getFullYear()}-${pad(now.getMonth() + 1)}-${pad(now.getDate())}`;
};

const openCreateDialog = () => {
  dialogFormTitle.value = "新增流水";
  selectedFlow.value = { day: localToday() };
  showFlowEditDialog.value = true;
};

const openEditDialog = (flow: Flow) => {
  dialogFormTitle.value = "修改流水";
  selectedFlow.value = { ...flow };
  showFlowEditDialog.value = true;
};

const formatMoney = (value: number | string | null | undefined) =>
  new Intl.NumberFormat("zh-CN", {
    style: "currency",
    currency: "CNY",
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(Number(value || 0));

const flowTypePrefix = (flowType?: string) => {
  if (flowType === "收入") return "+";
  if (flowType === "支出") return "-";
  return "";
};

onMounted(loadOverview);
watch(selectedMonth, loadOverview);
</script>
