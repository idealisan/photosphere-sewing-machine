# 多张照片自动拼接为 Photosphere：技术调研

> 状态：调研阶段结论，供后续实现参考
> 调研日期：2026-10-16
> 适用范围：纯前端（浏览器内完成全部计算，无后端、无上传）的 Web 小工具

---

## 0. 结论速览（TL;DR）

| 决策点 | 结论 | 依据 |
| --- | --- | --- |
| 输出格式 | 2:1 equirectangular（ERP）图像，即 Google/Street View/Photo Sphere Viewer 通用的 "photosphere" | 见 §1.1 |
| 核心算法 | 特征匹配 → 单应 RANSAC → 恢复每张照片的**旋转** R → 球面重投影 → 加权融合 | §3 |
| 计算载体 | OpenCV.js（wasm）做配准类 CPU 计算 + WebGL2 shader 做球面 warp 与融合 | §4.1 / §4.3 |
| `cv.Stitcher` | **不可用**。官方 opencv.js 编译时 `-DBUILD_opencv_stitching=OFF`，且 stitching 不在 JS 绑定白名单里 | §4.1 |
| `cv.imread/imdecode/imencode` | **不可用**（`-DBUILD_opencv_imgcodecs=OFF`）。解码走 `createImageBitmap`，编码走 `canvas.toBlob` | §4.1 |
| `cv.SIFT` | **不可用**，不在 features2d 白名单。改用 AKAZE（主）/ ORB（兜底） | §4.1 |
| 自编译 opencv.js | 可行（能拿到 `cv.Stitcher` + 球面 warp），但构建成本高，列为**可选加速路线**而非 MVP 依赖 | §4.2 方案 B |
| MVP 路线 | 方案 A：官方 opencv.js + 自研管线；先用 CPU `cv.remap` 打通，warp/融合再迁到 WebGL2 | §4.2 |
| 最大技术风险 | ①手机 HEIC 解码 ②移动端内存峰值 ③极点覆盖度不足 ④手持拍摄视差导致鬼影 | §6 |

---

## 1. 问题定义

### 1.1 什么是 photosphere

"Photosphere" 在本项目中的确切含义是 **equirectangular（等距圆柱，简称 ERP）投影的全景图**：

- 输出宽高比严格为 **2:1**
- 图像的 x 轴线性映射经度 `λ ∈ [-π, π)`，y 轴线性映射纬度 `φ ∈ [-π/2, π/2]`
- 因此**两极点在图像的左右边缘被拉伸成整行**，这是 ERP 的固有特征，不是 bug
- 这是 Google Street View、Photo Sphere Viewer、Facebook 360、KartaView 的事实标准

> **术语澄清**：OpenCV 文档里把这个投影叫 "spherical projection"（`cv::detail::SphericalWarper`），而图形学/全景圈通常叫 equirectangular / ERP。二者指的是同一组映射公式，见 §3.3 的公式与源码对应。阅读资料时不要被名字差异误导。

### 1.2 输入与输出

**输入**：N 张普通照片（N 典型值 4~20），来源可能是手机主摄 / 广角 / 相机，尽量有 EXIF。

**输出**：

1. 一张 2:1 ERP 图像（JPEG / WebP，建议最高 8192×4096）
2. 交互式预览（可拖拽环视）
3. 可选：多分辨率瓦片（Marzipano 格式）用于嵌入外部看图工具

### 1.3 物理前提：相机在光心纯旋转

拼接算法成立的核心假设是：**所有照片共享同一个投影中心（camera center）**，照片之间的差异只来自绕这个中心的旋转。手持手机拍摄时这个假设不严格成立，会产生**视差（parallax）**，表现为重影/鬼影（ghosting）。

这是本项目质量的**主要分水岭**，也是我们要在 UI 里明确引导用户的拍摄方式（详见 README 的「拍摄指南」）。

### 1.4 极点覆盖：Photosphere 独有的难点

普通「全景图」（panorama）只要覆盖 360°×约 120° 即可，是一个带状区域。而 photosphere 需要覆盖**整个球面 4π 立体角**，包括：

- **天底（nadir）**：设备正下方，通常需要把镜头朝下拍一张
- **天顶（zenith）**：设备正上方，通常需要把镜头朝上拍一张

用户经常忘记拍极点，导致输出图的两端出现大片空洞。有几个处理层次：

1. **不处理**：直接输出空洞，让用户看到问题（最诚实）
2. **警告**：在 UI 上用一张球面覆盖度示意图标出未覆盖区域（推荐）
3. **修补**：对空洞做 `cv.inpaint` 或纹理填充（视觉上很假，但比空洞好看）
4. **裁剪降级**：覆盖不足时自动降级输出为 partial panorama（只覆盖的部分 2:1 之外的比例）——复杂度高，不建议 MVP 做

覆盖度检测的做法（可实现、成本低）：

1. 从配准结果拿到每张照片的朝向 `R_i`
2. 在球面上撒约 2000~5000 个 Fibonacci 球面采样点
3. 对每个采样点 `d`，判断是否落在某张照片的视锥内（结合该照片的实际 FoV）
4. 未被任何照片覆盖的比例即"空洞率"；按纬度分桶统计，就能告诉用户"天顶缺 12%"

---

## 2. 相关工作与经典管线

### 2.1 参考文献

- Lowe, D. *Distinctive Image Features from Scale-Invariant Keypoints*. IJCV 2004.（SIFT）
- Brown, M., Lowe, D. *Automatic Panoramic Image Stitching using Invariant Features*. ICCV 2003.
- Brown, M., Lowe, D. *Automatic Panoramic Image Stitching with Planar-Based Camera Motion Models*. IEEE TPAMI 2007.（焦距估计、bundle adjustment 旋转模型，本项目的主要理论依据）
- Brown, M. *Panoramic Image Stitching with Laplacian Pyramids*. 2007.（多频段融合）
- Brown, M., Fergus, D. *Machine Learning for Automatic Image Geometry*. ICCV 2005.（学习畸变参数）
- Shum, H., Szeliski, R. *Panoramic Image Mosaics*. 2001.
- Gudanov, S. *Improved Source Camera Model for Panoramic Image Stitching*. 2014.（迭代式焦距估计）
- Zhang, Z. 等 *Construction of Panoramic Image Mosaics with Global and Local Alignment*（APAP）2014.（视差容忍）
- Pele, O., Werman, M. *Robust Features for Wide-Angle Fisheye Images*. 2008.（双鱼眼拼接）
- Szeliski, R., Kang, H. *Direct Methods for Visual Scene Reconstruction*. 1995.（视差纹理映射，效果最好但计算量极大，不适合浏览器）

### 2.2 离线/桌面方案的现状（作为质量上界参考）

| 工具 | 技术路线 | 特点 |
| --- | --- | --- |
| **COLMAP** | SfM + BA + 图优化 | 质量最高，能处理视差，但需要 GPU/长时计算，完全不适合浏览器 |
| **Hugin** | 基于 OpenCV stitching，管线与本项目一致 | 交互式调参，有焦距/畸变/接缝手动调整 UI，可作为"理想交互"的参考 |
| **OpenCV `stitching_detailed`** | Brown-Lowe 管线 | C++/Python 可用，是本项目算法蓝本 |
| **Photo Sphere Editor / Google Photo Sphere App** | 手机端球面拼接 | 交互与拍摄引导做得最好，是产品层面的参考 |
| **PTGui / Ricoh Theta** | 商用全景工具 | 输出质量对标 |

---

## 3. 算法管线设计

### 3.1 整体管线

```
[0] 导入 & 解码
      createImageBitmap(imageOrientation:'from-image') → 灰度工作图 + 彩色原图(缩放)
      exifr 读取 Orientation / FocalLength / FocalLengthIn35mmFilm / SensorWidth
      ↓
[1] 特征提取          AKAZE（主）/ ORB（兜底），工作在缩放后的灰度图上
      ↓
[2] 配对 & 匹配        最大重叠生成树 → BFMatcher.knnMatch → Lowe ratio + 交叉验证
      ↓
[3] 单应估计          cv.findHomography(RANSAC) → 内点集 → 重叠率估计
      ↓
[4] 相机模型          焦距：EXIF 优先，否则 Brown-Low / Gudanov 估计
      畸变：径向 k1,k2 可选估计 → cv.initUndistortRectifyMap
      ↓
[5] 旋转恢复          R_ij = K_j^-1 · H_ij · K_i，正交化(SVD) → 每张照片一个 R
      ↓
[6] 全局优化          在 SO(3) 上做 Levenberg–Marquardt，最小化所有内点的重投影误差
      ↓
[7] 覆盖度检查        Fibonacci 球面采样 → 空洞率 / 按纬度统计 → UI 警告
      ↓
[8] 球面 warp         输出 ERP 像素 → 单位球方向 → 转到各照片相机系 → 投影/畸变 → 采样源图
      ↓
[9] 曝光补偿          重叠区估计增益 → 解稀疏最小二乘得到每张图的 block gain
      ↓
[10] 融合             GPU 加权累积（权重 = 到有效区域边界的距离），或 CPU 多频段融合
      ↓
[11] 后处理 & 导出     裁剪地平线、inpaint 补洞、canvas.toBlob → JPEG/WebP
```

### 3.2 各阶段的关键设计决策

#### [2] 配对策略：不要做全连接

N 张照片全连接是 `O(N²)` 次匹配（N=20 → 190 对），每对是一次 BF 暴力匹配，开销很大且收益递减。

**做法**：构造**最大重叠生成树**。

1. 选一张（重叠最多或最居中的）作为锚点
2. 对锚点与所有其他图做匹配，取匹配数 Top-K（K=3）作为邻居
3. BFS 扩展队列，每个新节点只与已入队节点匹配
4. 得到 `N-1` 条边的稀疏图，用于全局优化

匹配细节：

- `BFMatcher(NORM_HAMMING)` 用于 ORB；`NORM_L2` 用于 AKAZE
- Lowe ratio test：`d1 < 0.8 × d2`
- 双向交叉验证：`(i→j)` 与 `(j→i)` 一致
- 手机横竖拍混用很常见。ORB/AKAZE 的方向归一化能吸收小角度旋转，但对 90° 整数倍旋转的鲁棒性依赖实现；**MVP 阶段建议额外做 4 路预旋转匹配**（0/90/180/270）取最优，代价是 4× 匹配时间

#### [3]→[5] 单应 vs 旋转：为什么必须做旋转恢复

直接把单应矩阵链式相乘（`H43 = H43·H32·H21`）是最常见的做法，也是很多 demo 项目用的做法，但它有三个严重问题：

1. **误差累积**：每一环的 1~2% 误差沿链相乘，N=10 时末端误差可达 10%+
2. **尺度漂移**：`H` 中含不可控的尺度分量，链式相乘后尺寸会爆掉
3. **无法做球面投影**：只有拿到纯旋转 `R`（单位正交矩阵，det=+1）才能正确地做球面 warp

**正确做法**（Brown & Lowe 2007）：

```
H_ij ≈ K_j · R_ij · K_i^-1        （纯旋转模型）
⇒ R_ij = K_j^-1 · H_ij · K_i
⇒ 对 R_ij 做 SVD 正交化：R = U·diag(1,1,det(UV^T))·V^T
```

这样每张照片只维护一个 `R`，误差不再沿链累积；再配合 [6] 的全局优化，整体精度会有量级提升。

实现上需要自己做 SVD（3×3 矩阵，可用 Jacobi 特征值法，几十行代码），或用 `cv.eigen` / `cv.svd`（注意：白名单里没有 `cv.svd`，需自己实现或用 `cv.eigen` 走一步）。

#### [4] 焦距估计

优先级：

1. **EXIF**（最快最准）：`FocalLengthIn35mmFilm` + `SensorWidth`（需要按 EXIF 的 focal plane resolution 换算）。手机 EXIF 缺失率不低
2. **单应约束估计**：对于纯旋转，`H` 满足
   ```
   h1'·h2' = h1''·h2''
   h1'^2 + h2'^2 = f²
   ```
   可直接解出 `f`（Brown-Low 闭式解），实现简单
3. **迭代估计**（Gudanov）：多张照片联合优化焦距与旋转，抗噪更好。实现成本中等，作为 v2

#### [4] 镜头畸变

手机主摄在大 FoV 下桶形畸变明显，不校正会在画面边缘产生可见的拉伸/"呼吸"。

- MVP：只支持 EXIF 提供的 `L`（lens distortion）或不做校正，但**必须把 f 估准**，否则畸变会被误当成焦距变化
- v2：估计径向 `k1,k2`（Brown-Fergus 的 MLP 或简单径向模型），用 `cv.initUndistortRectifyMap` 逐张校正，或直接在 shader 里加畸变项（更快，省一次重采样）

#### [6] 全局优化

目标函数：所有生成树边、所有内点匹配的重投影误差平方和。

```
E = Σ_{(i,j,p,q)∈M} ρ( || π_j(R_j · π_i(R_i^-1, p)) - q ||² )
```

参数：每张照片的 3 个旋转参数（用轴角或四元数避免奇异）+ 1 个焦距。

实现：`cv.solve`（白名单里有）解 LM 正规方程，或手写高斯-牛顿 + Huber 权重。规模很小（N≤20，每次迭代的雅可比是稀疏的），JS 里跑完全没问题，**几十毫秒级**。

> 注意：白名单里有 `cv.calibrateCameraExtended`，但它标定的是针孔+径向模型的多相机系统，需要已知 3D 点，不适合这个"无 3D 结构"场景。

#### [8] 球面重投影：这是 photosphere 的核心

见 §3.3 的公式。实现在 CPU 上是"构建 xmap/ymap + `cv.remap`"，在 GPU 上是一段 fragment shader。

#### [9] 曝光补偿

手机自动曝光会让相邻照片亮度不一致，拼接后出现明显色带。

- 简单有效：**块增益（block gain）**。在重叠区把图像切成若干小块，对每对重叠图求每块的最优增益比，再解一个稀疏最小二乘得到每张图的增益。可用 `cv.solve`。
- 更精细：局部增益场 + 高斯平滑（Brown-Lowe 原文做法）
- 备选：`cv.photo.AlignMTB`（**白名单里有**）先做位图对齐，再配合增益

#### [10] 融合策略对比

| 策略 | 优点 | 缺点 | 成本 | 建议 |
| --- | --- | --- | --- | --- |
| 线性羽化（距离变换加权） | 极快，实现简单 | 有视差时明显鬼影；权重突变处有"光晕" | 低 | **MVP 默认** |
| 多频段融合（Laplacian 金字塔） | 过渡最自然，业内标准 | N-1 次金字塔合成，内存随 N 线性增长；8K ERP 下压力大 | 中 | v2，低分辨率做 + GPU 加速 |
| 接缝查找（seam carving / graph cut） | 完全避开重叠区的鬼影，像素"零混合" | 实现复杂（需 Dijkstra / max-flow），接缝处亮度突变 | 高 | v3 或仅在检测到高视差时启用 |
| APAP | 对视差鲁棒 | 需要估计相机位置，平移假设下会引入非物理形变 | 高 | 谨慎，仅研究 |

**推荐策略**：先检测视差强度（见 §6 风险 4），低视差走羽化、高视差走接缝/多频段，并在 UI 上提示用户"检测到明显视差，建议使用三脚架重拍"。

### 3.3 球面投影公式（与 OpenCV 源码逐一对应）

OpenCV 的 `cv::detail::SphericalProjector` 定义在
`modules/stitching/include/opencv2/stitching/detail/warpers_inl.hpp`。
下面是它的公式与我们的用法（`scale` 是"每弧度多少像素"，输出 2:1 ERP 时 `scale = W / (2π)`）。

**正向**（源图像素 → ERP 坐标），`r_kinv = R · K^-1`：

```
x' = r_kinv[0]*x + r_kinv[1]*y + r_kinv[2]
y' = r_kinv[3]*x + r_kinv[4]*y + r_kinv[5]
z' = r_kinv[6]*x + r_kinv[7]*y + r_kinv[8]

u = scale * atan2(x', z')
w = y' / sqrt(x'² + y'² + z'²)
v = scale * (π - acos(w))
```

**反向**（ERP 像素 → 源图像素），`k_rinv = K · R^-1`。这是**我们要用的**方向（逆映射，才能处理空洞）：

```
u /= scale;  v /= scale;

sinv = sin(π - v)
x_ = sinv * sin(u)
y_ = cos(π - v)
z_ = sinv * cos(u)

x = k_rinv[0]*x_ + k_rinv[1]*y_ + k_rinv[2]*z_
y = k_rinv[3]*x_ + k_rinv[4]*y_ + k_rinv[5]*z_
z = k_rinv[6]*x_ + k_rinv[7]*y_ + k_rinv[8]*z_

if z > 0 { x /= z; y /= z; } else { x = y = -1 }   // -1 表示无效，remap 按 borderMode 处理
```

关键点：

- `u` 与 `v` 都是线性的 → **输出天然就是等距圆柱（2:1）**，不需要二次转换
- `z <= 0` 是"源图相机背面"，是**真实的几何空洞**，不能用插值糊过去
- 加上径向畸变（可选）：把 `p = (x, y)`（归一化像平面坐标）替换为
  ```
  r² = x² + y²
  scale_d = 1 + k1·r² + k2·r⁴ + k3·r⁶
  x *= scale_d;  y *= scale_d
  ```

### 3.4 特殊输入模式：双鱼眼 → ERP

这是一个**被低估的高价值特性**。Insta360、Insta360 Link、Google Cardboard（Pixel 的实拍模式）、部分安卓相机的"3D 全景"模式，输出的都是**左右两个鱼眼半图**拼接而成。

对这种输入，**不需要任何特征匹配**：

1. 识别：EXIF/MPO 标记（`MPOG` magic）、左右半图各含 ~180° FoV、或用户手动选择"双鱼眼模式"
2. 已知两个鱼眼的光轴相背 180°、光心重合
3. 直接按 §3.3 的反向公式做解析式重采样，左右两个纹理各覆盖半球

这条路径是**秒级**的，且质量远高于任何特征匹配方案（因为几何完全已知）。建议作为 v1.1 的独立"快速模式"，甚至可以先做它来验证渲染管线。

其它输入模式：

- **单鱼眼原图**（未拼接）：用 `cv.fisheye_initUndistortRectifyMap`（白名单里有）
- **已是 ERP 的图**：直接透传
- **竖排 3 段 / 横排 3 段**（部分相机的全景模式）：已知布局，固定参数重映射

---

## 4. 技术选型调研

### 4.1 OpenCV.js 能力边界（**最重要的一节**）

浏览器里的 OpenCV 是官方用 Emscripten 编译的 `opencv.js`，其能力**远小于**完整的 OpenCV。构建参数见 `platforms/js/build_js.py`，JS 绑定白名单见 `platforms/js/opencv_js.config.py`。

#### 被裁掉的能力（重要！）

| 能力 | 构建参数 | 后果 |
| --- | --- | --- |
| **stitching 模块** | `-DBUILD_opencv_stitching=OFF` | `cv.Stitcher`、`cv::detail::*Warpper`、`GraphCutSeamFinder` **全部不可用** |
| **imgcodecs 模块** | `-DBUILD_opencv_imgcodecs=OFF` | `cv.imread/imdecode/imencode/imwrite` **不可用**，编解码必须交给浏览器 API |
| **ml 模块** | `-DBUILD_opencv_ml=OFF` | 无 SVM/Boosting/EM |
| **videoio / highgui** | `=OFF` | 无关 |
| **SIFT** | 不在 `features2d` 白名单 | 只能用 AKAZE / ORB / KAZE / BRISK / GFTT / MSER |
| `cv.svd` | 不在 `core` 白名单 | 3×3 SVD 需自行实现 |
| `cv.buildPyramid` | 不在 `imgproc` 白名单 | 金字塔需用 `cv.pyrUp/pyrDown` 自己搭 |
| 线程 | 默认 `-s USE_PTHREADS=0` | 单线程，不需要 `SharedArrayBuffer`（对纯静态托管友好） |
| SIMD | 默认 `-s USE_PTHREADS=0` 且 `CV_ENABLE_INTRINSICS=OFF` | 无 SIMD 加速，性能预估要保守 |
| WASM 内嵌 | `-s SINGLE_FILE=1` | wasm 以 base64 内嵌在 js 里，**单文件约 8~11 MB**，无额外网络请求 |

> 官方构建同样没编译 `opencv_contrib`（xfeatures2d 等），除非自行指定 `--extra_modules`。

#### 可用且有用的能力（已核对白名单）

| 类别 | 可用 API |
| --- | --- |
| 特征 | `ORB`、`AKAZE`、`KAZE`、`BRISK`、`GFTTDetector`、`MSER`、`FastFeatureDetector`、`AgastFeatureDetector`、`Feature2D.detectAndCompute`、`drawKeypoints/drawMatches` |
| 匹配 | `BFMatcher.create`、`knnMatch`、`DescriptorMatcher` |
| 几何 | `findHomography`、`UsacParams`、`undistort`、`getDefaultNewCameraMatrix`、`initUndistortRectifyMap`、`fisheye_initUndistortRectifyMap`、`fisheye_projectPoints`、`projectPoints`、`Rodrigues`、`estimateAffine2D` |
| 变换 | `perspectiveTransform`、`getPerspectiveTransform`、`warpPerspective`、`warpAffine`、`remap`、`warpPolar` |
| 图像处理 | `cvtColor`、`resize`、`pyrUp`、`pyrDown`、`GaussianBlur`、`bilateralFilter`、`CLAHE`、`distanceTransform`、`threshold`、`Sobel`/`Laplacian`/`Scharr`、`flip`、`rotate`、`split/merge`、`convertScaleAbs`、`normalize`、`inRange`、`getStructuringElement`、`morphologyEx` |
| 线性代数 | `solve`、`invert`、`eigen`、`gemm`、`transform`、`determinant`、`trace` |
| 曝光 | `photo.AlignMTB`（`computeBitmaps`/`shiftMat`/`process`） |
| 修复 | `photo.inpaint` |
| 其它 | `setLogLevel`（调试很有用） |

#### 打包分发选项

| 方案 | 大小 | 备注 |
| --- | --- | --- |
| `@techstark/opencv-js` | dist ~11.6 MB | npm 官方封装，含 TS 类型，Apache-2.0，周下载 ~22.9 万，当前版本 `5.0.0-release.1`（对应 OpenCV 5.0） |
| `opencv-js-wasm` | 单文件内嵌 | jsDelivr 上有 CDN，便于先做原型 |
| 官方 CDN `docs.opencv.org/<ver>/opencv.js` | ~8 MB | 无 npm 封装，需要自己处理 ES module |
| **自编译（方案 B）** | 15~30 MB | 打开 stitching + imgcodecs + 自定义白名单；需 Emscripten，构建约 30~60 分钟 |

体积是一个真实问题：11 MB 的 wasm 意味着首屏加载慢。缓解手段：

- 用 Vite 的动态 `import()` 延迟加载 opencv.js（先渲染 UI 骨架）
- 或从 CDN 加载并加 `crossorigin` + 长缓存
- 提供"轻量模式"：用 WASM SIMD/threads 自编译版（体积更大但快 2~4×）

### 4.2 算法实现方案对比

#### 方案 A：官方 opencv.js + 自研管线（**推荐，MVP 采用**）

自己实现 OpenCV stitching 管线里的缺失部分：`RANSAC`（有现成的 `findHomography`）、**旋转恢复 + SVD 正交化**、**全局优化（LM）**、**球面 warp**（有现成公式）、**曝光 block gain**、**融合**。

- ✅ 零构建成本，立刻可跑
- ✅ 完全可控，能针对 photosphere 定制（覆盖度检查、极点处理、竖拍引导）
- ✅ 可以用现成的 `findHomography` / `BFMatcher` 等重型零件
- ⚠️ 融合（graph cut / 多频段）要自己写，工作量主要在这
- ⚠️ 算法代码量估计 **2000~3500 行 TS**

#### 方案 B：自编译 opencv.js，开启 stitching 模块

`build_js.py` 加 `--cmake_option=-DBUILD_opencv_stitching=ON`，并把 `stitching.hpp` 加进 JS 绑定白名单，然后 `cv.Stitcher.create(cv.Stitcher.PANORAMA)`。

- ✅ 直接得到久经考验的 Brown-Lowe 全管线（配准 + BA + 接缝 + 融合）
- ✅ 它的 `SphericalWarper` 输出**就是 2:1 equirectangular**（已在 §3.3 核对源码）
- ⚠️ 需要 Emscripten SDK + 完整 opencv 源码，构建 30~60 分钟，CI 上要缓存产物
- ⚠️ stitching 模块未被 emscripten 深度测试过，可能踩坑
- ⚠️ 失去算法层的控制权（无法插入覆盖度检查、极点引导、双鱼眼快速路径）
- ⚠️ 仍需自己做 warp 层的 GPU 加速（OpenCV.js 没有 OpenCL 后端）

**定位**：作为**质量上界**与**回归测试基准**——用它产出的结果作为我们自研管线的"正确性参照"。若 MVP 质量达标就不必引入。

#### 方案 C：纯 TS 自研（不引入 wasm）

自己实现 FAST/ORB 或 AKAZE、RANSAC、DLT、高斯消元……

- ✅ 零依赖，bundle 只有几百 KB，首屏极快，可精细优化（TypedArray、SIMD.js）
- ⚠️ 开发量巨大（估计 6000+ 行），性能与精度风险高
- ⚠️ **强烈不建议作为起点**。但可以**局部借鉴**：融合、覆盖度检测、旋转优化这些 opencv.js 帮不上忙的部分，本来就得自己写

#### 方案 D：服务端方案（colmap / Hugin / Python OpenCV）

违反"纯前端"约束，列出仅作质量参照与**未来可选的降级路径**（比如设备跑不动时可选上传模式）。

#### 决策矩阵

| 维度 | A：opencv.js + 自研 | B：自编译含 stitching | C：纯 TS | D：服务端 |
| --- | --- | --- | --- | --- |
| 开发量 | 中高 | 低（但构建链复杂） | 极高 | 低 |
| 上手速度 | 快 | 慢（需配 emsdk） | 慢 | 快 |
| 运行时体积 | ~11 MB | ~20 MB | <1 MB | — |
| 性能 | 中（无 SIMD/线程） | 中（同左） | 中 | 高 |
| 可控性 / 可定制 | **高** | 低 | 高 | 高 |
| 隐私合规 | ✅ 本地 | ✅ 本地 | ✅ 本地 | ❌ |
| 风险 | 融合算法要自研 | 构建与维护成本 | 性能与精度 | 架构偏离 |

### 4.3 Warp / 融合：WebGL2 vs CPU

#### 为什么建议放到 GPU

8192×4096 的 ERP 有 **3355 万像素**。CPU 端 `cv.remap` 需要为每张照片构建两张 `H×W` 的 `CV_32F` 映射表：

```
3355 万 × 4 字节 × 2 = 268 MB / 张
```

10 张照片如果同时驻留 → 2.7 GB，必爆。即使分块处理，JS↔wasm 的数据搬运本身也很慢。

GPU 上这件事**几乎是免费的**：一张 8192×4096 的 RGBA 纹理 = 128 MB，主流设备（桌面独显、集显、现代手机）都能承受；每张照片一次 draw call 即可。

#### GPU 管线设计

```
Pass 0（CPU，低分辨率 ≈ 1/8）
  对每张照片在 ERP 网格上做可见性测试 → 二值覆盖掩膜
  cv.distanceTransform → 归一化 → 权重纹理
      ↓
Pass 1（GPU，MRT 双附件：累加颜色 + 累加权重）
  顶点：一张全屏三角形
  片元着色器（每张照片一次 draw）：
      #version 300 es
      uniform mat3 uRinvKinv;   // K_j · R_j^-1  的 3x3（见 §3.3 反向公式）
      uniform vec2 uResolution; uniform vec2 uPrincipal;
      uniform float uDistK1, uDistK2;
      in vec2 vUv;
      layout(location=0) out vec4 oColor;
      layout(location=1) out vec4 oWeight;

      const float PI = 3.141592653589793;
      void main() {
        float lon = (vUv.x - 0.5) * 2.0 * PI;
        float lat = (0.5 - vUv.y) * PI;
        // 单位球方向（约定：+Z 前，+Y 上，+X 右）
        vec3 d = vec3(cos(lat)*sin(lon), sin(lat), cos(lat)*cos(lon));
        vec3 c = uRinvKinv * d;                    // 转到该照片相机系
        if (c.z <= 0.0) { oColor = vec4(0); oWeight = vec4(0); return; }
        vec2 p = c.xy / c.z;                       // 归一化像平面坐标
        float r2 = dot(p, p);
        p *= 1.0 + uDistK1*r2 + uDistK2*r2*r2;     // 可选径向畸变
        vec2 st = p + uPrincipal;                  // → 像素坐标
        if (any(lessThan(st, vec2(0))) || any(greaterThan(st, uResolution))) {
          oColor = vec4(0); oWeight = vec4(0); return;
        }
        float w = texture(uWeightTex, st / uResolution).r;
        oColor = vec4(texture(uSrcTex, st / uResolution).rgb * w, 0.0);
        oWeight = vec4(w);
      }
      ↓
Pass 2（GPU，全屏一趟）
  out = color.rgb / max(color.a, 1e-5)
```

工程要点：

- **不用 build map**：逆映射天然避免前向映射的空洞与重复采样
- **权重累加用加法混合**（`gl.blendFunc(ONE, ONE)`），分母单独存 alpha 通道
- **`textureLod`**：ERP 两极存在极端各向异性，mipmap 会跨纬度串色；建议 `textureLod(..., 0.0)` 或按纬度选择 LOD
- **移动端**：`MAX_TEXTURE_SIZE` 通常 ≥ 4096 但未必 8192；必须**运行时探测并自动降级**输出尺寸
- **降级路径**：无 WebGL2 时回退到 CPU 分块 `cv.remap`（慢但能出图）

#### 融合权重

`distanceTransform` 在覆盖掩膜上得到的"到最近无效像素的距离"，归一化后就是标准的羽化权重。要点：

- 在 1/8 分辨率上算足够（权重本来就很平滑），省内存和时间
- 上采样时用双线性
- 可选：对权重做 3×3 高斯，去掉"距离变换的方形等值线"痕迹

### 4.4 内存与性能预算

假设：10 张 4000×3000 手机照片，输出 8192×4096，桌面 Chrome（M 系列 / 近年 x86），单线程 wasm 无 SIMD。

| 阶段 | 数量级 | 估算耗时 |
| --- | --- | --- |
| 解码 10 张 JPEG（主线程 + worker 并行） | 10 × 12 MP | 1.5~4 s |
| 缩放到工作分辨率（1600px 灰度） | 10 × 2.5 MB | 0.2 s |
| AKAZE 特征提取（1600px，每图约 3000 点） | 10 张 | 1.5~4 s |
| 匹配（生成树 ≈ 27 对，BF knn） | 27 × (3000² ≈ 900 万距离) | 1~3 s |
| `findHomography` RANSAC × 27 | — | 0.1 s |
| 旋转恢复 + 全局优化（20 次 LM 迭代） | 27 边 × 数百内点 | < 0.2 s |
| 覆盖度检查（4000 采样点 × 10 图） | — | < 0.05 s |
| 曝光 block gain | — | < 0.2 s |
| **GPU warp + 融合（8192×4096，10 次 draw）** | 134 M 片元 | **0.2~0.8 s** |
| CPU warp（若走回退路径，分 8 块） | 3355 万像素 × 10 | **15~40 s** ⚠️ |
| 编码 JPEG（canvas.toBlob，quality 0.92） | — | 1~3 s |

**总计（GPU 路径）**：约 6~15 s。**这是可接受的产品体验**（给进度条 + 可取消）。

**中端 Android（骁龙 7 系）预估 2~4×**，即 20~50 s。这个体验已经比较勉强，因此：

- 提供**质量档位**（检测分辨率 / 输出尺寸 / 是否做畸变估计）
- 移动端默认 4096×2048 输出 + 1200px 检测分辨率
- 必须做**增量预览**：先出一个 1024×512 的快速结果给用户看"大致成不成功"，再跑高质量

**内存峰值（需重点监控）**：

| 分配 | 大小 |
| --- | --- |
| 单张解码后的彩色位图 4000×3000 RGBA | 48 MB |
| 10 张缩放后的彩色图（2000px） | 10 × 16 MB = 160 MB |
| 10 张灰度工作图（1600px） | 10 × 2.5 MB = 25 MB |
| wasm 堆（opencv.js 内部） | 50~150 MB |
| GPU：输出纹理 + 权重纹理 | 128 + 4 MB |
| **合计** | **~400~500 MB** |

移动端（尤其 iOS Safari 内存上限较低）是**真实风险**，必须：

- 逐张解码、逐张释放 `ImageBitmap.close()`
- 特征提取完成后立刻丢弃彩色大图，只保留缩放版
- 用 `cv.Mat.delete()` 显式释放每个中间 Mat
- 超过阈值时自动降级输出尺寸

### 4.5 浏览器与设备约束

| 约束 | 说明 | 应对 |
| --- | --- | --- |
| **HEIC/HEIF** | iPhone 默认格式。Safari 能解码；**Chrome/Edge 桌面版原生不支持** | 优先请用户上传 JPEG；若必须支持，引入 `heic2any`（wasm）或 `libheif-js`。这是最可能劝退 iPhone 用户的点 |
| **画布尺寸上限** | 与设备/GPU 有关。桌面 Chrome 单边上限 65535、总面积约 2.68 亿像素；iOS Safari 更严（历史上约 1670 万像素） | 运行时探测：先 `gl.getParameter(gl.MAX_TEXTURE_SIZE)`，再实际创建一次目标尺寸纹理做校验，失败则降一档 |
| **`SharedArrayBuffer`** | 多线程 wasm 需要 `COOP: same-origin` + `COEP: require-corp`，GitHub Pages 无法配置 | MVP 用官方单线程 opencv.js，**不需要**。若将来自编译多线程版，要么放弃纯静态托管，要么接受"必须过 COOP/COEP" |
| **WASM 内存上限** | wasm32 寻址 ≤ 4 GB，且 `opencv.js` 默认 `ALLOW_MEMORY_GROWTH` | 控制同时存活的 Mat 数量；大图分块 |
| **OffscreenCanvas** | Worker 里做编码与预览渲染的关键，Chrome/Firefox/Safari 16.4+ 都支持 | 用 `OffscreenCanvas.convertToBlob()` 在 worker 里直接出 JPEG |
| **iOS 内存** | Safari 对单个页面有几百 MB ~ 1 GB 的实际限制，超了就白屏杀掉 | 移动端默认低档位 + 明确提示 |
| **WebGL2** | Safari 15+ / Chrome 全部支持 | 必须做无 WebGL2 的 CPU 回退 |

### 4.6 其它依赖

| 库 | 版本（调研时） | 许可 | 用途 |
| --- | --- | --- | --- |
| `@techstark/opencv-js` | 5.0.0-release.1（对应 OpenCV 5.0） | Apache-2.0 | CV 计算 |
| `exifr` | 7.x | MIT | EXIF（orientation / 焦距 / 传感器尺寸） |
| `@photo-sphere-viewer/core` | 5.15.1（2026-08） | MIT | 交互预览（基于 three.js） |
| `three` | 随 PSV | MIT | PSV 依赖 |
| `vite` + `typescript` | — | MIT | 构建 |
| `heic2any`（可选） | 0.0.4 | MIT | HEIC → JPEG |
| `fflate` / `pako`（可选） | — | MIT | 生成 Marzipano 瓦片时的 gzip |
| `vitest` | — | MIT | 单元测试 |

**许可整体合规**：全部 Apache-2.0 / MIT，允许商用闭源分发。OpenCV 的 Apache-2.0 要求保留版权声明与 NOTICE，我们会在 `THIRD-PARTY-NOTICES` 里列全。

**依赖最小化原则**：`@photo-sphere-viewer/core` 只在进入预览页时动态加载，避免拖累首屏。

---

## 5. 工程架构建议

### 5.1 目录结构

```
photosphere-sewing-machine/
├─ README.md
├─ AGENTS.md
├─ docs/
│  └─ research.md                  ← 本文档
├─ index.html
├─ package.json
├─ vite.config.ts
├─ tsconfig.json
├─ src/
│  ├─ main.ts                      UI 入口
│  ├─ app/                         状态管理 / 视图
│  │  ├─ state.ts
│  │  └─ views/                    upload / progress / preview / settings
│  ├─ workers/
│  │  ├─ cv.worker.ts              主 CV 管线（串行状态机）
│  │  ├─ pool.ts                   worker 池（特征提取并行）
│  │  └─ render.worker.ts          WebGL2 / OffscreenCanvas 渲染与编码
│  ├─ cv/
│  │  ├─ opencv.ts                 opencv.js 加载与 Mat 生命周期
│  │  ├─ features.ts               AKAZE/ORB 提取
│  │  ├─ matching.ts               BF + Lowe + 交叉验证
│  │  ├─ homography.ts             findHomography 封装 + 校验
│  │  ├─ camera.ts                 K / R / SVD / EXIF 解析
│  │  ├─ optimize.ts               SO(3) 上的 LM 全局优化
│  │  ├─ coverage.ts               球面覆盖度检测
│  │  ├─ exposure.ts               block gain 估计
│  │  ├─ blend.ts                  权重生成 + 融合后处理
│  │  └─ linalg.ts                 3×3 SVD、最小二乘（白名单缺口）
│  ├─ gl/
│  │  ├─ renderer.ts               WebGL2 ERP 渲染器
│  │  ├─ shaders.ts                GLSL
│  │  └─ capabilities.ts           能力探测与降级
│  ├─ io/
│  │  ├─ decode.ts                 createImageBitmap / EXIF
│  │  └─ export.ts                 toBlob / 下载 / 瓦片打包
│  └─ types.ts
└─ test/
   ├─ unit/                        数学、投影、匹配的可测单测
   └─ e2e/                         Playwright 跑真实数据集
```

### 5.2 并行策略

- **阶段 1 并行**：`createImageBitmap` + EXIF + 缩放 + 特征提取 → Worker 池（`min(hardwareConcurrency-1, 4)`）
- **阶段 2~7 串行**：有数据依赖，放在单个 worker 里，避免跨线程传递大 Mat
- **阶段 8~11**：渲染 worker（独占 WebGL2 context）+ 主线程只负责 UI

opencv.js 的 wasm 实例**每个 worker 一份**（无法跨 worker 共享 Module），这会让内存 ×N。**结论：worker 池只用于阶段 1 的解码+缩放（纯 JS/浏览器 API，不用 cv），CV 部分单 worker 串行。**

### 5.3 进度与可中断

每个阶段上报 `{ stage, ratio, message }`；wasm 计算无法真正中断，但可以：

- 在阶段边界检查取消标志
- 把耗时最长的"特征提取 + 匹配"拆到 worker 池的独立 JS 实现里，或每张照片后检查一次
- 至少保证 UI 不卡死（这是硬要求）

---

## 6. 风险登记册

| # | 风险 | 概率 | 影响 | 缓解 |
| --- | --- | --- | --- | --- |
| 1 | **HEIC 无法解码** | 高（iPhone 用户） | 高 | MVP 即接入 `heic2any` 或在 UI 明确提示"请上传 JPEG"；测试 iOS Safari |
| 2 | **移动端内存超限白屏** | 中高 | 高 | 逐张释放、显式 `Mat.delete()`、质量档位、渲染前用 `performance.measureUserAgentSpecificMemory()`（Chromium）预警 |
| 3 | **极点覆盖不足** | 高（用户不会拍） | 中 | 覆盖度检测 + 球面示意图警告 + 拍摄引导；提供"自动裁剪为 partial panorama"作为 v2 降级 |
| 4 | **手持视差 → 鬼影** | 高 | 中高 | 视差强度检测（比较 BA 前后的残差/局部单应一致性）+ 明确警告 + 接缝融合兜底；README 强调三脚架 |
| 5 | 8K 输出超出移动端纹理上限 | 中 | 中 | 运行时探测 + 自动降档 |
| 6 | ORB 描述子匹配质量不足导致错配 | 中 | 中 | AKAZE 为主 + ratio test 严格 + 生成树只取 Top-K 边 + 错边可被全局优化剔除 |
| 7 | 多张图时间跨度大（白天/夜晚/移动物体） | 中 | 中 | 重叠区颜色差异检测，超阈值时告警 |
| 8 | 11 MB wasm 首屏慢 | 高 | 低 | 动态 import + CDN + 长缓存 |
| 9 | 融合算法自研质量不如 graph cut | 中 | 中 | 方案 B 作为质量基准；预留"调用 Stitcher"的开关 |
| 10 | 算法自研导致数学 bug 且难发现 | 中 | 高 | **用 Python OpenCV `stitching_detailed` 做黄金数据集回归**：同一批图，Python 结果作为断言基准 |

---

## 7. 里程碑建议

| 里程碑 | 内容 | 验收标准 |
| --- | --- | --- |
| **M0 调研验证（1~2 天）** | 3 个 spike：①opencv.js 能否在 worker 里加载并跑 `findHomography` ②GPU ERP 渲染器能否正确渲染单张已知姿态的图 ③CPU `cv.remap` 路径能否正确输出 2:1 ERP | 三个 spike 各有一个可运行的最小页面 |
| **M1 单图 → ERP（MVP 核心）** | UI 上传 + 解码 + EXIF + 手工指定视角，GPU 渲染出正确 ERP | 与理论投影公式逐像素比对通过 |
| **M2 自动配准** | 特征 + 匹配 + 单应 + 旋转恢复 + 全局优化 | 5~10 张真实照片能正确对齐，误差 < 0.5° |
| **M3 融合与导出** | 曝光补偿 + 距离变换羽化 + 导出 JPEG/WebP | 视觉无缝，无明显色带 |
| **M4 覆盖度 + 交互预览** | 覆盖度检测、球面示意图、Photo Sphere Viewer 预览 | 能准确指出天顶/天底空洞 |
| **M5 移动端适配** | 质量档位、内存优化、HEIC、降级路径 | 中端安卓可用 |
| **M6 双鱼眼快速模式** | 解析式 remap，无需特征匹配 | 秒级出图，质量优于匹配路径 |
| **M7 进阶融合** | 接缝查找 / 多频段、视差 APAP（可选） | 与 Python OpenCV 结果可比 |

**建议先做 M0 的三个 spike**，因为有三个"可能推翻方案"的未知：

1. opencv.js 在 Vite + Worker 下的加载与内存表现（已知有坑：wasm 的 `locateFile`、`SINGLE_FILE`、CSP）
2. WebGL2 在目标设备上能否稳定支撑 8192×4096 MRT
3. opencv.js 无 SIMD 单线程下的 AKAZE 实际耗时（决定质量档位默认值）

---

## 8. 待验证实验清单（Spike）

- [ ] **S1**：Vite 打包 `@techstark/opencv-js`，在 module worker 中加载；测首屏到 `cv.Mat` 可用的耗时与内存
- [ ] **S2**：用合成数据验证球面 warp 公式：给定已知 `R`，生成一张"棋盘格球面图"，ERP 化后再逆变换回来，误差应 < 0.5 px
- [ ] **S3**：WebGL2 在 iPhone 15 / 中端安卓上跑 8192×4096 MRT + 10 次 draw 的耗时与是否掉帧/OOM
- [ ] **S4**：AKAZE vs ORB 在 2000px 工作分辨率下，对 20 组真实手机照片的匹配召回率与耗时
- [ ] **S5**：Python `stitching_detailed` 跑同一批图，产出黄金结果与中间量（`H`、`K`、`R`），作为回归基准
- [ ] **S6**：`heic2any` 在 Chrome 桌面端解 iPhone HEIC 的耗时与内存
- [ ] **S7**：视差强度指标的设计与标定——需要构造"有视差/无视差"两组对照数据

---

## 9. 资料索引

- OpenCV Stitcher 文档：<https://docs.opencv.org/4.13.0/d2/d8d/classcv_1_1Stitcher.html>
- OpenCV stitching 教程：<https://docs.opencv.org/4.5.0/d8/d19/tutorial_stitcher.html>
- opencv.js 构建参数（决定能力边界）：<https://github.com/opencv/opencv/blob/4.x/platforms/js/build_js.py>
- opencv.js 绑定白名单：<https://github.com/opencv/opencv/blob/4.x/platforms/js/opencv_js.config.py>
- 球面投影源码（公式出处）：`modules/stitching/include/opencv2/stitching/detail/warpers_inl.hpp`
- `@techstark/opencv-js`：<https://github.com/TechStark/opencv-js>（Apache-2.0）
- Photo Sphere Viewer：<https://photo-sphere-viewer.js.org>（MIT）
- exifr：<https://github.com/mikekovarik/exifr>（MIT）
- Marzipano（多分辨率全景瓦片格式参考）：<https://www.marzipano.net/>
