<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Three.js 力导向拓扑图</title>
    <script src="https://cdn.jsdelivr.net/npm/three@0.132.2/build/three.min.js"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
            background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
            color: #e6e6e6;
        }
        #container {
            position: absolute;
            width: 100%;
            height: 100%;
        }
        #ui {
            position: absolute;
            top: 20px;
            right: 20px;
            width: 300px;
            background: rgba(26, 26, 46, 0.8);
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 4px 15px rgba(0, 0, 0, 0.5);
            backdrop-filter: blur(10px);
        }
        h1 {
            font-size: 1.5rem;
            margin-bottom: 15px;
            color: #4cc9f0;
            text-align: center;
        }
        .control-group {
            margin-bottom: 15px;
        }
        label {
            display: block;
            margin-bottom: 5px;
            font-weight: 500;
        }
        input[type="range"] {
            width: 100%;
            height: 5px;
            -webkit-appearance: none;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 5px;
            outline: none;
        }
        input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            width: 15px;
            height: 15px;
            border-radius: 50%;
            background: #4cc9f0;
            cursor: pointer;
        }
        .value-display {
            display: flex;
            justify-content: space-between;
            font-size: 0.8rem;
            color: #a0a0a0;
        }
        button {
            width: 100%;
            padding: 10px;
            background: #4361ee;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-weight: bold;
            transition: background 0.3s;
        }
        button:hover {
            background: #3a56d4;
        }
        .instructions {
            margin-top: 15px;
            font-size: 0.85rem;
            color: #a0a0a0;
            line-height: 1.4;
        }
        #status {
            position: absolute;
            bottom: 20px;
            left: 20px;
            background: rgba(26, 26, 46, 0.8);
            padding: 10px 15px;
            border-radius: 5px;
            font-size: 0.9rem;
            backdrop-filter: blur(10px);
        }
    </style>
</head>
<body>
    <div id="container"></div>
    
    <div id="ui">
        <h1>拓扑图力导向布局</h1>
        
        <div class="control-group">
            <label for="repulsion">节点斥力</label>
            <input type="range" id="repulsion" min="1" max="100" value="30">
            <div class="value-display">
                <span>弱</span>
                <span>强</span>
            </div>
        </div>
        
        <div class="control-group">
            <label for="attraction">连接引力</label>
            <input type="range" id="attraction" min="1" max="100" value="20">
            <div class="value-display">
                <span>弱</span>
                <span>强</span>
            </div>
        </div>
        
        <div class="control-group">
            <label for="velocity">速度阻尼</label>
            <input type="range" id="velocity" min="1" max="100" value="85">
            <div class="value-display">
                <span>快</span>
                <span>慢</span>
            </div>
        </div>
        
        <button id="reset">重置布局</button>
        
        <div class="instructions">
            <p>• 鼠标拖动：旋转视图</p>
            <p>• 鼠标滚轮：缩放视图</p>
            <p>• 拖动节点：手动调整位置</p>
        </div>
    </div>
    
    <div id="status">布局稳定性: 计算中...</div>

    <script>
        // 主要变量
        let scene, camera, renderer;
        let nodes = [], edges = [];
        let nodeMeshes = [], edgeMeshes = [];
        let selectedNode = null;
        
        // 力导向参数
        let params = {
            repulsion: 30,
            attraction: 20,
            damping: 0.85
        };
        
        // 初始化场景
        function init() {
            // 创建场景
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x0f172a);
            
            // 创建相机
            camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.z = 50;
            
            // 创建渲染器
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.getElementById('container').appendChild(renderer.domElement);
            
            // 添加光源
            const ambientLight = new THREE.AmbientLight(0x444444);
            scene.add(ambientLight);
            
            const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
            directionalLight.position.set(1, 1, 1).normalize();
            scene.add(directionalLight);
            
            // 创建网格地面
            const gridHelper = new THREE.GridHelper(40, 20, 0x3d348b, 0x3d348b);
            gridHelper.rotation.x = Math.PI / 2;
            scene.add(gridHelper);
            
            // 创建坐标轴
            const axesHelper = new THREE.AxesHelper(10);
            scene.add(axesHelper);
            
            // 生成随机拓扑图
            generateGraph(15);
            
            // 添加事件监听器
            setupEventListeners();
            
            // 开始动画循环
            animate();
        }
        
        // 生成随机拓扑图
        function generateGraph(nodeCount) {
            // 清除现有节点和边
            nodeMeshes.forEach(mesh => scene.remove(mesh));
            edgeMeshes.forEach(mesh => scene.remove(mesh));
            nodes = [];
            edges = [];
            nodeMeshes = [];
            edgeMeshes = [];
            
            // 创建节点
            for (let i = 0; i < nodeCount; i++) {
                const node = {
                    id: i,
                    x: Math.random() * 30 - 15,
                    y: Math.random() * 30 - 15,
                    z: 0,
                    vx: 0,
                    vy: 0,
                    vz: 0
                };
                nodes.push(node);
                
                // 创建球体表示节点
                const geometry = new THREE.SphereGeometry(0.8, 16, 16);
                const material = new THREE.MeshPhongMaterial({ 
                    color: new THREE.Color().setHSL(Math.random(), 0.8, 0.6),
                    emissive: new THREE.Color().setHSL(Math.random(), 0.8, 0.1)
                });
                const sphere = new THREE.Mesh(geometry, material);
                sphere.position.set(node.x, node.y, node.z);
                sphere.userData.node = node;
                scene.add(sphere);
                nodeMeshes.push(sphere);
            }
            
            // 创建连接（边）
            for (let i = 0; i < nodeCount; i++) {
                // 每个节点随机连接1-3个其他节点
                const connections = Math.floor(Math.random() * 3) + 1;
                for (let j = 0; j < connections; j++) {
                    const target = Math.floor(Math.random() * nodeCount);
                    if (target !== i && !edges.find(e => 
                        (e.source === i && e.target === target) || 
                        (e.source === target && e.target === i))) {
                        edges.push({ source: i, target: target });
                        
                        // 创建圆柱体表示边
                        const sourceNode = nodes[i];
                        const targetNode = nodes[target];
                        
                        const geometry = new THREE.CylinderGeometry(0.2, 0.2, 1, 8);
                        geometry.translate(0, 0.5, 0);
                        geometry.rotateX(Math.PI / 2);
                        
                        const material = new THREE.MeshPhongMaterial({ 
                            color: 0x4361ee,
                            transparent: true,
                            opacity: 0.6
                        });
                        
                        const cylinder = new THREE.Mesh(geometry, material);
                        updateEdgePosition(cylinder, sourceNode, targetNode);
                        scene.add(cylinder);
                        edgeMeshes.push(cylinder);
                    }
                }
            }
        }
        
        // 更新边的位置
        function updateEdgePosition(cylinder, source, target) {
            // 计算中点
            cylinder.position.set(
                (source.x + target.x) / 2,
                (source.y + target.y) / 2,
                (source.z + target.z) / 2
            );
            
            // 计算方向
            const dir = new THREE.Vector3(
                target.x - source.x,
                target.y - source.y,
                target.z - source.z
            );
            
            // 计算长度
            const length = dir.length();
            cylinder.scale.y = length;
            
            // 计算旋转
            cylinder.rotation.z = Math.atan2(dir.y, dir.x);
            cylinder.rotation.y = -Math.atan2(dir.z, Math.sqrt(dir.x * dir.x + dir.y * dir.y));
        }
        
        // 力导向布局算法
        function forceDirectedLayout() {
            // 计算节点间的斥力
            for (let i = 0; i < nodes.length; i++) {
                for (let j = i + 1; j < nodes.length; j++) {
                    const node1 = nodes[i];
                    const node2 = nodes[j];
                    
                    const dx = node1.x - node2.x;
                    const dy = node1.y - node2.y;
                    const dz = node1.z - node2.z;
                    
                    const distance = Math.max(0.1, Math.sqrt(dx * dx + dy * dy + dz * dz));
                    
                    // 库仑斥力
                    const force = params.repulsion / (distance * distance);
                    
                    const fx = force * dx / distance;
                    const fy = force * dy / distance;
                    const fz = force * dz / distance;
                    
                    if (!node1.fixed) {
                        node1.vx += fx;
                        node1.vy += fy;
                        node1.vz += fz;
                    }
                    
                    if (!node2.fixed) {
                        node2.vx -= fx;
                        node2.vy -= fy;
                        node2.vz -= fz;
                    }
                }
            }
            
            // 计算边的引力
            for (const edge of edges) {
                const source = nodes[edge.source];
                const target = nodes[edge.target];
                
                const dx = source.x - target.x;
                const dy = source.y - target.y;
                const dz = source.z - target.z;
                
                const distance = Math.max(0.1, Math.sqrt(dx * dx + dy * dy + dz * dz));
                
                // 胡克定律引力
                const force = params.attraction * Math.log(distance / 5);
                
                const fx = force * dx / distance;
                const fy = force * dy / distance;
                const fz = force * dz / distance;
                
                if (!source.fixed) {
                    source.vx -= fx;
                    source.vy -= fy;
                    source.vz -= fz;
                }
                
                if (!target.fixed) {
                    target.vx += fx;
                    target.vy += fy;
                    target.vz += fz;
                }
            }
            
            // 应用速度并限制在边界内
            let totalMovement = 0;
            for (const node of nodes) {
                if (node.fixed) continue;
                
                // 应用阻尼
                node.vx *= params.damping;
                node.vy *= params.damping;
                node.vz *= params.damping;
                
                // 更新位置
                node.x += node.vx * 0.1;
                node.y += node.vy * 0.1;
                node.z += node.vz * 0.1;
                
                // 限制在边界内
                const boundary = 20;
                node.x = Math.max(-boundary, Math.min(boundary, node.x));
                node.y = Math.max(-boundary, Math.min(boundary, node.y));
                node.z = Math.max(-boundary, Math.min(boundary, node.z));
                
                // 计算总移动距离（用于稳定性检测）
                totalMovement += Math.abs(node.vx) + Math.abs(node.vy) + Math.abs(node.vz);
            }
            
            // 更新状态显示
            const stability = Math.max(0, 100 - totalMovement).toFixed(1);
            document.getElementById('status').textContent = `布局稳定性: ${stability}%`;
            
            // 更新Three.js对象位置
            for (let i = 0; i < nodes.length; i++) {
                const node = nodes[i];
                const mesh = nodeMeshes[i];
                mesh.position.set(node.x, node.y, node.z);
            }
            
            // 更新边的位置
            for (let i = 0; i < edges.length; i++) {
                const edge = edges[i];
                const source = nodes[edge.source];
                const target = nodes[edge.target];
                updateEdgePosition(edgeMeshes[i], source, target);
            }
        }
        
        // 设置事件监听器
        function setupEventListeners() {
            // 鼠标拖动旋转场景
            let isDragging = false;
            let previousMousePosition = {
                x: 0,
                y: 0
            };
            
            renderer.domElement.addEventListener('mousedown', (e) => {
                isDragging = true;
                
                // 检查是否点击了节点
                const mouse = new THREE.Vector2(
                    (e.clientX / window.innerWidth) * 2 - 1,
                    -(e.clientY / window.innerHeight) * 2 + 1
                );
                
                const raycaster = new THREE.Raycaster();
                raycaster.setFromCamera(mouse, camera);
                
                const intersects = raycaster.intersectObjects(nodeMeshes);
                if (intersects.length > 0) {
                    selectedNode = intersects[0].object.userData.node;
                    selectedNode.fixed = true;
                }
            });
            
            renderer.domElement.addEventListener('mousemove', (e) => {
                const deltaMove = {
                    x: e.offsetX - previousMousePosition.x,
                    y: e.offsetY - previousMousePosition.y
                };
                
                if (isDragging) {
                    if (selectedNode) {
                        // 拖动节点
                        const mouse = new THREE.Vector2(
                            (e.clientX / window.innerWidth) * 2 - 1,
                            -(e.clientY / window.innerHeight) * 2 + 1
                        );
                        
                        const raycaster = new THREE.Raycaster();
                        raycaster.setFromCamera(mouse, camera);
                        
                        const plane = new THREE.Plane(new THREE.Vector3(0, 0, 1), 0);
                        const intersection = new THREE.Vector3();
                        raycaster.ray.intersectPlane(plane, intersection);
                        
                        selectedNode.x = intersection.x;
                        selectedNode.y = intersection.y;
                    } else {
                        // 旋转场景
                        camera.position.x -= deltaMove.x * 0.01;
                        camera.position.y += deltaMove.y * 0.01;
                        camera.lookAt(scene.position);
                    }
                }
                
                previousMousePosition = {
                    x: e.offsetX,
                    y: e.offsetY
                };
            });
            
            renderer.domElement.addEventListener('mouseup', () => {
                isDragging = false;
                if (selectedNode) {
                    selectedNode.fixed = false;
                    selectedNode = null;
                }
            });
            
            // 鼠标滚轮缩放
            renderer.domElement.addEventListener('wheel', (e) => {
                camera.position.z += e.deltaY * 0.05;
                camera.position.z = Math.max(10, Math.min(100, camera.position.z));
            });
            
            // 窗口大小调整
            window.addEventListener('resize', () => {
                camera.aspect = window.innerWidth / window.innerHeight;
                camera.updateProjectionMatrix();
                renderer.setSize(window.innerWidth, window.innerHeight);
            });
            
            // 参数控制
            document.getElementById('repulsion').addEventListener('input', (e) => {
                params.repulsion = parseInt(e.target.value);
            });
            
            document.getElementById('attraction').addEventListener('input', (e) => {
                params.attraction = parseInt(e.target.value) / 10;
            });
            
            document.getElementById('velocity').addEventListener('input', (e) => {
                params.damping = parseInt(e.target.value) / 100;
            });
            
            document.getElementById('reset').addEventListener('click', () => {
                generateGraph(15);
            });
        }
        
        // 动画循环
        function animate() {
            requestAnimationFrame(animate);
            
            forceDirectedLayout();
            
            renderer.render(scene, camera);
        }
        
        // 启动应用
        init();
    </script>
</body>
</html>