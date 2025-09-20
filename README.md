# Black_Hole_Simulation















<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Realistic Black Hole Simulation</title>
    <style>
        body { margin: 0; }
        canvas { display: block; }
    </style>
</head>
<body>
    <canvas id="canvas"></canvas>
    <script>
        // Get canvas and WebGL context
        const canvas = document.getElementById('canvas');
        const gl = canvas.getContext('webgl');
        if (!gl) {
            alert('WebGL not supported');
        }

        // Resize canvas to window
        function resizeCanvas() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
            gl.viewport(0, 0, canvas.width, canvas.height);
        }
        resizeCanvas();
        window.addEventListener('resize', resizeCanvas);

        // Vertex shader (full-screen quad)
        const vsSource = `
            attribute vec4 aPosition;
            void main() {
                gl_Position = aPosition;
            }
        `;

        // Fragment shader with black hole physics
        const fsSource = `
            precision highp float;
            uniform vec2 uResolution;
            uniform float uTime;

            // Procedural starfield background (lensed sky)
            vec3 starfield(vec2 uv) {
                float stars = 0.0;
                float fl = floor(uv.x * 100.0) + floor(uv.y * 100.0);
                stars += step(0.99, sin(fl * 123.456) * 0.5 + 0.5) * 0.8;
                vec3 col = mix(vec3(0.0, 0.0, 0.1), vec3(1.0, 1.0, 0.8), stars);
                col += vec3(0.1, 0.05, 0.2) * sin(uv.x * 2.0 + uTime * 0.1);
                return col;
            }

            void main() {
                vec2 uv = (gl_FragCoord.xy / uResolution) - 0.5;
                uv.x *= uResolution.x / uResolution.y; // Correct aspect ratio

                float r = length(uv);
                float M = 0.05; // Normalized Schwarzschild radius (adjust for size)
                float critical_b = 5.196 * M; // Approx photon sphere impact parameter

                if (r < M) {
                    // Inside event horizon: black
                    gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0);
                } else {
                    float b = r; // Impact parameter (normalized)
                    if (b < critical_b) {
                        // Ray captured by photon sphere: black
                        gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0);
                    } else {
                        // Approximate deflection (weak field)
                        float deflection = 4.0 * M / b;
                        float theta = atan(uv.y, uv.x);
                        theta -= deflection; // Bend the incoming angle

                        // Compute distorted direction
                        vec2 distorted = vec2(cos(theta), sin(theta)) * (b + deflection * 0.5);

                        // Sample background with some Doppler-like tint for realism
                        vec3 color = starfield(distorted);
                        color *= vec3(1.0, 0.8 + sin(uTime * 0.5) * 0.2, 0.9); // Simple relativistic effect sim

                        gl_FragColor = vec4(color, 1.0);
                    }
                }
            }
        `;

        // Compile shader
        function compileShader(source, type) {
            const shader = gl.createShader(type);
            gl.shaderSource(shader, source);
            gl.compileShader(shader);
            if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
                console.error(gl.getShaderInfoLog(shader));
            }
            return shader;
        }

        const vertexShader = compileShader(vsSource, gl.VERTEX_SHADER);
        const fragmentShader = compileShader(fsSource, gl.FRAGMENT_SHADER);

        // Link program
        const program = gl.createProgram();
        gl.attachShader(program, vertexShader);
        gl.attachShader(program, fragmentShader);
        gl.linkProgram(program);
        if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
            console.error(gl.getProgramInfoLog(program));
        }
        gl.useProgram(program);

        // Quad positions
        const positionBuffer = gl.createBuffer();
        gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
        const positions = new Float32Array([
            -1, -1,
            1, -1,
            -1, 1,
            -1, 1,
            1, -1,
            1, 1
        ]);
        gl.bufferData(gl.ARRAY_BUFFER, positions, gl.STATIC_DRAW);

        const aPosition = gl.getAttribLocation(program, 'aPosition');
        gl.enableVertexAttribArray(aPosition);
        gl.vertexAttribPointer(aPosition, 2, gl.FLOAT, false, 0, 0);

        // Uniforms
        const uResolution = gl.getUniformLocation(program, 'uResolution');
        const uTime = gl.getUniformLocation(program, 'uTime');

        // Render loop
        function render(time) {
            gl.uniform2f(uResolution, canvas.width, canvas.height);
            gl.uniform1f(uTime, time * 0.001);
            gl.drawArrays(gl.TRIANGLES, 0, 6);
            requestAnimationFrame(render);
        }
        requestAnimationFrame(render);
    </script>
</body>
</html>
