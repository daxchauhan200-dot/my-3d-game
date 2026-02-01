<!DOCTYPE html>
<html>
<head>
    <title>Jet Pilot Ultra - 1000 Levels</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Orbitron', sans-serif; }
        #ui {
            position: absolute; top: 20px; left: 20px; color: #00ffff;
            background: rgba(0,0,0,0.8); padding: 15px; border: 2px solid #00ffff;
            border-radius: 10px; pointer-events: none;
        }
        #cam-info {
            position: absolute; bottom: 20px; right: 20px; color: #fff;
            background: rgba(0,255,255,0.2); padding: 10px; font-size: 12px;
        }
        #msg {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            color: white; text-align: center; display: none; background: rgba(0,0,0,0.95);
            padding: 50px; border: 4px solid #ff0055; border-radius: 20px; z-index: 100;
        }
        button { padding: 15px 30px; background: #00ffff; border: none; cursor: pointer; font-weight: bold; margin-top: 20px; }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron&display=swap" rel="stylesheet">
</head>
<body>

    <div id="ui">
        LEVEL: <span id="level">1</span> | SCORE: <span id="score">0</span><br>
        SHIP CLASS: <span id="ship-name">SCOUT</span><br>
        <span style="color:#ff0055">DANGER: WALLS & RINGS ARE DEADLY!</span>
    </div>

    <div id="cam-info">Press 1-0 to Change Camera View</div>

    <div id="msg">
        <h1 id="death-reason" style="color: #ff0055;">CRASHED!</h1>
        <p>Final Score: <span id="final-score">0</span></p>
        <button onclick="location.reload()">RE-DEPLOY PILOT</button>
    </div>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

    <script>
        let scene, camera, renderer, tunnel, jetGroup, obstacles = [];
        let score = 0, level = 1, baseSpeed = 0.8, currentSpeed = 0.8;
        let isGameOver = false, targetPos = { x: 0, y: 0 };
        const TUNNEL_RADIUS = 8;

        function init() {
            scene = new THREE.Scene();
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderer.domElement);

            jetGroup = new THREE.Group();
            updateShipLayout(); // લેવલ મુજબ પ્લેન બનાવશે
            scene.add(jetGroup);

            // Lighting
            const light = new THREE.PointLight(0xffffff, 1.5, 100);
            light.position.set(0, 10, 10);
            scene.add(light);
            scene.add(new THREE.AmbientLight(0x404040));

            // Tunnel
            tunnel = new THREE.Mesh(
                new THREE.CylinderGeometry(TUNNEL_RADIUS, TUNNEL_RADIUS, 2000, 32, 1, true),
                new THREE.MeshStandardMaterial({ color: 0x444444, wireframe: true, side: THREE.BackSide })
            );
            tunnel.rotation.x = Math.PI / 2;
            scene.add(tunnel);

            changeCamera(1);
            spawnObstacle();
            animate();
        }

        // --- LEVEL-BASED SHIP LAYOUTS ---
        function updateShipLayout() {
            while(jetGroup.children.length > 0) jetGroup.remove(jetGroup.children[0]);
            
            const mat = new THREE.MeshStandardMaterial({color: 0x00ffff, metalness: 0.8});
            const shipInfo = document.getElementById('ship-name');

            if (level === 1) { // Scout Plane
                const b = new THREE.Mesh(new THREE.BoxGeometry(0.4, 0.2, 1.5), mat);
                const w = new THREE.Mesh(new THREE.BoxGeometry(1.5, 0.05, 0.5), mat);
                jetGroup.add(b, w);
                shipInfo.innerText = "SCOUT CLASS";
            } else if (level === 2) { // Fighter Jet
                const b = new THREE.Mesh(new THREE.ConeGeometry(0.3, 2, 4), mat);
                b.rotation.x = Math.PI/2;
                const w = new THREE.Mesh(new THREE.BoxGeometry(2.5, 0.05, 0.8), mat);
                jetGroup.add(b, w);
                shipInfo.innerText = "FIGHTER CLASS";
            } else { // Heavy Destroyer
                const b = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.5, 2), mat);
                const w1 = new THREE.Mesh(new THREE.BoxGeometry(3, 0.1, 1), mat);
                const w2 = new THREE.Mesh(new THREE.BoxGeometry(0.1, 1.5, 0.5), mat);
                jetGroup.add(b, w1, w2);
                shipInfo.innerText = "DESTROYER CLASS";
            }
        }

        // --- 10 CAMERA SYSTEM ---
        function changeCamera(num) {
            if (isGameOver) return;
            switch(num) {
                case 1: camera.position.set(0, 2, 7); break; // Behind
                case 2: camera.position.set(0, 0.2, 0.2); break; // Cockpit
                case 3: camera.position.set(0, 10, 0); break; // Top
                case 4: camera.position.set(-8, 0, 0); break; // Left Side
                case 5: camera.position.set(8, 0, 0); break; // Right Side
                case 6: camera.position.set(0, -5, 10); break; // Low Angle
                case 7: camera.position.set(0, 1.5, 15); break; // Far
                case 8: camera.position.set(5, 5, 5); break; // Isometric
                case 9: camera.position.set(0, 0.5, -3); break; // Front Look
                case 0: camera.position.set(Math.random()*10, 5, 5); break; // Chaos
            }
            camera.lookAt(jetGroup.position);
        }

        window.addEventListener('keydown', (e) => {
            if (e.key >= '0' && e.key <= '9') changeCamera(parseInt(e.key === '0' ? 0 : e.key));
        });

        window.addEventListener('mousemove', (e) => {
            targetPos.x = (e.clientX / window.innerWidth - 0.5) * 18;
            targetPos.y = -(e.clientY / window.innerHeight - 0.5) * 18;
        });

        function spawnObstacle() {
            if (isGameOver) return;
            const obs = new THREE.Mesh(
                new THREE.TorusGeometry(2, 0.3, 12, 24),
                new THREE.MeshStandardMaterial({ color: 0xff0055, emissive: 0x330000 })
            );
            obs.position.set((Math.random()-0.5)*12, (Math.random()-0.5)*12, -150);
            scene.add(obs);
            obstacles.push(obs);
            setTimeout(spawnObstacle, 800 / currentSpeed);
        }

        function animate() {
            if (isGameOver) return;
            requestAnimationFrame(animate);

            jetGroup.position.x += (targetPos.x - jetGroup.position.x) * 0.1;
            jetGroup.position.y += (targetPos.y - jetGroup.position.y) * 0.1;
            jetGroup.rotation.z = -(targetPos.x - jetGroup.position.x) * 0.3;

            // --- WALL COLLISION ---
            let distFromCenter = Math.sqrt(jetGroup.position.x**2 + jetGroup.position.y**2);
            if (distFromCenter > TUNNEL_RADIUS - 0.8) gameOver("WALL CRASH! Stay in the middle!");

            tunnel.rotation.y += 0.005;

            obstacles.forEach((obs, i) => {
                obs.position.z += currentSpeed;

                // --- RING COLLISION ---
                let d = jetGroup.position.distanceTo(obs.position);
                if (d < 2.2 && d > 0.7 && Math.abs(jetGroup.position.z - obs.position.z) < 0.5) {
                    gameOver("RING IMPACT! You hit the obstacle.");
                }

                if (obs.position.z > 10) {
                    scene.remove(obs);
                    obstacles.splice(i, 1);
                    score++;
                    document.getElementById('score').innerText = score;

                    // 100 Rings = Level Up, 25 Rings = 5x Speed Boost
                    if (score % 100 === 0) { 
                        level++; 
                        document.getElementById('level').innerText = level; 
                        updateShipLayout(); 
                    }
                    if (score % 25 === 0) {
                        currentSpeed = baseSpeed * 5;
                        setTimeout(() => currentSpeed = baseSpeed, 2000);
                    }
                }
            });

            camera.lookAt(jetGroup.position);
            renderer.render(scene, camera);
        }

        function gameOver(reason) {
            isGameOver = true;
            document.getElementById('death-reason').innerText = reason;
            document.getElementById('final-score').innerText = score;
            document.getElementById('msg').style.display = 'block';
        }

        init();
    </script>
</body>
</html>
