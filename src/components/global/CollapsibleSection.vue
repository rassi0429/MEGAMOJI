<script lang="ts">
import { defineComponent } from "vue";

export default defineComponent({
  props: {
    label: { type: String, required: true },
    defaultOpen: { type: Boolean, default: false },
  },
  data() {
    return {
      isOpen: this.defaultOpen,
    };
  },
  methods: {
    toggle() {
      this.isOpen = !this.isOpen;
    },
  },
});
</script>

<template>
  <div class="collapsible-section">
    <div class="header" @click="toggle">
      <span class="arrow" :class="{ open: isOpen }">▶</span>
      <span class="title">{{ label }}</span>
    </div>
    <div class="content" v-show="isOpen">
      <slot />
    </div>
  </div>
</template>

<style scoped>
.collapsible-section {
  width: 100%;
}

.header {
  display: flex;
  align-items: center;
  cursor: pointer;
  margin-bottom: var(--spacingMedium);
  user-select: none;
}

.arrow {
  display: inline-block;
  margin-right: var(--spacingMedium);
  transition: transform 0.2s ease;
  font-size: var(--fontSizeSmall);
}

.arrow.open {
  transform: rotate(90deg);
}

.title {
  font-size: var(--fontSizeMedium);
  font-weight: bold;
  line-height: 1;
  color: var(--fg);
}

.content {
  padding-left: var(--spacingLarge);
  margin-bottom: var(--spacingLarge);
}
</style>
