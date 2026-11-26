<!--
  参考实现（不会被 Nuxt 编译）
  目标位置：app/pages/user/overview.client.vue
  依赖：现有 FlowsStatistics、ChartsCommonPie、MonthBar、FlowEditDialog、doApi、showFlowEditDialog。
  说明：本文件用于指导开发节点按仓库现有 Nuxt 4 + Vue 3 + Tailwind 语义色实现，不新增接口、表或迁移。
-->
<template>
  <div class="p-2 md:p-4 min-h-full space-y-3">
    <div class="flex flex-col gap-3 md:flex-row md:items-end md:justify-between">
      <div>
        <p class="text-xs font-semibold tracking-widest text-primary-600">PERSONAL LEDGER</p>
        <h1 class="text-2xl font-bold text-foreground md:text-3xl">账本概览</h1>
        <p class="mt-1 text-sm text-muted">一眼看清本月收支、预算与消费方向</p>
      </div>

      <div class="flex flex-wrap items-center gap-2">
        <div class="flex items-center gap-1 rounded-full border border-border bg-surface px-2 py-1 shadow-sm">
          <button
            type="button"
            class="rounded-full p-1 text-foreground/60 hover:bg-surface-muted"
            aria-label="上个月"
            @click="changeMonth(-1)"
          >
            <ChevronLeftIcon class="h-4 w-4" />
          </button>
          <span class="px-2 text-sm font-medium">{{ monthLabel }}</span>
          <button
            type="button"
            class="rounded-full p-1 text-foreground/60 hover:bg-surface-muted disabled:opacity-30"
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
          class="px-2 py-1 text-xs font-medium text-primary-600"
          @click="goCurrentMonth"
        >
          回到本月
        </button>
        <button
          type="button"
          class="inline-flex items-center gap-1 rounded-md bg-primary-600 px-3 py-2 text-sm font-medium text-white shadow-sm transition-colors hover:bg-primary-500"
          @click="openCreateDialog"
        >
          <PlusIcon class="h-4 w-4" />
          记一笔
        </button>
      </div>
    </div>

    <FlowsStatistics :statistics="statistics" />

    <div class="grid grid-cols-1 gap-3 lg:grid-cols-2">
      <section class="rounded-lg border border-border bg-surface p-4 shadow-soft">
        <div class="mb-3 flex items-start justify-between gap-3">
          <div>
            <h2 class="font-semibold text-foreground">本月预算</h2>
            <p class="mt-1 text-xs text-muted">{{ monthLabel }} · 已用 {{ formatMoney(statistics.totalOut) }}</p>
          </div>
          <NuxtLink to="/user/budget" class="text-xs font-medium text-primary-600">调整</NuxtLink>
        </div>

        <template v-if="currentBudget">
          <div class="flex items-end justify-between gap-3">
            <p class="font-numeric text-2xl font-bold text-primary-600">
              {{ formatMoney(Math.abs(budgetRemaining)) }}
              <span class="text-xs font-medium text-muted">{{ budgetRemaining < 0 ? '已超支' : '剩余' }}</span>
            </p>
            <span class="rounded-full bg-primary-50 px-2 py-1 text-xs font-medium text-primary-600">
              {{ budgetPercent }}%
            </span>
          </div>
          <div class="mt-3 h-2 overflow-hidden rounded-full bg-surface-muted" role="progressbar" aria-label="预算使用进度" aria-valuemin="0" aria-valuemax="100" :aria-valuenow="Math.min(budgetPercent, 100)">
            <div
              class="h-full rounded-full"
              :class="budgetRemaining < 0 ? 'bg-red-500' : 'bg-primary-500'"
              :style="{ width: `${Math.min(budgetPercent, 100)}%` }"
            ></div>
          </div>
          <div class="mt-2 flex justify-between text-xs text-muted">
            <span>预算 <strong class="text-foreground">{{ formatMoney(currentBudget.budget) }}</strong></span>
            <span>已用 <strong class="text-foreground">{{ formatMoney(statistics.totalOut) }}</strong></span>
          </div>
        </template>
        <div v-else class="py-6 text-center">
          <p class="text-sm text-muted">未设置预算</p>
          <NuxtLink to="/user/budget" class="mt-2 inline-block text-xs font-medium text-primary-600">去设置预算</NuxtLink>
        </div>
      </section>

      <section class="rounded-lg border border-border bg-surface p-4 shadow-soft">
        <div class="mb-3 flex items-start justify-between gap-3">
          <div>
            <h2 class="font-semibold text-foreground">最近流水</h2>
            <p class="mt-1 text-xs text-muted">{{ monthLabel }} · 最近 {{ recentFlows.length }} 笔</p>
          </div>
          <NuxtLink to="/user/flows" class="text-xs font-medium text-primary-600">查看全部</NuxtLink>
        </div>

        <div v-if="recentFlows.length > 0" class="divide-y divide-border">
          <button
            v-for="flow in recentFlows"
            :key="flow.id"
            type="button"
            class="flex w-full items-center gap-3 py-2 text-left transition-colors hover:bg-surface-muted"
            @click="openEditDialog(flow)"
          >
            <span class="grid h-8 w-8 flex-shrink-0 place-items-center rounded-md bg-primary-50 text-xs font-semibold text-primary-600">
              {{ (flow.industryType || '其他').slice(0, 1) }}
            </span>
            <span class="min-w-0 flex-1">
              <span class="block truncate text-sm font-medium text-foreground">{{ flow.name || flow.industryType || '未命名流水' }}</span>
              <span class="block truncate text-xs text-muted">
                {{ flow.industryType || '未分类' }} · {{ flow.account?.name || '未关联账户' }} · {{ formatDay(flow.day) }}
              </span>
            </span>
            <span class="font-numeric flex-shrink-0 text-sm font-semibold" :class="flow.flowType === '收入' ? 'text-primary-600' : 'text-foreground'">
              {{ flow.flowType === '收入' ? '+' : flow.flowType === '支出' ? '-' : '' }}{{ formatMoney(flow.money) }}
            </span>
            <ChevronRightIcon class="h-4 w-4 flex-shrink-0 text-foreground/30" />
          </button>
        </div>
        <div v-else class="py-6 text-center">
          <p class="text-sm text-muted">本月还没有流水</p>
          <button type="button" class="mt-2 text-xs font-medium text-primary-600" @click="openCreateDialog">记一笔</button>
        </div>
      </section>

      <section class="rounded-lg border border-border bg-surface p-4 shadow-soft">
        <ChartsCommonPie
          title="支出构成"
          width="100%"
          height="260px"
          group-by="industryType"
          flow-type="支出"
          :start-day="monthRange.start"
          :end-day="monthRange.end"
          series-name="支出"
        />
      </section>

      <section class="rounded-lg border border-border bg-surface p-4 shadow-soft">
        <MonthBar title="近 6 个月趋势" width="100%" height="260px" />
      </section>
    </div>

    <FlowEditDialog
      v-if="showFlowEditDialog"
      :title="dialogFormTitle"
      :flow="selectedFlow"
      :success-callback="loadOverview"
    />
  </div>
</template>

<script setup lang="ts">
import { ChevronLeftIcon, ChevronRightIcon, PlusIcon } from '@heroicons/vue/24/outline';
import FlowsStatistics from '~/components/flows/FlowsStatistics.vue';
import FlowEditDialog from '~/components/dialog/FlowEditDialog.vue';
import MonthBar from '~/components/charts/MonthBar.vue';
import { doApi } from '~/utils/api';
import { showFlowEditDialog } from '~/utils/flag';

definePageMeta({
  layout: 'public',
  middleware: ['auth'],
});

const currentMonth = new Date().toISOString().slice(0, 7);
const selectedMonth = ref(currentMonth);
const loading = ref(false);
const flowPage = ref<any>({
  data: [],
  total: 0,
  totalIn: 0,
  totalOut: 0,
  notInOut: 0,
});
const budgets = ref<any[]>([]);
const dialogFormTitle = ref('新增流水');
const selectedFlow = ref<Record<string, any>>({});

const monthRange = computed(() => {
  const [year, month] = selectedMonth.value.split('-').map(Number);
  const lastDay = new Date(year, month, 0).getDate();
  return {
    start: `${selectedMonth.value}-01`,
    end: `${selectedMonth.value}-${String(lastDay).padStart(2, '0')}`,
  };
});

const monthLabel = computed(() => {
  const [year, month] = selectedMonth.value.split('-');
  return `${year}年${Number(month)}月`;
});

const isCurrentMonth = computed(() => selectedMonth.value === currentMonth);

const statistics = computed(() => ({
  totalIn: flowPage.value.totalIn || 0,
  totalOut: flowPage.value.totalOut || 0,
  notInOut: flowPage.value.notInOut || 0,
}));

const recentFlows = computed(() => (Array.isArray(flowPage.value.data) ? flowPage.value.data : []));

const currentBudget = computed(() =>
  budgets.value.find((item) => item.month === selectedMonth.value),
);

const budgetPercent = computed(() => {
  if (!currentBudget.value || !currentBudget.value.budget) return 0;
  return Math.round((statistics.value.totalOut / currentBudget.value.budget) * 100);
});

const budgetRemaining = computed(() => {
  if (!currentBudget.value) return 0;
  return Number((currentBudget.value.budget - statistics.value.totalOut).toFixed(2));
});

const loadOverview = async () => {
  loading.value = true;
  try {
    const [page, budgetList] = await Promise.all([
      doApi.post('api/entry/flow/page', {
        startDay: monthRange.value.start,
        endDay: monthRange.value.end,
        pageNum: 1,
        pageSize: 5,
      }),
      doApi.post('api/entry/budget/all', {}),
    ]);
    flowPage.value = page || {};
    budgets.value = Array.isArray(budgetList) ? budgetList : [];
  } finally {
    loading.value = false;
  }
};

const changeMonth = (delta: number) => {
  const [year, month] = selectedMonth.value.split('-').map(Number);
  const next = new Date(year, month - 1 + delta, 1);
  const nextKey = `${next.getFullYear()}-${String(next.getMonth() + 1).padStart(2, '0')}`;
  if (nextKey > currentMonth) return;
  selectedMonth.value = nextKey;
};

const goCurrentMonth = () => {
  selectedMonth.value = currentMonth;
};

const openCreateDialog = () => {
  dialogFormTitle.value = '新增流水';
  selectedFlow.value = {};
  showFlowEditDialog.value = true;
};

const openEditDialog = (flow: Record<string, any>) => {
  dialogFormTitle.value = '修改流水';
  selectedFlow.value = { ...flow };
  showFlowEditDialog.value = true;
};

const formatMoney = (value: number | string | null | undefined) =>
  `¥${Number(value || 0).toLocaleString('zh-CN', {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  })}`;

const formatDay = (day: string | Date | null | undefined) => {
  if (!day) return '';
  const date = typeof day === 'string' ? new Date(day) : day;
  return `${date.getMonth() + 1}月${date.getDate()}日`;
};

onMounted(loadOverview);
watch(selectedMonth, loadOverview);
</script>
