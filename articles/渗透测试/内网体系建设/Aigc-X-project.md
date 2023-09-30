## 理念(impact to real-world)

当今时代，人体三维建模需求量高，数据采集较为复杂，落地场景较为固定，如果能让AI来协助建模工程师，将显著降低企业人力成本，提质增效

AI的优越性在于智能化处理和分析大量数据、自动化执行任务、快速适应业务需求变化，利用单张图片进行数据模拟

将传统的原子化的模型生产方式重构，追求对需求的原子化

综合多种AI模型的高精度，高效率，低能耗的特点，协助建模工程师，快速建模

## Problems Sloved

解决了AI模型的本地部署与训练

颠覆了市场上先有需求后有供应的低速传统供应链

解决了前端实时加载解析obj文件的疑难

## technologies we used

html/css + javascript 实现前端UI设计，提供文件上传功能，通过three.js包实时预览obj格式模型

通过python flask实现web服务端的搭建，和**多个本地AI**模型的通信和管理

搭建基于python的本地AI模型，**实现核心功能：人体3D建模**

以下为开源项目的链接：

Removebg:https://www.remove.bg/zh
openpose:  https://github.com/CMU-Perceptual-Computing-Lab/openpose
PIFuHD:  https://github.com/facebookresearch/pifuhd

## Futuer plans

游戏厂商通过我们提供的API接口，即可完成基础模型的获取与个性化

建模工程师能够直接在已有模型的基础上进行进一步的精细化调整

玩家能够获取个性化的模型，亲自进入游戏世界

实现行业3D建模标准化，一次建模，多端使用并向前兼容

## 项目细节

### Step one  提供web页面，采集用户数据

部分代码演示

```html
<div class="grid">
				<div class="grid__item theme-3">
					<button class="action"><svg class="icon icon--rewind"><use xlink:href="#icon-rewind"></use></svg></button>
					<button class="particles-button" id="uploadButton">Action!</button>
					<input type="file" id="fileInput" style="display: none;">
				</div>
				<div class="grid__item theme-4">
					<button class="action"><svg class="icon icon--rewind"><use xlink:href="#icon-rewind"></use></svg></button>
					<button class="particles-button" onclick="redirectToPage()">View!</button>
				</div>
    ..........
```

服务端完全能够只通过单张图片的提交，完成对建模数据的采集，从而返回预览模型给用户

### Step  two  利用AI实现人物识别，提高建模精度

部分代码展示

```python
from removebg import RemoveBg
rmbg = RemoveBg("cKiVxy8acVYCTS5idwoH72p4", "error.log") 
rmbg.remove_background_from_img_file("photo/test.jpg")
............
```

通过该项目，实现了照片的人物提取，提高3D建模精度

### Step three  本地AI模型的骨架分析

通过服务端搭建的人体骨架AI分析模型，对上文预处理过的照片进行人体骨架分析，收集人体形态数据，为后文纵深分析提供高可信度数据，进一步提升3D建模的精细化

![](http://pic.bamboo22.top/image/adaafs.jpg)

### Step five   本地AI模型的纵深分析

通过服务端搭建的人体纵深AI分析模型，对上文已收集的形态数据进行处理，AI会通过光影，地面等3D参照物，对人体模型的纵深进行猜测，实例化该人像的3D端点，再次提高了后文3D建模的精细程度

![](http://pic.bamboo22.top/image/result_c_720 (1).png)

###  Step six  本地AI模型进行3D建模

服务端的人体3D建模AI对上文已经分析处理好的人物形象数据进行模型的搭建，生成3D人体模型，并保存在服务端本地，供厂商使用，同时也为用户提供模型的预览，模型部分加载代码如下

```html
<script>
        // 创建场景
        const scene = new THREE.Scene();

        // 创建相机
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.set(0, 0, 20);

        // 添加全局光源
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
        const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
        directionalLight.position.set(10, 10, 10);
        scene.add(ambientLight);
        scene.add(directionalLight);

        // 创建渲染器
        const renderer = new THREE.WebGLRenderer();
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('canvas').appendChild(renderer.domElement);
		renderer.setClearColor(0xffffff);
    ...........
```

![](http://pic.bamboo22.top/image/result_c_720.gif)
