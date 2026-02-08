<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>NEON STRIKER - STADIUM EDITION</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Orbitron', sans-serif; color: white; }
        #menu, #hud { position: absolute; inset: 0; display: flex; flex-direction: column; justify-content: center; align-items: center; pointer-events: none; }
        #menu { background: radial-gradient(circle, rgba(10,10,30,0.9) 0%, rgba(0,0,0,1) 100%); pointer-events: all; z-index: 10; }
        .panel { background: rgba(20, 20, 40, 0.95); padding: 40px; border: 3px solid #00f2fe; border-radius: 20px; text-align: center; min-width: 400px; box-shadow: 0 0 50px rgba(0, 242, 254, 0.4); }
        h1 { font-size: 3rem; color: #00f2fe; text-shadow: 0 0 20px #00f2fe; margin: 0 0 20px 0; }
        button { background: transparent; border: 2px solid #00f2fe; color: #00f2fe; padding: 15px 30px; font-family: 'Orbitron'; cursor: pointer; border-radius: 8px; margin: 10px; pointer-events: all; transition: 0.3s; font-weight: bold; }
        button:hover { background: #00f2fe; color: #000; box-shadow: 0 0 20px #00f2fe; }
        #hud { display: none; }
        #scoreboard { position: absolute; top: 20px; font-size: 48px; font-weight: 900; background: rgba(0,0,0,0.6); padding: 10px 40px; border-radius: 15px; border: 2px solid #333; }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;900&display=swap" rel="stylesheet">
</head>
<body>
    <div id="menu">
        <div class="panel">
            <h1>NEON STRIKER</h1>
            <button onclick="startGame(0.04)">EASY</button>
            <button onclick="startGame(0.09)">MEDIUM</button>
            <button onclick="startGame(0.18)">PRO</button>
            <p style="font-size: 11px; color: #777; margin-top: 15px;">STADIUM LOADED | 4X STEERING ACTIVE</p>
        </div>
    </div>
    <div id="hud"><div id="scoreboard"><span id="s-blue" style="color:#00f2fe">0</span> - <span id="s-orange" style="color:#ff8c00">0</span></div></div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js", "cannon": "https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/+esm" } }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import * as CANNON from 'cannon';

        let gameState = 'MENU', botDifficulty = 0.08, score = { blue: 0, orange: 0 }, jumpCount = 0;
        const keys = {};

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const world = new CANNON.World();
        world.gravity.set(0, -22, 0);

        // --- ARENA CONSTRUCTION ---
        const groundGeo = new THREE.PlaneGeometry(100, 160);
        const groundMat = new THREE.MeshStandardMaterial({ color: 0x050510 });
        const ground = new THREE.Mesh(groundGeo, groundMat);
        ground.rotation.x = -Math.PI/2;
        scene.add(ground);
        scene.add(new THREE.GridHelper(160, 40, 0x00f2fe, 0x111122));

        const groundBody = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
        groundBody.quaternion.setFromEuler(-Math.PI/2, 0, 0);
        world.addBody(groundBody);

        // Stadium Walls
        function createWall(w, h, d, x, z, color) {
            const mesh = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), new THREE.MeshStandardMaterial({ color: color, transparent: true, opacity: 0.2 }));
            mesh.position.set(x, h/2, z);
            scene.add(mesh);
            const body = new CANNON.Body({ mass: 0, shape: new CANNON.Box(new CANNON.Vec3(w/2, h/2, d/2)) });
            body.position.set(x, h/2, z);
            world.addBody(body);
        }
        createWall(100, 20, 2, 0, 80, 0xff8c00);  // Orange End
        createWall(100, 20, 2, 0, -80, 0x00f2fe); // Blue End
        createWall(2, 20, 160, 50, 0, 0x333333);  // Left
        createWall(2, 20, 160, -50, 0, 0x333333); // Right

        // Goals
        function createGoal(z, color) {
            const frame = new THREE.Mesh(new THREE.BoxGeometry(20, 10, 2), new THREE.MeshStandardMaterial({ color: color, emissive: color }));
            frame.position.set(0, 5, z);
            scene.add(frame);
        }
        createGoal(79, 0xff8c00);
        createGoal(-79, 0x00f2fe);

        // --- BALL ---
        const ballBody = new CANNON.Body({ mass: 2, shape: new CANNON.Sphere(2.5) });
        ballBody.position.set(0, 5, 0);
        ballBody.linearDamping = 0.1;
        world.addBody(ballBody);
        const ballMesh = new THREE.Mesh(new THREE.SphereGeometry(2.5, 32), new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0x333333 }));
        scene.add(ballMesh);

        // --- FENNEC BUILDER ---
        function createFennec(color) {
            const group = new THREE.Group();
            const mat = new THREE.MeshStandardMaterial({ color: color });
            const wheelMat = new THREE.MeshStandardMaterial({ color: 0x111111 });
            const body = new THREE.Mesh(new THREE.BoxGeometry(3.2, 1.2, 4.8), mat);
            body.position.y = 0.6;
            group.add(body);
            const cabin = new THREE.Mesh(new THREE.BoxGeometry(2.8, 1, 2.5), mat);
            cabin.position.set(0, 1.6, -0.5);
            group.add(cabin);
            const wheelGeo = new THREE.CylinderGeometry(0.6, 0.6, 0.4, 16);
            wheelGeo.rotateZ(Math.PI / 2);
            const wheels = [[1.8, 0.4, 1.6], [-1.8, 0.4, 1.6], [1.8, 0.4, -1.6], [-1.8, 0.4, -1.6]].map(pos => {
                const w = new THREE.Mesh(wheelGeo, wheelMat);
                w.position.set(...pos);
                group.add(w);
                return w;
            });
            scene.add(group);
            return { group, wheels };
        }

        const player = createFennec(0x00eaff);
        const playerBody = new CANNON.Body({ mass: 35, shape: new CANNON.Box(new CANNON.Vec3(1.6, 1.1, 2.4)) });
        playerBody.position.set(0, 2, 50);
        playerBody.angularDamping = 0.99;
        world.addBody(playerBody);

        const bot = createFennec(0xff8c00);
        const botBody = new CANNON.Body({ mass: 35, shape: new CANNON.Box(new CANNON.Vec3(1.6, 1.1, 2.4)) });
        botBody.position.set(0, 2, -50);
        botBody.angularDamping = 0.99;
        world.addBody(botBody);

        scene.add(new THREE.AmbientLight(0xffffff, 0.7));
        const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
        dirLight.position.set(0, 50, 0);
        scene.add(dirLight);

        window.startGame = (diff) => {
            botDifficulty = diff;
            gameState = 'PLAYING';
            document.getElementById('menu').style.display = 'none';
            document.getElementById('hud').style.display = 'block';
        };

        window.addEventListener('keydown', e => { 
            keys[e.code] = true;
            if(e.code === 'Space') {
                if(playerBody.position.y < 1.3) { playerBody.velocity.y = 13; jumpCount = 1; }
                else if(jumpCount === 1) {
                    jumpCount = 0;
                    if(keys['KeyW']) { playerBody.velocity.z -= 25; playerBody.angularVelocity.x = -10; }
                    else { playerBody.velocity.y += 10; }
                }
            }
            if(e.code === 'KeyR') reset();
        });
        window.addEventListener('keyup', e => keys[e.code] = false);

        function reset() {
            ballBody.position.set(0, 10, 0); ballBody.velocity.set(0,0,0); ballBody.angularVelocity.set(0,0,0);
            playerBody.position.set(0, 2, 50); playerBody.velocity.set(0,0,0); playerBody.quaternion.set(0,0,0,1);
            botBody.position.set(0, 2, -50); botBody.velocity.set(0,0,0); botBody.quaternion.set(0,0,0,1);
        }

        function update() {
            requestAnimationFrame(update);
            if(gameState === 'PLAYING') {
                world.fixedStep();
                const fwd = new THREE.Vector3(0,0,-1).applyQuaternion(player.group.quaternion);
                const airborne = playerBody.position.y > 1.5;

                if(keys['KeyW']) {
                    if(!airborne) {
                        playerBody.velocity.addScaledVector(1.6, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), playerBody.velocity);
                        player.wheels.forEach(w => w.rotation.x += 0.4);
                    } else playerBody.angularVelocity.x = -6;
                }
                if(keys['KeyS']) { if(!airborne) playerBody.velocity.addScaledVector(-1, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), playerBody.velocity); }
                if(keys['KeyA']) { if(!airborne) playerBody.angularVelocity.y = 26; else playerBody.angularVelocity.z = -8; }
                if(keys['KeyD']) { if(!airborne) playerBody.angularVelocity.y = -26; else playerBody.angularVelocity.z = 8; }

                // Bot AI
                const b2b = ballBody.position.vsub(botBody.position);
                botBody.quaternion.setFromEuler(0, Math.atan2(b2b.x, b2b.z) + Math.PI, 0);
                const bf = new THREE.Vector3(0,0,-1).applyQuaternion(bot.group.quaternion);
                botBody.velocity.addScaledVector(botDifficulty * 28, new CANNON.Vec3(bf.x, bf.y, bf.z), botBody.velocity);

                // Score Check
                if(ballBody.position.z > 78 && Math.abs(ballBody.position.x) < 10) { score.blue++; reset(); }
                if(ballBody.position.z < -78 && Math.abs(ballBody.position.x) < 10) { score.orange++; reset(); }
                document.getElementById('s-blue').innerText = score.blue;
                document.getElementById('s-orange').innerText = score.orange;

                player.group.position.copy(playerBody.position); player.group.quaternion.copy(playerBody.quaternion);
                bot.group.position.copy(botBody.position); bot.group.quaternion.copy(botBody.quaternion);
                ballMesh.position.copy(ballBody.position);
                camera.position.lerp(player.group.position.clone().add(new THREE.Vector3(0,9,20).applyQuaternion(player.group.quaternion)), 0.15);
                camera.lookAt(ballMesh.position);
            }
            renderer.render(scene, camera);
        }
        update();
    </script>
</body>
</html>
