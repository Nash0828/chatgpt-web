// 假设你的图数据结构如下：
const graph = {
  nodes: [
    { id: 1, nodeType: 'service' },
    { id: 2, nodeType: 'serviceInstance' },
    { id: 3, nodeType: 'other' },
    // ... 更多节点
  ],
  edges: [
    { source: 1, target: 2 },
    { source: 2, target: 3 },
    // ... 更多边
  ]
};

// 获取所有service节点
const serviceNodes = graph.nodes.filter(node => node.nodeType === 'service');

// 构建邻接表，方便遍历
const adjacencyList = {};
graph.edges.forEach(edge => {
  if (!adjacencyList[edge.source]) {
    adjacencyList[edge.source] = [];
  }
  adjacencyList[edge.source].push(edge.target);
});

// 用于存储所有路径的结果
const allPaths = [];

// DFS函数
function dfs(currentNodeId, currentPath, visited) {
  // 获取当前节点
  const currentNode = graph.nodes.find(node => node.id === currentNodeId);
  
  // 将当前节点加入路径
  currentPath.push(currentNode);
  
  // 如果当前节点不是serviceInstance，停止遍历并记录路径
  if (currentNode.nodeType !== 'serviceInstance') {
    allPaths.push([...currentPath]); // 复制当前路径
    currentPath.pop(); // 回溯
    return;
  }
  
  // 标记为已访问
  visited.add(currentNodeId);
  
  // 遍历邻居节点
  const neighbors = adjacencyList[currentNodeId] || [];
  for (const neighborId of neighbors) {
    if (!visited.has(neighborId)) {
      dfs(neighborId, currentPath, visited);
    }
  }
  
  // 回溯
  visited.delete(currentNodeId);
  currentPath.pop();
}

// 对每个service节点进行遍历
serviceNodes.forEach(serviceNode => {
  const visited = new Set();
  dfs(serviceNode.id, [], visited);
});

// 输出结果
console.log(allPaths);