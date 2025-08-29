<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>圆柱体透明度渐变效果</title>
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
    </style>
</head>
<body>
    <div id="container"></div>
    <div id="info">
        <h1>圆柱体侧面透明度渐变效果</h1>
        <p>使用自定义着色器实现从顶部到底部的透明度渐变</p>
        
        <div class="controls">
            <div class="slider-container">
                <label for="topRadius">顶部半径: <span id="topRadiusValue">5</span></label>
                <input type="range" id="topRadius" min="1" max="10" value="5" step="0.1">
            </div>
            
            <div class="slider-container">
                <label for="bottomRadius">底部半径: <span id="bottomRadiusValue">5</span></label>
                <input type="range" id="bottomRadius" min="1" max="10" value="5" step="0.1">
            </div>
            
            <div class="slider-container">
                <label for="height">高度: <span id="heightValue">10</span></label>
                <input type="range" id="height" min="5" max="20" value="10" step="0.1">
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
        const ambientLight = new THREE.AmbientLight(0x404040);
        scene.add(ambientLight);
        
        const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
        directionalLight.position.set(1, 1, 1);
        scene.add(directionalLight);

        // 创建自定义着色器材质
        const vertexShader = `
            varying vec2 vUv;
            
            void main() {
                vUv = uv;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
            }
        `;

        const fragmentShader = `
            uniform vec3 color;
            varying vec2 vUv;
            
            void main() {
                // 使用UV的y坐标控制透明度 - 从顶部(1.0)到底部(0.0)渐变
                float alpha = 1.0 - vUv.y;
                
                // 添加一些微妙的颜色变化
                vec3 finalColor = color;
                finalColor.r += 0.1 * (1.0 - vUv.y);
                finalColor.g += 0.05 * vUv.y;
                
                gl_FragColor = vec4(finalColor, alpha);
            }
        `;

        // 初始几何体参数
        let topRadius = 5;
        let bottomRadius = 5;
        let height = 10;
        
        // 创建圆柱体几何体
        const geometry = new THREE.CylinderGeometry(topRadius, bottomRadius, height, 32, 32, true);
        
        // 创建材质
        const material = new THREE.ShaderMaterial({
            uniforms: {
                color: { value: new THREE.Color(0.2, 0.5, 0.8) }
            },
            vertexShader: vertexShader,
            fragmentShader: fragmentShader,
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