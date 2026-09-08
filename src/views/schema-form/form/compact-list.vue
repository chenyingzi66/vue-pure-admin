<script setup lang="ts">
import { computed, reactive, ref, type Component } from "vue";
import { ElMessage } from "element-plus";
import Refresh from "~icons/ri/refresh-line";
import AddLine from "~icons/ri/add-line";
import GiftFill from "~icons/ri/gift-2-fill";
import CouponFill from "~icons/ri/coupon-3-fill";
import MoneyBoxFill from "~icons/ri/money-dollar-box-fill";
import MoneyCircleFill from "~icons/ri/money-dollar-circle-fill";

defineOptions({ name: "CompactRewardList" });

type RewardRow = {
  id: number;
  icon: Component;
  tone: "blue" | "purple" | "green";
  name: string;
  type: string;
  actualReward: string;
  amount: string;
  points: string;
  turnover: string;
  exchanged: number;
  sort: number;
  enabled: boolean;
  vip: string;
};

const baseRows: Omit<RewardRow, "id">[] = [
  {
    icon: GiftFill,
    tone: "blue",
    name: "新人长期奖励",
    type: "现金",
    actualReward: "长期活动奖励",
    amount: "1 BDT",
    points: "1.0000",
    turnover: "1.00",
    exchanged: 0,
    sort: 1,
    enabled: false,
    vip: "全部"
  },
  {
    icon: CouponFill,
    tone: "purple",
    name: "周末限定奖励",
    type: "优惠券",
    actualReward: "满减优惠券",
    amount: "5 BDT",
    points: "5.0000",
    turnover: "1.00",
    exchanged: 12,
    sort: 2,
    enabled: false,
    vip: "VIP0、VIP1"
  },
  {
    icon: MoneyBoxFill,
    tone: "green",
    name: "现金兑换活动",
    type: "现金",
    actualReward: "现金兑换奖励",
    amount: "10 BDT",
    points: "100.0000",
    turnover: "1.00",
    exchanged: 1011,
    sort: 3,
    enabled: true,
    vip: "全部"
  },
  {
    icon: MoneyCircleFill,
    tone: "green",
    name: "高等级会员专享",
    type: "现金",
    actualReward: "会员等级奖励",
    amount: "999999 BDT",
    points: "99999999.0000",
    turnover: "2.00",
    exchanged: 36,
    sort: 4,
    enabled: true,
    vip: "VIP0、VIP1、VIP2"
  }
];

function createRows(): RewardRow[] {
  return Array.from({ length: 18 }, (_, index) => {
    const row = baseRows[index % baseRows.length];
    return {
      ...row,
      id: index + 1,
      name: index < baseRows.length ? row.name : row.name + " " + (index + 1),
      sort: index + 1
    };
  });
}

const activePane = ref("exchange");
const rows = ref(createRows());
const query = reactive({ name: "", status: "" });
const appliedQuery = reactive({ name: "", status: "" });
const pagination = reactive({ current: 1, pageSize: 20 });

const filteredRows = computed(() => {
  return rows.value.filter(row => {
    const matchName =
      !appliedQuery.name ||
      row.name.toLowerCase().includes(appliedQuery.name.toLowerCase());
    const matchStatus =
      appliedQuery.status === "" ||
      row.enabled === (appliedQuery.status === "enabled");
    return matchName && matchStatus;
  });
});

const pageRows = computed(() => {
  const start = (pagination.current - 1) * pagination.pageSize;
  return filteredRows.value.slice(start, start + pagination.pageSize);
});

function search() {
  appliedQuery.name = query.name.trim();
  appliedQuery.status = query.status;
  pagination.current = 1;
}

function reset() {
  query.name = "";
  query.status = "";
  search();
}

function addReward() {
  rows.value.unshift({
    ...baseRows[0],
    id: Date.now(),
    name: "新建奖励",
    sort: 1,
    enabled: true
  });
  rows.value.forEach((row, index) => {
    row.sort = index + 1;
  });
  pagination.current = 1;
  ElMessage.success("已新增一条示例数据");
}

function sortRows() {
  rows.value = [...rows.value].sort((a, b) => a.sort - b.sort);
  ElMessage.success("已按排序值升序排列");
}

function refreshRows() {
  rows.value = createRows();
  reset();
  ElMessage.success("列表已刷新");
}

function showAction(action: string, row: object) {
  ElMessage.info(action + "：" + (row as RewardRow).name);
}
</script>

<template>
  <div class="compact-list-demo">
    <el-tabs v-model="activePane" class="inner-tabs">
      <el-tab-pane label="兑换配置" name="exchange">
        <div class="filter-bar">
          <div class="filter-item">
            <span class="filter-label">奖励名称</span>
            <el-input
              v-model="query.name"
              size="small"
              clearable
              placeholder="请输入奖励名称"
              @keyup.enter="search"
            />
          </div>
          <div class="filter-item">
            <span class="filter-label">状态</span>
            <el-select
              v-model="query.status"
              size="small"
              clearable
              placeholder="请选择状态"
            >
              <el-option label="启用" value="enabled" />
              <el-option label="禁用" value="disabled" />
            </el-select>
          </div>
          <el-button type="primary" size="small" @click="search">
            搜索
          </el-button>
          <el-button size="small" @click="reset">重置</el-button>
        </div>

        <div class="table-toolbar">
          <el-button
            type="primary"
            size="small"
            :icon="AddLine"
            @click="addReward"
          >
            新增
          </el-button>
          <div class="toolbar-right">
            <el-button type="warning" size="small" @click="sortRows">
              排序
            </el-button>
            <el-button
              circle
              size="small"
              :icon="Refresh"
              aria-label="刷新列表"
              @click="refreshRows"
            />
          </div>
        </div>

        <el-table
          :data="pageRows"
          border
          size="small"
          height="230"
          class="reward-table"
        >
          <el-table-column label="任务图片" width="86" align="center">
            <template #default="{ row }">
              <div :class="['reward-icon', 'is-' + row.tone]">
                <IconifyIconOffline :icon="row.icon" />
              </div>
            </template>
          </el-table-column>
          <el-table-column
            prop="name"
            label="奖励名称"
            width="140"
            show-overflow-tooltip
          />
          <el-table-column
            prop="type"
            label="奖励类型"
            width="92"
            align="center"
          />
          <el-table-column
            prop="actualReward"
            label="实际奖励"
            width="140"
            show-overflow-tooltip
          />
          <el-table-column
            prop="amount"
            label="奖励(金额/数量)"
            width="130"
            align="center"
          />
          <el-table-column
            prop="points"
            label="兑换积分"
            width="112"
            align="center"
            sortable
          />
          <el-table-column
            prop="turnover"
            label="流水倍数"
            width="96"
            align="center"
          />
          <el-table-column
            prop="exchanged"
            label="已兑换数量"
            width="112"
            align="center"
            sortable
          />
          <el-table-column
            prop="sort"
            label="排序"
            width="84"
            align="center"
            sortable
          />
          <el-table-column label="状态" width="100" align="center" sortable>
            <template #default="{ row }">
              <el-switch
                v-model="row.enabled"
                size="small"
                inline-prompt
                active-text="启用"
                inactive-text="禁用"
                style="

                  --el-switch-on-color: var(--el-color-primary);
                  --el-switch-off-color: var(--el-fill-color-darker);
                "
              />
            </template>
          </el-table-column>
          <el-table-column
            prop="vip"
            label="VIP等级"
            width="138"
            align="center"
            show-overflow-tooltip
          />
          <el-table-column
            label="操作"
            width="112"
            align="center"
            fixed="right"
          >
            <template #default="{ row }">
              <el-button
                link
                type="primary"
                size="small"
                @click="showAction('查看', row)"
              >
                查看
              </el-button>
              <el-button
                link
                type="warning"
                size="small"
                @click="showAction('编辑', row)"
              >
                编辑
              </el-button>
            </template>
          </el-table-column>
        </el-table>

        <div class="pagination-row">
          <el-pagination
            v-model:current-page="pagination.current"
            v-model:page-size="pagination.pageSize"
            size="small"
            background
            :page-sizes="[10, 20, 50]"
            :total="filteredRows.length"
            layout="total, sizes, prev, pager, next, jumper"
          />
        </div>
      </el-tab-pane>

      <el-tab-pane label="全局设置" name="global">
        <el-empty description="全局设置示例区域" :image-size="72" />
      </el-tab-pane>
    </el-tabs>
  </div>
</template>

<style scoped>
.compact-list-demo {
  width: 100%;
  padding: 0 2px;
}

.inner-tabs :deep(.el-tabs__header) {
  margin-bottom: 18px;
}

.inner-tabs :deep(.el-tabs__item) {
  height: 38px;
  padding: 0 20px;
  font-size: 14px;
  font-weight: 600;
}

.filter-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 10px 12px;
  align-items: center;
  padding: 4px 0 12px;
}

.filter-item {
  display: flex;
  gap: 8px;
  align-items: center;
}

.filter-item:first-child,
.filter-item:nth-child(2) {
  width: 250px;
}

.filter-label {
  flex: none;
  font-size: 13px;
  color: var(--el-text-color-regular);
}

.table-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.toolbar-right {
  display: flex;
  gap: 8px;
}

.reward-table {
  width: 100%;
}

.reward-table :deep(.el-table__header th) {
  height: 38px;
  padding: 0;
  font-weight: 600;
  color: var(--el-text-color-primary);
  background: var(--el-fill-color-extra-light);
}

.reward-table :deep(.el-table__row td) {
  height: 48px;
  padding: 0;
}

.reward-table :deep(.cell) {
  padding: 0 10px;
  line-height: 20px;
}

.reward-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 38px;
  height: 38px;
  font-size: 25px;
  border-radius: 8px;
}

.reward-icon.is-blue {
  color: #477bdc;
  background: #eaf2ff;
}

.reward-icon.is-purple {
  color: #8b5cf6;
  background: #f1ebff;
}

.reward-icon.is-green {
  color: #41a846;
  background: #eaf7e9;
}

.pagination-row {
  display: flex;
  justify-content: flex-end;
  padding-top: 14px;
}

@media (width <= 720px) {
  .filter-item,
  .filter-item:first-child,
  .filter-item:nth-child(2) {
    width: 100%;
  }

  .filter-item :deep(.el-input),
  .filter-item :deep(.el-select) {
    flex: 1;
  }

  .pagination-row {
    overflow-x: auto;
  }
}
</style>
