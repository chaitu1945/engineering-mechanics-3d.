<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Mechanics Visualization - Beam Loading</title>
    <style>
        :root {
            --primary: #c80000;      /* Red for vectors */
            --secondary: #00cccc;    /* Cyan for UI highlights */
            --bg-dark: #121212;
            --panel-bg: rgba(30, 30, 30, 0.85);
            --text-light: #ffffff;
        }

        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }

        body {
            background-color: var(--bg-dark);
            color: var(--text-light);
            overflow: hidden;
            height: 100vh;
        }

        #canvas-container {
            width: 100%;
            height: 100%;
            position: absolute;
            top: 0;
            left: 0;
            z-index: 1;
        }

        .ui-panel {
            position: absolute;
            top: 20px;
            left: 20px;
            z-index: 10;
            background: var(--panel-bg);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.1);
            padding: 20px;
            border-radius: 8px;
            width: 320px;
            box-shadow: 0 4px 20px rgba(0,0,0,0.5);
        }

        h1 {
            font-size: 1.25rem;
            margin-bottom: 5px;
            text-transform: uppercase;
            letter-spacing: 1px;
        }

        .subtitle {
            font-size: 0.85rem;
            color: #aaa;
            margin-bottom: 20px;
        }

        .control-group {
            margin-bottom: 15px;
        }

        label {
            display: block;
            font-size: 0.85rem;
            margin-bottom: 5px;
            color: #ddd;
        }

        label span.highlight {
            color: var(--secondary);
            font-weight: bold;
        }

        input[type="range"] {
            width: 100%;
            accent-color: var(--secondary);
            cursor: pointer;
        }

        .readout {
            display: flex;
            justify-content: space-between;
            font-size: 0.8rem;
            margin-top: 4px;
            color: #bbb;
        }

        .value-display {
            color: var(--primary);
            font-weight: bold;
        }

        .legend {
            margin-top: 20px;
            padding-top: 15px;
            border-top: 1px solid rgba(255,255,255,0.1);
            font-size: 0.8rem;
        }

        .legend-item {
            display: flex;
            align-items: center;
            margin-bottom: 5px;
        }

        .color-box {
            width: 12px;
            height: 12px;
            margin-right: 8px;
            border-radius: 2px;
        }

        #nav-hint {
            position: absolute;
            bottom: 20px;
            right: 20px;
            z-index: 10;
            background: rgba(0,0,0,0.6);
            padding: 8px 15px;
            border-radius: 20px;
            font-size: 0.8rem;
            pointer-events: none;
            border: 1px solid rgba(255,255,255,0.05);
        }
    </style>
    <!-- Three.js and OrbitControls via CDN -->
    <script src="https://cloudflare.com"></script>
    <script src="https://jsdelivr.net"></script>
</head>
<body>

    <div id="canvas-container"></div>

    <div class="ui-panel">
        <h1>3D Beam Mechanics</h1>
        <div class="subtitle">First Year Engineering Simulator</div>
        
        <div class="control-group">
            <label>Load Position (X-Axis): <span class="highlight">Distance</span></label>
            <input type="range" id="load-pos" min="-4" max="4" step="0.1" value="0">
            <div class="readout">
                <span>Left Support</span>
                <span class="value-display" id="pos-val">4.5 m</span>
                <span>Right Support</span>
            </div>
        </div>

        <div class="control-group">
            <label>Load Magnitude: <span class="highlight">Force Vector</span></label>
            <input type="range" id="load-mag" min="10" max="100" step="5" value="50">
            <div class="readout">
                <span>Minimum</span>
                <span class="value-display" id="mag-val">50 kN</span>
                <span>Maximum</span>
            </div>
        </div>

        <div class="legend">
            <div class="legend-item">
                <div class="color-box" style="background: var(--primary);"></div>
                <span>Point Load Vector (F)</span>
            </div>
            <div class="legend-item">
                <div class="color-box" style="background: #00ff00;"></div>
                <span>Reaction Forces (R_A, R_B)</span>
            </div>
            <div class="legend-item">
                <div class="color-box" style="background: #555555;"></div>
                <span>Structural Member Beam</span>
            </div>
        </div>
    </div>

    <div id="nav-hint">🖱️ Left-click + drag to rotate | Right-click + drag to pan | Scroll to zoom</div>

    <script>
        // --- Scene Setup ---
        const container = document.getElementById('canvas-container');
        const scene = new THREE.Scene();
        scene.background = new THREE.Color(0x121212);

        const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.set(0, 4, 10);

        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        container.appendChild(renderer.domElement);

        const controls = new THREE.OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;
        controls.maxPolarAngle = Math.PI / 2 + 0.1;

        // --- Lights ---
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
        scene.add(ambientLight);

        const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
        dirLight.position.set(5, 10, 7);
        scene.add(dirLight);

        // --- Floor Grid ---
        const gridHelper = new THREE.GridHelper(20, 20, 0x444444, 0x222222);
        gridHelper.position.y = -2;
        scene.add(gridHelper);

        // --- Beam & Supports Structure ---
        const beamLength = 10;
        const beamGeom = new THREE.BoxGeometry(beamLength, 0.3, 0.3);
        const beamMat = new THREE.MeshStandardMaterial({ color: 0x555555, roughness: 0.4 });
        const beam = new THREE.Mesh(beamGeom, beamMat);
        scene.add(beam);

        const supportGeom = new THREE.ConeGeometry(0.25, 0.4, 4);
        const supportMat = new THREE.MeshStandardMaterial({ color: 0x777777, flatShading: true });
        
        const supportA = new THREE.Mesh(supportGeom, supportMat);
        supportA.position.set(-4.5, -0.35, 0);
        scene.add(supportA);

        const supportB = new THREE.Mesh(supportGeom, supportMat);
        supportB.position.set(4.5, -0.35, 0);
        scene.add(supportB);

        // --- Vectors (Arrows) ---
        const loadArrow = new THREE.ArrowHelper(
            new THREE.Vector3(0, -1, 0),
            new THREE.Vector3(0, 2, 0),
            1,
            0xc80000, 
            0.3,
            0.2
        );
        scene.add(loadArrow);

        const reactionArrowA = new THREE.ArrowHelper(
            new THREE.Vector3(0, 1, 0),
            new THREE.Vector3(-4.5, -2, 0),
            1,
            0x00ff00,
            0.2,
            0.15
        );
        const reactionArrowB = new THREE.ArrowHelper(
            new THREE.Vector3(0, 1, 0),
            new THREE.Vector3(4.5, -2, 0),
            1,
            0x00ff00,
            0.2,
            0.15
        );
        scene.add(reactionArrowA);
        scene.add(reactionArrowB);

        // --- Calculations & Updates ---
        const posSlider = document.getElementById('load-pos');
        const magSlider = document.getElementById('load-mag');
        const posVal = document.getElementById('pos-val');
        const magVal = document.getElementById('mag-val');

        function updateMechanics() {
            const x = parseFloat(posSlider.value);
            const F = parseFloat(magSlider.value);

            posVal.innerText = `${(x + 4.5).toFixed(1)} m`;
            magVal.innerText = `${F} kN`;

            const span = 9.0;
            const distA = x - (-4.5);

            // Equilibrium equations calculation
            const RB = (F * distA) / span;
            const RA = F - RB;

            const scale = 0.025; 
            
            // Adjust Load Vector geometry position
            loadArrow.position.set(x, 0.15 + (F * scale), 0);
            loadArrow.setLength(F * scale, 0.4, 0.2);

            // Adjust Support Vector heights
            reactionArrowA.setLength(Math.max(0.1, RA * scale), 0.3, 0.15);
            reactionArrowB.setLength(Math.max(0.1, RB * scale), 0.3, 0.15);
        }

        posSlider.addEventListener('input', updateMechanics);
        magSlider.addEventListener('input', updateMechanics);

        updateMechanics();

        // --- Render loop ---
        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
        }
        animate();

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
