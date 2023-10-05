---
title: AR开发示例程序
date: '2023-10-05'
tags: ['AR开发']
draft: false
summary: 应用示例
---

# 简介

AR开发分为专用设备对应的ARkit，以及webar，目前专用设备的ar开发没有设备无法接触，所以先从webar开始入手

webar常见的ar库包括

- ar.js
- ARToolKit
- JSARToolKit: 使用WebGL&Three.js实现3D模型展示
- argonjs
- awe.js: 提供了增强现实标记，基于位置和leap motion传感器AR，使用WebRTC、WebGL和getUserMedia设备API在浏览器中实现AR功能
- three.ar.js
- tracking.js

# MindAR

- [文档链接](https://hiukim.github.io/mind-ar-js-doc/)
- [应用示例](https://github.com/hiukim/mind-ar-js/tree/master/examples)

> 一个基于ar.js以及three.js封装的开发库，能够方便的实现物品跟踪，人脸识别等ar功能
> 从MindAR v.1.2.0开始，threejs变成了外部依赖，需要自己安装，但是支持的最低版本是v137

## 安装

`图像跟踪`

```html
<script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/mind-ar@1.2.2/dist/mindar-image-aframe.prod.js"></script>

<!-- 引入threejs -->
<script async src="https://unpkg.com/es-module-shims@1.7.3/dist/es-module-shims.js"></script>
<script type="importmap">
{
  "imports": {
    "three": "https://unpkg.com/three@0.147.0/build/three.module.js",
    "three/addons/": "https://unpkg.com/three@0.147.0/examples/jsm/",
    "mindar-image-three":"https://cdn.jsdelivr.net/npm/mind-ar@1.2.2/dist/mindar-image-three.prod.js"
  }
}
</script>

<!-- 使用 -->
<script type="module">
  import * as THREE from 'three';
  import { MindARThree } from 'mindar-image-three';
</script>
```

`人脸跟踪`

```html
<script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/mind-ar@1.2.2/dist/mindar-face-aframe.prod.js"></script>

<!-- 引入threejs -->
<script async src="https://unpkg.com/es-module-shims@1.3.6/dist/es-module-shims.js"></script>
<script type="importmap">
{
  "imports": {
    "three": "https://unpkg.com/three@0.147.0/build/three.module.js",
    "three/addons/": "https://unpkg.com/three@0.147.0/examples/jsm/",
    "mindar-face-three":"https://cdn.jsdelivr.net/npm/mind-ar@1.2.2/dist/mindar-face-three.prod.js"
  }
}
</script>

<!-- 使用 -->
<script type="module">
  import * as THREE from 'three';
  import { MindARThree } from 'mindar-face-three';
</script>
```

---

或者使用npm包管理引入

```bash
> npm i mind-ar --save
> npm i aframe --save
```

`图像跟踪`

```javascript
import 'aframe';
import 'mind-ar/dist/mindar-image-aframe.prod.js';
import {MindARThree} from 'mind-ar/dist/mindar-image-three.prod.js';
```

`人脸跟踪`

```javascript
import 'aframe';
import 'mind-ar/dist/mindar-face-aframe.prod.js';
import {MindARThree} from 'mind-ar/dist/mindar-face-three.prod.js';
```

# 图像跟踪开发

使用一个静态html文件开发一个AR应用，包括启动设备相机，定位图像目标，然后展示增强对象在这个目标上。

> 增强对象可以是3D模型、图片、视频、CSS、音频等

对于需要检测的图片目标，需要先经过[数据转换](https://hiukim.github.io/mind-ar-js-doc/quick-start/compile)，上传图片后开始编译形成mind文件。


## AFRAME扩展

用于轻松构建3D场景，包括下面的这些html标签

- a-scene
- a-entity：检测目标
- a-plane：一些简单的图形
- a-camera
- a-assets
- a-asset-item
- a-gltf-model

## 3D资源

AFRAME几乎支持所有标准的3D格式，包括gltf格式

## 微调

采用某种多帧的滚动平均值来定位目标，使得内容更加稳定

`OneEuroFilter`

> 降低可以减少抖动
> 增加可以减少延迟

- filterMinCF：截止频率，默认值为0.001
- filterBeta：速度系数，默认值为1000

---

- warmupTolerance: 在指定的连续帧中检测到目标值就视为成功，避免误报，默认值为5
- missTolerance: 在指定的连续帧中未检测到目标值视为成功，默认值为5

## 示例

> 效果示意图

<img src="/images/blogs/ar开发/图像跟踪.jpg" />

```html
<html>
  <head>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/mind-ar@1.2.2/dist/mindar-image-aframe.prod.js"></script>
  </head>
  <body>
    <a-scene mindar-image="imageTargetSrc: ./targets.mind; showStats: true;" color-space="sRGB" renderer="colorManagement: true, physicallyCorrectLights" vr-mode-ui="enabled: false" device-orientation-permission-ui="enabled: false">
      <a-assets>
        <img id="card" src="https://cdn.jsdelivr.net/gh/hiukim/mind-ar-js@1.2.2/examples/image-tracking/assets/card-example/card.png" />
        <a-asset-item id="avatarModel" src="https://cdn.jsdelivr.net/gh/hiukim/mind-ar-js@1.2.2/examples/image-tracking/assets/card-example/softmind/scene.gltf"></a-asset-item>
      </a-assets>


      <a-camera position="0 0 0" look-controls="enabled: false"></a-camera>


      <a-entity mindar-image-target="targetIndex: 0">
        <a-plane src="#card" position="0 0 0" height="0.552" width="1" rotation="0 0 0"></a-plane>
        <a-gltf-model rotation="0 0 0 " position="0 0 0.1" scale="0.005 0.005 0.005" src="#avatarModel" animation="property: position; to: 0 0.1 0.1; dur: 1000; easing: easeInOutQuad; loop: true; dir: alternate">
      </a-entity>
    </a-scene>
  </body>
</html>
```

# 人脸跟踪开发

## 示例

> 效果示意图

<img src="/images/blogs/ar开发/人脸跟踪.png" />


```html

<html>
  <head>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/mind-ar@1.2.2/dist/mindar-face-aframe.prod.js"></script>
  </head>
  <body>
    <a-scene mindar-face embedded color-space="sRGB" renderer="colorManagement: true, physicallyCorrectLights" vr-mode-ui="enabled: false" device-orientation-permission-ui="enabled: false">
      <a-camera active="false" position="0 0 0"></a-camera>
      <a-entity mindar-face-target="anchorIndex: 1">
	<a-sphere color="green" radius="0.1"></a-sphere>
      </a-entity>
    </a-scene>
  </body>
</html>
```