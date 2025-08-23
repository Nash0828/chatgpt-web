<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>圆角矩形平面 - Three.js</title>
    <style>
        body { 
            margin: 0; 
            overflow: hidden; 
            background-color: #f0f0f0;
        }
        canvas { 
            display: block; 
        }
        .info {
            position: absolute;
            top: 20px;
            left: 20px;
            color: white;
            background: rgba(0,0,0,0.7);
            padding: 15px;
            border-radius: 5px;
            font-family: Arial, sans-serif;
        }
    </style>
</head>
<body>
    <div class="info">
        <h2>圆角矩形平面</h2>
        <p>厚度: 0.5单位</p>
        <p>尺寸: 10×6单位</p>
        <p>圆角半径: 1单位</p>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/three@0.132.2/build/three.min.js"></script>
    <script>
        // 初始化场景
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x444444);
        
        // 初始化相机
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.z = 20;
        camera.position.y = 5;
        camera.lookAt(0, 0, 0);
        
        // 初始化渲染器
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.documentElement);
        
        // 添加光源
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
        scene.add(ambientLight);
        
        const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
        directionalLight.position.set(5, 10, 7);
        scene.add(directionalLight);
        
        // 创建圆角矩形平面
        function createRoundedRect(width, height, radius, thickness) {
            // 创建圆角矩形形状
            const shape = new THREE.Shape();
            
            // 定义圆角矩形路径
            shape.moveTo(-width/2 + radius, -height/2);
            shape.lineTo(width/2 - radius, -height/2);
            shape.quadraticCurveTo(width/2, -height/2, width/2, -height/2 + radius);
            shape.lineTo(width/2, height/2 - radius);
            shape.quadraticCurveTo(width/2, height/2, width/2 - radius, height/2);
            shape.lineTo(-width/2 + radius, height/2);
            shape.quadraticCurveTo(-width/2, height/2, -width/2, height/2 - radius);
            shape.lineTo(-width/2, -height/2 + radius);
            shape.quadraticCurveTo(-width/2, -height/2, -width/2 + radius, -height/2);
            
            // 挤出设置
            const extrudeSettings = {
                depth: thickness,        // 厚度
                bevelEnabled: false      // 不启用斜角
            };
            
            // 创建几何体
            const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
            
            // 创建材质
            const material = new THREE.MeshPhongMaterial({ 
                color: 0x3498db,
                shininess: 100,
                side: THREE.DoubleSide
            });
            
            // 创建网格
            const mesh = new THREE.Mesh(geometry, material);
            
            return mesh;
        }
        
        // 创建圆角矩形（宽度10，高度6，圆角半径1，厚度0.5）
        const roundedRect = createRoundedRect(10, 6, 1, 0.5);
        scene.add(roundedRect);
        
        // 添加网格辅助线
        const gridHelper = new THREE.GridHelper(20, 20);
        scene.add(gridHelper);
        
        const axesHelper = new THREE.AxesHelper(5);
        scene.add(axesHelper);
        
        // 动画循环
        function animate() {
            requestAnimationFrame(animate);
            
            // 缓慢旋转模型以便观察
            roundedRect.rotation.y += 0.01;
            
            renderer.render(scene, camera);
        }
        
        // 处理窗口大小变化
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
        
        animate();
    </script>
</body>
</html>