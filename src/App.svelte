<script>
  import { onMount } from 'svelte';
  import HouseViewer from './lib/HouseViewer.svelte';

  let viewer;
  let isExploring = false;
  let isMobile = false;

  function startTour() {
    viewer?.start();
    isExploring = true;
  }

  function resetTour() {
    viewer?.resetToMenu();
    isExploring = false;
  }

  onMount(() => {
    const mobileQuery = window.matchMedia('(max-width: 900px) and (pointer: coarse)');
    const updateDevice = () => isMobile = mobileQuery.matches;
    updateDevice();
    mobileQuery.addEventListener('change', updateDevice);
    return () => mobileQuery.removeEventListener('change', updateDevice);
  });
</script>

<svelte:head>
  <meta name="description" content="Recorrido arquitectónico de una casa en primera persona." />
</svelte:head>

<main>
  <section class="viewer-shell" aria-label="Visor de la casa">
    <HouseViewer bind:this={viewer} />

    <header class="topbar">
      <div>
        <p class="eyebrow">RECORRIDO ARQUITECTÓNICO</p>
        <h1>Casa virtual</h1>
      </div>
      <button class="secondary" onclick={resetTour}>Reiniciar posición</button>
    </header>

    {#if !isExploring}
      <div class="start-card">
        <span class="badge">PRIMERA PERSONA</span>
        <h2>Explora la estructura</h2>
        {#if isMobile}
          <p>Usa las flechas de la pantalla para caminar y desliza sobre la escena para mirar alrededor.</p>
        {:else}
          <p>Usa <kbd>W</kbd><kbd>A</kbd><kbd>S</kbd><kbd>D</kbd> para caminar, el mouse para mirar y <kbd>Espacio</kbd> para saltar.</p>
        {/if}
        <button class="primary" onclick={startTour}>Iniciar recorrido</button>
      </div>
    {/if}

    <footer>
      <span>Modelo esperado: <code>/public/models/casa.glb</code></span>
      <span>Shift: correr · Esc: liberar cursor</span>
    </footer>
  </section>
</main>
