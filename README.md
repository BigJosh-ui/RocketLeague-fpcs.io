<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>NEON STRIKER - GOAL EXPLOSION</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Orbitron', sans-serif; color: white; }
        #gui, #menu { position: absolute; inset: 0; display: flex; flex-direction: column; justify-content: center; align-items: center; pointer-events: none; }
        #menu { background: radial-gradient(circle, rgba(10,10,30,0.9) 0%, rgba(0,0,0,1) 100%); pointer-events: all; z-index: 10; }
        .panel { background: rgba(20, 20, 40, 0.85); padding: 40px; border: 2px solid #00f2fe; border-radius: 20px; text-align: center; min-width: 450px; }
        h1 { font-size: 42px; margin-bottom: 30px; color: #00f2fe; text-shadow: 0 0 15px #00f2fe; }
        button { background: transparent; border: 2px solid #00f2fe; color: #00f2fe; padding: 12px 30px; font-family: 'Orbitron'; cursor: pointer; transition: 0.3s; font-weight: bold; }
        button:hover { background: #00f2fe; color: #000; box-shadow: 0 0 25px #00f2fe; }
        
        /* Flash Overlay */
        #goal-flash { position: absolute; inset: 0; background: white; opacity: 0; pointer-events: none; z-index: 5; transition: opacity 0.1s; }
        #goal-text { position: absolute; top: 40%; left: 50%; transform: translate(-50%, -50%); font-size: 80px; font-weight: 900; color: #ff00ff; text-shadow: 0 0 30px #ff00ff; display: none; z-index: 6; }

        #hud { position: absolute; inset: 0; pointer-events: none; display: none; }
        #scoreboard { position: absolute; top: 30px; left: 50%; transform: translateX(-50%); font-size: 36px; }
        #boost-container { position: absolute; bottom: 50px; right: 50px; width: 300px; }
        #boost-bar { height: 14px; background: rgba(0,242,254,0.1); border: 2px solid #00f2fe; border-radius: 20px; overflow: hidden; }
        #boost-fill { height: 100%; width: 100%; background: linear-gradient(90deg, #00f2fe, #7000ff); }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;900&display=swap" rel="stylesheet">
</head>
<body>

    <div id="goal-flash"></div>
    <div id="goal-text">GOAL!</div>

    <div id="menu">
        <div class="panel">
            <h1>CAR GARAGE*</h1>
            <button style="font-size: 24px; width: 100%; border-color: #7000ff; color: #7000ff;" onclick="startGame()">ENTER ARENA</button>
        </div>
    </div>

    <div id="hud">
        <div id="scoreboard"><span style="color:#00f2fe">BLUE</span> <span id="s-blue">0</span> | <span id="s-orange">0</span> <span style="color:#ff8c00">ORANGE</span></div>
        <div id="boost-container">
            <div id="boost-bar"><div id="boost-fill"></div></div>
        </div>
    </div>

    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "cannon": "https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/+esm"
            }
        }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import * as CANNON from 'cannon';

        let gameState = 'MENU';
        const keys = {};
        let score = { blue: 0, orange: 0 };
        let boost = 100;
        let ballCam = false;
        let shakeIntensity = 0;

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const world = new CANNON.World();
        world.gravity.set(0, -25, 0);

        // Ground & Ball Setup
        scene.add(new THREE.GridHelper(300, 60, 0x00f2fe, 0x050515));
        const ballBody = new CANNON.Body({ mass: 2, shape: new CANNON.Sphere(2.5) });
        ballBody.position.set(0, 5, 0);
        world.addBody(ballBody);
        const ballMesh = new THREE.Mesh(new THREE.SphereGeometry(2.5, 32, 32), new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0x00f2fe }));
        scene.add(ballMesh);

        // Floor
        const groundBody = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
        groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
        world.addBody(groundBody);

        // Car Setup
        let carBody, carMesh;
        function createCar() {
            carBody = new CANNON.Body({ mass: 35, shape: new CANNON.Box(new CANNON.Vec3(1.6, 1.1, 2.4)) });
            carBody.position.set(0, 2, 30);
            world.addBody(carBody);
            carMesh = new THREE.Mesh(new THREE.BoxGeometry(3.2, 2.2, 4.8), new THREE.MeshStandardMaterial({ color: 0x00eaff, metalness: 0.9 }));
            scene.add(carMesh);
        }
        createCar();

        scene.add(new THREE.DirectionalLight(0xffffff, 2).set(20, 50, 10));
        scene.add(new THREE.AmbientLight(0xffffff, 0.4));

        function triggerGoalExplosion(team) {
            shakeIntensity = 1.5;
            const flash = document.getElementById('goal-flash');
            const text = document.getElementById('goal-text');
            
            flash.style.opacity = '1';
            text.style.display = 'block';
            text.style.color = team === 'blue' ? '#00f2fe' : '#ff8c00';
            
            setTimeout(() => { flash.style.opacity = '0'; }, 150);
            setTimeout(() => { 
                text.style.display = 'none';
                resetMatch();
            }, 2000);
        }

        function resetMatch() {
            ballBody.position.set(0, 10, 0);
            ballBody.velocity.set(0, 0, 0);
            carBody.position.set(0, 2, 30);
            carBody.velocity.set(0, 0, 0);
        }

        window.startGame = () => {
            gameState = 'PLAYING';
            document.getElementById('menu').style.display = 'none';
            document.getElementById('hud').style.display = 'block';
        };

        window.addEventListener('keydown', e => {
            keys[e.code] = true;
            if(e.code === 'KeyK' && Math.abs(carBody.velocity.y) < 0.2) carBody.velocity.y = 14;
        });
        window.addEventListener('keyup', e => keys[e.code] = false);

        function animate() {
            requestAnimationFrame(animate);
            if(gameState === 'PLAYING') {
                world.fixedStep();
                
                // Movement
                const fwd = new THREE.Vector3(0,0,-1).applyQuaternion(carMesh.quaternion);
                if(keys['KeyW']) carBody.velocity.addScaledVector(0.7, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), carBody.velocity);
                if(keys['KeyS']) carBody.velocity.addScaledVector(-0.4, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), carBody.velocity);
                if(keys['KeyA']) carBody.angularVelocity.y = 3.5;
                else if(keys['KeyD']) carBody.angularVelocity.y = -3.5;
                else carBody.angularVelocity.y *= 0.9;

                // Goal Check
                if(ballBody.position.z < -50) { score.blue++; triggerGoalExplosion('blue'); ballBody.position.z = 0; }
                if(ballBody.position.z > 50) { score.orange++; triggerGoalExplosion('orange'); ballBody.position.z = 0; }

                carMesh.position.copy(carBody.position);
                carMesh.quaternion.copy(carBody.quaternion);
                ballMesh.position.copy(ballBody.position);

                // Score Update
                document.getElementById('s-blue').innerText = score.blue;
                document.getElementById('s-orange').innerText = score.orange;

                // Camera + Shake
                const camOff = new THREE.Vector3(0, 6, 14).applyQuaternion(carMesh.quaternion);
                camera.position.lerp(carMesh.position.clone().add(camOff), 0.1);
                
                if (shakeIntensity > 0) {
                    camera.position.x += (Math.random() - 0.5) * shakeIntensity;
                    camera.position.y += (Math.random() - 0.5) * shakeIntensity;
                    shakeIntensity *= 0.9; // Fade out shake
                }
                camera.lookAt(carMesh.position);
            }
            renderer.render(scene, camera);
        }
        animate();
    </script>
</body>
</html>
