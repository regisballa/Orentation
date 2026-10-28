<script>
  import { fly, fade, scale } from 'svelte/transition';

  let number = $state(1);
  let rolling = $state(false);
  let started = $state(false);
  let error = $state('');

  let beta = 0;
  let gamma = 0;

  // Pip positions on a 3x3 grid (cells 1 to 9)
  const pips = {
    1: [5],
    2: [1, 9],
    3: [1, 5, 9],
    4: [1, 3, 7, 9],
    5: [1, 3, 5, 7, 9],
    6: [1, 3, 4, 6, 7, 9]
  };

  // Cube rotation that brings each face to the front
  const faceRotation = {
    1: 'rotateX(0deg) rotateY(0deg)',
    2: 'rotateX(0deg) rotateY(-90deg)',
    3: 'rotateX(-90deg) rotateY(0deg)',
    4: 'rotateX(90deg) rotateY(0deg)',
    5: 'rotateX(0deg) rotateY(90deg)',
    6: 'rotateX(0deg) rotateY(180deg)'
  };

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

  function roll() {
    if (rolling) return;
    rolling = true;
    setTimeout(() => {
      number = readNumber();
      rolling = false;
      navigator.vibrate?.(40); // small buzz on landing (Android)
    }, 1000);
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
    error = '';
    try {
      // iPhone needs permission (must be triggered by a button tap)
      if (typeof DeviceMotionEvent?.requestPermission === 'function') {
        const motion = await DeviceMotionEvent.requestPermission();
        const orientation = await DeviceOrientationEvent.requestPermission();
        if (motion !== 'granted' || orientation !== 'granted') {
          error = 'Motion access was denied.';
          return;
        }
      }
    } catch {
      error = 'Could not get motion access.';
      return;
    }

    window.addEventListener('devicemotion', onMotion);
    window.addEventListener('deviceorientation', onOrientation);
    started = true;
  }
</script>

<main>
  <div class="glow"></div>

  {#if !started}
    <section class="intro" out:fade={{ duration: 250 }}>
      <div class="mini-die" aria-hidden="true">
        {#each Array(9) as _, i}
          <span class:pip={pips[5].includes(i + 1)}></span>
        {/each}
      </div>
      <h1>Dice</h1>
      <p class="sub">Shake your phone to roll</p>
      <button class="start" onclick={start}>Start</button>
      {#if error}
        <p class="error" in:fly={{ y: 8 }}>{error}</p>
      {/if}
    </section>
  {:else}
    <section class="game" in:scale={{ start: 0.85, duration: 450 }}>
      <!-- Tap also rolls, handy for testing on a laptop -->
      <div
        class="scene"
        role="button"
        tabindex="0"
        aria-label="Roll the dice"
        onclick={roll}
        onkeydown={(e) => e.key === 'Enter' && roll()}
      >
        <div class="tilt">
          <div class="tumble" class:rolling>
            <div class="cube" style:transform={faceRotation[number]}>
              {#each [1, 2, 3, 4, 5, 6] as face}
                <div class="face f{face}">
                  {#each Array(9) as _, i}
                    <span class:pip={pips[face].includes(i + 1)}></span>
                  {/each}
                </div>
              {/each}
            </div>
          </div>
        </div>
        <div class="shadow" class:rolling></div>
      </div>

      <div class="result">
        {#if rolling}
          <span class="label" in:fade={{ duration: 150 }}>Rolling…</span>
        {:else}
          {#key number}
            <span class="value" in:fly={{ y: 20, duration: 400 }}>{number}</span>
          {/key}
        {/if}
      </div>

      <p class="hint"><span class="phone">📱</span> Shake the phone to roll</p>
    </section>
  {/if}
</main>

<style>
  :global(body) {
    margin: 0;
    background: #0b0a17;
  }

  main {
    --size: 140px;
    --accent-1: #8b5cf6;
    --accent-2: #ec4899;

    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    min-height: 100dvh;
    overflow: hidden;
    color: #f4f2ff;
    font-family: system-ui, -apple-system, 'Segoe UI', sans-serif;
    background: radial-gradient(circle at 50% 25%, #2a1d5e 0%, #0b0a17 65%);
    user-select: none;
    -webkit-tap-highlight-color: transparent;
  }

  .glow {
    position: absolute;
    width: 420px;
    height: 420px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(139, 92, 246, 0.35), transparent 70%);
    filter: blur(40px);
    animation: breathe 6s ease-in-out infinite;
    pointer-events: none;
  }

  /* ---------- Intro ---------- */
  .intro,
  .game {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;
  }

  h1 {
    margin: 1.5rem 0 0.25rem;
    font-size: 3rem;
    font-weight: 800;
    letter-spacing: -0.03em;
    background: linear-gradient(90deg, #fff, #c4b5fd);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
  }

  .sub {
    margin: 0 0 2rem;
    color: #a9a3c9;
  }

  .mini-die {
    display: grid;
    grid-template: repeat(3, 1fr) / repeat(3, 1fr);
    width: 84px;
    height: 84px;
    padding: 12px;
    box-sizing: border-box;
    border-radius: 20px;
    background: linear-gradient(145deg, #ffffff, #ddd8ef);
    box-shadow: 0 20px 40px rgba(0, 0, 0, 0.45), inset 0 -4px 10px rgba(0, 0, 0, 0.12);
    animation: float 3s ease-in-out infinite;
  }

  .start {
    font: inherit;
    font-size: 1.15rem;
    font-weight: 700;
    color: white;
    padding: 1rem 3rem;
    border: none;
    border-radius: 999px;
    background: linear-gradient(135deg, var(--accent-1), var(--accent-2));
    box-shadow: 0 10px 30px rgba(236, 72, 153, 0.35);
    cursor: pointer;
    animation: pulse 2.4s ease-in-out infinite;
    transition: transform 0.15s ease;
  }

  .start:active {
    transform: scale(0.95);
  }

  .error {
    margin-top: 1rem;
    color: #fca5a5;
    font-size: 0.9rem;
  }

  /* ---------- 3D dice ---------- */
  .scene {
    position: relative;
    width: var(--size);
    height: var(--size);
    margin: 3rem 0 2.5rem;
    perspective: 900px;
    cursor: pointer;
    outline: none;
  }

  .tilt,
  .tumble,
  .cube {
    width: 100%;
    height: 100%;
    transform-style: preserve-3d;
  }

  .tilt {
    transform: rotateX(-22deg) rotateY(-28deg);
  }

  .tumble.rolling {
    animation: tumble 1s cubic-bezier(0.3, 0.6, 0.4, 1);
  }

  .cube {
    position: relative;
    transition: transform 0.7s cubic-bezier(0.34, 1.56, 0.64, 1);
  }

  .face {
    position: absolute;
    inset: 0;
    display: grid;
    grid-template: repeat(3, 1fr) / repeat(3, 1fr);
    padding: 18%;
    box-sizing: border-box;
    border-radius: 18px;
    background: linear-gradient(145deg, #ffffff 0%, #ece9f7 60%, #d9d4ec 100%);
    box-shadow: inset 0 0 22px rgba(60, 40, 120, 0.18);
    backface-visibility: hidden;
  }

  .f1 { transform: rotateY(0deg) translateZ(calc(var(--size) / 2)); }
  .f6 { transform: rotateY(180deg) translateZ(calc(var(--size) / 2)); }
  .f2 { transform: rotateY(90deg) translateZ(calc(var(--size) / 2)); }
  .f5 { transform: rotateY(-90deg) translateZ(calc(var(--size) / 2)); }
  .f3 { transform: rotateX(90deg) translateZ(calc(var(--size) / 2)); }
  .f4 { transform: rotateX(-90deg) translateZ(calc(var(--size) / 2)); }

  .pip {
    align-self: center;
    justify-self: center;
    width: 72%;
    aspect-ratio: 1;
    border-radius: 50%;
    background: radial-gradient(circle at 35% 30%, #4a4560, #12101c 70%);
    box-shadow: inset 0 2px 3px rgba(0, 0, 0, 0.6), 0 1px 0 rgba(255, 255, 255, 0.8);
  }

  .f1 .pip {
    width: 95%;
    background: radial-gradient(circle at 35% 30%, #ff6b8a, #c0153a 70%);
  }

  .shadow {
    position: absolute;
    left: 50%;
    bottom: -48px;
    width: 120%;
    height: 26px;
    border-radius: 50%;
    background: radial-gradient(ellipse, rgba(0, 0, 0, 0.55), transparent 70%);
    transform: translateX(-50%);
  }

  .shadow.rolling {
    animation: shadow 1s cubic-bezier(0.3, 0.6, 0.4, 1);
  }

  /* ---------- Result ---------- */
  .result {
    display: grid;
    place-items: center;
    height: 5rem;
  }

  .value {
    font-size: 4.5rem;
    font-weight: 800;
    line-height: 1;
    background: linear-gradient(135deg, #fff, #f0abfc);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
  }

  .label {
    font-size: 1.1rem;
    letter-spacing: 0.2em;
    text-transform: uppercase;
    color: #c4b5fd;
  }

  .hint {
    margin-top: 1.5rem;
    padding: 0.6rem 1.2rem;
    border-radius: 999px;
    background: rgba(255, 255, 255, 0.06);
    border: 1px solid rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(10px);
    color: #a9a3c9;
    font-size: 0.95rem;
  }

  .phone {
    display: inline-block;
    animation: wiggle 2.5s ease-in-out infinite;
  }

  /* ---------- Keyframes ---------- */
  @keyframes tumble {
    0%   { transform: translateY(0)      rotateX(0deg)   rotateY(0deg)   rotateZ(0deg); }
    30%  { transform: translateY(-110px) rotateX(250deg) rotateY(160deg) rotateZ(80deg); }
    60%  { transform: translateY(-10px)  rotateX(520deg) rotateY(430deg) rotateZ(220deg); }
    80%  { transform: translateY(-40px)  rotateX(650deg) rotateY(630deg) rotateZ(320deg); }
    100% { transform: translateY(0)      rotateX(720deg) rotateY(720deg) rotateZ(360deg); }
  }

  @keyframes shadow {
    0%, 100% { transform: translateX(-50%) scale(1);   opacity: 1; }
    30%      { transform: translateX(-50%) scale(0.45); opacity: 0.35; }
    60%      { transform: translateX(-50%) scale(0.95); opacity: 0.9; }
    80%      { transform: translateX(-50%) scale(0.75); opacity: 0.6; }
  }

  @keyframes float {
    0%, 100% { transform: translateY(0) rotate(-6deg); }
    50%      { transform: translateY(-10px) rotate(6deg); }
  }

  @keyframes pulse {
    0%, 100% { box-shadow: 0 10px 30px rgba(236, 72, 153, 0.35); }
    50%      { box-shadow: 0 10px 45px rgba(139, 92, 246, 0.6); }
  }

  @keyframes breathe {
    0%, 100% { transform: scale(1);   opacity: 0.8; }
    50%      { transform: scale(1.15); opacity: 1; }
  }

  @keyframes wiggle {
    0%, 60%, 100% { transform: rotate(0deg); }
    65% { transform: rotate(-15deg); }
    70% { transform: rotate(15deg); }
    75% { transform: rotate(-10deg); }
    80% { transform: rotate(10deg); }
  }

  @media (prefers-reduced-motion: reduce) {
    .tumble.rolling,
    .shadow.rolling,
    .glow,
    .mini-die,
    .start,
    .phone {
      animation: none;
    }
  }
</style>