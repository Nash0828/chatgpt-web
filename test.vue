<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>圆柱体透明度渐变效果优化</title>
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
    </style>
</head>
<body>
    <div id="container"></div>
    <div id="info">
        <h1>圆柱体侧面透明度渐变效果优化</h1>
        <p>解决了侧面交界处的明暗不一致问题</p>
        
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
                <button id="mode1" class="active">渐变模式1</button>
                <button id="mode2">渐变模式2</button>
            </div>
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

        // 添加灯光
        const ambientLight = new THREE.AmbientLight(0x404040, 0.8);
        scene.add(ambientLight);
        
        const directionalLight1 = new THREE.DirectionalLight(0xffffff, 0.6);
        directionalLight1.position.set(1, 1, 1);
        scene.add(directionalLight1);
        
        const directionalLight2 = new THREE.DirectionalLight(0xffffff, 0.4);
        directionalLight2.position.set(-1, -1, -1);
        scene.add(directionalLight2);

        // 顶点着色器 - 传递位置和法线信息
        const vertexShader = `
            varying vec3 vNormal;
            varying vec3 vPosition;
            varying vec2 vUv;
            
            void main() {
                vNormal = normalize(normalMatrix * normal);
                vPosition = position;
                vUv = uv;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
            }
        `;

        // 片元着色器 - 模式1：基于法线的平滑渐变
        const fragmentShader1 = `
            uniform vec3 color;
            varying vec3 vNormal;
            varying vec3 vPosition;
            varying vec2 vUv;
            
            void main() {
                // 使用法线的y分量和UV的y值结合创建平滑渐变
                float normalFactor = abs(vNormal.y);
                float alpha = 1.0 - smoothstep(0.0, 1.0, vUv.y);
                
                // 结合法线因子和UV渐变
                alpha = mix(alpha, alpha * (1.0 - normalFactor * 0.5), 0.3);
                
                // 添加一些微妙的颜色变化
                vec3 finalColor = color;
                finalColor.r += 0.1 * (1.0 - vUv.y);
                finalColor.g += 0.05 * vUv.y;
                
                gl_FragColor = vec4(finalColor, alpha);
            }
        `;

        // 片元着色器 - 模式2：基于位置的渐变
        const fragmentShader2 = `
            uniform vec3 color;
            varying vec3 vNormal;
            varying vec3 vPosition;
            varying vec2 vUv;
            
            void main() {
                // 基于位置的高度计算渐变
                float height = (vPosition.y + 5.0) / 10.0; // 归一化到0-1范围
                float alpha = 1.0 - smoothstep(0.0, 1.0, height);
                
                // 添加一些微妙的颜色变化
                vec3 finalColor = color;
                finalColor.r += 0.1 * (1.0 - height);
                finalColor.g += 0.05 * height;
                
                gl_FragColor = vec4(finalColor, alpha);
            }
        `;

        // 初始几何体参数
        let topRadius = 2;
        let bottomRadius = 5;
        let height = 10;
        let currentShader = 1;
        
        // 创建圆柱体几何体
        const geometry = new THREE.CylinderGeometry(topRadius, bottomRadius, height, 32, 32, true);
        
        // 创建材质
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

        // 添加参考网格
        const gridHelper = new THREE.GridHelper(30, 30, 0x444444, 0x222222);
        scene.add(gridHelper);

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

        function updateButtonStates() {
            document.getElementById('mode1').classList.toggle('active', currentShader === 1);
            document.getElementById('mode2').classList.toggle('active', currentShader === 2);
        }

        function updateGeometry() {
            const newGeometry = new THREE.CylinderGeometry(
                topRadius, bottomRadius, height, 32, 32, true
            );
            cylinder.geometry.dispose();
            cylinder.geometry = newGeometry;
        }
    </script>
</body>
</html>