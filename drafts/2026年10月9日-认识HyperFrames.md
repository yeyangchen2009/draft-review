---
title: 写一段 HTML，渲染出一条视频：认识 HyperFrames
date: 2026-10-09
tags: [博客]
status: 草稿
---

叶扬最近在琢磨一件事——让 AI 帮忙做视频。

这话听着不新鲜，剪映、Pr 谁都能剪。可真要让 AI 动手，两条老路都不顺：要么在剪辑软件里拖轨道、对关键帧，这是给人手设计的活儿，AI 没有鼠标没有眼睛，根本下不去手；要么开个录屏软件让 AI 自己操作，可录屏这东西每次都不一样——鼠标快一点慢一点、系统卡一下、弹个通知，出来的画面随机、会抖，还没法精确复现。

一边是 AI 碰不了，一边是能自动化却不靠谱。中间这块空白，就是这一篇的主角要站的位置。

## 一、它叫 HyperFrames：给它网页，还你 MP4

这个工具叫 **HyperFrames**。一句话讲清楚它干什么——你写一个网页，它还给你一个 MP4。

![HyperFrames 的名字和三句口号](https://raw.githubusercontent.com/yeyangchen2009/draft-review/main/assets/hf01/hf01-slogan.png)

*视频第一幕：名字登场，下面是三行口号*

它官网挂着三行口号，值得逐句读：

- **Write HTML**——你要写的就是普通 HTML，网页那一套：div、CSS、字体、动画；
- **Render video**——产出是渲染出来的视频，不是录屏、不是动图；
- **Built for agents**——从设计之初就是给 AI agent 用的，这句话最关键，后面专门展开。

几个可以核实的背景：HyperFrames 现在挂在数字人公司 **HeyGen** 名下，命令行里登录、云端渲染都走 HeyGen；项目本身是 **Apache 2.0** 开源的；叶扬这一系列教程把版本**钉死在 0.8.107**——命令统一用 `npx --yes hyperframes@0.8.107` 调用，不追最新版，免得哪天工具悄悄改了行为，渲染出来的成片对不上。

## 二、最要紧的认知：它是「渲染」，不是「录屏」

理解 HyperFrames，最重要的一件事就是分清「渲染」和「录屏」。

![渲染与录屏的对比](https://raw.githubusercontent.com/yeyangchen2009/draft-review/main/assets/hf01/hf01-render-vs-record.png)

*红卡是录屏：你操作一遍，它在旁边实时拍；蓝卡是渲染：像摆拍，每一帧精确算出来再拼*

录屏是什么？你在前面真实操作一遍，软件在旁边拿摄像机实时拍下来。拍成什么样，取决于当时那一遍操作——是「抓拍」。

渲染完全反过来，更像**摆拍**：你先把画面、动作、时间轴全安排好，告诉它「第 200 秒应该长这样」，它就把那一帧**精确地算出来**，然后一秒 30 帧、帧帧算好，拼成片。没有「现场」，也就没有现场的意外。

这一字之差，是后面一切好处的根。

## 三、为什么「渲染」对 AI 这么关键：确定性

渲染带来的东西叫**确定性**——同样的输入，今天算和明天算、这台机器算和那台机器算，每一帧都能逐像素对得上。

为了做到这点，HyperFrames 渲染时甚至干脆**关掉 GPU、改用软件渲染**（参数 `--no-browser-gpu`，底层是 SwiftShader）。因为不同显卡对同一个 CSS 可能渲染出细微差别，软件渲染把这点差异也抹平了。于是「跳到任意一帧」成为可能：想知道第 200 秒的画面，直接跳过去算那一帧就行，不用从头播放。时间轴上每一帧都是可寻址、可截图、可核对的。

这对人是方便，对 AI 就是决定性的。

设想 AI 改稿的过程：如果成片靠录屏，AI 改完想确认效果，只能重新录、再从头看一遍，而每次录的还不一样——它根本不知道画面里的变化是自己改出来的，还是手抖造成的，闭环直接断掉。换成渲染，路径就通了：

1. AI 修改 HTML 源文件；
2. 让它渲染几个关键时间点的帧（比如每一幕的中间帧）；
3. 自动核对这些帧对不对、对比度够不够、有没有叠压；
4. 错了就回到第 1 步继续改。

**改、渲、核**，三步形成闭环，全程不需要人盯着。这就是「Built for agents」真正的落点——不是一句营销话，是一套能让 AI 自己迭代的工程基础。

## 四、上手前先体检：Node、Chrome、FFmpeg 齐不齐

渲染要三样东西在背后撑着：**Node.js**（跑这个 CLI）、**Chrome**（无头浏览器，负责把网页画出来）、**FFmpeg**（把帧编码成视频）。自己一个个查版本太麻烦，HyperFrames 自带一个体检命令：

```bash
npx --yes hyperframes@0.8.107 doctor
```

![doctor 体检输出](https://raw.githubusercontent.com/yeyangchen2009/draft-review/main/assets/hf01/hf01-doctor.png)

*doctor 逐项打勾：核心三件套全绿，红叉都标着 optional*

叶扬这台是台很普通的笔记本——i7-8550U、15.9 GB 内存，不是什么高配工作站。体检结果可以看上图：

- **Chrome、FFmpeg、FFprobe** 全绿（FFmpeg 是 7.1.1），这三样是主线，齐了就能干活；
- 下面一串红叉——`whisper-cpp`（本地语音转写）、`Kokoro`（本地配音）、`MusicGen`（本地配乐）、`Docker`——每一个后面都标着 **optional**，全是可选项。不装它们，一点不影响「写 HTML、渲染视频」这条主线；
- Chrome 甚至不用自己装，首次用的时候它会自动下载一个专用的 headless shell。

这点其实挺破除迷思：做视频不等于要高配电脑。这台老笔记本照样把 237 秒的成片渲染完了，只是慢——满片 7050 帧，渲了约 20 分钟。慢，但确定跑得完。

## 五、一个 CLI、40 条命令，主流程就四步

HyperFrames 是个「全家桶」式的命令行——`--help` 拉出来整整 40 条子命令，分成起步、项目、工具、部署、AI 集成、账户、设置好几组。

![hyperframes --help 的命令全家桶](https://raw.githubusercontent.com/yeyangchen2009/draft-review/main/assets/hf01/hf01-help.png)

*命令全家桶的下半区：云端部署、AI 集成（转写/本地配音/抠像）、账户、设置*

上图那一片（云端渲染、Lambda 分布式、`transcribe` 转写、`tts` 本地配音、`remove-background` 抠像……）大多是进阶功能。真正要记住的主流程只有四步，四条命令：

| 命令 | 干什么 |
|---|---|
| `init` | 起脚手架，生成一个新项目的目录和文件 |
| `preview` | 起本地 studio，浏览器里边改边看 |
| `check` | 一道门：lint ＋ 运行时校验 ＋ 布局检查（JS 报错、缺资源、对比度） |
| `render` | 渲染成 MP4 或 WebM |

对应一条流水线：**`init` 起项目 → `preview` 边写边看 → `check` 把关 → `render` 出片**。这一篇只负责「认识」，下一篇就从 `init` 开始，看一条命令搭出来的骨架里到底有什么。

## 六、那它和 Remotion 有啥不一样

「用代码写视频」这件事，最有名的前辈是 **Remotion**。那 HyperFrames 跟它差在哪？

![HyperFrames 与 Remotion 的对比](https://raw.githubusercontent.com/yeyangchen2009/draft-review/main/assets/hf01/hf01-remotion.png)

*左卡 Remotion：用 React 写视频，得按 React 那套来；右卡 HyperFrames：纯 HTML、不绑框架*

差别一句话：**Remotion 用 React 写视频**，你得按 React 的方式组织组件、理解那套状态和渲染心智模型；**HyperFrames 是纯 HTML，不绑任何框架**，一个网页文件浏览器直接双击就能打开，门槛低得多。

至于前面说的「跳帧」能力为什么是 HyperFrames 的灵魂、它内部具体怎么实现——那是这个系列后面要专门展开的内容，这里先记住结论：它从底层就是为「能跳到任意一帧」设计的。

## 七、它到底卡在哪块空白

绕回开头那两条老路，HyperFrames 的位置就清楚了。

![三种做视频方式的定位](https://raw.githubusercontent.com/yeyangchen2009/draft-review/main/assets/hf01/hf01-three-ways.png)

*三象限：剪辑软件给人手拖、录屏能自动化但随机、HyperFrames 卡中间这块空白*

- **剪辑软件**：强大，但是给人手拖的，AI 下不了手；
- **录屏**：能自动化，可拍下的东西有随机性、会抖、不能复现；
- **HyperFrames**：卡中间这块空白——**既能全自动，结果又是确定的**。

做视频这件事，第一次有了像写代码一样的工作流：HTML 当源文件，纯文本、能进 git、能看版本差异；一条命令自动构建成片；产物还能逐帧复现。对天天和代码打交道的人来说，这种「踏实感」本身就很动人。

## 下一篇

认识完了，下一篇动手：跑一条 `init`，看它在三十秒内搭出一个能直接渲染的项目骨架，再逐行看看生成的文件各自管什么。

---

*本文是「HyperFrames 视频教程」系列第一集《认识 HyperFrames》的配套图文。*
