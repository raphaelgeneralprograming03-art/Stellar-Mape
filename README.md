
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Stellar Map - Gliese Catalog FTL</title>
    <style>
        :root {
            --primary: #00d4ff;
            --bg: #010810;
            --panel: rgba(10, 25, 41, 0.85);
        }

        body, html {
            margin: 0; padding: 0;
            width: 100%; height: 100%;
            background-color: var(--bg);
            color: white;
            font-family: 'Segoe UI', Roboto, sans-serif;
            overflow: hidden;
        }

        #canvas { display: block; }

        #ui-layer {
            position: absolute;
            top: 20px; left: 20px;
            width: 280px;
            background: var(--panel);
            backdrop-filter: blur(15px);
            border: 1px solid rgba(0, 212, 255, 0.3);
            border-radius: 12px;
            padding: 20px;
            box-shadow: 0 0 30px rgba(0,0,0,0.8);
            user-select: none;
        }

        h1 { 
            font-size: 16px; margin: 0 0 15px 0; 
            color: var(--primary); 
            text-transform: uppercase; letter-spacing: 2px;
            border-bottom: 1px solid rgba(255,255,255,0.1);
            padding-bottom: 10px;
        }
        
        .star-info {
            font-size: 13px; margin-bottom: 15px;
            line-height: 1.6; color: #e0e0e0;
        }

        .label { color: var(--primary); font-weight: bold; font-size: 10px; text-transform: uppercase; }

        select {
            width: 100%; background: #051221; color: white;
            border: 1px solid var(--primary); padding: 8px;
            border-radius: 4px; margin-bottom: 15px; outline: none;
        }

        button {
            width: 100%; background: var(--primary); color: #000;
            border: none; padding: 12px; border-radius: 4px;
            font-weight: bold; cursor: pointer; transition: 0.3s;
            text-transform: uppercase; letter-spacing: 1px;
        }

        button:hover { background: #fff; box-shadow: 0 0 20px var(--primary); }

        .coords {
            font-family: 'Courier New', monospace;
            font-size: 11px; color: #50a0b0;
            margin-top: 15px; padding-top: 10px;
            border-top: 1px solid rgba(255,255,255,0.1);
        }
    </style>
</head>
<body>

    <div id="ui-layer">
        <h1>Stellar Mapper</h1>
        <div class="star-info" id="star-display">
            <span class="label">Status:</span> Estacionário<br>
            <span class="label">Alvo:</span> Sol (Origem)<br>
            <span class="label">Distância:</span> 0.00 LY
        </div>
        
        <label class="label">Destino (Catálogo Gliese):</label>
        <select id="star-select"></select>
        
        <button onclick="engageFTL()">INICIAR SALTO FTL</button>
        
        <div class="coords" id="cam-coords">POS: 0.00, 0.00, 0.00</div>
    </div>

    <canvas id="canvas"></canvas>

    <script>
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        const starSelect = document.getElementById('star-select');
        const starDisplay = document.getElementById('star-display');
        const camCoordsDisplay = document.getElementById('cam-coords');

        // Dados do Catálogo Gliese (Amostra de Estrelas Próximas)
        const glieseData = [
            { name: "Sol", x: 0, y: 0, z: 0, mag: 4, color: "#fffeb0" },
            { name: "Proxima Centauri", x: -1.2, y: 0.5, z: 4.2, mag: 2, color: "#ff6b6b" },
            { name: "Alpha Centauri A", x: -1.3, y: 0.6, z: 4.3, mag: 5, color: "#ffd93d" },
            { name: "Barnard's Star", x: -0.1, y: 1.9, z: 5.9, mag: 2, color: "#ff9292" },
            { name: "Wolf 359", x: -3.6, y: 2.8, z: 7.8, mag: 1.5, color: "#ff4d4d" },
            { name: "Sirius A", x: -1.6, y: -6.1, z: 8.6, mag: 8, color: "#a5d8ff" },
            { name: "Epsilon Eridani", x: -6.7, y: -3.8, z: 10.5, mag: 4, color: "#ffcc00" },
            { name: "Procyon A", x: -3.2, y: 7.5, z: 11.4, mag: 6, color: "#f1f2f6" },
            { name: "61 Cygni A", x: 10.4, y: 3.2, z: 11.4, mag: 3, color: "#ff9f43" },
            { name: "Tau Ceti", x: -2.5, y: -4.1, z: 11.9, mag: 4, color: "#ffffff" }
        ];

        // Gerar estrelas de fundo aleatórias
        for(let i=0; i<200; i++) {
            glieseData.push({
                name: `Gliese ${Math.floor(Math.random()*999 + 100)}`,
                x: (Math.random() - 0.5) * 100,
                y: (Math.random() - 0.5) * 100,
                z: (Math.random() - 0.5) * 100,
                mag: Math.random() * 3 + 1,
                color: ["#a5d8ff", "#ffffff", "#ffd93d", "#ff6b6b"][Math.floor(Math.random()*4)]
            });
        }

        // Configuração da Câmera
        let camera = { x: 0, y: 0, z: -15, targetX: 0, targetY: 0, targetZ: -15, fov: 400 };
        let currentTarget = glieseData[0];

        function init() {
            window.addEventListener('resize', resize);
            resize();
            
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
            currentTarget = glieseData[starSelect.value];
            // Posiciona a câmera um pouco recuada do alvo no eixo Z para visualização
            camera.targetX = -currentTarget.x;
            camera.targetY = -currentTarget.y;
            camera.targetZ = -currentTarget.z + 10; 
        }

        function drawStar(x, y, size, color, name, distance) {
            const screenX = canvas.width / 2 + x;
            const screenY = canvas.height / 2 + y;

            if (screenX < -100 || screenX > canvas.width + 100 || screenY < -100 || screenY > canvas.height + 100) return;

            ctx.globalCompositeOperation = 'lighter';
            
            // Brilho (Glow)
            const gradient = ctx.createRadialGradient(screenX, screenY, 0, screenX, screenY, size * 4);
            gradient.addColorStop(0, color);
            gradient.addColorStop(1, 'transparent');
            
            ctx.fillStyle = gradient;
            ctx.beginPath();
            ctx.arc(screenX, screenY, size * 4, 0, Math.PI * 2);
            ctx.fill();

            // Núcleo
            ctx.fillStyle = "#fff";
            ctx.beginPath();
            ctx.arc(screenX, screenY, size * 0.8, 0, Math.PI * 2);
            ctx.fill();
            
            // Label para estrelas próximas
            if (distance < 20) {
                ctx.globalCompositeOperation = 'source-over';
                ctx.fillStyle = "rgba(255,255,255,0.5)";
                ctx.font = "10px monospace";
                ctx.fillText(name, screenX + size + 5, screenY + 3);
            }
        }

        function animate() {
            // Fundo escuro com leve rastro (Motion Blur)
            ctx.globalCompositeOperation = 'source-over';
            ctx.fillStyle = 'rgba(1, 8, 16, 0.2)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Suavização da Câmera (Lerp)
            camera.x += (camera.targetX - camera.x) * 0.05;
            camera.y += (camera.targetY - camera.y) * 0.05;
            camera.z += (camera.targetZ - camera.z) * 0.05;

            // Atualizar UI
            const distToTarget = Math.sqrt(
                Math.pow(currentTarget.x + camera.x, 2) + 
                Math.pow(currentTarget.y + camera.y, 2) + 
                Math.pow(currentTarget.z + camera.z, 2)
            );

            starDisplay.innerHTML = `
                <span class="label">Status:</span> ${distToTarget > 0.1 ? 'Em Trânsito FTL' : 'Estacionário'}<br>
                <span class="label">Alvo:</span> ${currentTarget.name}<br>
                <span class="label">Distância:</span> ${distToTarget.toFixed(2)} LY
            `;
            camCoordsDisplay.innerText = `POS: X:${(-camera.x).toFixed(2)} Y:${(-camera.y).toFixed(2)} Z:${(-camera.z).toFixed(2)}`;

            // Ordenar por profundidade (Z)
            const renderedStars = glieseData.map(s => {
                return {
                    ...s,
                    relX: s.x + camera.x,
                    relY: s.y + camera.y,
                    relZ: s.z + camera.z
                };
            })
            .filter(s => s.relZ > 0.5) // Clip stars behind camera
            .sort((a, b) => b.relZ - a.relZ);

            renderedStars.forEach(star => {
                const scale = camera.fov / star.relZ;
                const px = star.relX * scale;
                const py = star.relY * scale;
                const pSize = (star.mag * (scale / 100));

                drawStar(px, py, Math.max(0.5, pSize), star.color, star.name, star.relZ);
            });

            requestAnimationFrame(animate);
        }

        init();
    </script>
</body>
</html>
