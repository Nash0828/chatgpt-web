import * as THREE from 'three';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';

// 创建场景、相机等
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// 创建圆盘几何体
const geometry = new THREE.CircleGeometry(5, 32);
const material = new THREE.MeshBasicMaterial({ 
    color: 0xffffff,
    emissive: 0x0000ff, // 蓝色自发光
    emissiveIntensity: 0 // 初始强度为0
});
const disk = new THREE.Mesh(geometry, material);
scene.add(disk);

// 创建后期处理器
const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

// 创建辉光效果
const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    1.5, // 强度
    0.4, // 半径
    0.85 // 阈值
);
composer.addPass(bloomPass);

// 控制光晕的变量
let isGlowing = false;

// 切换光晕效果函数
function toggleGlow() {
    isGlowing = !isGlowing;
    
    if (isGlowing) {
        material.emissiveIntensity = 1; // 启用自发光
        bloomPass.strength = 1.5; // 增强辉光强度
    } else {
        material.emissiveIntensity = 0; // 关闭自发光
        bloomPass.strength = 0; // 关闭辉光
    }
}

// 渲染循环
function animate() {
    requestAnimationFrame(animate);
    composer.render();
}

// 添加切换按钮
const button = document.createElement('button');
button.textContent = '切换光晕';
button.style.position = 'absolute';
button.style.top = '10px';
button.style.left = '10px';
button.addEventListener('click', toggleGlow);
document.body.appendChild(button);

animate();