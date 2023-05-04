<template>
  <div id="main">
    <div class="chatlog-upload">
      <n-upload
        v-if="!chatlog"
        :max="1"
        :show-file-list="false"
        directory-dnd
        accept="text/plain"
        ref="chatlog"
        @change="onChatlogUpload"
      >
        <n-upload-dragger>
          <div style="margin-bottom: 12px">
            <n-icon size="48" :depth="3">
              <chat-icon />
            </n-icon>
          </div>
          <n-text style="font-size: 16px">
            Выберите чатлог
          </n-text>
        </n-upload-dragger>
      </n-upload>
      <n-checkbox-group
        v-if="chatlog"
        v-model:value="selectedLinesToImage"
        class="chatlog-select"
      >
        <n-grid :y-gap="8" :cols="1">
          <n-gi v-for="(line, i) in chatlog" :key="i">
            <n-checkbox :value="line" :label="line" />
          </n-gi>
        </n-grid>
      </n-checkbox-group>
    </div>
    <div class="background-upload">
      <n-upload
        v-if="!background"
        :max="1"
        :show-file-list="false"
        directory-dnd
        accept="image/png, image/jpeg"
        @change="onBackUpload"
      >
        <n-upload-dragger>
          <div style="margin-bottom: 12px">
            <n-icon size="48" :depth="3">
              <image-icon />
            </n-icon>
          </div>
          <n-text style="font-size: 16px">
            Выберите фон
          </n-text>
        </n-upload-dragger>
      </n-upload>
      <canvas 
        v-if="background" 
        id="canvas" 
        ref="canvas"
      ></canvas>
    </div>
  </div>
</template>

<script>
import { defineComponent } from "vue";
import { ImageOutline as ImageIcon } from "@vicons/ionicons5";
import { DocumentTextOutline as ChatIcon } from "@vicons/ionicons5";

export default defineComponent({
  components: {
    ImageIcon,
    ChatIcon
  },
  data: () => ({
    chatlog: null,
    background: null,
    backgroundCanvas: null,
    selectedLinesToImage: null,
    regexTRP: {
      timestamp: /\[\d{2}:\d{2}:\d{2}\]/g,
      ic_chat: /(\w+)_(\w+)\s+сказал[а]?:\s*(.+)/g,
      silent_ic_chat: /(\w+)_(\w+)\s+тихо сказал[а]?:\s*(.+)/g,
      fraction_ic_chat: /\w+\s(\w+)_(\w+):\s(\W+!)/g,
      me_rp: /\*\s\w+_\w+\s\W+\s/g,
      do_rp: /\*\s\W+\s\(\(\s\w+_\w+\s\)\)/g,
      cdo_rp_chat: /\W+,\s\w+_\w+\sсказал[а]?:\s\W+\s/g,
      ame_rp: /!\s(\w+_\w+) \W+\s/g,
    },
  }),
  methods: {
    onBackUpload(files) {
      const reader = new FileReader();
      reader.onload = () => {
        this.background = new Image();
        this.background.onload = () => {
          const ctx = this.$refs.canvas.getContext('2d');
          const { width, height } = this.background;
          this.$refs.canvas.width = width;
          this.$refs.canvas.height = height;
          ctx.drawImage(this.background, 0, 0);
        }
        this.background.src = reader.result;
      }
      reader.readAsDataURL(files.fileList[0].file);
    },
    onChatlogUpload(files) {
      const file = files.fileList[0].file;
      if (file.type !== 'text/plain') return this.$refs.chatlog.clearFiles();

      const reader = new FileReader();
      reader.onload = () => {
        const all_lines = reader.result.split(/\r?\n/);
        const rp_lines = [];

        for (let line of all_lines) this.isRPLine(line) && rp_lines.push(line)

        this.chatlog = rp_lines;
      }
      reader.readAsText(file, 'windows-1251');
    },
    isRPLine(line) {
      const { 
        ic_chat, 
        silent_ic_chat, 
        fraction_ic_chat,
        me_rp,
        do_rp,
        cdo_rp_chat, 
        ame_rp, 
      } = this.regexTRP;
      
      return cdo_rp_chat.test(line) || ic_chat.test(line) || silent_ic_chat.test(line) || fraction_ic_chat.test(line) || me_rp.test(line) || do_rp.test(line) || ame_rp.test(line);
    },
    addLinesToCanvas(lines) {
      const { timestamp } = this.regexTRP;
      const { width, height } = this.$refs.canvas;
      const ctx = this.$refs.canvas.getContext("2d");

      ctx.clearRect(0, 0, width, height); // очищаем полотно перед всеми изменениями от предыдущих
      ctx.drawImage(this.background, 0, 0); // заливаем первостепенно фон
      ctx.font = "18px Verdana";
      ctx.fillStyle = "white"

      lines.forEach((line, i) => {
        const rp_line = line.replace(timestamp, '');
        ctx.fillText(rp_line, 20, i * 20 + 50);
      })

      // this.addWatermarkToCanvas()
    },
    addWatermarkToCanvas() {
      const { width } = this.$refs.canvas;
      const ctx = this.$refs.canvas.getContext('2d');

      ctx.font = "bold 20px system-ui";
      ctx.fillStyle = '#ffffff60';

      ctx.fillText('samp-ss.online', width - 180, 40)
    },
  },
  watch: {
    selectedLinesToImage(lines) {
      this.addLinesToCanvas(lines)
    }
  }
});
</script>

<style lang="stylus">
#main
  padding: 10px
  display flex
  gap 10px
.background-upload
  width 75%
  position relative
.chatlog-upload
  width 25%
.lines-select
  width 100%
  height 100%
#canvas
  width 100%
  height 98vh
  object-fit contain
</style>