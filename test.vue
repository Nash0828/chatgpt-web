import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

// 创建场景
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87CEEB);

// 创建相机
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(10, 8, 10);

// 创建渲染器
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

// 添加轨道控制器
const controls = new OrbitControls(camera, renderer.domElement);

// 添加环境光
const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
scene.add(ambientLight);

// 添加平行光
const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
directionalLight.position.set(10, 20, 15);
directionalLight.castShadow = true;
scene.add(directionalLight);

// 创建烟囱主体
function createChimney() {
    const group = new THREE.Group();
    
    // 烟囱主要曲线 - 弧形
    const curve = new THREE.CatmullRomCurve3([
        new THREE.Vector3(0, 0, 5),      // 底部开口较大
        new THREE.Vector3(1, 3, 3),      // 中间收缩
        new THREE.Vector3(0, 6, 0),      // 顶部较小
        new THREE.Vector3(-1, 9, -3),    // 弧形弯曲
        new THREE.Vector3(0, 12, -5)     // 顶部开口
    ]);
    
    // 生成烟囱的管道几何体
    const chimneyGeometry = new THREE.TubeGeometry(
        curve,
        64,           // 分段数
        4,           // 半径
        8,           // 径向分段数
        false        // 是否闭合
    );
    
    // 烟囱材质
    const chimneyMaterial = new THREE.MeshStandardMaterial({
        color: 0x7A7A7A,
        metalness: 0.4,
        roughness: 0.6,
        side: THREE.DoubleSide
    });
    
    const chimney = new THREE.Mesh(chimneyGeometry, chimneyMaterial);
    chimney.castShadow = true;
    chimney.receiveShadow = true;
    group.add(chimney);
    
    // 创建底部圆形开口
    const bottomHoleGeometry = new THREE.RingGeometry(3.5, 4, 32);
    const bottomHoleMaterial = new THREE.MeshStandardMaterial({
        color: 0x555555,
        metalness: 0.5,
        roughness: 0.7,
        side: THREE.DoubleSide
    });
    const bottomHole = new THREE.Mesh(bottomHoleGeometry, bottomHoleMaterial);
    bottomHole.rotation.x = Math.PI / 2;
    bottomHole.position.set(0, 0, 5);
    bottomHole.receiveShadow = true;
    group.add(bottomHole);
    
    // 创建顶部圆形开口
    const topHoleGeometry = new THREE.RingGeometry(2.5, 3, 32);
    const topHoleMaterial = new THREE.MeshStandardMaterial({
        color: 0x555555,
        metalness: 0.5,
        roughness: 0.7,
        side: THREE.DoubleSide
    });
    const topHole = new THREE.Mesh(topHoleGeometry, topHoleMaterial);
    topHole.rotation.x = Math.PI / 2;
    topHole.position.set(0, 12, -5);
    topHole.receiveShadow = true;
    group.add(topHole);
    
    return { group, curve };
}

// 创建毛笔笔锋效果（一撇一捺）
function createBrushStrokeEffects(curve) {
    const group = new THREE.Group();
    
    // 获取曲线上的点
    const points = curve.getPoints(100);
    
    // 左侧笔锋效果
    const leftStroke = createStroke(points, -1, 0xFF4500); // 橙红色
    group.add(leftStroke);
    
    // 右侧笔锋效果
    const rightStroke = createStroke(points, 1, 0x4169E1); // 宝蓝色
    group.add(rightStroke);
    
    return group;
}

function createStroke(points, side, color) {
    // 创建笔锋效果的几何体
    const geometry = new THREE.BufferGeometry();
    
    const positions = [];
    const indices = [];
    const uvs = [];
    
    const strokeWidth = 0.8; // 笔锋宽度
    const taperFactor = 0.3; // 锥度因子
    
    for (let i = 0; i < points.length; i++) {
        const point = points[i];
        const t = i / (points.length - 1);
        
        // 计算法向量和切向量
        let tangent = new THREE.Vector3();
        if (i < points.length - 1) {
            tangent.subVectors(points[i + 1], point).normalize();
        } else {
            tangent.subVectors(point, points[i - 1]).normalize();
        }
        
        const normal = new THREE.Vector3(0, 1, 0);
        const binormal = new THREE.Vector3().crossVectors(normal, tangent).normalize();
        
        // 创建锥形笔锋效果
        const width = strokeWidth * (1 - taperFactor * t);
        const offset = binormal.multiplyScalar(side * width);
        
        // 添加顶点
        positions.push(
            point.x + offset.x,
            point.y + offset.y,
            point.z + offset.z
        );
        
        // 也添加中心点用于创建三角形带
        positions.push(point.x, point.y, point.z);
        
        // UV 坐标
        uvs.push(t, 0);
        uvs.push(t, 1);
    }
    
    // 创建索引
    for (let i = 0; i < points.length - 1; i++) {
        const idx = i * 2;
        indices.push(idx, idx + 1, idx + 2);
        indices.push(idx + 1, idx + 3, idx + 2);
    }
    
    geometry.setIndex(indices);
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
    geometry.setAttribute('uv', new THREE.Float32BufferAttribute(uvs, 2));
    geometry.computeVertexNormals();
    
    // 创建渐变材质
    const material = new THREE.ShaderMaterial({
        uniforms: {
            color1: { value: new THREE.Color(color) },
            color2: { value: new THREE.Color(0xFFFF00) },
            time: { value: 0 },
            opacity: { value: 0.8 }
        },
        vertexShader: `
            varying vec2 vUv;
            varying vec3 vPosition;
            void main() {
                vUv = uv;
                vPosition = position;
                gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
            }
        `,
        fragmentShader: `
            uniform vec3 color1;
            uniform vec3 color2;
            uniform float time;
            uniform float opacity;
            varying vec2 vUv;
            varying vec3 vPosition;
            
            void main() {
                // 创建毛笔笔触的渐变效果
                float gradient = vUv.x;
                vec3 color = mix(color1, color2, gradient * 0.5);
                
                // 添加一些纹理效果
                float noise = sin(vPosition.y * 5.0 + time) * 0.1;
                color += vec3(noise * 0.2);
                
                // 边缘淡化
                float edgeFade = 1.0 - smoothstep(0.8, 1.0, abs(vUv.y - 0.5) * 2.0);
                
                gl_FragColor = vec4(color, opacity * edgeFade);
            }
        `,
        transparent: true,
        side: THREE.DoubleSide,
        blending: THREE.AdditiveBlending
    });
    
    const mesh = new THREE.Mesh(geometry, material);
    
    // 添加一些粒子效果增强笔锋
    const particleCount = 200;
    const particleGeometry = new THREE.BufferGeometry();
    const particlePositions = new Float32Array(particleCount * 3);
    const particleSizes = new Float32Array(particleCount);
    
    for (let i = 0; i < particleCount; i++) {
        const t = Math.random();
        const pointIndex = Math.floor(t * (points.length - 1));
        const point = points[pointIndex];
        
        let tangent = new THREE.Vector3();
        if (pointIndex < points.length - 1) {
            tangent.subVectors(points[pointIndex + 1], point).normalize();
        } else {
            tangent.subVectors(point, points[pointIndex - 1]).normalize();
        }
        
        const normal = new THREE.Vector3(0, 1, 0);
        const binormal = new THREE.Vector3().crossVectors(normal, tangent).normalize();
        
        const width = strokeWidth * (1 - taperFactor * t);
        const offset = binormal.multiplyScalar(side * width * (0.8 + Math.random() * 0.4));
        
        particlePositions[i * 3] = point.x + offset.x;
        particlePositions[i * 3 + 1] = point.y + offset.y + (Math.random() - 0.5) * 0.5;
        particlePositions[i * 3 + 2] = point.z + offset.z;
        
        particleSizes[i] = Math.random() * 0.1 + 0.05;
    }
    
    particleGeometry.setAttribute('position', new THREE.BufferAttribute(particlePositions, 3));
    particleGeometry.setAttribute('size', new THREE.BufferAttribute(particleSizes, 1));
    
    const particleMaterial = new THREE.ShaderMaterial({
        uniforms: {
            color: { value: new THREE.Color(color) },
            time: { value: 0 }
        },
        vertexShader: `
            attribute float size;
            varying float vAlpha;
            uniform float time;
            
            void main() {
                vAlpha = 0.5 + 0.5 * sin(time + position.y * 2.0);
                vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
                gl_PointSize = size * (300.0 / -mvPosition.z);
                gl_Position = projectionMatrix * mvPosition;
            }
        `,
        fragmentShader: `
            uniform vec3 color;
            varying float vAlpha;
            
            void main() {
                float r = distance(gl_PointCoord, vec2(0.5, 0.5));
                if (r > 0.5) discard;
                float alpha = (0.5 - r) * 2.0 * vAlpha;
                gl_FragColor = vec4(color, alpha);
            }
        `,
        transparent: true,
        blending: THREE.AdditiveBlending
    });
    
    const particles = new THREE.Points(particleGeometry, particleMaterial);
    mesh.userData.particles = particles;
    mesh.userData.particleMaterial = particleMaterial;
    
    const strokeGroup = new THREE.Group();
    strokeGroup.add(mesh);
    strokeGroup.add(particles);
    
    return strokeGroup;
}

// 创建地面
function createGround() {
    const groundGeometry = new THREE.PlaneGeometry(50, 50);
    const groundMaterial = new THREE.MeshStandardMaterial({
        color: 0x8B7355,
        roughness: 0.8,
        metalness: 0.2
    });
    const ground = new THREE.Mesh(groundGeometry, groundMaterial);
    ground.rotation.x = -Math.PI / 2;
    ground.position.y = -1;
    ground.receiveShadow = true;
    return ground;
}

// 创建场景
const ground = createGround();
scene.add(ground);

const chimneyResult = createChimney();
const chimney = chimneyResult.group;
scene.add(chimney);

const brushStrokes = createBrushStrokeEffects(chimneyResult.curve);
scene.add(brushStrokes);

// 动画循环
const clock = new THREE.Clock();
function animate() {
    const time = clock.getElapsedTime();
    
    // 更新笔锋效果的动画
    brushStrokes.traverse((child) => {
        if (child.material && child.material.uniforms) {
            if (child.material.uniforms.time) {
                child.material.uniforms.time.value = time;
            }
        }
        if (child.userData && child.userData.particleMaterial) {
            child.userData.particleMaterial.uniforms.time.value = time;
        }
    });
    
    // 缓慢旋转烟囱以便观察
    chimney.rotation.y += 0.002;
    brushStrokes.rotation.y += 0.002;
    
    controls.update();
    renderer.render(scene, camera);
    requestAnimationFrame(animate);
}

// 窗口大小调整
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// 开始动画
animate();