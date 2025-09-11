// 修正后的DFS实现
function dfs(currentNodeId, currentPath, visited) {
  const currentNode = graph.nodes.find(node => node.id === currentNodeId);
  
  // 将当前节点加入路径
  currentPath.push(currentNode);
  
  // 检查当前节点的邻居（子节点）
  const neighbors = adjacencyList[currentNodeId] || [];
  
  // 如果没有邻居，或者当前节点不是serviceInstance，记录路径
  if (neighbors.length === 0 || currentNode.nodeType !== 'serviceInstance') {
    allPaths.push([...currentPath]); // 记录当前路径
    currentPath.pop(); // 回溯
    return;
  }
  
  // 标记为已访问
  visited.add(currentNodeId);
  
  // 继续遍历邻居节点（只有当前节点是serviceInstance时才继续）
  for (const neighborId of neighbors) {
    if (!visited.has(neighborId)) {
      dfs(neighborId, currentPath, visited);
    }
  }
  
  // 回溯
  visited.delete(currentNodeId);
  currentPath.pop();
}