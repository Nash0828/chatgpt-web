// 创建简单的AO贴图来增强阴影
const createAOTexture = (width = 256, height = 256) => {
  const canvas = document.createElement('canvas');
  canvas.width = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d');
  
  const gradient = ctx.createRadialGradient(
    width / 2, height / 2, 0,
    width / 2, height / 2, width / 2
  );
  gradient.addColorStop(0, 'rgba(0, 0, 0, 0.8)');
  gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');
  
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, width, height);
  
  return new THREE.CanvasTexture(canvas);
};

// 应用AO贴图
loader.load('model.glb', (gltf) => {
  const model = gltf.scene;
  
  model.traverse((node) => {
    if (node.isMesh) {
      const material = node.material;
      
      if (material.isMeshStandardMaterial && !material.aoMap) {
        material.aoMap = createAOTexture();
        material.aoMapIntensity = 0.5;
        material.needsUpdate = true;
      }
    }
  });
});