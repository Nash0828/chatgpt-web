<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>斜向斑马条纹背景</title>
    <style>
        .container {
            width: 100%;
            max-width: 800px;
            margin: 40px auto;
            padding: 20px;
        }
        
        .zebra-box {
            width: 100%;
            height: 300px;
            border: 1px solid #ccc;
            border-radius: 8px;
            margin-bottom: 30px;
            position: relative;
            overflow: hidden;
        }
        
        /* 方法1: 使用重复线性渐变 */
        .method-1 {
            background: repeating-linear-gradient(
                45deg,
                #f0f0f0,
                #f0f0f0 10px,
                #e0e0e0 10px,
                #e0e0e0 20px
            );
        }
        
        /* 方法2: 使用多个渐变组合 */
        .method-2 {
            background: 
                linear-gradient(45deg, rgba(0,0,0,0.1) 25%, transparent 25%, transparent 50%, 
                rgba(0,0,0,0.1) 50%, rgba(0,0,0,0.1) 75%, transparent 75%, transparent);
            background-size: 20px 20px;
        }
        
        /* 方法3: 使用伪元素创建斜条纹 */
        .method-3 {
            background-color: #f0f0f0;
            position: relative;
            overflow: hidden;
        }
        
        .method-3::before {
            content: "";
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background: repeating-linear-gradient(
                45deg,
                transparent,
                transparent 10px,
                rgba(0,0,0,0.05) 10px,
                rgba(0,0,0,0.05) 20px
            );
            z-index: 1;
        }
        
        .content {
            position: relative;
            z-index: 2;
            padding: 20px;
            color: #333;
            font-family: Arial, sans-serif;
        }
        
        h2 {
            color: #2c3e50;
        }
        
        .method-label {
            position: absolute;
            top: 10px;
            right: 10px;
            background: rgba(0,0,0,0.7);
            color: white;
            padding: 5px 10px;
            border-radius: 4px;
            font-size: 14px;
            z-index: 3;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>斜向斑马条纹背景实现方法</h1>
        
        <div class="zebra-box method-1">
            <div class="method-label">方法1: repeating-linear-gradient</div>
            <div class="content">
                <h2>重复线性渐变实现</h2>
                <p>使用CSS的repeating-linear-gradient属性创建斜向斑马条纹。</p>
            </div>
        </div>
        
        <div class="zebra-box method-2">
            <div class="method-label">方法2: linear-gradient + background-size</div>
            <div class="content">
                <h2>线性渐变配合背景尺寸</h2>
                <p>使用linear-gradient创建图案，然后通过background-size控制重复。</p>
            </div>
        </div>
        
        <div class="zebra-box method-3">
            <div class="method-label">方法3: 伪元素叠加</div>
            <div class="content">
                <h2>使用伪元素创建条纹</h2>
                <p>在伪元素上应用条纹背景，实现内容与背景的分离。</p>
            </div>
        </div>
    </div>
</body>
</html>