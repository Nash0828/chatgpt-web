<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>圆柱体透明度渐变效果 - 完全修复版</title>
    <style>
        body { 
            margin: 0; 
            overflow: hidden; 
            background: linear-gradient(to bottom, #1a1a2e, #16213e);
            color: white;
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
            background: rgba(0,0,0,0.7); 
            padding: 15px; 
            border-radius: 5px;
            max-width: 320px;
        }
        h1 { 
            font-size: 18px; 
            margin-top: 0;
            color: #2ecc71;
        }
        .controls {
            margin-top: 15px;
        }
        .slider-container {
            margin: 10px 0;
        }
        label {
            display: block;
            margin-bottom: 5px;
        }
        input[type="range"] {
            width: 100%;
        }
        .mode-buttons {
            display: flex;
            gap: 10px;
            margin-top: 15px;
        }
        button {
            padding: 8px 12px;
            background: #3498db;
            border: none;
            border-radius: 4px;
            color: white;
            cursor: pointer;
            flex: 1;
        }
        button.active {
            background: #2ecc71;
        }
        .explanation {
            margin-top: 15px;
            font-size: 14px;
            color: #ddd;
        }
    </style>
</head>
<body>
    <div id="container"></div>
    <div id="info">
        <h1>圆柱体透明度渐变效果 - 完全修复版</h1>
        <p>使用无光照着色器彻底解决明暗交界问题</p>
        
        <div class="controls">
            <div class="slider-container">
                <label for="topRadius">顶部半径: <span id="topRadiusValue">2</span></label>
                <input type="range" id="topRadius" min="1" max="10" value="2" step="0.1">
            </div>
            
            <div class="slider-container">
                <label for="bottomRadius">底部半径: <span id="bottomRadiusValue">5</span></label>
                <input type="range" id="bottomRadius" min="1" max="10" value="5" step="0.1">
            </div>
            
            <div class="slider-container">
                <label for="height">高度: <span id="heightValue">10</span></label>
                <input type="range" id="height" min="5" max="20" value="10" step="0.1">
            </div>
            
            <div class="mode-buttons">
                <button id="mode1" class="active">顶部透明</button>
                <button id="mode2">底部透明</button>
                <button id="mode3">双向渐变</button>
            </div>
        </div>

        <div class="explanation">
            <p>修复方案：使用无光照着色器，仅基于UV坐标计算渐变，彻底避免法线导致的明暗问题。</p>
        </div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/three@0.132.2/build/three.min.js"></script>
    <script>
        // 初始化场景
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setClearColor(0x000000, 0);
        document.getElementById('container').appendChild(renderer.domElement);

        // 顶点着色器 - 只传递UV坐标
        const vertexShader = `
            varying vec2 vUv;
            
            void main() {
                vUv = uv;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
            }
        `;

        // 片元着色器 - 模式1：顶部透明，底部不透明
        const fragmentShader1 = `
            uniform vec3 color;
            varying vec2 vUv;
            
            void main() {
                // 基于UV的y坐标创建渐变 - 顶部透明，底部不透明
                float alpha = vUv.y;
                
                // 添加一些微妙的颜色变化
                vec3 finalColor = color;
                finalColor.r += 0.1 * (1.0 - vUv.y);
                finalColor.g += 0.05 * vUv.y;
                
                gl_FragColor = vec4(finalColor, alpha);
            }
        `;

        // 片元着色器 - 模式2：底部透明，顶部不透明
        const fragmentShader2 = `
            uniform vec3 color;
            varying vec2 vUv;
            
            void main() {
                // 基于UV的y坐标创建渐变 - 底部透明，顶部不透明
                float alpha = 1.0 - vUv.y;
                
                // 添加一些微妙的颜色变化
                vec3 finalColor = color;
                finalColor.r += 0.1 * vUv.y;
                finalColor.g += 0.05 * (1.0 - vUv.y);
                
                gl_FragColor = vec4(finalColor, alpha);
            }
        `;

        // 片元着色器 - 模式3：双向渐变（中间透明，两端不透明）
        const fragmentShader3 = `
            uniform vec3 color;
            varying vec2 vUv;
            
            void main() {
                // 双向渐变 - 中间透明，两端不透明
                float alpha = 1.0 - 2.0 * abs(vUv.y - 0.5);
                
                // 添加一些微妙的颜色变化
                vec3 finalColor = color;
                finalColor.r += 0.1 * (1.0 - vUv.y);
                finalColor.g += 0.05 * vUv.y;
                
                gl_FragColor = vec4(finalColor, alpha);
            }
        `;

        // 初始几何体参数
        let topRadius = 2;
        let bottomRadius = 5;
        let height = 10;
        let currentShader = 1;
        
        // 创建圆柱体几何体
        const geometry = new THREE.CylinderGeometry(topRadius, bottomRadius, height, 64, 32, true);
        
        // 创建材质 - 使用无光照的着色器
        const material = new THREE.ShaderMaterial({
            uniforms: {
                color: { value: new THREE.Color(0.2, 0.5, 0.8) }
            },
            vertexShader: vertexShader,
            fragmentShader: fragmentShader1,
            transparent: true,
            side: THREE.DoubleSide
        });

        // 创建网格
        const cylinder = new THREE.Mesh(geometry, material);
        scene.add(cylinder);

        // 添加参考网格和坐标轴
        const gridHelper = new THREE.GridHelper(30, 30, 0x444444, 0x222222);
        scene.add(gridHelper);
        
        const axesHelper = new THREE.AxesHelper(10);
        scene.add(axesHelper);

        // 设置相机位置
        camera.position.set(20, 15, 20);
        camera.lookAt(0, 0, 0);

        // 动画循环
        function animate() {
            requestAnimationFrame(animate);
            
            // 旋转圆柱体
            cylinder.rotation.y += 0.005;
            
            renderer.render(scene, camera);
        }
        animate();

        // 响应窗口调整
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        // 添加控制功能
        document.getElementById('topRadius').addEventListener('input', (e) => {
            topRadius = parseFloat(e.target.value);
            document.getElementById('topRadiusValue').textContent = topRadius.toFixed(1);
            updateGeometry();
        });

        document.getElementById('bottomRadius').addEventListener('input', (e) => {
            bottomRadius = parseFloat(e.target.value);
            document.getElementById('bottomRadiusValue').textContent = bottomRadius.toFixed(1);
            updateGeometry();
        });

        document.getElementById('height').addEventListener('input', (e) => {
            height = parseFloat(e.target.value);
            document.getElementById('heightValue').textContent = height.toFixed(1);
            updateGeometry();
        });

        // 切换着色器模式
        document.getElementById('mode1').addEventListener('click', () => {
            currentShader = 1;
            material.fragmentShader = fragmentShader1;
            material.needsUpdate = true;
            updateButtonStates();
        });

        document.getElementById('mode2').addEventListener('click', () => {
            currentShader = 2;
            material.fragmentShader = fragmentShader2;
            material.needsUpdate = true;
            updateButtonStates();
        });

        document.getElementById('mode3').addEventListener('click', () => {
            currentShader = 3;
            material.fragmentShader = fragmentShader3;
            material.needsUpdate = true;
            updateButtonStates();
        });

        function updateButtonStates() {
            document.getElementById('mode1').classList.toggle('active', currentShader === 1);
            document.getElementById('mode2').classList.toggle('active', currentShader === 2);
            document.getElementById('mode3').classList.toggle('active', currentShader === 3);
        }

        function updateGeometry() {
            const newGeometry = new THREE.CylinderGeometry(
                topRadius, bottomRadius, height, 64, 32, true
            );
            cylinder.geometry.dispose();
            cylinder.geometry = newGeometry;
        }

        // 初始按钮状态
        updateButtonStates();
    </script>
</body>
</html>