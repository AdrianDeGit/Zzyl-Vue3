<template>
  <div class="app-container">
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="名称" prop="name">
        <el-input
          v-model="queryParams.name"
          placeholder="请输入名称"
          clearable
          @keyup.enter="handleQuery"
        />
      </el-form-item>
      <el-form-item label="状态" prop="status">
        <el-select v-model="queryParams.status" placeholder="请选择状态" clearable>
          <el-option
              v-for="item in nursing_level_status"
              :key="item.value"
              :label="item.label"
              :value="item.value" />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
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
          v-hasPermi="['serve:level:add']"
        >新增护理等级</el-button>
      </el-col>
    </el-row>

    <el-table v-loading="loading" :data="levelList" @selection-change="handleSelectionChange">
      <el-table-column label="序号" type="index" align="center" width="50" />
      <el-table-column label="护理等级名称" align="center" prop="name" />
      <el-table-column label="执行护理计划" align="center" prop="lplanId" />
      <el-table-column label="护理费用(元/月)" align="center" prop="fee" />
      <el-table-column label="状态" align="center" prop="status" >
        <template #default="scope">
          <el-tag :type="scope.row.status === 1 ? 'success' : 'danger'">{{ scope.row.status === 1 ? '启用' : '禁用' }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column label="等级说明" align="center" prop="description" />
      <el-table-column label="创建时间" align="center" prop="createTime" width="180">
        <template #default="scope">
            <span>{{ parseTime(scope.row.createTime, '{y}-{m}-{d} {h}:{i}:{s}') }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" class-name="small-padding fixed-width" fixed="right" width="200">
        <template #default="scope">
          <el-button link type="primary" icon="Edit" @click="handleUpdate(scope.row)" v-hasPermi="['serve:level:edit']">修改</el-button>
          <el-button link type="primary" icon="Delete" @click="handleDelete(scope.row)" v-hasPermi="['serve:level:remove']">删除</el-button>
          <el-button link type="primary" :icon="scope.row.status === 0 ? 'Lock' : 'Unlock'" @click="handleEnable(scope.row)" >{{ scope.row.status === 0 ? '启用' : '禁用' }}</el-button>
        </template>
      </el-table-column>
    </el-table>
    
    <pagination
      v-show="total>0"
      :total="total"
      v-model:page="queryParams.pageNum"
      v-model:limit="queryParams.pageSize"
      @pagination="getList"
    />

    <!-- 添加或修改护理等级对话框 -->
    <el-dialog :title="title" v-model="open" width="500px" append-to-body>
      <el-form ref="levelRef" :model="form" :rules="rules" label-width="100px">
        <el-form-item label="等级名称" prop="name" required>
          <el-input v-model="form.name" placeholder="请输入等级名称" />
        </el-form-item>
        <el-form-item label="护理计划" prop="lplanId" required>
          <el-select
            v-model="form.lplanId"
            filterable
            clearable
            placeholder="请选择护理计划"
          >
            <el-option
              v-for="item in planOptions"
              :key="item.id"
              :label="item.planName"
              :value="item.id"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="护理费用" prop="fee" required>
          <el-input-number
            v-model="form.fee"
            :min="0"
            :step="1"
            placeholder="请输入"
          />
        </el-form-item>
        <el-form-item label="状态" prop="status" required>
          <el-radio-group v-model="form.status">
            <el-radio :label="1">启用</el-radio>
            <el-radio :label="0">禁用</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="等级说明" prop="description" required>
          <el-input v-model="form.description" placeholder="请输入等级说明" type="textarea" />
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <el-input v-model="form.remark" placeholder="请输入备注" />
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button type="primary" @click="submitForm">确 定</el-button>
          <el-button @click="cancel">取 消</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup name="Level">
import { listLevel, getLevel, delLevel, addLevel, updateLevel } from "@/api/serve/level"
import { listPlan } from "@/api/serve/plan";

const { proxy } = getCurrentInstance()

const levelList = ref([])
const open = ref(false)
const loading = ref(true)
const showSearch = ref(true)
const ids = ref([])
const single = ref(true)
const multiple = ref(true)
const total = ref(0)
const title = ref("")
const planOptions = ref([]);

const data = reactive({
  form: {
    name: null,
    lplanId: null,
    fee: null,
    status: 1,
    description: null,
    remark: null
  },
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    name: null,
    status: null,
  },
  rules: {
    name: [{ required: true, message: '请输入等级名称', trigger: 'blur' }],
    lplanId: [{ required: true, message: '请选择护理计划', trigger: 'change' }],
    fee: [
      { required: true, message: '请输入护理费用', trigger: 'blur' },
      { type: 'number', min: 0, message: '护理费用不能小于0', trigger: 'blur' }
    ],
    status: [{ required: true, message: '请选择状态', trigger: 'change' }],
    description: [{ required: true, message: '请输入等级说明', trigger: 'blur' }]
  }
})

const { queryParams, form, rules } = toRefs(data)

function fetchPlanOptions() {
  listPlan({ pageNum: 1, pageSize: 1000 }).then(res => {
    planOptions.value = res.rows || [];
  });
}

/** 查询护理等级列表 */
function getList() {
  loading.value = true
  listLevel(queryParams.value).then(response => {
    levelList.value = response.rows
    total.value = response.total
    loading.value = false
  })
}

// 取消按钮
function cancel() {
  open.value = false
  reset()
}

// 表单重置
function reset() {
  form.value = {
    name: null,
    lplanId: null,
    fee: null,
    status: 1,
    description: null,
    remark: null
  }
  fetchPlanOptions();
  proxy.resetForm("levelRef")
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1
  getList()
}

/** 重置按钮操作 */
function resetQuery() {
  proxy.resetForm("queryRef")
  handleQuery()
}

// 多选框选中数据
function handleSelectionChange(selection) {
  ids.value = selection.map(item => item.id)
  single.value = selection.length != 1
  multiple.value = !selection.length
}

/** 新增按钮操作 */
function handleAdd() {
  reset();
  fetchPlanOptions();
  open.value = true;
  title.value = "添加护理等级";
}

/** 修改按钮操作 */
function handleUpdate(row) {
  reset();
  fetchPlanOptions();
  const _id = row.id || ids.value;
  getLevel(_id).then(response => {
    form.value = response.data;
    open.value = true;
    title.value = "修改护理等级";
  });
}

/** 提交按钮 */
function submitForm() {
  proxy.$refs["levelRef"].validate(valid => {
    if (valid) {
      if (form.value.id != null) {
        updateLevel(form.value).then(response => {
          proxy.$modal.msgSuccess("修改成功")
          open.value = false
          getList()
        })
      } else {
        addLevel(form.value).then(response => {
          proxy.$modal.msgSuccess("新增成功")
          open.value = false
          getList()
        })
      }
    }
  })
}

/** 删除按钮操作 */
function handleDelete(row) {
  const _ids = row.id || ids.value
  proxy.$modal.confirm('是否确认删除护理等级序号为"' + _ids + '"的数据项？').then(function() {
    return delLevel(_ids)
  }).then(() => {
    getList()
    proxy.$modal.msgSuccess("删除成功")
  }).catch(() => {})
}

/** 导出按钮操作 */
function handleExport() {
  proxy.download('serve/level/export', {
    ...queryParams.value
  }, `level_${new Date().getTime()}.xlsx`)
}

getList()

//引用数据字典，定义的变量名称必须和字典类型保持一致
const { nursing_level_status } = proxy.useDict("nursing_level_status");

/* 启用禁用按钮操作 */
function handleEnable(row) {
  // 获取当前状态
  const status = row.status;
  // 提示信息
  const msg = status === 1 ? '禁用' : '启用';

  // 构建参数
  const params = {
    id: row.id,
    status: status === 1 ? 0 : 1
  };

  proxy.$modal.confirm(`是否确认${msg}该护理等级的数据项？`).then(function() {
    return updateLevel(params);
  }).then(() => {
    getList();
    proxy.$modal.msgSuccess(`${msg}成功`);
  }).catch(() => {});
}
</script>

<style scoped>
</style>
