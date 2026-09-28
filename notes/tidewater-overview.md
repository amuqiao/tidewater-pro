# Tidewater 项目总览：一个 Python 开发者眼中的浏览器游戏

一句话：Tidewater 是一个跑在浏览器里的 3D 岛屿钓鱼游戏，它自带一套渲染引擎——所以它同时是"一个游戏"和"一个小型图形框架"，读代码时先把这两件事分开，其余都会变简单。

## 适用读者

有 Python 开发经验、想读懂这个仓库，但没有游戏开发或图形编程经验。文中不预设 WebGL / three.js / 图形学背景。

## 这篇文档负责什么

- 建立"这个项目在干什么"的整体心智模型。
- 给出目录地图，说明每层代码的职责边界。
- 讲清**一帧的生命周期**——它是整个项目的骨架。
- 说明玩法逻辑层的组织方式，以及测试和构建怎么跑。
- 给出建议的阅读顺序。

## 不负责什么

- 不解释每个渲染效果的数学与着色器写法：那是 [tidewater-rendering.md](tidewater-rendering.md) 的范围。
- 不复述玩法说明与键位（见根目录 `README.md`）。
- 不介绍开发流程约定（见 `AGENTS.md`）。

---

## 1. 先建立心智模型

### 1.1 最容易搞错的三个前提

如果你只带 Python 经验来读这份代码，下面三件事最容易误解，而它们决定了一半的代码形状。

**① 画面不是"更新"，而是每帧从零重画。**

一个 Python 后端服务的直觉是：数据变了，去更新那个发生变化的地方。
3D 实时渲染不是这样。每个画面（一帧）都是重新画一张完整的画：

```text
第 1 帧: [清空] [画地形] [画建筑] [画水] [画后处理] -> 屏幕
第 2 帧: [清空] [画地形] [画建筑] [画水] [画后处理] -> 屏幕   (全部重来)
第 3 帧: ...
```

没有"上一帧的像素"这回事（少数效果除外，比如时间抗锯齿会参考上一帧）。
这解释了两件事：
- 为什么代码里到处是 `update( dt )`——每帧都要把世界推进一点点；
- 为什么"性能"在这个项目里是头等大事，而在普通 Web 应用里往往不是。

**② GPU 是另一个"进程"，你只是给它录命令，它稍后执行。**

JavaScript 代码（CPU 侧）不直接画像素。它做的是：

```text
CPU: 录制命令  ->  录制命令  ->  录制命令  ->  ...  ->  一次性提交给 GPU
GPU:                                              (异步执行, 你不知道何时画完)
```

用 Python 打比方：像用 `sqlite3` 时先把语句攒进一个 transaction，最后 `commit()`；
或者像给一个远程任务队列连续 `enqueue`，最后统一触发。
GPU 的这套模型有个重要后果：**同一个缓冲区在一帧里写两次，只有最后一次生效**（命令是延后执行的）。代码里多处注释在提醒这一点。

**③ 每个模块都遵守同一个隐式约定：`update( dt )`。**

这个项目没有大型框架、没有依赖注入容器、没有事件总线（除了少量例外）。
它靠一个约定把几十个系统串起来：

```text
每个系统 = 一个普通类
    new System( { 依赖 } )      构造时拿到引用
    system.update( dt )         每帧被调用一次, dt 是上一帧到现在的秒数
    system.mesh                 如果它要出现在画面里, 就暴露一个可加入场景的网格
    system.module               如果别的系统要在着色器里调用它, 就暴露一段 WGSL 代码
```

这和 Python 里"每个类实现 `step()` 或 `__call__()`"是同一类设计——只是约定没有写在接口文件里，而是体现在 `App` 的调用顺序中。
**所以读这个项目的正确方式不是从上到下读每个文件，而是先读 `src/App.js`，看它按什么顺序调用谁。**

### 1.2 概念对照表（Python → 这个项目）

| Python 世界 | 这个项目 | 说明 |
|---|---|---|
| `pip` / `requirements.txt` | `npm` / `package.json` | 只有一个运行时脚本入口：`npm test` |
| `venv/` | `node_modules/`（`npm install` 生成） | 未提交到仓库 |
| 模块 `import` | ESM `import` / `export` | 没有 `__init__.py`；目录下的 `index.js` 起聚合导出的作用 |
| `class` + 组合 | 同样的 `class` + 组合 | 几乎没有继承；组合 + `update( dt )` 是主旋律 |
| `dataclass` / 配置表 | 普通对象字面量（`export const FISH = { ... }`） | 玩法数据就是"模块里的一个大字典" |
| `random.seed( 42 )` | 注入 `rng = () => ...` 参数 | 测试中可复现的随机数靠传函数实现 |
| `pytest` | `test/*.mjs`（`npm test`） | 两类测试：纯逻辑、GPU 冒烟。见第 7 节 |
| 长跑的服务进程 | `requestAnimationFrame` 循环 | 浏览器每屏刷新一次就调一次回调 |
| `numpy` 数组 | `Float32Array` / `StorageBuffer` | 定长、同类型的紧凑数组；GPU 版还能驻留显存 |
| 多进程 / Celery worker | GPU（`device` / `queue` / `encoder`） | 另一套执行设备，异步、批量提交 |
| 生成报告给前端 | 画一帧到 `<canvas>` | 输出的不是 JSON，而是一张图 |

### 1.3 一个游戏中立的直觉：世界状态 vs 每帧派生

代码里有两类数据，读的时候要分清，否则会觉得"这个值到底谁负责"：

```text
世界状态（会随游戏推进而变）        每帧派生（每帧重新算出来）
  - 玩家位置、朝向                     - 相机矩阵
  - 船的速度、剩余油量                 - 当前潮位、水面高度
  - 时间（几点）、天气参数             - 天空颜色、太阳方向
  - 鱼篓里的鱼、装备等级               - 所有可见网格的最终变换与光照
```

规则很简单：**能算出来的就不存**。存档里只有左边那一列（见 `src/game/GameState.js`），右边的全部每帧重算。

---

## 2. 项目分四层

弄清楚层次，就能判断一个文件"该不该出现在这里"。

```text
src/
├── engine/        [第 1 层] 引擎：数学、场景图、几何、GPU 资源、着色器组装
├── ocean/ sky/    [第 2 层] 内容系统：水、天空、地形、村庄、生物、船、后处理
│   world/ post/
│   fx/ materials/
├── game/          [第 3 层] 玩法：鱼、咬钩、溜鱼、鱼篓、商店、HUD 数据
├── player/        [第 2.5 层] 玩家/摄像机：走路、游泳、开船、自由相机
├── audio/ ui/     [第 3 层] 声音与界面（DOM 层）
├── core/          [装配辅助] 全部是转发壳，见下方提示
├── App.js         [装配层] 把上面所有东西按顺序造出来，并定义每帧的调用顺序
└── main.js        [入口] 建 App，跑起来
```

| 层 | 目录 | 职责 | 判断标准 |
|---|---|---|---|
| 引擎 | `src/engine/` | 与"做的是什么游戏"无关的能力：向量矩阵、场景树、缓冲区、渲染管线、阴影 | 换个游戏还能用 |
| 内容系统 | `src/ocean/`、`src/sky/`、`src/world/`、`src/post/`、`src/fx/`、`src/materials/` | 这座岛的每个组成部分：海浪、云、地形、礁石、村民、飞鸟、后处理 | 换成沙漠场景就要重写 |
| 玩法 | `src/game/` | 钓鱼规则：什么鱼在什么地方咬钩、怎么溜、卖多少钱、升级什么 | 和画面没关系，纯逻辑 |
| 装配 | `src/App.js` | 决定"世界由哪些系统组成"和"每帧按什么顺序推进" | 唯一的全局协调者 |

### 2.1 两个容易踩的坑

**坑一：`src/core/` 里几乎什么也没有。**

```js
// src/core/Engine.js 的全部内容
export { Engine } from '../engine/Engine.js';
```

`src/core/Globals.js`、`src/core/SceneRenderer.js` 也一样，各一两行，只是把 `src/engine/` 里的东西换个名字再导出。
这是移植期的过渡层（历史见 `docs/PORTING.md`）：老代码从 `core/` 导入，实现已经搬进 `engine/`。
**看到 `src/core/*.js` 就去 `src/engine/` 找真正的实现。**

**坑二：引擎有两个入口文件，分别对应 CPU 侧和 GPU 侧。**

```text
src/engine/index.js    CPU 侧：数学、场景图、几何，API 刻意做成 three.js 兼容
src/engine/webgpu.js   GPU 侧：设备、纹理、缓冲区、着色器模块、计算内核
```

`engine/index.js` 顶部注释写得很直白：`import * as THREE from '../engine/index.js'` 这种写法是可以工作的，是给移植文件用的临时手段。
所以 `import * as THREE from '../engine/index.js'` 里那个 `THREE` 并不是第三方库，是项目自己的引擎。

这两个入口也对应了读代码时的两种视角：

- 想理解**场景里有什么、物体在哪** → 看 CPU 侧（`Scene`、`Mesh`、`Vector3`…）；
- 想理解**像素怎么被算出来** → 看 GPU 侧（`GPU`、`Texture`、`ShaderModule`、`ComputeKernel`…）。

---

## 3. 启动：从打开网页到第一帧

浏览器加载 `index.html`，它做两件事：放一个加载画面（loading screen，带美术图和提示文字），然后加载 `src/main.js`。

`src/main.js` 很短，做完三件事就结束：

```text
1. new UI()        建界面（DOM，不是画进 canvas）
2. new App()       建应用
3. app.init( 进度回调 )      -> 等它完成
   app.ui = new AppUI( app, ui )
   ui.hideLoader()
   app.start()      -> 开始每帧循环
```

### 3.1 `App.init()`：一个顺序敏感的长函数

它按固定顺序建造整个世界，每个阶段都会给加载进度条报一个百分比：

| 进度 | 阶段文字 | 建立了什么 |
|---|---|---|
| 0.02 | Starting WebGPU… | `Engine`：canvas、WebGPU 设备、场景、相机 |
| 0.04 | Building the atmosphere… | 大气、天空、云、级联阴影、环境光照 |
| 0.06 | Shaping the island… | 地形高度数据、碰撞体 |
| 0.12 | Building the village… | 村庄（会先把建筑地基压平到高度图里，所以必须早于地形数据派生） |
| 0.14 | Planting the island… | 植被（可被 `?noVeg` 关掉） |
| 0.19 | Rolling in the swell… | 岸线场、地形 GPU 纹理、地形网格、岩石、海漂杂物 |
| 0.23 | Growing the reef… | 礁石 |
| 0.30 | Simulating the ocean… | FFT 海面、浅水模拟、焦散、浪花、船、玩家、野生动物 |
| 0.34 | Preparing the shaders… | 后处理链、水下效果、雾气、渲染缩放、音频与玩法 |
| 0.36 → 0.95 | Compiling shaders… | 预编译所有渲染管线（最慢的一段，见下） |
| 0.96 | Warming up… | 先跑两帧，把首次使用的开销也留在加载画面里 |

这个顺序不是随手写的，它编码了真实依赖：

```text
村庄要先建  ->  因为它会改写地形高度（压平建筑地基）
地形要先于   ->  岸边场、GPU 纹理、植被散射、礁石、野生动物(它们都要查询地形高度)
海面 FFT 要先 ->  礁石(珊瑚随浪摆动)、船(浮力)、水下游光、喷雾、鲸鱼
```

**预编译（precompile）值得单独记一笔。**
在浏览器里，第一次用到某个材质组合时，GPU 需要编译对应的着色器，这会造成明显卡顿。
这个项目选择在加载阶段就把所有管线编译完：遍历所有网格、渲染所有 pass、连水下和水上的两个水面变体都强制走一遍。
代价是首次加载可能要一分钟以上，收益是进游戏后不掉帧。这就是 `README.md` 里"第一次加载会编译几百个着色器"的来源。

### 3.2 主循环只有十行

`App.start()` 把控制权交给引擎，之后每秒跑约 60 次：

```js
// src/engine/Engine.js 的 start()，语义重述
loop( t ):
    更新计时器
    dt = 距上一帧的秒数;   if dt > 0.1: dt = 0.1     // 切后台回来时钳制，避免物理炸掉
    frame += 1
    update( dt, elapsed )      // 这就是 App._frame
    排下一帧 requestAnimationFrame( loop )
```

和 Python 的对照：这本质上是一个 `while True` 事件循环，只不过"事件"是屏幕刷新；`dt` 相当于"两次迭代之间的时间差"。
`dt` 被钳制在 0.1 秒是游戏开发的常规防御：标签页被切到后台时浏览器会停掉动画帧，回来后 `dt` 可能是几十秒。

---

## 4. 一帧里发生了什么（全项目骨架）

`App._frame()` 是整个项目最值得精读的函数：它把几十个系统的调用顺序写成了一个列表。
顺序本身就是设计文档——它表达了"谁依赖谁"。

```text
帧开始
 │
 ├─ 1. 开帧         GPU.beginFrame()、帧计数、累积时间、推进一天中的时刻
 │
 ├─ 2. 处理按键     F 自由视角 / T 时间暂停 / L 手电 / M 静音（"本帧刚按下"语义）
 │
 ├─ 3. 玩家与船     boatCtl.update -> boatSpray -> wake        （船先走，相机才能跟上本帧的位姿）
 │                  player.update 或 fly.update                 （玩家或自由相机）
 │                  game.update                                （钓鱼玩法）
 │                  updateSun                                  （由时刻算出太阳/月亮方向）
 │
 ├─ 4. 天空与水面   atmosphere.update -> applyAtmosphereReadback（把 GPU 回读的太阳透过率写进全局量）
 │                  fft.update -> seaDetail.update（波浪频谱演化）
 │                  waterQuery：把本帧要查询的水面高度排进队列 -> 回读 -> 决定相机是否在水下
 │                  caustics / marineSnow / airMotes / shoreSim / 水下光照 / 浪花 / 喷雾 / 云 / 环境
 │
 ├─ 5. 世界         oceanLOD / terrain / rocks / debris / reef / village / vegetation
 │                  whale / boat / wildlife / localLights       （按相机位置做 LOD 与剔除）
 │
 ├─ 6. 渲染         exposure -> 相机速度
 │                  post.beginFrame()（决定内部渲染分辨率与 TAA 抖动，并写入相机矩阵）
 │                  shadows.render()      （级联阴影贴图）
 │                  sceneRenderer.render()（不透明 -> 复制 -> 船体遮罩 -> 水与透明，见渲染篇）
 │                  post.render()         （后处理链，最后画进 canvas）
 │
 ├─ 7. 提交         GPU.submit()（把整帧录制的命令一次性交给 GPU）、profiler
 │
 └─ 8. 收尾         updateAudio（按世界状态更新声音）、ui.update、input.endFrame（清掉"刚按下"）
帧结束
```

### 4.1 读这个顺序时要抓住的四件事

**① 先推进世界，再渲染。** 前半段全是 `update`（改状态），真正"画"的部分集中在第 6 步。
这就是第 1 节说的"世界状态 vs 每帧派生"：渲染阶段基本只读状态、不改状态。

**② 船的物理排在玩家之前。** 注释写明了原因：相机要跟随本帧的船姿，所以船必须先算完。

**③ 水的高度是"问"出来的，不是"猜"出来的。**
波浪在 GPU 上以频谱形式存在，CPU 想知道"某点水面多高"就必须发一次查询（`WaterQuery`），本帧排队、回读，再据此判断相机是否在水面以下。
这是 GPU 编程的一个典型模式：**数据在 GPU 上，CPU 需要时就产生一次往返**，所以代码会把多个查询攒起来一次问完。

**④ 输入有"电平"和"边沿"两种读法。**
`input.down( 'KeyW' )` 是"现在按着吗"（用于持续移动），`input.hit( 'KeyR' )` 是"本帧刚按下吗"（用于开关类动作）。
`endFrame()` 在每帧末尾清掉"刚按下"集合，所以 `hit` 每按一次只会返回一次 `true`。
这相当于 Python 里区分 `is_pressed` 和 `just_pressed` 两个状态，写 UI 或游戏逻辑时弄混会导致"按一下触发 60 次"。

### 4.2 两条"看不见的轨道"

除了这个调用列表，还有两条贯穿全帧的约定，读代码时会反复遇到：

```text
GPU 命令轨道
   所有系统在 update/render 里只是"录制"命令（画这个、算那个），
   GPU.submit() 一次性提交。好处是顺序完全由代码决定，代价是同一缓冲区一帧只能有一个值。

着色器共享轨道
   需要被别的系统在着色器里复用的系统，暴露 this.module（一段具名的 WGSL 代码）。
   例如地形提供 terrainHeightAt( xz )，浅水模拟提供 shoreSimSample( xz )，
   水面材质在顶点着色器里就能直接调用它们——相当于"GPU 上的函数调用"。
   前缀命名规则和完整清单见 docs/PORTING.md 的 ShaderModule 一节。
```

---

## 5. 玩法是怎么实现的

玩法层里最接近 Python 后端直觉的是**规则部分**：鱼表、咬钩判定、溜鱼模型、存档，都是纯逻辑 + 数据表 + 状态机，不需要 GPU，也能脱离浏览器运行（见第 7 节）。
表现部分（鱼竿的三维模型、摊位、HUD）仍然会创建 3D 对象并每帧更新，但它只读规则层算出来的结果。

### 5.1 四个组成部分

```text
FishTable.js    数据表：18 种可钓的鱼，每种一行（体重范围、栖息地、价格、挣扎强度、活跃时段…）
Bites.js        纯函数：给定"钓点 + 时刻"，算出那里有什么鱼、多久咬钩、多重
FishingRod.js   状态机 + 表现：杆、线、浮漂的姿态与状态转换
GameState.js    存档：钱、鱼篓、图鉴、装备等级（localStorage）
Game.js         胶水：每帧读输入、驱动上面几个、把结果交给 HUD 和音频
```

### 5.2 数据表驱动的设计

`src/game/FishTable.js` 里没有逻辑，只有一张表：

```js
silverside: { name: 'Hardhead silverside', habitat: { shallows: 1, pier: 0.6, bay: 0.2 },
              kg: [ 0.02, 0.08 ], price: 3, fight: 0.05, stamina: 1.5, time: 'any', rarity: 0.2 },
```

想加一种鱼，改这张表就够了——它同时影响咬钩概率、体重分布、售价、溜鱼难度和 HUD 显示。
这就是"数据驱动"的含义：**把可调参数集中到数据里，逻辑代码只负责解释数据**。
Python 里对应的做法是把配置写成 dict/JSON，而不是把常量散落在函数中。

顺带一个跨层的连接点：表里的 `model` 字段指向 `src/world/fish/FishSpecies.js` 里的造型参数——
也就是说**一条鱼的"玩法数据"和"三维建模数据"是两套独立的表，靠名字对齐**。
玩法层完全不知道那条鱼长什么样，世界层也不知道它值多少钱。

### 5.3 纯函数 + 注入随机数

`Bites.js` 里有三个关键函数，签名都是"输入 + 一个随机数函数"：

```js
habitatAt( { depth, reefDist, pierDist } )    -> 各处水域的"鱼的密度"权重（0..1）
pickSpecies( habitat, hour, rng )             -> 按权重抽一种鱼
rollWeight( id, rng )                         -> 抽一个体重（偏向小值：大鱼罕见）
biteDelay( habitat, hour, rng )               -> 下一次咬钩要等几秒
```

`rng` 默认是 `Math.random`，但测试里会传一个可复现的伪随机函数（见第 7 节）。
这相当于 Python 里把 `random.random` 作为参数传进去，而不是在函数内部直接调用——
**可测试性来自"依赖可以替换"，而不是来自 mock 框架**。

### 5.4 钓鱼是一次状态机流转

`FishingRod` 用一个字符串表示当前状态，玩法层根据状态决定"这次点击意味着什么"：

| 状态 | 怎么进入 | 此时左键的含义 |
|---|---|---|
| `stowed` | 初始 | —（先按 `R` 取出鱼竿） |
| `idle` | 按 `R` | 按住 = 开始蓄力 |
| `windup` | 按住左键 | 松开 = 抛出（蓄力越久越远） |
| `flick` | 松开左键 | —（抛出动作的短过渡） |
| `flying` | 抛竿完成 | —（右键可提前收回） |
| `floating` | 落水 | 咬钩时左键 = 扬竿；右键 = 收线 |
| `retrieving` | 右键收线 | —（线在回收） |
| `fighting` | 扬竿成功 | 按住 = 收线，松开 = 放线 |
| `landing` | 鱼被拉上岸 | 左键 / `E` / `Esc` = 关掉战利品卡片 |

主路径可以简化成一条线：

```text
stowed --R--> idle --按住--> windup --松开--> flick --> flying --落水--> floating
                                                                        |
                                                   咬钩+左键 --> fighting --> landing
```

同一时刻"左键按下"在不同状态下含义完全不同（蓄力 / 扬竿 / 收线），这就是为什么 `Game.update()` 里是一串 `if ( rod.state === ... )`。
游戏开发里这是常规写法，不是坏味道：状态机的分支本身就是设计。

### 5.5 存档只存"不可推导的东西"

`GameState` 的内容是钱、鱼篓条目、图鉴（每种鱼的最大记录）、装备等级、剩余油量，
存进浏览器的 `localStorage`（键名 `tidewater.save.v1`），每次变更后立即保存。

它给 Python 开发者提供的两个可借鉴点：

- **存储介质是注入的**：构造函数接收一个 `storage` 对象（默认 `localStorage`），测试传入一个 `Map` 包装器。浏览器隐私模式会禁用存储，所以每次访问都做了保护并且只在必要时降级。
- **版本与迁移**：键名里带 `v1`；读取旧存档时会把缺失的字段（例如后来才加入的鱼长）补算出来，而不是直接报错。

---

## 6. 输入、音频、界面

| 系统 | 位置 | 它是什么 | 值得记的一点 |
|---|---|---|---|
| 输入 | `src/core/Input.js` | 键盘/鼠标状态 + 指针锁定 | 只记录状态，不做任何"哪个键该干什么"的判断；后者属于 `Player` 和 `Game` |
| 音频 | `src/audio/SoundScape.js` | 基于真实录音的采样播放器 | 全部音频都是真实录音，没有合成；海浪声与模拟的每一个破浪事件对齐 |
| 界面 | `src/ui/UI.js`、`src/ui/ui.css` | 纯 DOM（HTML 元素），不是画进 canvas 的 | 所以 HUD/设置面板可以用浏览器的调试器直接检查 |
| 玩家 | `src/player/Player.js` | 四种模式：`walk` / `swim` / `boat` / `deck` | 模式决定用哪套物理，切换点写在玩家代码里 |

两个跨系统的约定：

- **音频不是独立播放的，是被世界驱动的。** `App.updateAudio()` 每帧把"相机在哪、是否在水下、离岸多远、浪多大、是否在船上、引擎转速"打包交给 `SoundScape`，由它决定播放什么。
- **界面和数据分离。** `GameHUD`、`Minimap`、`Guide` 从玩法层读取状态来显示；玩法层不直接操作 DOM。

---

## 7. 测试与构建

### 7.1 两类测试，一条命令

```sh
npm test     # = node test/game-logic.mjs && node test/engine-smoke.mjs
```

| 测试 | 位置 | 需要 GPU 吗 | 测什么 |
|---|---|---|---|
| 玩法逻辑 | `test/game-logic.mjs` | 不需要 | 咬钩分布、溜鱼结果、鱼篓/钱包、存档往返、旧存档迁移 |
| 引擎冒烟 | `test/engine-smoke.mjs` | 需要（无头） | 真的建 WebGPU 设备、画一帧、把结果写成 PNG |

玩法测试就是普通的 Node 脚本，但它有两个值得学的技巧：

```text
① 可复现的随机数
   let seed = 12345;
   const rng = () => ( seed = ( seed * 1664525 + 1013904223 ) >>> 0 ) / 4294967296;
   所有涉及随机的玩法函数都接受这个 rng。等价于 Python 里自己实现一个带种子的随机源。

② 用"策略函数"模拟玩家
   careful: g => g.tension < 0.68 && g.surge < 0.6     // 会收放线的玩家
   mash:    () => true                                 // 一直按住收线
   idle:    () => false                                // 完全不收
   同一个溜鱼模型跑三种玩家策略，断言"谨慎的人能上小鱼、猛拉的人断线、不收线的人跑鱼"。
   这比断言具体数值稳定得多，也更像在描述游戏设计意图。
```

引擎冒烟测试展示了无头 GPU 测试的完整套路（`test/headless.mjs` 提供基础设施）：

```text
import './headless.mjs'         装好 navigator.gpu（用 npm 包 webgpu 提供的 Dawn 实现）
await GPU.init( { headless: true } )
建一个小场景（地面 + 盒子 + 金属球 + 实例化方块）
GPU.beginFrame() -> 阴影 -> 渲染 -> 色调映射 -> GPU.submit()
readTexture() 把结果读回内存 -> writePNG() 写成 PNG（自己实现的极简 PNG 编码器）
```

也就是说：**没有浏览器也能跑 GPU 代码并把画面存成图片**，看图片就知道渲染对不对。

### 7.2 `test/` 里其它文件是什么

`test/` 下共有数十个 `.mjs` 和几个 `.html`，但 `npm test` 只跑上面两个。
其余按系统命名（`ocean-*`、`sky-*`、`world-*`、`life-*`），是各系统的开发期 harness：单独运行时用来验证某个系统、输出参考图或做计时。
还有几个 `.html` 是浏览器里的调试页（HUD、加载器）。**它们不是 CI 的一部分，改代码时按需手动跑。**

### 7.3 构建与部署

```sh
npm install
npm run dev      # http://127.0.0.1:5189   （开发服务器，端口由 package.json 指定）
npm run build    # 产物在 dist/
```

- 构建工具是 Vite：把 `src/` 下几十个 ESM 模块打包、压缩成浏览器可分发的静态文件。相当于"Python 应用打成一个可部署目录"，只是不需要运行时解释器——浏览器就是运行时。
- `vite.config.js` 里 `base: './'` 让产物使用相对路径，因此可以部署在任意子路径下（GitHub Pages 把它放在 `/tidewater/` 下）。
- 注意一个容易困惑的细节：`vite.config.js` 里写的是 `server.port: 5188`，但 `package.json` 的 `dev` 脚本用 `--port 5189`。命令行参数优先，实际开发端口是 **5189**。
- 部署：`.github/workflows/deploy.yml`——每次 push 到 `main` 就 `npm ci && npm run build`，把 `dist/` 发布到 GitHub Pages。

---

## 8. 建议的阅读顺序

不要按目录顺序读。按"骨架 → 分支"的顺序：

```text
① 建立骨架
   src/main.js              47 行，看它如何组装 UI 和 App
   src/App.js               只看 init() 的进度阶段 + _frame() 的调用顺序（其余先略读）
   src/engine/Engine.js     约 100 行，看主循环与 canvas/相机/场景的建立

② 理解一帧里"更新"的部分
   src/core/Input.js
   src/player/Player.js     模式切换
   src/game/Game.js         玩法主循环（先看 update() 的分支结构）

③ 理解玩法
   src/game/FishTable.js    数据表
   src/game/Bites.js        纯函数
   src/game/GameState.js    存档
   test/game-logic.mjs      读测试往往比读实现更快理解预期行为

④ 再进入渲染（此时读 tidewater-rendering.md）
   src/engine/render/SceneRenderer.js
   src/engine/gpu/GPU.js
   src/App.js 的渲染段

⑤ 按兴趣挑内容系统
   水：src/ocean/OceanFFT.js -> WaterSurface.js -> WaterMaterial.js
   天空：src/sky/Atmosphere.js -> Sky.js -> Clouds.js
   地形：src/world/TerrainData.js -> TerrainGPU.js -> Terrain.js
```

配套操作建议：

- `npm run dev` 打开页面，用 URL 参数逐个关掉系统（`?noClouds`、`?noCaustics`、`?noVeg`、`?noSim`），观察画面差异——比读代码快得多。
- 按 `H` 打开设置面板改时间、海况、雾；按 `F1` 看键位。
- `?bench` 会启动帧时间基准测试（`src/core/Bench.js`），`?fly` 直接进自由相机。

---

## 9. 一句话总结

```text
App.js 定义了"世界由什么组成、每帧按什么顺序推进"；
engine/ 提供与游戏无关的绘制能力；
ocean/ sky/ world/ 用数学和着色器把这座岛"造"出来；
game/ 用数据表和状态机在上面叠一层钓鱼规则；
一切通过 update( dt ) 这个隐式约定串成一条每秒 60 次的流水线。
```

读完本篇后，下一站是 [tidewater-rendering.md](tidewater-rendering.md)：把上面第 4 节里"渲染"那一步展开，看像素究竟是怎么算出来的。
