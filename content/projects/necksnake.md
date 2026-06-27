---
title: "NeckSnake —— 用头部动作控制贪吃蛇"
date: 2025-07-01
weight: 5
draft: false
tags:
  - MediaPipe
  - TypeScript
  - Canvas
  - 姿态检测
  - 浏览器游戏
categories:
  - 创意项目
summary: "基于 MediaPipe 实时姿态检测的贪吃蛇游戏，用头部转动方向替代键盘操控，纯前端实现，零框架依赖。"
ShowToc: true
---

## 项目简介

NeckSnake 是一个**用头部动作操控的贪吃蛇游戏**。通过摄像头实时检测玩家头部的转动方向（上/下/左/右），将其转化为游戏的方向指令。整个过程在浏览器内完成，无需后端服务。

项目地址：[GitHub - NeckSnake](https://github.com/gainbandwidth/NeckSnake)

## 技术栈

| 层次 | 技术 |
|------|------|
| 语言 | TypeScript（strict 模式，ES2022） |
| 构建工具 | Vite 6.2+ |
| 渲染 | HTML5 Canvas 2D |
| 姿态检测 | MediaPipe Tasks Vision（PoseLandmarker） |
| UI | 原生 DOM（无框架） |

整个项目只有**一个运行时依赖**（`@mediapipe/tasks-vision`），源码约 1,370 行 TypeScript + 217 行 CSS。

## 核心架构

### 头部动作检测（HeadMotionController）

这是项目的核心模块，实现了一套完整的实时手势识别流水线：

1. **摄像头采集**：通过 `getUserMedia` 以 960x540@30fps 开启摄像头
2. **姿态推理**：MediaPipe PoseLandmarker 实时检测鼻子位置
3. **基准校准**：12 帧采样（约 1.8 秒）建立中立姿态基准
4. **方向判定**：追踪鼻子相对于肩膀中心的偏移量，判定转动方向

### 抗抖动策略

为了让操控体验流畅，设计了多层抗抖动机制：

- **EMA 平滑**（alpha=0.45）对偏移量做指数移动平均
- **轴主导逻辑**（比率 1.12）防止对角线串扰
- **非对称阈值**：「向上」比「向下」更严格，减少误触发
- **动态确认帧**：高置信度（≥72%）1 帧触发，低置信度需 2 帧
- **漂移补偿**：中立区基准自适应调整，吸收长时间坐姿变化

### 贪吃蛇引擎（SnakeGame）

经典贪吃蛇逻辑，支持运行时调节参数：

- 网格尺寸可配（12-60 列，10-40 行）
- 穿墙/碰壁模式切换
- 速度实时调节（4-14 格/秒），无需重启
- HiDPI 感知的 Canvas 渲染
- localStorage 持久化最高分

## 设计原则

项目遵循三个设计原则：

**模块解耦**：动作检测只发射方向事件，游戏引擎只消费事件。两者完全独立，可以轻松替换输入源（比如手机 IMU）而不改动游戏代码。

**轻量优先**：不使用 UI 框架和游戏引擎，原生 DOM + Canvas 保持包体小巧，调试方便。

**参数化设计**：所有游戏参数集中在 `SnakeGameOptions`，通过 `updateOptions()` 智能判断是否需要重启。

## 运行方式

```bash
npm install
npm run dev
```

浏览器打开 `http://localhost:5173`，打开摄像头 → 校准中立姿态 → 开始游戏。
