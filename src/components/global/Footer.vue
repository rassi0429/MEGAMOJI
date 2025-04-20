<script lang="ts">
import { defineComponent } from "vue";
import Button from "../inputs/Button.vue";

export default defineComponent({
  components: {
    Button,
  },
  methods: {
    backstage() {
      window.open("https://scrapbox.io/MEGAMOJI/%F0%9F%93%8C_%E3%81%93%E3%82%8C%E3%81%AF%E3%81%AA%E3%81%AB%EF%BC%9F");
    },
    donation() {
      window.open("https://github.com/zk-phi/MEGAMOJI?tab=License-1-ov-file#readme");
    },
    share() {
      window.open("http://twitter.com/share?url=https://zk-phi.github.io/MEGAMOJI&text=カスタム絵文字メーカー%20MEGAMOJI&hashtags=メガ文字");
    },
  },
});
</script>

<template>
  <footer class="footer">
    <p class="p">
      Built with ♡ by zk-phi
      <Button type="text" @click="donation">
        ライセンス
      </Button>
<!--      /-->
<!--      <Button type="text" @click="backstage">-->
<!--        <template #icon>-->
<!--          🛠-->
<!--        </template>-->
<!--        開発室-->
<!--      </Button>-->
<!--      /-->
<!--      <Button type="text" @click="share">-->
<!--        <template #icon>-->
<!--          🤝-->
<!--        </template>-->
<!--        Xでシェア-->
<!--      </Button>-->
    </p>
<!--    <p class="p">-->
<!--      このページでは Cookie (アクセス数などの集計に Google Analytics、エラーの検知に Rollbar) を使用しています。-->
<!--    </p>-->
    <p class="p footnote">
      Peace, Love, Unity, Respect 🫶
    </p>
  </footer>
</template>

<style scoped>
.footer {
  font-size: var(--fontSizeMedium);
  line-height: var(--multilineTextLineHeight);
  color: var(--fg);
}

.footnote {
  font-style: italic;
  color: var(--border);
}

.p {
  margin: 0 0 var(--spacingMedium);
}
</style>
