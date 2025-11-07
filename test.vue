class OptimizedForceGraph3D {
    updatePhysics() {
        const repulsionForce = 100;
        const attractionForce = 0.1;
        const centerForce = 0.03;
        const damping = 0.85;
        const timeStep = 0.016;
        
        // 调整为宽屏边界 (16:9 比例)
        const boundaryWidth = 35;   // x轴边界
        const boundaryHeight = 20;  // y轴边界
        const boundaryDepth = 15;   // z轴边界（可以更浅）
        
        // 1. 节点间斥力
        for (let i = 0; i < this.nodes.length; i++) {
            for (let j = i + 1; j < this.nodes.length; j++) {
                const delta = new THREE.Vector3()
                    .subVectors(this.positions[i], this.positions[j]);
                
                const distance = delta.length();
                if (distance > 0 && distance < 15) {
                    const force = repulsionForce / (distance * distance + 0.1);
                    delta.normalize().multiplyScalar(force * timeStep);
                    
                    this.velocities[i].add(delta);
                    this.velocities[j].sub(delta);
                }
            }
        }
        
        // 2. 连接边引力
        for (const link of this.links) {
            const delta = new THREE.Vector3()
                .subVectors(this.positions[link.target], this.positions[link.source]);
            
            const distance = delta.length();
            if (distance > 0) {
                const force = attractionForce * Math.min(distance, 10);
                delta.normalize().multiplyScalar(force * timeStep);
                
                this.velocities[link.source].add(delta);
                this.velocities[link.target].sub(delta);
            }
        }
        
        // 3. 椭圆边界约束（宽屏适配）
        for (let i = 0; i < this.nodes.length; i++) {
            const pos = this.positions[i];
            const xRatio = Math.abs(pos.x) / boundaryWidth;
            const yRatio = Math.abs(pos.y) / boundaryHeight;
            const zRatio = Math.abs(pos.z) / boundaryDepth;
            
            // 计算在椭圆边界内的"距离"
            const ellipseDistance = Math.sqrt(
                (pos.x * pos.x) / (boundaryWidth * boundaryWidth) +
                (pos.y * pos.y) / (boundaryHeight * boundaryHeight) +
                (pos.z * pos.z) / (boundaryDepth * boundaryDepth)
            );
            
            if (ellipseDistance > 0.8) { // 当接近边界时施加力
                const toCenter = new THREE.Vector3()
                    .subVectors(new THREE.Vector3(0, 0, 0), pos)
                    .normalize();
                
                // 根据各轴的比例调整力的大小
                const forceX = xRatio > 0.8 ? (xRatio - 0.8) * 0.5 : 0;
                const forceY = yRatio > 0.8 ? (yRatio - 0.8) * 0.5 : 0;
                const forceZ = zRatio > 0.8 ? (zRatio - 0.8) * 0.5 : 0;
                
                const totalForce = (forceX + forceY + forceZ) * boundaryForce;
                toCenter.multiplyScalar(totalForce * timeStep);
                this.velocities[i].add(toCenter);
            }
        }
        
        // 4. 非均匀中心引力（在y轴和z轴施加更强的中心引力）
        for (let i = 0; i < this.nodes.length; i++) {
            const pos = this.positions[i];
            
            // 在y轴和z轴方向施加更强的中心引力
            const centerPullY = -pos.y * centerForce * 1.5 * timeStep; // y轴引力更强
            const centerPullZ = -pos.z * centerForce * 1.2 * timeStep; // z轴引力稍强
            const centerPullX = -pos.x * centerForce * 0.8 * timeStep; // x轴引力较弱
            
            this.velocities[i].x += centerPullX;
            this.velocities[i].y += centerPullY;
            this.velocities[i].z += centerPullZ;
        }
        
        // 5. 更新位置和速度
        for (let i = 0; i < this.nodes.length; i++) {
            this.velocities[i].multiplyScalar(damping);
            
            // 限制最大速度（在y轴方向限制更严格）
            const maxSpeedX = 2.0;  // x轴允许较快移动
            const maxSpeedY = 1.2;  // y轴移动较慢
            const maxSpeedZ = 1.5;  // z轴移动中等
            
            if (Math.abs(this.velocities[i].x) > maxSpeedX) {
                this.velocities[i].x = Math.sign(this.velocities[i].x) * maxSpeedX;
            }
            if (Math.abs(this.velocities[i].y) > maxSpeedY) {
                this.velocities[i].y = Math.sign(this.velocities[i].y) * maxSpeedY;
            }
            if (Math.abs(this.velocities[i].z) > maxSpeedZ) {
                this.velocities[i].z = Math.sign(this.velocities[i].z) * maxSpeedZ;
            }
            
            this.positions[i].add(this.velocities[i]);
            this.nodes[i].position.copy(this.positions[i]);
        }
        
        this.updateLinks();
    }
}