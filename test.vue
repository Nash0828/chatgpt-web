import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

const loader = new GLTFLoader();
loader.load('model.glb', (gltf) => {
  const model = gltf.scene;
  
  model.traverse((node) => {
    if (node.isMesh) {
      const material = node.material;
      
      // 如果材质已经有颜色贴图，可以调整其强度
      if (material.map) {
        // 创建更饱和的颜色贴图
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');
        canvas.width = material.map.image.width;
        canvas.height = material.map.image.height;
        
        ctx.drawImage(material.map.image, 0, 0);
        const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        const data = imageData.data;
        
        // 增强颜色饱和度
        for (let i = 0; i < data.length; i += 4) {
          // 增加RGB值来加深颜色
          data[i] = Math.min(255, data[i] * 1.2);     // R
          data[i + 1] = Math.min(255, data[i + 1] * 1.2); // G
          data[i + 2] = Math.min(255, data[i + 2] * 1.2); // B
        }
        
        ctx.putImageData(imageData, 0, 0);
        
        const enhancedTexture = new THREE.CanvasTexture(canvas);
        material.map = enhancedTexture;
        material.needsUpdate = true;
      }
    }
  });
  
  scene.add(model);
});