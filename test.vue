<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Three.js 折线流动动画</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #000;
        }
        canvas {
            display: block;
        }
        #info {
            position: absolute;
            top: 10px;
            width: 100%;
            text-align: center;
            color: white;
            font-family: Arial, sans-serif;
            pointer-events: none;
        }
    </style>
</head>
<body>
    <div id="info">Three.js 折线流动动画 - 水滴效果</div>
    <script src="https://cdn.jsdelivr.net/npm/three@0.132.2/build/three.min.js"></script>
    <script>
        // 初始化场景
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        // 添加环境光
        const ambientLight = new THREE.AmbientLight(0x404040);
        scene.add(ambientLight);

        // 添加方向光
        const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
        directionalLight.position.set(1, 1, 1);
        scene.add(directionalLight);

        // 创建节点
        const nodes = [];
        const nodeGeometry = new THREE.SphereGeometry(0.3, 16, 16);
        const nodeMaterial = new THREE.MeshPhongMaterial({ color: 0x00ff00 });
        
        // 随机生成节点位置
        for (let i = 0; i < 8; i++) {
            const node = new THREE.Mesh(nodeGeometry, nodeMaterial);
            node.position.set(
                (Math.random() - 0.5) * 10,
                (Math.random() - 0.5) * 10,
                (Math.random() - 0.5) * 10
            );
            nodes.push(node);
            scene.add(node);
        }

        // 创建折线
        const polylines = [];
        
        // 创建几条示例折线
        const line1 = createPolyline([
            nodes[0].position,
            nodes[1].position,
            nodes[2].position
        ]);
        
        const line2 = createPolyline([
            nodes[3].position,
            nodes[4].position,
            nodes[5].position,
            nodes[6].position
        ]);
        
        const line3 = createPolyline([
            nodes[1].position,
            nodes[3].position,
            nodes[7].position
        ]);

        // 创建水滴
        const droplets = [];
        
        // 为每条折线创建水滴
        createDroplet(line1);
        createDroplet(line2);
        createDroplet(line3);

        // 设置相机位置
        camera.position.z = 15;

        // 动画循环
        function animate() {
            requestAnimationFrame(animate);
            
            // 更新水滴位置
            updateDroplets();
            
            renderer.render(scene, camera);
        }
        animate();

        // 窗口大小调整
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        // 创建折线函数
        function createPolyline(points) {
            const lineGeometry = new THREE.BufferGeometry();
            const lineMaterial = new THREE.LineBasicMaterial({ color: 0xffffff });
            
            // 创建折线顶点
            const vertices = [];
            for (let i = 0; i < points.length; i++) {
                vertices.push(points[i].x, points[i].y, points[i].z);
            }
            
            lineGeometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
            const line = new THREE.Line(lineGeometry, lineMaterial);
            scene.add(line);
            
            // 存储折线信息
            const polyline = {
                points: points,
                line: line,
                segments: []
            };
            
            // 计算折线段的长度和方向
            for (let i = 0; i < points.length - 1; i++) {
                const start = points[i];
                const end = points[i + 1];
                const length = start.distanceTo(end);
                const direction = new THREE.Vector3().subVectors(end, start).normalize();
                
                polyline.segments.push({
                    start: start,
                    end: end,
                    length: length,
                    direction: direction
                });
            }
            
            polylines.push(polyline);
            return polyline;
        }

        // 创建水滴函数
        function createDroplet(polyline) {
            const dropletGeometry = new THREE.SphereGeometry(0.1, 8, 8);
            const dropletMaterial = new THREE.MeshPhongMaterial({ 
                color: 0x00aaff,
                transparent: true,
                opacity: 0.8
            });
            
            const droplet = new THREE.Mesh(dropletGeometry, dropletMaterial);
            scene.add(droplet);
            
            // 存储水滴信息
            const dropletInfo = {
                mesh: droplet,
                polyline: polyline,
                progress: 0, // 在整条折线上的进度 (0-1)
                segmentIndex: 0, // 当前所在的线段索引
                segmentProgress: 0 // 在当前线段上的进度 (0-1)
            };
            
            droplets.push(dropletInfo);
            return dropletInfo;
        }

        // 更新水滴位置
        function updateDroplets() {
            const speed = 0.005; // 水滴移动速度
            
            droplets.forEach(droplet => {
                // 更新进度
                droplet.progress += speed;
                
                // 如果进度超过1，重置
                if (droplet.progress > 1) {
                    droplet.progress = 0;
                    droplet.segmentIndex = 0;
                    droplet.segmentProgress = 0;
                }
                
                // 计算当前线段和在线段上的进度
                let accumulatedLength = 0;
                let totalLength = 0;
                
                // 计算折线总长度
                droplet.polyline.segments.forEach(segment => {
                    totalLength += segment.length;
                });
                
                // 计算当前线段
                let currentLength = droplet.progress * totalLength;
                
                for (let i = 0; i < droplet.polyline.segments.length; i++) {
                    const segment = droplet.polyline.segments[i];
                    
                    if (currentLength <= accumulatedLength + segment.length) {
                        droplet.segmentIndex = i;
                        droplet.segmentProgress = (currentLength - accumulatedLength) / segment.length;
                        break;
                    }
                    
                    accumulatedLength += segment.length;
                }
                
                // 获取当前线段
                const currentSegment = droplet.polyline.segments[droplet.segmentIndex];
                
                // 计算水滴位置
                const position = new THREE.Vector3()
                    .copy(currentSegment.start)
                    .lerp(currentSegment.end, droplet.segmentProgress);
                
                droplet.mesh.position.copy(position);
                
                // 计算水滴方向（朝向当前线段方向）
                if (droplet.segmentProgress < 0.99) {
                    droplet.mesh.lookAt(
                        position.x + currentSegment.direction.x,
                        position.y + currentSegment.direction.y,
                        position.z + currentSegment.direction.z
                    );
                }
            });
        }
    </script>
</body>
</html>