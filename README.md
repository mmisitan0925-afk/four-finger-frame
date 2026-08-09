# 四指拉框 — 开发对话记录

## 最终产物
`index.html` — 单文件，可直接浏览器打开（需摄像头权限 + HTTPS 或 localhost）

## 功能清单
- MediaPipe Hands 实时手势识别（拇指[4] + 食指[8]）
- 四指构成矩形，反转色填充（Canvas clip + difference）
- 手指交叉时矩形平滑变形为 8 字形
- 内嵌音乐（Escape Your Mind!），♫ 按钮播放/暂停
- 摄像头分辨率选择（320×240 / 640×480 / 1280×720 / 1920×1080）
- 镜像翻转开关
- 丢指渐隐、FPS 显示

## 对话迭代记录

### 第1轮：手指拉丝特效（手动拖拽）
- 实现了纯 Canvas 2D 的弹簧阻尼拖尾效果
- 支持触摸和鼠标，多点触控
- HSL 色相循环 + shadowBlur 发光

### 第2轮：接入摄像头手势识别
- 引入 MediaPipe Hands（CDN @mediapipe/tasks-vision）
- 双层 Canvas（视频层 + 特效层）
- 坐标映射：归一化 → cover 适配 → 镜像翻转
- 帧采样：每 2 帧推理一次，渲染 60fps 独立循环

### 第3轮：四指拉矩形 + 反转色 + 8字变形
- 检测食指[8]+中指[12]，四点构成矩形
- 反转色填充：Canvas compositing（destination-atop + difference）
- 8字变形：signedArea 负值 → Catmull-Rom 样条路径
- 角点分配：右手→角0/1，左手→角2/3

### 第4轮：用户修正需求
- **只取食指和大拇指**：LM = [8, 4]
- **反转色只填充矩形内部**：改用 clip path 裁切
- **增加镜像开关**：CSS transform:scaleX(-1) + 坐标映射联动

### 第5轮：追踪精度修复
- 去掉弹簧物理，改用 lerp(0.75) 直接插值
- 8字触发改为对角线实际相交检测（segmentsCross 参数 t,u ∈ (0.01, 0.99)）

### 第6轮：矩形角序修正
- 用户要求：左食指 → 右食指 → 右拇指 → 左拇指
- 修正角位分配：左手 base=0, 右手 base=1

### 第7轮：镜像坐标修复
- 发现双重翻转 bug：mapPt 翻转 + CSS 翻转 = 抵消
- 修复：镜像ON 时不翻转坐标（CSS 处理），镜像OFF 时手动翻转

### 第8轮：图片填充功能（后删除）
- 添加图片上传 + 三角形网格透视变形
- 遇到 setTransform 与 outer clip 冲突问题
- 尝试临时画布方案，仍有问题
- 用户要求删除图片功能

### 第9轮：最终版本
- 删除图片功能
- 内嵌音乐（base64 data URI）
- 添加摄像头分辨率选择器

## 技术要点

### 坐标映射
```
MediaPipe 归一化(0~1) → 视频像素 → cover 缩放+偏移 → Canvas CSS 像素
镜像ON: CSS scaleX(-1) 处理视觉翻转,坐标不翻
镜像OFF: 手动翻转 x = 1 - lm.x
```

### 反转色实现
```
ctx.save()
ctx.clip(形状路径)        ← 裁切到矩形/8字
ctx.drawImage(视频)       ← 在 clip 内画视频
ctx.difference + 白色     ← 仅 clip 内反转
ctx.restore()             ← clip 解除,其余透明
```

### 8字变形检测
```
signedArea < 0 且 segmentsCross(对角线参数 t,u ∈ (0.01, 0.99))
→ smoothMorph 平滑插值到目标变形量
→ morph > 0.5 时路径从圆角矩形切换到自交叉四边形
```

### 音乐内嵌
```html
<audio id="bgm" loop preload="auto" src="data:audio/mpeg;base64,..."></audio>
```

## 依赖
- MediaPipe Hands（CDN，WASM 运行时 + 模型自动下载）
- 无其他外部依赖
