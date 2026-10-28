<script>
  let number = $state(1);
  let rolling = $state(false);
  let started = $state(false);

  let beta = 0;
  let gamma = 0;

  // Which number is shown for the current phone position?
  function readNumber() {
    const b = (beta * Math.PI) / 180;
    const g = (gamma * Math.PI) / 180;

    // Direction of gravity relative to the phone
    const up = Math.cos(b) * Math.cos(g);    // + = face up, - = face down
    const top = Math.sin(b) * Math.cos(g);   // + = top of phone up, - = top down
    const side = Math.sin(g);                // + = right side down, - = left side down

    const tolerance = 0.6; // higher = stricter, lower = more tolerant

    if (up > tolerance) return 6;      // flat, face up
    if (up < -tolerance) return 1;     // flat, face down
    if (top > tolerance) return 2;     // upright
    if (top < -tolerance) return 3;    // upside down
    if (side > tolerance) return 4;    // landscape
    if (side < -tolerance) return 5;   // landscape other way

    return number; // in between: keep the old number
  }

  function roll() {
    if (rolling) return;
    rolling = true;
    setTimeout(() => {
      number = readNumber();
      rolling = false;
    }, 1000);
  }

  function onMotion(event) {
    const a = event.accelerationIncludingGravity;
    if (!a) return;
    const force = Math.hypot(a.x, a.y, a.z);
    if (Math.abs(force - 9.8) > 12) roll(); // shake detected
  }

  function onOrientation(event) {
    beta = event.beta ?? 0;
    gamma = event.gamma ?? 0;
  }

  async function start() {
    // iPhone needs permission (must be triggered by a button tap)
    if (typeof DeviceMotionEvent.requestPermission === 'function') {
      const motion = await DeviceMotionEvent.requestPermission();
      const orientation = await DeviceOrientationEvent.requestPermission();
      if (motion !== 'granted' || orientation !== 'granted') return;
    }

    window.addEventListener('devicemotion', onMotion);
    window.addEventListener('deviceorientation', onOrientation);
    started = true;
  }
</script>

<main>
  {#if !started}
    <button onclick={start}>Start</button>
  {:else}
    <div class="dice">{rolling ? '🎲' : number}</div>
    <p>Shake the phone to roll!</p>
  {/if}
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100vh;
    font-family: sans-serif;
  }
  .dice {
    font-size: 10rem;
  }
  button {
    font-size: 2rem;
    padding: 1rem 2rem;
  }
</style>