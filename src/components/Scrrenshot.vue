<template>
    <div>
        <el-upload class="upload-wrapper" ref="uploadRef" v-model:file-list="uploadFiles" list-type="picture-card"
            :auto-upload="false" :limit="3" style="width: auto" :on-preview="onPreview">
            <template #trigger>ADD</template>
            <el-button @click="onScreenshot" type="success">截图</el-button>
            <el-dialog class="preview-dialog" v-model="isPreviewImg" title="图片预览" width="80%">
                <img class="preview-img" :src="previewImg" alt="预览图片" />
            </el-dialog>
        </el-upload>
        <div class="screenshot-preview">
            <img v-if="isShowSreenshot && screenshotUrl" :src="screenshotUrl" alt="截图" />
        </div>

    </div>
</template>

<script>
export default {
    name: 'Scrrenshot',
    data() {
        return {
            uploadFiles: [],
            isPreviewImg: false,
            previewImg: '',
            isShowSreenshot: false,
            screenshotUrl: '',
            stream: null,
            originImage: null
        }
    },
    methods: {
        async onScreenshot() {
            console.log('截图功能待实现')
            try {
                this.stream = await navigator.mediaDevices.getDisplayMedia({
                    video: {
                        mdeiaSource: 'screen',
                        cursor: 'motion'
                    }
                });
                const video = document.createElement('video');
                video.srcObject = this.stream;
                video.play();
                video.onloadedmetadata = () => {
                    const canvas = document.createElement('canvas');
                    canvas.width = video.videoWidth;
                    canvas.height = video.videoHeight;
                    const ctx = canvas.getContext('2d');
                    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
                    this.screenshotUrl = canvas.toDataURL('image/png');
                    this.originImage = new Image();
                    this.originImage.src = this.screenshotUrl;
                    this.isShowSreenshot = true;
                }
            } catch (error) {
                console.error(error);
            } finally {
                this.stopCapture();
            }
        },
        stopCapture() {
            this.stream.getTracks().forEach(track => track.stop());
            this.stream = null;
        },
        onPreview(file) {
            console.log("### 图片")
            this.previewImg = file.url;
            this.isPreviewImg = true;
        }
    }
}
</script>

<style scoped>
.screenshot-preview {
    position: fixed;
    width: 600px;
    height: 400px;
    top: 20px;
    left: 20px;
    background-color: #bbb;
    z-index: 10;
    border: 4px dashed #165151;
    box-sizing: border-box;
}

/* 自定义上传组件图片预览框大小 */
:deep(.el-upload-list--picture-card .el-upload-list__item) {
    width: 100px;
    height: 100px;
}

/* 自定义上传按钮框大小 */
:deep(.el-upload--picture-card) {
    width: 100px;
    height: 100px;
}

/* 调整图片大小以适应容器 */
:deep(.el-upload-list--picture-card .el-upload-list__item-thumbnail) {
    object-fit: cover;
}

/* 预览对话框样式 */
.preview-img {
    max-width: 100%;
    max-height: 80vh;
    display: block;
    margin: 0 auto;
}
</style>
