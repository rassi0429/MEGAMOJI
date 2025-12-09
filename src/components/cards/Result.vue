<script lang="ts">
import {defineComponent, PropType} from "vue";
import {saveAs} from "file-saver";
import filenamify from "filenamify/browser";
import {extension, prepareDownloadFile} from "../../utils/file";
import Analytics from "../../utils/analytics";
import RawResult from "../emoji/RawResult.vue";
import Preview from "../emoji/Preview.vue";
import Button from "../inputs/Button.vue";
import Checkbox from "../inputs/Checkbox.vue";
import Space from "../global/Space.vue";
import Card from "../global/Card.vue";
import Effect from "../icons/Effect.vue";
import Back from "../icons/Back.vue";
import Save from "../icons/Save.vue";
import {NODE_ENV} from "../../utils/env";

export default defineComponent({
  components: {
    RawResult, Preview, Checkbox, Card, Space, Button, Effect, Back, Save,
  },
  props: {
    images: {type: Array as PropType<Blob[][]>, required: true},
    name: {type: String, default: null},
    showTarget: {type: Boolean, required: false},
    emojiName: {type: String, default: null},
    existingEmojis: {type: Array as PropType<string[]>, default: () => []},
    registrationStatus: {type: String as PropType<'loading' | 'success' | 'error' | null>, default: null},
  },
  emits: [
    "toggleShowTarget",
    "registrationStart",
  ],
  data() {
    return {
      previewMode: false,
      rounded: false,
      isDev: NODE_ENV === "development",
    };
  },
  computed: {
    resultImageUrls(): string[][] {
      return this.images.map((row) => row.map((cell) => URL.createObjectURL(cell)));
    },
    totalSize(): number {
      return this.images.reduce((l, r) => (
          l + r.reduce((ll, rr) => ll + rr.size, 0)
      ), 0);
    },
    isEmojiNameDuplicate(): boolean {
      if (!this.emojiName) return false;
      return this.existingEmojis.includes(this.emojiName);
    },
    canReact(): boolean {
      return !!(
        this.name &&
        this.emojiName &&
        !this.isEmojiNameDuplicate &&
        this.registrationStatus !== 'loading'
      );
    },
    buttonText(): string {
      if (this.registrationStatus === 'loading') return '登録中...';
      if (this.isEmojiNameDuplicate) return '名前が重複';
      if (!this.emojiName) return '名前を入力';
      return 'リアクション';
    },
  },
  methods: {
    onDownload(): void {
      const download = prepareDownloadFile(this.images);
      const filename = filenamify(this.name ?? "", {replacement: ""}).normalize() || "megamoji";
      download.then((res) => saveAs(res, `${filename}.${extension(res)}`));
      Analytics.download();
    },
    reaction(): void {
      if (!this.canReact) return;

      console.log("reactionClicked");
      this.$emit('registrationStart');

      if (this.images.length > 0 && this.images[0].length > 0) {
        const blob = this.images[0][0];

        const reader = new FileReader();
        reader.onloadend = () => {
          const base64data = reader.result;
          window.parent.postMessage(JSON.stringify({
            "source": "emoji-gen",
            "emojiName": this.emojiName,
            "binaryData": base64data
          }), "*");
        };
        reader.readAsDataURL(blob);
      }
    },
  },
});
</script>

<template>
  <div class="result-container">
    <!-- プレビュー画像 -->
    <div class="preview-wrapper">
      <div class="preview-image" :class="{ 'has-image': resultImageUrls[0]?.length > 0 }">
        <RawResult
            v-if="resultImageUrls[0]?.length > 0"
            :images="resultImageUrls"
            :rounded="rounded"/>
        <div v-else class="placeholder">
          <span class="placeholder-icon">🎨</span>
          <span class="placeholder-text">テキストを入力</span>
        </div>
      </div>
    </div>

    <!-- リアクションボタン -->
    <button
        v-show="name"
        class="reaction-button"
        :class="{
          'is-loading': registrationStatus === 'loading',
          'is-disabled': !canReact,
          'is-error': isEmojiNameDuplicate
        }"
        :disabled="!canReact"
        @click="reaction">
      <span v-if="registrationStatus === 'loading'" class="spinner"></span>
      {{ buttonText }}
    </button>
  </div>
</template>

<style scoped>
.result-container {
  display: flex;
  align-items: center;
  gap: var(--spacing-md, 12px);
}

.preview-wrapper {
  flex-shrink: 0;
}

.preview-image {
  width: 64px;
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: center;
  background:
    linear-gradient(45deg, #333 25%, transparent 25%),
    linear-gradient(-45deg, #333 25%, transparent 25%),
    linear-gradient(45deg, transparent 75%, #333 75%),
    linear-gradient(-45deg, transparent 75%, #333 75%);
  background-size: 8px 8px;
  background-position: 0 0, 0 4px, 4px -4px, -4px 0px;
  background-color: #2a2a2a;
  border-radius: var(--borderRadius, 8px);
  overflow: hidden;
  border: 1px solid var(--border, #404040);
}

.preview-image.has-image {
  background: none;
  background-color: transparent;
  border: none;
}

.preview-image :deep(img) {
  max-width: 64px;
  max-height: 64px;
  object-fit: contain;
}

.placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
  color: var(--fg-muted, #666);
}

.placeholder-icon {
  font-size: 24px;
  opacity: 0.5;
}

.placeholder-text {
  font-size: 10px;
  opacity: 0.6;
}

.reaction-button {
  flex: 1;
  padding: 12px 20px;
  font-size: 14px;
  font-weight: 600;
  color: white;
  background: linear-gradient(135deg, var(--accent, #86b300) 0%, var(--accentDarker, #6d9200) 100%);
  border: none;
  border-radius: var(--borderRadius, 8px);
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 2px 8px rgba(134, 179, 0, 0.3);
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  min-height: 44px;
}

.reaction-button:hover:not(:disabled) {
  background: linear-gradient(135deg, var(--accentLighter, #9ed119) 0%, var(--accent, #86b300) 100%);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(134, 179, 0, 0.4);
}

.reaction-button:active:not(:disabled) {
  transform: translateY(0);
  box-shadow: 0 1px 4px rgba(134, 179, 0, 0.3);
}

.reaction-button.is-disabled {
  background: var(--bg-tertiary, #2d2d2d);
  color: var(--fg-muted, #666);
  cursor: not-allowed;
  box-shadow: none;
  border: 1px solid var(--border, #404040);
}

.reaction-button.is-error {
  background: linear-gradient(135deg, var(--danger, #ff6b6b) 0%, var(--dangerDarker, #e55555) 100%);
  box-shadow: 0 2px 8px rgba(255, 107, 107, 0.3);
}

.reaction-button.is-loading {
  background: linear-gradient(135deg, var(--primary, #86b300) 0%, var(--primaryDarker, #6d9200) 100%);
  cursor: wait;
  opacity: 0.8;
}

.spinner {
  width: 16px;
  height: 16px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
</style>
