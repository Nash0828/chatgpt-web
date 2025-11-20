class ForceDirectedGraph2D {
    constructor(container) {
        this.container = container;
        this.scene = new THREE.Scene();
        this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        this.renderer = new THREE.WebGLRenderer({ antialias: true });
        
        this.nodes = new Map();
        this.links = [];
        this.nodeMeshes = [];
        this.linkLines = [];
        
        // 稳定性控制
        this.stableIterations = 0;
        this.stableThreshold = 30;
        this.isStable = false;
        
        // 矩形区域约束
        this.boundary = {
            x: [-25, 25],   // x轴范围 [-25, 25]
            y: [-15, 15],   // y轴范围 [-15, 15]
            z: 0            // 所有节点z坐标固定为0
        };
        
        this.init();
    }
    
    init() {
        this.renderer.setSize(window.innerWidth, window.innerHeight);
        this.renderer.setClearColor(0x0f172a);
        this.container.appendChild(this.renderer.domElement);
        
        this.camera.position.z = 40;
        
        // 轨道控制器
        this.controls = new OrbitControls(this.camera, this.renderer.domElement);
        
        // 环境光
        const ambientLight = new THREE.AmbientLight(0x404040);
        this.scene.add(ambientLight);
        
        // 方向光
        const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
        directionalLight.position.set(1, 1, 1);
        this.scene.add(directionalLight);
        
        // 加载数据
        this.loadDataFromAPI();
        this.animate();
    }
    
    // 二维确定性位置生成
    getDeterministicPosition(nodeId) {
        const numericValue = this.stringToHash(nodeId);
        
        // 在矩形区域内生成位置
        const width = this.boundary.x[1] - this.boundary.x[0];
        const height = this.boundary.y[1] - this.boundary.y[0];
        
        const x = this.boundary.x[0] + (numericValue * 123.45) % width;
        const y = this.boundary.y[0] + (numericValue * 67.89) % height;
        
        return new THREE.Vector3(x, y, this.boundary.z);
    }
    
    stringToHash(str) {
        let hash = 0;
        for (let i = 0; i < str.length; i++) {
            const char = str.charCodeAt(i);
            hash = ((hash << 5) - hash) + char;
            hash = hash & hash;
        }
        return Math.abs(hash);
    }
    
    addNode(nodeData) {
        const position = this.getDeterministicPosition(nodeData.id);
        
        const geometry = new THREE.SphereGeometry(0.8, 16, 16);
        const material = new THREE.MeshPhongMaterial({ 
            color: this.getNodeColor(nodeData.type) 
        });
        const mesh = new THREE.Mesh(geometry, material);
        mesh.position.copy(position);
        
        this.nodes.set(nodeData.id, {
            mesh,
            position: position.clone(),
            velocity: new THREE.Vector3(0, 0, 0),
            data: nodeData
        });
        
        this.nodeMeshes.push(mesh);
        this.scene.add(mesh);
    }
    
    addLink(sourceId, targetId) {
        const sourceNode = this.nodes.get(sourceId);
        const targetNode = this.nodes.get(targetId);
        
        if (!sourceNode || !targetNode) return;
        
        const geometry = new THREE.BufferGeometry();
        const material = new THREE.LineBasicMaterial({ 
            color: 0x666666,
            transparent: true,
            opacity: 0.6
        });
        
        const points = [sourceNode.position, targetNode.position];
        geometry.setFromPoints(points);
        
        const line = new THREE.Line(geometry, material);
        
        this.links.push({
            line,
            source: sourceId,
            target: targetId
        });
        
        this.scene.add(line);
    }
    
    updatePhysics() {
        if (this.isStable) {
            // 稳定后强制约束到矩形区域
            this.applyBoundaryConstraint();
            return;
        }
        
        const repulsionForce = 80;
        const attractionForce = 0.08;
        const damping = 0.85;
        const timeStep = 0.016;
        
        const nodeArray = Array.from(this.nodes.values());
        
        // 1. 节点间斥力（二维）
        for (let i = 0; i < nodeArray.length; i++) {
            for (let j = i + 1; j < nodeArray.length; j++) {
                const nodeA = nodeArray[i];
                const nodeB = nodeArray[j];
                
                const delta = new THREE.Vector3()
                    .subVectors(nodeA.position, nodeB.position);
                
                // 忽略z轴，只计算二维距离
                delta.z = 0;
                const distance = delta.length();
                
                if (distance > 0 && distance < 20) {
                    const force = repulsionForce / (distance * distance + 0.1);
                    delta.normalize().multiplyScalar(force * timeStep);
                    
                    nodeA.velocity.add(delta);
                    nodeB.velocity.sub(delta);
                }
            }
        }
        
        // 2. 连接边引力（二维）
        for (const link of this.links) {
            const sourceNode = this.nodes.get(link.source);
            const targetNode = this.nodes.get(link.target);
            
            if (!sourceNode || !targetNode) continue;
            
            const delta = new THREE.Vector3()
                .subVectors(targetNode.position, sourceNode.position);
            
            // 忽略z轴
            delta.z = 0;
            const distance = delta.length();
            
            if (distance > 0) {
                const force = attractionForce * Math.min(distance, 8);
                delta.normalize().multiplyScalar(force * timeStep);
                
                sourceNode.velocity.add(delta);
                targetNode.velocity.sub(delta);
            }
        }
        
        // 3. 中心引力（帮助节点向中心聚集）
        for (const node of nodeArray) {
            const toCenter = new THREE.Vector3()
                .subVectors(new THREE.Vector3(0, 0, 0), node.position);
            toCenter.z = 0; // 忽略z轴
            
            const distanceToCenter = toCenter.length();
            if (distanceToCenter > 10) {
                const force = 0.02 * distanceToCenter;
                toCenter.normalize().multiplyScalar(force * timeStep);
                node.velocity.add(toCenter);
            }
        }
        
        // 4. 更新位置（保持z坐标为0）
        for (const node of nodeArray) {
            node.velocity.multiplyScalar(damping);
            
            // 限制最大速度
            const speed = node.velocity.length();
            const maxSpeed = 1.5;
            if (speed > maxSpeed) {
                node.velocity.normalize().multiplyScalar(maxSpeed);
            }
            
            node.position.add(node.velocity);
            node.position.z = this.boundary.z; // 强制z坐标为0
            
            node.mesh.position.copy(node.position);
        }
        
        // 5. 边界约束（运算期间较弱）
        if (!this.isStable) {
            this.applySoftBoundaryConstraint();
        }
        
        this.updateLinks();
        
        // 6. 检测稳定性
        if (this.checkStability()) {
            this.stableIterations++;
            if (this.stableIterations >= this.stableThreshold) {
                this.stopPhysics();
            }
        } else {
            this.stableIterations = 0;
        }
    }
    
    // 软边界约束（运算期间使用）
    applySoftBoundaryConstraint() {
        const boundaryForce = 0.3;
        const margin = 2; // 边界裕量
        
        this.nodes.forEach(node => {
            const pos = node.position;
            
            // X轴边界
            if (pos.x < this.boundary.x[0] + margin) {
                node.velocity.x += boundaryForce;
            } else if (pos.x > this.boundary.x[1] - margin) {
                node.velocity.x -= boundaryForce;
            }
            
            // Y轴边界
            if (pos.y < this.boundary.y[0] + margin) {
                node.velocity.y += boundaryForce;
            } else if (pos.y > this.boundary.y[1] - margin) {
                node.velocity.y -= boundaryForce;
            }
        });
    }
    
    // 硬边界约束（稳定后使用）
    applyBoundaryConstraint() {
        this.nodes.forEach(node => {
            const pos = node.position;
            
            // 强制限制在矩形区域内
            pos.x = Math.max(this.boundary.x[0], Math.min(this.boundary.x[1], pos.x));
            pos.y = Math.max(this.boundary.y[0], Math.min(this.boundary.y[1], pos.y));
            pos.z = this.boundary.z;
            
            node.mesh.position.copy(pos);
        });
        
        this.updateLinks();
    }
    
    checkStability() {
        let maxVelocity = 0;
        this.nodes.forEach(node => {
            // 只检查二维速度
            const velocity2D = new THREE.Vector2(node.velocity.x, node.velocity.y);
            maxVelocity = Math.max(maxVelocity, velocity2D.length());
        });
        
        return maxVelocity < 0.1;
    }
    
    stopPhysics() {
        this.isStable = true;
        console.log('二维布局已稳定，应用硬边界约束');
        
        // 应用最终边界约束
        this.applyBoundaryConstraint();
        
        // 速度归零
        this.nodes.forEach(node => {
            node.velocity.set(0, 0, 0);
        });
        
        this.onLayoutStable();
    }
    
    onLayoutStable() {
        console.log('二维力导向布局完成！');
        // 验证所有节点都在矩形区域内
        this.validateBoundary();
    }
    
    validateBoundary() {
        let outOfBoundNodes = 0;
        
        this.nodes.forEach((node, nodeId) => {
            const pos = node.position;
            if (pos.x < this.boundary.x[0] || pos.x > this.boundary.x[1] ||
                pos.y < this.boundary.y[0] || pos.y > this.boundary.y[1]) {
                outOfBoundNodes++;
                console.warn(`节点 ${nodeId} 超出边界: (${pos.x}, ${pos.y})`);
            }
        });
        
        if (outOfBoundNodes === 0) {
            console.log('✅ 所有节点都在矩形区域内');
        } else {
            console.log(`⚠️  ${outOfBoundNodes} 个节点超出边界，已强制约束`);
        }
    }
    
    updateLinks() {
        for (const link of this.links) {
            const sourceNode = this.nodes.get(link.source);
            const targetNode = this.nodes.get(link.target);
            
            if (!sourceNode || !targetNode) continue;
            
            const geometry = link.line.geometry;
            const positions = geometry.attributes.position.array;
            
            positions[0] = sourceNode.position.x;
            positions[1] = sourceNode.position.y;
            positions[2] = sourceNode.position.z;
            
            positions[3] = targetNode.position.x;
            positions[4] = targetNode.position.y;
            positions[5] = targetNode.position.z;
            
            geometry.attributes.position.needsUpdate = true;
        }
    }
    
    getNodeColor(type) {
        const colors = {
            server: 0xff6b6b,
            client: 0x4ecdc4,
            router: 0x45b7d1,
            switch: 0x96ceb4,
            database: 0xfeca57,
            default: 0x778beb
        };
        return colors[type] || colors.default;
    }
    
    // 可视化矩形边界（调试用）
    showBoundary() {
        const width = this.boundary.x[1] - this.boundary.x[0];
        const height = this.boundary.y[1] - this.boundary.y[0];
        
        const geometry = new THREE.PlaneGeometry(width, height);
        const material = new THREE.MeshBasicMaterial({
            color: 0x00ff00,
            transparent: true,
            opacity: 0.1,
            side: THREE.DoubleSide
        });
        
        const plane = new THREE.Mesh(geometry, material);
        plane.position.set(
            (this.boundary.x[0] + this.boundary.x[1]) / 2,
            (this.boundary.y[0] + this.boundary.y[1]) / 2,
            this.boundary.z
        );
        
        this.scene.add(plane);
    }
    
    animate() {
        requestAnimationFrame(() => this.animate());
        this.updatePhysics();
        this.controls.update();
        this.renderer.render(this.scene, this.camera);
    }
    
    // 重新布局
    restartLayout() {
        this.isStable = false;
        this.stableIterations = 0;
        
        // 重置位置到初始状态
        this.nodes.forEach((node, nodeId) => {
            const newPos = this.getDeterministicPosition(nodeId);
            node.position.copy(newPos);
            node.velocity.set(0, 0, 0);
            node.mesh.position.copy(newPos);
        });
        
        this.updateLinks();
        console.log('重新开始二维布局');
    }
}