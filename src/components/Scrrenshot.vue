<template>
    <div style="display: flex; flex-direction: column; gap:16px; align-items: center;">
        <el-radio-group v-model="annotationType" fill="#0d47a1">
            <el-radio-button label="rect" value="rect">矩形</el-radio-button>
            <el-radio-button label="arrow" value="arrow">箭头</el-radio-button>
            <el-radio-button label="text" value="text">文字</el-radio-button>
        </el-radio-group>

        <el-color-picker v-model="annotationColor" />
        <div style="display: flex; align-items: center; gap: 8px;">
            <span>粗细:</span>
            <el-slider v-model="lineWidth" :min="1" :max="10" :step="1" style="width: 100px;"></el-slider>
            <span>{{lineWidth}}px</span>
        </div>
        <div style="display: flex; align-items: center; gap: 8px;">
            <span>字号:</span>
            <el-slider v-model="fontSize" :min="12" :max="36" :step="2" style="width: 100px;"></el-slider>
            <span>{{fontSize}}px</span>
        </div>

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
        <div class="screenshot-preview" @mousedown="startDrawing" @mousemove="draw" @mouseup="stopDrawing"
            @click="handlePreviewClick">
            <img v-if="isShowSreenshot && screenshotUrl" :src="screenshotUrl" alt="截图" />
            <canvas v-if="isShowSreenshot && screenshotUrl" class="annotation-canvas" ref="annotationCanvas"></canvas>
            <!-- 文字输入框 -->
            <textarea v-if="showTextInput" ref="textInput" v-model="tempText" class="text-input"
                :style="{ left: textInputX + 'px', top: textInputY + 'px' }" @keydown="handleKeyDown"
                @blur="cancelTextInput" placeholder="输入文字，按回车确认，Shift+回车换行"></textarea>
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
            annotationColor: '#ff0000',
            lineWidth: 2,
            fontSize: 16,
            // 矩形标注相关
            isDrawing: false,
            startX: 0,
            startY: 0,
            annotations: [],
            // 文字标注相关
            showTextInput: false,
            tempText: '',
            textInputX: 0,
            textInputY: 0
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
            if (this.annotationType === 'text') return; // 文字标注不需要拖动绘制

            const preview = this.$el.querySelector('.screenshot-preview');
            const rect = preview.getBoundingClientRect();

            this.isDrawing = true;
            this.startX = event.clientX - rect.left;
            this.startY = event.clientY - rect.top;
        },
        // 绘制过程
        draw(event) {
            if (!this.isDrawing) return;

            const preview = this.$el.querySelector('.screenshot-preview');
            const canvas = this.$refs.annotationCanvas;
            const rect = preview.getBoundingClientRect();

            if (!canvas) return;

            const currentX = event.clientX - rect.left;
            const currentY = event.clientY - rect.top;

            // 清除并重绘
            this.redrawAnnotations();

            const ctx = canvas.getContext('2d');
            ctx.strokeStyle = this.annotationColor;
            ctx.lineWidth = this.lineWidth;

            if (this.annotationType === 'rect') {
                // 绘制当前矩形
                ctx.strokeRect(
                    this.startX,
                    this.startY,
                    currentX - this.startX,
                    currentY - this.startY
                );
            } else if (this.annotationType === 'arrow') {
                // 绘制当前箭头
                this.drawArrow(ctx, this.startX, this.startY, currentX, currentY);
            }
        },
        // 停止绘制
        stopDrawing(event) {
            if (!this.isDrawing) return;

            const preview = this.$el.querySelector('.screenshot-preview');
            const rect = preview.getBoundingClientRect();

            const endX = event.clientX - rect.left;
            const endY = event.clientY - rect.top;

            // 保存标注数据
            if (Math.abs(endX - this.startX) > 5 || Math.abs(endY - this.startY) > 5) {
                if (this.annotationType === 'rect') {
                    this.annotations.push({
                        type: 'rect',
                        startX: this.startX,
                        startY: this.startY,
                        width: endX - this.startX,
                        height: endY - this.startY,
                        color: this.annotationColor,
                        lineWidth: this.lineWidth
                    });
                } else if (this.annotationType === 'arrow') {
                    this.annotations.push({
                        type: 'arrow',
                        startX: this.startX,
                        startY: this.startY,
                        endX: endX,
                        endY: endY,
                        color: this.annotationColor,
                        lineWidth: this.lineWidth
                    });
                }

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
                ctx.strokeStyle = annotation.color;
                ctx.lineWidth = annotation.lineWidth;

                if (annotation.type === 'rect') {
                    ctx.strokeRect(
                        annotation.startX,
                        annotation.startY,
                        annotation.width,
                        annotation.height
                    );
                } else if (annotation.type === 'arrow') {
                    this.drawArrow(ctx, annotation.startX, annotation.startY, annotation.endX, annotation.endY);
                } else if (annotation.type === 'text') {
                    this.drawMultilineText(ctx, annotation.text, annotation.x, annotation.y, annotation.color || this.annotationColor, 1, annotation.fontSize || this.fontSize);
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
                    ctx.strokeStyle = annotation.color;
                    ctx.lineWidth = annotation.lineWidth * scaleX; // 线宽也需要按比例调整

                    if (annotation.type === 'rect') {
                        ctx.strokeRect(
                            annotation.startX * scaleX,
                            annotation.startY * scaleY,
                            annotation.width * scaleX,
                            annotation.height * scaleY
                        );
                    } else if (annotation.type === 'arrow') {
                        this.drawArrow(ctx,
                            annotation.startX * scaleX,
                            annotation.startY * scaleY,
                            annotation.endX * scaleX,
                            annotation.endY * scaleY,
                            scaleX // 传入缩放比例，使箭头大小按比例调整
                        );
                    } else if (annotation.type === 'text') {
                        this.drawMultilineText(ctx, annotation.text, annotation.x * scaleX, annotation.y * scaleY, annotation.color || this.annotationColor, scaleX, annotation.fontSize || this.fontSize);
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
        },
        // 绘制多行文本
        drawMultilineText(ctx, text, x, y, color, scale = 1, fontSize = this.fontSize) {
            ctx.font = `${fontSize * scale}px Arial`; // 字体大小根据比例缩放
            ctx.fillStyle = color;

            // 分割文本为多行
            const lines = text.split('\n');
            const lineHeight = (fontSize + 4) * scale; // 行高也按比例缩放，保持为字体大小+4px

            // 逐行绘制文本
            lines.forEach((line, index) => {
                ctx.fillText(line, x, y + (index * lineHeight));
            });
        },
        // 绘制箭头
        drawArrow(ctx, fromX, fromY, toX, toY, scale = 1) {
            const headLength = 10 * scale; // 箭头长度，根据比例缩放
            const angle = Math.atan2(toY - fromY, toX - fromX);

            // 绘制线条
            ctx.beginPath();
            ctx.moveTo(fromX, fromY);
            ctx.lineTo(toX, toY);
            ctx.stroke();

            // 绘制三角形箭头
            ctx.beginPath();
            ctx.moveTo(toX, toY);
            // 计算三角形箭头的两个顶点
            const point1X = toX - headLength * Math.cos(angle - Math.PI / 6);
            const point1Y = toY - headLength * Math.sin(angle - Math.PI / 6);
            const point2X = toX - headLength * Math.cos(angle + Math.PI / 6);
            const point2Y = toY - headLength * Math.sin(angle + Math.PI / 6);
            
            // 绘制三角形箭头
            ctx.moveTo(point1X, point1Y);
            ctx.lineTo(toX, toY);
            ctx.lineTo(point2X, point2Y);
            ctx.closePath(); // 闭合三角形
            ctx.stroke();
            // 可选：填充三角形内部，使箭头更明显
            ctx.fillStyle = ctx.strokeStyle;
            ctx.fill();
        },
        // 处理图片区域点击
        handlePreviewClick(event) {
            // 如果不是文字标注模式或者正在绘制，不处理
            if (this.annotationType !== 'text' || this.isDrawing) return;

            const preview = this.$el.querySelector('.screenshot-preview');
            const rect = preview.getBoundingClientRect();

            // 计算点击位置相对于预览区域的坐标
            const x = event.clientX - rect.left;
            const y = event.clientY - rect.top;

            // 显示文字输入框
            this.textInputX = x;
            this.textInputY = y;
            this.showTextInput = true;
            this.tempText = '';

            // 在下一个DOM更新周期中聚焦输入框
            this.$nextTick(() => {
                if (this.$refs.textInput) {
                    this.$refs.textInput.focus();
                }
            });
        },
        // 处理键盘事件
        handleKeyDown(event) {
            // 如果按下Enter键
            if (event.key === 'Enter') {
                // 如果同时按下了Shift键或Ctrl键，则手动添加换行
                if (event.shiftKey || event.ctrlKey) {
                    // 阻止默认行为，手动添加换行
                    event.preventDefault();
                    // 在光标位置插入换行符
                    const start = this.tempText.substring(0, event.target.selectionStart);
                    const end = this.tempText.substring(event.target.selectionEnd);
                    this.tempText = start + '\n' + end;
                    // 在下一个DOM更新周期中恢复光标位置
                    this.$nextTick(() => {
                        event.target.selectionStart = event.target.selectionEnd = start.length + 1;
                    });
                    return;
                }
                // 如果只是按下Enter键，则确认添加标注
                event.preventDefault(); // 阻止默认行为，避免换行
                this.addTextAnnotation();
            }
        },
        // 添加文字标注
        addTextAnnotation() {
            if (!this.tempText.trim()) {
                this.cancelTextInput();
                return;
            }

            // 保存文字标注
            this.annotations.push({
                type: 'text',
                text: this.tempText,
                x: this.textInputX,
                y: this.textInputY,
                color: this.annotationColor,
                fontSize: this.fontSize // 保存当前的字体大小
            });

            // 重绘所有标注
            this.redrawAnnotations();

            // 重置输入状态
            this.cancelTextInput();
        },
        // 取消文字输入
        cancelTextInput() {
            this.showTextInput = false;
            this.tempText = '';
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

/* 文字输入框样式 */
.text-input {
    position: absolute;
    background-color: rgba(255, 255, 255, 0.8);
    border: 1px solid #ff0000;
    border-radius: 4px;
    padding: 4px 8px;
    font-size: 14px;
    outline: none;
    z-index: 10;
    min-width: 120px;
    min-height: 30px;
    resize: none;
    line-height: 1.4;
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
