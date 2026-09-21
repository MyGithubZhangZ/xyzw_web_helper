<template>
  <n-modal
    v-model:show="visible"
    :mask-closable="false"
    preset="card"
    :closable="false"
    class="about-modal"
    style="width: 480px; max-width: calc(100vw - 32px)"
    :bordered="false"
    :segmented="{ content: true }"
    display-directive="show"
    transform-origin="center"
  >
    <template #header>
      <div class="about-modal-header">
        <span class="about-title">关于旋律</span>
        <n-button quaternary circle size="small" @click="handleClose">
          <template #icon>
            <n-icon size="20">
              <Close />
            </n-icon>
          </template>
        </n-button>
      </div>
    </template>

    <div class="about-content">
      <section class="about-section">
        <div class="section-title">
          <n-icon size="18" class="section-icon">
            <InformationCircleOutline />
          </n-icon>
          <span>作者声明</span>
        </div>
        <ul class="section-list">
          <li>
            本软件基于Github开发者w1249178256发布的开源项目二次开发打包。
          </li>
          <li>
            免费分享，定期更新新功能。未在任何平台发售，请勿付费购买。
          </li>
          <li>
            本软件仅供个人学习交流使用，请勿用于商业用途。
          </li>
          <li>
            任何人不得将本软件用于商业用途，否则后果自负。
          </li>
        </ul>
      </section>

      <section class="about-section">
        <div class="section-title">
          <n-icon size="18" class="section-icon">
            <HelpCircleOutline />
          </n-icon>
          <span>使用说明</span>
        </div>
        <ul class="section-list">
          <li>无相关功能说明,自行研究，或者联系旋律</li>
        </ul>
      </section>

      <section class="about-section">
        <div class="section-title">
          <n-icon size="18" class="section-icon">
            <GiftOutline />
          </n-icon>
          <span>赞赏支持</span>
        </div>
        <ul class="section-list">
          <li>您的支持是我持续更新的动力！</li>
          <li>感谢您的支持！未成年人请勿打赏！</li>
        </ul>
      </section>

      <div class="qrcode-area">
        <n-image
          class="qrcode-image"
          :src="qrcodeUrl"
          :preview-src="previewQrcodeUrl"
          object-fit="contain"
          alt="赞赏二维码"
          img-class="qrcode-img"
          :fallback-src="qrcodeUrl"
        >
          <template #error>
            <div class="qrcode-error">
              <n-icon size="24"><ImageOutline /></n-icon>
              <span>图片加载失败</span>
            </div>
          </template>
        </n-image>
        <p class="qrcode-tip">点击二维码可查看大图</p>
      </div>
    </div>
  </n-modal>
</template>

<script setup>
import { computed, watch } from "vue";
import { NModal, NButton, NIcon, NImage } from "naive-ui";
import {
  Close,
  ImageOutline,
  InformationCircleOutline,
  HelpCircleOutline,
  GiftOutline,
} from "@vicons/ionicons5";

const props = defineProps({
  show: {
    type: Boolean,
    default: false,
  },
  // 弹窗内显示的二维码图片（小图）
  qrcodeUrl: {
    type: String,
    default: "/qrcode/small-qrcode.png",
  },
  // 点击放大预览的二维码图片（大图）
  previewQrcodeUrl: {
    type: String,
    default: "/qrcode/about-qrcode.jpg",
  },
});

const emit = defineEmits(["update:show", "close"]);

const visible = computed({
  get: () => props.show,
  set: (val) => emit("update:show", val),
});

watch(
  () => props.show,
  (val) => {
    visible.value = val;
  }
);

const handleClose = () => {
  visible.value = false;
  emit("close");
};

</script>

<style scoped>
.about-modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
}

.about-title {
  font-size: 18px;
  font-weight: 600;
  color: #333;
}

.about-content {
  padding: 8px 4px;
}

.about-section {
  margin-bottom: 16px;
}

.section-title {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  background: linear-gradient(135deg, #6b7cff 0%, #8b5cf6 100%);
  border-radius: 8px;
  color: #fff;
  font-size: 15px;
  font-weight: 500;
  margin-bottom: 12px;
}

.section-icon {
  display: flex;
  align-items: center;
  justify-content: center;
}

.section-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.section-list li {
  position: relative;
  padding-left: 18px;
  margin-bottom: 10px;
  font-size: 14px;
  line-height: 1.6;
  color: #555;
}

.section-list li::before {
  content: "";
  position: absolute;
  left: 0;
  top: 7px;
  width: 8px;
  height: 8px;
  background: linear-gradient(135deg, #6b7cff 0%, #8b5cf6 100%);
  border-radius: 50%;
}

.qrcode-area {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 8px;
}

.qrcode-image {
  width: 240px;
  height: 240px;
  border-radius: 8px;
  border: 1px solid #eee;
  background: #fafafa;
  cursor: zoom-in;
}

.qrcode-image :deep(img) {
  image-rendering: -webkit-optimize-contrast;
  image-rendering: crisp-edges;
  image-rendering: pixelated;
}

.qrcode-error {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 8px;
  width: 100%;
  height: 100%;
  color: #999;
  font-size: 13px;
}

.qrcode-tip {
  margin-top: 10px;
  font-size: 13px;
  color: #999;
  text-align: center;
}
</style>
