import * as THREE from 'three';

class GradientGridHelper {
  constructor(size, divisions, color = 0x888888, centerAlpha = 0.8, edgeAlpha = 0.1) {
    this.size = size;
    this.divisions = divisions;
    
    // 创建几何体
    const geometry = new THREE.BufferGeometry();
    const vertices = [];
    
    const halfSize = size / 2;
    const step = size / divisions;
    
    // 生成网格线 - 每条线单独定义，确保渐变沿着线方向
    for (let i = 0; i <= divisions; i++) {
      const offset = i * step - halfSize;
      
      // 垂直线（x方向的线）
      vertices.push(-halfSize, 0, offset);  // 起点
      vertices.push(halfSize, 0, offset);   // 终点
      
      // 水平线（z方向的线）
      vertices.push(offset, 0, -halfSize);  // 起点
      vertices.push(offset, 0, halfSize);   // 终点
    }
    
    // 创建顶点距离属性 - 为每个顶点计算到中心的距离
    const distances = [];
    const alphas = [];
    
    for (let i = 0; i < vertices.length; i += 3) {
      const x = vertices[i];
      const z = vertices[i + 2];
      const distance = Math.sqrt(x * x + z * z);
      distances.push(distance);
      
      // 计算透明度（在CPU端先计算一个基础值）
      const maxDistance = Math.sqrt(2 * halfSize * halfSize);
      const normalizedDistance = distance / maxDistance;
      const alpha = centerAlpha * (1 - normalizedDistance) + edgeAlpha * normalizedDistance;
      alphas.push(alpha);
    }
    
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
    geometry.setAttribute('distance', new THREE.Float32BufferAttribute(distances, 1));
    
    // 调试：确保顶点和距离正确
    console.log('顶点数量:', vertices.length / 3);
    console.log('距离范围:', Math.min(...distances), 'to', Math.max(...distances));
    
    // 简化的着色器材质 - 先确保基本渐变有效
    const material = new THREE.ShaderMaterial({
      uniforms: {
        color: { value: new THREE.Color(color) },
        maxDistance: { value: Math.sqrt(2 * halfSize * halfSize) },
        centerAlpha: { value: centerAlpha },
        edgeAlpha: { value: edgeAlpha },
        fadePower: { value: 1.0 }
      },
      vertexShader: `
        attribute float distance;
        varying float vAlpha;
        uniform float maxDistance;
        uniform float centerAlpha;
        uniform float edgeAlpha;
        uniform float fadePower;
        
        void main() {
          // 计算归一化距离
          float normalizedDistance = distance / maxDistance;
          
          // 使用fadePower控制渐变
          float distanceFactor = pow(normalizedDistance, fadePower);
          
          // 计算顶点透明度
          vAlpha = mix(centerAlpha, edgeAlpha, distanceFactor);
          
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: `
        varying float vAlpha;
        uniform vec3 color;
        
        void main() {
          gl_FragColor = vec4(color, vAlpha);
        }
      `,
      transparent: true,
      depthWrite: false
    });
    
    this.mesh = new THREE.LineSegments(geometry, material);
  }
  
  getMesh() {
    return this.mesh;
  }
  
  setFadePower(power) {
    this.mesh.material.uniforms.fadePower.value = power;
  }
}