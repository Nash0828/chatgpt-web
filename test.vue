// 修改后的代码，按service节点组织路径
const pathsByService = {};

serviceNodes.forEach(serviceNode => {
  const servicePaths = [];
  const visited = new Set();
  
  // 修改DFS函数为内部函数，可以访问servicePaths
  function dfs(currentNodeId, currentPath) {
    const currentNode = graph.nodes.find(node => node.id === currentNodeId);
    currentPath.push(currentNode);
    
    if (currentNode.nodeType !== 'serviceInstance') {
      servicePaths.push([...currentPath]);
      currentPath.pop();
      return;
    }
    
    visited.add(currentNodeId);
    
    const neighbors = adjacencyList[currentNodeId] || [];
    for (const neighborId of neighbors) {
      if (!visited.has(neighborId)) {
        dfs(neighborId, currentPath);
      }
    }
    
    visited.delete(currentNodeId);
    currentPath.pop();
  }
  
  dfs(serviceNode.id, []);
  pathsByService[serviceNode.id] = servicePaths;
});

console.log(pathsByService);