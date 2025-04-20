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
    emojiName: {type: String, default: null}
  },
  emits: [
    "toggleShowTarget",
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
  },
  methods: {
    onDownload(): void {
      const download = prepareDownloadFile(this.images);
      const filename = filenamify(this.name ?? "", {replacement: ""}).normalize() || "megamoji";
      download.then((res) => saveAs(res, `${filename}.${extension(res)}`));
      Analytics.download();
    },
    reaction() : void {
      console.log("reactionClicked");

      // Get the first blob (assuming it's the one we want to send)
      // If you need a specific blob, adjust the indices accordingly
      if (this.images.length > 0 && this.images[0].length > 0) {
        const blob = this.images[0][0];

        // Convert blob to base64
        const reader = new FileReader();
        reader.onloadend = () => {
          const base64data = reader.result;
          // Send the base64 data to parent
          window.parent.postMessage(JSON.stringify({
            "source": "emoji-gen",
            "emojiName": this.emojiName,
            "binaryData": base64data
          }), "*");
        };
        reader.readAsDataURL(blob);

        setTimeout(() => {
          location.reload()
        }, 100)
      } else {
        // Fallback if no images
        window.parent.postMessage(JSON.stringify({ "source": "emoji-gen"}), "*");
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
      <button v-show="name" @click="reaction" style="position:absolute; right:0; bottom:0 ;background-color: darkgreen; height: 20px">
        リアクション
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
}
</style>
