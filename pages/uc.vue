<template>
  <div>
    <h1>用户中心</h1>
    <div ref="drag" id="drag">
      <input type="file" name="file" @change="handeFileChange" />
    </div>
    <el-progress
      :stroke-width="20"
      :text-inside="true"
      :percentage="uploadProgress"
    ></el-progress>
    <div>
      <h2>计算Hash进度</h2>
      <el-progress
        :stroke-width="20"
        :text-inside="true"
        :percentage="hashProgress"
      ></el-progress>
    </div>
    <el-button @click="uploadFile">上传</el-button>
  </div>
</template>

<script>
const CHUNK_SIZE = 0.1 * 1024 * 1024;
import sparkMD5 from "spark-md5";

export default {
  data() {
    return {
      file: null,
      uploadProgress: 0,
      hashProgress: 0
    };
  },
  async mounted() {
    const ret = await this.$http.get("/user/info");
    console.log(ret);

    this.bindEvents();
  },
  methods: {
    // 拖拽上传
    bindEvents() {
      const drag = this.$refs.drag;
      console.log(drag);

      drag.addEventListener("dragover", e => {
        drag.style.borderColor = "red";
        e.preventDefault();
      });

      drag.addEventListener("dragleave", e => {
        drag.style.borderColor = "#eee";
        e.preventDefault();
      });

      drag.addEventListener("drop", e => {
        const fileList = e.dataTransfer.files;
        drag.style.borderColor = "#eee";
        e.preventDefault();
        console.log(fileList);
        this.file = fileList[0];
      });
    },
    handeFileChange(e) {
      console.log(e.target.files);
      const [file] = e.target.files;

      if (!file) return;

      this.file = file;
    },

    async blobToString(blob) {
      console.log("fisrt-blob ", blob);
      return new Promise(resolve => {
        const reader = new FileReader();
        reader.onload = function() {
          console.log("reader.result", reader.result);
          const ret = reader.result
            .split("")
            .map(v => v.charCodeAt())
            .map(v => v.toString(16).toUpperCase())
            .map(v => v.padStart(2, "0"))
            .join(" ");
          console.log("ret  ", ret);
          resolve(ret);
        };
        reader.readAsBinaryString(blob);
      });
    },

    async isGif(file) {
      // GIF89a GIF87a
      // 前六个十六进制 '47 49 46 38 39 61'  '47 49 46 38 37 61'
      const ret = await this.blobToString(file.slice(0, 6));
      console.log("blobToStringGif ", ret);
      const isGif = ret === "47 49 46 38 39 61" || ret === "47 49 46 38 37 61";

      return isGif;
    },

    async isPng(file) {
      // 前8个十六进制 89 50 4E 47 0D 0A 1A 0A
      const ret = await this.blobToString(file.slice(0, 8));
      console.log("blobToStringPng ", ret);
      const ispng = ret === "89 50 4E 47 0D 0A 1A 0A";
      return ispng;
    },

    async isJpg(file) {
      // 前2个十六进制'FF D8' 后2个十六进制 'FF D9'
      const len = file.size;
      const start = await this.blobToString(file.slice(0, 2));
      const tail = await this.blobToString(file.slice(-2, len));
      console.log("blobToStringJpg ", start, tail);
      const isJpg = start === "FF D8" && tail === "FF D9";
      return isJpg;
    },

    // 图片类型判断
    async isImage(file) {
      // 通过文件流来判断
      return (
        (await this.isGif(file)) ||
        (await this.isPng(file)) ||
        (await this.isJpg(file))
      );
    },
    // 文件切片
    createFileChunk(file, size = CHUNK_SIZE) {
      const chunks = [];
      let cur = 0;
      while (cur < this.file.size) {
        chunks.push({ index: cur, file: this.file.slice(cur, cur + size) });
        cur += size;
      }
      return chunks;
    },
    // 用web worker 生成文件hash
    async calculateHashWorker() {
      return new Promise(resolve => {
        this.worker = new Worker("/hash.js");
        this.worker.postMessage({ chunks: this.chunks });
        this.worker.onmessage = e => {
          console.log("e.data", e.data);
          const { progress, hash } = e.data;
          this.hashProgress = Number(progress.toFixed(2));
          if (hash) {
            resolve(hash);
          }
        };
      });
    },
    // 用 时间切片 生成文件hash
    async calculateHashIdle() {
      const chunks = this.chunks;

      return new Promise(resolve => {
        const spark = new sparkMD5.ArrayBuffer();
        let count = 0;

        const appendToSpark = async file => {
          return new Promise(resolve => {
            const reader = new FileReader();
            reader.readAsArrayBuffer(file);
            reader.onload = e => {
              spark.append(e.target.result);
              resolve();
            };
          });
        };

        const workLoop = async deadline => {
          while (count < chunks.length && deadline.timeRemaining() > 1) {
            // 空闲时间且有任务
            await appendToSpark(chunks[count].file);
            count++;
            if (count < chunks.length) {
              this.hashProgress = Number(
                ((100 * count) / chunks.length).toFixed(2)
              );
            } else {
              this.hashProgress = 100;
              resolve(spark.end());
            }
          }

          window.requestIdleCallback(workLoop);
        };
        window.requestIdleCallback(workLoop);
      });
    },
    async uploadFile() {
      //   if (!(await this.isImage(this.file))) {
      //     console.log("图片格式不正确");
      //     return;
      //   }
      this.chunks = this.createFileChunk(this.file);
      console.log("chunks", this.chunks);

      const hash = await this.calculateHashWorker();
      const hash1 = await this.calculateHashIdle();

      console.log("文件hash", hash);
      console.log("文件hash1", hash);

      return;

      const form = new FormData();
      form.append("name", "file");
      form.append("file", this.file);

      const ret = await this.$http.post("/uploadfile", form, {
        // 上传进度条显示
        onUploadProgress: progress => {
          this.uploadProgress = Number(
            ((progress.loaded / progress.total) * 100).toFixed(2)
          );
        }
      });
      console.log(ret);
    }
  }
};
</script>
<style>
#drag {
  height: 100px;
  line-height: 100px;
  border: 2px dashed #eee;
  text-align: center;
}
</style>
