class SimpleGradientGrid {
  constructor(size, divisions, color = 0x888888) {
    this.size = size;
    this.divisions = divisions;
    
    // 创建几何体
    const geometry = new THREE.BufferGeometry();
    const halfSize = size / 2;
    const step = size / divisions;
    
    // 创建顶点和颜色数组
    const vertices = [];
    const colors = [];
    
    const colorObj = new THREE.Color(color);
    
    // 生成网格线
    for (let i = 0; i <= divisions; i++) {
      const offset = i * step - halfSize;
      
      // 垂直线
      const startX = -halfSize;
      const endX = halfSize;
      const z = offset;
      
      // 起点和终点
      vertices.push(startX, 0, z, endX, 0, z);
      
      // 计算起点和终点的透明度
      const dist1 = Math.sqrt(startX * startX + z * z);
      const dist2 = Math.sqrt(endX * endX + z * z);
      const maxDist = Math.sqrt(2 * halfSize * halfSize);
      
      const alpha1 = 1.0 - (dist1 / maxDist);
      const alpha2 = 1.0 - (dist2 / maxDist);
      
      // 起点颜色
      colors.push(colorObj.r, colorObj.g, colorObj.b, alpha1);
      // 终点颜色
      colors.push(colorObj.r, colorObj.g, colorObj.b, alpha2);
      
      // 水平线
      const x = offset;
      const startZ = -halfSize;
      const endZ = halfSize;
      
      vertices.push(x, 0, startZ, x, 0, endZ);
      
      const dist3 = Math.sqrt(x * x + startZ * startZ);
      const dist4 = Math.sqrt(x * x + endZ * endZ);
      
      const alpha3 = 1.0 - (dist3 / maxDist);
      const alpha4 = 1.0 - (dist4 / maxDist);
      
      colors.push(colorObj.r, colorObj.g, colorObj.b, alpha3);
      colors.push(colorObj.r, colorObj.g, colorObj.b, alpha4);
    }
    
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
    geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 4));
    
    // 使用基本材质验证渐变
    const material = new THREE.LineBasicMaterial({
      vertexColors: true,
      transparent: true,
      opacity: 1.0
    });
    
    this.mesh = new THREE.LineSegments(geometry, material);
    
    console.log('网格顶点数:', vertices.length / 3);
    console.log('颜色数组长度:', colors.length / 4);
  }
  
  getMesh() {
    return this.mesh;
  }
}