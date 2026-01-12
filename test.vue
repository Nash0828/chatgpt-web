import * as THREE from 'three';

class GradientGridHelper {
  constructor(size, divisions, color = 0x888888, centerAlpha = 0.8, edgeAlpha = 0.1) {
    this.size = size;
    this.divisions = divisions;
    
    // 创建几何体
    const geometry = new THREE.BufferGeometry();
    const vertices = [];
    const distances = []; // 存储每个顶点到中心的距离
    
    const halfSize = size / 2;
    const step = size / divisions;
    
    // 生成网格线顶点
    for (let i = 0; i <= divisions; i++) {
      const offset = i * step - halfSize;
      
      // x方向的线（平行于z轴）- 在XZ平面上
      vertices.push(-halfSize, 0, offset);
      vertices.push(halfSize, 0, offset);
      
      // z方向的线（平行于x轴）
      vertices.push(offset, 0, -halfSize);
      vertices.push(offset, 0, halfSize);
    }
    
    // 计算每个顶点到中心的距离
    for (let i = 0; i < vertices.length; i += 3) {
      const x = vertices[i];
      const z = vertices[i + 2];
      const distance = Math.sqrt(x * x + z * z);
      distances.push(distance);
    }
    
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
    geometry.setAttribute('distance', new THREE.Float32BufferAttribute(distances, 1));
    
    // 自定义着色器材质
    const material = new THREE.ShaderMaterial({
      uniforms: {
        color: { value: new THREE.Color(color) },
        fadePower: { value: 2.0 },
        maxDistance: { value: Math.sqrt(2 * halfSize * halfSize) }, // 最大距离（对角线）
        centerAlpha: { value: centerAlpha },
        edgeAlpha: { value: edgeAlpha }
      },
      vertexShader: `
        attribute float distance;
        varying float vDistance;
        
        void main() {
          vDistance = distance;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: `
        uniform vec3 color;
        uniform float fadePower;
        uniform float maxDistance;
        uniform float centerAlpha;
        uniform float edgeAlpha;
        varying float vDistance;
        
        void main() {
          // 计算归一化距离
          float normalizedDistance = vDistance / maxDistance;
          
          // 使用fadePower控制渐变曲线
          float distanceFactor = pow(normalizedDistance, fadePower);
          
          // 计算透明度：中心最清晰，边缘最模糊
          float alpha = mix(centerAlpha, edgeAlpha, distanceFactor);
          
          // 确保透明度在合理范围内
          alpha = clamp(alpha, 0.0, 1.0);
          
          gl_FragColor = vec4(color, alpha);
        }
      `,
      transparent: true,
      depthWrite: false,
      linewidth: 1
    });
    
    this.mesh = new THREE.LineSegments(geometry, material);
  }
  
  getMesh() {
    return this.mesh;
  }
  
  setFadePower(power) {
    this.mesh.material.uniforms.fadePower.value = power;
  }
  
  setAlphaRange(centerAlpha, edgeAlpha) {
    this.mesh.material.uniforms.centerAlpha.value = centerAlpha;
    this.mesh.material.uniforms.edgeAlpha.value = edgeAlpha;
  }
  
  setColor(color) {
    this.mesh.material.uniforms.color.value.set(color);
  }
  
  setSize(size) {
    // 更新网格大小需要重新创建几何体
    const halfSize = size / 2;
    this.mesh.material.uniforms.maxDistance.value = Math.sqrt(2 * halfSize * halfSize);
  }
}