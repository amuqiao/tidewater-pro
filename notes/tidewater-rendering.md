# Tidewater 渲染与模拟：像素是怎么算出来的

一句话：这一篇把 [总览](tidewater-overview.md) 里"渲染"那一步展开——先建立 GPU 编程的心智模型，再看场景是怎么一步步画到屏幕上的，最后看海面和整座岛是怎么被算出来的。

## 适用读者

已经读过 [tidewater-overview.md](tidewater-overview.md)，或者至少接受其中第 1、4 节的心智模型。仍然假设你没有图形编程经验。

## 这篇文档负责什么

- 用直觉解释 GPU 编程的四个核心概念（命令录制、着色器、渲染目标、资源绑定）。
- 讲清 `SceneRenderer` 的四步渲染顺序，以及每一步为什么存在。
- 讲清海面（FFT 海洋）为什么必须放在 GPU 上算，以及它和其他系统怎么协作。
- 说明世界与资产是怎么"造"出来的（程序化生成 vs 离线流水线）。
- 指出调试与性能观察的入口。

## 不负责什么

- 不推导数学（Tessendorf 频谱、Hillaire 大气模型等），只说明它们在项目里承担什么角色。
- 不逐个讲后期效果的实现细节。
- 不替代 `docs/PORTING.md`：那份文档是"three.js 概念 ↔ 本项目概念"的逐条对照表，要改渲染代码时请直接看它。

---

## 1. 心智模型：流水线、着色器、渲染目标

### 1.1 一次绘制调用背后是什么

对 CPU 来说，画一个物体只是"提交一条命令"。对 GPU 来说，这条命令意味着一整条流水线：

```text
       一条 draw 命令
            │
            ▼
   ① 顶点着色器：对物体的每个顶点跑一次
        输入：顶点位置、法线、UV、材质参数
        输出：这个顶点在屏幕上该出现在哪、它的世界坐标是多少
            │
            ▼
   ② 光栅化：把三角形"涂"成像素（硬件固定功能，不由你写代码）
        决定哪些像素被三角形覆盖，并插值顶点数据
            │
            ▼
   ③ 片元着色器（也叫像素着色器）：对每个被覆盖的像素跑一次
        输入：插值后的数据 + 纹理 + 光照参数
        输出：这个像素的颜色（可以带多个输出：颜色、速度、遮罩…）
            │
            ▼
   ④ 混合与写入：把结果写进目标图（可能和已有内容做 alpha 混合）
```

最关键的一条直觉：**着色器不是"画一个东西"的程序，而是"对每个元素跑一次"的函数**。
顶点着色器是 `map(顶点 -> 新顶点)`，片元着色器是 `map(像素 -> 颜色)`。
这和 Python 里 `[f(x) for x in items]` 或 numpy 的向量化思维接近，只是这里的"每个元素"可能以百万计，而且必须真的能并行。

所以着色器里**没有循环去遍历其他物体**。想知道其他物体在哪，只能提前把数据准备好（传进去一张纹理或一个缓冲区）。
这就是为什么这个项目里有那么多"预计算"和"渲染到一张图，再拿这张图当输入"的做法。

### 1.2 三个必须先接受的概念

| 概念 | 直觉解释 | 项目里的例子 |
|---|---|---|
| 着色器（shader） | 用 WGSL 语言写的小函数，GPU 批量执行 | `src/engine/render/wgsl/common.js` 里是共享的噪声/法线工具函数 |
| 渲染目标（render target） | "画到一张图里"而不是画到屏幕；这张图随后可以当纹理被读取 | `SceneRenderer` 先把场景画进 `sceneRT`，再交给后处理链 |
| 绑定（binding / bind group） | 着色器要用的资源（纹理、缓冲区、采样器）必须提前按约定槽位"绑"上去，运行时不能临时找 | 约定：group 0 = 每帧的相机与全局量，group 1 = 材质资源，group 2 = 每次绘制 |

一句话：**GPU 代码不能自由访问内存，所有输入都必须被显式安排好。**

### 1.3 命令录制模型

再强调总览里那条：CPU 侧只是录制。

```text
GPU.beginFrame()                开始录
  ... 各系统调用 update/render，实际上都在往同一个 encoder 里追加命令 ...
GPU.submit()                    一次性提交，GPU 异步执行
```

代码里有一条很容易踩的规则，`src/engine/gpu/GPU.js` 顶部专门注释了它：

```text
queue.writeBuffer() 的数据在整帧命令执行前就落地，
所以同一段缓冲区在一帧内写两次，所有 pass 看到的都是最后一次的值。
=> 一帧内会变化的数据（例如不同视角的相机矩阵）必须各自占一个缓冲区。
```

`src/engine/render/Frame.js` 里的 `createViewUniforms()` 就是这条规则的产物：主相机一份 `frame` 块，阴影级联、反射相机等每个额外视角各一份，只覆盖相机相关字段，其余字段沿用主块。

---

## 2. 资源类型速查

读渲染代码时遇到的名字，都可以归到下面几类。左边是 GPU 概念，中间是 Python 类比，右边是项目中的类。

| GPU 概念 | Python 类比 | 项目中的类 / 位置 |
|---|---|---|
| 设备与队列 | 连接池 + 任务队列 | `GPU` 单例（`device`、`queue`、`encoder`、`samplers`） |
| 纹理 / 贴图 | 定形的 numpy 数组（可能带 mipmap 金字塔） | `Texture`（`src/engine/gpu/Texture.js`） |
| 渲染目标 | "一张可被继续读取的图"，含若干颜色附件 + 深度附件 | `RenderTarget` |
| 存储缓冲区 | 一块结构化的显存数组（可读写） | `StorageBuffer` |
| Uniform 块 | 每帧/每次绘制传给着色器的常量结构体 | `UniformBlock`（如 `FrameUniforms`） |
| 着色器模块与计算内核 | 一个具名函数库 / 一次批量计算任务 | `ShaderModule`、`ComputeKernel` |
| 全屏通道 | 对整个画面做一次处理（滤镜） | `FullscreenPass` |
| 回读 | 把 GPU 里的结果拷回 CPU 内存（慢，要等） | `Readback`、`readTexture` / `readBuffer` |

关于回读，项目里有两个重要用法：

- **水面高度查询**：波浪在 GPU 上，CPU 想知道某点水面多高就必须回读一次（`src/ocean/WaterQuery.js`）。代码会把多个查询攒起来一次问完——因为这是同步等待，代价高。
- **大气回读**：天空的散射结果被回读成太阳颜色与天空辐照度，供 CPU 侧逻辑使用（`App.applyAtmosphereReadback()`）。

### 2.1 全局帧用量 `G` 与着色器里的 `frame`

有一个名字你会反复看到：`G`。

```text
G.sunDir.value             CPU 侧读写
frame.sunDir               对应的 WGSL 结构体字段（每个着色器都能看到）
```

它们是**同一块数据的两个叫法**：`G` 是 `FrameUniforms` 各字段的句柄集合（`src/engine/render/Frame.js`），
`frame` 是它在 WGSL 里的名字。所以 `G.exposure.value = 0.55` 就是在给所有着色器设置曝光。
`src/core/Globals.js` 只是把 `G` 再导出一次，没有别的逻辑。

`frame` 里既有相机信息（各种矩阵、分辨率、TAA 抖动），也有仿真全局量（时间、风速风向、太阳方向与颜色、水下标志、水的吸收散射系数）。
把"每帧所有着色器都要知道的量"集中在一个结构体里，是这个引擎的核心简化手段。

---

## 3. 一帧的渲染顺序：`SceneRenderer` 的四步

`src/engine/render/SceneRenderer.js` 的注释直接给出了顺序，这也是本项目渲染部分最该先读懂的地方：

```text
第 1 步  不透明层 -> sceneRT
         （HDR 颜色 + 速度 + 水线遮罩，reversed-Z 深度），
         然后把天空画到"没被任何东西覆盖"的像素上
              │
第 2 步  复制：颜色和深度 -> opaqueCopy（给水面做折射/吸收用）
         再把深度复制成半精度的半分辨率副本 opaqueDepthHalf（给需要大量散布采样的效果用）
              │
第 3 步  船体遮罩（hull mask）：把"海面不该被绘制"的封闭体积画出来
              │
第 4 步  水面 + 透明层 -> 叠加到 sceneRT 上
```

`sceneRT` 有三张颜色附件和一个深度附件，各有明确用途：

| 附件 | 格式 | 用途 |
|---|---|---|
| 颜色 | `rgba16float` | 最终的 HDR 颜色（后处理链的输入） |
| 速度 | `rgba16float` | 屏幕空间的运动矢量（xy = 当前帧 uv 减上一帧 uv），给时间抗锯齿和运动模糊 |
| 水线遮罩 | `rgba8unorm` | r = 当前可见表面是从水下看到的；g = 当前可见表面就是水面 |
| 深度 | `depth32float` | **reversed-Z**（近处为 1，远处为 0，天空为 0），浮点深度配合这种约定能在远距离保持精度 |

### 3.1 为什么第 2 步要复制一遍场景

水是透的：你能看见水下的沙地、礁石和船底。但水面的着色器在画水面，它怎么知道"底下有什么"？

做法是"把水面之前画好的东西复制一份，当作纹理给水面着色器采样"：

```text
opaqueCopy（水面之前的不透明画面）
        │
        ▼
水面着色器：按折射方向采样 opaqueCopy，把水下的画面"歪一点"显示出来
           再按水的吸收系数（frame.waterAbsorption）给深的地方染色
```

这就是 `src/ocean/RefractionPass.js` 的职责，它由 `sceneRenderer.onBeforeWater` 钩子触发，只渲染水面以下的场景并保持在半分辨率（因为折射本来就模糊，全分辨率浪费）。

一句话总结这条设计：**在实时渲染里，"看到别的物体"通常意味着"先把它们渲染到一张纹理里"。**

### 3.2 为什么需要船体遮罩

如果从船里往外看，海面是一个一直延伸到天边的巨大网格，它会穿透船体出现在船舱内部。
引擎的解决办法不是给船加复杂的裁剪，而是：

```text
把船体内部当成一个"封闭体积"画进一张遮罩纹理，
水面着色器在遮罩覆盖的像素上直接丢弃（不画）。
```

这只在相机位于船内时才启用（`SceneRenderer._renderHullMasks()` 用包围盒和视锥判断），
并且预编译阶段会强制走一遍"有遮罩"和"无遮罩"两种变体，避免首次上船时卡顿。

### 3.3 分层（layers）是控制"哪些东西在哪一步画"的开关

```text
LAYERS.OPAQUE         普通不透明物体，第 1 步画
LAYERS.WATER          海面，第 4 步画
LAYERS.TRANSPARENT    玻璃、渔网、布等，第 4 步画（在水之后，或与水混合）
LAYERS.REFLECT_ONLY   只在反射里出现的物体
```

`MeshRenderer.render()` 接收 `layerMask`，所以同一份场景可以按需要只画某一层。
`App.js` 里有一段专门把船上的渔网、玻璃等挪到 `TRANSPARENT` 层，注释解释了原因：
屏幕空间环境光遮蔽和折射副本只读不透明层，这些薄物体如果放在第 1 步会产生难看的黑边。

### 3.4 `MeshRenderer` 在做什么

它是"把场景里的一堆网格变成一串 draw 命令"的执行者：遍历、视锥剔除、按材质/管线排序、上传几何、维护管线缓存、写入每次绘制的参数、录制命令。
它的头部注释给出了全部可选项（`kind`、`late`、`layerMask`、`filter`、`after`…），也是排查"某个物体没画出来"时的第一站：

```text
常见原因：不在相机视锥内 / 层不对 / 材质透明但排在错误的一步 / 被 hull mask 丢弃
```

---

## 4. 后处理链

场景画完之后并不直接上屏，而是经过一串"整屏处理"。`src/post/PostFX.js` 头部写明了顺序：

```text
场景（HDR 颜色 + 速度）
  → GTAO 环境光遮蔽（半分辨率、按 5×5 图案旋转采样 + 空间去噪）
  → AO 合成 + 水下/水线合成（一次 pass 完成）
  → TAAU 时间抗锯齿 + 时间上采样（放大到输出分辨率）
  → bloom（从半分辨率开始，13 次采样的降采样 + 帐篷滤波升采样链）
  → 调色（饱和度/对比/色温）+ 暗角 + 颗粒
  → ACES 色调映射 + sRGB 转换
  → 画布（或测试用的离屏纹理）
```

对 Python 开发者的类比：这就像一条数据处理流水线，每一级拿上一级的输出，最后输出到"文件"（这里是画布）。

三个值得注意的设计点：

- **内部渲染分辨率可以低于输出分辨率**：`renderScale` 设置在 0.5–1 之间，中间画面用小尺寸渲染，最后由 TAAU 放大并重建细节。这是"性能不够时降画质"的主要旋钮。
- **TAAU 需要抖动与历史**：每帧给相机投影加一个亚像素偏移（`frame.jitter`），再与历史帧对齐累积。所以代码里到处都有"上一帧"的量（`prevViewProjNoJitter`、`prevCameraPos`、`prevJitter`）。这也是为什么速度缓冲（velocity）是必需附件——没有它就无法对齐历史。动态物体需要正确的运动矢量，而地形和水面是**在顶点着色器里生成顶点**的，默认运动矢量会是垃圾值，所以 `App.js` 显式把它们标记为"静态速度"（`useStaticVelocity`），让重建按相机运动来处理。
- **自动曝光**：曝光值由 GPU 统计出来（基于降采样后的亮度），CPU 侧只提供 `frame.exposure` 作为可调系数。用 Python 打比方：像"根据历史数据自动调整的缩放系数"，而不是每帧在 CPU 上循环像素。

---

## 5. 海面：为什么必须用 FFT，以及它怎么工作

这是本项目的招牌，也是最值得理解的一段。

### 5.1 直觉：海面不是"建模"出来的，是"算"出来的

传统做法是用美术制作的波浪网格。但这里要的是一望无际、随海况与风变化的真实海浪。
做法是物理模拟的经典套路（Tessendorf 频谱法）：

```text
把海面看成很多个不同波长、不同方向的正弦波叠加
     ↓ 用经验海浪谱（Horvath / JONSWAP）给每个频率定"能量"
     ↓ 每帧让相位随时间演化（波在传播）
     ↓ 用 FFT 从"频率空间"一次算出"空间空间的波形"
得到一张高度场（位移）纹理
     ↓ 水面网格的顶点着色器采样这张纹理，把顶点抬高、左右推（choppy）
```

关键判断是：**这件事必须放在 GPU 上做。**

```text
如果用 CPU（Python/numpy 的类比）：
   4 层级联 × 256×256 复数网格 × 每帧 2 次 IFFT
   还要每帧上传到显存  -> 带宽和时间都不可接受
用 GPU：
   每帧两次 compute dispatch（行变换 + 列变换），结果直接留在显存里当纹理用
```

### 5.2 两次 dispatch 与"级联"

`src/ocean/OceanFFT.js` 顶部注释写得很完整，概括为：

```text
每帧两次 compute dispatch，处理所有级联：
  ① 行 pass：频谱随时间演化 -> 4 个打包复场 -> 每行做 256 点 radix-2 IFFT（用 workgroup 共享内存）
  ② 列 pass：每列 IFFT -> 符号修正 -> 基于雅可比的泡沫累积 -> 写出位移/导数数组纹理
```

"级联"（cascade）是解决**尺度矛盾**的手段：一个网格不可能同时表现 700 米的大涌和 7 米的小碎波。

```text
cascade 尺寸（米）：733  |  157  |  33.3  |  7.1
各自负责：      大涌      中浪      小浪      细波
```

尺寸之间刻意取非整数比（733 / 157 ≈ 4.67 而不是 4），避免四层重叠时出现肉眼可见的重复图案——代码注释里专门说了这点。
级联内还带 LOD，远处用更粗的级别，因此海面的几何是 CDLOD（连续细节层次）网格，顶点在顶点着色器里被放置。

### 5.3 输出给谁用：着色器"函数库"的约定

FFT 的输出是显存里的数组纹理（每层一个级联）：

```text
oceanDisplacement：Dx, Dy, Dz, foam      位移 + 泡沫
oceanDerivatives： dDy/dx, dDy/dz, ...   法线/雅可比所需导数
```

别的系统要读它，靠的是两条约定（`docs/PORTING.md` 有全表）：

```text
① 每个系统暴露 this.module  —— 一段带名字的 WGSL 代码
② 函数名统一带前缀           —— OceanFFT 的函数叫 oceanXxx，地形叫 terrainXxx，浅水模拟叫 shoreSimXxx

于是水面材质可以直接写：
    let h = terrainHeightAt( xz );            // 地形模块提供
    let d = oceanSampleDisplacement( xz );     // 海洋模块提供
```

这相当于**在 GPU 上做模块化编程**：着色器在编译期被拼起来，调用别人暴露的函数。
读渲染代码时，遇到 `xxxFoo()` 形式的 WGSL 函数，就用前缀回到对应文件去找实现——这是本项目最实用的定位技巧。

### 5.4 海面的"周边系统"

海面本身只是高度场，让它看起来像海的是周围一圈系统：

| 系统 | 文件 | 作用 |
|---|---|---|
| 水面网格与材质 | `src/ocean/WaterSurface.js`、`WaterMaterial.js` | 顶点位移、着色、泡沫、折射、水下/水上切换 |
| 潮线浅水模拟 | `src/ocean/ShoreSim.js` | 单独模拟水在沙滩上来回（swash），并把"湿度/沙上泡沫"以函数形式提供给地形材质 |
| 岸浪 | `src/ocean/ShoreWaves.js` | 靠岸时的相位与形态，破浪的时机由它决定 |
| 破浪与喷雾 | `src/ocean/Breakers.js`、`src/fx/Spray.js`、`SurfFoam.js` | 卷破的浪头、水花粒子、白沫质感 |
| 船迹与尾流 | `src/ocean/WakeSim.js`、`src/player/BoatSpray.js` | Kelvin 尾迹、船首浪、螺旋桨泡沫，并反过来影响水面材质 |
| 水下光照与体积感 | `src/ocean/Caustics.js`、`UnderwaterLighting.js`、`src/post/Underwater.js`、`src/fx/MarineSnow.js` | 水底焦散、光轴、水下雾、海水里的悬浮颗粒 |
| 水面高度查询 | `src/ocean/WaterQuery.js` | 给 CPU 侧（船浮力、玩家游泳、声音、喷雾朝向）用 |

这张表也解释了 `App.init()` 里那条依赖链：**FFT 必须先建好**，因为它下面的所有东西都要读它。

顺带一个有意思的设计：涟漪和波浪不只是画面，也是玩法判据。
`WaterQuery` 提供的水深和 `Bites.js` 的栖息地判定共享同一套世界数据——
你钓到哪种鱼，最终取决于同一份被 GPU 算出来的海面与地形数据。

---

## 6. 世界是怎么造出来的

这座岛几乎全部由**代码生成**，只有少量"扫描件"是外部资产。理解这条分界线，就能知道某个东西该去哪里找。

### 6.1 程序化生成的部分

| 东西 | 入口文件 | 生成方式 |
|---|---|---|
| 岛屿形状与高度 | `src/world/TerrainData.js`、`src/world/terrain/IslandShape.js`、`src/world/terrain/TerrainNoise.js` | 噪声函数 + 岛屿形状函数合成高度图（2048×2048），海滩、山丘、岬角由此而来 |
| 地形渲染与烘焙 | `src/world/TerrainGPU.js`、`src/world/terrain/TerrainBake.js`、`src/world/terrain/TerrainShading.js` | 高度图上传为纹理；烘焙出法线、细节纹理；并以 WGSL 模块形式对外提供 `terrainHeightAt()` |
| 村庄 | `src/world/Village.js`、`src/world/village/*` | 程序化建筑 + 纹理烘焙，并**把地基压平写回高度图**（因此必须最先建） |
| 礁石 | `src/world/Reef.js`、`src/world/reef/*` | 噪声驱动的礁体几何与材质，随浪摆动 |
| 植被 | `src/world/Vegetation.js`、`src/world/vegetation/*` | 按规则散射实例，用 impostor（用一张图代替完整模型）+ 抖动淡出实现远景 |
| 岩石、海漂杂物 | `src/world/Rocks.js`、`src/world/Debris.js` | 程序化形状 + 部分扫描件资源 |
| 野生动物与鲸鱼 | `src/world/wildlife/*`、`src/world/marine/*` | 程序化几何 + 行为逻辑（`WhaleBrain.js` 是鲸的决策，`BirdPose.js` 是鸟的姿势） |
| 船只 | `src/world/boat/*` | 由若干几何构建器拼出来（`HullBuilder.js`、`Wheelhouse.js`…） |
| 鱼 | `src/world/fish/*` | 见下 |

### 6.2 一份参数表同时驱动几何、皮肤和玩法

鱼类是"数据驱动"在这两个层同时生效的典型例子：

```text
src/world/fish/FishSpecies.js   造型参数表（SPECIES）
    每个物种：体高/体宽随体长的曲线、嘴形、眼睛、鳃盖、鳞片、
              背鳍/臀鳍/胸鳍/腹鳍/尾鳍的形态与鳍条数、皮肤图案编号
        │
        ├──> FishGeometry.js   按参数生成三维网格
        ├──> FishMaterial.js   按 pattern 编号选择皮肤着色器
        └──> FishProps.js      附带的小道具（例如吸盘鱼）

src/game/FishTable.js           玩法参数表（FISH）
    每个物种：栖息地权重、体重区间、价格、挣扎强度、耐力、活跃时段、稀有度
    model 字段 -> 指向上面的 SPECIES 键
```

也就是说，要让一条鱼"能钓、好看、行为合理"，需要填两张表并通过名字对齐。
这种"一个概念的两套数据分开管理"的做法，在真实项目里很常见，读代码时要记得**两边都可能需要改**。

### 6.3 离线资产与三条流水线

`public/` 下是运行时直接加载的静态资源，`tools/` 下是"重新生成这些资源"的脚本（不参与构建，按需手动跑）：

| 资源 | 运行时位置 | 来源与流水线 |
|---|---|---|
| 钓鱼音效（四十多个 ogg） | `public/audio/` | `tools/audio/`：从 Freesound 搜 CC0 录音 → 下载 → 切片/循环做等功率交叉淡化 → 响度归一化 → 输出到 `public/audio/`，并把测量出的响度写进 `src/audio/soundBank.js` |
| 摊位道具与招牌 | `public/models/props/` | `tools/props/`：Poly Haven CC0 扫描件 → Blender 减面 → 打包成 `props.bin` / `props.json`，用 ImageMagick 生成招牌图 |
| 两位摊主角色 | `public/models/characters/` | `tools/characters/`：Microsoft Rocketbox 头像 → 贴图转换 → Blender 重定向动画并导出 GLB |

三个共同的工程特征，对后端开发者也不陌生：

- **生成脚本与产物分离**：产物提交进仓库（因为运行时要么不能联网、要么不该在浏览器里做解码转换），脚本保留可复现性。
- **体积是被认真对待的**：减面、贴图降分辨率、把多个模型打进一个二进制包、把多个音效切进一个 bank，都是为了减少请求数和显存占用。
- **许可与来源被记录**：`CREDITS.md`、`public/audio/CREDITS.md`、`public/models/*/CREDITS.md` 记录了每个外部资源的来源与许可证。

### 6.4 世界坐标是集中定义的

`src/world/WorldLayout.js` 是一个普通的常量对象，定义了整座岛的"平面图"：地形范围、海滩区间、码头位置与朝向、船停泊点、村庄与礁石的中心和半径、玩家出生点、涌浪方向。

好处是显而易见的：需要"在码头上放一个灯"时，不需要猜坐标，直接读 `WORLD.pier`。
读世界相关代码时，**先看这个文件**能省很多时间。

---

## 7. 着色器代码是怎么组织的

一个物体要长成什么样，最终落在一个 `Material` 上。这个项目的材质系统不是"选一个内置材质类型"，而是**往一个统一的网格着色器模板里插几段 WGSL**：

```js
new Material( {
    name: 'rock',
    modules: [ terrainModule ],                     // 我需要调用别人提供的 WGSL 函数
    uniforms: { tint: [ 'vec3f', color ] },         // 传给我的常量（JS 侧是 mat.tint）
    textures: { rockAlbedo: tex },                  // 传给我的纹理
    vertex:  `v.position += v.normal * 0.01;`,      // 顶点阶段：改位置/输出额外插值量
    surface: `s.albedo = mat.tint; s.roughness = 0.8;`,  // 表面阶段：写反照率/粗糙度/法线/自发光…
    output:  `r.color = ...;`,                      // 输出阶段：多渲染目标写哪个附件
} )
```

三段钩子各自对应上一节流水线里的位置：

```text
vertex   -> 流水线 ①（顶点着色器）
surface  -> 流水线 ③ 的前半：决定这个像素的材质属性
output   -> 流水线 ③ 的后半：决定写进哪些附件（颜色 / 速度 / 遮罩）
```

所以"读懂某个物体的外观"= 找到它的材质 = 读它插进去的那几段 WGSL。
这些片段通常很短，真正的工作由 `modules`（共享函数库）和引擎模板完成。

另外有三类开关会以编译期定义的形式注入材质（影响性能与正确性，值得留意）：
`underwaterLighting`（水下光照等级）、`appliesHillShadow`（是否自算地形阴影）、`localLightsCheap`（局部光照是否只算 Lambert）。
`App.js` 里成批设置它们，是在权衡"着色器要采样的贴图数量"和"画面质量"——GPU 每个着色器能采样的纹理数量是有上限的。

---

## 8. 性能与调试入口

这个项目把"能自我观测"当成了一等公民，调试入口几乎都在 URL 参数或键盘上：

| 入口 | 作用 |
|---|---|
| `?bench` | 启动帧时间基准测试（`src/core/Bench.js`），会自己驱动帧，可用于自动化 |
| `?bench&shots=view1,view2` | 从命名视角输出参考截图（`src/core/DebugViews.js`） |
| `?wdbg=N` | 切换水面着色器的调试视图 |
| 设置面板（按 `H`） | 海况、时刻、太阳方位、云、雾、后处理、渲染缩放 |
| `F` | 自由相机（排查"这个物体到底在哪"最有效的工具） |
| fps 计数器（左上角） | 显示帧率、平均帧时间、最差帧时间；profiler 开启时还显示 GPU 计算/渲染耗时 |
| `Profiler` + GPU 时间戳 | 追踪指定计算内核（如 FFT 的行/列 pass、天空视角 LUT）的 GPU 时间 |
| `?noClouds` 等开关 | 逐项关掉系统做 A/B 对比，定位性能或画面问题的来源 |

性能相关的三条经验（都写在代码注释里，值得记住）：

1. **采样纹理的数量是硬约束**：材质里 sampler 越少越快，`LOCAL_LIGHTS_CHEAP` 这类开关就是为此存在的。
2. **半分辨率是常用手段**：AO、折射、bloom、深度的半精度副本都在降低带宽，而带宽通常是瓶颈。
3. **首次使用的编译开销必须提前支付**：预编译 + 预热两帧，是为了把卡顿关在加载画面里。

如果只想快速验证某个系统，`test/` 下对应名字的脚本通常能无头跑出图片：

```sh
node test/ocean-fft.mjs /tmp/ocean.png      # 各系统的命名规则：主题-对象.mjs
node test/engine-smoke.mjs /tmp/smoke.png
```

---

## 9. 延伸阅读

```text
docs/PORTING.md             three.js / TSL 与本引擎概念的完整对照表；WGSL 陷阱清单
src/engine/render/SceneRenderer.js   四步渲染顺序的原始注释
src/post/PostFX.js                   后处理链的原始注释
src/ocean/OceanFFT.js                FFT 海洋的完整数据布局与模块接口
src/engine/render/Frame.js           frame 结构体全部字段（知道有哪些全局量可用）
tools/*/README.md                    三条资产流水线的操作说明
```

---

## 10. 一句话总结

```text
CPU 每帧只做两件事：把世界推进一点点，然后录制一串 GPU 命令。
GPU 侧的一切都是"输入安排好、函数对每个元素跑一次"：
    顶点着色器摆位置，片元着色器算颜色，
    需要"看到别的东西"就先把它渲染到一张纹理里。
海面、天空、地形这些复杂效果，靠的是把物理模型放进 GPU 的批量计算，
再用统一的 WGSL 模块约定把它们拼在一起。
```
