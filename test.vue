class ForceDirectedGraph3D {
    constructor(container) {
        this.container = container;
        this.scene = new THREE.Scene();
        this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        this.renderer = new THREE.WebGLRenderer({ antialias: true });
        
        // 使用Map来存储节点和边的关系
        this.nodes = new Map(); // key: nodeId, value: { mesh, position, velocity, data }
        this.links = []; // 存储边信息
        this.nodeMeshes = []; // 用于Three.js渲染的网格数组
        this.linkLines = []; // 用于Three.js渲染的线数组
        
        this.init();
    }
    
    // 从接口加载数据
    async loadDataFromAPI() {
        try {
            // 模拟接口数据
            const mockData = {
                nodes: [
                    { id: '550e8400-e29b-41d4-a716-446655440000', name: 'Node A', type: 'server' },
                    { id: '550e8400-e29b-41d4-a716-446655440001', name: 'Node B', type: 'server' },
                    { id: '550e8400-e29b-41d4-a716-446655440002', name: 'Node C', type: 'client' },
                    { id: '550e8400-e29b-41d4-a716-446655440003', name: 'Node D', type: 'client' },
                    { id: '550e8400-e29b-41d4-a716-446655440004', name: 'Node E', type: 'router' },
                    // ... 更多节点
                ],
                links: [
                    { source: '550e8400-e29b-41d4-a716-446655440000', target: '550e8400-e29b-41d4-a716-446655440001' },
                    { source: '550e8400-e29b-41d4-a716-446655440000', target: '550e8400-e29b-41d4-a716-446655440002' },
                    { source: '550e8400-e29b-41d4-a716-446655440001', target: '550e8400-e29b-41d4-a716-446655440003' },
                    { source: '550e8400-e29b-41d4-a716-446655440002', target: '550e8400-e29b-41d4-a716-446655440004' },
                    // ... 更多边
                ]
            };
            
            // 实际使用时替换为：
            // const response = await fetch('/api/topology');
            // const data = await response.json();
            
            this.buildGraph(mockData);
            
        } catch (error) {
            console.error('Failed to load data:', error);
        }
    }
    
    // 构建图结构
    buildGraph(data) {
        // 清空现有图形
        this.clearGraph();
        
        // 创建节点
        data.nodes.forEach(nodeData => {
            this.addNode(nodeData);
        });
        
        // 创建边
        data.links.forEach(linkData => {
            this.addLink(linkData.source, linkData.target);
        });
    }
    
    // 添加节点
    addNode(nodeData) {
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 40,
            (Math.random() - 0.5) * 40,
            (Math.random() - 0.5) * 40
        );
        
        // 根据节点类型设置不同颜色
        const color = this.getNodeColor(nodeData.type);
        const geometry = new THREE.SphereGeometry(0.8, 16, 16);
        const material = new THREE.MeshPhongMaterial({ color });
        const mesh = new THREE.Mesh(geometry, material);
        mesh.position.copy(position);
        
        // 存储节点信息
        this.nodes.set(nodeData.id, {
            mesh,
            position,
            velocity: new THREE.Vector3(0, 0, 0),
            data: nodeData
        });
        
        this.nodeMeshes.push(mesh);
        this.scene.add(mesh);
    }
    
    // 根据节点类型获取颜色
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
    
    // 添加边
    addLink(sourceId, targetId) {
        const sourceNode = this.nodes.get(sourceId);
        const targetNode = this.nodes.get(targetId);
        
        if (!sourceNode || !targetNode) {
            console.warn(`Cannot create link: ${sourceId} -> ${targetId}, node not found`);
            return;
        }
        
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
        
        this.linkLines.push(line);
        this.scene.add(line);
    }
    
    // 更新物理模拟
    updatePhysics() {
        const repulsionForce = 100;
        const attractionForce = 0.1;
        const centerForce = 0.03;
        const damping = 0.85;
        const timeStep = 0.016;
        
        const boundaryWidth = 35;
        const boundaryHeight = 20;
        const boundaryDepth = 15;
        
        // 将节点转换为数组用于计算
        const nodeArray = Array.from(this.nodes.values());
        
        // 1. 节点间斥力
        for (let i = 0; i < nodeArray.length; i++) {
            for (let j = i + 1; j < nodeArray.length; j++) {
                const nodeA = nodeArray[i];
                const nodeB = nodeArray[j];
                
                const delta = new THREE.Vector3()
                    .subVectors(nodeA.position, nodeB.position);
                
                const distance = delta.length();
                if (distance > 0 && distance < 15) {
                    const force = repulsionForce / (distance * distance + 0.1);
                    delta.normalize().multiplyScalar(force * timeStep);
                    
                    nodeA.velocity.add(delta);
                    nodeB.velocity.sub(delta);
                }
            }
        }
        
        // 2. 连接边引力
        for (const link of this.links) {
            const sourceNode = this.nodes.get(link.source);
            const targetNode = this.nodes.get(link.target);
            
            if (!sourceNode || !targetNode) continue;
            
            const delta = new THREE.Vector3()
                .subVectors(targetNode.position, sourceNode.position);
            
            const distance = delta.length();
            if (distance > 0) {
                const force = attractionForce * Math.min(distance, 10);
                delta.normalize().multiplyScalar(force * timeStep);
                
                sourceNode.velocity.add(delta);
                targetNode.velocity.sub(delta);
            }
        }
        
        // 3. 椭圆边界约束
        for (const node of nodeArray) {
            const pos = node.position;
            const ellipseDistance = Math.sqrt(
                (pos.x * pos.x) / (boundaryWidth * boundaryWidth) +
                (pos.y * pos.y) / (boundaryHeight * boundaryHeight) +
                (pos.z * pos.z) / (boundaryDepth * boundaryDepth)
            );
            
            if (ellipseDistance > 0.8) {
                const toCenter = new THREE.Vector3()
                    .subVectors(new THREE.Vector3(0, 0, 0), pos)
                    .normalize();
                
                const force = (ellipseDistance - 0.8) * 0.5;
                toCenter.multiplyScalar(force * timeStep);
                node.velocity.add(toCenter);
            }
        }
        
        // 4. 非均匀中心引力
        for (const node of nodeArray) {
            const pos = node.position;
            
            const centerPullY = -pos.y * centerForce * 1.5 * timeStep;
            const centerPullZ = -pos.z * centerForce * 1.2 * timeStep;
            const centerPullX = -pos.x * centerForce * 0.8 * timeStep;
            
            node.velocity.x += centerPullX;
            node.velocity.y += centerPullY;
            node.velocity.z += centerPullZ;
        }
        
        // 5. 更新位置和速度
        for (const node of nodeArray) {
            node.velocity.multiplyScalar(damping);
            
            // 限制最大速度
            const maxSpeedX = 2.0;
            const maxSpeedY = 1.2;
            const maxSpeedZ = 1.5;
            
            if (Math.abs(node.velocity.x) > maxSpeedX) {
                node.velocity.x = Math.sign(node.velocity.x) * maxSpeedX;
            }
            if (Math.abs(node.velocity.y) > maxSpeedY) {
                node.velocity.y = Math.sign(node.velocity.y) * maxSpeedY;
            }
            if (Math.abs(node.velocity.z) > maxSpeedZ) {
                node.velocity.z = Math.sign(node.velocity.z) * maxSpeedZ;
            }
            
            node.position.add(node.velocity);
            node.mesh.position.copy(node.position);
        }
        
        this.updateLinks();
    }
    
    // 更新连线位置
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
    
    // 清空图形
    clearGraph() {
        // 移除所有节点mesh
        this.nodeMeshes.forEach(mesh => {
            this.scene.remove(mesh);
            mesh.geometry.dispose();
            mesh.material.dispose();
        });
        
        // 移除所有连线
        this.linkLines.forEach(line => {
            this.scene.remove(line);
            line.geometry.dispose();
            line.material.dispose();
        });
        
        // 清空数据
        this.nodes.clear();
        this.links = [];
        this.nodeMeshes = [];
        this.linkLines = [];
    }
    
    // 根据ID查找节点
    getNodeById(nodeId) {
        return this.nodes.get(nodeId);
    }
    
    // 高亮特定节点
    highlightNode(nodeId, highlight = true) {
        const node = this.nodes.get(nodeId);
        if (node) {
            node.mesh.material.emissive.setHex(highlight ? 0x444444 : 0x000000);
            node.mesh.scale.setScalar(highlight ? 1.2 : 1.0);
        }
    }
    
    // 高亮与节点相关的边
    highlightNodeLinks(nodeId, highlight = true) {
        this.links.forEach(link => {
            if (link.source === nodeId || link.target === nodeId) {
                link.line.material.color.setHex(highlight ? 0xff6b6b : 0x666666);
                link.line.material.opacity = highlight ? 1.0 : 0.6;
            }
        });
    }
}