<script lang="ts">
import { defineComponent } from "vue";
import Analytics from "../utils/analytics";
import Header from "./global/Header.vue";
import Footer from "./global/Footer.vue";
import TextSource from "./cards/TextSource.vue";
import FileSource from "./cards/FileSource.vue";
import FukumojiSource from "./cards/FukumojiSource.vue";
import Target from "./cards/Target.vue";
import Result from "./cards/Result.vue";
import Tutorial from "./cards/Tutorial.vue";
import TabButton from "./inputs/TabButton.vue";
import TabGroup from "./inputs/TabGroup.vue";
import Space from "./global/Space.vue";
import Grid from "./global/Grid.vue";
import GridItem from "./global/GridItem.vue";
import Image from "./icons/Image.vue";
import Text from "./icons/Text.vue";
import Emoji from "./icons/Emoji.vue";
import "../css/destyle.css";
import Card from "./global/Card.vue";

export default defineComponent({
  components: {
    Card,
    TextSource,
    FileSource,
    FukumojiSource,
    Target,
    Result,
    Tutorial,
    TabButton,
    TabGroup,
    Header,
    Footer,
    Space,
    Grid,
    GridItem,
    Image,
    Text,
    Emoji,
  },
  data() {
    return {
      baseImage: null as (HTMLImageElement | HTMLCanvasElement | null),
      name: null as (string | null),
      emojiName: null as (string | null),
      resultImages: [[]] as Blob[][],
      previewMode: false,
      emojiSize: null as (number | null),
      /* Misskey連携用 */
      existingEmojis: [] as string[],
      registrationStatus: null as ('loading' | 'success' | 'error' | null),
      registrationMessage: '' as string,
      /* ui */
      ui: {
        mode: "text",
        showTargetPanel: true,
        showTargetDetails: false,
      },
    };
  },
  mounted() {
    Analytics.switchMode("text");
    // Misskeyからのメッセージを受信
    window.addEventListener('message', this.handleMisskeyMessage);
  },
  beforeUnmount() {
    window.removeEventListener('message', this.handleMisskeyMessage);
  },
  methods: {
    handleMisskeyMessage(event: MessageEvent): void {
      try {
        // データが文字列の場合はパース、オブジェクトの場合はそのまま使用
        let data = event.data;
        if (typeof data === 'string') {
          data = JSON.parse(data);
        }

        if (data && data.source === 'misskey-emoji-picker') {
          console.log('Received from Misskey:', data.type, data);

          if (data.type === 'emoji-list') {
            // 既存絵文字リストを受信
            this.existingEmojis = data.emojis || [];
            console.log('Emoji list updated:', this.existingEmojis.length, 'emojis');
          } else if (data.type === 'result') {
            // 登録結果を受信
            console.log('Registration result:', data);
            console.log('Before update - status:', this.registrationStatus, 'message:', this.registrationMessage);
            if (data.success) {
              this.registrationStatus = 'success';
              this.registrationMessage = `絵文字「${data.emojiName}」を登録しました！`;
              console.log('After update - status:', this.registrationStatus, 'message:', this.registrationMessage);
              // 3秒後にリセット
              setTimeout(() => {
                this.registrationStatus = null;
                this.registrationMessage = '';
              }, 3000);
            } else {
              this.registrationStatus = 'error';
              this.registrationMessage = data.error || '登録に失敗しました';
              console.log('After update (error) - status:', this.registrationStatus, 'message:', this.registrationMessage);
              // 5秒後にリセット
              setTimeout(() => {
                this.registrationStatus = null;
                this.registrationMessage = '';
              }, 5000);
            }
          }
        }
      } catch (e) {
        // 他のメッセージ（React DevToolsなど）は無視
      }
    },
    onToggleShowTarget(): void {
      this.ui.showTargetPanel = !this.ui.showTargetPanel;
      Analytics.switchMode(this.ui.showTargetPanel ? "target" : this.ui.mode, true);
    },
    onSelectMode(value: string): void {
      this.ui.mode = value;
      this.ui.showTargetPanel = false;
      Analytics.switchMode(value, true);
    },
    onRenderTarget(imgs: Blob[][]): void {
      this.resultImages = imgs;
      Analytics.render();
    },
    onRender(img: HTMLImageElement, name: string, emojiName: string): void {
      this.baseImage = img;
      this.name = name;
      this.emojiName = emojiName
    },
    setRegistrationLoading(): void {
      this.registrationStatus = 'loading';
      this.registrationMessage = '登録中...';
    },
  },
});
</script>

<template>
  <div class="app">
    <!-- 登録ステータス表示 -->
    <div v-if="registrationStatus" class="registration-status" :class="registrationStatus">
      {{ registrationMessage }}
    </div>

    <!-- プレビューエリア -->
    <div class="preview-area">
      <Result
          :images="resultImages"
          :name="name"
          :emojiName="emojiName"
          :existing-emojis="existingEmojis"
          :registration-status="registrationStatus"
          :show-target="ui.showTargetPanel"
          @toggle-show-target="onToggleShowTarget"
          @registration-start="setRegistrationLoading"
      />
    </div>

    <!-- 入力エリア -->
    <div class="input-area">
      <TextSource
          :show="ui.mode == 'text'"
          :emoji-size="emojiSize"
          :existing-emojis="existingEmojis"
          @render="onRender"
      />
      <Target
          v-model:emoji-size="emojiSize"
          :show="ui.showTargetPanel"
          :base-image="baseImage"
          @render="onRenderTarget" />
    </div>
  </div>
</template>

<style>
:root {
  /* colors - モダンなダークテーマベース */
  --fg:             #e8e8e8;
  --fg-muted:       #a0a0a0;
  --bg:             #1a1a1a;
  --bg-secondary:   #252525;
  --bg-tertiary:    #2d2d2d;
  --accentBg:       #ffffff08;
  --border:         #404040;
  --border-light:   #505050;

  /* アクセントカラー */
  --accent:         #86b300;
  --accentLighter:  #9ed119;
  --accentDarker:   #6d9200;
  --accentBg:       #86b30015;

  --primary:        #86b300;
  --primaryLighter: #9ed119;
  --primaryDarker:  #6d9200;
  --primaryBg:      #86b30020;
  --primaryShadow:  0 0 0 2px #86b30030;
  --primaryHover:   #9ed119;
  --primaryActive:  #6d9200;

  --danger:         #ff6b6b;
  --dangerLighter:  #ff8585;
  --dangerDarker:   #e55555;
  --dangerBg:       #ff6b6b20;
  --dangerShadow:   0 0 0 2px #ff6b6b30;
  --dangerHover:    #ff8585;
  --dangerActive:   #e55555;

  --success:        #51cf66;
  --successBg:      #51cf6620;

  /* text colors */
  --light:          #ffffff;
  --dark:           #1a1a1a;
  --distantFg:      #ffffff;

  --elevatedBg:      #2d2d2d;
  --elevationShadow: 0 4px 12px rgba(0, 0, 0, 0.4);

  /* typography */
  --fontSizeSmall: 12px;
  --fontSizeMedium: 13px;
  --fontSizeLarge: 14px;
  --fontSizeXLarge: 16px;
  --fontSizeSmallIcon: 16px;
  --fontSizeIcon: 24px;
  --fontSizePart: 32px;

  /* layout */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 12px;
  --spacing-lg: 16px;
  --spacing-xl: 24px;
  --spacingInlineSmall: 4px;
  --spacingMedium: 8px;
  --spacingLarge: 16px;
  --borderRadius: 8px;
  --borderRadiusSmall: 6px;
  --borderRadiusMicro: 3px;
  --paddingMinimal: 4px;
  --padding: 8px;

  /* slider */
  --sliderKnobSize: 16px;
  --sliderRailHeight: 6px;
  --sliderMarkHeight: 8px;
  --sliderValueWidth: 40px;
  --sliderValueMargin: 4px;

  /* shadows */
  --pressableShadowDefault: 0 1px 2px rgba(0, 0, 0, 0.2);
  --pressableShadowPrimary: 0 1px 2px rgba(134, 179, 0, 0.3);
  --pressableShadowPrimaryHover: 0 2px 4px rgba(134, 179, 0, 0.4);
  --pressableShadowDanger: 0 1px 2px rgba(255, 107, 107, 0.3);
  --pressableShadowDangerHover: 0 2px 4px rgba(255, 107, 107, 0.4);
}

* {
  box-sizing: border-box;
}

html, body {
  margin: 0;
  padding: 0;
  height: 100%;
  overflow-x: hidden;
  scrollbar-width: thin;
  scrollbar-color: var(--border) var(--bg-secondary);
}

html {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  font-size: var(--fontSizeMedium);
  line-height: 1.4;
  color: var(--fg);
  background: var(--bg);
}

.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  padding: var(--spacing-sm);
  padding-top: calc(80px + var(--spacing-sm) * 2);
  gap: var(--spacing-sm);
}

/* プレビューエリア */
.preview-area {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 100;
  background: var(--bg-secondary);
  border-radius: 0 0 var(--borderRadius) var(--borderRadius);
  padding: 0.16rem;
  border-bottom: 1px solid var(--border);
}

/* 入力エリア */
.input-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: var(--spacing-sm);
  overflow-y: auto;
}

/* 共通入力スタイル */
input, textarea, select {
  background: var(--bg-tertiary);
  border: 1px solid var(--border);
  border-radius: var(--borderRadiusSmall);
  color: var(--fg);
  font-size: var(--fontSizeMedium);
  padding: 10px 12px;
  width: 100%;
  transition: border-color 0.2s, box-shadow 0.2s;
}

input:focus, textarea:focus, select:focus {
  outline: none;
  border-color: var(--accent);
  box-shadow: 0 0 0 2px var(--accentBg);
}

input::placeholder, textarea::placeholder {
  color: var(--fg-muted);
}

/* 登録ステータス表示 */
.registration-status {
  position: fixed;
  top: var(--spacing-sm);
  left: var(--spacing-sm);
  right: var(--spacing-sm);
  padding: 12px 16px;
  text-align: center;
  font-weight: 600;
  font-size: var(--fontSizeMedium);
  z-index: 1000;
  border-radius: var(--borderRadius);
  animation: slideDown 0.3s ease-out;
}

.registration-status.loading {
  background: var(--primaryBg);
  color: var(--primary);
  border: 1px solid var(--primary);
}

.registration-status.success {
  background: var(--successBg);
  color: var(--success);
  border: 1px solid var(--success);
}

.registration-status.error {
  background: var(--dangerBg);
  color: var(--danger);
  border: 1px solid var(--danger);
}

@keyframes slideDown {
  from {
    transform: translateY(-20px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

/* ラベルスタイル */
label, .label {
  display: block;
  font-size: var(--fontSizeSmall);
  color: var(--fg-muted);
  margin-bottom: var(--spacing-xs);
  font-weight: 500;
}
</style>
