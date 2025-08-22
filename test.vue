function calculateCircleIntersectionPoints(r1, r2) {
    // 确保r1是大圆半径，r2是小圆半径
    if (r1 < r2) {
        [r1, r2] = [r2, r1];
    }
    
    const points = [];
    // 4条直线，将圆分成8等份，每条线间隔45度
    const angles = [0, 45, 90, 135, 180, 225, 270, 315];
    
    // 计算每条线与两个圆的交点
    for (let angle of angles) {
        // 转换为弧度
        const radian = angle * Math.PI / 180;
        
        // 计算大圆上的点
        const x1 = r1 * Math.sin(radian);
        const y1 = -r1 * Math.cos(radian); // 注意：y坐标取负，因为画布坐标系y轴向下
        
        // 计算小圆上的点
        const x2 = r2 * Math.sin(radian);
        const y2 = -r2 * Math.cos(radian);
        
        // 添加大圆和小圆上的点
        points.push({x: x1, y: y1, radius: r1, angle: angle});
        points.push({x: x2, y: y2, radius: r2, angle: angle});
    }
    
    // 按指定规则排序：从6点钟方向(270度)开始顺时针，先大圆后小圆
    points.sort((a, b) => {
        // 调整角度，使270度(6点钟方向)为起点
        let angleA = (a.angle + 90) % 360;
        let angleB = (b.angle + 90) % 360;
        
        // 如果角度不同，按角度排序
        if (angleA !== angleB) {
            return angleA - angleB;
        }
        
        // 角度相同，按半径排序（大圆在前）
        return b.radius - a.radius;
    });
    
    // 只返回点的坐标数组
    return points.map(point => [point.x, point.y]);
}

// 使用示例
const r1 = 100; // 大圆半径
const r2 = 60;  // 小圆半径
const intersectionPoints = calculateCircleIntersectionPoints(r1, r2);
console.log(intersectionPoints);