# front

> My astounding Nuxt.js project

## Build Setup

```bash
# install dependencies
$ npm install

# serve with hot reload at localhost:3000
$ npm run dev

# build for production and launch server
$ npm run build
$ npm run start

# generate static project
$ npm run generate
```

For detailed explanation on how things work, check out [Nuxt.js docs](https://nuxtjs.org).
<br>

<hr>

## 功能

#### 通过文件头标识判断图片格式

> 通过 File 对象的 type 去判断图片格式，在手动更改图片格式的情况下不准确，所以需要通过文件头标识判断

```js
async blobToString(blob) {
      return new Promise(resolve => {
        const reader = new FileReader();
        reader.onload = function() {
          const ret = reader.result
            .split("")
            .map(v => v.charCodeAt())
            .map(v => v.toString(16).toUpperCase())
            .map(v => v.padStart(2, "0"))
            .join(" ");
          resolve(ret);
        };
        reader.readAsBinaryString(blob);
      });
    },

    async isGif(file) {
      // GIF89a GIF87a
      // 前6个十六进制 '47 49 46 38 39 61'  '47 49 46 38 37 61'
      const ret = await this.blobToString(file.slice(0, 6));
      const isGif = ret === "47 49 46 38 39 61" || ret === "47 49 46 38 37 61";

      return isGif;
    },

    async isPng(file) {
      // 前8个十六进制 '89 50 4E 47 0D 0A 1A 0A'
      const ret = await this.blobToString(file.slice(0, 8));
      const ispng = ret === "89 50 4E 47 0D 0A 1A 0A";

      return ispng;
    },

    async isJpg(file) {
      // 前2个十六进制'FF D8' 后2个十六进制 'FF D9'
      const len = file.size;
      const start = await this.blobToString(file.slice(0, 2));
      const tail = await this.blobToString(file.slice(-2, len));
      const isJpg = start === "FF D8" && tail === "FF D9";

      return isJpg;
    },
```

#### 计算文件 hash

> 在文件很大的情况下，需要对文件进行切片，然后计算文件 hash, 详见 `/pages/uc.vue`

- Web Worker
  - 借助 `spark-md5`进行计算
- 利用浏览器渲染每帧的空余时间(`window.requestIdleCallback()`)
- 抽样Hash

#### 切片上传
#### 后端文件合并
#### 秒传
#### 断点续传
