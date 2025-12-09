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
  <Space vertical large>
    <Card class="result">
      <RawResult
          :images="resultImageUrls"
          :rounded="rounded"/>
      <!--        <span v-if="isDev && images[0].length === 1">-->
      <!--          ファイルサイズ: {{ totalSize / 1024 }} KiB-->
      <!--        </span>-->
      <!-- <Checkbox v-model="previewMode" name="サンプル表示">
        {{ "サンプル表示" }}
      </Checkbox>
      <Checkbox v-model="rounded" name="角丸">
        {{ "角丸プレビュー" }}
      </Checkbox> -->
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

    </Card>
    <Space class="buttons">
      <!-- <Button
          v-if="showTarget"
          name="効果をつける(戻る)"
          @click="$emit('toggleShowTarget', $event)">
        <template #icon>
          <Back />
        </template>
        もどる
      </Button>
      <Button
          v-else
          name="効果をつける"
          @click="$emit('toggleShowTarget', $event)">
        <template #icon>
          <Effect />
        </template>
        効果をつける
      </Button> -->
      <!--      <Button type="primary" name="保存" @click="onDownload">-->
      <!--        <template #icon>-->
      <!--          <Save />-->
      <!--        </template>-->
      <!--        絵文字を保存-->
      <!--      </Button>-->
    </Space>
  </Space>
</template>

<style scoped>
.result {
  background-image: linear-gradient(
      45deg,
      var(--bg) 25%,
      transparent 25%,
      transparent 75%,
      var(--bg) 75%,
      var(--bg)
  ),
  linear-gradient(
      45deg,
      var(--bg) 25%,
      transparent 25%,
      transparent 75%,
      var(--bg) 75%,
      var(--bg)
  );
  background-position: 0 0, 10px 10px;
  background-size: 20px 20px;
  position: relative;
}

.reaction-button {
  position: absolute;
  right: 8px;
  bottom: 8px;
  padding: 8px 16px;
  font-size: 14px;
  font-weight: bold;
  color: white;
  background: linear-gradient(135deg, #28a745 0%, #20863b 100%);
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  display: flex;
  align-items: center;
  gap: 6px;
}

.reaction-button:hover:not(:disabled) {
  background: linear-gradient(135deg, #34ce57 0%, #28a745 100%);
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.25);
}

.reaction-button:active:not(:disabled) {
  transform: translateY(0);
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

.reaction-button.is-disabled {
  background: #666;
  cursor: not-allowed;
  opacity: 0.7;
}

.reaction-button.is-error {
  background: linear-gradient(135deg, var(--danger) 0%, var(--dangerDarker) 100%);
}

.reaction-button.is-loading {
  background: linear-gradient(135deg, var(--primary) 0%, var(--primaryDarker) 100%);
  cursor: wait;
}

.spinner {
  width: 14px;
  height: 14px;
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
