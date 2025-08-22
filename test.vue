// 创建深色乘法贴图
const createMultiplyTexture = (color, width = 512, height = 512) => {
  const canvas = document.createElement('canvas');
  canvas.width = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d');
  
  ctx.fillStyle = color;
  ctx.fillRect(0, 0, width, height);
  
  return new THREE.CanvasTexture(canvas);
};

// 应用乘法贴图
loader.load('model.glb', (gltf) => {
  const model = gltf.scene;
  
  model.traverse((node) => {
    if (node.isMesh) {
      const material = node.material;
      
      // 创建深色乘法贴图（例如深棕色）
      const multiplyTexture = createMultiplyTexture('#8B4513');
      
      // 如果是标准材质，可以使用emissive贴图来增强颜色
      if (material.isMeshStandardMaterial) {
        material.emissive = new THREE.Color(0x333333); // 轻微自发光
        material.emissiveIntensity = 0.1;
      }
      
      // 或者创建自定义着色器材质来应用乘法
      material.onBeforeCompile = (shader) => {
        shader.uniforms.multiplyTexture = { value: multiplyTexture };
        
        shader.fragmentShader = shader.fragmentShader.replace(
          '#include <map_fragment>',
          `
          #include <map_fragment>
          vec4 multiplyColor = texture2D(multiplyTexture, vUv);
          diffuseColor.rgb *= multiplyColor.rgb * 1.5;
          `
        );
      };
    }
  });
});