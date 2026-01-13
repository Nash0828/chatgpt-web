import * as THREE from 'three';

// 假设你的 .bin 文件格式：
// 1. 前4个字节：顶点数量 (uint32)
// 2. 之后：顶点数据 (每个顶点3个float32，x, y, z)

const loader = new THREE.FileLoader();
loader.setResponseType('arraybuffer');

loader.load(
    'model.bin',
    function (data) {
        const arrayBuffer = data;
        const geometry = parseBinaryGeometry(arrayBuffer);
        
        // 创建网格
        const material = new THREE.MeshNormalMaterial();
        const mesh = new THREE.Mesh(geometry, material);
        scene.add(mesh);
    }
);

function parseBinaryGeometry(buffer) {
    const dataView = new DataView(buffer);
    let byteOffset = 0;
    
    // 1. 读取顶点数量
    const vertexCount = dataView.getUint32(byteOffset, true);
    byteOffset += 4; // uint32 占4字节
    
    console.log(`顶点数量: ${vertexCount}`);
    
    // 2. 读取顶点数据
    const vertices = new Float32Array(vertexCount * 3);
    for (let i = 0; i < vertexCount * 3; i++) {
        vertices[i] = dataView.getFloat32(byteOffset, true);
        byteOffset += 4; // float32 占4字节
    }
    
    // 3. 创建几何体
    const geometry = new THREE.BufferGeometry();
    geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
    
    // 4. 计算法线（如果需要）
    geometry.computeVertexNormals();
    
    return geometry;
}