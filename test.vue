<template>
  <div id="app">
    <div class="header">
      <h1>专业级平面光晕效果</h1>
      <p>使用后处理技术实现真实的发光效果</p>
    </div>
    
    <div class="controls">
      <div class="control-group">
        <label for="glow-color">光晕颜色:</label>
        <input type="color" id="glow-color" v-model="glowColor">
      </div>
      <div class="control-group">
        <label for="bloom-intensity">发光强度:</label>
        <input type="range" id="bloom-intensity" v-model="bloomIntensity" min="0.1" max="2" step="0.1">
        <span>{{ bloomIntensity }}</span>
      </div>
      <div class="control-group">
        <label for="bloom-threshold">发光阈值:</label>
        <input type="range" id="bloom-threshold" v-model="bloomThreshold" min="0" max="1" step="0.05">
        <span>{{ bloomThreshold }}</span>
      </div>
      <div class="control-group">
        <label for="rotation-speed">旋转速度:</label>
        <input type="range" id="rotation-speed" v-model="rotationSpeed" min="0" max="2" step="0.1">
        <span>{{ rotationSpeed }}</span>
      </div>
    </div>
    
    <div class="container">
      <div ref="container" class="canvas-container"></div>
      
      <div class="info-panel">
        <h3>实现说明</h3>
        <p>• 使用后处理(Post Processing)技术</p>
        <p>• 真实的Bloom发光效果</p>
        <p>• 平面边缘发射光晕</p>
        <p>• 可调节发光参数</p>
      </div>
    </div>
  </div>
</template>

<script>
import * as THREE from 'three';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
import { ref, onMounted, onUnmounted, watch } from 'vue';

export default {
  name: 'App',
  setup() {
    const container = ref(null);
    const glowColor = ref('#3a86ff');
    const bloomIntensity = ref('1.0');
    const bloomThreshold = ref('0.3');
    const rotationSpeed = ref('0.5');
    
    let scene, camera, renderer, composer, bloomPass, plane, frameId;
    let clock = new THREE.Clock();
    
    // 创建发光平面材质
    const createEmissiveMaterial = () => {
      return new THREE.ShaderMaterial({
        uniforms: {
          time: { value: 0 },
          glowColor: { value: new THREE.Color(glowColor.value) },
          glowIntensity: { value: 1.5 }
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
          uniform float time;
          uniform vec3 glowColor;
          uniform float glowIntensity;
          varying vec2 vUv;
          varying vec3 vPosition;
          
          void main() {
            // 计算边缘距离
            float edgeX = min(vUv.x, 1.0 - vUv.x) * 2.0;
            float edgeY = min(vUv.y, 1.0 - vUv.y) * 2.0;
            float edge = min(edgeX, edgeY);
            
            // 边缘发光强度
            float glow = pow(1.0 - edge, 3.0) * glowIntensity;
            
            // 添加流动效果
            float flow = sin(time * 2.0 + vPosition.x * 3.0 + vPosition.z * 3.0) * 0.3 + 0.7;
            
            // 基础颜色（稍微透明）
            vec3 baseColor = mix(vec3(0.2, 0.4, 0.8), glowColor, 0.5);
            
            // 最终颜色 - 只有边缘发光部分会被bloom处理
            vec3 finalColor = baseColor + glowColor * glow * flow;
            float alpha = 0.8 + glow * 0.2;
            
            gl_FragColor = vec4(finalColor, alpha);
          }
        `,
        transparent: true,
        side: THREE.DoubleSide
      });
    };
    
    const initThreeJS = () => {
      // 创建场景
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x0a0a1a);
      
      // 创建相机
      const aspectRatio = container.value.clientWidth / container.value.clientHeight;
      camera = new THREE.PerspectiveCamera(60, aspectRatio, 0.1, 1000);
      camera.position.set(6, 4, 6);
      camera.lookAt(0, 0, 0);
      
      // 创建渲染器
      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(container.value.clientWidth, container.value.clientHeight);
      renderer.setPixelRatio(window.devicePixelRatio);
      container.value.appendChild(renderer.domElement);
      
      // 添加灯光
      const ambientLight = new THREE.AmbientLight(0x404040, 0.6);
      scene.add(ambientLight);
      
      const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
      directionalLight.position.set(5, 10, 7);
      scene.add(directionalLight);
      
      // 创建网格地面
      const gridHelper = new THREE.GridHelper(20, 20, 0x444444, 0x222222);
      gridHelper.position.y = -0.01;
      scene.add(gridHelper);
      
      // 创建发光平面
      const planeGeometry = new THREE.PlaneGeometry(4, 4);
      const planeMaterial = createEmissiveMaterial();
      
      plane = new THREE.Mesh(planeGeometry, planeMaterial);
      plane.rotation.x = -Math.PI / 2;
      scene.add(plane);
      
      // 创建平面边框（增强发光效果）
      const edgesGeometry = new THREE.EdgesGeometry(planeGeometry);
      const lineMaterial = new THREE.LineBasicMaterial({ 
        color: 0x3a86ff, 
        linewidth: 2 
      });
      const edges = new THREE.LineSegments(edgesGeometry, lineMaterial);
      edges.rotation.x = -Math.PI / 2;
      plane.add(edges);
      
      // 设置后处理
      composer = new EffectComposer(renderer);
      composer.addPass(new RenderPass(scene, camera));
      
      // 添加Bloom效果
      bloomPass = new UnrealBloomPass(
        new THREE.Vector2(container.value.clientWidth, container.value.clientHeight),
        parseFloat(bloomIntensity.value),
        parseFloat(bloomThreshold.value),
        0.4
      );
      composer.addPass(bloomPass);
      
      // 添加坐标轴
      const axesHelper = new THREE.AxesHelper(3);
      scene.add(axesHelper);
    };
    
    const onWindowResize = () => {
      if (!camera || !renderer) return;
      
      camera.aspect = container.value.clientWidth / container.value.clientHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(container.value.clientWidth, container.value.clientHeight);
      composer.setSize(container.value.clientWidth, container.value.clientHeight);
    };
    
    const render = () => {
      frameId = requestAnimationFrame(render);
      
      // 更新时间
      const elapsedTime = clock.getElapsedTime();
      if (plane && plane.material.uniforms.time) {
        plane.material.uniforms.time.value = elapsedTime;
      }
      
      // 旋转平面
      if (plane) {
        plane.rotation.y += parseFloat(rotationSpeed.value) * 0.01;
      }
      
      // 缓慢旋转相机
      const time = elapsedTime * 0.2;
      camera.position.x = 6 * Math.cos(time);
      camera.position.z = 6 * Math.sin(time);
      camera.lookAt(0, 0, 0);
      
      composer.render();
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
      if (plane && plane.material.uniforms.glowColor) {
        plane.material.uniforms.glowColor.value = new THREE.Color(newColor);
      }
    });
    
    watch(bloomIntensity, (newIntensity) => {
      if (bloomPass) {
        bloomPass.strength = parseFloat(newIntensity);
      }
    });
    
    watch(bloomThreshold, (newThreshold) => {
      if (bloomPass) {
        bloomPass.threshold = parseFloat(newThreshold);
      }
    });
    
    return {
      container,
      glowColor,
      bloomIntensity,
      bloomThreshold,
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
  gap: 20px;
  padding: 15px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 12px;
  margin-bottom: 25px;
  flex-wrap: wrap;
  max-width: 1000px;
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
  min-width: 80px;
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
  width: 100px;
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
  
  .control-group {
    width: 100%;
    justify-content: space-between;
  }
  
  label {
    min-width: 100px;
  }
}
</style>