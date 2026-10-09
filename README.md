
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Stellar Map - Gliese Catalog FTL</title>
    <style>
        :root {
            --primary: #00d4ff;
            --bg: #020b16;
            --panel: rgba(10, 25, 41, 0.8);
        }

        body, html {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            background-color: var(--bg);
            color: white;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
        }

        #canvas {
            display: block;
        }

        /* Interface UI */
        #ui-layer {
            position: absolute;
            top: 20px;
            left: 20px;
            width: 300px;
            background: var(--panel);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.1);
            border-radius: 12px;
            padding: 20px;
            pointer-events: auto;
            box-shadow: 0 8px 32px rgba(0,0,0,0.5);
        }

        h1 { font-size: 18px; margin-bottom: 15px; color: var(--primary); text-transform: uppercase; letter-spacing: 2px; }
        
        .star-info {
            font-size: 14px;
            margin-bottom: 20px;
            border-left: 2px solid var(--primary);
            padding-left: 10px;
        }

        select {
            width: 100%;
            background: #051221;
            color: white;
            border: 1px solid var(--primary);
            padding: 8px;
            border-radius: 4px;
            margin-bottom: 10px;
        }

        button {
            width: 100%;
            background: var(--primary);
            color: #000;
            border: none;
            padding: 10px;
            border-radius: 4px;
            font-weight: bold;
            cursor: pointer;
            transition: all 0.3s;
        }

        button:hover {
            background: #fff;
            box-shadow: 0 0 15px var(--primary);
        }

        .coords {
            font-family: monospace;
            font-size: 12px;
            color: #888;
            margin-top: 10px;
        }
    </style>
</head>
<body>

    <div id="ui-layer">
        <h1>Stellar Mapper</h1>
        <div class="star-info" id="star-display">
            <strong>Alvo:</strong> Sol (Origem)<br>
            <strong>Distância:</strong> 0 LY
        </div>
        
        <label for="star-select">Destino (Catálogo Gliese):</label>
        <select id="star-select"></select>
        
        <button onclick="engageFTL()">INICIAR VIAGEM FTL</button>
        
        <div class="coords" id="cam-coords">Cam: 0, 0, 0</div>
    </div>

    <canvas id="canvas"></canvas>

    <script>
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        const starSelect = document.getElementById('star-select');
        const starDisplay = document.getElementById('star-display');
        const camCoordsDisplay = document.getElementById('cam-coords');

        // Dados Reais (Amostra do Catálogo Gliese / Próximas)
        // Coordenadas aproximadas em Anos-Luz (LY)
        const glieseData = [
            { name: "Sol", x: 0, y: 0, z: 0, mag: 2, color: "#fff" },
            { name: "Proxima Centauri", x: -1.2, y: 4.0, z: 1.1, mag: 1.5, color: "#ff6b6b" },
            { name: "Alpha Centauri A", x: -1.3, y: 4.1, z: 1.3, mag: 2.5, color: "#ffd93d" },
            { name: "Barnard's Star", x: -0.1, y: 5.9, z: 0.2, mag: 1.2, color: "#ff9292" },
            { name: "Wolf 359", x: -3.6, y: 6.8, z: -1.5, mag: 1.0, color: "#ff4d4d" },
            { name: "Lalande 21185", x: 0.5, y: 8.2, z: 1.2, mag: 1.8, color: "#ff8b8b" },
            { name: "Sirius A", x: -1.6, y: -6.1, z: 5.8, mag: 3.5, color: "#a5d8ff" },
            { name: "Luyten 726-8", x: -1.5, y: -0.5, z: 8.6, mag: 1.1, color: "#ff6b6b" },
            { name: "Ross 154", x: -0.5, y: -2.3, z: 9.4, mag: 1.3, color: "#ff4d4d" },
            { name: "Ross 248", x: 0.8, y: -1.5, z: 10.2, mag: 1.0, color: "#ff4d4d" },
            { name: "Epsilon Eridani", x: -6.7, y: -3.8, z: 7.5, mag: 2.2, color: "#ffcc00" },
            { name: "61 Cygni A", x: 10.4, y: 3.2, z: 4.1, mag: 2.0, color: "#ff9f43" },
            { name: "Procyon A", x: -3.2, y: 7.5, z: 8.4, mag: 3.0, color: "#f1f2f6" },
            { name: "Tau Ceti", x: -2.5, y: -4.1, z: 10.8, mag: 2.1, color: "#fff" }
        ];

        // Gerar mais 80 estrelas aleatórias para completar o catálogo visualmente
        for(let i=0; i<86; i++) {
            glieseData.push({
                name: `Gliese ${Math.floor(Math.random()*999)}`,
                x: (Math.random() - 0.5) * 40,
                y: (Math.random() - 0.5) * 40,
                z: (Math.random() - 0.5) * 40,
                mag: Math.random() * 2,
                color: ["#a5d8ff", "#fff", "#ffd93d", "#ff6b6b"][Math.floor(Math.random()*4)]
            });
        }

        // Configuração da Câmera
        let camera = { x: 0, y: 0, z: -20, targetX: 0, targetY: 0, targetZ: -20, fov: 600 };
        let isTraveling = false;

        function init() {
            window.addEventListener('resize', resize);
            resize();
            
            // Popular Select
            glieseData.forEach((star, index) => {
                const opt = document.createElement('option');
                opt.value = index;
                opt.textContent = star.name;
                starSelect.appendChild(opt);
            });

            animate();
        }

        function resize() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }

        function engageFTL() {
            const star = glieseData[starSelect.value];
            camera.targetX = -star.x;
            camera.targetY = -star.y;
            camera.targetZ = -star.z - 5; // Para parar um pouco antes da estrela
            isTraveling = true;
            
            starDisplay.innerHTML = `<strong>Alvo:</strong> ${star.name}<br><strong>Coordenadas:</strong> [${star.x.toFixed(2)}, ${star.y.toFixed(2)}, ${star.z.toFixed(2)}]`;
        }

        function drawStar(x, y, size, color, name) {
            const screenX = canvas.width / 2 + x;
            const screenY = canvas.height / 2 + y;

            // Bloom Effect - Camadas de Canvas
            ctx.shadowBlur = size * 5;
            ctx.shadowColor = color;
            
            // Brilho Externo (Glow)
            ctx.fillStyle = color;
            ctx.beginPath();
            ctx.arc(screenX, screenY, size * 2, 0, Math.PI * 2);
            ctx.globalAlpha = 0.2;
            ctx.fill();

            // Núcleo da Estrela
            ctx.globalAlpha = 1.0;
            ctx.beginPath();
            ctx.arc(screenX, screenY, size, 0, Math.PI * 2);
            ctx.fill();
            
            // Label
            if (size > 1.5) {
                ctx.shadowBlur = 0;
                ctx.fillStyle = "rgba(255,255,255,0.7)";
                ctx.font = "10px monospace";
                ctx.fillText(name, screenX + size + 5, screenY + 3);
            }
        }

        function animate() {
            // Limpar com rastro leve (opcional para efeito de velocidade)
            ctx.fillStyle = 'rgba(2, 11, 22, 0.3)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Suavização da Câmera (Interpolação/FTL)
            const lerpFactor = 0.05;
            camera.x += (camera.targetX - camera.x) * lerpFactor;
            camera.y += (camera.targetY - camera.y) * lerpFactor;
            camera.z += (camera.targetZ - camera.z) * lerpFactor;

            camCoordsDisplay.innerText = `Cam: X:${camera.x.toFixed(2)} Y:${camera.y.toFixed(2)} Z:${camera.z.toFixed(2)}`;

            // Ordenar estrelas pela distância Z (Painters Algorithm) para profundidade
            const sortedStars = glieseData.map(s => {
                const relativeZ = s.z + camera.z;
                return { ...s, relativeZ };
            }).sort((a, b) => b.relativeZ - a.relativeZ);

            sortedStars.forEach(star => {
                const x = star.x + camera.x;
                const y = star.y + camera.y;
                const z = star.z + camera.z;

                // Não desenhar se estiver atrás da câmera
                if (z <= 0) return;

                // Projeção 3D -> 2D
                const scale = camera.fov / z;
                const px = x * scale;
                const py = y * scale;
                const pSize = Math.max(0.1, (star.mag * scale) / 500);

                // Só desenha se estiver dentro da tela
                if (px > -canvas.width/2 && px < canvas.width/2 && py > -canvas.height/2 && py < canvas.height/2) {
                    drawStar(px, py, pSize, star.color, star.name);
                }
            });

            requestAnimationFrame(animate);
        }

        init();
    </script>
</body>
</html>
