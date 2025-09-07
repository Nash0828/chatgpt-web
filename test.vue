<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>贝塞尔曲线拖尾动画</title>
    <script src="https://d3js.org/d3.v7.min.js"></script>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            margin: 0;
            padding: 20px;
            background: linear-gradient(135deg, #1a2a6c, #b21f1f, #fdbb2d);
            color: white;
            display: flex;
            flex-direction: column;
            align-items: center;
            min-height: 100vh;
        }
        
        .container {
            background: rgba(0, 0, 0, 0.7);
            border-radius: 15px;
            padding: 20px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
            max-width: 1000px;
            width: 90%;
        }
        
        h1 {
            text-align: center;
            margin-bottom: 30px;
            text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
        }
        
        .svg-container {
            display: flex;
            justify-content: center;
            margin-bottom: 20px;
        }
        
        #bezier-canvas {
            background: rgba(255, 255, 255, 0.1);
            border-radius: 8px;
        }
        
        .controls {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 15px;
            margin-top: 20px;
            padding: 20px;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 10px;
        }
        
        .control-group {
            display: flex;
            flex-direction: column;
        }
        
        label {
            margin-bottom: 5px;
            font-weight: bold;
        }
        
        input[type="range"] {
            width: 100%;
        }
        
        .value-display {
            font-size: 0.9em;
            color: #fdbb2d;
            margin-top: 5px;
        }
        
        .buttons {
            display: flex;
            justify-content: center;
            gap: 15px;
            margin-top: 20px;
        }
        
        button {
            padding: 10px 20px;
            border: none;
            border-radius: 5px;
            background: #fdbb2d;
            color: #1a2a6c;
            font-weight: bold;
            cursor: pointer;
            transition: all 0.3s;
        }
        
        button:hover {
            background: #ffcc44;
            transform: translateY(-2px);
            box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
        }
        
        .info-panel {
            margin-top: 20px;
            padding: 15px;
            background: rgba(255, 255, 255, 0.1);
            border-radius: 10px;
            font-size: 0.9em;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>D3.js 贝塞尔曲线拖尾动画</h1>
        
        <div class="svg-container">
            <svg id="bezier-canvas" width="800" height="400"></svg>
        </div>
        
        <div class="controls">
            <div class="control-group">
                <label for="animation-speed">动画速度</label>
                <input type="range" id="animation-speed" min="0.1" max="5" step="0.1" value="1">
                <div class="value-display">值: 1.0</div>
            </div>
            
            <div class="control-group">
                <label for="trail-length">拖尾长度</label>
                <input type="range" id="trail-length" min="10" max="100" step="5" value="50">
                <div class="value-display">值: 50</div>
            </div>
            
            <div class="control-group">
                <label for="trail-width">拖尾宽度</label>
                <input type="range" id="trail-width" min="1" max="10" step="1" value="3">
                <div class="value-display">值: 3</div>
            </div>
            
            <div class="control-group">
                <label for="curve-tension">曲线曲率</label>
                <input type="range" id="curve-tension" min="0.1" max="2" step="0.1" value="1">
                <div class="value-display">值: 1.0</div>
            </div>
        </div>
        
        <div class="buttons">
            <button id="play-pause">暂停</button>
            <button id="reset">重置</button>
            <button id="randomize">随机曲线</button>
        </div>
        
        <div class="info-panel">
            <p>这个示例展示了如何使用D3.js创建带有拖尾动画效果的贝塞尔曲线。拖尾会沿着曲线的弯曲路径移动，使用路径点计算和线性插值实现平滑的动画效果。</p>
        </div>
    </div>

    <script>
        // 设置和初始化
        const svg = d3.select('#bezier-canvas');
        const width = +svg.attr('width');
        const height = +svg.attr('height');
        
        // 初始控制点
        let controlPoints = [
            { x: 100, y: 300 },
            { x: 200, y: 100 },
            { x: 400, y: 100 },
            { x: 500, y: 300 },
            { x: 600, y: 200 },
            { x: 700, y: 350 }
        ];
        
        // 动画状态
        let animationId = null;
        let isPlaying = true;
        let progress = 0;
        let speed = 1;
        let trailLength = 50;
        let trailWidth = 3;
        let curveTension = 1;
        
        // 创建曲线生成器
        const line = d3.line()
            .curve(d3.curveBasis);
        
        // 创建拖尾组
        const trailGroup = svg.append('g');
        
        // 绘制贝塞尔曲线
        function drawBezierCurve() {
            // 清除之前的曲线
            svg.selectAll('.bezier-curve').remove();
            
            // 绘制新的曲线
            svg.append('path')
                .attr('class', 'bezier-curve')
                .attr('d', line(controlPoints))
                .attr('fill', 'none')
                .attr('stroke', 'rgba(255, 255, 255, 0.3)')
                .attr('stroke-width', 2);
        }
        
        // 计算路径上的点
        function getPointsOnPath() {
            const path = svg.select('.bezier-curve').node();
            if (!path) return [];
            
            const points = [];
            const pathLength = path.getTotalLength();
            const numPoints = 200; // 采样点数
            
            for (let i = 0; i <= numPoints; i++) {
                const point = path.getPointAtLength(i * pathLength / numPoints);
                points.push({
                    x: point.x,
                    y: point.y
                });
            }
            
            return points;
        }
        
        // 创建拖尾动画
        function createTrailAnimation() {
            const points = getPointsOnPath();
            if (points.length === 0) return;
            
            // 清除之前的拖尾
            trailGroup.selectAll('*').remove();
            
            // 计算当前进度对应的点
            const totalPoints = points.length;
            const currentIndex = Math.floor(progress * totalPoints) % totalPoints;
            
            // 绘制拖尾
            for (let i = 0; i < trailLength; i++) {
                const index = (currentIndex - i + totalPoints) % totalPoints;
                const point = points[index];
                
                if (point) {
                    const opacity = 1 - (i / trailLength);
                    const radius = trailWidth * opacity;
                    
                    trailGroup.append('circle')
                        .attr('cx', point.x)
                        .attr('cy', point.y)
                        .attr('r', radius)
                        .attr('fill', `rgba(255, ${200 * opacity}, ${50 * opacity}, ${opacity})`);
                }
            }
            
            // 更新进度
            progress += 0.005 * speed;
            if (progress > 1) progress = 0;
        }
        
        // 动画循环
        function animate() {
            createTrailAnimation();
            animationId = requestAnimationFrame(animate);
        }
        
        // 初始化
        function init() {
            drawBezierCurve();
            if (isPlaying) {
                animate();
            }
            
            // 添加事件监听器
            d3.select('#animation-speed').on('input', function() {
                speed = +this.value;
                d3.select(this).next().text(`值: ${speed.toFixed(1)}`);
            });
            
            d3.select('#trail-length').on('input', function() {
                trailLength = +this.value;
                d3.select(this).next().text(`值: ${trailLength}`);
            });
            
            d3.select('#trail-width').on('input', function() {
                trailWidth = +this.value;
                d3.select(this).next().text(`值: ${trailWidth}`);
            });
            
            d3.select('#curve-tension').on('input', function() {
                curveTension = +this.value;
                d3.select(this).next().text(`值: ${curveTension.toFixed(1)}`);
                // 更新曲线曲率
                line.curve(d3.curveBasis.tension(curveTension));
                drawBezierCurve();
            });
            
            d3.select('#play-pause').on('click', function() {
                isPlaying = !isPlaying;
                if (isPlaying) {
                    animate();
                    this.textContent = '暂停';
                } else {
                    cancelAnimationFrame(animationId);
                    this.textContent = '播放';
                }
            });
            
            d3.select('#reset').on('click', function() {
                progress = 0;
            });
            
            d3.select('#randomize').on('click', function() {
                // 生成随机控制点
                controlPoints = [];
                const numPoints = 4 + Math.floor(Math.random() * 4);
                
                for (let i = 0; i < numPoints; i++) {
                    controlPoints.push({
                        x: 50 + Math.random() * (width - 100),
                        y: 50 + Math.random() * (height - 100)
                    });
                }
                
                drawBezierCurve();
            });
        }
        
        // 启动
        init();
    </script>
</body>
</html>