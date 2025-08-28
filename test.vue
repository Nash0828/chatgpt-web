<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Vue 3 + Three.js 电灯泡照明效果</title>
    <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        body {
            margin: 0;
            padding: 0;
            font-family: 'Arial', sans-serif;
            background: linear-gradient(135deg, #1a237e 0%, #00838f 100%);
            color: #fff;
            min-height: 100vh;
            overflow-x: hidden;
        }
        #app {
            display: flex;
            flex-direction: column;
            align-items: center;
            padding: 20px;
        }
        .container {
            display: flex;
            flex-wrap: wrap;
            gap: 20px;
            max-width: 1200px;
            margin-top: 20px;
        }
        .canvas-container {
            flex: 1;
            min-width: 300px;
            height: 400px;
            border-radius: 12px;
            overflow: hidden;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
            background: #2c3e50;
        }
        .controls {
            flex: 1;
            min-width: 300px;
            background: rgba(255, 255, 255, 0.1);
            backdrop-filter: blur(10px);
            padding: 20px;
            border-radius: 12px;
            box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
        }
        h1 {
            text-align: center;
            margin-bottom: 10px;
            color: #e0f7fa;
            text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
        }
        .description {
            text-align: center;
            margin-bottom: 30px;
            max-width: 800px;
            line-height: 1.6;
        }
        .slider-container {
            margin-bottom: 20px;
        }
        label {
            display: block;
            margin-bottom: 8px;
            font-weight: bold;
        }
        input[type="range"] {
            width: 100%;
            height: 6px;
            border-radius: 3px;
            background: #546e7a;
            outline: none;
            -webkit-appearance: none;
        }
        input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            width: 20px;
            height: 20px;
            border-radius: 50%;
            background: #00b0ff;
            cursor: pointer;
            box-shadow: 0 0 10px rgba(0, 176, 255, 0.7);
        }
        .value-display {
            display: flex;
            justify-content: space-between;
            margin-top: 5px;
        }
        .preset-buttons {
            display: grid;
            grid-template-columns: repeat(2, 1fr);
            gap: 10px;
            margin-top: 20px;
        }
        button {
            padding: 10px;
            border: none;
            border-radius: 6px;
            background: #00b0ff;
            color: white;
            font-weight: bold;
            cursor: pointer;
            transition: all 0.3s ease;
        }
        button:hover {
            background: #0091ea;
            transform: translateY(-2px);
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
        }
        .light-intensity {
            display: flex;
            align-items: center;
            justify-content: center;
            margin-top: 20px;
            font-size: 18px;
        }
        .icon {
            margin-right: 10px;
            font-size: 24px;
        }
        .instructions {
            margin-top: 30px;
            padding: 15px;
            background: rgba(0, 0, 0, 0.2);
            border-radius: 8px;
            font-size: 14px;
        }
        .footer {
            margin-top: 30px;
            text-align: center;
            font-size: 14px;
            opacity: 0.8;
        }
    </style>
</head>
<body>
    <div id="app">
        <h1>Vue 3 + Three.js 电灯泡照明效果</h1>
        <p class="description">使用Vue 3的Composition API和Three.js实现的电灯泡向下照明效果，模拟灯罩的聚光作用。</p>
        
        <div class="container">
            <div class="canvas-container">
                <canvas id="threeCanvas"></canvas>
            </div>
            
            <div class="controls">
                <div class="slider-container">
                    <label for="lightAngle">灯光角度: {{ lightAngle.toFixed(2) }}°</label>
                    <input type="range" id="lightAngle" min="15" max="60" step="1" v-model.number="lightAngle">
                    <div class="value-display">
                        <span>窄</span>
                        <span>宽</span>
                    </div>
                </div>
                
                <div class="slider-container">
                    <label for="lightIntensity">灯光强度: {{ lightIntensity.toFixed(1) }}</label>
                    <input type="range" id="lightIntensity" min="0.5" max="2" step="0.1" v-model.number="lightIntensity">
                    <div class="value-display">
                        <span>柔和</span>
                        <span>明亮</span>
                    </div>
                </div>
                
                <div class="slider-container">
                    <label for="rotationSpeed">旋转速度: {{ rotationSpeed.toFixed(1) }}</label>
                    <input type="range" id="rotationSpeed" min="0" max="0.02" step="0.002" v-model.number="rotationSpeed">
                    <div class="value-display">
                        <span>慢</span>
                        <span>快</span>
                    </div>
                </div>
                
                <div class="preset-buttons">
                    <button @click="setPreset('soft')">柔和光效</button>
                    <button @click="setPreset('bright')">明亮光效</button>
                    <button @click="setPreset('spot')">聚光效果</button>
                    <button @click="setPreset('reset')">重置设置</button>
                </div>
                
                <div class="light-intensity">
                    <span class="icon">💡</span>
                    <span>当前光照强度: {{ (lightIntensity * 50).toFixed(0) }}%</span>
                </div>
                
                <div class="instructions">
                    <p><strong>使用说明:</strong> 拖动滑块调整灯光参数，点击预设按钮应用不同光效。灯泡会自动旋转展示效果。</p>
                </div>
            </div>
        </div>
        
        <div class="footer">
            <p>Vue 3 Composition API 与 Three.js 集成示例</p>
        </div>
    </div>

    <script type="module">
        const { createApp, ref, watch, onMounted } = Vue;
        
        createApp({
            setup() {
                // 响应式数据
                const lightAngle = ref(45);
                const lightIntensity = ref(1.0);
                const rotationSpeed = ref(0.005);
                
                // Three.js 相关变量
                let scene, camera, renderer, bulbGroup, spotLight, spotLightHelper;
                
                // 初始化场景
                const initScene = () => {
                    // 创建场景
                    scene = new THREE.Scene();
                    scene.background = new THREE.Color(0x1a1a1a);
                    
                    // 创建相机
                    camera = new THREE.PerspectiveCamera(75, 1, 0.1, 1000);
                    camera.position.set(0, 2, 5);
                    
                    // 创建渲染器
                    const canvas = document.getElementById('threeCanvas');
                    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
                    updateRendererSize();
                    
                    // 启用阴影
                    renderer.shadowMap.enabled = true;
                    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
                    
                    // 添加环境光
                    const ambientLight = new THREE.AmbientLight(0x404040, 0.5);
                    scene.add(ambientLight);
                    
                    // 创建灯泡组
                    bulbGroup = new THREE.Group();
                    scene.add(bulbGroup);
                    
                    // 创建灯泡球体
                    const bulbGeometry = new THREE.SphereGeometry(0.3, 32, 32);
                    const bulbMaterial = new THREE.MeshStandardMaterial({ 
                        color: 0xffffcc,
                        emissive: 0xffff33,
                        emissiveIntensity: 0.2
                    });
                    const bulb = new THREE.Mesh(bulbGeometry, bulbMaterial);
                    bulb.position.y = 0.5;
                    bulb.castShadow = true;
                    bulbGroup.add(bulb);
                    
                    // 创建灯罩
                    const lampshadeGeometry = new THREE.ConeGeometry(0.8, 1, 32);
                    const lampshadeMaterial = new THREE.MeshStandardMaterial({
                        color: 0x888888,
                        metalness: 0.3,
                        roughness: 0.7,
                        side: THREE.DoubleSide
                    });
                    const lampshade = new THREE.Mesh(lampshadeGeometry, lampshadeMaterial);
                    lampshade.position.y = 0;
                    lampshade.rotation.x = Math.PI;
                    lampshade.castShadow = true;
                    bulbGroup.add(lampshade);
                    
                    // 创建向下照射的聚光灯
                    spotLight = new THREE.SpotLight(0xffffaa, lightIntensity.value);
                    spotLight.position.set(0, 0.3, 0);
                    spotLight.angle = lightAngle.value * Math.PI / 180;
                    spotLight.penumbra = 0.2;
                    spotLight.decay = 1;
                    spotLight.distance = 10;
                    spotLight.castShadow = true;
                    spotLight.shadow.mapSize.width = 1024;
                    spotLight.shadow.mapSize.height = 1024;
                    bulbGroup.add(spotLight);
                    
                    // 添加聚光灯辅助对象
                    spotLightHelper = new THREE.SpotLightHelper(spotLight);
                    scene.add(spotLightHelper);
                    
                    // 创建地面
                    const floorGeometry = new THREE.PlaneGeometry(20, 20);
                    const floorMaterial = new THREE.MeshStandardMaterial({ 
                        color: 0x333333,
                        roughness: 0.8,
                        metalness: 0.2
                    });
                    const floor = new THREE.Mesh(floorGeometry, floorMaterial);
                    floor.rotation.x = -Math.PI / 2;
                    floor.position.y = -1;
                    floor.receiveShadow = true;
                    scene.add(floor);
                    
                    // 添加一些物体来展示光照效果
                    const cubeGeometry = new THREE.BoxGeometry(0.5, 0.5, 0.5);
                    const cubeMaterial = new THREE.MeshStandardMaterial({ color: 0x00aaff });
                    
                    for (let i = 0; i < 5; i++) {
                        const cube = new THREE.Mesh(cubeGeometry, cubeMaterial);
                        cube.position.set(i - 2, -0.75, -2);
                        cube.castShadow = true;
                        cube.receiveShadow = true;
                        scene.add(cube);
                    }
                    
                    // 添加动画
                    animate();
                };
                
                // 更新渲染器尺寸
                const updateRendererSize = () => {
                    const container = document.querySelector('.canvas-container');
                    const width = container.clientWidth;
                    const height = container.clientHeight;
                    
                    if (renderer) {
                        renderer.setSize(width, height);
                    }
                    
                    if (camera) {
                        camera.aspect = width / height;
                        camera.updateProjectionMatrix();
                    }
                };
                
                // 动画循环
                const animate = () => {
                    requestAnimationFrame(animate);
                    
                    // 旋转灯泡组
                    if (bulbGroup) {
                        bulbGroup.rotation.y += rotationSpeed.value;
                    }
                    
                    // 更新聚光灯辅助对象
                    if (spotLightHelper) {
                        spotLightHelper.update();
                    }
                    
                    if (renderer && scene && camera) {
                        renderer.render(scene, camera);
                    }
                };
                
                // 设置预设
                const setPreset = (preset) => {
                    switch(preset) {
                        case 'soft':
                            lightAngle.value = 60;
                            lightIntensity.value = 0.7;
                            break;
                        case 'bright':
                            lightAngle.value = 30;
                            lightIntensity.value = 1.5;
                            break;
                        case 'spot':
                            lightAngle.value = 20;
                            lightIntensity.value = 1.2;
                            break;
                        case 'reset':
                            lightAngle.value = 45;
                            lightIntensity.value = 1.0;
                            rotationSpeed.value = 0.005;
                            break;
                    }
                };
                
                // 监听窗口大小变化
                window.addEventListener('resize', updateRendererSize);
                
                // 监听参数变化
                watch(lightAngle, (newVal) => {
                    if (spotLight) {
                        spotLight.angle = newVal * Math.PI / 180;
                    }
                });
                
                watch(lightIntensity, (newVal) => {
                    if (spotLight) {
                        spotLight.intensity = newVal;
                    }
                });
                
                // 初始化
                onMounted(() => {
                    initScene();
                });
                
                return {
                    lightAngle,
                    lightIntensity,
                    rotationSpeed,
                    setPreset
                };
            }
        }).mount('#app');
    </script>
</body>
</html>