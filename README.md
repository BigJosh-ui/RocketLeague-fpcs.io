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
            const mesh = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), new THREE.MeshStandardMaterial({ color: color, transparent: true, opacity
