import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

// 创建场景
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xf0f8ff);

// 创建相机
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(15, 10, 15);

// 创建渲染器
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

// 添加轨道控制器
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

// 添加灯光
const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
directionalLight.position.set(20, 30, 20);
directionalLight.castShadow = true;
directionalLight.shadow.mapSize.width = 2048;
directionalLight.shadow.mapSize.height = 2048;
scene.add(directionalLight);

// 创建核电站烟囱（上小下大，腰部弧形）
function createNuclearChimney() {
    const group = new THREE.Group();
    
    const height = 20;
    const bottomRadius = 6;
    const topRadius = 4;
    const waistHeight = height * 0.6; // 腰部位置
    const waistRadius = 4.5; // 腰部半径比顶部大，比底部小
    
    // 创建烟囱壁的曲线（腰部内凹）
    const curvePoints = [];
    const segments = 64;
    
    for (let i = 0; i <= segments; i++) {
        const t = i / segments;
        const y = t * height;
        
        // 计算当前高度的半径（使用贝塞尔曲线实现小蛮腰）
        let radius;
        if (y < waistHeight) {
            // 底部到腰部
            const t2 = y / waistHeight;
            radius = bottomRadius + (waistRadius - bottomRadius) * (1 - Math.pow(1 - t2, 1.5));
        } else {
            // 腰部到顶部
            const t2 = (y - waistHeight) / (height - waistHeight);
            radius = waistRadius + (topRadius - waistRadius) * Math.pow(t2, 1.2);
        }
        
        curvePoints.push(new THREE.Vector3(radius, y, 0));
    }
    
    // 创建旋转几何体（LatheGeometry）
    const chimneyGeometry = new THREE.LatheGeometry(
        curvePoints,
        32  // 径向分段数
    );
    
    // 烟囱材质
    const chimneyMaterial = new THREE.MeshStandardMaterial({
        color: 0xa9a9a9,
        metalness: 0.3,
        roughness: 0.7,
        side: THREE.DoubleSide
    });
    
    const chimney = new THREE.Mesh(chimneyGeometry, chimneyMaterial);
    chimney.castShadow = true;
    chimney.receiveShadow = true;
    group.add(chimney);
    
    // 创建顶部开口
    const topHoleGeometry = new THREE.RingGeometry(topRadius * 0.95, topRadius, 32);
    const topHoleMaterial = new THREE.MeshStandardMaterial({
        color: 0x555555,
        metalness: 0.5,
        roughness: 0.6
    });
    const topHole = new THREE.Mesh(topHoleGeometry, topHoleMaterial);
    topHole.position.y = height;
    group.add(topHole);
    
    // 创建底部开口
    const bottomHoleGeometry = new THREE.RingGeometry(bottomRadius * 0.95, bottomRadius, 32);
    const bottomHoleMaterial = new THREE.MeshStandardMaterial({
        color: 0x555555,
        metalness: 0.5,
        roughness: 0.6
    });
    const bottomHole = new THREE.Mesh(bottomHoleGeometry, bottomHoleMaterial);
    bottomHole.position.y = 0;
    group.add(bottomHole);
    
    // 返回烟囱数据和几何体用于笔锋效果
    return {
        group,
        height,
        bottomRadius,
        topRadius,
        waistHeight,
        waistRadius,
        curvePoints
    };
}

// 创建毛笔笔锋效果（在烟囱壁外侧两侧）
function createBrushStrokesOnChimney(chimneyData) {
    const group = new THREE.Group();
    
    const { height, bottomRadius, topRadius, waistHeight, waistRadius, curvePoints } = chimneyData;
    
    // 左侧笔锋（一撇效果）
    const leftStroke = createSingleBrushStroke(
        chimneyData,
        -Math.PI / 4, // 从左上到右下
        0xff6b6b,      // 红色系
        -0.2           // 向外偏移
    );
    group.add(leftStroke);
    
    // 右侧笔锋（一捺效果）
    const rightStroke = createSingleBrushStroke(
        chimneyData,
        Math.PI / 4,   // 从右上到左下  
        0x4ecdc4,      // 青色系
        0.2            // 向外偏移
    );
    group.add(rightStroke);
    
    return group;
}

function createSingleBrushStroke(chimneyData, angle, color, sideOffset) {
    const { height, curvePoints } = chimneyData;
    const segments = curvePoints.length;
    
    // 创建笔锋的几何体
    const geometry = new THREE.BufferGeometry();
    
    const positions = [];
    const normals = [];
    const uvs = [];
    const indices = [];
    
    const strokeWidth = 0.8;  // 笔锋基础宽度
    const taperStart = 0.7;   // 开始变细的位置比例
    
    // 创建顶点
    for (let i = 0; i <= segments; i++) {
        const t = i / segments;
        const curvePoint = curvePoints[i];
        
        // 计算笔锋的宽度（从底部到顶部逐渐变细）
        let width;
        if (t < taperStart) {
            width = strokeWidth * (1 - t * 0.3);
        } else {
            width = strokeWidth * (1 - taperStart * 0.3) * (1 - (t - taperStart) / (1 - taperStart));
        }
        
        // 计算笔锋在烟囱表面的位置（向外偏移）
        const radius = curvePoint.x + 0.1; // 向外偏移一点
        
        // 计算角度（使笔锋沿烟囱表面展开）
        const phi = angle + sideOffset * (0.5 + t * 0.3);
        
        // 计算顶点位置
        const x = radius * Math.cos(phi);
        const z = radius * Math.sin(phi);
        const y = curvePoint.y;
        
        // 计算向外扩展的位置（模拟笔锋厚度）
        const xOuter = (radius + width) * Math.cos(phi);
        const zOuter = (radius + width) * Math.sin(phi);
        
        // 添加内层和外层顶点
        positions.push(x, y, z);          // 内层顶点
        positions.push(xOuter, y, zOuter); // 外层顶点
        
        // 计算法向量
        const normal = new THREE.Vector3(x, 0, z).normalize();
        normals.push(normal.x, normal.y, normal.z);
        normals.push(normal.x, normal.y, normal.z);
        
        // UV坐标
        uvs.push(t, 0);
        uvs.push(t, 1);
    }
    
    // 创建三角形索引
    const vertexCount = (segments + 1) * 2;
    for (let i = 0; i < segments; i++) {
        const idx = i * 2;
        // 两个三角形组成一个四边形
        indices.push(idx, idx + 1, idx + 2);
        indices.push(idx + 1, idx + 3, idx + 2);
    }
    
    geometry.setIndex(indices);
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
    geometry.setAttribute('normal', new THREE.Float32BufferAttribute(normals, 3));
    geometry.setAttribute('uv', new THREE.Float32BufferAttribute(uvs, 2));
    
    // 创建毛笔笔锋材质
    const material = new THREE.ShaderMaterial({
        uniforms: {
            color: { value: new THREE.Color(color) },
            time: { value: 0 },
            gradient: { value: 1.0 },
            opacity: { value: 0.85 }
        },
        vertexShader: `
            varying vec2 vUv;
            varying float vGradient;
            
            void main() {
                vUv = uv;
                vGradient = uv.x; // 从下到上的渐变
                
                vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
                gl_Position = projectionMatrix * mvPosition;
            }
        `,
        fragmentShader: `
            uniform vec3 color;
            uniform float time;
            uniform float opacity;
            varying vec2 vUv;
            varying float vGradient;
            
            void main() {
                // 基础颜色
                vec3 baseColor = color;
                
                // 添加一些变化
                float variation = sin(vGradient * 10.0 + time * 2.0) * 0.05;
                baseColor += vec3(variation * 0.5, variation, variation * 0.8);
                
                // 边缘淡化效果
                float edgeFade = 1.0;
                
                // 顶部和底部淡化
                edgeFade *= smoothstep(0.0, 0.1, vUv.x);
                edgeFade *= 1.0 - smoothstep(0.9, 1.0, vUv.x);
                
                // 左右边缘淡化（模拟笔锋）
                float edgeY = 1.0 - smoothstep(0.0, 0.3, abs(vUv.y - 0.5));
                edgeFade *= edgeY;
                
                // 毛笔笔触纹理
                float brushTexture = sin(vUv.y * 30.0 + time) * 0.1 + 0.9;
                
                gl_FragColor = vec4(baseColor * brushTexture, opacity * edgeFade);
            }
        `,
        transparent: true,
        side: THREE.DoubleSide,
        blending: THREE.NormalBlending,
        depthWrite: false
    });
    
    const mesh = new THREE.Mesh(geometry, material);
    
    // 添加发光粒子效果
    addGlowParticles(mesh, chimneyData, angle, color, sideOffset);
    
    return mesh;
}

function addGlowParticles(strokeMesh, chimneyData, angle, color, sideOffset) {
    const { height, curvePoints } = chimneyData;
    const particleCount = 100;
    
    const particleGeometry = new THREE.BufferGeometry();
    const particlePositions = new Float32Array(particleCount * 3);
    const particleColors = new Float32Array(particleCount * 3);
    const particleSizes = new Float32Array(particleCount);
    
    for (let i = 0; i < particleCount; i++) {
        const t = Math.random();
        const segmentIndex = Math.floor(t * (curvePoints.length - 1));
        const curvePoint = curvePoints[segmentIndex];
        
        // 计算粒子位置（在笔锋路径上）
        const radius = curvePoint.x + 0.15;
        const phi = angle + sideOffset * (0.5 + t * 0.3) + (Math.random() - 0.5) * 0.2;
        const width = 0.6 * (1 - t * 0.5);
        
        const x = (radius + width * Math.random()) * Math.cos(phi);
        const z = (radius + width * Math.random()) * Math.sin(phi);
        const y = curvePoint.y + (Math.random() - 0.5) * 0.3;
        
        particlePositions[i * 3] = x;
        particlePositions[i * 3 + 1] = y;
        particlePositions[i * 3 + 2] = z;
        
        // 颜色渐变
        const colorObj = new THREE.Color(color);
        const lightColor = colorObj.clone().multiplyScalar(1.5);
        const mixedColor = colorObj.clone().lerp(lightColor, Math.random() * 0.5);
        
        particleColors[i * 3] = mixedColor.r;
        particleColors[i * 3 + 1] = mixedColor.g;
        particleColors[i * 3 + 2] = mixedColor.b;
        
        particleSizes[i] = Math.random() * 0.08 + 0.02;
    }
    
    particleGeometry.setAttribute('position', new THREE.BufferAttribute(particlePositions, 3));
    particleGeometry.setAttribute('color', new THREE.BufferAttribute(particleColors, 3));
    particleGeometry.setAttribute('size', new THREE.BufferAttribute(particleSizes, 1));
    
    const particleMaterial = new THREE.ShaderMaterial({
        uniforms: {
            time: { value: 0 }
        },
        vertexShader: `
            attribute vec3 color;
            attribute float size;
            varying vec3 vColor;
            uniform float time;
            
            void main() {
                vColor = color;
                
                // 轻微浮动
                vec3 pos = position;
                pos.y += sin(time * 2.0 + position.x * 10.0) * 0.02;
                
                vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
                gl_PointSize = size * (300.0 / -mvPosition.z);
                gl_Position = projectionMatrix * mvPosition;
            }
        `,
        fragmentShader: `
            varying vec3 vColor;
            
            void main() {
                // 圆形粒子
                float r = length(gl_PointCoord - vec2(0.5));
                if (r > 0.5) discard;
                
                // 边缘淡化
                float alpha = (0.5 - r) * 2.0;
                
                // 中心更亮
                float center = 1.0 - smoothstep(0.0, 0.5, r);
                vec3 finalColor = mix(vColor, vec3(1.0), center * 0.3);
                
                gl_FragColor = vec4(finalColor, alpha * 0.6);
            }
        `,
        transparent: true,
        blending: THREE.AdditiveBlending,
        depthWrite: false
    });
    
    const particles = new THREE.Points(particleGeometry, particleMaterial);
    strokeMesh.add(particles);
    
    // 保存引用以便动画更新
    strokeMesh.userData.particles = particles;
    strokeMesh.userData.particleMaterial = particleMaterial;
}

// 创建地面
function createGround() {
    const groundGeometry = new THREE.PlaneGeometry(100, 100);
    const groundMaterial = new THREE.MeshStandardMaterial({
        color: 0x8B7355,
        roughness: 0.9,
        metalness: 0.1
    });
    const ground = new THREE.Mesh(groundGeometry, groundMaterial);
    ground.rotation.x = -Math.PI / 2;
    ground.position.y = -0.5;
    ground.receiveShadow = true;
    return ground;
}

// 创建天空
function createSky() {
    const skyGeometry = new THREE.SphereGeometry(500, 32, 32);
    const skyMaterial = new THREE.MeshBasicMaterial({
        color: 0x87CEEB,
        side: THREE.BackSide
    });
    const sky = new THREE.Mesh(skyGeometry, skyMaterial);
    return sky;
}

// 初始化场景
const ground = createGround();
scene.add(ground);

const sky = createSky();
scene.add(sky);

// 创建烟囱
const chimneyData = createNuclearChimney();
const chimney = chimneyData.group;
scene.add(chimney);

// 创建笔锋效果
const brushStrokes = createBrushStrokesOnChimney(chimneyData);
scene.add(brushStrokes);

// 将笔锋效果附加到烟囱上
brushStrokes.position.copy(chimney.position);

// 添加一些烟雾效果
function addSmokeEffect() {
    const smokeGroup = new THREE.Group();
    
    const smokeGeometry = new THREE.SphereGeometry(1, 8, 8);
    const smokeMaterial = new THREE.MeshBasicMaterial({
        color: 0xffffff,
        transparent: true,
        opacity: 0.3
    });
    
    for (let i = 0; i < 5; i++) {
        const smoke = new THREE.Mesh(smokeGeometry, smokeMaterial);
        smoke.scale.set(1 + Math.random() * 2, 1 + Math.random() * 2, 1 + Math.random() * 2);
        smoke.position.y = chimneyData.height + i * 3;
        smoke.position.x = (Math.random() - 0.5) * 2;
        smoke.position.z = (Math.random() - 0.5) * 2;
        smokeGroup.add(smoke);
    }
    
    scene.add(smokeGroup);
    return smokeGroup;
}

const smoke = addSmokeEffect();

// 动画循环
const clock = new THREE.Clock();
function animate() {
    const time = clock.getElapsedTime();
    
    // 更新笔锋效果的动画
    brushStrokes.traverse((child) => {
        if (child.material && child.material.uniforms && child.material.uniforms.time) {
            child.material.uniforms.time.value = time;
        }
        if (child.userData && child.userData.particleMaterial) {
            child.userData.particleMaterial.uniforms.time.value = time;
        }
    });
    
    // 烟雾动画
    smoke.children.forEach((smokeMesh, i) => {
        smokeMesh.position.y = chimneyData.height + i * 3 + Math.sin(time + i) * 0.5;
        smokeMesh.scale.x = 1 + Math.sin(time * 0.5 + i) * 0.5;
        smokeMesh.scale.z = 1 + Math.cos(time * 0.5 + i) * 0.5;
    });
    
    controls.update();
    renderer.render(scene, camera);
    requestAnimationFrame(animate);
}

// 窗口调整
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// 开始动画
animate();