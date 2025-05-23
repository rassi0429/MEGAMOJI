<script lang="ts">
import {defineComponent, PropType} from "vue";
import Analytics from "../../utils/analytics";
import EffectBlock from "../formblocks/EffectBlock.vue";
import CellcountBlock from "../formblocks/CellcountBlock.vue";
import Button from "../inputs/Button.vue";
import Select from "../inputs/Select.vue";
import Checkbox from "../inputs/Checkbox.vue";
import NumberInput from "../inputs/Number.vue";
import Slider from "../inputs/Slider.vue";
import Fieldset from "../inputs/Fieldset.vue";
import Color from "../inputs/Color.vue";
import Space from "../global/Space.vue";
import Card from "../global/Card.vue";
import Grid from "../global/Grid.vue";
import GridItem from "../global/GridItem.vue";
import CollapsibleSection from "../global/CollapsibleSection.vue";
import DevTool from "./DevTool.vue";

import {Animation, Effect, WebGLEffect} from "../../types";
import animations from "../../constants/animations";
import effects from "../../constants/effects";
import bgeffects from "../../constants/bgeffects";
import staticeffects from "../../constants/staticeffects";
import webgleffects from "../../constants/webgleffects";
import easings from "../../constants/easings";

import {renderAllCells} from "../../utils/emoji";
import {
  EMOJI_SIZE,
  ANIMATED_EMOJI_SIZE,
  BINARY_SIZE_LIMIT,
  FRAMERATE_MAX,
  FRAMECOUNT_MAX,
} from "../../constants/emoji";

import {NODE_ENV} from "../../utils/env";

type AnimationOption = { label: string, value: Animation };
type EffectOption = { label: string, value: Effect };
type WebGLEffectOption = { label: string, value: WebGLEffect };
type SpeedOption = { label: string, value: number };

const TRIMMING_OPTIONS = [
  {label: "ぴっちり", value: ""},
  // { label: "はみだす (アス比維持)", value: "cover" },
  {label: "おさめる (アス比維持)", value: "contain"},
  {label: "そのまま (長方形)", value: "stretch"},
];

const SPEED_OPTIONS = [
  {label: "コマ送り", value: 2.0},
  {label: "ゆっくり", value: 1.3},
  {label: "ふつう", value: 0.8},
  {label: "はやい", value: 0.3},
];

export default defineComponent({
  components: {
    Color,
    EffectBlock,
    Checkbox,
    CellcountBlock,
    Card,
    Button,
    Number: NumberInput,
    Grid,
    GridItem,
    Fieldset,
    Space,
    Select,
    Slider,
    CollapsibleSection,
    DevTool,
  },
  props: {
    baseImage: {type: Object as PropType<HTMLImageElement | HTMLCanvasElement>, default: null},
    show: {type: Boolean, required: true},
    emojiSize: {type: Number, default: null},
  },
  emits: [
    "render",
    "update:emojiSize",
  ],
  data() {
    return {
      animations,
      effects,
      bgeffects,
      staticeffects,
      webgleffects,
      easings,
      TRIMMING_OPTIONS,
      SPEED_OPTIONS,
      isDev: NODE_ENV === "development",
      conf: {
        /* basic */
        trimming: TRIMMING_OPTIONS[0],
        targetAspect: 1,
        speed: SPEED_OPTIONS[2],
        cells: [1, 1],
        animation: null as (AnimationOption | null),
        animationInvert: false,
        staticEffects: [] as EffectOption[],
        effects: [] as EffectOption[],
        webglEffects: [] as WebGLEffectOption[],
        /* advanced */
        trimH: [0, 0],
        trimV: [0, 0],
        noCrop: false,
        easing: easings[0],
        duration: SPEED_OPTIONS[2].value,
        backgroundColor: "#ffffff",
        transparent: true,
      },
      showDetails: true,
      devMode: false,
      /* internals */
      running: false,
      dirty: false,
    };
  },
  computed: {
    naturalAspect(): number {
      return (this.conf.trimH[1] - this.conf.trimH[0]) / (this.conf.trimV[1] - this.conf.trimV[0]);
    },
  },
  watch: {
    baseImage: {
      handler(): void {
        if (this.baseImage) {
          this.refreshDefaultSettings();
          this.render(true);
        }
      },
    },
    emojiSize: {
      handler(): void {
        if (this.baseImage) {
          this.render(true);
        }
      },
    },
    conf: {
      handler(): void {
        const animationName = this.conf.animation ? this.conf.animation.label : "";
        const effectNames = [
          ...this.conf.staticEffects.map((e) => e.label),
          ...this.conf.effects.map((e) => e.label),
          ...this.conf.webglEffects.map((e) => e.label),
        ];
        Analytics.changeAnimation(animationName, effectNames);
        this.render(true);
      },
      deep: true,
    },
    devMode: {
      handler(): void {
        this.conf.animation = null;
        this.conf.effects = [];
        this.conf.webglEffects = [];
      },
    },
  },
  mounted() {
    Analytics.changeAnimation("", []);
  },
  methods: {
    refreshDefaultSettings(): void {
      if (this.baseImage) {
        const image = this.baseImage;
        const height = image instanceof HTMLImageElement ? image.naturalHeight : image.height;
        const width = image instanceof HTMLImageElement ? image.naturalWidth : image.width;
        const hCells = this.conf.cells[0];
        const vCells = this.conf.cells[1];
        let widthPerCell = width / hCells;
        let heightPerCell = height / vCells;
        let aspect = 1;
        if (this.conf.trimming.value === "cover") {
          widthPerCell = Math.min(widthPerCell, heightPerCell);
          heightPerCell = widthPerCell;
        } else if (this.conf.trimming.value === "contain") {
          widthPerCell = Math.max(widthPerCell, heightPerCell);
          heightPerCell = widthPerCell;
        } else if (this.conf.trimming.value === "stretch") {
          aspect = width / height;
        }
        const offsetH = Math.floor((width - widthPerCell * hCells) / 2);
        const offsetV = Math.floor((height - heightPerCell * vCells) / 2);
        this.conf.trimH = [offsetH, width - offsetH];
        this.conf.trimV = offsetV < 0 ? (
            [offsetV, height - offsetV]
        ) : (
            [0, height - offsetV * 2]
        );
        this.conf.targetAspect = aspect;
      }
    },
    selectSpeed(speed: SpeedOption): void {
      this.conf.duration = speed.value;
    },
    toggleAutoSize(value: boolean): void {
      this.$emit("update:emojiSize", value ? null : EMOJI_SIZE);
    },
    changeEmojiSize(value: number): void {
      this.$emit("update:emojiSize", value);
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
      if (this.baseImage) {
        const animated = !!(
            this.conf.animation
            || this.conf.effects.length
            || this.conf.webglEffects.length
        );

        const framerate = Math.min(FRAMERATE_MAX, Math.ceil(FRAMECOUNT_MAX / this.conf.duration));
        const framecount = Math.floor(this.conf.duration * framerate);

        const maxSize = this.emojiSize || (animated ? ANIMATED_EMOJI_SIZE : EMOJI_SIZE);
        const aspectCoef = Math.sqrt(this.conf.targetAspect);
        const binarySizeLimit = this.emojiSize ? Infinity : BINARY_SIZE_LIMIT;
        renderAllCells(
          this.baseImage,
            this.conf.trimH[0],
            this.conf.trimV[0],
            this.conf.cells[0],
            this.conf.cells[1],
            this.conf.trimH[1] - this.conf.trimH[0],
            this.conf.trimV[1] - this.conf.trimV[0],
            Math.max(maxSize * aspectCoef, 1),
            Math.max(maxSize / aspectCoef, 1),
            this.conf.noCrop,
            animated,
            this.conf.animation ? this.conf.animation.value : null,
            this.conf.animationInvert,
            this.conf.effects.concat(this.conf.staticEffects).map((eff) => eff.value),
            this.conf.webglEffects.map((eff) => eff.value),
            this.conf.easing.value,
            framerate,
            framecount,
            this.conf.backgroundColor,
            this.conf.transparent,
            binarySizeLimit,
        ).then((res) => {
          this.$emit("render", res);
          this.running = false;
          this.render();
        });
      }
    },
  },
});
</script>

<template>
  <div style="display: flex; flex-direction: column; gap: 1em">
    <Fieldset label="背景色">
      <Space vertical full>
        <Color
            v-model="conf.backgroundColor"
            name="背景色"
            block
            @update:model-value="conf.transparent = false"/>
        <Checkbox v-model="conf.transparent" name="背景色(透過)">
          {{ "透過" }}
        </Checkbox>
      </Space>
    </Fieldset>
    <Fieldset v-if="showDetails" label="絵文字の横幅">
      <Select
          v-model="conf.trimming"
          name="切り抜き"
          :options="TRIMMING_OPTIONS"
          @update:model-value="refreshDefaultSettings"/>
    </Fieldset>
    
    <CollapsibleSection label="アス比" :default-open="false">
    <Fieldset v-if="showDetails" label="アス比">
            <Slider
                v-model="conf.targetAspect"
                block
                nonzero
                :step="0.01"
                :marks="[1, naturalAspect]"
                :min="Math.min(0.2, naturalAspect)"
                :max="Math.max(5, naturalAspect)" />
          </Fieldset>
        </CollapsibleSection>
    <CollapsibleSection label="アニメーション" :default-open="false">
      <Space vertical full>
        <Select
            v-model="conf.animation"
            name="アニメーション"
            nullable :options="animations"/>
        <Checkbox v-model="conf.animationInvert" name="逆再生">
          {{ "逆再生" }}
        </Checkbox>

        <Fieldset v-if="showDetails" label="速度 (アニメ)">
          <Select
              v-model="conf.speed"
              name="速度(アニメ)"
              :options="SPEED_OPTIONS"
              @update:model-value="selectSpeed($event)"/>
        </Fieldset>
        <Fieldset v-if="showDetails" label="長さ (アニメ)">
          <Slider
              v-model="conf.duration"
              block
              :min="0.1"
              :step="0.1"
              :max="2.0"/>
        </Fieldset>
        <Fieldset v-if="showDetails" label="イージング (アニメ)">
          <Select v-model="conf.easing" name="イージング" :options="easings"/>
        </Fieldset>
      </Space>
    </CollapsibleSection>
    <CollapsibleSection label="特殊効果とか" :default-open="false">
      <EffectBlock v-model="conf.webglEffects" :effects="webgleffects"/>
      <EffectBlock v-model="conf.effects" :effects="effects"/>
      <EffectBlock v-if="showDetails" v-model="conf.effects" :effects="bgeffects"/>
      <EffectBlock v-model="conf.staticEffects" :effects="staticeffects"/>
    </CollapsibleSection>
  </div>
</template>
