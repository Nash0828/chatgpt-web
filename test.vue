class ForceDirectedGraph3D {
    constructor(container) {
        // ... 其他初始化代码 ...
        this.initialPositions = new Map(); // 存储节点的初始位置
    }
    
    // 基于节点ID生成确定性位置
    getDeterministicPosition(nodeId, index) {
        // 使用节点ID生成hash值
        const hash = this.stringToHash(nodeId);
        const seed = hash % 10000;
        
        // 使用伪随机数生成器（相同的seed总是返回相同的结果）
        const rng = new Math.seedrandom(seed);
        
        return new THREE.Vector3(
            (rng() - 0.5) * 60,  // x: [-30, 30]
            (rng() - 0.5) * 30,  // y: [-15, 15] 
            (rng() - 0.5) * 20   // z: [-10, 10]
        );
    }
    
    // 字符串转hash值
    stringToHash(str) {
        let hash = 0;
        for (let i = 0; i < str.length; i++) {
            const char = str.charCodeAt(i);
            hash = ((hash << 5) - hash) + char;
            hash = hash & hash; // Convert to 32bit integer
        }
        return Math.abs(hash);
    }
    
    addNode(nodeData) {
        // 使用确定性位置
        const position = this.getDeterministicPosition(nodeData.id, this.nodes.size);
        
        // 存储初始位置
        this.initialPositions.set(nodeData.id, position.clone());
        
        // ... 创建mesh的代码 ...
    }
    
    // 重置到初始位置（可选）
    resetToInitialPositions() {
        this.nodes.forEach((node, nodeId) => {
            const initialPos = this.initialPositions.get(nodeId);
            if (initialPos) {
                node.position.copy(initialPos);
                node.velocity.set(0, 0, 0);
                node.mesh.position.copy(initialPos);
            }
        });
        this.updateLinks();
    }
}