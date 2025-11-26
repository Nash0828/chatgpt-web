<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Three.js 折线拖尾动画</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #000;
            font-family: Arial, sans-serif;
        }
        #container {
            position: relative;
            width: 100vw;
            height: 100vh;
        }
        #info {
            position: absolute;
            top: 20px;
            left: 20px;
            color: white;
            background-color: rgba(0,0,0,0.7);
            padding: 10px;
            border-radius: 5px;
            z-index: 100;
        }
    </style>
</head>
<body>
    <div id="container"></div>
    <div id="info">
        <h3>Three.js 折线拖尾动画</h3>
        <p>使用着色器实现无限循环的拖尾效果</p>
    </div>

    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.158.0/build/three.module.js",
                "three/addons/": "https://unpkg.com/three@0.158.0/examples/jsm/"
            }
        }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

        // 初始化场景
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x111122);
        
        // 初始化相机
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.set(2, 2, 5);
        camera.lookAt(0, 0, 0);
        
        // 初始化渲染器
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        document.getElementById('container').appendChild(renderer.domElement);
        
        // 添加轨道控制器
        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        
        // 添加环境光
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
        scene.add(ambientLight);
        
        // 添加方向光
        const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
        directionalLight.position.set(1, 1, 1);
        scene.add(directionalLight);
        
        // 定义四个点
        const points = [
            new THREE.Vector3(0, 2, 0),
            new THREE.Vector3(0, 1, 0),
            new THREE.Vector3(1, 1, 0),
            new THREE.Vector3(1, 0, 0)
        ];
        
        // 创建折线几何体
        const lineGeometry = new THREE.BufferGeometry();
        lineGeometry.setFromPoints(points);
        
        // 创建折线材质
        const lineMaterial = new THREE.LineBasicMaterial({
            color: 0xffffff,
            linewidth: 2
        });
        
        // 创建折线对象并添加到场景
        const line = new THREE.Line(lineGeometry, lineMaterial);
        scene.add(line);
        
        // 创建拖尾效果的着色器材质
        const trailMaterial = new THREE.ShaderMaterial({
            uniforms: {
                time: { value: 0.0 }, // 时间统一变量，用于动画
                color: { value: new THREE.Color(0x00ffff) }, // 拖尾颜色
                length: { value: 0.5 } // 拖尾长度
            },
            vertexShader: `
                uniform float time; // 时间
                uniform float length; // 拖尾长度
                
                attribute float progress; // 每个点在路径上的进度 (0-1)
                
                varying float vProgress; // 传递给片段着色器的进度
                
                void main() {
                    vProgress = progress;
                    
                    // 计算动画偏移
                    float animationOffset = mod(time * 0.5, 1.0); // 动画偏移，循环在0-1之间
                    float animatedProgress = mod(progress + animationOffset, 1.0); // 应用动画偏移
                    
                    // 计算拖尾效果 - 当点在拖尾范围内时显示
                    float trailVisibility = step(animatedProgress, length);
                    
                    // 根据拖尾可见性调整位置
                    vec3 pos = position;
                    if(trailVisibility < 0.5) {
                        // 不在拖尾范围内的点移到远处，使其不可见
                        pos = vec3(0.0, 0.0, -100.0);
                    }
                    
                    gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
                }
            `,
            fragmentShader: `
                uniform vec3 color; // 拖尾颜色
                varying float vProgress; // 从顶点着色器传递的进度
                
                void main() {
                    // 根据进度调整透明度 - 拖尾头部更不透明，尾部更透明
                    float alpha = 1.0 - vProgress;
                    
                    // 应用颜色和透明度
                    gl_FragColor = vec4(color, alpha);
                }
            `,
            transparent: true, // 启用透明度
            blending: THREE.AdditiveBlending, // 使用叠加混合，增强视觉效果
            depthWrite: false // 禁用深度写入，防止拖尾被其他物体遮挡
        });
        
        // 创建拖尾几何体
        const trailGeometry = new THREE.BufferGeometry();
        
        // 为折线路径生成更多的点，使拖尾更平滑
        const curve = new THREE.CatmullRomCurve3(points);
        const trailPoints = curve.getPoints(100); // 生成100个点
        
        // 设置位置属性
        const positions = new Float32Array(trailPoints.length * 3);
        trailPoints.forEach((point, i) => {
            positions[i * 3] = point.x;
            positions[i * 3 + 1] = point.y;
            positions[i * 3 + 2] = point.z;
        });
        trailGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        
        // 设置进度属性 (0-1)
        const progress = new Float32Array(trailPoints.length);
        for (let i = 0; i < trailPoints.length; i++) {
            progress[i] = i / (trailPoints.length - 1);
        }
        trailGeometry.setAttribute('progress', new THREE.BufferAttribute(progress, 1));
        
        // 创建拖尾对象并添加到场景
        const trail = new THREE.Points(trailGeometry, trailMaterial);
        scene.add(trail);
        
        // 添加坐标轴辅助
        const axesHelper = new THREE.AxesHelper(2);
        scene.add(axesHelper);
        
        // 添加网格辅助
        const gridHelper = new THREE.GridHelper(5, 10);
        scene.add(gridHelper);
        
        // 动画循环
        const clock = new THREE.Clock();
        
        function animate() {
            requestAnimationFrame(animate);
            
            // 更新时间统一变量
            trailMaterial.uniforms.time.value = clock.getElapsedTime();
            
            // 更新控制器
            controls.update();
            
            // 渲染场景
            renderer.render(scene, camera);
        }
        
        animate();
        
        // 响应窗口大小变化
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>