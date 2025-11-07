class OptimizedForceGraph3D extends ForceDirectedGraph3D {
    updatePhysics() {
        const repulsionForce = 100;
        const attractionForce = 0.1;
        const centerForce = 0.03;
        const damping = 0.85;
        const timeStep = 0.016;
        const boundaryRadius = 20;
        const boundaryForce = 0.3;
        
        // 1. 节点间斥力
        for (let i = 0; i < this.nodes.length; i++) {
            for (let j = i + 1; j < this.nodes.length; j++) {
                const delta = new THREE.Vector3()
                    .subVectors(this.positions[i], this.positions[j]);
                
                const distance = delta.length();
                if (distance > 0 && distance < 15) { // 距离阈值优化
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
                const force = attractionForce * Math.min(distance, 10); // 限制最大引力
                delta.normalize().multiplyScalar(force * timeStep);
                
                this.velocities[link.source].add(delta);
                this.velocities[link.target].sub(delta);
            }
        }
        
        // 3. 中心引力（温和）
        for (let i = 0; i < this.nodes.length; i++) {
            const distanceToCenter = this.positions[i].length();
            if (distanceToCenter > 5) { // 只有离中心较远时才施加中心引力
                const toCenter = new THREE.Vector3()
                    .subVectors(new THREE.Vector3(0, 0, 0), this.positions[i]);
                
                const force = centerForce * Math.min(distanceToCenter, 15);
                toCenter.normalize().multiplyScalar(force * timeStep);
                this.velocities[i].add(toCenter);
            }
        }
        
        // 4. 边界约束
        for (let i = 0; i < this.nodes.length; i++) {
            const distanceToCenter = this.positions[i].length();
            
            if (distanceToCenter > boundaryRadius) {
                const toCenter = new THREE.Vector3()
                    .subVectors(new THREE.Vector3(0, 0, 0), this.positions[i])
                    .normalize();
                
                const force = boundaryForce * (distanceToCenter - boundaryRadius);
                toCenter.multiplyScalar(force * timeStep);
                this.velocities[i].add(toCenter);
            }
        }
        
        // 5. 更新位置和速度
        for (let i = 0; i < this.nodes.length; i++) {
            this.velocities[i].multiplyScalar(damping);
            
            // 限制最大速度
            const speed = this.velocities[i].length();
            const maxSpeed = 2;
            if (speed > maxSpeed) {
                this.velocities[i].normalize().multiplyScalar(maxSpeed);
            }
            
            this.positions[i].add(this.velocities[i]);
            this.nodes[i].position.copy(this.positions[i]);
        }
        
        this.updateLinks();
    }
}