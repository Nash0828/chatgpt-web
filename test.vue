// 示例：加载自定义二进制格式
const loader = new THREE.FileLoader();
loader.setResponseType('arraybuffer');

loader.load(
    'models/your-model.bin',
    function (data) {
        // 解析二进制数据
        const buffer = new Uint8Array(data);
        
        // 根据你的二进制格式进行解析
        // 这里需要你知道文件的详细格式结构
        parseCustomBinaryFormat(buffer);
    },
    function (xhr) {
        console.log((xhr.loaded / xhr.total * 100) + '% loaded');
    },
    function (error) {
        console.error('加载失败:', error);
    }
);

function parseCustomBinaryFormat(buffer) {
    // 根据你的文件格式实现解析逻辑
    // 例如：创建 BufferGeometry
    const geometry = new THREE.BufferGeometry();
    
    // 假设前 n 个字节是顶点数据
    const vertices = new Float32Array(buffer.buffer, 0, vertexCount * 3);
    geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
    
    // 添加材质
    const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
    const mesh = new THREE.Mesh(geometry, material);
    scene.add(mesh);
}