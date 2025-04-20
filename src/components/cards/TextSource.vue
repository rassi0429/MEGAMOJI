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
  <Card v-if="show">
    <Grid :columns="[[450, 1], [Infinity, 3]]" spaced>
      <GridItem>
        <FontSelectBlock
            v-model="conf.font"
            :show-details="showDetails"/>
      </GridItem>
      <GridItem :span="2">
        <Space vertical xlarge full>
          <Fieldset label="絵文字名(アルファベット)">
              <Input
                  :value="conf.emojiName"
                  name="絵文字名"
                  block
                  autofocus
                  :rows="1"
                  @input="validateEmojiName($event.target.value)"
                  @keydown="preventNonAlphanumeric"
                  @compositionend="handleCompositionEnd"/>
          </Fieldset>
          <Fieldset label="テキスト (改行可)">
            <Space vertical full>
              <Textarea
                  v-model="conf.content"
                  name="テキスト"
                  block
                  autofocus
                  :rows="2"/>
              <div>
                <select
                    v-model="conf.align"
                    class="font-select"
                >
                  <option value="stretch">両端揃え</option>
                  <option value="center">中央揃え</option>
                  <option value="left">左揃え</option>
                  <option value="right">右揃え</option>
                </select>
                <FontColorSelectBlock
                    v-model="conf.color"
                    v-model:gradient="conf.gradient"
                    :show-details="true"/>
              </div>
            </Space>
          </Fieldset>
          <OutlineBlock
              v-model="conf.outlines"
              v-model:thickness="conf.outlineThickness"
              v-model:posX="conf.outlineX"
              v-model:posY="conf.outlineY"
              :base-color="conf.color"
              :show-details="showDetails"/>
          <Fieldset v-if="showDetails" label="行間 (文字分)">
            <Slider
                v-model="conf.lineSpacing"
                block
                :min="0"
                :max="1"
                :step="0.01"/>
          </Fieldset>
          <Fieldset v-if="showDetails" label="余白">
            <Slider
                v-model="conf.paddingValue"
                block
                :min="0"
                :max="0.5"
                :step="0.01"/>
          </Fieldset>
          <!-- <Fieldset v-else label="余白">
            <Select
                v-model="conf.padding"
                name="余白"
                :options="PADDING_OPTIONS"
                @update:model-value="selectPadding($event)" />
          </Fieldset> -->
        </Space>
      </GridItem>
    </Grid>
    <!-- <template #footer>
      <Checkbox v-model="showDetails" name="職人モード(テキスト)">
        {{ "職人モード" }}
      </Checkbox>
    </template> -->
  </Card>
</template>

<style scoped>
.font-preview {
  height: 1em;
  vertical-align: baseline;
}

.font-select {
  width: 100%;
  font-size: 1em;
  background-color: #111;
  color: #fff;
  padding: 0.6em 0.8em;
  border: 1px solid #333;
  border-radius: 4px;
  appearance: none;
  cursor: pointer;
  transition: border-color 0.2s, box-shadow 0.2s;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%23888' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='6 9 12 15 18 9'%3E%3C/polyline%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 0.8em center;
  background-size: 1em;
}

.font-select:hover {
  border-color: #555;
}

.font-select:focus {
  outline: none;
  border-color: #666;
  box-shadow: 0 0 0 2px rgba(100, 100, 100, 0.3);
}

.font-select option {
  padding: 0.6em;
  background-color: #222;
}
</style>
