<template>
  <div id="app">
    <div class="header">
      <h1>水平面四周光晕效果</h1>
      <p>平面放置在水平面上，光晕沿Y轴向上渐变</p>
    </div>
    
    <div class="controls">
      <div class="control-group">
        <label for="glow-color">光晕颜色:</label>
        <input type="color" id="glow-color" v-model="glowColor">
      </div>
      <div class="control-group">
        <label for="glow-height">光晕高度:</label>
        <input type="range" id="glow-height" v-model="glowHeight" min="0.5" max="3" step="0.1">
        <span>{{ glowHeight }}</span>
      </div>
      <div class="control-group">
        <label for="flow-speed">流动速度:</label>
        <input type="range" id="flow-speed" v-model="flowSpeed" min="0" max="2" step="0.1">
        <span>{{ flowSpeed }}</span>
      </div>
    </div>
    
    <div class="container">
      <div ref="container" class="canvas-container"></div>
      
      <div class="info-panel">
        <h3>实现说明</h3>
        <p>平面放置在水平面(XZ平面)上，四周光晕沿Y轴向上渐变</p>
        <p>光晕具有流动效果，从平面边缘向上扩散并逐渐消失</p>
        <p>使用顶点着色器实现高度渐变和流动动画</p>
      </div>
    </div>
  </div>
</template>

<script>
import * as THREE from 'three';
import { ref, onMounted, onUnmounted, watch } from 'vue';

// 顶点着色器 - 添加高度信息和流动效果
const vertexShader = `
  uniform float time;
  varying vec3 vPosition;
  varying vec2 vUv;
  
  void main() {
    vPosition = position;
    vUv = uv;
    
    // 添加轻微的波浪效果使平面更自然
    vec3 pos = position;
    float wave = sin(position.x * 2.0 + time) * 0.02 + cos(position.z * 2.0 + time) * 0.02;
    pos.y += wave;
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
  }
`;

// 片段着色器 - 创建四周向上渐变的光晕
const fragmentShader = `
  uniform vec3 glowColor;
  uniform float glowHeight;
  uniform float time;
  varying vec3 vPosition;
  varying vec2 vUv;
  
  void main() {
    // 计算到边缘的距离（0到1之间）
    float edgeX = min(vUv.x, 1.0 - vUv.x) * 2.0;
    float edgeZ = min(vUv.y, 1.0 - vUv.y) * 2.0;
    float edgeDistance = min(edgeX, edgeZ);
    
    // 边缘检测 - 只在靠近边缘的地方产生光晕
    float edgeFactor = 1.0 - smoothstep(0.3, 0.5, edgeDistance);
    
    // 计算高度因子 - Y轴向上渐变
    float heightFactor = 1.0 - smoothstep(0.0, glowHeight, vPosition.y);
    
    // 流动效果 - 随时间变化
    float flow = sin(time * 2.0 + vPosition.x * 5.0 + vPosition.z * 5.0) * 0.5 + 0.5;
    
    // 组合所有因素
    float glowIntensity = edgeFactor * heightFactor * flow * 2.0;
    
    // 设置最终颜色
    vec3 finalColor = glowColor * glowIntensity;
    float alpha = glowIntensity * 0.8;
    
    gl_FragColor = vec4(finalColor, alpha);
  }
`;

export default {
  name: 'App',
  setup() {
    const container = ref(null);
    const glowColor = ref('#3a86ff');
    const glowHeight = ref('1.5');
    const flowSpeed = ref('0.8');
    
    let scene, camera, renderer, plane, glowMesh, frameId;
    let clock = new THREE.Clock();
    
    const initThreeJS = () => {
      // 创建场景
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x0a0a1a);
      
      // 创建相机 - 从斜上方视角观看水平面
      const aspectRatio = container.value.clientWidth / container.value.clientHeight;
      camera = new THREE.PerspectiveCamera(60, aspectRatio, 0.1, 1000);
      camera.position.set(5, 3, 5);
      camera.lookAt(0, 0, 0);
      
      // 创建渲染器
      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(container.value.clientWidth, container.value.clientHeight);
      container.value.appendChild(renderer.domElement);
      
      // 添加环境光
      const ambientLight = new THREE.AmbientLight(0x404040, 0.6);
      scene.add(ambientLight);
      
      // 添加定向光
      const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
      directionalLight.position.set(5, 10, 7);
      directionalLight.castShadow = true;
      scene.add(directionalLight);
      
      // 创建网格地面
      const gridHelper = new THREE.GridHelper(20, 20, 0x444444, 0x222222);
      gridHelper.position.y = -0.01;
      scene.add(gridHelper);
      
      // 创建平面几何体（水平放置）
      const planeGeometry = new THREE.PlaneGeometry(4, 4);
      const planeMaterial = new THREE.MeshPhongMaterial({
        color: 0x4cc9f0,
        transparent: true,
        opacity: 0.9,
        side: THREE.DoubleSide
      });
      
      plane = new THREE.Mesh(planeGeometry, planeMaterial);
      plane.rotation.x = -Math.PI / 2; // 水平放置
      plane.position.y = 0;
      scene.add(plane);
      
      // 创建光晕几何体（围绕平面的四周）
      const glowGeometry = new THREE.BoxGeometry(4.2, 1.5, 4.2);
      const glowMaterial = new THREE.ShaderMaterial({
        uniforms: {
          glowColor: { value: new THREE.Color(glowColor.value) },
          glowHeight: { value: parseFloat(glowHeight.value) },
          time: { value: 0 }
        },
        vertexShader: vertexShader,
        fragmentShader: fragmentShader,
        transparent: true,
        side: THREE.DoubleSide,
        blending: THREE.AdditiveBlending
      });
      
      glowMesh = new THREE.Mesh(glowGeometry, glowMaterial);
      glowMesh.rotation.x = -Math.PI / 2; // 水平放置
      glowMesh.position.y = parseFloat(glowHeight.value) / 2; // 居中于高度
      scene.add(glowMesh);
      
      // 添加坐标轴辅助工具
      const axesHelper = new THREE.AxesHelper(3);
      scene.add(axesHelper);
    };
    
    const onWindowResize = () => {
      if (!camera || !renderer) return;
      
      camera.aspect = container.value.clientWidth / container.value.clientHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(container.value.clientWidth, container.value.clientHeight);
    };
    
    const render = () => {
      frameId = requestAnimationFrame(render);
      
      // 更新时间uniform
      if (glowMesh) {
        glowMesh.material.uniforms.time.value = clock.getElapsedTime() * parseFloat(flowSpeed.value);
      }
      
      // 轻微旋转相机，更好地展示效果
      const time = clock.getElapsedTime();
      camera.position.x = 5 * Math.cos(time * 0.1);
      camera.position.z = 5 * Math.sin(time * 0.1);
      camera.lookAt(0, 0, 0);
      
      renderer.render(scene, camera);
    };
    
    onMounted(() => {
      initThreeJS();
      window.addEventListener('resize', onWindowResize);
      render();
    });
    
    onUnmounted(() => {
      if (frameId) cancelAnimationFrame(frameId);
      window.removeEventListener('resize', onWindowResize);
      if (container.value && renderer?.domElement) {
        container.value.removeChild(renderer.domElement);
      }
    });
    
    watch(glowColor, (newColor) => {
      if (glowMesh) {
        glowMesh.material.uniforms.glowColor.value = new THREE.Color(newColor);
      }
    });
    
    watch(glowHeight, (newHeight) => {
      if (glowMesh) {
        const height = parseFloat(newHeight);
        glowMesh.material.uniforms.glowHeight.value = height;
        
        // 更新光晕网格几何体高度
        glowMesh.geometry.dispose();
        glowMesh.geometry = new THREE.BoxGeometry(4.2, height, 4.2);
        glowMesh.position.y = height / 2;
      }
    });
    
    return {
      container,
      glowColor,
      glowHeight,
      flowSpeed
    };
  }
};
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', 'Arial', sans-serif;
  background: linear-gradient(135deg, #0a0a1a 0%, #1a1a2e 100%);
  color: #fff;
  min-height: 100vh;
  overflow-x: hidden;
}

#app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  padding: 20px;
}

.header {
  text-align: center;
  padding: 20px 0 30px;
}

h1 {
  font-size: 2.5rem;
  margin-bottom: 10px;
  background: linear-gradient(90deg, #4361ee, #3a86ff, #4cc9f0);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.header p {
  font-size: 1.1rem;
  color: #a0a0c0;
  max-width: 600px;
  margin: 0 auto;
}

.controls {
  display: flex;
  justify-content: center;
  gap: 25px;
  padding: 15px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 12px;
  margin-bottom: 25px;
  flex-wrap: wrap;
  max-width: 900px;
  margin-left: auto;
  margin-right: auto;
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.control-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

label {
  font-weight: 600;
  color: #3a86ff;
  font-size: 0.9rem;
}

input[type="color"] {
  width: 50px;
  height: 30px;
  border: 2px solid rgba(58, 134, 255, 0.5);
  border-radius: 6px;
  cursor: pointer;
  background: rgba(10, 10, 26, 0.8);
}

input[type="range"] {
  width: 120px;
  height: 6px;
  -webkit-appearance: none;
  background: linear-gradient(to right, #4361ee, #3a86ff);
  border-radius: 3px;
  outline: none;
}

input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: #fff;
  cursor: pointer;
  box-shadow: 0 0 8px rgba(58, 134, 255, 0.8);
}

.container {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
}

.canvas-container {
  width: 100%;
  height: 500px;
  border-radius: 12px;
  overflow: hidden;
  background: rgba(0, 0, 0, 0.2);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
}

.info-panel {
  position: absolute;
  bottom: 15px;
  left: 15px;
  background: rgba(10, 10, 26, 0.8);
  padding: 15px;
  border-radius: 10px;
  max-width: 280px;
  backdrop-filter: blur(10px);
  border: 1px solid rgba(58, 134, 255, 0.3);
}

.info-panel h3 {
  margin-bottom: 10px;
  color: #3a86ff;
  font-size: 1.1rem;
}

.info-panel p {
  line-height: 1.5;
  color: #c0c0e0;
  margin-bottom: 8px;
  font-size: 0.9rem;
}

@media (max-width: 768px) {
  .controls {
    flex-direction: column;
    align-items: center;
    gap: 12px;
  }
  
  h1 {
    font-size: 2rem;
  }
  
  .info-panel {
    position: relative;
    margin-top: 15px;
    left: 0;
    max-width: 100%;
  }
  
  .canvas-container {
    height: 400px;
  }
}
</style>