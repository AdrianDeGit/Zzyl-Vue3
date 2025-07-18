<template>
  <div class="app-container">
    <el-form
        :model="queryParams"
        ref="queryRef"
        :inline="true"
        v-show="showSearch"
        label-width="68px"
    >
      <el-form-item label="名称" prop="planName">
        <el-input
            v-model="queryParams.planName"
            placeholder="请输入名称"
            clearable
            @keyup.enter="handleQuery"
        />
      </el-form-item>
      <el-form-item label="状态" prop="status">
        <el-select
            v-model="queryParams.status"
            placeholder="请选择状态"
            clearable
        >
          <el-option
              v-for="dict in nursing_plan_status"
              :key="dict.value"
              :label="dict.label"
              :value="dict.value"
          />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery"
        >搜索</el-button
        >
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8" type="flex" justify="end">
      <el-col :span="1.5">
        <el-button
            type="primary"
            plain
            icon="Plus"
            @click="handleAdd"
        >新增护理计划</el-button
        >
      </el-col>
    </el-row>

    <el-table
        v-loading="loading"
        :data="planList"
        @selection-change="handleSelectionChange"
    >
      <el-table-column label="序号" type="index" width="50" />
      <el-table-column label="名称" align="center" prop="planName" />
      <el-table-column label="状态" align="center" prop="status">
        <template #default="scope">
          <el-tag :type="scope.row.status === 1 ? 'success' : 'danger'">
            {{ scope.row.status === 1 ? '启用' : '禁用' }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column
          label="创建时间"
          align="center"
          prop="createTime"
          width="180"
      >
        <template #default="scope">
          <span>{{
              parseTime(scope.row.createTime, '{y}-{m}-{d} {h}:{i}:{s}')
            }}</span>
        </template>
      </el-table-column>
      <el-table-column
          label="操作"
          align="center"
          fixed="right"
          width="280"
          class-name="small-padding fixed-width"
      >
        <template #default="scope">
          <el-button
              link
              type="primary"
              icon="Edit"
              :class="scope.row.count ? 'disabled' : ''"
              @click="handleUpdate(scope.row)"
          >修改</el-button
          >
          <el-button
              link
              type="primary"
              icon="Delete"
              :class="scope.row.count ? 'disabled' : ''"
              @click="handleDelete(scope.row)"
          >删除</el-button
          >
          <el-button
              link
              type="primary"
              icon="Delete"
              @click="handleLook(scope.row)"
          >查看</el-button
          >
          <el-button
              link
              type="primary"
              :icon="scope.row.status == 0 ? 'Unlock' : 'lock'"
              @click="handleEnable(scope.row)"
          >{{ scope.row.status == 1 ? '禁用' : '启用' }}</el-button
          >
        </template>
      </el-table-column>
    </el-table>

    <pagination
        v-show="total > 0"
        :total="total"
        v-model:page="queryParams.pageNum"
        v-model:limit="queryParams.pageSize"
        @pagination="getNursingPlanList"
    />

    <!-- //////////////////////////////// -->
    <el-dialog
        :title="title"
        v-model="dialogVisible"
        width="840"
        @close="cancel()"
    >
      <el-form
          ref="planRef"
          :model="formData"
          :rules="rules"
          label-width="120px"
      >
        <el-row gutter="10">
          <el-col :span="24" class="elcolFlex">
            <el-form-item label="护理计划名称：" prop="planName">
              <el-input
                  v-model="formData.planName"
                  :disabled="isLook"
              ></el-input>
            </el-form-item>
            <el-form-item label="状态：" prop="status">
              <el-radio-group v-model="formData.status" :disabled="isLook">
                <el-radio
                    v-for="dict in nursing_plan_status"
                    :value="dict.value"
                    :label="dict.value"
                    :key="dict.value"
                >{{ dict.label }}</el-radio
                >
              </el-radio-group>
            </el-form-item>
            <el-form-item label="排序：" prop="sortNo">
              <el-input-number
                  :disabled="isLook"
                  v-model="formData.sortNo"
                  :min="0"
                  large-number
                  :max="999999"
                  :decimal-places="0"
                  @blur="textBlurNo"
                  @change="textBlurNo"
              ></el-input-number>
            </el-form-item>
          </el-col>
        </el-row>

        <el-form-item label="护理项目：" prop="price">
          <div class="info family">
            <div class="tableHead">
              <div>护理项目名称</div>
              <div>期望服务时间</div>
              <div>执行周期</div>
              <div>执行频次(次)</div>
              <div v-if="!isLook">操作</div>
            </div>
            <div class="tableBody">
              <div
                  class="tableColumn"
                  v-for="(item, index) in nursingPlanList"
                  :key="index"
              >
                <div class="column">
                  <el-select
                      :disabled="isLook"
                      v-model="item.projectId"
                      placeholder="请选择"
                  >
                    <el-option
                        v-for="item in nursingProjectOptions"
                        :key="item.value"
                        :label="item.label"
                        :value="item.value"
                    ></el-option>
                  </el-select>
                </div>
                <div class="column">
                  <el-time-picker
                      v-model="item.executeTime"
                      format="HH:mm:ss"
                      value-format="HH:mm:ss"
                      :style="{ width: '100%' }"
                      placeholder="请选择时间选择"
                      :disabled="isLook"
                      clearable
                  ></el-time-picker>
                </div>
                <div class="column">
                  <el-select
                      v-model="item.executeCycle"
                      placeholder="请选择"
                      :disabled="isLook"
                  >
                    <el-option
                        v-for="item in executeCycleOptions"
                        :key="item.value"
                        :label="item.label"
                        :value="item.value"
                    ></el-option>
                  </el-select>
                </div>
                <div class="column">
                  <el-input-number
                      v-model="item.executeFrequency"
                      :controls="false"
                      :max="7"
                      :min="1"
                      :disabled="isLook"
                  />
                </div>
                <div class="column" v-if="!isLook">
                  <el-icon
                      v-if="nursingPlanList.length > 1"
                      @click="handleRowDel(item, index)"
                      class="delect"
                  ><Minus
                  /></el-icon>
                  <el-icon @click="handleRowAdd" class="add"><Plus /></el-icon>
                </div>
              </div>
            </div>
          </div>
        </el-form-item>
      </el-form>
      <div class="dialog-footer" v-if="!isLook">
        <el-button type="primary" @click="submitForm">确定</el-button>
        <el-button @click="cancel()">取消</el-button>
      </div>
    </el-dialog>
    <!-- //////////////////////////////// -->
  </div>
</template>

<script setup name="Plan">
import {
  listPlan,
  getPlan,
  delPlan,
  addPlan,
  updatePlan,
  planStatus,
} from '@/api/serve/plan';
import { getAll } from '@/api/serve/project';
import { onMounted } from 'vue';

const { proxy } = getCurrentInstance();
const { nursing_plan_status } = proxy.useDict('nursing_plan_status');
const queryRef = ref(); // 表单
const planList = ref([]);
const open = ref(false);
const loading = ref(true);
const showSearch = ref(true);
const ids = ref([]);
const single = ref(true);
const multiple = ref(true);
const total = ref(0);
const title = ref('');
const dialogVisible = ref(false);
const nursingProjectOptions = ref([]);
const isLook = ref(false);
const formData = ref({
  status: '1',
  sortNo: 1,
});
// const planRef = ref({})

const queryParams = ref({
  pageNum: 1,
  pageSize: 10,
  planName: null,
  status: null,
});

const rules = ref({
  planName: [
    {
      required: true,
      message: '护理计划名称为空，请输入护理计划名称',
      trigger: 'blur',
    },
  ],
  status: [
    {
      required: true,
      message: '状态为空，请选择状态',
      trigger: 'change',
    },
  ],
  sortNo: [
    {
      required: true,
      message: '排序为空，请选择排序',
      trigger: 'change',
    },
  ],
});

const executeCycleOptions = ref([
  { label: '天', value: '0' },
  { label: '周', value: '1' },
  { label: '月', value: '2' },
]);

onMounted(() => {
  getAllProjectList();
});

//查询所有护理项目
const getAllProjectList = () => {
  getAll().then((res) => {
    nursingProjectOptions.value = res.data;
  });
};

const nursingPlanList = ref([
  {
    projectId: '',
    executeTime: '',
    executeCycle: '',
    executeFrequency: '',
  },
]);

const handleRowAdd = () => {
  const obj = {
    projectId: '',
    executeTime: '',
    executeCycle: '',
    executeFrequency: '',
  };
  nursingPlanList.value.push(obj);
};
//删除行数据
const handleRowDel = (item, index) => {
  if (nursingPlanList.value.length === 1) return;
  nursingPlanList.value.splice(index, 1);
};

/** 查询护理计划列表 */
function getNursingPlanList() {
  loading.value = true;
  listPlan(queryParams.value).then((response) => {
    planList.value = response.rows;
    total.value = response.total;
    loading.value = false;
  });
}

// 取消按钮
function cancel() {
  dialogVisible.value = false;
  isLook.value = false;
  reset();
  // 清除表单校验状态
  proxy.$refs['planRef']?.clearValidate();
}

// 表单重置
function reset() {
  formData.value = {
    planName: null,
    status: '1',
    sortNo: 1,
  };
  nursingPlanList.value = [
    {
      projectId: '',
      executeTime: '',
      executeCycle: '',
      executeFrequency: '1',
    },
  ];
  console.log(nursingPlanList.value);
  // 重置表单校验状态
  proxy.resetForm('planRef');
  // 清除所有校验状态
  proxy.$refs['planRef']?.clearValidate();
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1;
  getNursingPlanList();
}

/** 重置按钮操作 */
function resetQuery() {
  dialogVisible.value = false;
  handleQuery();
}

// 多选框选中数据
function handleSelectionChange(selection) {
  ids.value = selection.map((item) => item.id);
  single.value = selection.length != 1;
  multiple.value = !selection.length;
}

/** 新增按钮操作 */
function handleAdd() {
  reset(); // 先重置表单
  // 重新获取护理项目列表，确保选项是最新的
  getAllProjectList();
  dialogVisible.value = true;
  title.value = '新增护理计划';
}
/** 修改按钮操作 */
function handleUpdate(row) {
  const _id = row.id || ids.value;
  getDetails(_id);
}
// 获取详情
const getDetails = (id) => {
  // 先获取护理项目列表，确保选项是最新的
  getAllProjectList();
  getPlan(id).then((response) => {
    formData.value = response.data;
    formData.value.status = String(formData.value.status);
    nursingPlanList.value = formData.value.projectPlans;
    dialogVisible.value = true;
    // 根据是否为查看模式设置标题
    title.value = isLook.value ? '查看护理计划' : '修改护理计划';
  });
};
/** 提交按钮 */
function submitForm() {
  //判断选择的护理项目是否有重复的
  if (hasDuplicateIds(nursingPlanList.value)) {
    // 提示重复
    proxy.$modal.msgError('请勿选择重复的护理项目');
    return;
  }
  formData.value['projectPlans'] = nursingPlanList.value;
  console.log(formData.value);

  proxy.$refs['planRef'].validate((valid) => {
    if (valid) {
      if (formData.value.id != null) {
        updatePlan(formData.value).then((response) => {
          proxy.$modal.msgSuccess('修改成功');
          cancel();
          getNursingPlanList();
        });
      } else {
        addPlan(formData.value).then((response) => {
          proxy.$modal.msgSuccess('新增成功');
          cancel();
          getNursingPlanList();
        });
      }
    }
  });
}

//帮我使用js代码，编写一段逻辑，判断数组中的对象的id是否有相同的(问心一言提供)
function hasDuplicateIds(objectsArray) {
  // 如果没有提供数组或数组为空，则直接返回false（因为没有元素可比较）
  if (!Array.isArray(objectsArray) || objectsArray.length === 0) {
    return false;
  }

  // 创建一个新的Set来存储id
  const idSet = new Set();

  // 遍历数组中的每个对象
  for (const obj of objectsArray) {
    // 假设每个对象都有一个id属性
    if (obj.projectId !== undefined) {
      // 尝试将id新增到Set中
      idSet.add(obj.projectId);
    }
  }

  // 比较Set的大小和数组的大小
  // 如果Set的大小小于数组的大小，说明有重复的id
  return idSet.size !== objectsArray.length;
}

// 使用 async/await 语法优化异步操作
const handleEnable = async (row) => {
  try {
    // 获取状态
    const status = row.status;
    const info = status === 0 ? '启用' : '禁用';

    // 使用模板字符串
    const confirmMessage = `是否确认${info}护理项目的数据项？`;

    // 确认操作
    if (await proxy.$modal.confirm(confirmMessage)) {
      // 更新参数
      const param = {
        id: row.id,
        status: status === 0 ? 1 : 0,
      };

      // 执行更新操作
      await planStatus(param);
      // 刷新列表
      getNursingPlanList();
      // 成功消息
      proxy.$modal.msgSuccess(`${info}成功`);
    }
  } catch (error) {
    // 异常处理：这里可以根据实际需求进行调整，例如打印错误日志或显示用户友好的错误信息
    console.error('操作失败，请重试或联系管理员。');
  }
};

/** 删除按钮操作 */
function handleDelete(row) {
  const _ids = row.id || ids.value;
  proxy.$modal
      .confirm('是否确认删除护理计划编号为"' + _ids + '"的数据项？')
      .then(function () {
        return delPlan(_ids);
      })
      .then(() => {
        getNursingPlanList();
        proxy.$modal.msgSuccess('删除成功');
      })
      .catch(() => {});
}
//监听排序
const textBlurNo = () => {
  const data = formData.value.sortNo;
  if (data <= 1) {
    formData.value.sortNo = 1;
  }
  // 最大值设置
  if (data > 99999999) {
    formData.value.sortNo = 99999999;
  }
};
// 查看
const handleLook = (row) => {
  isLook.value = true;
  dialogVisible.value = true;
  title.value = '查看护理计划';
  getDetails(row.id);
};
getNursingPlanList();
</script>
<style src="./index.scss" lang="scss"></style>
