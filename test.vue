// 即使没有法线贴图，也可以创建简单的法线贴图来增强立体感
const createSimpleNormalMap = (intensity = 1.0) => {
  const size = 128;
  const data = new Uint8Array(size * size * 4);
  
  for (let i = 0; i < size; i++) {
    for (let j = 0; j < size; j++) {
      const index = (i * size + j) * 4;
      // 创建简单的法线图案
      data[index] = 128 + Math.sin(i * 0.1) * 127 * intensity;     // R
      data[index + 1] = 128 + Math.cos(j * 0.1) * 127 * intensity; // G
      data[index + 2] = 255; // B (指向正前方)
      data[index + 3] = 255; // A
    }
  }
  
  const texture = new THREE.DataTexture(data, size, size, THREE.RGBAFormat);
  texture.needsUpdate = true;
  return texture;
};

// 应用法线贴图
loader.load('model.glb', (gltf) => {
  const model = gltf.scene;
  
  model.traverse((node) => {
    if (node.isMesh) {
      const material = node.material;
      
      if (material.isMeshStandardMaterial && !material.normalMap) {
        material.normalMap = createSimpleNormalMap(0.3);
        material.normalScale.set(0.5, 0.5);
        material.needsUpdate = true;
      }
    }
  });
});