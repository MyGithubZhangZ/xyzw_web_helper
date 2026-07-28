<template>
  <!-- 手动输入表单 -->
  <NForm
    :model="importForm"
    label-placement="top"
    size="large"
    :show-label="true"
  >
    <NFormItem label="游戏角色名称" :show-label="true">
      <NInput
        v-model:value="importForm.name"
        placeholder="例如：主号战士"
        clearable
      />
    </NFormItem>

    <NFormItem label="bin文件" :show-label="true">
      <a-upload
        multiple
        accept="*.bin,*.dmp"
        @before-upload="uploadBin"
        draggable
        dropzone
        placeholder="粘贴Token字符串..."
        clearable
      >
        <!-- <div class="dropzone-content">
          请点击上传或将bind文件拖拽到此处
        </div> -->
      </a-upload>
    </NFormItem>
    <a-list>
      <a-list-item v-for="(role, roleIndex) in roleList" :key="roleIndex">
        <div>
          <strong>角色名称:</strong> {{ role.name || "未命名角色" }}<br />
          <strong>Token:</strong>
          <span style="word-break: break-all">{{ role.token }}</span
          ><br />
          <strong>服务器:</strong> {{ role.server || "未指定" }}
        </div>
      </a-list-item>
    </a-list>

    <!-- 角色详情 -->
    <NCollapse>
      <NCollapseItem title="角色详情 (可选)" name="optional">
        <div class="optional-fields">
          <NFormItem label="服务器">
            <NInput
              v-model:value="importForm.server"
              placeholder="服务器名称"
            />
          </NFormItem>

          <NFormItem label="自定义连接地址">
            <NInput
              v-model:value="importForm.wsUrl"
              placeholder="留空使用默认连接"
            />
          </NFormItem>
        </div>
      </NCollapseItem>
    </NCollapse>

    <div class="form-actions">
      <NButton
        type="primary"
        size="large"
        block
        :loading="isImporting"
        @click="handleImport"
      >
        <template #icon>
          <NIcon>
            <CloudUpload />
          </NIcon>
        </template>
        添加Token
      </NButton>

      <NButton v-if="tokenStore.hasTokens" size="large" block @click="cancel">
        取消
      </NButton>
    </div>
  </NForm>
</template>

<script lang="ts" setup>
import { CloudUpload } from "@vicons/ionicons5";
import {
  NButton,
  NCollapse,
  NCollapseItem,
  NForm,
  NFormItem,
  NIcon,
  NInput,
  useMessage,
} from "naive-ui";
import PQueue from "p-queue";

import { reactive, ref } from "vue";

import useIndexedDB from "@/hooks/useIndexedDB";
import { useTokenStore } from "@/stores/tokenStore";
import { getTokenId, transformToken } from "@/utils/token";

const $emit = defineEmits(["cancel", "ok"]);

const { storeArrayBuffer } = useIndexedDB();

const cancel = () => {
  roleList.value = [];
  $emit("cancel");
};

const tokenStore = useTokenStore();
const message = useMessage();
const isImporting = ref(false);
const importForm = reactive({
  name: "",
  server: "",
  wsUrl: "",
  importMethod: "",
});
const roleList = ref<
  Array<{
    id: string;
    name: string;
    token: string;
    server: string;
    wsUrl: string;
    importMethod: string;
  }>
>([]);

const tQueue = new PQueue({ concurrency: 1, interval: 1000 });

const initName = (fileName: string) => {
  if (!fileName) return;
  fileName = fileName.trim();
  const binRes = fileName.match(/^bin-(.*?)服-([0-2])-(\d{6,12})-(.*)\.bin$/);
  console.log(binRes);
      server: binRes[1],
      roleIndex: binRes[2],
      roleId: binRes[3],
      roleName: binRes[4],
      // StableId 生成：bin-服务器-索引-角色ID
      stableId: `bin-${binRes[1]}-${binRes[2]}-${binRes[3]}`,
    };
    server: "",
    roleIndex: "",
    roleId: "",
    roleName: importForm.name || "",
    stableId: fileName, // 回退：使用文件名本身
  };
};
const uploadBin = (binFile: File) => {
  tQueue.add(async () => {
    console.log("上传文件数据:", binFile);
    const roleMeta = initName(binFile.name) as any;
    const reader = new FileReader();
    reader.onload = async (e) => {
      const userToken = e.target?.result as ArrayBuffer;
      // 根据管理模式选择不同的ID生成策略
      // StableId模式：使用文件名生成的stableId，相同文件名视为同一角色
      // Hash模式：使用文件内容Hash，内容相同视为同一角色
      // console.log('转换Token:', userToken);
      const tokenId = getTokenId(userToken);
      const saved = await storeArrayBuffer(tokenId, userToken);
      if (!saved) {
        message.error("保存BIN数据到IndexedDB失败");
        return;
      }

      // 上传列表中发现已存在的重复名称，提示消息
      if (roleList.value.some((role) => role.id === tokenId)) {
      
        return;
      }
      // 检查待上传的角色是否已在tokenStore中存在
      const existingToken = tokenStore.gameTokens.find((t) => t.id === tokenId);
      if (existingToken) {
        message.warning(`角色"${roleName}"已存在，将更新该角色的Token`);
      }
      message.success("Token读取成功，请检查角色名称等信息后提交");
      roleList.value.push({
        id: tokenId,
        token: roleToken,
        name: roleName,
        server: `${roleMeta.server}${roleMeta.roleIndex}` || "",
        wsUrl: importForm.wsUrl || "",
        importMethod: "bin",
      });
    };
    reader.onerror = () => {
      message.error("读取文件失败，请重试");
    };
    reader.readAsArrayBuffer(binFile);
  });
  return false; // 阻止自动上传
};

const handleImport = async () => {
  if (roleList.value.length === 0) {
    message.error("请先上传bin文件！");
    return;
  }
  roleList.value.forEach((role) => {
    // StableId模式下：记录旧token的分组关联，用于后续恢复
    let oldTokenGroups: string[] = [];
    if (binManageMode.value === 'stableId') {
      console.log("移除同名token:", gameToken);
      // tokenStore.removeToken(gameToken.id);
      tokenStore.updateToken(gameToken.id, {
        ...role,
      });
    } else {
      tokenStore.addToken({
        ...role,
      });
    }

    // StableId模式下：重新关联到原分组
    if (binManageMode.value === 'stableId' && oldTokenGroups.length > 0) {
  roleList.value = [];
  $emit("ok");
};
</script>

<style scoped lang="scss">
.optional-fields {
  display: flex;
  gap: 16px;
  flex-wrap: wrap;

  n-form-item {
    flex: 1;
    min-width: 200px;
  }
}

.form-actions {
  margin-top: 24px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.dropzone-content {
  width: 100%;
  border: 1px dashed #fcc;
  border-radius: 8px;
  text-align: center;
  color: #888;
  padding: 40px 20px;
  font-size: 12px;
}

/* BIN文件管理方式样式 */
.mode-selection {
{
  "name": "/TokenImport/singlebin",
  "path": "/TokenImport/singlebin"
}
</route>
