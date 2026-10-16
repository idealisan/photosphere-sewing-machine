# Photosphere Sewing Machine · 全景缝合机

**把一堆普通照片，在浏览器里缝成一张 360° 全景球（photosphere）。**
不上传、不联网、不装 App —— 所有计算都在你自己的浏览器里完成。

> 纯前端 Web 小工具 · 无后端 · 无上传 · 图片永远不离开你的设备

**项目状态：技术调研阶段，代码尚未开始实现。** 本文档描述的是设计目标，最终形态可能随实现调整。

---

## 这是什么

选入 4~20 张照片，工具会自动完成：特征匹配 → 相位对齐 → 球面重投影 → 曝光补偿 → 无缝融合，
输出 **2:1 等距圆柱（equirectangular）全景图**，即 Google Street View / Photo Sphere Viewer 通用的 photosphere 格式。

**技术调研文档见 [`docs/research.md`](./docs/research.md)** —— 里面记录了算法管线选型、OpenCV.js 的能力边界（有哪些 API 在浏览器里**不可用**）、GPU 渲染方案、内存预算与风险清单。动手前建议先读。

---

## 特性

| | 能力 |
| --- | --- |
| 🎯 | 自动配准 —— 不用手动摆顺序、定锚点、调焦距 |
| 🔒 | 完全本地 —— `<input type="file">` 读图，照片不发出浏览器 |
| 🧭 | 覆盖度检查 —— 自动算出天顶/天底有没有拍漏，并在球面示意图上标出来 |
| 🎚 | 可控质量 —— 检测分辨率、输出尺寸、畸变校正、融合强度均可调 |
| 🔭 | 交互预览 —— 拖拽环视，所见即所得 |
| 📤 | 导出 —— JPEG / WebP，最高 8192×4096 |
| 📱 | 移动端可用 —— 带自动降档，照顾手机内存 |
| ⚡ | 双鱼眼快速模式（规划中）—— 已知几何，无需匹配，秒级出图 |

---

## 拍摄指南（重要）

拼接质量 90% 取决于拍摄方式。工具能救算法救不了的输入。

### 必须遵守

1. **固定设备，只旋转不移动**
   三脚架、云台、靠在墙/栏杆上、把手机压在饮料杯里都可以。
   唯一不要做的就是"边走边拍"—— 这会产生**视差**，导致重影（鬼影）。

2. **相邻照片重叠 30%~50%**
   大致就是"转过 1/3 个视野再拍下一张"。
   - 1x 主摄（约 75° 对角视野）：绕一圈至少 **6~8 张**
   - 0.5x 超广角（约 110°+）：**4~6 张** 就够

3. **锁定曝光与对焦**
   iPhone 长按取景框锁定 AE/AF，关闭 HDR 自动、夜景模式、闪光灯自动。

4. **同一时间段拍完**
   太阳位置变化、色温漂移、路人走过都会造成难以修复的接缝。

5. **不要混用不同镜头/变焦档位**
   1x 和 2x 之间切换会导致焦距不一致，工具只能尽力补偿。

### 拍到天顶和天底

Photosphere 覆盖的是**整个球面**，包括正上方和正下方。

- 最后**补拍两张**：一张镜头垂直朝上，一张垂直朝下（注意手别入镜）。
- 没拍也没关系 —— 工具会检测并告诉你缺了多少，然后在结果两端留下空洞。

### 反例速查

| ❌ 别这样 | ✅ 这样 |
| --- | --- |
| 手持转一圈 | 三脚架固定 |
| 转 90° 才拍一张（重叠太少） | 转 30° 拍一张 |
| 自动曝光一路跳 | 锁定 AE/AF |
| 中途有人走过 | 清场后再拍 |
| 只拍了一圈水平带 | 补拍天顶 + 天底 |

---

## 技术栈

| 层 | 选型 |
| --- | --- |
| 构建 | Vite + TypeScript |
| CV 计算 | OpenCV.js（wasm，Apache-2.0） |
| Warp / 融合 | WebGL2 自研 ERP 渲染器（`gl/`），无 WebGL2 时回退 CPU `cv.remap` |
| 并发 | Module Worker（解码/缩放并行 + CV 串行 + 渲染独立） |
| 预览 | `@photo-sphere-viewer/core`（MIT，按需动态加载） |
| EXIF | `exifr`（MIT） |

> **注意**：官方 opencv.js 编译时关闭了 `stitching` 和 `imgcodecs` 模块，
> 因此 **`cv.Stitcher`、`cv.imread/imdecode/imencode`、`cv.SIFT` 在浏览器里都不可用**。
> 本项目的拼接管线是围绕这些缺口自研的。详见调研文档 §4.1。

---

## 开发

```bash
npm install
npm run dev      # 开发服务器
npm run build    # 产出静态文件到 dist/，可直接丢到任意静态托管
npm run preview  # 预览构建产物
npm run test     # 单元测试
```

构建产物是**纯静态文件**（无后端、无 API、可离线运行）。

### 目录结构

```
src/
├─ main.ts          UI 入口
├─ app/             状态与视图（上传 / 进度 / 预览 / 设置）
├─ cv/              算法：特征、匹配、单应、相机模型、全局优化、覆盖度、曝光、融合
├─ gl/              WebGL2 ERP 渲染器与 GLSL
├─ io/              解码、EXIF、导出
└─ workers/         Worker 池与渲染 Worker
```

---

## 路线图

- [x] 技术调研（[`docs/research.md`](./docs/research.md)）
- [ ] M0 三个 spike：opencv.js in Worker / GPU ERP 渲染 / CPU remap 回退
- [ ] M1 单图 → 正确 ERP 输出
- [ ] M2 自动配准（特征 → 匹配 → 旋转恢复 → 全局优化）
- [ ] M3 曝光补偿 + 无缝融合 + 导出
- [ ] M4 覆盖度检测 + 交互预览
- [ ] M5 移动端适配（HEIC 解码、内存、降档）
- [ ] M6 双鱼眼快速模式
- [ ] M7 进阶融合（接缝查找 / 视差 APAP）

---

## 输出规格

| 项 | 值 |
| --- | --- |
| 格式 | 2:1 等距圆柱（equirectangular） |
| 尺寸 | 最高 8192×4096（可降档；移动端默认 4096×2048） |
| 编码 | JPEG / WebP（浏览器 `canvas.toBlob`） |
| 兼容 | Google Street View、Photo Sphere Viewer、Marzipano、Facebook 360 |

---

## 隐私

- 图片通过 File API 读取，**全程不发送任何网络请求**
- 无遥测、无统计、无账号
- 关掉页面即失去一切中间数据
- 依赖的第三方库全部为 Apache-2.0 / MIT 许可

---

## 许可

代码许可待定。依赖许可：

- OpenCV / opencv.js — Apache-2.0
- Photo Sphere Viewer、exifr、three.js — MIT

---

## 贡献

欢迎提 issue 和 PR，尤其是**测试数据集**（带视差 / 不带视差 / 逆光 / 夜景 / 极小重叠的实拍照片组）—— 这是本项目最需要的东西。

---

## 相关链接

- [技术调研文档](./docs/research.md)
- [OpenCV Stitcher 文档](https://docs.opencv.org/4.13.0/d2/d8d/classcv_1_1Stitcher.html)
- [Photo Sphere Viewer](https://photo-sphere-viewer.js.org)
