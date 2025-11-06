import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

class ForceDirectedGraph3D {
    constructor(container) {
        this.container = container;
        this.scene = new THREE.Scene();
        this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        this.renderer = new THREE.WebGLRenderer({ antialias: true });
        
        this.nodes = [];
        this.links = [];
        this.positions = [];
        this.velocities = [];
        
        this.init();
    }
    
    init() {
        // 渲染器设置
        this.renderer.setSize(window.innerWidth, window.innerHeight);
        this.renderer.setClearColor(0x0f172a);
        this.container.appendChild(this.renderer.domElement);
        
        // 相机位置
        this.camera.position.z = 50;
        
        // 轨道控制器
        this.controls = new OrbitControls(this.camera, this.renderer.domElement);
        
        // 环境光
        const ambientLight = new THREE.AmbientLight(0x404040);
        this.scene.add(ambientLight);
        
        // 方向光
        const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
        directionalLight.position.set(1, 1, 1);
        this.scene.add(directionalLight);
        
        // 生成示例数据
        this.generateSampleData();
        
        // 动画循环
        this.animate();
    }
    
    generateSampleData() {
        // 创建节点
        const nodeCount = 30;
        for (let i = 0; i < nodeCount; i++) {
            this.addNode(i);
        }
        
        // 创建连接
        for (let i = 0; i < nodeCount * 1.5; i++) {
            const source = Math.floor(Math.random() * nodeCount);
            let target = Math.floor(Math.random() * nodeCount);
            while (target === source) {
                target = Math.floor(Math.random() * nodeCount);
            }
            this.addLink(source, target);
        }
    }
    
    addNode(id) {
        // 随机初始位置
        const position = new THREE.Vector3(
            (Math.random() - 0.5) * 40,
            (Math.random() - 0.5) * 40,
            (Math.random() - 0.5) * 40
        );
        
        // 创建球体几何体
        const geometry = new THREE.SphereGeometry(0.8, 16, 16);
        const material = new THREE.MeshPhongMaterial({ 
            color: new THREE.Color().setHSL(Math.random(), 0.8, 0.6) 
        });
        const sphere = new THREE.Mesh(geometry, material);
        sphere.position.copy(position);
        
        this.scene.add(sphere);
        this.nodes.push(sphere);
        this.positions.push(position);
        this.velocities.push(new THREE.Vector3(0, 0, 0));
    }
    
    addLink(source, target) {
        // 创建连接线
        const geometry = new THREE.BufferGeometry();
        const material = new THREE.LineBasicMaterial({ 
            color: 0x666666,
            transparent: true,
            opacity: 0.6
        });
        
        const points = [
            this.positions[source],
            this.positions[target]
        ];
        
        geometry.setFromPoints(points);
        const line = new THREE.Line(geometry, material);
        this.scene.add(line);
        this.links.push({
            line: line,
            source: source,
            target: target
        });
    }
    
    updatePhysics() {
        const repulsionForce = 100;  // 斥力强度
        const attractionForce = 0.1; // 引力强度
        const damping = 0.8;         // 阻尼系数
        const timeStep = 0.016;      // 时间步长
        
        // 计算节点间的斥力
        for (let i = 0; i < this.nodes.length; i++) {
            for (let j = i + 1; j < this.nodes.length; j++) {
                const delta = new THREE.Vector3()
                    .subVectors(this.positions[i], this.positions[j]);
                
                const distance = delta.length();
                if (distance > 0) {
                    // 斥力计算 (与距离平方成反比)
                    const force = repulsionForce / (distance * distance);
                    delta.normalize().multiplyScalar(force * timeStep);
                    
                    this.velocities[i].add(delta);
                    this.velocities[j].sub(delta);
                }
            }
        }
        
        // 计算连接边的引力
        for (const link of this.links) {
            const delta = new THREE.Vector3()
                .subVectors(this.positions[link.target], this.positions[link.source]);
            
            const distance = delta.length();
            if (distance > 0) {
                // 胡克定律计算引力
                const force = attractionForce * distance;
                delta.normalize().multiplyScalar(force * timeStep);
                
                this.velocities[link.source].add(delta);
                this.velocities[link.target].sub(delta);
            }
        }
        
        // 更新位置和速度
        for (let i = 0; i < this.nodes.length; i++) {
            // 应用阻尼
            this.velocities[i].multiplyScalar(damping);
            
            // 更新位置
            this.positions[i].add(this.velocities[i]);
            this.nodes[i].position.copy(this.positions[i]);
        }
        
        // 更新连线
        this.updateLinks();
    }
    
    updateLinks() {
        for (const link of this.links) {
            const geometry = link.line.geometry;
            const positions = geometry.attributes.position.array;
            
            positions[0] = this.positions[link.source].x;
            positions[1] = this.positions[link.source].y;
            positions[2] = this.positions[link.source].z;
            
            positions[3] = this.positions[link.target].x;
            positions[4] = this.positions[link.target].y;
            positions[5] = this.positions[link.target].z;
            
            geometry.attributes.position.needsUpdate = true;
        }
    }
    
    animate() {
        requestAnimationFrame(() => this.animate());
        
        // 更新物理模拟
        this.updatePhysics();
        
        // 更新控制器
        this.controls.update();
        
        // 渲染
        this.renderer.render(this.scene, this.camera);
    }
}

// 使用示例
const graph = new ForceDirectedGraph3D(document.body);