<template>
    <div>
        <h1>用户中心</h1>
        <div ref="drag" id="drag">
            <input type="file" name="file" @change="handeFileChange" />
        </div>
        <el-progress :stroke-width="20" :text-inside="true" :percentage="uploadProgress"></el-progress>
        <div>
            <h2>计算Hash进度</h2>
            <el-progress :stroke-width="20" :text-inside="true" :percentage="hashProgress"></el-progress>
        </div>
        <el-button @click="uploadFile">上传</el-button>

        <!-- chunk.progress 
        progress <0 报错 红色
        = 100 成功
        别的数字，方块高度显示-->
        <div class="cube-container" :style="{'width': cubeWidth + 'px'}">
            <div class="cube" v-for="chunk in chunks" :key="chunk.name">
                <div
                    :class="{
               'uploading': chunk.progress > 0 && chunk.progress < 100,
               'success': chunk.progress == 100,
               'error': chunk.progress < 0
             }"
                    :style="{'height': chunk.progress + '%'}"
                >
                    <i
                        v-if="chunk.progress < 100 && chunk.progress > 0"
                        class="el-icon-loading"
                        style="color: #f56c6c;"
                    ></i>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
const CHUNK_SIZE = 0.1 * 1024 * 1024;
import sparkMD5 from "spark-md5";

export default {
    data() {
        return {
            file: null,
            // uploadProgress: 0,
            hashProgress: 0,
            chunks: []
        };
    },
    async mounted() {
        const ret = await this.$http.get("/user/info");
        console.log(ret);

        this.bindEvents();
    },
    computed: {
        cubeWidth() {
            return Math.ceil(Math.sqrt(this.chunks.legnth)) * 16;
        },
        uploadProgress() {
            if (!this.file || this.chunks.length) {
                return 0;
            }
            let loaded = this.chunks
                .map(item => item.chunk.size * item.progress)
                .reduce((ac, cur) => acc + cur, 0);
            return Number(((loaded * 100) / this.file.size).toFixed(2));
        }
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
            const isGif =
                ret === "47 49 46 38 39 61" || ret === "47 49 46 38 37 61";

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
                chunks.push({
                    index: cur,
                    file: this.file.slice(cur, cur + size)
                });
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
                    while (
                        count < chunks.length &&
                        deadline.timeRemaining() > 1
                    ) {
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

        // 抽样hash
        async calcuteHashSample() {
            // 参考 布隆过滤器 损失部分精度换取效率 判断一个数据存在与否
            // 1个G的文件，抽样后5M以内
            // hash一样，文件不一定一样
            // hash不一样，文件一定不一样
            return new Promise(resolve => {
                const spark = new sparkMD5.ArrayBuffer();
                const reader = new FileReader();

                const file = this.file;
                const size = file.size;
                const offset = 2 * 1024 * 1024;
                // 第一个2M， 最后区块数据全要
                let chunks = [file.slice(0, offset)];

                let cur = offset;
                while (cur <= offset) {
                    if (cur + offset >= size) {
                        // 最后一个区块
                        chunks.push(file.slice(cur, cur + offset));
                    } else {
                        // 中间区块, 取前中后各2个字节
                        const mid = cur + offset / 2;
                        const end = cur + offset;
                        chunks.push(file.slice(cur, cur + 2));
                        chunks.push(file.slice(mid, mid + 2));
                        chunks.push(file.slice(end - 2, end));
                    }
                    cur += offset;
                }
                reader.readAsArrayBuffer(new Blob(chunks));
                reader.onload = e => {
                    spark.append(e.target.result);
                    this.hashProgress = 100;
                    resolve(spark.end());
                };
            });
        },
        async uploadFile() {
            // if (!(await this.isImage(this.file))) {
            //   console.log("图片格式不正确");
            //   return;
            // }
            const chunks = this.createFileChunk(this.file);

            // const hash = await this.calculateHashWorker();
            // const hash1 = await this.calculateHashIdle();
            // const hash2 = await this.calcuteHashSample();

            // console.log("文件hash", hash);
            // console.log("文件hash1", hash1);
            // console.log("文件hash2", hash2);
            const hash = await this.calcuteHashSample();
            this.hash = hash;

            // 问一下后端，文件是否上传过，如果没有，是否有存在的切片
            const {
                data: { uploaded, uploadedList }
            } = await this.$http.post("/checkfile", {
                hash: this.hash,
                ext: this.file.name.split(".").pop()
            });

            if (uploaded) {
                return this.$message.success("秒传成功");
            }

            this.chunks = chunks.map((chunk, index) => {
                // 切片的名字 hash + index
                const name = hash + "-" + index;
                return {
                    hash,
                    name,
                    index,
                    chunk: chunk.file,
                    // 设置进度条，已经上传的，设为100
                    progress: uploadedList.indexOf(name) > -1 ? 100 : 0
                };
            });
            console.log("chunks", this.chunks);
            await this.uploadChunks(uploadedList);

            // return;

            // const form = new FormData();
            // form.append("name", "file");
            // form.append("file", this.file);

            // const ret = await this.$http.post("/uploadfile", form, {
            //     // 上传进度条显示
            //     onUploadProgress: progress => {
            //         this.uploadProgress = Number(
            //             ((progress.loaded / progress.total) * 100).toFixed(2)
            //         );
            //     }
            // });
            // console.log(ret);
        },
        async uploadChunks(uploadedList) {
            console.log("uploadedList", uploadedList);
            const requests = this.chunks
                .filter(chunk => uploadedList.indexOf(chunk.name) === -1)
                .map((chunk, index) => {
                    const form = new FormData();
                    form.append("chunk", chunk.chunk);
                    form.append("hash", chunk.hash);
                    form.append("name", chunk.name);
                    return {
                        form,
                        index: chunk.index,
                        error: 0
                    };
                });
            // .map(({form, index}) =>
            //     this.$http.post("/uploadfile", form, {
            //         onUploadProgress: progress => {
            //             // 每个区块的进度条
            //             this.chunks[index].progress = Number(
            //                 (
            //                     (progress.loaded / progress.total) *
            //                     100
            //                 ).toFixed(2)
            //             );
            //         }
            //     })
            // );
            // @to-do并发控制
            // 尝试申请tcp连接过多，也会造成卡顿
            // 异步的并发数控制
            // await Promise.all(requests);
            await this.sendRequest(requests);
            // 给后端发送文件合并请求
            this.mergeRequest();
        },
        // 上传可能报错
        // 报错之后进度条变红，开始重试
        // 一个切片重试失败三次，整体全部终止
        async sendRequest(chunks, limit = 3) {
            return new Promise((resolve, reject) => {
                const len = chunks.length;
                let counter = 0;
                let isStop = false;

                const start = async () => {
                    if(isStop) {
                        return;
                    }
                    
                    const task = chunks.shift();
                    if (task) {
                        const { form, index } = task;

                        try {
                            await this.$http.post("/uploadfile", form, {
                                onUploadProgress: progress => {
                                    // 每个区块的进度条
                                    this.chunks[index].progress = Number(
                                        (
                                            (progress.loaded / progress.total) *
                                            100
                                        ).toFixed(2)
                                    );
                                }
                            });

                            if (counter == len - 1) {
                                // 最后一个任务
                                resolve();
                            } else {
                                counter++;
                                // 启动下一个任务
                                start();
                            }
                        } catch (e) {
                            this.chunks[index].progress = -1;
                            if(task.error < 3) {
                                task.error++;
                                chunks.unshift(task); // 出错后马上重试
                                start();
                            } else {
                                // 错误三次
                                isStop = true;
                                reject();
                            }
                        }
                    }
                };
                while (limit > 0) {
                    // 启动limit个任务
                    // 模拟一下延时
                    setTimeout(() => {
                        start();
                    }, Math.random() * 2000);

                    limit -= 1;
                }
            });
        },
        async mergeRequest() {
            this.$http.post("/mergefile", {
                ext: this.file.name.split(".").pop(),
                size: CHUNK_SIZE,
                hash: this.hash
            });
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

.cube-container .cube {
    width: 14px;
    height: 14px;
    line-height: 12px;
    border: 1px solid #000;
    background: #eee;
    float: left;
}

.cube-container .cube .success {
    background-color: green;
}
.cube-container .cube .error {
    background-color: red;
}
</style>
