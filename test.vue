import * as THREE from 'three';

class GradientGridHelper {
  constructor(size, divisions, mainColor = 0x888888, centerColor = 0x444444) {
    this.size = size;
    this.divisions = divisions;
    
    // 创建几何体
    const geometry = new THREE.BufferGeometry();
    const vertices = [];
    const colors = [];
    
    const halfSize = size / 2;
    const step = size / divisions;
    
    // 生成网格线顶点和颜色
    for (let i = 0; i <= divisions; i++) {
      // x方向线条（平行于y轴）
      vertices.push(-halfSize, i * step - halfSize, 0);
      vertices.push(halfSize, i * step - halfSize, 0);
      
      // y方向线条（平行于x轴）
      vertices.push(i * step - halfSize, -halfSize, 0);
      vertices.push(i * step - halfSize, halfSize, 0);
    }
    
    // 设置顶点颜色用于渐变
    const vertexCount = vertices.length / 3;
    for (let i = 0; i < vertexCount; i++) {
      // 获取顶点的y坐标（在Three.js中通常是z-up，但这里我们按y-up处理）
      const y = vertices[i * 3 + 1];
      // 根据y值计算渐变透明度
      const normalizedY = Math.abs(y) / halfSize;
      const alpha = 1.0 - normalizedY; // 中心最清晰，边缘最模糊
      
      // 设置颜色（这里使用RGBA格式）
      colors.push(mainColor, alpha);
      colors.push(mainColor, alpha);
      colors.push(mainColor, alpha);
    }
    
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
    geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 4));
    
    // 自定义着色器材质
    const material = new THREE.ShaderMaterial({
      uniforms: {
        fadeDistance: { value: size * 0.8 },
        centerIntensity: { value: 1.0 },
        edgeIntensity: { value: 0.2 }
      },
      vertexShader: `
        attribute vec4 color;
        varying vec4 vColor;
        
        void main() {
          vColor = color;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `,
      fragmentShader: `
        varying vec4 vColor;
        
        void main() {
          // 使用顶点颜色的alpha通道
          gl_FragColor = vec4(vColor.rgb, vColor.a);
        }
      `,
      transparent: true,
      side: THREE.DoubleSide
    });
    
    this.mesh = new THREE.LineSegments(geometry, material);
    this.mesh.rotation.x = -Math.PI / 2; // 让网格平放在XZ平面
  }
  
  getMesh() {
    return this.mesh;
  }
  
  setFadeDistance(distance) {
    this.mesh.material.uniforms.fadeDistance.value = distance;
  }
}