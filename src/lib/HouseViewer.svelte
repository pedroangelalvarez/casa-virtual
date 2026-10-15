<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import { Octree } from 'three/examples/jsm/math/Octree.js';
  import { Capsule } from 'three/examples/jsm/math/Capsule.js';
  import { PointerLockControls } from 'three/examples/jsm/controls/PointerLockControls.js';

  let container;
  let message = 'Preparando recorrido…';
  let renderer;
  let camera;
  let controls;
  let animationFrame;
  let isMobile = false;
  let isLocked = false;
  let mobileTourActive = false;
  let lookPointerId = null;
  let lastTouchX = 0;
  let lastTouchY = 0;

  const keys = new Set();
  const worldOctree = new Octree();
  const playerVelocity = new THREE.Vector3();
  const playerCollider = new Capsule(
    new THREE.Vector3(0, 0.35, 0),
    new THREE.Vector3(0, 1.65, 0),
    0.35
  );
  const spawn = new THREE.Vector3(0, 1.65, 4);
  let playerOnFloor = false;

  export function start() {
    if (isMobile) {
      mobileTourActive = true;
      message = 'Recorrido táctil activo';
      return;
    }
    controls?.lock();
  }

  export function resetPlayer() {
    playerVelocity.set(0, 0, 0);
    playerCollider.start.set(spawn.x, spawn.y - 1.3, spawn.z);
    playerCollider.end.copy(spawn);
    camera?.position.copy(playerCollider.end);
  }

  export function resetToMenu() {
    mobileTourActive = false;
    lookPointerId = null;
    controls?.unlock();
    resetPlayer();
    message = 'Posición reiniciada';
  }

  function pressControl(event, code) {
    event.preventDefault();
    event.stopPropagation();
    keys.add(code);
    event.currentTarget.setPointerCapture?.(event.pointerId);
  }

  function releaseControl(event, code) {
    event.preventDefault();
    event.stopPropagation();
    keys.delete(code);
  }

  function startLooking(event) {
    if (!isMobile || !mobileTourActive || event.pointerType !== 'touch') return;
    lookPointerId = event.pointerId;
    lastTouchX = event.clientX;
    lastTouchY = event.clientY;
    event.currentTarget.setPointerCapture?.(event.pointerId);
  }

  function lookAround(event) {
    if (event.pointerId !== lookPointerId) return;
    const deltaX = event.clientX - lastTouchX;
    const deltaY = event.clientY - lastTouchY;
    camera.rotation.y -= deltaX * 0.008;
    camera.rotation.x = THREE.MathUtils.clamp(camera.rotation.x - deltaY * 0.008, -1.45, 1.45);
    lastTouchX = event.clientX;
    lastTouchY = event.clientY;
  }

  function stopLooking(event) {
    if (event.pointerId === lookPointerId) lookPointerId = null;
  }

  function makeFallbackHouse(scene) {
    const group = new THREE.Group();
    const concrete = new THREE.MeshStandardMaterial({ color: 0xb8b5ae, roughness: 0.82 });
    const floorMaterial = new THREE.MeshStandardMaterial({ color: 0x78716c, roughness: 0.92 });
    const addBox = (width, height, depth, x, y, z, material = concrete) => {
      const mesh = new THREE.Mesh(new THREE.BoxGeometry(width, height, depth), material);
      mesh.position.set(x, y, z);
      mesh.castShadow = true;
      mesh.receiveShadow = true;
      group.add(mesh);
    };

    addBox(18, 0.2, 14, 0, -0.1, 0, floorMaterial);
    addBox(18, 3.2, 0.25, 0, 1.6, -7);
    addBox(18, 3.2, 0.25, 0, 1.6, 7);
    addBox(0.25, 3.2, 14, -9, 1.6, 0);
    addBox(0.25, 3.2, 14, 9, 1.6, 0);
    addBox(0.25, 3.2, 5, -2.5, 1.6, -4.5);
    addBox(0.25, 3.2, 4.5, -2.5, 1.6, 3.5);
    addBox(5, 3.2, 0.25, 2.25, 1.6, 1.2);
    addBox(0.25, 3.2, 3.5, 4.8, 1.6, -3.25);
    scene.add(group);
    return group;
  }

  function addLights(scene) {
    scene.add(new THREE.HemisphereLight(0xf6f7ff, 0x504a43, 2.4));
    const sun = new THREE.DirectionalLight(0xfff4df, 3.2);
    sun.position.set(8, 15, 5);
    sun.castShadow = true;
    sun.shadow.mapSize.set(2048, 2048);
    scene.add(sun);
  }

  function prepareModel(model) {
    model.traverse((node) => {
      if (!node.isMesh) return;
      node.castShadow = true;
      node.receiveShadow = true;
      if (Array.isArray(node.material)) {
        node.material.forEach((material) => material.needsUpdate = true);
      } else if (node.material) {
        node.material.needsUpdate = true;
      }
    });

    const spawnMarker = model.getObjectByName('Spawn');
    if (spawnMarker) spawn.copy(spawnMarker.getWorldPosition(new THREE.Vector3()));
  }

  function playerCollisions() {
    const result = worldOctree.capsuleIntersect(playerCollider);
    playerOnFloor = false;
    if (!result) return;

    playerOnFloor = result.normal.y > 0;
    if (!playerOnFloor) {
      playerVelocity.addScaledVector(result.normal, -result.normal.dot(playerVelocity));
    } else {
      playerVelocity.y = 0;
    }
    playerCollider.translate(result.normal.multiplyScalar(result.depth));
  }

  function getForwardVector(target) {
    camera.getWorldDirection(target);
    target.y = 0;
    target.normalize();
    return target;
  }

  function getSideVector(target) {
    camera.getWorldDirection(target);
    target.y = 0;
    target.normalize();
    target.cross(camera.up);
    return target;
  }

  function movePlayer(delta) {
    const damping = Math.exp(-8 * delta) - 1;
    playerVelocity.addScaledVector(playerVelocity, damping);
    if (!playerOnFloor) playerVelocity.y -= 24 * delta;

    const speed = keys.has('ShiftLeft') || keys.has('ShiftRight') ? 14 : 7.5;
    const forward = new THREE.Vector3();
    const side = new THREE.Vector3();
    if (keys.has('KeyW')) playerVelocity.add(getForwardVector(forward).multiplyScalar(speed * delta));
    if (keys.has('KeyS')) playerVelocity.add(getForwardVector(forward).multiplyScalar(-speed * delta));
    if (keys.has('KeyA')) playerVelocity.add(getSideVector(side).multiplyScalar(-speed * delta));
    if (keys.has('KeyD')) playerVelocity.add(getSideVector(side).multiplyScalar(speed * delta));

    const deltaPosition = playerVelocity.clone().multiplyScalar(delta);
    playerCollider.translate(deltaPosition);
    playerCollisions();
    camera.position.copy(playerCollider.end);

    if (camera.position.y < -15) resetPlayer();
  }

  onMount(() => {
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x9fb4c2);
    scene.fog = new THREE.Fog(0x9fb4c2, 24, 80);

    camera = new THREE.PerspectiveCamera(70, 1, 0.05, 200);
    camera.rotation.order = 'YXZ';
    camera.position.copy(spawn);

    renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: 'high-performance' });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.15;
    container.appendChild(renderer.domElement);

    controls = new PointerLockControls(camera, renderer.domElement);
    controls.addEventListener('lock', () => {
      isLocked = true;
      message = 'Recorrido activo';
    });
    controls.addEventListener('unlock', () => {
      isLocked = false;
      keys.clear();
      message = 'Cursor liberado — pulsa Iniciar recorrido para continuar';
    });
    addLights(scene);

    const mobileQuery = window.matchMedia('(max-width: 900px) and (pointer: coarse)');
    const updateDevice = () => isMobile = mobileQuery.matches;
    updateDevice();
    mobileQuery.addEventListener('change', updateDevice);

    const loader = new GLTFLoader();
    loader.load(
      '/models/casa.glb',
      (gltf) => {
        const model = gltf.scene;
        scene.add(model);
        model.updateMatrixWorld(true);
        prepareModel(model);
        worldOctree.fromGraphNode(model);
        resetPlayer();
        message = 'Modelo casa.glb cargado';
      },
      undefined,
      () => {
        const fallback = makeFallbackHouse(scene);
        fallback.updateMatrixWorld(true);
        worldOctree.fromGraphNode(fallback);
        resetPlayer();
        message = 'Modo demostración — añade public/models/casa.glb';
      }
    );

    const resize = () => {
      const { width, height } = container.getBoundingClientRect();
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
      renderer.setSize(width, height, false);
    };
    const keydown = (event) => {
      if (event.code === 'Space' && playerOnFloor && controls.isLocked) {
        playerVelocity.y = 8.5;
        event.preventDefault();
      }
      keys.add(event.code);
    };
    const keyup = (event) => keys.delete(event.code);
    let lastFrame = performance.now();
    const animate = (currentFrame) => {
      const delta = Math.min((currentFrame - lastFrame) / 1000, 0.05);
      lastFrame = currentFrame;
      if (controls.isLocked || mobileTourActive) movePlayer(delta);
      renderer.render(scene, camera);
      animationFrame = requestAnimationFrame(animate);
    };

    window.addEventListener('resize', resize);
    window.addEventListener('keydown', keydown);
    window.addEventListener('keyup', keyup);
    resize();
    animate();

    return () => {
      cancelAnimationFrame(animationFrame);
      window.removeEventListener('resize', resize);
      window.removeEventListener('keydown', keydown);
      window.removeEventListener('keyup', keyup);
      mobileQuery.removeEventListener('change', updateDevice);
      controls?.unlock();
      renderer?.dispose();
    };
  });
</script>

<div class="stage" bind:this={container} role="application" aria-label="Escena de recorrido táctil" onpointerdown={startLooking} onpointermove={lookAround} onpointerup={stopLooking} onpointercancel={stopLooking}>
  <span class="crosshair" aria-hidden="true"></span>
  <p class="viewer-status">{message}</p>
  {#if isMobile && (isLocked || mobileTourActive)}
    <div class="touch-controls" aria-label="Controles táctiles de movimiento">
      <button class="touch-button up" aria-label="Caminar hacia adelante" onpointerdown={(event) => pressControl(event, 'KeyW')} onpointerup={(event) => releaseControl(event, 'KeyW')} onpointercancel={(event) => releaseControl(event, 'KeyW')} onlostpointercapture={(event) => releaseControl(event, 'KeyW')}>↑</button>
      <button class="touch-button left" aria-label="Caminar hacia la izquierda" onpointerdown={(event) => pressControl(event, 'KeyA')} onpointerup={(event) => releaseControl(event, 'KeyA')} onpointercancel={(event) => releaseControl(event, 'KeyA')} onlostpointercapture={(event) => releaseControl(event, 'KeyA')}>←</button>
      <button class="touch-button down" aria-label="Caminar hacia atrás" onpointerdown={(event) => pressControl(event, 'KeyS')} onpointerup={(event) => releaseControl(event, 'KeyS')} onpointercancel={(event) => releaseControl(event, 'KeyS')} onlostpointercapture={(event) => releaseControl(event, 'KeyS')}>↓</button>
      <button class="touch-button right" aria-label="Caminar hacia la derecha" onpointerdown={(event) => pressControl(event, 'KeyD')} onpointerup={(event) => releaseControl(event, 'KeyD')} onpointercancel={(event) => releaseControl(event, 'KeyD')} onlostpointercapture={(event) => releaseControl(event, 'KeyD')}>→</button>
    </div>
  {/if}
</div>
