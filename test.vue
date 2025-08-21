<template>
  <div ref="container" class="three-container"></div>
</template>

<script>
import * as THREE from 'three';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';

export default {
  name: 'GlowingPlane',
  data() {
    return {
      scene: null,
      camera: null,
      renderer: null,
      composer: null,
      plane: null,
      animationId: null
    };
  },
  mounted() {
    this.initThree();
    this.animate();
    window.addEventListener('resize', this.handleResize);
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.handleResize);
    cancelAnimationFrame(this.animationId);
    if (this.renderer) {
      this.renderer.dispose();
    }
  },
  methods: {
    initThree() {
      // 初始化场景
      this.scene = new THREE.Scene();
      
      // 初始化相机
      this.camera = new THREE.PerspectiveCamera(
        75,
        this.$refs.container.clientWidth / this.$refs.container.clientHeight,
        0.1,
        1000
      );
      this.camera.position.set(0, 5, 5);
      this.camera.lookAt(0, 0, 0);
      
      // 初始化渲染器
      this.renderer = new THREE.WebGLRenderer({ antialias: true });
      this.renderer.setSize(
        this.$refs.container.clientWidth,
        this.$refs.container.clientHeight
      );
      this.$refs.container.appendChild(this.renderer.domElement);
      
      // 创建灰色平面
      const planeGeometry = new THREE.BoxGeometry(5, 5, 0.05);
      const planeMaterial = new THREE.MeshStandardMaterial({
        color: 0x808080, // 灰色
        emissive: 0x0000ff, // 蓝色自发光
        emissiveIntensity: 0.5 // 自发光强度
      });
      this.plane = new THREE.Mesh(planeGeometry, planeMaterial);
      this.plane.rotation.x = -Math.PI / 2; // 使平面水平
      this.scene.add(this.plane);
      
      // 添加光源
      const ambientLight = new THREE.AmbientLight(0x404040);
      this.scene.add(ambientLight);
      
      const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
      directionalLight.position.set(1, 1, 1);
      this.scene.add(directionalLight);
      
      // 初始化后处理
      this.composer = new EffectComposer(this.renderer);
      this.composer.addPass(new RenderPass(this.scene, this.camera));
      
      // 创建Bloom效果
      const bloomPass = new UnrealBloomPass(
        new THREE.Vector2(
          this.$refs.container.clientWidth,
          this.$refs.container.clientHeight
        ),
        1.5, // 强度
        0.4, // 半径
        0.85 // 阈值
      );
      this.composer.addPass(bloomPass);
      
      // 调整Bloom参数
      bloomPass.threshold = 0;
      bloomPass.strength = 2;
      bloomPass.radius = 0.5;
    },
    animate() {
      this.animationId = requestAnimationFrame(this.animate);
      this.composer.render();
    },
    handleResize() {
      this.camera.aspect =
        this.$refs.container.clientWidth / this.$refs.container.clientHeight;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(
        this.$refs.container.clientWidth,
        this.$refs.container.clientHeight
      );
      this.composer.setSize(
        this.$refs.container.clientWidth,
        this.$refs.container.clientHeight
      );
    }
  }
};
</script>

<style scoped>
.three-container {
  width: 100%;
  height: 100vh;
  position: relative;
  overflow: hidden;
}
</style>
