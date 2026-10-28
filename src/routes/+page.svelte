<script>
  import { onMount } from 'svelte';
  import { fade, fly } from 'svelte/transition';
  import * as THREE from 'three';
  import { RoundedBoxGeometry } from 'three/addons/geometries/RoundedBoxGeometry.js';
  import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js';

  let number = $state(1);
  let rolling = $state(false);
  let started = $state(false);
  let motionOn = $state(true);
  let canvas = $state();

  let beta = 0;
  let gamma = 0;

  // ---------- Dice data ----------
  const UP = new THREE.Vector3(0, 1, 0);

  // Which side of the cube carries which number (opposite sides add up to 7)
  const FACE_NORMALS = {
    1: new THREE.Vector3(0, 1, 0),
    6: new THREE.Vector3(0, -1, 0),
    2: new THREE.Vector3(0, 0, 1),
    5: new THREE.Vector3(0, 0, -1),
    3: new THREE.Vector3(1, 0, 0),
    4: new THREE.Vector3(-1, 0, 0)
  };

  // Pip positions on a 3x3 grid, from -1 to 1
  const PIPS = {
    1: [[0, 0]],
    2: [[-1, -1], [1, 1]],
    3: [[-1, -1], [0, 0], [1, 1]],
    4: [[-1, -1], [1, -1], [-1, 1], [1, 1]],
    5: [[-1, -1], [1, -1], [0, 0], [-1, 1], [1, 1]],
    6: [[-1, -1], [-1, 0], [-1, 1], [1, -1], [1, 0], [1, 1]]
  };

  const ROLL_TIME = 1.0;   // seconds in the air
  const SETTLE_TIME = 0.6; // seconds to tip onto the final side

  // ---------- Animation state (used by the render loop) ----------
  let dice;
  let phase = 'showcase'; // showcase | roll | settle | rest
  let phaseTime = 0;
  let startOffset = 0;
  let ringT = 1;
  const spinAxis = new THREE.Vector3();
  const settleFrom = new THREE.Quaternion();
  const settleTo = new THREE.Quaternion();
  const tmpMatrix = new THREE.Matrix4();

  // Which number is shown for the current phone position?
  function readNumber() {
    const b = (beta * Math.PI) / 180;
    const g = (gamma * Math.PI) / 180;

    const up = Math.cos(b) * Math.cos(g);  // + = face up, - = face down
    const top = Math.sin(b) * Math.cos(g); // + = top of phone up, - = top down
    const side = Math.sin(g);              // + = right side down, - = left side down

    const tolerance = 0.6;

    if (up > tolerance) return 6;
    if (up < -tolerance) return 1;
    if (top > tolerance) return 2;
    if (top < -tolerance) return 3;
    if (side > tolerance) return 4;
    if (side < -tolerance) return 5;

    return number;
  }

  // How high the center must be so the cube never sinks into the floor
  function restHeight(q) {
    const e = tmpMatrix.makeRotationFromQuaternion(q).elements;
    const raw = 0.5 * (Math.abs(e[1]) + Math.abs(e[5]) + Math.abs(e[9]));
    return 0.5 + (raw - 0.5) * 0.85; // rounded corners sit a bit lower
  }

  // Rotation that puts number n on top, turned so it needs the least spin
  function targetRotation(n) {
    const align = new THREE.Quaternion().setFromUnitVectors(FACE_NORMALS[n], UP);
    let best = null;
    let bestAngle = Infinity;
    for (let k = 0; k < 4; k++) {
      const yaw = new THREE.Quaternion().setFromAxisAngle(
        UP,
        Math.PI / 4 + (k * Math.PI) / 2 + (Math.random() - 0.5) * 0.3
      );
      const candidate = yaw.multiply(align);
      const angle = dice.quaternion.angleTo(candidate);
      if (angle < bestAngle) {
        bestAngle = angle;
        best = candidate;
      }
    }
    return best;
  }

  function roll() {
    if (!started || rolling || !dice) return;
    rolling = true;
    phase = 'roll';
    phaseTime = 0;
    startOffset = dice.position.y - restHeight(dice.quaternion);
    spinAxis
      .set(Math.random() - 0.5, Math.random() - 0.5, Math.random() - 0.5)
      .normalize();
  }

  function onMotion(event) {
    const a = event.accelerationIncludingGravity;
    if (!a) return;
    const force = Math.hypot(a.x, a.y, a.z);
    if (Math.abs(force - 9.8) > 12) roll();
  }

  function onOrientation(event) {
    beta = event.beta ?? 0;
    gamma = event.gamma ?? 0;
  }

  async function start() {
    try {
      // iPhone needs permission (must be triggered by a button tap)
      if (
        typeof DeviceMotionEvent !== 'undefined' &&
        typeof DeviceMotionEvent.requestPermission === 'function'
      ) {
        const motion = await DeviceMotionEvent.requestPermission();
        const orientation = await DeviceOrientationEvent.requestPermission();
        motionOn = motion === 'granted' && orientation === 'granted';
      }
    } catch {
      motionOn = false;
    }

    // Without motion access the dice still works by tapping
    if (motionOn) {
      window.addEventListener('devicemotion', onMotion);
      window.addEventListener('deviceorientation', onOrientation);
    }
    started = true;
    roll(); // first drop onto the table
  }

  // Soft round gradient, used for the table and the glow
  function radialTexture() {
    const c = document.createElement('canvas');
    c.width = c.height = 256;
    const ctx = c.getContext('2d');
    const g = ctx.createRadialGradient(128, 128, 0, 128, 128, 128);
    g.addColorStop(0, '#ffffff');
    g.addColorStop(0.35, '#b0b0b0');
    g.addColorStop(0.7, '#303030');
    g.addColorStop(1, '#000000');
    ctx.fillStyle = g;
    ctx.fillRect(0, 0, 256, 256);
    return new THREE.CanvasTexture(c);
  }

  // The two flat directions along a face, used to place the pips
  function faceAxes(n) {
    if (n.y) return [new THREE.Vector3(1, 0, 0), new THREE.Vector3(0, 0, 1)];
    if (n.z) return [new THREE.Vector3(1, 0, 0), new THREE.Vector3(0, 1, 0)];
    return [new THREE.Vector3(0, 0, 1), new THREE.Vector3(0, 1, 0)];
  }

  onMount(() => {
    // ---------- Renderer ----------
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFShadowMap;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.1;

    const scene = new THREE.Scene();
    const pmrem = new THREE.PMREMGenerator(renderer);
    scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;
    scene.environmentIntensity = 0.7;

    const camera = new THREE.PerspectiveCamera(36, 1, 0.1, 100);
    camera.position.set(0, 5.2, 5.6);
    const lookTarget = new THREE.Vector3(0, 0.7, 0);

    // ---------- Lights ----------
    scene.add(new THREE.HemisphereLight(0xc4b5fd, 0x1a1033, 0.7));

    const keyLight = new THREE.DirectionalLight(0xffffff, 2.4);
    keyLight.position.set(3, 7, 4);
    keyLight.castShadow = true;
    keyLight.shadow.mapSize.set(1024, 1024);
    Object.assign(keyLight.shadow.camera, { left: -3, right: 3, top: 3, bottom: -3, near: 1, far: 20 });
    keyLight.shadow.bias = -0.0005;
    keyLight.shadow.radius = 6;
    keyLight.shadow.blurSamples = 16;
    scene.add(keyLight);

    const pinkLight = new THREE.PointLight(0xec4899, 22, 14);
    pinkLight.position.set(-3, 2.5, -2);
    scene.add(pinkLight);

    const purpleLight = new THREE.PointLight(0x8b5cf6, 18, 14);
    purpleLight.position.set(3, 1.5, -3);
    scene.add(purpleLight);

    // ---------- Table ----------
    const fadeTex = radialTexture();

    const table = new THREE.Mesh(
      new THREE.CircleGeometry(6, 96),
      new THREE.MeshStandardMaterial({
        color: 0x2a1b5c,
        roughness: 0.55,
        metalness: 0.15,
        transparent: true,
        alphaMap: fadeTex
      })
    );
    table.rotation.x = -Math.PI / 2;
    table.receiveShadow = true;
    scene.add(table);

    const glow = new THREE.Mesh(
      new THREE.CircleGeometry(1.8, 48),
      new THREE.MeshBasicMaterial({
        color: 0xa855f7,
        map: fadeTex,
        transparent: true,
        opacity: 0.25,
        blending: THREE.AdditiveBlending,
        depthWrite: false
      })
    );
    glow.rotation.x = -Math.PI / 2;
    glow.position.y = 0.004;
    scene.add(glow);

    const ring = new THREE.Mesh(
      new THREE.RingGeometry(0.62, 0.7, 64),
      new THREE.MeshBasicMaterial({
        color: 0xd8b4fe,
        transparent: true,
        opacity: 0,
        blending: THREE.AdditiveBlending,
        depthWrite: false
      })
    );
    ring.rotation.x = -Math.PI / 2;
    ring.position.y = 0.01;
    scene.add(ring);

    // ---------- Dice ----------
    dice = new THREE.Group();

    const body = new THREE.Mesh(
      new RoundedBoxGeometry(1, 1, 1, 6, 0.13),
      new THREE.MeshPhysicalMaterial({
        color: 0xf8f6ff,
        roughness: 0.22,
        clearcoat: 1,
        clearcoatRoughness: 0.08
      })
    );
    body.castShadow = true;
    body.receiveShadow = true;
    dice.add(body);

    const pipGeo = new THREE.SphereGeometry(1, 24, 16);
    const pipMat = new THREE.MeshPhysicalMaterial({ color: 0x17121f, roughness: 0.35, clearcoat: 1 });
    const redMat = new THREE.MeshPhysicalMaterial({ color: 0xe11d48, roughness: 0.3, clearcoat: 1 });

    for (const [key, normal] of Object.entries(FACE_NORMALS)) {
      const n = Number(key);
      const [u, v] = faceAxes(normal);
      const r = n === 1 ? 0.14 : 0.085;
      for (const [a, b] of PIPS[n]) {
        const pip = new THREE.Mesh(pipGeo, n === 1 ? redMat : pipMat);
        pip.position
          .copy(normal)
          .multiplyScalar(0.5)
          .addScaledVector(u, a * 0.26)
          .addScaledVector(v, b * 0.26);
        pip.scale.set(normal.x ? r * 0.3 : r, normal.y ? r * 0.3 : r, normal.z ? r * 0.3 : r);
        dice.add(pip);
      }
    }

    dice.position.y = 1.4;
    dice.rotation.set(0.5, 0.6, 0.2);
    scene.add(dice);

    // ---------- Resize ----------
    function resize() {
      const w = window.innerWidth;
      const h = window.innerHeight;
      renderer.setSize(w, h, false);
      camera.aspect = w / h;
      camera.fov = w < h ? 40 : 32;
      camera.updateProjectionMatrix();
    }
    resize();
    window.addEventListener('resize', resize);

    // ---------- Render loop ----------
    let last = performance.now();
    let time = 0;
    const dq = new THREE.Quaternion();
    const showAxis = new THREE.Vector3(0.3, 1, 0.2).normalize();
    let raf;

    function tick() {
      raf = requestAnimationFrame(tick);
      const now = performance.now();
      const dt = Math.min((now - last) / 1000, 0.05);
      last = now;
      time += dt;
      phaseTime += dt;

      if (phase === 'showcase') {
        // Floating and slowly turning on the start screen
        dice.position.y = 1.4 + Math.sin(time * 1.6) * 0.12;
        dq.setFromAxisAngle(showAxis, dt * 0.8);
        dice.quaternion.premultiply(dq);
      } else if (phase === 'roll') {
        const t = Math.min(phaseTime / ROLL_TIME, 1);

        spinAxis.applyAxisAngle(UP, dt * 3);
        dq.setFromAxisAngle(spinAxis, dt * 16 * (1 - t * 0.7));
        dice.quaternion.premultiply(dq);

        const bounce = Math.abs(Math.sin(t * Math.PI * 2.5)) * 1.6 * Math.pow(1 - t, 1.6);
        dice.position.y = restHeight(dice.quaternion) + bounce + startOffset * Math.pow(1 - t, 3);
        dice.position.x = Math.sin(t * Math.PI * 2) * 0.5 * (1 - t);
        dice.position.z = Math.sin(t * Math.PI * 1.5) * 0.3 * (1 - t);

        if (t >= 1) {
          number = readNumber();
          settleFrom.copy(dice.quaternion);
          settleTo.copy(targetRotation(number));
          phase = 'settle';
          phaseTime = 0;
        }
      } else if (phase === 'settle') {
        const s = Math.min(phaseTime / SETTLE_TIME, 1);
        const ease = 1 - Math.pow(1 - s, 3);
        dice.quaternion.slerpQuaternions(settleFrom, settleTo, ease);
        dice.position.y = restHeight(dice.quaternion) + Math.sin(s * Math.PI) * 0.2 * (1 - s);
        dice.position.x = 0;
        dice.position.z = 0;

        if (s >= 1) {
          phase = 'rest';
          rolling = false;
          ringT = 0;
          navigator.vibrate?.(40);
        }
      }

      // Landing ring
      if (ringT < 1) {
        ringT = Math.min(ringT + dt / 0.7, 1);
        ring.scale.setScalar(1 + ringT * 1.8);
        ring.material.opacity = 0.7 * (1 - ringT);
        glow.material.opacity = 0.25 + 0.3 * (1 - ringT);
      }

      // Glow follows the dice
      glow.position.x = dice.position.x;
      glow.position.z = dice.position.z;

      // Camera moves a little with the phone tilt
      const tx = Math.max(-1, Math.min(1, gamma / 45)) * 0.8 + Math.sin(time * 0.3) * 0.25;
      camera.position.x += (tx - camera.position.x) * 0.05;
      camera.lookAt(lookTarget);

      renderer.render(scene, camera);
    }
    tick();

    return () => {
      cancelAnimationFrame(raf);
      window.removeEventListener('resize', resize);
      window.removeEventListener('devicemotion', onMotion);
      window.removeEventListener('deviceorientation', onOrientation);
      pmrem.dispose();
      renderer.dispose();
    };
  });
</script>

<main>
  <canvas bind:this={canvas} onclick={roll}></canvas>

  {#if !started}
    <section class="intro" out:fade={{ duration: 300 }}>
      <h1>Dice</h1>
      <p class="sub">Shake your phone to roll</p>
      <button class="start" onclick={start}>Start</button>
    </section>
  {:else}
    <header class="result" in:fade={{ delay: 300 }}>
      {#if rolling}
        <span class="label" in:fade={{ duration: 150 }}>Rolling…</span>
      {:else}
        {#key number}
          <span class="value" in:fly={{ y: 24, duration: 450 }}>{number}</span>
        {/key}
      {/if}
    </header>

    <p class="hint" in:fly={{ y: 20, delay: 500 }}>
      {#if motionOn}
        <span class="phone">📱</span> Shake or tap to roll
      {:else}
        👆 Tap the dice to roll
      {/if}
    </p>
  {/if}
</main>

<style>
  :global(body) {
    margin: 0;
    background: #0b0a17;
  }

  main {
    position: fixed;
    inset: 0;
    overflow: hidden;
    color: #f4f2ff;
    font-family: system-ui, -apple-system, 'Segoe UI', sans-serif;
    background:
      radial-gradient(circle at 50% 35%, rgba(139, 92, 246, 0.35), transparent 55%),
      radial-gradient(circle at 50% 100%, #1d1240 0%, #0b0a17 70%);
    user-select: none;
    -webkit-tap-highlight-color: transparent;
  }

  canvas {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    display: block;
    cursor: pointer;
  }

  /* ---------- Start screen ---------- */
  .intro {
    position: absolute;
    left: 0;
    right: 0;
    bottom: 10vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;
    pointer-events: none;
  }

  h1 {
    margin: 0 0 0.25rem;
    font-size: 3.2rem;
    font-weight: 800;
    letter-spacing: -0.03em;
    background: linear-gradient(90deg, #fff, #c4b5fd);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
  }

  .sub {
    margin: 0 0 1.8rem;
    color: #a9a3c9;
  }

  .start {
    pointer-events: auto;
    font: inherit;
    font-size: 1.15rem;
    font-weight: 700;
    color: white;
    padding: 1rem 3.2rem;
    border: none;
    border-radius: 999px;
    background: linear-gradient(135deg, #8b5cf6, #ec4899);
    box-shadow: 0 10px 30px rgba(236, 72, 153, 0.35);
    cursor: pointer;
    animation: pulse 2.4s ease-in-out infinite;
    transition: transform 0.15s ease;
  }

  .start:active {
    transform: scale(0.95);
  }

  /* ---------- Game screen ---------- */
  .result {
    position: absolute;
    top: 9vh;
    left: 0;
    right: 0;
    display: grid;
    place-items: center;
    height: 6rem;
    pointer-events: none;
  }

  .result > * {
    grid-area: 1 / 1;
  }

  .value {
    font-size: 5.5rem;
    font-weight: 800;
    line-height: 1;
    background: linear-gradient(135deg, #fff, #f0abfc);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
    filter: drop-shadow(0 6px 24px rgba(236, 72, 153, 0.45));
  }

  .label {
    font-size: 1.05rem;
    letter-spacing: 0.25em;
    text-transform: uppercase;
    color: #c4b5fd;
  }

  .hint {
    position: absolute;
    bottom: 6vh;
    left: 50%;
    transform: translateX(-50%);
    margin: 0;
    padding: 0.65rem 1.3rem;
    white-space: nowrap;
    border-radius: 999px;
    background: rgba(255, 255, 255, 0.06);
    border: 1px solid rgba(255, 255, 255, 0.12);
    backdrop-filter: blur(12px);
    color: #c9c3e6;
    font-size: 0.95rem;
    pointer-events: none;
  }

  .phone {
    display: inline-block;
    animation: wiggle 2.5s ease-in-out infinite;
  }

  @keyframes pulse {
    0%, 100% { box-shadow: 0 10px 30px rgba(236, 72, 153, 0.35); }
    50%      { box-shadow: 0 10px 45px rgba(139, 92, 246, 0.6); }
  }

  @keyframes wiggle {
    0%, 60%, 100% { transform: rotate(0deg); }
    65% { transform: rotate(-15deg); }
    70% { transform: rotate(15deg); }
    75% { transform: rotate(-10deg); }
    80% { transform: rotate(10deg); }
  }
</style>