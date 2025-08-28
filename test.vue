<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Vue 3 + Three.js 电灯泡光束效果</title>
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
            height: 500px;
            border-radius: 12px;
            overflow: hidden;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
            background: #000;
        }
        .controls {
            flex: 1;
            min-width: 300px;
            background: rgba(0, 0, 0, 0.6);
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
            color: #80deea;
        }
        input[type="range"] {
            width: 100%;
            height: 6px;
            border-radius: 3px;
            background: #37474f;
            outline: none;
            -webkit-appearance: none;
        }
        input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            width: 20px;
            height: 20px;
            border-radius: 50%;
            background: #00e5ff;
            cursor: pointer;
            box-shadow: 0 0 10px rgba(0, 229, 255, 0.8);
        }
        .value-display {
            display: flex;
            justify-content: space-between;
            margin-top: 5px;
            font-size: 0.9em;
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
            background: #00b8d4;
            color: white;
            font-weight: bold;
            cursor: pointer;
            transition: all 0.3s ease;
        }
        button:hover {
            background: #00e5ff;
            transform: translateY(-2px);
            box-shadow: 0 4px 8px rgba(0, 229, 255, 0.4);
        }
        .light-intensity {
            display: flex;
            align-items: center;
            justify-content: center;
            margin-top: 20px;
            font-size: 18px;
            padding: 10px;
            background: rgba(0, 184, 212, 0.2);
            border-radius: 8px;
        }
        .icon {
            margin-right: 10px;
            font-size: 24px;
        }
        .instructions {
            margin-top: 30px;
            padding: 15px;
            background: rgba(0, 184, 212, 0.1);
            border-radius: 8px;
            font-size: 14px;
            border-left: 4px solid #00e5ff;
        }
        .footer {
            margin-top: 30px;
            text-align: center;
            font-size: 14px;
            opacity: 0.8;
        }
        .beam-visible {
            display: flex;
            align-items: center;
            margin-top: 15px;
        }
        .toggle-switch {
            position: relative;
            display: inline-block;
            width: 50px;
            height: 24px;
            margin-left: 10px;
        }
        .toggle-switch input {
            opacity: 0;
            width: 0;
            height: 0;
        }
        .slider {
            position: absolute;
            cursor: pointer;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background-color: #37474f;
            transition: .4s;
            border-radius: 24px;
        }
        .slider:before {
            position: absolute;
            content: "";
            height: 16px;
            width: 16px;
            left: 4px;
            bottom: 4px;
            background-color: white;
            transition: .4s;
            border-radius: 50%;
        }
        input:checked + .slider {
            background-color: #00e5ff;
        }
        input:checked + .slider:before {
            transform: translateX(26px);
        }
    </style>
</head>
<body>
    <div id="app">
        <h1>Vue 3 + Three.js 电灯泡光束效果</h1>
        <p class="description">开放式光束圆锥体 - 无顶盖设计，更真实的灯光效果</p>
        
        <div class="container">
            <div class="canvas-container">
                <canvas id="threeCanvas"></canvas>
            </div>
            
            <div class="controls">
                <div class="slider-container">
                    <label for="lightAngle">灯光角度: {{ lightAngle.toFixed(0) }}°</label>
                    <input type="range" id="lightAngle" min="15" max="60" step="1" v-model.number="lightAngle">
                    <div class="value-display">
                        <span>窄光束</span>
                        <span>宽光束</span>
                    </div>
                </div>
                
                <div class="slider-container">
                    <label for="lightIntensity">灯光强度: {{ lightIntensity.toFixed(1) }}</label>
                    <input type="range" id="lightIntensity" min="0.5" max="3" step="0.1" v-model.number="lightIntensity">
                    <div class="value-display">
                        <span>柔和</span>
                        <span>明亮</span>
                    </div>
                </div>
                
                <div class="slider-container">
                    <label for="beamIntensity">光束强度: {{ beamIntensity.toFixed(1) }}</label>
                    <input type="range" id="beamIntensity" min="0.1" max="2" step="0.1" v-model.number="beamIntensity">
                    <div class="value-display">
                        <span>微弱</span>
                        <span>强烈</span>
                    </div>
                </div>
                
                <div class="slider-container">
                    <label for="beamLength">光束长度: {{ beamLength.toFixed(1) }}</label>
                    <input type="range" id="beamLength" min="3" max="10" step="0.5" v-model.number="beamLength">
                    <div class="value-display">
                        <span>短</span>
                        <span>长</span>
                    </div>
                </div>
                
                <div class="slider-container">
                    <label for="rotationSpeed">旋转速度: {{ rotationSpeed.toFixed(3) }}</label>
                    <input type="range" id="rotationSpeed" min="0" max="0.01" step="0.001" v-model.number="rotationSpeed">
                    <div class="value-display">
                        <span>静止</span>
                        <span>旋转</span>
                    </div>
                </div>
                
                <div class="beam-visible">
                    <label>显示光束:</label>
                    <label class="toggle-switch">
                        <input type="checkbox" v-model="showBeam">
                        <span class="slider"></span>
                    </label>
                </div>
                
                <div class="preset-buttons">
                    <button @click="setPreset('soft')">柔和光效</button>
                    <button @click="setPreset('bright')">明亮光效</button>
                    <button @click="setPreset('spot')">聚光效果</button>
                    <button @click="setPreset('disco')">炫彩效果</button>
                </div>
                
                <div class="light-intensity">
                    <span class="icon">💡</span>
                    <span>光束长度: {{ beamLength.toFixed(1) }} 单位</span>
                </div>
                
                <div class="instructions">
                    <p><strong>使用说明:</strong> 开放式光束圆锥体，无顶盖设计，可以看到光束从灯泡发出并向下散射的效果。</p>
                </div>
            </div>
        </div>
        
        <div class="footer">
            <p>Vue 3 Composition API 与 Three.js 开放式光束效果</p>
        </div>
    </div>

    <script type="module">
        const { createApp, ref, watch, onMounted } = Vue;
        
        createApp({
            setup() {
                // 响应式数据
                const lightAngle = ref(35);
                const lightIntensity = ref(1.5);
                const beamIntensity = ref(0.8);
                const beamLength = ref(5);
                const rotationSpeed = ref(0.003);
                const showBeam = ref(true);
                
                // Three.js 相关变量
                let scene, camera, renderer, bulbGroup, spotLight, beamCone;
                
                // 创建开放式光束圆锥体（无顶盖）
                const createOpenConeGeometry = (radius, height, radialSegments, openTop = true) => {
                    const geometry = new THREE.ConeGeometry(radius, height, radialSegments, 1, openTop);
                    return geometry;
                };
                
                // 创建光束效果 - 开放式圆锥体
                const createLightBeam = () => {
                    // 创建开放式圆锥体作为光束（无顶盖）
                    const coneGeometry = createOpenConeGeometry(0.1, beamLength.value, 32, true);
                    
                    const coneMaterial = new THREE.MeshPhongMaterial({
                        color: 0xffffaa,
                        transparent: true,
                        opacity: beamIntensity.value * 0.6,
                        side: THREE.DoubleSide,
                        blending: THREE.AdditiveBlending,
                        wireframe: false
                    });
                    
                    beamCone = new THREE.Mesh(coneGeometry, coneMaterial);
                    beamCone.position.y = -beamLength.value / 2;
                    beamCone.rotation.x = Math.PI;
                    beamCone.visible = showBeam.value;
                    
                    return beamCone;
                };
                
                // 更新光束效果
                const updateLightBeam = () => {
                    if (!beamCone) return;
                    
                    // 移除旧的光束
                    bulbGroup.remove(beamCone);
                    
                    // 创建新的光束（更新长度）
                    beamCone = createLightBeam();
                    bulbGroup.add(beamCone);
                    
                    // 根据灯光角度调整光束圆锥的角度
                    const scale = Math.tan(spotLight.angle) * beamLength.value;
                    beamCone.scale.set(scale, 1, scale);
                    
                    // 更新光束材质
                    beamCone.material.opacity = beamIntensity.value * 0.6;
                    beamCone.material.color = new THREE.Color(0xffffaa).lerp(
                        new THREE.Color(0xffaa00), 
                        1 - (lightIntensity.value / 3)
                    );
                    
                    beamCone.visible = showBeam.value;
                };
                
                // 初始化场景
                const initScene = () => {
                    // 创建场景
                    scene = new THREE.Scene();
                    scene.background = new THREE.Color(0x0a0a20);
                    scene.fog = new THREE.Fog(0x0a0a20, 5, 20);
                    
                    // 创建相机
                    camera = new THREE.PerspectiveCamera(60, 1, 0.1, 1000);
                    camera.position.set(3, 4, 6);
                    camera.lookAt(0, 0, 0);
                    
                    // 创建渲染器
                    const canvas = document.getElementById('threeCanvas');
                    renderer = new THREE.WebGLRenderer({ 
                        canvas, 
                        antialias: true,
                        alpha: true
                    });
                    updateRendererSize();
                    
                    // 启用阴影
                    renderer.shadowMap.enabled = true;
                    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
                    
                    // 添加环境光
                    const ambientLight = new THREE.AmbientLight(0x404040, 0.3);
                    scene.add(ambientLight);
                    
                    // 创建灯泡组
                    bulbGroup = new THREE.Group();
                    scene.add(bulbGroup);
                    
                    // 创建灯泡球体
                    const bulbGeometry = new THREE.SphereGeometry(0.3, 32, 32);
                    const bulbMaterial = new THREE.MeshStandardMaterial({ 
                        color: 0xffffcc,
                        emissive: 0xffff66,
                        emissiveIntensity: 0.5
                    });
                    const bulb = new THREE.Mesh(bulbGeometry, bulbMaterial);
                    bulb.position.y = 0.5;
                    bulb.castShadow = true;
                    bulbGroup.add(bulb);
                    
                    // 创建灯罩
                    const lampshadeGeometry = new THREE.ConeGeometry(0.8, 1, 32);
                    const lampshadeMaterial = new THREE.MeshStandardMaterial({
                        color: 0x555555,
                        metalness: 0.5,
                        roughness: 0.5,
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
                    spotLight.distance = 20;
                    spotLight.castShadow = true;
                    spotLight.shadow.mapSize.width = 1024;
                    spotLight.shadow.mapSize.height = 1024;
                    bulbGroup.add(spotLight);
                    
                    // 创建光束效果
                    const lightBeam = createLightBeam();
                    bulbGroup.add(lightBeam);
                    
                    // 创建地面
                    const floorGeometry = new THREE.PlaneGeometry(30, 30);
                    const floorMaterial = new THREE.MeshStandardMaterial({ 
                        color: 0x222222,
                        roughness: 0.8,
                        metalness: 0.2
                    });
                    const floor = new THREE.Mesh(floorGeometry, floorMaterial);
                    floor.rotation.x = -Math.PI / 2;
                    floor.position.y = -3;
                    floor.receiveShadow = true;
                    scene.add(floor);
                    
                    // 添加一些物体来展示光照效果
                    const cubeGeometry = new THREE.BoxGeometry(0.5, 0.5, 0.5);
                    const sphereGeometry = new THREE.SphereGeometry(0.3, 32, 32);
                    const cylinderGeometry = new THREE.CylinderGeometry(0.3, 0.3, 1, 32);
                    const cubeMaterial = new THREE.MeshStandardMaterial({ color: 0x00aaff });
                    const sphereMaterial = new THREE.MeshStandardMaterial({ color: 0xff6b6b });
                    const cylinderMaterial = new THREE.MeshStandardMaterial({ color: 0x66bb6a });
                    
                    // 添加立方体
                    for (let i = 0; i < 5; i++) {
                        const cube = new THREE.Mesh(cubeGeometry, cubeMaterial);
                        cube.position.set(i - 2, -2.5, -4);
                        cube.castShadow = true;
                        cube.receiveShadow = true;
                        scene.add(cube);
                    }
                    
                    // 添加球体
                    for (let i = 0; i < 4; i++) {
                        const sphere = new THREE.Mesh(sphereGeometry, sphereMaterial);
                        sphere.position.set(i - 1.5, -2.5, -2);
                        sphere.castShadow = true;
                        sphere.receiveShadow = true;
                        scene.add(sphere);
                    }
                    
                    // 添加圆柱体
                    for (let i = 0; i < 3; i++) {
                        const cylinder = new THREE.Mesh(cylinderGeometry, cylinderMaterial);
                        cylinder.position.set(i - 1, -2, 0);
                        cylinder.castShadow = true;
                        cylinder.receiveShadow = true;
                        scene.add(cylinder);
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
                    
                    // 更新光束效果
                    if (beamCone && spotLight) {
                        const scale = Math.tan(spotLight.angle) * beamLength.value;
                        beamCone.scale.set(scale, 1, scale);
                    }
                    
                    if (renderer && scene && camera) {
                        renderer.render(scene, camera);
                    }
                };
                
                // 设置预设
                const setPreset = (preset) => {
                    switch(preset) {
                        case 'soft':
                            lightAngle.value = 50;
                            lightIntensity.value = 1.0;
                            beamIntensity.value = 0.4;
                            beamLength.value = 4;
                            showBeam.value = true;
                            break;
                        case 'bright':
                            lightAngle.value = 30;
                            lightIntensity.value = 2.5;
                            beamIntensity.value = 0.7;
                            beamLength.value = 6;
                            showBeam.value = true;
                            break;
                        case 'spot':
                            lightAngle.value = 20;
                            lightIntensity.value = 2.0;
                            beamIntensity.value = 1.2;
                            beamLength.value = 8;
                            showBeam.value = true;
                            break;
                        case 'disco':
                            lightAngle.value = 45;
                            lightIntensity.value = 2.0;
                            beamIntensity.value = 1.5;
                            beamLength.value = 7;
                            rotationSpeed.value = 0.008;
                            showBeam.value = true;
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
                
                watch(beamIntensity, () => {
                    if (beamCone) {
                        beamCone.material.opacity = beamIntensity.value * 0.6;
                    }
                });
                
                watch(beamLength, updateLightBeam);
                watch(showBeam, () => {
                    if (beamCone) {
                        beamCone.visible = showBeam.value;
                    }
                });
                
                // 初始化
                onMounted(() => {
                    initScene();
                });
                
                return {
                    lightAngle,
                    lightIntensity,
                    beamIntensity,
                    beamLength,
                    rotationSpeed,
                    showBeam,
                    setPreset
                };
            }
        }).mount('#app');
    </script>
</body>
</html>