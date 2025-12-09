<script lang="ts">
import {defineComponent} from "vue";
import Analytics from "../../utils/analytics";
import FontSelectBlock from "../formblocks/FontSelectBlock.vue";
import FontColorSelectBlock from "../formblocks/FontColorSelectBlock.vue";
import OutlineBlock from "../formblocks/OutlineBlock.vue";
import Checkbox from "../inputs/Checkbox.vue";
import Textarea from "../inputs/Textarea.vue";
import ToggleButton from "../inputs/ToggleButton.vue";
import Fieldset from "../inputs/Fieldset.vue";
import Slider from "../inputs/Slider.vue";
import Select from "../inputs/Select.vue";
import Space from "../global/Space.vue";
import Card from "../global/Card.vue";
import Grid from "../global/Grid.vue";
import GridItem from "../global/GridItem.vue";
import AlignJustify from "../icons/AlignJustify.vue";
import AlignCenter from "../icons/AlignCenter.vue";
import AlignLeft from "../icons/AlignLeft.vue";
import AlignRight from "../icons/AlignRight.vue";

import {ColorStop} from "../../types";
import {absColor} from "../../utils/color";
import {makeTextImage} from "../../utils/textimage";
import {EMOJI_SIZE} from "../../constants/emoji";
import fonts from "../../constants/fonts";
import Input from "../inputs/Input.vue";

type PaddingOption = { label: string, value: number };

const PADDING_OPTIONS = [
  {label: "極小 (推奨)", value: 0.02},
  {label: "大きめ (角丸対応)", value: 0.1},
  {label: "なし", value: 0},
];

export default defineComponent({
  components: {
    Input,
    FontSelectBlock,
    FontColorSelectBlock,
    OutlineBlock,
    Fieldset,
    Slider,
    Select,
    Checkbox,
    Textarea,
    Grid,
    GridItem,
    Space,
    Card,
    ToggleButton,
    AlignJustify,
    AlignCenter,
    AlignLeft,
    AlignRight,
  },
  props: {
    show: {type: Boolean, required: true},
    emojiSize: {type: Number, default: null},
    existingEmojis: {type: Array as () => string[], default: () => []},
  },
  emits: [
    "render",
  ],
  data() {
    return {
      PADDING_OPTIONS,
      conf: {
        /* basic */
        content: "",
        emojiName: "",
        emojiNameValid: false,
        align: "stretch",
        color: "#ffda00",
        gradient: [] as ColorStop[],
        outlines: [] as string[],
        outlineThickness: 8,
        outlineX: 0,
        outlineY: 0,
        padding: PADDING_OPTIONS[0],
        font: fonts[0].fonts[0].value,
        /* advanced */
        lineSpacing: 0.05,
        paddingValue: 0.02,
      },
      showDetails: false,
      /* internals */
      running: false,
      dirty: false,
    };
  },
  computed: {
    absoluteOutlines(): string[] {
      return this.conf.outlines.map((outline) => absColor(outline, this.conf.color));
    },
    absoluteGradient(): { color: string, pos: number }[] {
      return this.conf.gradient.map((cs) => ({
        color: absColor(cs.color, this.conf.color),
        pos: cs.pos,
      }));
    },
    isEmojiNameDuplicate(): boolean {
      if (!this.conf.emojiName) return false;
      return this.existingEmojis.includes(this.conf.emojiName);
    },
  },
  watch: {
    conf: {
      handler(): void {
        Analytics.changeFont(this.conf.font);
        this.render(true);
      },
      deep: true,
    },
    emojiName: {
      handler(): void {
        this.render(true);
      },
    },
    emojiSize: {
      handler(): void {
        this.render(true);
      },
    },
  },
  mounted() {
    Analytics.changeFont(this.conf.font);
  },
  methods: {
    validateEmojiName(value: string): void {
      // Filter out non-alphanumeric characters
      const validatedName = value.replace(/[^a-zA-Z0-9]/g, '');

      // Update the emojiName with the validated value
      this.conf.emojiName = validatedName;

      // Set emojiNameValid based on whether the name is at least 1 character long
      this.conf.emojiNameValid = validatedName.length >= 1;
    },
    preventNonAlphanumeric(event: KeyboardEvent): void {
      // Allow only alphanumeric keys, plus control keys like backspace, delete, arrows, etc.
      const isAlphanumeric = /^[a-zA-Z0-9]$/.test(event.key);
      const isControlKey = event.key.length > 1; // Control keys like Backspace, Delete, arrows, etc.

      if (!isAlphanumeric && !isControlKey) {
        event.preventDefault();
      }
    },
    handleCompositionEnd(event: CompositionEvent): void {
      // This event fires when IME composition is completed
      // We need to validate the input immediately after IME confirmation
      const input = event.target as HTMLInputElement;
      const value = input.value;

      // Filter out non-alphanumeric characters
      const validatedName = value.replace(/[^a-zA-Z0-9]/g, '');

      // Update the input value directly
      input.value = validatedName;

      // Also update our model
      this.conf.emojiName = validatedName;
      this.conf.emojiNameValid = validatedName.length >= 1;

      // Prevent the default behavior to ensure our filtered value is used
      event.preventDefault();
    },
    render(dirty?: boolean): void {
      if (dirty) {
        this.dirty = true;
      }
      if (!this.dirty || this.running) {
        return;
      }
      this.running = true;
      this.dirty = false;
      if (this.conf.content) {
        const canvas = makeTextImage(
            this.conf.content,
            this.conf.color,
            this.conf.font,
            this.emojiSize || EMOJI_SIZE,
            this.conf.align,
            Number(this.conf.lineSpacing),
            this.absoluteOutlines,
            this.conf.outlineThickness,
            this.conf.outlineX,
            this.conf.outlineY,
            this.absoluteGradient,
            Number(this.conf.paddingValue),
        );
        const name = this.conf.content.replace(/\n/g, "");
        const emojiName = this.conf.emojiName.replace(/\n/g, "");
        this.$emit("render", canvas, name, emojiName);
      }
      window.setTimeout(() => {
        this.running = false;
        this.render();
      }, 50);
    },
    selectPadding(padding: PaddingOption): void {
      this.conf.paddingValue = padding.value;
    },
  },
});
</script>

<template>
  <div v-if="show" class="text-source">
    <!-- 絵文字名 -->
    <div class="form-group">
      <label class="form-label">絵文字名</label>
      <input
          type="text"
          :value="conf.emojiName"
          class="form-input"
          :class="{ 'input-error': isEmojiNameDuplicate }"
          placeholder="英数字のみ"
          @input="validateEmojiName($event.target.value)"
          @keydown="preventNonAlphanumeric"
          @compositionend="handleCompositionEnd"/>
      <div v-if="isEmojiNameDuplicate" class="duplicate-warning">
        この絵文字名は既に使用されています
      </div>
    </div>

    <!-- テキスト入力 -->
    <div class="form-group">
      <label class="form-label">テキスト</label>
      <textarea
          v-model="conf.content"
          class="form-textarea"
          placeholder="絵文字にしたい文字を入力"
          rows="2"/>
    </div>

    <!-- オプション行 -->
    <div class="options-row">
      <!-- フォント選択 -->
      <div class="option-item">
        <label class="form-label">フォント</label>
        <FontSelectBlock
            v-model="conf.font"
            :show-details="false"
            compact/>
      </div>

      <!-- 揃え -->
      <div class="option-item">
        <label class="form-label">揃え</label>
        <select v-model="conf.align" class="form-select">
          <option value="stretch">両端</option>
          <option value="center">中央</option>
          <option value="left">左</option>
          <option value="right">右</option>
        </select>
      </div>

      <!-- 色 -->
      <div class="option-item color-option">
        <label class="form-label">色</label>
        <FontColorSelectBlock
            v-model="conf.color"
            v-model:gradient="conf.gradient"
            :show-details="false"
            compact/>
      </div>
    </div>

    <!-- 縁取り（シンプル化） -->
    <div class="form-group">
      <OutlineBlock
          v-model="conf.outlines"
          v-model:thickness="conf.outlineThickness"
          v-model:posX="conf.outlineX"
          v-model:posY="conf.outlineY"
          :base-color="conf.color"
          :show-details="false"
          compact/>
    </div>
  </div>
</template>

<style scoped>
.text-source {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-md, 12px);
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: var(--spacing-xs, 4px);
}

.form-label {
  font-size: var(--fontSizeSmall, 12px);
  color: var(--fg-muted, #a0a0a0);
  font-weight: 500;
}

.form-input,
.form-textarea,
.form-select {
  background: var(--bg-tertiary, #2d2d2d);
  border: 1px solid var(--border, #404040);
  border-radius: var(--borderRadiusSmall, 6px);
  color: var(--fg, #e8e8e8);
  font-size: var(--fontSizeMedium, 13px);
  padding: 10px 12px;
  width: 100%;
  transition: border-color 0.2s, box-shadow 0.2s;
}

.form-input:focus,
.form-textarea:focus,
.form-select:focus {
  outline: none;
  border-color: var(--accent, #86b300);
  box-shadow: 0 0 0 2px var(--accentBg, rgba(134, 179, 0, 0.15));
}

.form-input::placeholder,
.form-textarea::placeholder {
  color: var(--fg-muted, #666);
}

.form-textarea {
  resize: vertical;
  min-height: 60px;
  line-height: 1.4;
}

.form-select {
  appearance: none;
  cursor: pointer;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%23888' stroke-width='2'%3E%3Cpolyline points='6 9 12 15 18 9'%3E%3C/polyline%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 10px center;
  background-size: 14px;
  padding-right: 32px;
}

.options-row {
  display: flex;
  gap: var(--spacing-sm, 8px);
  flex-wrap: wrap;
}

.option-item {
  flex: 1;
  min-width: 80px;
  display: flex;
  flex-direction: column;
  gap: var(--spacing-xs, 4px);
}

.color-option {
  flex: 0 0 auto;
}

.duplicate-warning {
  color: var(--danger, #ff6b6b);
  font-size: 11px;
  padding: 6px 10px;
  background: var(--dangerBg, rgba(255, 107, 107, 0.15));
  border-radius: var(--borderRadiusSmall, 6px);
  border: 1px solid var(--danger, #ff6b6b);
  animation: shake 0.3s ease-in-out;
}

.input-error {
  border-color: var(--danger, #ff6b6b) !important;
  box-shadow: 0 0 0 2px var(--dangerBg, rgba(255, 107, 107, 0.15)) !important;
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-3px); }
  75% { transform: translateX(3px); }
}
</style>
