<template>
    <div style="display: flex; flex-direction: column; gap:16px; align-items: center;">
        <el-radio-group v-model="annotationType" fill="#0d47a1">
            <el-radio-button label="rect" value="rect">矩形</el-radio-button>
            <el-radio-button label="arrow" value="arrow">箭头</el-radio-button>
        </el-radio-group>

        <el-button style="margin:24px 0;" @click="addAnnotatedScreenshot"
            :disabled="!isShowSreenshot">添加标注图到上传列表</el-button>

        <el-button @click="onScreenshot" type="success">截图</el-button>

        <el-upload class="upload-wrapper" ref="uploadRef" v-model:file-list="uploadFiles" list-type="picture-card"
            :auto-upload="false" :limit="3" style="width: auto" :on-preview="onPreview">
            <template #trigger>ADD</template>

            <el-dialog class="preview-dialog" v-model="isPreviewImg" title="图片预览" width="80%">
                <img class="preview-img" :src="previewImg" alt="预览图片" />
            </el-dialog>
        </el-upload>
        <div class="screenshot-preview" @mousedown="startDrawing" @mousemove="draw" @mouseup="stopDrawing">
            <img v-if="isShowSreenshot && screenshotUrl" :src="screenshotUrl" alt="截图" />
            <canvas v-if="isShowSreenshot && screenshotUrl" class="annotation-canvas" ref="annotationCanvas"></canvas>
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
            originImage: null,
            annotationType: 'rect',
            // 矩形标注相关
            isDrawing: false,
            startX: 0,
            startY: 0,
            annotations: []
        }
    },
    mounted() {
        // 监听originImage加载完成，设置canvas尺寸
        this.$nextTick(() => {
            if (this.originImage) {
                this.originImage.onload = () => {
                    this.initCanvas();
                };
            }
        });
    },
    methods: {
        async onScreenshot() {
            console.log('开始截图功能')
            try {
                this.stream = await navigator.mediaDevices.getDisplayMedia({
                    video: {
                        mediaSource: 'screen', // 修复拼写错误
                        cursor: 'motion'
                    }
                });

                const video = document.createElement('video');
                video.srcObject = this.stream;
                video.play();

                // 等待视频元数据加载完成
                await new Promise(resolve => {
                    video.onloadedmetadata = resolve;
                });

                const canvas = document.createElement('canvas');
                canvas.width = video.videoWidth;
                canvas.height = video.videoHeight;
                const ctx = canvas.getContext('2d');
                ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

                // 使用 nextTick 确保 DOM 更新
                this.$nextTick(() => {
                    this.screenshotUrl = canvas.toDataURL('image/png');
                    this.originImage = new Image();
                    this.originImage.src = this.screenshotUrl;
                    this.isShowSreenshot = true;

                    // 重置标注状态
                    this.annotations = [];

                    // 图片加载完成后初始化canvas
                    this.originImage.onload = () => {
                        this.initCanvas();
                    };
                });

                // 在图片处理完成后再停止流
                this.stopCapture();

            } catch (error) {
                console.error('截图失败:', error);
                alert('截图失败: ' + error.message);
            }
        },
        stopCapture() {
            this.stream.getTracks().forEach(track => track.stop());
            this.stream = null;
        },
        onPreview(file) {
            this.previewImg = file.url;
            this.isPreviewImg = true;
        },
        // 初始化标注canvas
        initCanvas() {
            const preview = this.$el.querySelector('.screenshot-preview');
            const canvas = this.$refs.annotationCanvas;
            const img = preview.querySelector('img');

            if (canvas && img) {
                // 设置canvas与图片相同的尺寸
                canvas.width = img.clientWidth;
                canvas.height = img.clientHeight;

                // 清除之前的内容
                const ctx = canvas.getContext('2d');
                ctx.clearRect(0, 0, canvas.width, canvas.height);

                // 重绘所有已有标注
                this.redrawAnnotations();
            }
        },
        // 开始绘制
        startDrawing(event) {
            if (this.annotationType !== 'rect') return;

            const preview = this.$el.querySelector('.screenshot-preview');
            const rect = preview.getBoundingClientRect();

            this.isDrawing = true;
            this.startX = event.clientX - rect.left;
            this.startY = event.clientY - rect.top;
        },
        // 绘制过程
        draw(event) {
            if (!this.isDrawing || this.annotationType !== 'rect') return;

            const preview = this.$el.querySelector('.screenshot-preview');
            const canvas = this.$refs.annotationCanvas;
            const rect = preview.getBoundingClientRect();

            if (!canvas) return;

            const currentX = event.clientX - rect.left;
            const currentY = event.clientY - rect.top;

            // 清除并重绘
            this.redrawAnnotations();

            // 绘制当前矩形
            const ctx = canvas.getContext('2d');
            ctx.strokeStyle = '#ff0000';
            ctx.lineWidth = 2;
            ctx.strokeRect(
                this.startX,
                this.startY,
                currentX - this.startX,
                currentY - this.startY
            );
        },
        // 停止绘制
        stopDrawing(event) {
            if (!this.isDrawing || this.annotationType !== 'rect') return;

            const preview = this.$el.querySelector('.screenshot-preview');
            const rect = preview.getBoundingClientRect();

            const endX = event.clientX - rect.left;
            const endY = event.clientY - rect.top;

            // 保存矩形数据
            if (Math.abs(endX - this.startX) > 5 && Math.abs(endY - this.startY) > 5) {
                this.annotations.push({
                    type: 'rect',
                    startX: this.startX,
                    startY: this.startY,
                    width: endX - this.startX,
                    height: endY - this.startY,
                    color: '#ff0000',
                    lineWidth: 2
                });

                // 重绘所有标注
                this.redrawAnnotations();
            }

            this.isDrawing = false;
        },
        // 重绘所有标注
        redrawAnnotations() {
            const canvas = this.$refs.annotationCanvas;
            if (!canvas) return;

            const ctx = canvas.getContext('2d');
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // 重绘所有已保存的标注
            this.annotations.forEach(annotation => {
                if (annotation.type === 'rect') {
                    ctx.strokeStyle = annotation.color;
                    ctx.lineWidth = annotation.lineWidth;
                    ctx.strokeRect(
                        annotation.startX,
                        annotation.startY,
                        annotation.width,
                        annotation.height
                    );
                }
            });
        },
        // 添加带标注的截图到上传列表
        addAnnotatedScreenshot() {
            // 创建临时canvas合并原图和标注
            const tempCanvas = document.createElement('canvas');
            const img = new Image();
            img.src = this.screenshotUrl;

            img.onload = () => {
                // 设置canvas尺寸与原图一致
                tempCanvas.width = img.width;
                tempCanvas.height = img.height;

                const ctx = tempCanvas.getContext('2d');

                // 先绘制原图
                ctx.drawImage(img, 0, 0);

                // 计算缩放比例
                const previewImg = this.$el.querySelector('.screenshot-preview img');
                const canvas = this.$refs.annotationCanvas;
                const scaleX = img.width / previewImg.clientWidth;
                const scaleY = img.height / previewImg.clientHeight;

                // 绘制所有标注（需要按比例调整）
                this.annotations.forEach(annotation => {
                    if (annotation.type === 'rect') {
                        ctx.strokeStyle = annotation.color;
                        ctx.lineWidth = annotation.lineWidth * scaleX; // 线宽也需要按比例调整
                        ctx.strokeRect(
                            annotation.startX * scaleX,
                            annotation.startY * scaleY,
                            annotation.width * scaleX,
                            annotation.height * scaleY
                        );
                    }
                });

                // 转换为blob并添加到上传列表
                tempCanvas.toBlob(blob => {
                    const fileName = `annotated_${Date.now()}.png`;
                    const file = new File([blob], fileName, { type: 'image/png' });
                    const url = URL.createObjectURL(file);

                    // 添加到上传列表
                    this.uploadFiles.push({
                        name: fileName,
                        url: url,
                        raw: file
                    });

                    console.log('已添加带标注的截图到上传列表');
                }, 'image/png');
            };
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

.screenshot-preview img {
    width: 100%;
    height: 100%;
    object-fit: fill;
}

/* 标注canvas样式 */
.annotation-canvas {
    position: absolute;
    top: 0;
    left: 0;
    cursor: crosshair;
    z-index: 1;
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
