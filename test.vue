<template>
  <div id="app">
    <div class="header">
      <h1>Vue3 + Three.js 平面光晕效果</h1>
      <p>基于Vite创建的项目，展示平面四周的光晕效果</p>
    </div>
    
    <div class="controls">
      <div class="control-group">
        <label for="glow-color">光晕颜色:</label>
        <input type="color" id="glow-color" v-model="glowColor">
      </div>
      <div class="control-group">
        <label for="glow-intensity">光晕强度:</label>
        <input type="range" id="glow-intensity" v-model="glowIntensity" min="1" max="10" step="0.1">
        <span>{{ glowIntensity }}</span>
      </div>
      <div class="control-group">
        <label for="rotation">旋转速度:</label>
        <input type="range" id="rotation" v-model="rotationSpeed" min="0" max="2" step="0.1">
        <span>{{ rotationSpeed }}</span>
      </div>
    </div>
    
    <div class="container">
      <div ref="container" class="canvas-container"></div>
      
      <div class="info-panel">
        <h3>实现说明</h3>
        <p>这个示例使用Three.js创建了一个平面几何体，并通过自定义着色器材质实现四周的光晕效果。</p>
        <p>光晕沿着Y轴方向渐变，通过控制emissive属性实现发光效果。</p>
        <p>使用上方的控制面板可以调整光晕的颜色和强度。</p>
      </div>
    </div>
  </div>
</template>

<script>
import * as THREE from 'three';
import { ref, onMounted, onUnmounted, watch } from 'vue';

// 顶点着色器
const vertexShader = `
  varying vec2 vUv;
  void main() {
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
`;

// 片段着色器 - 创建边缘光晕效果
const fragmentShader = `
  uniform vec3 glowColor;
  uniform float glowIntensity;
  varying vec2 vUv;
  
  void main() {
    // 计算边缘距离（从中心到边缘的渐变）
    float edge = max(
      abs(vUv.x - 0.5) * 2.0,
      abs(vUv.y - 0.5) * 2.0
    );
    
    // 计算光晕强度（边缘最强，中心最弱）
    float glow = pow(1.0 - edge, 4.0) * glowIntensity;
    
    // 设置颜色和透明度
    gl_FragColor = vec4(glowColor * glow, glow);
  }
`;

export default {
  name: 'App',
  setup() {
    const container = ref(null);
    const glowColor = ref('#5b9cff');
    const glowIntensity = ref('3');
    const rotationSpeed = ref('0.5');
    
    let scene, camera, renderer, plane, frameId;
    
    const initThreeJS = () => {
      // 创建场景
      scene = new THREE.Scene();
      
      // 创建相机
      const aspectRatio = container.value.clientWidth / container.value.clientHeight;
      camera = new THREE.PerspectiveCamera(75, aspectRatio, 0.1, 1000);
      camera.position.z = 5;
      
      // 创建渲染器
      renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
      renderer.setSize(container.value.clientWidth, container.value.clientHeight);
      renderer.setClearColor(0x000000, 0);
      container.value.appendChild(renderer.domElement);
      
      // 添加环境光
      const ambientLight = new THREE.AmbientLight(0x333333);
      scene.add(ambientLight);
      
      // 添加点光源
      const pointLight = new THREE.PointLight(0xffffff, 1);
      pointLight.position.set(5, 5, 5);
      scene.add(pointLight);
      
      // 创建平面几何体
      const geometry = new THREE.BoxGeometry(3, 3, 0.05);
      
      // 创建自定义着色器材质
      const material = new THREE.ShaderMaterial({
        uniforms: {
          glowColor: { value: new THREE.Color(glowColor.value) },
          glowIntensity: { value: parseFloat(glowIntensity.value) }
        },
        vertexShader: vertexShader,
        fragmentShader: fragmentShader,
        transparent: true,
        side: THREE.DoubleSide
      });
      
      // 创建平面网格
      plane = new THREE.Mesh(geometry, material);
      scene.add(plane);
      
      // 添加坐标轴辅助工具
      const axesHelper = new THREE.AxesHelper(5);
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
      
      // 旋转平面
      if (plane) {
        plane.rotation.x += parseFloat(rotationSpeed.value) * 0.01;
        plane.rotation.y += parseFloat(rotationSpeed.value) * 0.01;
      }
      
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
      if (plane) {
        plane.material.uniforms.glowColor.value = new THREE.Color(newColor);
      }
    });
    
    watch(glowIntensity, (newIntensity) => {
      if (plane) {
        plane.material.uniforms.glowIntensity.value = parseFloat(newIntensity);
      }
    });
    
    return {
      container,
      glowColor,
      glowIntensity,
      rotationSpeed
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
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
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
  padding: 20px 0 40px;
}

h1 {
  font-size: 2.8rem;
  margin-bottom: 10px;
  background: linear-gradient(90deg, #ff6b6b, #ffa86b, #ffda6b, #6bff9e, #6bc7ff, #9d6bff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.header p {
  font-size: 1.2rem;
  color: #a0a0c0;
  max-width: 600px;
  margin: 0 auto;
}

.controls {
  display: flex;
  justify-content: center;
  gap: 30px;
  padding: 20px;
  background: rgba(0, 0, 0, 0.3);
  border-radius: 15px;
  margin-bottom: 30px;
  flex-wrap: wrap;
  max-width: 1000px;
  margin-left: auto;
  margin-right: auto;
}

.control-group {
  display: flex;
  align-items: center;
  gap: 10px;
}

label {
  font-weight: 600;
  color: #6bc7ff;
}

input[type="color"] {
  width: 60px;
  height: 35px;
  border: 2px solid #4a4a80;
  border-radius: 6px;
  cursor: pointer;
  background: #0d0d1b;
}

input[type="range"] {
  width: 150px;
  height: 8px;
  -webkit-appearance: none;
  background: linear-gradient(to right, #5b9cff, #9d6bff);
  border-radius: 4px;
  outline: none;
}

input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #fff;
  cursor: pointer;
  box-shadow: 0 0 10px rgba(91, 156, 255, 0.8);
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
  border-radius: 15px;
  overflow: hidden;
  background: rgba(0, 0, 0, 0.2);
  box-shadow: 0 0 30px rgba(91, 156, 255, 0.2);
}

.info-panel {
  position: absolute;
  bottom: 20px;
  left: 20px;
  background: rgba(0, 0, 0, 0.7);
  padding: 20px;
  border-radius: 15px;
  max-width: 320px;
  backdrop-filter: blur(10px);
  border: 1px solid rgba(91, 156, 255, 0.3);
}

.info-panel h3 {
  margin-bottom: 15px;
  color: #6bc7ff;
  font-size: 1.3rem;
}

.info-panel p {
  line-height: 1.6;
  color: #c0c0e0;
  margin-bottom: 10px;
}

@media (max-width: 768px) {
  .controls {
    flex-direction: column;
    align-items: center;
    gap: 15px;
  }
  
  h1 {
    font-size: 2rem;
  }
  
  .info-panel {
    position: relative;
    margin-top: 20px;
    left: 0;
    max-width: 100%;
  }
}
</style>