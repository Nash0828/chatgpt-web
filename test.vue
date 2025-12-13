import * as THREE from 'three';
import { Line2 } from 'three/examples/jsm/lines/Line2.js';
import { LineGeometry } from 'three/examples/jsm/lines/LineGeometry.js';
import { LineMaterial } from 'three/examples/jsm/lines/LineMaterial.js';

// 创建两个节点
const node1 = new THREE.Vector3(0, 0, 0);
const node2 = new THREE.Vector3(10, 0, 0);

// 计算控制点（用于创建弧线效果）
function createArcPoints(start, end, height = 5, segments = 50) {
  const points = [];
  
  // 计算中点并向上偏移形成弧线
  const midPoint = new THREE.Vector3()
    .addVectors(start, end)
    .multiplyScalar(0.5);
    
  midPoint.y += height; // 控制弧线高度
  
  // 二次贝塞尔曲线
  for (let i = 0; i <= segments; i++) {
    const t = i / segments;
    
    // 二次贝塞尔曲线公式
    const point = new THREE.Vector3();
    point.copy(start).multiplyScalar((1 - t) * (1 - t));
    point.addScaledVector(midPoint, 2 * (1 - t) * t);
    point.addScaledVector(end, t * t);
    
    points.push(point);
  }
  
  return points;
}

// 创建弧线几何体
function createArcLine(start, end, color = 0x00ff00, linewidth = 2) {
  const points = createArcPoints(start, end);
  const geometry = new LineGeometry();
  
  // 将点转换为几何体数据
  const positions = [];
  points.forEach(point => {
    positions.push(point.x, point.y, point.z);
  });
  
  geometry.setPositions(positions);
  
  // 创建材质
  const material = new LineMaterial({
    color: color,
    linewidth: linewidth,
    vertexColors: false,
    dashed: false,
    alphaToCoverage: true,
  });
  
  // 创建Line2对象
  const line = new Line2(geometry, material);
  line.computeLineDistances();
  
  return line;
}

// 使用示例
const arcLine = createArcLine(node1, node2, 0xff0000, 3);
scene.add(arcLine);