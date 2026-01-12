import * as THREE from 'three';

class GradientGridHelper {
  constructor(size, divisions, color = 0x888888, centerAlpha = 1.0, edgeAlpha = 0.1) {
    this.size = size;
    this.divisions = divisions;
    
    // 创建几何体
    const geometry = new THREE.BufferGeometry();
    const vertices = [];
    const alphas = [];
    
    const halfSize = size / 2;
    const step = size / divisions;
    
    // 生成网格线顶点
    for (let i = 0; i <= divisions; i++) {
      const offset = i * step - halfSize;
      
      // x方向的线（平行于y轴）
      vertices.push(-halfSize, 0, offset);
      vertices.push(halfSize, 0, offset);
      
      // y方向的线（平行于x轴） - 注意：网格在XZ平面上
      vertices.push(offset, 0, -halfSize);
      vertices.push(offset, 0, halfSize);
    }
    
    // 为每个顶点计算基于距离的透明度
    for (let i = 0; i < vertices.length; i += 6) {
      // 每条线有两个顶点
      const x1 = vertices[i];
      const z1 = vertices[i + 2];
      const x2 = vertices[i + 3];
      const z2 = vertices[i + 5];
      
      // 计算每个顶点的距离（从中心到顶点的距离）
      const distance1 = Math.sqrt(x1 * x1 + z1 * z1);
      const distance2 = Math.sqrt(x2 * x2 + z2 * z2);
      
      // 计算透明度（距离越大越透明）
      const maxDistance = Math.sqrt(2 * halfSize * halfSize); // 对角线距离
      const alpha1 = centerAlpha * (1 - distance1 / maxDistance) + edgeAlpha * (distance1 / maxDistance);
      const alpha2 = centerAlpha * (1 - distance2 / maxDistance) + edgeAlpha * (distance2 / maxDistance);
      
      alphas.push(alpha1, alpha2);
    }
    
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
    geometry.setAttribute('alpha', new THREE.Float32BufferAttribute(alphas, 1));
    
    // 自定义着色器材质
    const material = new THREE.ShaderMaterial({
      uniforms: {
        color: { value: new THREE.Color(color) },
        fadePower: { value: 1.0 } // 渐变幂次，值越大渐变越明显
      },
      vertexShader: `
        attribute float alpha;
        varying float vAlpha;
        
        void main() {
          vAlpha = alpha;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: `
        uniform vec3 color;
        varying float vAlpha;
        
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
  
  setColor(color) {
    this.mesh.material.uniforms.color.value.set(color);
  }
}