---
confidence: 0.51
created: '2026-04-04'
domain: unreal-engine
path: unreal-engine/rendering
related: []
status: draft
tags:
- C++
- FAutoRegister
- ISceneViewExtension
- SceneViewExtension
- UE5
- scene-proxy
- scene-view
- viewport
- 后处理
- 渲染管线
title: ISceneViewExtension 用法指南
updated: '2026-04-04'
---

# ISceneViewExtension 用法指南

# ISceneViewExtension 用法指南

## 概述

`ISceneViewExtension` 是 Unreal Engine 提供的一套接口，允许开发者在**不修改引擎源码**的情况下，向渲染管线的后处理特定阶段插入自定义 Pass。

### 核心限制
- **仅支持延迟渲染路径（Deferred Shading）**，Mobile 平台不支持
- 可在以下 4 个后处理 Pass 之后插入回调：
  1. `MotionBlur`（运动模糊）
  2. `Tonemap`（色调映射）
  3. `FXAA`（快速近似抗锯齿）
  4. `VisualizeDepthOfField`（景深可视化）

---

## 完整使用步骤

### Step 1：创建扩展类

继承 `FSceneViewExtensionBase`，**构造函数第一个参数必须是 `FAutoRegister`**：
