<script setup lang="ts">
import { reactive, ref } from "vue";
import { ElMessage, type FormInstance, type FormRules } from "element-plus";

defineOptions({ name: "CompactDialogForm" });

const visible = ref(false);
const submitting = ref(false);
const formRef = ref<FormInstance>();

const form = reactive({
  memberScope: "all",
  rewardName: "",
  periodType: "long",
  periodRange: [],
  publishNow: true,
  rewardType: "cash",
  rewardAmount: 0,
  turnoverMultiple: 0,
  exchangePoints: 0,
  userDailyLimit: 0,
  dailyLimit: 0,
  totalLimit: 0,
  sort: 0
});

const rules = reactive<FormRules>({
  memberScope: [
    { required: true, message: "请选择参与会员", trigger: "change" }
  ],
  rewardName: [{ required: true, message: "请输入奖励名称", trigger: "blur" }],
  rewardType: [{ required: true, message: "请选择奖励类型", trigger: "change" }]
});

async function submitForm() {
  if (!formRef.value) return;
  const valid = await formRef.value.validate().catch(() => false);
  if (!valid) return;

  submitting.value = true;
  window.setTimeout(() => {
    submitting.value = false;
    visible.value = false;
    ElMessage.success("奖励配置已提交");
  }, 500);
}
</script>

<template>
  <div class="compact-dialog-demo">
    <div class="demo-intro">
      <div>
        <p class="demo-title">紧凑型弹框表单</p>
        <p class="demo-desc">小号控件、分组表格布局，字段标签不显示冒号。</p>
      </div>
      <el-button type="primary" size="small" @click="visible = true">
        打开示例
      </el-button>
    </div>

    <el-dialog
      v-model="visible"
      title="新增奖励配置"
      width="780px"
      class="compact-config-dialog"
      append-to-body
      destroy-on-close
      draggable
    >
      <el-form
        ref="formRef"
        :model="form"
        :rules="rules"
        label-width="128px"
        label-position="right"
        label-suffix=""
        size="small"
      >
        <section class="compact-section">
          <div class="section-title">基本信息</div>
          <div class="compact-grid">
            <el-form-item label="参与会员" prop="memberScope" class="span-2">
              <el-radio-group v-model="form.memberScope">
                <el-radio value="all">全部会员</el-radio>
                <el-radio value="level">指定玩家等级</el-radio>
                <el-radio value="group">指定玩家分组</el-radio>
              </el-radio-group>
            </el-form-item>
            <el-form-item label="奖励名称" prop="rewardName" class="span-2">
              <el-input
                v-model="form.rewardName"
                maxlength="30"
                placeholder="请输入奖励名称"
              />
            </el-form-item>
          </div>
        </section>

        <section class="compact-section">
          <div class="section-title">奖励配置</div>
          <div class="compact-grid">
            <el-form-item label="奖励时间" class="span-2">
              <div class="period-row">
                <el-radio-group v-model="form.periodType">
                  <el-radio-button value="long">长期</el-radio-button>
                  <el-radio-button value="custom">自定义</el-radio-button>
                </el-radio-group>
                <el-date-picker
                  v-if="form.periodType === 'custom'"
                  v-model="form.periodRange"
                  type="datetimerange"
                  start-placeholder="开始时间"
                  end-placeholder="结束时间"
                  range-separator="至"
                />
                <el-input v-else model-value="长期有效" disabled />
              </div>
            </el-form-item>

            <el-form-item label="创建后立即上架" class="span-2">
              <el-switch v-model="form.publishNow" />
            </el-form-item>

            <el-form-item label="奖励类型" prop="rewardType" class="span-2">
              <el-select v-model="form.rewardType" placeholder="请选择奖励类型">
                <el-option label="现金" value="cash" />
                <el-option label="积分" value="points" />
                <el-option label="优惠券" value="coupon" />
              </el-select>
            </el-form-item>

            <el-form-item label="奖励金额">
              <el-input-number
                v-model="form.rewardAmount"
                :min="0"
                :precision="2"
                :controls="false"
              />
            </el-form-item>
            <el-form-item label="流水倍数">
              <el-input-number
                v-model="form.turnoverMultiple"
                :min="0"
                :precision="1"
              />
            </el-form-item>
            <el-form-item label="兑换积分">
              <el-input-number
                v-model="form.exchangePoints"
                :min="0"
                :controls="false"
              />
            </el-form-item>
            <el-form-item label="单个用户当日上限">
              <el-input-number
                v-model="form.userDailyLimit"
                :min="0"
                :controls="false"
              />
            </el-form-item>
            <el-form-item label="当日兑换上限">
              <el-input-number
                v-model="form.dailyLimit"
                :min="0"
                :controls="false"
              />
            </el-form-item>
            <el-form-item label="累计兑换上限">
              <el-input-number
                v-model="form.totalLimit"
                :min="0"
                :controls="false"
              />
            </el-form-item>
            <el-form-item label="排序">
              <el-input-number v-model="form.sort" :min="0" :controls="false" />
            </el-form-item>
          </div>
        </section>
      </el-form>

      <template #footer>
        <el-button size="small" @click="visible = false">关闭</el-button>
        <el-button
          type="primary"
          size="small"
          :loading="submitting"
          @click="submitForm"
        >
          提交
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<style scoped>
.compact-dialog-demo {
  max-width: 780px;
}

.demo-intro {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 18px;
  background: var(--el-fill-color-extra-light);
  border: 1px solid var(--el-border-color-light);
  border-radius: 6px;
}

.demo-title {
  margin: 0 0 4px;
  font-size: 14px;
  font-weight: 600;
  color: var(--el-text-color-primary);
}

.demo-desc {
  margin: 0;
  font-size: 12px;
  color: var(--el-text-color-secondary);
}
</style>

<style>
.compact-config-dialog {
  max-width: calc(100vw - 32px);
  border-radius: 6px;
}

.compact-config-dialog .el-dialog__header {
  padding: 13px 16px;
  margin-right: 0;
  border-bottom: 1px solid var(--el-border-color-lighter);
}

.compact-config-dialog .el-dialog__title {
  font-size: 15px;
  font-weight: 600;
}

.compact-config-dialog .el-dialog__headerbtn {
  width: 44px;
  height: 44px;
}

.compact-config-dialog .el-dialog__body {
  max-height: 68vh;
  padding: 12px 16px;
  overflow-y: auto;
}

.compact-config-dialog .el-dialog__footer {
  padding: 10px 16px;
  border-top: 1px solid var(--el-border-color-lighter);
}

.compact-config-dialog .compact-section + .compact-section {
  margin-top: 12px;
}

.compact-config-dialog .section-title {
  padding: 7px 10px;
  font-size: 13px;
  font-weight: 600;
  color: var(--el-text-color-primary);
  background: var(--el-fill-color-light);
  border: 1px solid var(--el-border-color-lighter);
  border-bottom: 0;
}

.compact-config-dialog .compact-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  border-top: 1px solid var(--el-border-color-lighter);
  border-left: 1px solid var(--el-border-color-lighter);
}

.compact-config-dialog .compact-grid .span-2 {
  grid-column: 1 / -1;
}

.compact-config-dialog .compact-grid .el-form-item {
  display: flex;
  min-width: 0;
  min-height: 42px;
  margin: 0;
  border-right: 1px solid var(--el-border-color-lighter);
  border-bottom: 1px solid var(--el-border-color-lighter);
}

.compact-config-dialog .compact-grid .el-form-item__label {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  height: auto;
  min-height: 41px;
  padding: 0 10px;
  line-height: 18px;
  color: var(--el-text-color-regular);
  background: var(--el-fill-color-extra-light);
  border-right: 1px solid var(--el-border-color-lighter);
}

.compact-config-dialog .compact-grid .el-form-item__content {
  min-width: 0;
  min-height: 41px;
  padding: 5px 8px;
  line-height: 30px;
}

.compact-config-dialog .compact-grid .el-form-item__error {
  inset: auto 10px 0 auto;
  padding: 0;
  font-size: 11px;
  line-height: 14px;
}

.compact-config-dialog .el-select,
.compact-config-dialog .el-input-number,
.compact-config-dialog .el-date-editor {
  width: 100%;
}

.compact-config-dialog .period-row {
  display: flex;
  gap: 8px;
  width: 100%;
}

.compact-config-dialog .period-row .el-radio-group {
  flex: none;
}

.compact-config-dialog .el-radio {
  margin-right: 20px;
}

@media (width <= 720px) {
  .compact-config-dialog .compact-grid {
    grid-template-columns: 1fr;
  }

  .compact-config-dialog .compact-grid .span-2 {
    grid-column: auto;
  }

  .compact-config-dialog .period-row {
    flex-direction: column;
  }
}
</style>
