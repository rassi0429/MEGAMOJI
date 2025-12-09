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
        const data = JSON.parse(event.data);
        if (data.source === 'misskey-emoji-picker') {
          if (data.type === 'emoji-list') {
            // 既存絵文字リストを受信
            this.existingEmojis = data.emojis || [];
            console.log('Received emoji list from Misskey:', this.existingEmojis.length, 'emojis');
          } else if (data.type === 'result') {
            // 登録結果を受信
            if (data.success) {
              this.registrationStatus = 'success';
              this.registrationMessage = `絵文字「${data.emojiName}」を登録しました！`;
              // 3秒後にリセット
              setTimeout(() => {
                this.registrationStatus = null;
                this.registrationMessage = '';
              }, 3000);
            } else {
              this.registrationStatus = 'error';
              this.registrationMessage = data.error || '登録に失敗しました';
              // 5秒後にリセット
              setTimeout(() => {
                this.registrationStatus = null;
                this.registrationMessage = '';
              }, 5000);
            }
          }
        }
      } catch (e) {
        // JSON parse error - ignore non-JSON messages
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
    <Space vertical large full>

      <!-- 登録ステータス表示 -->
      <div v-if="registrationStatus" class="registration-status" :class="registrationStatus">
        {{ registrationMessage }}
      </div>

      <Grid :columns="[[760, 1], [Infinity, 3]]" spaced>
        <GridItem :span="3">
          <Result
              :images="resultImages"
              :name="name"
              :emojiName="emojiName"
              :existing-emojis="existingEmojis"
              :registration-status="registrationStatus"
              :show-target="ui.showTargetPanel"
              @toggle-show-target="onToggleShowTarget"
              @registration-start="setRegistrationLoading"
              style="position: fixed; height: 80px; width: 100%"
          />
          <TextSource
              :show="ui.mode == 'text'"
              :emoji-size="emojiSize"
              :existing-emojis="existingEmojis"
              @render="onRender"
              style="margin-top: 80px"
          />
          <Target
              v-model:emoji-size="emojiSize"
              :show="ui.showTargetPanel"
              :base-image="baseImage"
              @render="onRenderTarget" />
        </GridItem>
      </Grid>

      <Footer />
    </Space>
  </div>
</template>

<style>
:root {
  /* colors */
  --fg:             #000000d0;
  --bg:             #ffffffff;
  --accentBg:       #00000004;
  --border:         #00000040;
  --primaryLighter: #ffb81c; /* l = 80 */
  --primary:        #eeaa00; /* okhsl(80, 100, 75) */
  --primaryDarker:  #cd9200; /* l = 65 */
  --dangerLighter:  #fa837e; /* l = 70 */
  --danger:         #f76b68; /* okhsl(24, 90, 65) */
  --dangerDarker:   #e53c42; /* l = 55 */
  --primaryBg:      #eeaa0020;
  --dangerBg:       #f76b6820;
  --primaryShadow:  0 0 0 2px #eeaa0030;
  --dangerShadow:   0 0 0 2px #f76b6830;

  --elevatedBg:      #ffffffff;
  --elevationShadow: rgb(0 0 0 / 0.19) 0 10px 20px, rgb(0 0 0 / 0.23) 0 6px 6px;

  --pressableShadowDefault:      0 1px #00000040;
  --pressableShadowPrimary:      0 1px #00000020, 0 1px var(--primary);
  --pressableShadowPrimaryHover: 0 1px #00000020, 0 1px var(--primaryLighter);
  --pressableShadowDanger:       0 1px #00000020, 0 1px var(--danger);
  --pressableShadowDangerHover:  0 1px #00000020, 0 1px var(--dangerLighter);

  --distantFg:     var(--bg);
  --light:         var(--bg);
  --dark:          var(--fg);
  --primaryHover:  var(--primaryLighter);
  --primaryActive: var(--primaryDarker);
  --dangerHover:   var(--dangerLighter);
  --dangerActive:  var(--dangerDarker);
}

@media (prefers-color-scheme: dark) {
  :root {
    --fg:             #ffffffd0;
    --bg:             #222222ff;
    --accentBg:       #ffffff10;
    --border:         #ffffff40;
    --primaryLighter: #e6af47; /* l = 75 */
    --primary:        #d7a139; /* okhsl(80, 80, 70) */
    --primaryDarker:  #c79431; /* l = 65 */
    --dangerLighter:  #f38882; /* l = 70 */
    --danger:         #ee736e; /* okhsl(24, 80, 65) */
    --dangerDarker:   #e65f5c; /* k = 60 */
    --primaryBg:      #d7a13920;
    --dangerBg:       #ee736e20;
    --primaryShadow:  0 0 0 2px #e6af4730;
    --dangerShadow:   0 0 0 2px #ee736e30;

    --elevatedBg:      #555555ff;
    --elevationShadow: none;

    --pressableShadowDefault:      0 1px #00000080;
    --pressableShadowPrimary:      0 1px #00000040, 0 1px var(--primary);
    --pressableShadowPrimaryHover: 0 1px #00000040, 0 1px var(--primaryDarker);
    --pressableShadowDanger:       0 1px #00000040, 0 1px var(--danger);
    --pressableShadowDangerHover:  0 1px #00000040, 0 1px var(--dangerDarker);

    --distantFg:     var(--bg);
    --light:         var(--fg);
    --dark:          var(--bg);
    --primaryHover:  var(--primaryDarker);
    --primaryActive: var(--primaryLighter);
    --dangerHover:   var(--dangerDarker);
    --dangerActive:  var(--dangerLighter);
  }
}

:root {
  /* typography */
  --fontSizeTitle: 28px;
  --fontSizeXLarge: 18px;
  --fontSizeLarge: 16px;
  --fontSizeMedium: 14px;
  --fontSizeSmallIcon: 16px;
  --fontSizeIcon: 26px;
  --fontSizePart: 48px;
  --multilineTextLineHeight: 1.5;

  /* layout */
  --spacingSmall: 0.25rem;
  --spacingInlineSmall: 0.375rem;
  --spacingMedium: 0.5rem;
  --spacingLarge: 0.2rem;
  --spacingXLarge: 0.2rem;
  --paddingV: 0.75rem;
  --paddingH: 1rem;
  --padding: var(--paddingV) var(--paddingH);
  --paddingMinimal: 4px;
  --borderRadiusMicro: 4px; /* elements that should not be "too rounded" (eg. checkboxes) */
  --borderRadiusSmall: 4px;
  --borderRadius: 8px;

  /* other */
  --textareaLineHeight: 1.4;
  --selectPadding: var(--paddingV) calc(var(--paddingH) + 1rem) var(--paddingV) var(--paddingH);
  --numberPadding: var(--paddingV) calc(var(--paddingH) + 2rem) var(--paddingV) var(--paddingH);
  --sliderRailHeight: 0.375em;
  --sliderMarkHeight: 0.75em;
  --sliderKnobSize: 1.25em;
  --sliderValueMargin: 0.125em;
  --sliderValueWidth: 2.5em;
  --colorSliderRailHeight: 1.125em;
  --mediaIconSize: 34px;
  --tabButtonPadding: 0 calc(var(--paddingH) * 0.75) calc(var(--paddingV) - 3px);
}

/* stylelint-disable-next-line selector-max-type */
html {
  padding: var(--spacingLarge);
  font-family:
    "Helvetica Neue",
    Arial,
    "Hiragino Kaku Gothic ProN",
    "Hiragino Sans",
    Meiryo,
    sans-serif;
  font-size: var(--fontSizeMedium);
  line-height: 1;
  color: var(--fg);
  background-color: var(--bg);
}

/* 登録ステータス表示 */
.registration-status {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  padding: 12px 16px;
  text-align: center;
  font-weight: bold;
  z-index: 1000;
  animation: slideDown 0.3s ease-out;
}

.registration-status.loading {
  background: var(--primaryBg);
  color: var(--primary);
  border-bottom: 2px solid var(--primary);
}

.registration-status.success {
  background: #d4edda;
  color: #155724;
  border-bottom: 2px solid #28a745;
}

.registration-status.error {
  background: var(--dangerBg);
  color: var(--danger);
  border-bottom: 2px solid var(--danger);
}

@keyframes slideDown {
  from {
    transform: translateY(-100%);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}
</style>
