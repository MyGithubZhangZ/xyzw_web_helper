<template>
  <!-- 手动输入表单 -->
  <n-form :model="importForm" :label-placement="'top'" :size="'large'" :show-label="true">
    <!-- BIN文件管理方式 -->
    <n-form-item :label="'BIN文件管理方式'" :show-label="true">
      <div class="mode-selection">
        <div
          class="mode-option"
          :class="{ active: binManageMode === 'stableId' }"
          @click="binManageMode = 'stableId'"
        >
          <div class="mode-title">StableId模式</div>
          <div class="mode-description">文件名相同则视为同一文件，适合BIN变动但文件名不变的情况。</div>
        </div>
        <div
          class="mode-option"
          :class="{ active: binManageMode === 'hash' }"
          @click="binManageMode = 'hash'"
        >
          <div class="mode-title">Hash值模式</div>
          <div class="mode-description">文件内容相同即Hash相同则视为同一文件。</div>
        </div>
      </div>
    </n-form-item>

    <div class="form-actions">
      <n-button type="primary" size="large" block :loading="isImporting" @click="handleImport">
        <template #icon>
          <n-icon>
            <CloudUpload />
          </n-icon>
        </template>
        添加Token
      </n-button>

      <n-button v-if="tokenStore.hasTokens" size="large" block @click="cancel">
        取消
      </n-button>
    </div>

    <n-form-item :label="'游戏角色名称'" :show-label="true">
      <n-input v-model:value="importForm.name" placeholder="例如：主号战士" clearable />
    </n-form-item>

    <n-form-item :label="'bin文件'" :show-label="true">
      <a-upload multiple accept="*.bin,*.dmp" @before-upload="uploadBin" draggable dropzone placeholder="粘贴Token字符串..."
        clearable>
        <!-- <div class="dropzone-content">
          请点击上传或将bind文件拖拽到此处
        </div> -->
      </a-upload>
    </n-form-item>
    <a-list>
      <a-list-item v-for="(role, index) in roleList" :key="index">
        <div>
          <strong>角色名称:</strong> {{ role.name || "未命名角色" }}<br />
          <strong>Token:</strong>
          <span style="word-break: break-all">{{ role.token }}</span><br />
          <strong>服务器:</strong> {{ role.server || "未指定" }}
        </div>
      </a-list-item>
    </a-list>

    <!-- 角色详情 -->
    <n-collapse>
      <n-collapse-item title="角色详情 (可选)" name="optional">
        <div class="optional-fields">
          <n-form-item label="服务器">
            <n-input v-model:value="importForm.server" placeholder="服务器名称" />
          </n-form-item>

          <n-form-item label="自定义连接地址">
            <n-input v-model:value="importForm.wsUrl" placeholder="留空使用默认连接" />
          </n-form-item>
        </div>
      </n-collapse-item>
    </n-collapse>


  </n-form>
</template>

<script lang="ts" setup>
import { ref, reactive } from "vue";
import { useTokenStore } from "@/stores/tokenStore";
import { CloudUpload } from "@vicons/ionicons5";

import {
  NForm,
  NFormItem,
  NInput,
  NButton,
  NIcon,
  NCollapse,
  NCollapseItem,
  useMessage,
} from "naive-ui";

import PQueue from "p-queue";
import useIndexedDB from "@/hooks/useIndexedDB";
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
const binManageMode = ref("stableId"); // BIN文件管理方式：stableId-文件名模式，hash-Hash值模式
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
  // 匹配格式：bin-{服务器}服-{角色索引}-{角色ID}-{角色名}.bin
  let binRes = fileName.match(/^bin-(.*?)服-([0-2])-([0-9]{6,12})-(.*)\.bin$/);

  if (binRes) {
    importForm.name = `${binRes[1]}_${binRes[2]}_${binRes[4]}`;
    return {
      server: binRes[1],
      roleIndex: binRes[2],
      roleId: binRes[3],
      roleName: binRes[4],
      // StableId 生成：bin-服务器-索引-角色ID
      stableId: `bin-${binRes[1]}-${binRes[2]}-${binRes[3]}`,
    };
  }
  return {
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
      const tokenId = binManageMode.value === 'stableId'
        ? roleMeta.stableId
        : getTokenId(userToken);
      const roleToken = await transformToken(userToken);
      const roleName = roleMeta.roleName || binFile.name.split(".")?.[0] || "";
      // 刷新indexDB数据库token数据
      const saved = await storeArrayBuffer(tokenId, userToken);
      if (!saved) {
        message.error("保存BIN数据到IndexedDB失败");
        return;
      }

      // 上传列表中发现已存在的重复名称，提示消息
      if (roleList.value.some((role) => role.id === tokenId)) {
        message.error("上传列表中已存在同名角色! ");
        return;
      }
      // 检查待上传的角色是否已在tokenStore中存在
      const existingToken = tokenStore.gameTokens.find(
        (t) => t.id === tokenId,
      );
      if (existingToken) {
        message.warning(`角色"${roleName}"已存在，将更新该角色的Token`);
      }
      message.success("Token读取成功，请检查角色名称等信息后提交");
      roleList.value.push({
        id: tokenId,
        token: roleToken,
        name: roleName,
        server: roleMeta.server + "" + roleMeta.roleIndex || "",
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
      const oldToken = tokenStore.gameTokens.find((t) => t.id === role.id);
      if (oldToken) {
        oldTokenGroups = tokenStore.getTokenGroups(role.id).map(g => g.id);
      }
    }

    // tokenStore.gameTokens中发现已存在的重复名称，则移出token后重新添加
    const gameToken = tokenStore.gameTokens.find((t) => t.id === role.id);
    if (gameToken) {
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
      oldTokenGroups.forEach(groupId => {
        tokenStore.addTokenToGroup(groupId, role.id);
      });
    }
  });
  console.log("当前Token列表:", tokenStore.gameTokens);
  message.success("Token添加成功");
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
  display: flex;
  flex-direction: column;
  gap: 12px;
  width: 80%;
}

.mode-option {
  display: flex;
  flex-direction: column;
  padding: 16px;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  background: #fff;

  &:hover {
    border-color: #667eea;
    box-shadow: 0 2px 8px rgba(102, 126, 234, 0.15);
  }

  &.active {
    border-color: #667eea;
    background: linear-gradient(135deg, rgba(102, 126, 234, 0.1) 0%, rgba(118, 75, 162, 0.1) 100%);
    box-shadow: 0 0 0 2px rgba(102, 126, 234, 0.2);

    .mode-description {
      color: #667eea;
    }
  }
}

.mode-title {
  font-size: 16px;
  font-weight: 600;
  color: #1d2129;
}

.mode-description {
  font-size: 13px;
  color: #86909c;
  line-height: 1.5;
  margin-top: 4px;
}

/* 深色主题适配 */
[data-theme="dark"] .mode-option {
  border-color: rgba(255, 255, 255, 0.2);
  background: rgba(255, 255, 255, 0.05);

  &:hover {
    border-color: #667eea;
  }

  &.active {
    background: linear-gradient(135deg, rgba(102, 126, 234, 0.2) 0%, rgba(118, 75, 162, 0.2) 100%);
  }
}

[data-theme="dark"] .mode-title {
  color: #ffffff;
}

[data-theme="dark"] .mode-description {
  color: rgba(255, 255, 255, 0.6);
}

[data-theme="dark"] .mode-option.active .mode-description {
  color: #a5b4fc;
}
</style>
