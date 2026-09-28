# 运行与部署 Tidewater

一句话：这个项目有三条互不冲突的"上线"通道——开发服务器（改代码用）、构建产物 + 静态服务（验证发布用）、GitHub Pages 自动部署（给别人玩用）；选哪条取决于你要做什么，而不是越靠后越"正式"。

## 适用读者

想在这台机器上把游戏跑起来、或想把它发到公网的人。不要求你懂前端构建。

## 这篇文档负责什么

- 列出运行前需要什么。
- 给出本地启动、构建、预览的准确命令与实测结果。
- 说明公网部署（GitHub Pages）的完整链路与一次性设置。
- 给出"打不开 / 打不开得慢 / 帧率低"的排查顺序。

## 不负责什么

- 玩法说明、键位和 URL 参数清单：见根目录 `README.md`。
- 代码结构与渲染原理：见 [tidewater-overview.md](tidewater-overview.md) 和 [tidewater-rendering.md](tidewater-rendering.md)。

---

## 1. 心智模型：三条通道

```text
                    ┌─ ① npm run dev ────────────> http://127.0.0.1:5189   （开发用，带热更新）
   一份源码 ────────┼─ ② npm run build + preview ─> http://127.0.0.1:4173   （验证发布产物）
                    └─ ③ git push origin main ────> GitHub Actions ──> GitHub Pages（仓库自带 workflow，见 4.4）

                    └─ ④ python3 -m http.server -d dist   （无 Node 的最简部署，见 4.1）
```

四条通道产出的是同一样东西：**一堆静态文件**。这个项目没有后端、没有数据库、没有服务端渲染——所有计算（物理、海浪、天气）都在浏览器里跑，进度存在浏览器的 `localStorage`。
所以"部署"在这里的含义只是"把这些静态文件放到一个能被浏览器通过 HTTP 取到的地方"。

| 通道 | 用到 Node 吗 | 改代码就生效吗 | 适合 |
|---|---|---|---|
| ① 开发服务器 | 需要 | 会热更新，改完刷新即可 | 读代码、改代码 |
| ② 构建 + 预览 | 需要 | 需要重新构建 | 确认"发布版本能跑" |
| ③ GitHub Pages | 只需要一次 push | 需要 push | 让别人也能玩（仓库自带 workflow，见 4.4） |
| ④ 静态服务器 | 不需要 | 需要重新构建 | 只想有个本地地址能玩 |

---

## 2. 前置条件

| 条件 | 说明 |
|---|---|
| Node.js + npm | 用于安装依赖和构建。CI 用的是 Node 22；本文实测环境是 Node v25.9.0 / npm 11.12.1 |
| 支持 WebGPU 的浏览器 | 较新的 Chrome、Edge 或 Safari |
| 一块像样的 GPU | 目标是 Apple M5 Pro 上 2560×1267 跑 60 fps；慢的机器由动态分辨率兜底 |
| **不需要** Python、数据库、后端服务 | 整个游戏是纯静态的 |

一条容易被忽略的硬约束：**WebGPU 只在"安全上下文"里可用**，也就是 `https://` 或 `localhost` / `127.0.0.1`。
这决定了第 4.1 节里"局域网访问"为什么不直接可行。

---

## 3. 本地启动

### 3.1 最快路径

```sh
npm ci          # 按 package-lock.json 精确安装（也可用 npm install）
npm run dev     # 启动开发服务器
```

实测（本机，首次安装）：

```text
npm ci        ->  added 20 packages in 18s
npm run dev   ->  VITE v8.3.0  ready in 101 ms
                  ➜  Local:   http://127.0.0.1:5189/
curl 检查      ->  HTTP 200
```

浏览器打开 **http://127.0.0.1:5189/** 即可。第一次进游戏会先看到加载画面（美术图 + 进度条 + 操作提示），
这一段是真实工作：它要建 WebGPU 设备、生成地形、模拟海面，并**编译几百个着色器**——首次可能一两分钟，之后的访问会快很多，因为浏览器缓存了编译结果。

**端口说明（容易困惑的一点）**：`vite.config.js` 里写的是 `server.port: 5188`，但 `package.json` 的 `dev` 脚本带了 `--port 5189`，命令行优先，所以实际端口是 **5189**。
另外该配置开了 `strictPort: true`：端口被占用时开发服务器会直接报错退出，而不是自动换一个端口。

开发服务器是前台进程，在跑它的终端里按 `Ctrl+C` 就能停；后台进程、`preview` 与 Python 服务的停止方式见 4.2。

### 3.2 玩之前先知道的三件事

- **指针锁定**：点击画面后鼠标被捕获，按 `Esc` 释放。
- **按 `H`** 打开设置面板（时刻、海况、云、雾、后处理、渲染缩放），调完立刻能看到效果——这是理解渲染系统最快的入口。
- **`F1`** 列出全部键位。

### 3.3 想看"发布版本"长什么样

```sh
npm run build      # 产物写入 dist/
npm run preview    # 用 dist/ 起一个本地静态服务
```

实测：

```text
npm run build     ->  ✓ 214 modules transformed  ✓ built in 620ms
npm run preview   ->  ➜  Local:   http://127.0.0.1:4173/
curl 检查          ->  HTTP 200
```

`preview` 用的是 Vite 默认端口 4173（`package.json` 里没有为它指定端口）。要换端口：

```sh
npm run preview -- --port 5189
```

### 3.4 构建产物里有什么

实测本次构建产出的主要内容：

| 产物 | 大小 | gzip | 说明 |
|---|---|---|---|
| `dist/index.html` | 3.80 kB | 1.67 kB | 入口页（含加载画面） |
| `dist/assets/index-*.js` | 1800.79 kB | 619.84 kB | 引擎 + 游戏的主包 |
| `dist/assets/index-*.css` | 45.86 kB | 9.10 kB | 界面样式 |
| `dist/assets/Frame-*.js`、`GPU-*.js`、`Bench-*.js` | 5–67 kB | — | 按需拆出的分块 |
| `dist/audio/`、`dist/models/`、`dist/clouds/`、`dist/textures/`、`dist/ui/` | 合计约 60 MB | — | 原样拷贝的静态资源（录音、模型、贴图） |

两个构建输出值得注意：

- 会打印一条 `[INEFFECTIVE_DYNAMIC_IMPORT]` 警告（`src/materials/LocalLights.js` 既被动态导入又被静态导入）。它是提示"这个动态导入不会真的拆包"，**不影响产物可用性**。
- 主包接近 2 MB 是这类项目的常态（引擎 + 全部游戏代码在同一个模块图里）。项目通过 `chunkSizeWarningLimit: 4000` 主动放宽了体积警告阈值，所以不会因此报错。

---

## 4. 部署

### 4.1 最简部署：把 `dist/` 用 HTTP 提供出去

`npm run build` 之后，`dist/` 本身就是一个可以部署的完整站点。两种等价做法：

```sh
# 有 Node：用 Vite 自带的静态服务
npm run preview                        # http://127.0.0.1:4173/

# 没有 Node 也行：用 Python 起一个静态服务
python3 -m http.server 8000 -d dist    # http://127.0.0.1:8000/
```

两者只是实现不同，产物和效果一样：把 `dist/` 通过 HTTP 暴露给浏览器。

**关于局域网访问**：`python3 -m http.server` 默认监听所有网卡，同一局域网的其它设备可以用 `http://<本机 IP>:8000/` 打开——
但**WebGPU 需要安全上下文，而这个地址不是 https**，浏览器会拒绝提供 WebGPU，游戏起不来。
要跨设备玩，请给服务配一层 HTTPS（反向代理 + 自签证书），或使用 4.3 的托管服务。

### 4.2 停止本地服务

上面几种启动方式都是前台进程，停止方法一致：在运行它的那个终端里按 `Ctrl+C`。

| 启动命令 | 端口 | 停止 |
|---|---|---|
| `npm run dev` | 5189 | 终端里 `Ctrl+C` |
| `npm run preview` | 4173 | 终端里 `Ctrl+C` |
| `python3 -m http.server 8000 -d dist` | 8000 | 终端里 `Ctrl+C` |

三种常见卡住的情况：

```sh
# ① 终端已关掉、或用 nohup / & 在后台启动：Ctrl+C 对它无效，要按进程号结束
pgrep -fl "vite"            # 列出 Vite 的服务进程（dev 与 preview 都匹配）
pgrep -fl "http.server"     # Python 的静态服务
kill <PID>                  # 换成上面查到的进程号

# ② kill 之后端口仍在监听：先确认进程号没搞错，再用一次强杀
kill -9 <PID>
```

确认端口真的空了（两种都行）：

```sh
curl -s -o /dev/null -w "%{http_code}\n" --max-time 3 http://127.0.0.1:4173/   # 000 = 连不上，已停
lsof -nP -iTCP:4173 -sTCP:LISTEN                                                # 无输出 = 已停
```

几个要点：

- 停止本地服务**不影响**已部署的线上站点，两者互不相干。
- `Ctrl+C` 只作用于当前终端的前台进程；在另一个窗口启动的服务要在那个窗口停，或按 PID 结束。
- 关掉终端窗口一般会带走它的子进程，但用 `nohup` / `&` 起的不会，必须按 PID 处理。
- 端口被占用时 `npm run dev` 会直接报错（`strictPort`），这通常意味着上一次的服务还在跑：先按上面的方式停掉它。

### 4.3 部署到任意静态托管

因为产物是纯静态、且使用相对路径（`vite.config.js` 的 `base: './'`），`dist/` 整个目录可以直接交给任何静态托管：

```text
Netlify / Vercel / Cloudflare Pages   构建命令 npm run build，发布目录 dist
对象存储（S3 / OSS / COS）+ CDN        上传 dist/ 内容，开启静态网站托管
自有 Nginx / Caddy                    把 dist/ 作为站点根目录或某个子路径
```

两个必须满足的条件：

1. **通过 HTTP(S) 访问**，不要用 `file://` 直接打开 `dist/index.html`——ES 模块和资源加载在 `file://` 下会被浏览器拦截，结果是白屏。
2. **提供 HTTPS**（或限制在 `localhost`）——见第 2 节的"安全上下文"约束。

### 4.4 仓库自带的 GitHub Pages workflow

仓库里有 `.github/workflows/deploy.yml`，想发到公网时链路是现成的：

```text
触发：push 到 main，或在 Actions 页面手动触发（workflow_dispatch）
   │
   ├─ build job:  checkout -> setup-node 22（带 npm 缓存）-> npm ci -> npm run build
   │              -> configure-pages -> upload-pages-artifact( path: dist )
   │
   └─ deploy job: 需要 build 成功 -> environment: github-pages -> actions/deploy-pages
```

产物使用相对路径，所以放在 `https://<用户名>.github.io/<仓库名>/` 这样的子路径下也能正确加载。

**两个必要条件**（都是仓库设置，`git push` 无法代替）：

1. `Settings -> Pages -> Build and deployment -> Source` 必须是 **GitHub Actions**。
   未启用 Pages 时 `configure-pages` 会直接失败，后面的 `upload-pages-artifact` 与 `deploy` 全部 skipped。
2. Source 不能停在 **Deploy from a branch**：那会让 GitHub 用 `jekyll-build-pages` 直接发布仓库根目录，
   站点上是**源码而不是构建产物**，症状很容易误判：

```text
/                  -> 200  但 index.html 与仓库根目录的同一份文件逐字节一致（没经过 vite build）
/src/main.js       -> 200  源码在对外服务
/package.json      -> 200  整个仓库根被当作站点
/assets/index-*.js -> 404  构建产物根本不在站点上
/models/... /audio/... /clouds/... /ui/...  -> 404
```

模型和音频 404 的原因不是路径写错：代码里资源 URL 走 `import.meta.env.BASE_URL`（见 `src/world/marine/Whale.js`、`src/audio/SoundScape.js`），
而 `import.meta.env` 是构建工具注入的，源码模式下它不存在，于是 fallback 到 `'/'`，请求落到了域名根而不是仓库子路径。
结论：页面能加载 HTML 和 JS，但游戏起不来。

设置正确后，到 Actions 页面 Run workflow（或再 push 一次）即可完成部署。

**怎么自查站点上的是哪一版**：

```text
GET /assets/index-*.js   200 = 构建产物（正确）
GET /src/main.js         404 = 构建产物（正确）；200 = 正在发布源码（Source 选错了）
```

#### 下线已发布的站点

停止对外发布和停止自动部署是两件事：

```text
只停自动部署：  删除或改名 .github/workflows/deploy.yml
                -> 以后 push 不再部署，但线上还是最后一次部署的内容
下线当前站点：  Settings -> Pages -> 在 "Your site is live at ..." 右侧点 “...” -> Unpublish site
                -> 当前部署被删除，站点立刻变成 404（设置与仓库内容不受影响，可随时重新发布）
```

如果连 Pages 配置本身也要清掉（`Unpublish site` 之后 `GET /repos/{owner}/{repo}/pages` 仍会返回配置），
需要带认证调用 REST API：`DELETE /repos/{owner}/{repo}/pages`。

---

## 5. 验证部署是否成功

按这个顺序看，能快速区分"没部署好"和"部署好了但浏览器不支持"：

```text
① 打开地址能看到加载画面（美术图 + 进度条）        -> 静态文件已经在被正确提供
② 进度条走到 100%，加载画面淡出、出现 fps 计数器   -> WebGPU 正常、着色器编译完成
③ 点击画面后能转视角、按 H 能开设置面板            -> 游戏真的在跑
```

出问题时，用根目录 `README.md` 里的 URL 参数做最小化自检，比逐项排查快：

```text
?noAudio     关掉声音（排除音频解码问题）
?noClouds    跳过体积云（最重的一项）
?noHaze      跳过雾与光轴
?noCaustics  跳过焦散
?noVeg       跳过植被
?noSim       跳过浅水模拟
?fly         直接进自由相机
?bench       跑帧时间基准（不进玩法循环）
```

---

## 6. 排查表

| 症状 | 最可能的原因 | 处理 |
|---|---|---|
| 页面白屏，控制台有模块/资源加载错误 | 用 `file://` 打开了 `index.html` | 用第 4.1 或 4.3 节的静态服务方式提供 |
| 控制台报 `WebGPU is not available in this browser.` | 浏览器太旧，或页面不在安全上下文（http 非 localhost） | 换新版 Chrome/Edge/Safari；改用 https 或 localhost |
| 报 `No WebGPU adapter found.` | 浏览器支持 WebGPU 但拿不到适配器（硬件加速被关、驱动问题、虚拟机） | 打开 `chrome://gpu` 看 WebGPU 状态；开启浏览器硬件加速；换机器 |
| 加载画面停住不动 | 首次着色器编译（可能一两分钟）；或设备丢失 | 等；看控制台是否有 `WebGPU device lost` 或 `uncapturederror` 输出 |
| 进游戏后帧率低 | GPU 压力大 | 按 `H` 降低渲染缩放/关云关雾；用 `?noClouds`、`?noVeg` 等逐项定位 |
| `npm run dev` 直接报端口占用 | 该配置开了 `strictPort` | 先关掉占用的进程，或用 `npm run dev -- --port 5190` |
| 构建时的分块体积 / 动态导入警告 | Vite 的提示信息 | 无害，见 3.4；不需要处理 |
| `npm test` 里 GPU 冒烟测试失败 | 无头 WebGPU 初始化失败（依赖未装全） | 先 `npm ci`；该测试依赖 `webgpu` 包提供的无头实现 |

---

## 7. 实测记录（写文档时的环境与结果）

留一份可对照的基线：换机器或升级依赖后，结果差异过大时，说明环境变了。

```text
环境      macOS, Node v25.9.0, npm 11.12.1
npm ci    20 packages in 18s
npm run build
          vite v8.3.0, 214 modules transformed, built in 620ms
          dist/ 约 60 MB（含 audio/ models/ clouds/ textures/ ui/）
npm run dev      ready in 101ms -> http://127.0.0.1:5189/   (HTTP 200)
npm run preview  -> http://127.0.0.1:4173/                    (HTTP 200)
npm test         2.5s，全部通过
                 game-logic: "all passed"
                 engine-smoke: stats { draws: 12, triangles: 7386, pipelines: 8 }

GitHub Actions（push 到 main 触发）
                 build job: npm ci 与 npm run build 均成功
                 configure-pages: 在仓库未启用 Pages 时失败；Source 选成分支部署时会发布源码而不是构建产物（见 4.4）
```

`npm test` 的两条命令对应两种验证：玩法逻辑不需要 GPU（可以在任何机器、任何 CI 上跑），引擎冒烟需要无头 WebGPU（用 `webgpu` 包提供的 Dawn 实现，见 `test/headless.mjs`）。

---

## 8. 一句话总结

```text
本地开发：      npm ci && npm run dev              -> http://127.0.0.1:5189
本地玩发布版：  npm run build && npm run preview   -> http://127.0.0.1:4173
最简部署：      python3 -m http.server 8000 -d dist -> http://127.0.0.1:8000
发布公网：      git push origin main              -> 仓库自带的 Pages workflow（Source 需为 GitHub Actions，见 4.4）

只有两条硬约束：必须是 HTTP(S)（不是 file://），而且必须是安全上下文（https 或 localhost）。
```
