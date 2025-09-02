<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>节点连通性检测</title>
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            line-height: 1.6;
            color: #333;
            background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
            display: grid;
            grid-template-columns: 1fr 2fr;
            gap: 20px;
        }
        header {
            grid-column: 1 / -1;
            text-align: center;
            margin-bottom: 20px;
        }
        h1 {
            color: #2c3e50;
            margin-bottom: 10px;
        }
        .description {
            color: #7f8c8d;
            max-width: 800px;
            margin: 0 auto;
        }
        .control-panel {
            background: white;
            border-radius: 10px;
            padding: 20px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }
        .visualization {
            background: white;
            border-radius: 10px;
            padding: 20px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            position: relative;
            overflow: hidden;
        }
        .form-group {
            margin-bottom: 15px;
        }
        label {
            display: block;
            margin-bottom: 5px;
            font-weight: 500;
            color: #2c3e50;
        }
        select, button {
            width: 100%;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 5px;
            font-size: 16px;
        }
        button {
            background: #3498db;
            color: white;
            border: none;
            cursor: pointer;
            transition: background 0.3s;
            margin-top: 10px;
        }
        button:hover {
            background: #2980b9;
        }
        #canvas {
            width: 100%;
            height: 400px;
            border: 1px solid #ddd;
            border-radius: 5px;
        }
        .result {
            margin-top: 20px;
            padding: 15px;
            background: #f8f9fa;
            border-radius: 5px;
            max-height: 200px;
            overflow-y: auto;
        }
        .result h3 {
            margin-bottom: 10px;
            color: #2c3e50;
        }
        .path {
            margin: 5px 0;
            padding: 8px;
            background: #e8f4fc;
            border-radius: 4px;
            font-family: monospace;
        }
        .node {
            position: absolute;
            width: 50px;
            height: 50px;
            border-radius: 50%;
            background: #3498db;
            color: white;
            display: flex;
            align-items: center;
            justify-content: center;
            font-weight: bold;
            cursor: move;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            transition: transform 0.2s, background 0.3s;
        }
        .node:hover {
            transform: scale(1.05);
        }
        .node.start {
            background: #2ecc71;
        }
        .node.end {
            background: #e74c3c;
        }
        .node.highlight {
            background: #f1c40f;
        }
        .edge {
            position: absolute;
            background: #7f8c8d;
            transform-origin: 0 0;
            z-index: -1;
        }
        .edge.highlight {
            background: #f1c40f;
            height: 4px;
            z-index: 1;
        }
        .legend {
            display: flex;
            justify-content: center;
            gap: 15px;
            margin-top: 15px;
        }
        .legend-item {
            display: flex;
            align-items: center;
            gap: 5px;
        }
        .legend-color {
            width: 15px;
            height: 15px;
            border-radius: 50%;
        }
        .legend-line {
            width: 20px;
            height: 3px;
            margin: 0 5px;
        }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <h1>节点连通性检测与路径查找</h1>
            <p class="description">此工具可帮助您可视化节点拓扑关系，检测两个节点之间的连通性，并查找所有可能的路径。</p>
        </header>

        <div class="control-panel">
            <h2>控制面板</h2>
            <div class="form-group">
                <label for="start-node">起始节点:</label>
                <select id="start-node">
                    <option value="">-- 选择起始节点 --</option>
                </select>
            </div>
            <div class="form-group">
                <label for="end-node">目标节点:</label>
                <select id="end-node">
                    <option value="">-- 选择目标节点 --</option>
                </select>
            </div>
            <button id="check-btn">检测连通性</button>
            <button id="reset-btn">重置视图</button>

            <div class="result">
                <h3>检测结果:</h3>
                <div id="result-content">请选择起始节点和目标节点后点击检测按钮</div>
            </div>

            <div class="legend">
                <div class="legend-item">
                    <div class="legend-color" style="background: #2ecc71;"></div>
                    <span>起始节点</span>
                </div>
                <div class="legend-item">
                    <div class="legend-color" style="background: #e74c3c;"></div>
                    <span>目标节点</span>
                </div>
                <div class="legend-item">
                    <div class="legend-color" style="background: #f1c40f;"></div>
                    <span>路径高亮</span>
                </div>
            </div>
        </div>

        <div class="visualization">
            <h2>拓扑可视化</h2>
            <div id="canvas"></div>
        </div>
    </div>

    <script>
        // 示例数据
        const nodeList = [
            { id: 1, name: "节点A", x: 100, y: 100 },
            { id: 2, name: "节点B", x: 250, y: 150 },
            { id: 3, name: "节点C", x: 150, y: 250 },
            { id: 4, name: "节点D", x: 300, y: 300 },
            { id: 5, name: "节点E", x: 400, y: 200 },
            { id: 6, name: "节点F", x: 500, y: 100 },
            { id: 7, name: "节点G", x: 600, y: 300 }
        ];

        const edgeList = [
            { id: 1, from: 1, to: 2 },
            { id: 2, from: 1, to: 3 },
            { id: 3, from: 2, to: 4 },
            { id: 4, from: 3, to: 4 },
            { id: 5, from: 4, to: 5 },
            { id: 6, from: 5, to: 6 },
            { id: 7, from: 5, to: 7 },
            { id: 8, from: 6, to: 7 }
        ];

        // 初始化
        document.addEventListener('DOMContentLoaded', function() {
            initNodeDropdowns();
            renderGraph();
            initEventListeners();
        });

        // 初始化节点下拉框
        function initNodeDropdowns() {
            const startNodeSelect = document.getElementById('start-node');
            const endNodeSelect = document.getElementById('end-node');
            
            nodeList.forEach(node => {
                const option1 = document.createElement('option');
                option1.value = node.id;
                option1.textContent = node.name;
                
                const option2 = option1.cloneNode(true);
                
                startNodeSelect.appendChild(option1);
                endNodeSelect.appendChild(option2);
            });
        }

        // 渲染图
        function renderGraph() {
            const canvas = document.getElementById('canvas');
            canvas.innerHTML = '';
            
            // 先绘制边
            edgeList.forEach(edge => {
                const fromNode = nodeList.find(n => n.id === edge.from);
                const toNode = nodeList.find(n => n.id === edge.to);
                
                if (fromNode && toNode) {
                    const edgeElement = document.createElement('div');
                    edgeElement.className = 'edge';
                    edgeElement.id = `edge-${edge.id}`;
                    
                    const length = Math.sqrt(Math.pow(toNode.x - fromNode.x, 2) + Math.pow(toNode.y - fromNode.y, 2));
                    const angle = Math.atan2(toNode.y - fromNode.y, toNode.x - fromNode.x) * 180 / Math.PI;
                    
                    edgeElement.style.width = `${length}px`;
                    edgeElement.style.height = '2px';
                    edgeElement.style.left = `${fromNode.x}px`;
                    edgeElement.style.top = `${fromNode.y}px`;
                    edgeElement.style.transform = `rotate(${angle}deg)`;
                    
                    canvas.appendChild(edgeElement);
                }
            });
            
            // 再绘制节点（确保节点在顶部）
            nodeList.forEach(node => {
                const nodeElement = document.createElement('div');
                nodeElement.className = 'node';
                nodeElement.id = `node-${node.id}`;
                nodeElement.style.left = `${node.x - 25}px`;
                nodeElement.style.top = `${node.y - 25}px`;
                nodeElement.textContent = node.name;
                
                // 添加拖拽功能
                makeDraggable(nodeElement, node);
                
                canvas.appendChild(nodeElement);
            });
        }

        // 使节点可拖拽
        function makeDraggable(element, nodeData) {
            let isDragging = false;
            let offsetX, offsetY;
            
            element.addEventListener('mousedown', function(e) {
                isDragging = true;
                offsetX = e.clientX - element.getBoundingClientRect().left;
                offsetY = e.clientY - element.getBoundingClientRect().top;
                element.style.cursor = 'grabbing';
            });
            
            document.addEventListener('mousemove', function(e) {
                if (isDragging) {
                    const canvas = document.getElementById('canvas');
                    const canvasRect = canvas.getBoundingClientRect();
                    
                    let x = e.clientX - canvasRect.left - offsetX;
                    let y = e.clientY - canvasRect.top - offsetY;
                    
                    // 限制在画布内
                    x = Math.max(0, Math.min(canvasRect.width - 50, x));
                    y = Math.max(0, Math.min(canvasRect.height - 50, y));
                    
                    element.style.left = `${x}px`;
                    element.style.top = `${y}px`;
                    
                    // 更新节点数据
                    nodeData.x = x + 25;
                    nodeData.y = y + 25;
                    
                    // 更新边
                    updateEdges(nodeData.id);
                }
            });
            
            document.addEventListener('mouseup', function() {
                isDragging = false;
                element.style.cursor = 'grab';
            });
        }

        // 更新边
        function updateEdges(nodeId) {
            const node = nodeList.find(n => n.id === nodeId);
            if (!node) return;
            
            // 找到所有与该节点相连的边
            const connectedEdges = edgeList.filter(edge => 
                edge.from === nodeId || edge.to === nodeId
            );
            
            connectedEdges.forEach(edge => {
                const fromNode = nodeList.find(n => n.id === edge.from);
                const toNode = nodeList.find(n => n.id === edge.to);
                
                if (fromNode && toNode) {
                    const edgeElement = document.getElementById(`edge-${edge.id}`);
                    
                    const length = Math.sqrt(Math.pow(toNode.x - fromNode.x, 2) + Math.pow(toNode.y - fromNode.y, 2));
                    const angle = Math.atan2(toNode.y - fromNode.y, toNode.x - fromNode.x) * 180 / Math.PI;
                    
                    edgeElement.style.width = `${length}px`;
                    edgeElement.style.left = `${fromNode.x}px`;
                    edgeElement.style.top = `${fromNode.y}px`;
                    edgeElement.style.transform = `rotate(${angle}deg)`;
                }
            });
        }

        // 初始化事件监听器
        function initEventListeners() {
            document.getElementById('check-btn').addEventListener('click', checkConnectivity);
            document.getElementById('reset-btn').addEventListener('click', resetView);
        }

        // 构建图结构
        function buildGraph() {
            const graph = {};
            nodeList.forEach(node => {
                graph[node.id] = [];
            });
            
            edgeList.forEach(edge => {
                graph[edge.from].push(edge.to);
                graph[edge.to].push(edge.from); // 无向图，双向添加
            });
            
            return graph;
        }

        // 检测连通性
        function checkConnectivity() {
            const startNodeId = parseInt(document.getElementById('start-node').value);
            const endNodeId = parseInt(document.getElementById('end-node').value);
            
            if (!startNodeId || !endNodeId) {
                alert('请选择起始节点和目标节点');
                return;
            }
            
            // 清除之前的高亮
            clearHighlights();
            
            // 标记选中的节点
            document.getElementById(`node-${startNodeId}`).classList.add('start');
            document.getElementById(`node-${endNodeId}`).classList.add('end');
            
            const graph = buildGraph();
            const paths = findAllPaths(graph, startNodeId, endNodeId);
            
            displayResult(paths, startNodeId, endNodeId);
            highlightPaths(paths);
        }

        // 查找所有路径
        function findAllPaths(graph, start, end) {
            const paths = [];
            const visited = new Set();
            
            function dfs(node, path) {
                visited.add(node);
                path.push(node);
                
                if (node === end) {
                    paths.push([...path]);
                } else {
                    for (const neighbor of graph[node]) {
                        if (!visited.has(neighbor)) {
                            dfs(neighbor, path);
                        }
                    }
                }
                
                path.pop();
                visited.delete(node);
            }
            
            dfs(start, []);
            return paths;
        }

        // 显示结果
        function displayResult(paths, startNodeId, endNodeId) {
            const resultDiv = document.getElementById('result-content');
            
            if (paths.length === 0) {
                resultDiv.innerHTML = `<p>节点 ${getNodeName(startNodeId)} 和节点 ${getNodeName(endNodeId)} 之间没有连通路径。</p>`;
                return;
            }
            
            resultDiv.innerHTML = `
                <p>找到 ${paths.length} 条从 ${getNodeName(startNodeId)} 到 ${getNodeName(endNodeId)} 的路径:</p>
            `;
            
            paths.forEach((path, index) => {
                const pathNodes = path.map(id => getNodeName(id)).join(' → ');
                resultDiv.innerHTML += `
                    <div class="path">路径 ${index + 1}: ${pathNodes}</div>
                `;
            });
        }

        // 高亮显示路径
        function highlightPaths(paths) {
            paths.forEach(path => {
                // 高亮节点
                path.forEach(nodeId => {
                    const nodeElement = document.getElementById(`node-${nodeId}`);
                    if (nodeElement) {
                        nodeElement.classList.add('highlight');
                    }
                });
                
                // 高亮边
                for (let i = 0; i < path.length - 1; i++) {
                    const fromId = path[i];
                    const toId = path[i + 1];
                    
                    // 查找边
                    const edge = edgeList.find(e => 
                        (e.from === fromId && e.to === toId) || 
                        (e.from === toId && e.to === fromId)
                    );
                    
                    if (edge) {
                        const edgeElement = document.getElementById(`edge-${edge.id}`);
                        if (edgeElement) {
                            edgeElement.classList.add('highlight');
                        }
                    }
                }
            });
        }

        // 清除高亮
        function clearHighlights() {
            // 清除节点高亮
            document.querySelectorAll('.node').forEach(node => {
                node.classList.remove('start', 'end', 'highlight');
            });
            
            // 清除边高亮
            document.querySelectorAll('.edge').forEach(edge => {
                edge.classList.remove('highlight');
            });
        }

        // 重置视图
        function resetView() {
            // 重置节点位置
            nodeList[0].x = 100; nodeList[0].y = 100;
            nodeList[1].x = 250; nodeList[1].y = 150;
            nodeList[2].x = 150; nodeList[2].y = 250;
            nodeList[3].x = 300; nodeList[3].y = 300;
            nodeList[4].x = 400; nodeList[4].y = 200;
            nodeList[5].x = 500; nodeList[5].y = 100;
            nodeList[6].x = 600; nodeList[6].y = 300;
            
            // 重置下拉选择
            document.getElementById('start-node').value = '';
            document.getElementById('end-node').value = '';
            
            // 清除结果
            document.getElementById('result-content').textContent = '请选择起始节点和目标节点后点击检测按钮';
            
            // 清除高亮并重新渲染
            clearHighlights();
            renderGraph();
        }

        // 根据ID获取节点名称
        function getNodeName(nodeId) {
            const node = nodeList.find(n => n.id === nodeId);
            return node ? node.name : `节点${nodeId}`;
        }
    </script>
</body>
</html>