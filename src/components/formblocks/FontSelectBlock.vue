<script lang="ts">
import { defineComponent } from "vue";
import FileSelect from "../inputs/FileSelect.vue";
import Checkbox from "../inputs/Checkbox.vue";
import Input from "../inputs/Input.vue";
import Fieldset from "../inputs/Fieldset.vue";
import Space from "../global/Space.vue";
import Text from "../icons/Text.vue";
import fonts, { fontStatusWatcher } from "../../constants/fonts";

const validateFont = (font: string): boolean => {
  const s = new Option().style;
  s.font = font;
  return s.font !== "";
};

export default defineComponent({
  components: {
    Checkbox, Input, Space, Fieldset, FileSelect, Text,
  },
  props: {
    modelValue: { type: String, required: true },
    showDetails: { type: Boolean, required: true },
  },
  emits: [
    "update:modelValue",
  ],
  data: (props) => ({
    fonts,
    stringValue: props.modelValue,
    stringIsValid: true,
    rerenderKey: 0,
  }),
  watch: {
    modelValue: {
      handler() {
        this.stringValue = this.modelValue;
        this.stringIsValid = true;
      },
    },
    stringValue: {
      handler() {
        this.stringIsValid = validateFont(this.stringValue);
        if (this.stringIsValid) {
          this.$emit("update:modelValue", this.stringValue);
        }
      },
    },
  },
  mounted() {
    fontStatusWatcher.on("load", () => {
      this.rerenderKey += 1;
    });
  },
  methods: {
    onLoadFont(font: FontFace) {
      font.load().then(() => {
        document.fonts.add(font);
        this.$emit("update:modelValue", `bold 1em ${font.family}`);
      });
    },
  },
});
</script>

<template>
  <Space :key="rerenderKey" vertical xlarge full>
    <Fieldset label="フォント選択">
      <select
        :value="modelValue"
        @change="$emit('update:modelValue', $event.target.value)"
        class="font-select"
      >
        <!-- <option
          v-for="category in fonts"
          :key="category.label"
          disabled
        >{{ category.label }}</option> -->
        <optgroup
          v-for="category in fonts"
          :key="category.label + '-group'"
          :label="category.label"
        >
          <option
            v-for="font in category.fonts"
            :key="font.label"
            :value="font.value"
            :style="{ font: font.value, lineHeight: 1 }"
          >
            {{ font.loaded ? font.label : 'loading ...' }}
          </option>
        </optgroup>
      </select>
    </Fieldset>
    <Fieldset label="その他のフォント">
      <Space vertical>
        <FileSelect type="font" name="ファイルを選ぶ" @load="onLoadFont">
          <Text /> ファイルを選ぶ
        </FileSelect>
        <Input
            v-if="showDetails"
            v-model="stringValue"
            name="font-spec"
            block :error="!stringIsValid" />
      </Space>
    </Fieldset>
  </Space>
</template>

<style scoped>
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

.font-select optgroup {
  font-weight: bold;
  color: #aaa;
  background-color: #1a1a1a;
}
</style>
