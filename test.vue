<template>
  <div id="app">
    <div class="header">
      <h1>水平面四周光晕效果</h1>
      <p>平面放置在水平面上，光晕沿Y轴向上渐变流动</p>
    </div>
    
    <div class="controls">
      <div class="control-group">
        <label for="glow-color">光晕颜色:</label>
        <input type="color" id="glow-color" v-model="glowColor">
      </div>
      <div class="control-group">
        <label for="glow-intensity">光晕强度:</label>
        <input type="range" id="glow-intensity" v-model="glowIntensity" min="1" max="10" step="0.5">
        <span>{{ glowIntensity }}</span>
      </div>
      <div class="control-group">
        <label for="flow-speed">流动速度:</label>
        <input type="range" id="flow-speed" v-model="flowSpeed" min="0" max="3" step="0.1">
        <span>{{ flowSpeed }}</span>
      </div>
    </div>
    
    <div class="container">
      <div ref="container" class="canvas-container"></div>
      
      <div class="info-panel">
        <h3>实现说明</h3>
        <p>• 平面水平放置在XZ平面上</p>
        <p>• 四周使用粒子系统创建光晕效果</p>
        <p>• 光晕沿Y轴向上渐变并流动</p>
        <p>• 使用自定义着色器实现渐变和动画</p>
      </div>
    </div>
  </div>
</template>

<script>
import * as THREE from 'three';
import { ref, onMounted, onUnmounted, watch } from 'vue';

// 光晕粒子的顶点着色器
const glowVertexShader = `
  uniform float time;
  uniform float flowSpeed;
  varying float vAlpha;
  varying vec3 vColor;
  
  void main() {
    // 流动效果 - 随时间向上移动
    vec3 pos = position;
    pos.y += time * flowSpeed;
    
    // 重置位置（循环效果）
    if (pos.y > 2.0) {
      pos.y = -1.0;
    }
    
    // 计算透明度 - 底部和顶部透明，中间最亮
    float heightFactor = (pos.y + 1.0) / 3.0; // 从0到1
    vAlpha = sin(heightFactor * 3.14159); // 正弦曲线渐变
    
    // 传递颜色
    vColor = vec3(0.2, 0.5, 1.0);
    
    gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    gl_PointSize = 8.0;
  }
`;

// 光晕粒子的片段着色器
const glowFragmentShader = `
  uniform vec3 glowColor;
  varying float vAlpha;
  varying vec3 vColor;
  
  void main() {
    // 创建圆形粒子
    vec2 coord = gl_PointCoord - vec2(0.5);
    float distance = length(coord);
    if (distance > 0.5) {
      discard;
    }
    
    // 中心亮边缘暗的渐变
    float intensity = 1.0 - smoothstep(0.3, 0.5, distance);
    
    // 最终颜色
    gl_FragColor = vec4(glowColor * intensity, vAlpha * intensity);
  }
`;

export default {
  name: 'App',
  setup() {
    const container = ref(null);
    const glowColor = ref('#3a86ff');
    const glowIntensity = ref('5');
    const flowSpeed = ref('1.0');
    
    let scene, camera, renderer, plane, glowParticles, frameId;
    let clock = new THREE.Clock();
    
    const createGlowEffect = () => {
      // 创建粒子几何体
      const particleCount = 2000;
      const geometry = new THREE.BufferGeometry();
      
      const positions = new Float32Array(particleCount * 3);
      const colors = new Float32Array(particleCount * 3);
      
      // 在平面四周创建粒子
      const planeSize = 4;
      for (let i = 0; i < particleCount; i++) {
        const i3 = i * 3;
        
        // 随机分布在平面四周
        const side = Math.floor(Math.random() * 4); // 0-3: 四个边
        let x, z;
        
        switch (side) {
          case 0: // 前边
            x = (Math.random() - 0.5) * planeSize;
            z = planeSize / 2;
            break;
          case 1: // 后边
            x = (Math.random() - 0.5) * planeSize;
            z = -planeSize / 2;
            break;
          case 2: // 左边
            x = -planeSize / 2;
            z = (Math.random() - 0.5) * planeSize;
            break;
          case 3: // 右边
            x = planeSize / 2;
            z = (Math.random() - 0.5) * planeSize;
            break;
        }
        
        // 随机高度（从底部开始）
        const y = Math.random() * 2 - 1; // -1 到 1
        
        positions[i3] = x;
        positions[i3 + 1] = y;
        positions[i3 + 2] = z;
        
        // 随机颜色变化
        const colorVariation = 0.2;
        colors[i3] = 0.2 + Math.random() * colorVariation;
        colors[i3 + 1] = 0.5 + Math.random() * colorVariation;
        colors[i3 + 2] = 1.0 + Math.random() * colorVariation;
      }
      
      geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
      geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
      
      // 创建粒子材质
      const material = new THREE.ShaderMaterial({
        uniforms: {
          time: { value: 0 },
          flowSpeed: { value: parseFloat(flowSpeed.value) },
          glowColor: { value: new THREE.Color(glowColor.value) }
        },
        vertexShader: glowVertexShader,
        fragmentShader: glowFragmentShader,
        transparent: true,
        blending: THREE.AdditiveBlending,
        depthWrite: false
      });
      
      // 创建粒子系统
      glowParticles = new THREE.Points(geometry, material);
      glowParticles.position.y = 0.01; // 稍微高于平面
      scene.add(glowParticles);
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
      
      // 创建平面
      const planeGeometry = new THREE.PlaneGeometry(4, 4);
      const planeMaterial = new THREE.MeshPhongMaterial({
        color: 0x2a6bc6,
        transparent: true,
        opacity: 0.8,
        side: THREE.DoubleSide
      });
      
      plane = new THREE.Mesh(planeGeometry, planeMaterial);
      plane.rotation.x = -Math.PI / 2;
      scene.add(plane);
      
      // 创建光晕效果
      createGlowEffect();
      
      // 添加坐标轴
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
      
      // 更新时间
      if (glowParticles) {
        glowParticles.material.uniforms.time.value = clock.getElapsedTime();
      }
      
      // 缓慢旋转相机
      const time = clock.getElapsedTime() * 0.2;
      camera.position.x = 6 * Math.cos(time);
      camera.position.z = 6 * Math.sin(time);
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
      if (glowParticles) {
        glowParticles.material.uniforms.glowColor.value = new THREE.Color(newColor);
      }
    });
    
    watch(flowSpeed, (newSpeed) => {
      if (glowParticles) {
        glowParticles.material.uniforms.flowSpeed.value = parseFloat(newSpeed);
      }
    });
    
    return {
      container,
      glowColor,
      glowIntensity,
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