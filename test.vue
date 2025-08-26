const glowVertexShader = `
    varying vec2 vUv;
    void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
`;

const glowFragmentShader = `
    uniform vec3 glowColor;
    uniform float glowIntensity;
    uniform float glowRadius;
    varying vec2 vUv;
    
    void main() {
        // 计算到圆心的距离
        float distance = length(vUv - vec2(0.5, 0.5));
        
        // 边缘光晕效果
        float glow = smoothstep(0.5 - glowRadius, 0.5, distance);
        
        // 基础颜色
        vec4 baseColor = vec4(1.0, 1.0, 1.0, 1.0);
        
        // 混合光晕颜色
        vec4 finalColor = mix(baseColor, vec4(glowColor, 1.0), glow * glowIntensity);
        
        gl_FragColor = finalColor;
    }
`;

// 创建带光晕的材质
const glowMaterial = new THREE.ShaderMaterial({
    uniforms: {
        glowColor: { value: new THREE.Color(0x0000ff) },
        glowIntensity: { value: 0.0 }, // 初始强度为0
        glowRadius: { value: 0.1 }
    },
    vertexShader: glowVertexShader,
    fragmentShader: glowFragmentShader
});

const disk = new THREE.Mesh(geometry, glowMaterial);
scene.add(disk);

// 切换光晕效果
function toggleGlow() {
    isGlowing = !isGlowing;
    glowMaterial.uniforms.glowIntensity.value = isGlowing ? 1.0 : 0.0;
    glowMaterial.needsUpdate = true;
}