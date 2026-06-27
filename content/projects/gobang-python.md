---
title: "Gobang-Python —— AI 五子棋对战"
date: 2025-09-01
weight: 6
draft: false
tags:
  - Python
  - FastAPI
  - 博弈搜索
  - Alpha-Beta
  - AI
categories:
  - 创意项目
summary: "Python 重写的 AI 五子棋，后端 FastAPI + 经典博弈搜索（Negamax + Alpha-Beta 剪枝 + VCT/VCF），前端纯静态页面。"
ShowToc: true
---

## 项目简介

Gobang-Python 是一个**AI 驱动的五子棋对战游戏**。Python 后端实现完整的博弈搜索引擎，前端为纯静态页面，通过 REST API 与后端交互。项目是对 [lihongxun945/gobang](https://github.com/lihongxun945/gobang)（JavaScript 版）的 Python 重写，并在此基础上做了大量优化。

项目地址：[GitHub - gobang-python](https://github.com/gainbandwidth/gobang-python)

## 技术栈

| 层次 | 技术 |
|------|------|
| 前端 | HTML + CSS + 原生 JavaScript |
| 后端 API | Python 3 + FastAPI |
| HTTP 服务 | Uvicorn（ASGI） |
| 数据校验 | Pydantic v2 |
| AI 引擎 | 纯 Python 经典博弈搜索 |

## 系统架构

```
浏览器 (HTML/CSS/JS)
    │  fetch() POST /api/v1/game/*
    ▼
FastAPI (app.py)
    │  Pydantic 请求校验
    ▼
SessionManager (session.py)
    │  线程安全的会话管理
    ▼
Board (board.py) ←── Zobrist 哈希 + 置换表
    │  落子/悔棋/评估
    ▼
Evaluate (eval.py) ←── 棋型识别 (shape.py)
    │  候选着法生成 / 局面评估
    ▼
Negamax + Alpha-Beta + VCT/VCF (minmax.py)
```

## AI 引擎详解

AI 引擎是本项目的核心，完全基于经典搜索算法，不依赖神经网络：

### Negamax + Alpha-Beta 剪枝

采用 Negamax 统一极大极小搜索框架（通过取负简化 min/max），配合 Alpha-Beta 剪枝大幅减少搜索节点。

### 迭代加深

从深度 2 开始逐层加深搜索，确保在任意时间点都能返回一个可用着法。

### 棋型识别与评估

通过正则表达式和快速计数方法，在 4 个方向上识别连五、活四、冲四、活三、眠三等棋型，将局面评估转化为双方棋型价值之和。

### 战术搜索：VCT 与 VCF

- **VCT（连续威胁取胜）**：深层战术搜索，聚焦强制活三/冲四序列，检测必胜路径
- **VCF（连续冲四取胜）**：更深的搜索，仅限冲四着法

### 候选着法过滤

优先级剪枝：连五 > 活四 > 双四 > 四三 > 双三 > 冲四 > 活三 > …… 每个节点只探索前 N 个候选（默认 20）。

### Zobrist 哈希 + 置换表

使用 Zobrist 哈希为每个局面生成唯一键，配合 100 万容量的 FIFO 置换表避免重复评估。

## REST API

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/api/v1/healthz` | 健康检查 |
| POST | `/api/v1/game/start` | 创建新对局（可配棋盘大小、AI 先手、搜索深度） |
| POST | `/api/v1/game/move` | 玩家落子，AI 返回应对着法 |
| POST | `/api/v1/game/undo` | 悔棋（撤销玩家+AI各一步） |
| POST | `/api/v1/game/end` | 结束对局 |

## 前端界面

纯静态页面，FastAPI 直接托管：

- 15×15 棋盘，CSS Grid 渲染
- 操作按钮：开始、悔棋、认输
- 设置项：AI 先手切换、难度等级（搜索深度 2/4/6/8）
- AI 思考时显示加载遮罩

## 运行方式

一键启动，自动安装依赖：

```bash
python run.py
```

浏览器打开 `http://localhost:8000` 即可开始对弈。
