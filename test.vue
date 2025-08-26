<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>圆盘力导向拓扑布局</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.0.0/d3.min.js"></script>
    <style>
        body {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background: linear-gradient(to bottom, #1a1a2e, #16213e);
            color: #fff;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        .container {
            display: flex;
            flex-direction: column;
            height: 100vh;
        }
        .header {
            text-align: center;
            padding: 15px;
            background: rgba(0, 0, 0, 0.3);
            border-bottom: 1px solid rgba(255, 255, 255, 0.1);
        }
        .content {
            flex: 1;
            display: flex;
        }
        #canvas-container {
            flex: 1;
            position: relative;
        }
        .controls {
            width: 250px;
            padding: 20px;
            background: rgba(0, 0, 0, 0.2);
            border-left: 1px solid rgba(255, 255, 255, 0.1);
        }
        .control-group {
            margin-bottom: 20px;
        }
        h1 {
            margin: 0;
            font-size: 24px;
            font-weight: 300;
        }
        h2 {
            font-size: 16px;
            margin-top: 0;
            margin-bottom: 15px;
            color: #6ccefc;
        }
        button {
            background: #3498db;
            color: white;
            border: none;
            padding: 10px 15px;
            border-radius: 4px;
            cursor: pointer;
            width: 100%;
            margin-bottom: 10px;
            transition: background 0.3s;
        }
        button:hover {
            background: #2980b9;
        }
        .slider-container {
            margin-bottom: 15px;
        }
        label {
            display: block;
            margin-bottom: 5px;
            font-size: 14px;
        }
        input[type="range"] {
            width: 100%;
        }
        .info-panel {
            background: rgba(0, 0, 0, 0.3);
            padding: 15px;
            border-radius: 5px;
            font-size: 14px;
        }
        .legend {
            display: flex;
            justify-content: space-around;
            margin-top: 15px;
        }
        .legend-item {
            display: flex;
            align-items: center;
            margin-right: 15px;
        }
        .legend-color {
            width: 15px;
            height: 15px;
            border-radius: 50%;
            margin-right: 5px;
        }
        .instructions {
            margin-top: 20px;
            font-size: 12px;
            color: #aaa;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>圆盘力导向拓扑布局</h1>
        </div>
        <div class="content">
            <div id="canvas-container"></div>
            <div class="controls">
                <div class="control-group">
                    <h2>布局控制</h2>
                    <button id="layout-btn">开始力导向布局</button>
                    <button id="reset-btn">重置布局</button>
                </div>
                
                <div class="control-group">
                    <h2>参数调整</h2>
                    <div class="slider-container">
                        <label for="charge">节点斥力: <span id="charge-value">-50</span></label>
                        <input type="range" id="charge" min="-200" max="-10" value="-50">
                    </div>
                    <div class="slider-container">
                        <label for="link">连接引力: <span id="link-value">30</span></label>
                        <input type="range" id="link" min="1" max="50" value="30">
                    </div>
                    <div class="slider-container">
                        <label for="radius">圆盘半径: <span id="radius-value">15</span></label>
                        <input type="range" id="radius" min="5" max="30" value="15">
                    </div>
                </div>
                
                <div class="control-group">
                    <h2>图例</h2>
                    <div class="legend">
                        <div class="legend-item">
                            <div class="legend-color" style="background-color: #ff7e5f;"></div>
                            <span>核心节点</span>
                        </div>
                        <div class="legend-item">
                            <div class="legend-color" style="background-color: #6ccefc;"></div>
                            <span>边缘节点</span>
                        </div>
                    </div>
                </div>
                
                <div class="info-panel">
                    <p>节点数量: <span id="node-count">30</span></p>
                    <p>连接数量: <span id="link-count">45</span></p>
                    <p>布局状态: <span id="layout-status">就绪</span></p>
                </div>
                
                <div class="instructions">
                    <p>使用鼠标拖动可以旋转场景</p>
                    <p>滚轮可以缩放场景</p>
                    <p>布局完成后节点位置将固定</p>
                </div>
            </div>
        </div>
    </div>

    <script>
        // 全局变量
        let scene, camera, renderer, controls;
        let nodes = [], links = [];
        let nodeMeshes = [], linkMeshes = [];
        let simulation;
        const radius = 15; // 圆盘半径
        const nodeCount = 30;
        const linkCount = 45;

        // 初始化Three.js场景
        function initThreeJS() {
            // 创建场景
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x0b132b);

            // 创建相机
            camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(0, 30, 40);
            camera.lookAt(0, 0, 0);

            // 创建渲染器
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth - 250, window.innerHeight - 80);
            renderer.shadowMap.enabled = true;
            document.getElementById('canvas-container').appendChild(renderer.domElement);

            // 添加光源
            const ambientLight = new THREE.AmbientLight(0x404040);
            scene.add(ambientLight);

            const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
            directionalLight.position.set(0, 1, 0);
            directionalLight.castShadow = true;
            scene.add(directionalLight);

            const backLight = new THREE.DirectionalLight(0x222255, 0.5);
            backLight.position.set(0, 0, -1);
            scene.add(backLight);

            // 添加圆盘
            const diskGeometry = new THREE.CircleGeometry(radius, 32);
            const diskMaterial = new THREE.MeshPhongMaterial({ 
                color: 0x16213e, 
                side: THREE.DoubleSide,
                transparent: true,
                opacity: 0.3
            });
            const disk = new THREE.Mesh(diskGeometry, diskMaterial);
            disk.rotation.x = -Math.PI / 2;
            disk.position.y = -0.1;
            scene.add(disk);

            // 添加圆盘边缘
            const edgeGeometry = new THREE.RingGeometry(radius - 0.05, radius, 32);
            const edgeMaterial = new THREE.MeshBasicMaterial({ 
                color: 0x6ccefc, 
                side: THREE.DoubleSide,
                transparent: true,
                opacity: 0.5
            });
            const edge = new THREE.Mesh(edgeGeometry, edgeMaterial);
            edge.rotation.x = -Math.PI / 2;
            edge.position.y = -0.09;
            scene.add(edge);

            // 添加坐标轴辅助
            const axesHelper = new THREE.AxesHelper(10);
            scene.add(axesHelper);

            // 初始化轨道控制器
            controls = new THREE.OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;
            controls.dampingFactor = 0.05;

            // 响应窗口大小变化
            window.addEventListener('resize', onWindowResize);
        }

        // 窗口大小变化处理
        function onWindowResize() {
            camera.aspect = (window.innerWidth - 250) / (window.innerHeight - 80);
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth - 250, window.innerHeight - 80);
        }

        // 生成随机节点数据
        function generateData() {
            nodes = [];
            links = [];
            
            // 创建节点
            for (let i = 0; i < nodeCount; i++) {
                const type = i < 5 ? 'core' : 'edge'; // 前5个为核心节点
                const color = type === 'core' ? 0xff7e5f : 0x6ccefc;
                const size = type === 'core' ? 1.0 : 0.6;
                
                nodes.push({
                    id: i,
                    type: type,
                    color: color,
                    size: size
                });
            }
            
            // 创建连接（确保所有节点都连接）
            for (let i = 1; i < nodes.length; i++) {
                // 每个节点至少有一个连接
                const source = i;
                const target = Math.floor(Math.random() * i);
                links.push({ source: source, target: target });
            }
            
            // 添加额外的随机连接
            while (links.length < linkCount) {
                const source = Math.floor(Math.random() * nodes.length);
                let target = Math.floor(Math.random() * nodes.length);
                if (source !== target) {
                    // 检查连接是否已存在
                    const linkExists = links.some(link => 
                        (link.source === source && link.target === target) ||
                        (link.source === target && link.target === source)
                    );
                    if (!linkExists) {
                        links.push({ source: source, target: target });
                    }
                }
            }
            
            // 更新UI计数
            document.getElementById('node-count').textContent = nodes.length;
            document.getElementById('link-count').textContent = links.length;
        }

        // 创建节点和连接的3D对象
        function createGraphObjects() {
            // 清除现有对象
            nodeMeshes.forEach(mesh => scene.remove(mesh));
            linkMeshes.forEach(mesh => scene.remove(mesh));
            nodeMeshes = [];
            linkMeshes = [];
            
            // 创建节点
            nodes.forEach(node => {
                const geometry = new THREE.SphereGeometry(node.size, 16, 16);
                const material = new THREE.MeshPhongMaterial({ color: node.color });
                const sphere = new THREE.Mesh(geometry, material);
                sphere.position.set(
                    (Math.random() - 0.5) * radius * 1.5,
                    0,
                    (Math.random() - 0.5) * radius * 1.5
                );
                sphere.castShadow = true;
                sphere.receiveShadow = true;
                sphere.userData = { node: node };
                scene.add(sphere);
                nodeMeshes.push(sphere);
                node.mesh = sphere;
            });
            
            // 创建连接
            links.forEach(link => {
                const sourceNode = nodes[link.source];
                const targetNode = nodes[link.target];
                
                if (sourceNode && targetNode && sourceNode.mesh && targetNode.mesh) {
                    const sourcePos = sourceNode.mesh.position;
                    const targetPos = targetNode.mesh.position;
                    
                    const distance = sourcePos.distanceTo(targetPos);
                    const cylinderGeometry = new THREE.CylinderGeometry(0.1, 0.1, distance, 8);
                    cylinderGeometry.translate(0, distance/2, 0);
                    cylinderGeometry.rotateZ(Math.PI/2);
                    
                    const material = new THREE.MeshPhongMaterial({ 
                        color: 0x3498db,
                        transparent: true,
                        opacity: 0.6
                    });
                    
                    const cylinder = new THREE.Mesh(cylinderGeometry, material);
                    
                    // 设置圆柱体的位置和方向
                    const center = new THREE.Vector3()
                        .addVectors(sourcePos, targetPos)
                        .multiplyScalar(0.5);
                    cylinder.position.copy(center);
                    
                    const direction = new THREE.Vector3()
                        .subVectors(targetPos, sourcePos)
                        .normalize();
                    const axis = new THREE.Vector3(0, 1, 0).cross(direction);
                    const angle = Math.acos(new THREE.Vector3(0, 1, 0).dot(direction));
                    cylinder.quaternion.setFromAxisAngle(axis, angle);
                    
                    scene.add(cylinder);
                    linkMeshes.push(cylinder);
                    link.mesh = cylinder;
                }
            });
        }

        // 初始化力导向模拟
        function initSimulation() {
            // 清除现有模拟
            if (simulation) {
                simulation.stop();
            }
            
            // 创建力导向模拟
            simulation = d3.forceSimulation(nodes)
                .force("charge", d3.forceManyBody().strength(-50))
                .force("link", d3.forceLink(links).id(d => d.id).distance(30))
                .force("center", d3.forceCenter(0, 0))
                .force("collision", d3.forceCollide().radius(d => d.size + 0.5))
                .force("radial", radialForce(radius))
                .on("tick", ticked);
                
            document.getElementById('layout-status').textContent = '布局中...';
            
            // 添加径向力，将节点限制在圆盘内
            function radialForce(radius) {
                let nodes;
                function force() {
                    const center = { x: 0, z: 0 };
                    nodes.forEach(node => {
                        const dx = node.x - center.x;
                        const dz = node.z - center.z;
                        const dist = Math.sqrt(dx * dx + dz * dz);
                        const r = radius - node.size;
                        
                        if (dist > r) {
                            node.x = center.x + dx * r / dist;
                            node.z = center.z + dz * r / dist;
                        }
                    });
                }
                force.initialize = function(_) {
                    nodes = _;
                };
                return force;
            }
        }

        // 力导向模拟的每一帧更新
        function ticked() {
            // 更新节点位置
            nodes.forEach(node => {
                if (node.mesh) {
                    node.mesh.position.x = node.x || 0;
                    node.mesh.position.z = node.z || 0;
                    // 保持y坐标为0（水平面）
                    node.mesh.position.y = 0;
                }
            });
            
            // 更新连接位置
            links.forEach(link => {
                if (link.mesh && nodes[link.source] && nodes[link.target]) {
                    const sourcePos = nodes[link.source].mesh.position;
                    const targetPos = nodes[link.target].mesh.position;
                    
                    const distance = sourcePos.distanceTo(targetPos);
                    const center = new THREE.Vector3()
                        .addVectors(sourcePos, targetPos)
                        .multiplyScalar(0.5);
                    
                    link.mesh.position.copy(center);
                    
                    const direction = new THREE.Vector3()
                        .subVectors(targetPos, sourcePos)
                        .normalize();
                    
                    const axis = new THREE.Vector3(0, 1, 0).cross(direction);
                    const angle = Math.acos(new THREE.Vector3(0, 1, 0).dot(direction));
                    link.mesh.quaternion.setFromAxisAngle(axis, angle);
                    
                    // 更新圆柱体高度
                    link.mesh.scale.y = distance;
                    link.mesh.geometry = new THREE.CylinderGeometry(0.1, 0.1, distance, 8);
                    link.mesh.geometry.translate(0, distance/2, 0);
                    link.mesh.geometry.rotateZ(Math.PI/2);
                }
            });
        }

        // 动画循环
        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
        }

        // 初始化
        function init() {
            initThreeJS();
            generateData();
            createGraphObjects();
            
            // 添加事件监听器
            document.getElementById('layout-btn').addEventListener('click', function() {
                initSimulation();
                
                // 10秒后停止布局
                setTimeout(() => {
                    if (simulation) {
                        simulation.stop();
                        document.getElementById('layout-status').textContent = '布局完成';
                    }
                }, 10000);
            });
            
            document.getElementById('reset-btn').addEventListener('click', function() {
                generateData();
                createGraphObjects();
                document.getElementById('layout-status').textContent = '就绪';
            });
            
            document.getElementById('charge').addEventListener('input', function() {
                const value = parseInt(this.value);
                document.getElementById('charge-value').textContent = value;
                if (simulation) {
                    simulation.force("charge").strength(value);
                    simulation.alpha(0.5).restart();
                }
            });
            
            document.getElementById('link').addEventListener('input', function() {
                const value = parseInt(this.value);
                document.getElementById('link-value').textContent = value;
                if (simulation) {
                    simulation.force("link").distance(value);
                    simulation.alpha(0.5).restart();
                }
            });
            
            document.getElementById('radius').addEventListener('input', function() {
                const value = parseInt(this.value);
                document.getElementById('radius-value').textContent = value;
                if (simulation) {
                    simulation.force("radial", radialForce(value));
                    simulation.alpha(0.5).restart();
                }
            });
            
            // 开始动画
            animate();
        }

        // 启动应用
        init();
    </script>
</body>
</html>