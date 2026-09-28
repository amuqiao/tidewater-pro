# Tidewater 笔记（notes/）

一句话：这里放**读 Tidewater 时用得上的讲解笔记与操作指南**，主要面向"写过 Python、没写过游戏或图形程序"的读者，不替代仓库根目录的 `README.md`。

## 这批笔记解决什么问题

根目录 `README.md` 回答的是"这个游戏是什么、怎么玩、怎么跑起来"。
它默认读者能看懂 `src/` 里的引擎、渲染、模拟代码——而这些代码大量使用游戏开发和 GPU 编程的行业惯例（每帧循环、着色器、渲染目标、命令编码器），对有 Python/后端背景的人是一道陡坡。

这批笔记补的是这道坡：

```text
README.md          -> 玩什么、怎么跑（面向玩家 / 使用者）
notes/（本目录）    -> 代码为什么这么写、数据怎么流动（面向第一次读代码的人）
docs/PORTING.md    -> 从 three.js 移植到自研 WebGPU 引擎的对照表（面向要改渲染代码的人）
```

## 阅读顺序

| 顺序 | 文档 | 读完你会知道 | 大约篇幅 |
|---|---|---|---|
| 1 | [tidewater-overview.md](tidewater-overview.md) | 项目分几层、一帧里依次发生什么、玩法是怎么组织的、测试和构建怎么跑 | 中篇 |
| 2 | [tidewater-rendering.md](tidewater-rendering.md) | WebGPU 的心智模型、场景是怎么画到屏幕上的、海面为什么用 FFT、世界和资产是怎么造出来的 | 中篇 |
| 3 | [tidewater-running-and-deploying.md](tidewater-running-and-deploying.md) | 怎么本地启动、构建、预览，怎么发布到公网，以及打不开/卡顿时的排查顺序 | 短篇 |

这三篇都假设读者只有 Python 经验：不预设 WebGL / three.js / 图形学背景，遇到行业术语会先给直觉解释，再给项目里的真实位置。

建议读法：**先读第一篇的第 1、2、4 节建立骨架，再读第二篇，最后回来读第一篇剩下的章节。** 先有"一帧的骨架"，后面所有细节才有地方挂。

只想先把游戏跑起来、暂时不读代码，直接跳到第三篇。

## 想要更深入时读什么

| 主题 | 去哪里 |
|---|---|
| 某个目录里有什么 | 根目录 `README.md` 的 Project layout 表 |
| three.js 概念 ↔ 本项目引擎概念的对照 | `docs/PORTING.md`（本仓库最密集的术语表） |
| 三个离线资产流水线（音频、道具、角色） | `tools/audio/README.md`、`tools/props/README.md`、`tools/characters/README.md` |
| 提交规范 | `AGENTS.md` |

## 不负责的内容

- 不复述 `README.md` 已有的玩法说明、控制键位和 URL 参数清单。
- 不是 API 参考：不逐个列出类和方法签名，需要时请直接读源码。
- 不描述尚未实现的能力或路线图。

## 维护规则

- 新增笔记时，文件名沿用"主题 + 对象"的英文小写加连字符风格（如 `tidewater-rendering.md`），并在上面的阅读顺序表里加一行。
- 笔记里的路径、符号和事实以仓库当前代码为准；改动相关代码后，如果笔记里的事实随之变化，请同步更新笔记。
- 相对链接优先（`tidewater-*.md`、`../README.md`），便于整个目录迁移。
