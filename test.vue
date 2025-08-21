<template>
  <div class="steam-demo">
    <div ref="container" class="canvas-container"></div>
    
    <div class="controls">
      <div class="control-group">
        <label for="intensity">蒸汽强度: {{ steamIntensity }}%</label>
        <input type="range" id="intensity" v-model="steamIntensity" min="10" max="100">
      </div>
      
      <div class="control-group">
        <label for="height">蒸汽高度: {{ steamHeight }}%</label>
        <input type="range" id="height" v-model="steamHeight" min="10" max="100">
      </div>
      
      <div class="control-group">
        <label for="speed">蒸汽速度: {{ steamSpeed }}%</label>
        <input type="range" id="speed" v-model="steamSpeed" min="10" max="100">
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'
import * as THREE from 'three'

// 响应式变量
const container = ref(null)
const steamIntensity = ref(50)
const steamHeight = ref(50)
const steamSpeed = ref(50)

// Three.js相关变量
let scene, camera, renderer, plane, steamParticles = []
let animationId = null

// 初始化Three.js场景
const initThreeJS = () => {
  // 创建场景
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x0a192f)

  // 创建相机
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.z = 5
  camera.position.y = 2

  // 创建渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(container.value.clientWidth, container.value.clientHeight)
  renderer.setPixelRatio(window.devicePixelRatio)
  container.value.appendChild(renderer.domElement)

  // 添加光源
  const ambientLight = new THREE.AmbientLight(0x404040)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
  directionalLight.position.set(1, 1, 1)
  scene.add(directionalLight)

  // 创建平面
  const planeGeometry = new THREE.BoxGeometry(3, 0.05, 2)
  const planeMaterial = new THREE.MeshPhongMaterial({ 
    color: 0x3498db,
    shininess: 100
  })
  plane = new THREE.Mesh(planeGeometry, planeMaterial)
  plane.position.y = -1
  scene.add(plane)

  // 初始化蒸汽粒子
  createSteamParticles()
}

// 创建蒸汽粒子
const createSteamParticles = () => {
  // 清除现有粒子
  steamParticles.forEach(particle => scene.remove(particle))
  steamParticles = []

  // 创建粒子系统
  const particleCount = 100 + Math.floor(steamIntensity.value / 100 * 200)
  
  for (let i = 0; i < particleCount; i++) {
    const size = 0.02 + Math.random() * 0.03
    const geometry = new THREE.SphereGeometry(size, 16, 16)
    const material = new THREE.MeshBasicMaterial({
      color: 0xffffff,
      transparent: true,
      opacity: 0.6
    })
    
    const particle = new THREE.Mesh(geometry, material)
    resetParticle(particle)
    
    scene.add(particle)
    steamParticles.push(particle)
  }
}

// 重置粒子位置和属性
const resetParticle = (particle) => {
  // 在平面四周随机位置生成粒子
  const width = 3
  const depth = 2
  
  // 随机选择平面的一边
  const side = Math.floor(Math.random() * 4)
  
  switch(side) {
    case 0: // 前边
      particle.position.x = (Math.random() - 0.5) * width
      particle.position.z = -depth/2
      break
    case 1: // 后边
      particle.position.x = (Math.random() - 0.5) * width
      particle.position.z = depth/2
      break
    case 2: // 左边
      particle.position.x = -width/2
      particle.position.z = (Math.random() - 0.5) * depth
      break
    case 3: // 右边
      particle.position.x = width/2
      particle.position.z = (Math.random() - 0.5) * depth
      break
  }
  
  // 设置粒子在平面高度
  particle.position.y = -1
  
  // 设置初始速度和随机参数
  particle.velocity = new THREE.Vector3(
    (Math.random() - 0.5) * 0.01,
    Math.random() * 0.02 + 0.01,
    (Math.random() - 0.5) * 0.01
  )
  
  particle.initialY = particle.position.y
  particle.opacity = 0
  particle.fadeIn = true
}

// 更新蒸汽效果
const updateSteam = () => {
  const intensityFactor = steamIntensity.value / 100
  const heightFactor = steamHeight.value / 100
  const speedFactor = steamSpeed.value / 100
  
  steamParticles.forEach(particle => {
    // 更新位置
    particle.position.x += particle.velocity.x * speedFactor
    particle.position.y += particle.velocity.y * speedFactor
    particle.position.z += particle.velocity.z * speedFactor
    
    // 淡入淡出效果
    if (particle.fadeIn) {
      particle.material.opacity += 0.02 * intensityFactor
      if (particle.material.opacity >= 0.6) {
        particle.fadeIn = false
      }
    } else {
      particle.material.opacity -= 0.005 * intensityFactor
    }
    
    // 重置超出范围的粒子
    const maxHeight = -1 + 4 * heightFactor
    if (particle.position.y > maxHeight || particle.material.opacity <= 0) {
      resetParticle(particle)
    }
  })
}

// 动画循环
const animate = () => {
  animationId = requestAnimationFrame(animate)
  
  // 旋转相机视角
  camera.position.x = Math.sin(Date.now() * 0.0002) * 5
  camera.position.z = Math.cos(Date.now() * 0.0002) * 5
  camera.lookAt(0, 0, 0)
  
  // 更新蒸汽效果
  updateSteam()
  
  // 渲染场景
  renderer.render(scene, camera)
}

// 处理窗口大小变化
const handleResize = () => {
  if (!camera || !renderer) return
  
  camera.aspect = container.value.clientWidth / container.value.clientHeight
  camera.updateProjectionMatrix()
  renderer.setSize(container.value.clientWidth, container.value.clientHeight)
}

// 组件挂载时初始化
onMounted(() => {
  initThreeJS()
  window.addEventListener('resize', handleResize)
  animate()
})

// 组件卸载时清理资源
onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  if (renderer) {
    renderer.dispose()
  }
  window.removeEventListener('resize', handleResize)
})

// 监听蒸汽强度变化
watch(steamIntensity, () => {
  createSteamParticles()
})
</script>

<style scoped>
.steam-demo {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
  color: white;
}

.canvas-container {
  flex: 1;
  width: 100%;
}

.controls {
  padding: 20px;
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(10px);
}

.control-group {
  margin-bottom: 15px;
}

.control-group label {
  display: block;
  margin-bottom: 8px;
  color: #4cc9f0;
}

.control-group input[type="range"] {
  width: 100%;
  height: 6px;
  -webkit-appearance: none;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 3px;
  outline: none;
}

.control-group input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #4cc9f0;
  cursor: pointer;
  box-shadow: 0 0 10px rgba(76, 201, 240, 0.5);
}
</style>