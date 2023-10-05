---
title: threejs上手
date: '2023-10-05'
tags: ['AR开发']
draft: false
summary: 3D模型
---

# 简介

- [threejs文档](https://threejs.org/docs/index.html)
- [threejs示例](https://threejs.org/examples/)

对于webar中，3D模型资源是很重要的一个组成部分。可以通过建模或者使用threejs搭建一些简单的3D场景

> 一个人的能力始终是有边界的，所以对于复杂专业的建模软件的学习，暂时不进行展开，要么使用现成的3D模型进行学习使用，或者对于一些简单的3D场景，学习使用threejs来完成

## 安装

> 我这里采用vite构建工具，并且开发框架使用svelte对于一些小型的页面我比较喜欢使用svelte

```bash
npm create vite@latest 3d-assets -- --template svelte

cd 3d-assets

pnpm install --save three

pnpm run dev
```

然后在入口js中添加核心资源, 注意由于这些是客户端资源，所以必须在onMount挂载后初始化

```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { onMount } from "svelte";
```

# 开发示例

## 主要对象

- 场景
- 相机
- 形状
- 材质

---

```javascript
import * as THREE from 'three';

let threescene;

  onMount(() => {
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      1000
    );
    const renderer = new THREE.WebGLRenderer();
    renderer.setSize(window.innerWidth, window.innerHeight);
    threescene.appendChild(renderer.domElement);
  });
```

## 一个旋转的立方体

`创建形状以及材质`

```javascript
const geometry = new THREE.BoxGeometry( 1, 1, 1 );
const material = new THREE.MeshBasicMaterial( { color: 0x00ff00 } );
const cube = new THREE.Mesh( geometry, material );
scene.add( cube );

camera.position.z = 5;
```

`渲染出结果`

当我们在animate的回调中不断改变cube这个正方体的rotation值，就会看上去像旋转一样

```javascript
function animate() {
	requestAnimationFrame( animate );

    cube.rotation.x += 0.01;
    cube.rotation.y += 0.01;

	renderer.render( scene, camera );
}
animate();
```

<img src="/images/blogs/ar开发/threejs-cube.jpg" />

## 线

```javascript
const material = new THREE.LineBasicMaterial({ color: 0x0000ff });
const points = [];
points.push(new THREE.Vector3(-10, 0, 0));
points.push(new THREE.Vector3(0, 10, 0));
points.push(new THREE.Vector3(10, 0, 0));
const geometry = new THREE.BufferGeometry().setFromPoints(points);
const line = new THREE.Line(geometry, material);
```

## 3D文字

> 需要为three安装FontLoader以及TextGeometry两个扩展
> 如果是使用npm安装的three那么就直接有了，不需要额外操作

对于字体来说，需要将ttf格式转换成json文件才能导入使用, [在线转化](https://gero3.github.io/facetype.js/)

<img src="/images/blogs/ar开发/threejs-text.jpg" />

```javascript
import * as THREE from 'three';
import { FontLoader } from "three/examples/jsm/loaders/FontLoader.js";
import { TextGeometry } from "three/examples/jsm/geometries/TextGeometry.js";

function loadFont() {
    const loader = new FontLoader();
    loader.load( 'fonts/' + fontName + '_' + fontWeight + '.typeface.json', function ( response ) {

        font = response;

        refreshText();

    } );

}

function createText() {
    textGeo = new TextGeometry( text, {
        font: font,
        size: size,
        height: height,
        curveSegments: curveSegments,
        bevelThickness: bevelThickness,
        bevelSize: bevelSize,
        bevelEnabled: bevelEnabled
    } );

    textGeo.computeBoundingBox();

    const centerOffset = - 0.5 * ( textGeo.boundingBox.max.x - textGeo.boundingBox.min.x );

    textMesh1 = new THREE.Mesh( textGeo, materials );

    textMesh1.position.x = centerOffset;
    textMesh1.position.y = hover;
    textMesh1.position.z = 0;

    textMesh1.rotation.x = 0;
    textMesh1.rotation.y = Math.PI * 2;

    group.add( textMesh1 );
}

function refreshText() {
    group.remove( textMesh1 );
    if ( mirror ) group.remove( textMesh2 );

    if ( ! text ) return;

    createText();
}
```

## GUI

```javascript
import { GUI } from "three/examples/jsm/libs/lil-gui.module.min.js";

const params = {
    changeColor: function () {
        pointLight.color.setHSL(Math.random(), 1, 0.5);
    },
    changeFont: function () {
        fontIndex++;

        fontName =
            reverseFontMap[fontIndex % reverseFontMap.length];

        loadFont();
    },
    changeWeight: function () {
        if (fontWeight === "bold") {
            fontWeight = "regular";
        } else {
            fontWeight = "bold";
        }

        loadFont();
    },
    changeBevel: function () {
        bevelEnabled = !bevelEnabled;

        refreshText();
    },
};

//

const gui = new GUI();

gui.add(params, "changeColor").name("change color");
// gui.add(params, "changeFont").name("change font");
// gui.add(params, "changeWeight").name("change weight");
gui.add(params, "changeBevel").name("change bevel");
gui.open();
```


## 3D模型

GLTF（Graphics Language Transmission Format）是一种标准的3D模型文件格式，它以JSON的形式存储3D模型信息，例如模型的层次结构、材质、动画、纹理等。

模型中依赖的静态资源，比如图片，可以通过外部URI的方式来引入，也可以转成base64直接插入在GLTF文件中。
它包含两种形式的后缀，分别是.gltf（JSON/ASCII）和.glb（Binary）。.gltf是以JSON的形式存储信息。.glb则是.gltf的扩展格式，它以二进制的形式存储信息，因此导出的模型体积也更小一些。如果我们不需要通过JSON对.gltf模型进行直接修改，建议使用.glb模型，它更小、加载更快。

```javascript
import GLTFLoader from 'GLTFLoader'
const loader = new GLTFLoader()
loader.load('path/to/gallery.glb',
  gltf => {
    scene.add(gltf.scene) // 添加到场景中
  } 
)
```