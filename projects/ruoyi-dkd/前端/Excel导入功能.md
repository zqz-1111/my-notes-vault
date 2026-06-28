---
date: 2026-06-28
tags: [ruoyi-dkd, 前端, Vue3, ElementPlus]
---

# Excel 导入功能

## 一、写在前面

1. 为什么学这个：很多业务模块都需要 Excel 导入功能，这是一个通用能力
2. 学完你能掌握：el-upload 手动上传、Token 鉴权、文件类型/大小校验、上传成功/失败处理

## 二、正文

### 2.1 导入按钮

```html
<el-col :span="1.5">
  <el-button type="warning" plain icon="Upload" @click="handleExcelImport" v-hasPermi="['manage:sku:add']">导入</el-button>
</el-col>
```

### 2.2 导入对话框

```html
<el-dialog title="数据导入" v-model="excelOpen" width="400px" append-to-body>
  <el-upload ref="uploadRef" class="upload-demo"
    :action="uploadExcelUrl"
    :headers="headers"
    :on-success="handleUploadSuccess"
    :on-error="handleUploadError"
    :before-upload="handleBeforeUpload"
    :limit="1"
    :auto-upload="false">
    <template #trigger>
      <el-button type="primary">上传文件</el-button>
    </template>
    <el-button class="ml-3" type="success" @click="submitUpload">上传</el-button>
    <template #tip>
      <div class="el-upload__tip">上传文件仅支持，xls/xlsx格式，文件大小不得超过1M</div>
    </template>
  </el-upload>
</el-dialog>
```

### 2.3 脚本实现

```javascript
import { getToken } from "@/utils/auth";

const excelOpen = ref(false);
const uploadExcelUrl = ref(import.meta.env.VITE_APP_BASE_API + "/manage/sku/import");
const headers = ref({ Authorization: "Bearer " + getToken() });
const uploadRef = ref({});

function handleExcelImport() {
  excelOpen.value = true;
}

function submitUpload() {
  uploadRef.value.submit(); // 手动触发上传
}

// 上传前校验
function handleBeforeUpload(file) {
  // 校验文件类型
  const fileType = ["xls", "xlsx"];
  let fileExtension = file.name.slice(file.name.lastIndexOf(".") + 1);
  const isExcel = fileType.some(type => fileExtension.indexOf(type) > -1);
  if (!isExcel) {
    proxy.$modal.msgError(`文件格式不正确, 请上传${fileType.join("/")}格式文件!`);
    return false;
  }
  // 校验文件大小
  if (file.size / 1024 / 1024 >= 1) {
    proxy.$modal.msgError("上传excel大小不能超过 1 MB!");
    return false;
  }
  proxy.$modal.loading("正在上传excel，请稍候...");
}

function handleUploadSuccess(res) {
  if (res.code === 200) {
    proxy.$modal.msgSuccess("上传excel成功");
    excelOpen.value = false;
    getList(); // 刷新列表
  } else {
    proxy.$modal.msgError(res.msg);
  }
  uploadRef.value.clearFiles();
  proxy.$modal.closeLoading();
}

function handleUploadError() {
  proxy.$modal.msgError("上传excel失败");
  uploadRef.value.clearFiles();
  proxy.$modal.closeLoading();
}
```

## 三、踩坑避坑指南

**坑 1：忘记携带 Token**

el-upload 的 `:headers` 必须携带 Authorization Token，否则后端会返回 401。因为 el-upload 不经过 axios 拦截器，需要手动设置请求头。

**坑 2：自动上传 vs 手动上传**

- `auto-upload="false"` 禁止选择文件后自动上传
- 用户先选文件，点"上传"按钮才触发 `submitUpload()`
- 体验更好，给用户确认的机会

**坑 3：文件列表未清空**

上传成功或失败后都要 `clearFiles()`，否则下次打开对话框还有上次的文件。

**坑 4：handleBeforeUpload 返回值**

- 返回 `false` 阻止上传
- 返回 `true` 或不返回继续上传
- 返回一个 `Promise` 可以异步控制

## 四、复用指南

这个功能可以直接复制到其他需要 Excel 导入的模块，只需要修改：
1. `uploadExcelUrl` — 改为对应模块的导入接口地址
2. `v-hasPermi` — 改为对应模块的权限标识
3. 文件大小和类型限制按需调整

## 五、关联笔记

- [[projects/ruoyi-dkd/前端/前端概述]]
- [[projects/ruoyi-dkd/前端/商品管理模块改造]]

## 六、代码变更

- 修改：`src/views/manage/sku/index.vue` — 新增导入按钮、导入对话框、上传校验和回调函数
