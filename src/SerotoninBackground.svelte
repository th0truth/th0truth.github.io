<script lang="ts">
  import { onMount } from 'svelte';
  import * as THREE from 'three';

  let canvasEl: HTMLCanvasElement;

  const hoNoiseShader = `
    vec3 mod289(vec3 x) {
      return x - floor(x * (1.0 / 289.0)) * 289.0;
    }

    vec4 mod289(vec4 x) {
      return x - floor(x * (1.0 / 289.0)) * 289.0;
    }

    vec4 permute(vec4 x) {
      return mod289(((x * 34.0) + 1.0) * x);
    }

    vec4 taylorInvSqrt(vec4 r) {
      return 1.79284291400159 - 0.85373472095314 * r;
    }

    float snoise(vec3 v) {
      const vec2 C = vec2(1.0 / 6.0, 1.0 / 3.0);
      const vec4 D = vec4(0.0, 0.5, 1.0, 2.0);

      vec3 i  = floor(v + dot(v, C.yyy));
      vec3 x0 = v - i + dot(i, C.xxx);

      vec3 g = step(x0.yzx, x0.xyz);
      vec3 l = 1.0 - g;
      vec3 i1 = min(g.xyz, l.zxy);
      vec3 i2 = max(g.xyz, l.zxy);

      vec3 x1 = x0 - i1 + C.xxx;
      vec3 x2 = x0 - i2 + C.yyy;
      vec3 x3 = x0 - D.yyy;

      i = mod289(i);
      vec4 p = permute(permute(permute(
                i.z + vec4(0.0, i1.z, i2.z, 1.0))
              + i.y + vec4(0.0, i1.y, i2.y, 1.0))
              + i.x + vec4(0.0, i1.x, i2.x, 1.0));

      float n_ = 0.142857142857;
      vec3 ns = n_ * D.wyz - D.xzx;

      vec4 j = p - 49.0 * floor(p * ns.z * ns.z);

      vec4 x_ = floor(j * ns.z);
      vec4 y_ = floor(j - 7.0 * x_);

      vec4 x = x_ * ns.x + ns.yyyy;
      vec4 y = y_ * ns.x + ns.yyyy;
      vec4 h = 1.0 - abs(x) - abs(y);

      vec4 b0 = vec4(x.xy, y.xy);
      vec4 b1 = vec4(x.zw, y.zw);

      vec4 s0 = floor(b0) * 2.0 + 1.0;
      vec4 s1 = floor(b1) * 2.0 + 1.0;
      vec4 sh = -step(h, vec4(0.0));

      vec4 a0 = b0.xzyw + s0.xzyw * sh.xxyy;
      vec4 a1 = b1.xzyw + s1.xzyw * sh.zzww;

      vec3 p0 = vec3(a0.xy, h.x);
      vec3 p1 = vec3(a0.zw, h.y);
      vec3 p2 = vec3(a1.xy, h.z);
      vec3 p3 = vec3(a1.zw, h.w);

      vec4 norm = taylorInvSqrt(vec4(dot(p0, p0), dot(p1, p1), dot(p2, p2), dot(p3, p3)));
      p0 *= norm.x;
      p1 *= norm.y;
      p2 *= norm.z;
      p3 *= norm.w;

      vec4 m = max(0.6 - vec4(dot(x0, x0), dot(x1, x1), dot(x2, x2), dot(x3, x3)), 0.0);
      m = m * m;
      return 42.0 * dot(m * m, vec4(dot(p0, x0), dot(p1, x1), dot(p2, x2), dot(p3, x3)));
    }

    vec3 snoiseVec3(vec3 x) {
      float s  = snoise(vec3(x));
      float s1 = snoise(vec3(x.y - 19.1, x.z + 33.4, x.x + 47.2));
      float s2 = snoise(vec3(x.z + 74.2, x.x - 124.5, x.y + 99.4));
      return vec3(s, s1, s2);
    }

    vec3 curlNoise(vec3 p) {
      const float e = 0.1;
      vec3 dx = vec3(e, 0.0, 0.0);
      vec3 dy = vec3(0.0, e, 0.0);
      vec3 dz = vec3(0.0, 0.0, e);

      vec3 p_x0 = snoiseVec3(p - dx);
      vec3 p_x1 = snoiseVec3(p + dx);
      vec3 p_y0 = snoiseVec3(p - dy);
      vec3 p_y1 = snoiseVec3(p + dy);
      vec3 p_z0 = snoiseVec3(p - dz);
      vec3 p_z1 = snoiseVec3(p + dz);

      float x = p_y1.z - p_y0.z - p_z1.y + p_z0.y;
      float y = p_z1.x - p_z0.x - p_x1.z + p_z0.z;
      float z = p_x1.y - p_x0.y - p_y1.x + p_y0.x;

      const float divisor = 1.0 / (2.0 * e);
      return normalize(vec3(x, y, z) * divisor);
    }
  `;

  const quadVertexShader = `
    void main() {
      gl_Position = vec4(position, 1.0);
    }
  `;

  // Organic Position Simulation Shader
  const positionFragmentShader = `
    precision highp float;
    uniform sampler2D uOriginalPositionsTexture;
    uniform sampler2D uPositionsTexture;
    uniform sampler2D uVelocitiesTexture;
    uniform vec2 uTextureResolution;
    uniform float uNoiseFrequency;
    uniform float uNoiseAmplitude;
    uniform float uTime;
    uniform float uDeltaTime;

    ${hoNoiseShader}

    void main() {
      vec2 uv = gl_FragCoord.xy / uTextureResolution;
      vec3 originalPosition = texture2D(uOriginalPositionsTexture, uv).rgb;
      vec3 position = texture2D(uPositionsTexture, uv).rgb;
      vec4 velData = texture2D(uVelocitiesTexture, uv);
      vec3 velocity = velData.rgb;

      vec3 diffPosition = originalPosition - position;

      // Soft organic fluid curl drift
      vec3 curl = curlNoise((originalPosition + uTime * 0.05) * uNoiseFrequency) * uNoiseAmplitude;
      
      position += curl;
      position += velocity * uDeltaTime;

      // Gentle spring return
      position += diffPosition * (uDeltaTime * 0.36);

      float temperature = clamp(length(diffPosition) * 0.70, 0.0, 2.5);

      gl_FragColor = vec4(position, temperature);
    }
  `;

  // Silky smooth Velocity Shader
  const velocityFragmentShader = `
    precision highp float;
    uniform sampler2D uOriginalPositionsTexture;
    uniform sampler2D uPositionsTexture;
    uniform sampler2D uVelocitiesTexture;
    uniform vec2 uTextureResolution;
    uniform vec3 uPointer;
    uniform float uGravity;
    uniform vec3 uPointerStart;
    uniform float uDeltaTime;

    ${hoNoiseShader}

    void main() {
      vec2 uv = gl_FragCoord.xy / uTextureResolution;
      vec3 originalPosition = texture2D(uOriginalPositionsTexture, uv).rgb;
      vec3 position = texture2D(uPositionsTexture, uv).rgb;
      vec3 velocity = texture2D(uVelocitiesTexture, uv).rgb;

      vec3 diffToPointer = position - uPointer;
      float dist2D = length(diffToPointer.xy);
      float safeDist = max(dist2D, 0.001);
      vec3 dirToPointer = diffToPointer / safeDist;

      float pointerSpeed = min(distance(uPointer.xy, uPointerStart.xy), 2.0);
      
      float radius = 1.20;
      float hitInfluence = 1.0 - smoothstep(0.0, radius, dist2D);

      float forceMag = 0.0;
      if (uGravity >= 0.0) {
        // Natural gentle fluid parting
        forceMag = hitInfluence * (0.85 + pointerSpeed * 1.1);
      } else {
        // Gravitational vortex pull on click
        forceMag = -hitInfluence * 2.5;
      }

      vec3 noiseOffset = snoise(velocity * 0.28 + position * 0.20) * 0.07;
      vec3 target = originalPosition + dirToPointer * forceMag + noiseOffset;

      // Smooth acceleration with drag damping
      vec3 desiredVel = (target - position) * 2.4;
      velocity = mix(velocity * 0.92, desiredVel, 0.08);

      // Safe clamp
      float maxVel = 5.0;
      if (length(velocity) > maxVel) {
        velocity = normalize(velocity) * maxVel;
      }

      gl_FragColor = vec4(velocity, 1.0);
    }
  `;

  onMount(() => {
    if (!canvasEl) return;

    let width = window.innerWidth;
    let height = window.innerHeight;
    const isMobile = width < 600;
    const isTablet = width >= 600 && width < 1024;

    // Desktop: 112x112 = 12,544 particles
    // Tablet: 80x80 = 6,400 particles
    // Mobile: 56x56 = 3,136 particles
    const f = isMobile ? 56 : isTablet ? 80 : 112;
    const u = f;
    const T = f * u;

    const highlightColor = new THREE.Vector3(0.58, 0.64, 0.76); // Subtle muted silver-slate
    const mainScene = new THREE.Scene();

    let aspect = width / height;
    const frustumSize = 2;
    const camera = new THREE.OrthographicCamera(
      (frustumSize * aspect) / -2,
      (frustumSize * aspect) / 2,
      frustumSize / 2,
      frustumSize / -2,
      0.1,
      100
    );
    camera.position.z = width < 500 ? 76 : width < 980 ? 46 : 26;

    const simScene = new THREE.Scene();
    const simCamera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);

    const displayUniforms: { [key: string]: THREE.IUniform } = {
      uPositionsTexture: { value: null },
      uVelocitiesTexture: { value: null },
      uGravity: { value: 1.0 },
      uHighlightColor: { value: highlightColor }
    };

    const simUniforms: { [key: string]: THREE.IUniform } = {
      uOriginalPositionsTexture: { value: null },
      uPositionsTexture: { value: null },
      uVelocitiesTexture: { value: null },
      uTextureResolution: { value: new THREE.Vector2(f, u) },
      uGravity: { value: 1.0 },
      uPointer: { value: new THREE.Vector3(9999, 9999, 0) },
      uPointerStart: { value: new THREE.Vector3(9999, 9999, 0) },
      uNoiseFrequency: { value: 0.35 },
      uNoiseAmplitude: { value: 0.0014 },
      uTime: { value: 0 },
      uDeltaTime: { value: 0.003 }
    };

    const quadGeo = new THREE.BufferGeometry();
    const quadVertices = new Float32Array([
      -1, -1, 0,
       1, -1, 0,
       1,  1, 0,
      -1, -1, 0,
       1,  1, 0,
      -1,  1, 0
    ]);
    quadGeo.setAttribute('position', new THREE.BufferAttribute(quadVertices, 3));
    const simMaterial = new THREE.ShaderMaterial({
      uniforms: simUniforms,
      vertexShader: quadVertexShader,
      fragmentShader: positionFragmentShader
    });
    const simMesh = new THREE.Mesh(quadGeo, simMaterial);
    simScene.add(simMesh);

    const rtOptions = {
      minFilter: THREE.NearestFilter,
      magFilter: THREE.NearestFilter,
      format: THREE.RGBAFormat,
      type: THREE.HalfFloatType
    };

    let fboPos1 = new THREE.WebGLRenderTarget(f, u, rtOptions);
    let fboPos2 = fboPos1.clone();
    let fboVel1 = fboPos1.clone();
    let fboVel2 = fboPos1.clone();

    const particleGeometry = new THREE.BufferGeometry();
    const lookupCoords = new Float32Array(T * 3);
    const initPosData = new Float32Array(T * 4);
    const initVelData = new Float32Array(T * 4);

    // Isotropic 3D Volumetric dispersion: ZERO seam lines, ZERO polar clustering!
    for (let i = 0; i < T; i++) {
      const i3 = i * 3;
      const i4 = i * 4;

      const uRand = Math.random();
      const vRand = Math.random();
      const theta = uRand * 2.0 * Math.PI;
      const phi = Math.acos(2.0 * vRand - 1.0);
      
      // Radius distribution with volume spread
      const r = (0.55 + Math.pow(Math.random(), 0.6) * 0.95) * 1.35;

      const xSpread = aspect > 1.2 ? 1.35 : 1.05;
      const x = r * Math.sin(phi) * Math.cos(theta) * xSpread;
      const y = r * Math.sin(phi) * Math.sin(theta);
      const z = r * Math.cos(phi) * 0.75;

      initPosData[i4 + 0] = x;
      initPosData[i4 + 1] = y;
      initPosData[i4 + 2] = z;
      initPosData[i4 + 3] = 1.0;

      lookupCoords[i3 + 0] = ((i % f) + 0.5) / f;
      lookupCoords[i3 + 1] = (Math.floor(i / f) + 0.5) / u;
      lookupCoords[i3 + 2] = 0;
    }

    particleGeometry.setAttribute('position', new THREE.BufferAttribute(lookupCoords, 3));

    const origPosTex = new THREE.DataTexture(initPosData, f, u, THREE.RGBAFormat, THREE.FloatType);
    origPosTex.needsUpdate = true;
    simUniforms.uOriginalPositionsTexture.value = origPosTex;

    const curPosTex = new THREE.DataTexture(new Float32Array(initPosData), f, u, THREE.RGBAFormat, THREE.FloatType);
    curPosTex.needsUpdate = true;
    simUniforms.uPositionsTexture.value = curPosTex;

    const curVelTex = new THREE.DataTexture(initVelData, f, u, THREE.RGBAFormat, THREE.FloatType);
    curVelTex.needsUpdate = true;
    simUniforms.uVelocitiesTexture.value = curVelTex;

    displayUniforms.uPositionsTexture.value = curPosTex;

    // Serotonin-style muted slate particles: soft, non-glaring, not bright white
    const pointsMaterial = new THREE.PointsMaterial({
      size: isMobile ? 1.2 : 0.95,
      opacity: 0.35,
      transparent: true,
      depthWrite: false
    });

    pointsMaterial.onBeforeCompile = (shader) => {
      shader.uniforms = {
        ...shader.uniforms,
        ...displayUniforms
      };

      shader.vertexShader = `
        uniform sampler2D uPositionsTexture;
        uniform sampler2D uVelocitiesTexture;
        varying float vTemperature;
        ${shader.vertexShader}
      `.replace(
        '#include <begin_vertex>',
        `
        #include <begin_vertex>
        vec4 posData = texture2D(uPositionsTexture, position.xy);
        transformed = posData.rgb;
        vTemperature = posData.a;
        `
      );

      shader.fragmentShader = `
        uniform sampler2D uPositionsTexture;
        uniform vec3 uHighlightColor;
        varying float vTemperature;
        ${shader.fragmentShader}
      `.replace(
        'vec4 diffuseColor = vec4( diffuse, opacity );',
        `
        vec3 baseParticleColor = vec3(0.26, 0.28, 0.32);
        vec3 activeParticleColor = vec3(0.55, 0.60, 0.72);
        vec3 finalColor = mix(baseParticleColor, activeParticleColor, clamp(vTemperature * 0.45, 0.0, 1.0));
        float alpha = opacity * smoothstep(0.01, 0.85, vTemperature / 1.6 + 0.28);
        vec4 diffuseColor = vec4( finalColor, clamp(alpha, 0.12, 0.45) );
        `
      );
    };

    const particlePoints = new THREE.Points(particleGeometry, pointsMaterial);
    particlePoints.position.set(0, 0, 0);
    mainScene.add(particlePoints);

    const renderer = new THREE.WebGLRenderer({
      canvas: canvasEl,
      powerPreference: 'high-performance',
      antialias: false,
      alpha: true,
      preserveDrawingBuffer: false
    });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 1.5));
    renderer.setSize(width, height);
    renderer.setClearColor(0x000000, 0);
    renderer.toneMapping = THREE.ACESFilmicToneMapping;

    const raycastPlaneGeo = new THREE.PlaneGeometry(100, 100);
    const raycastPlaneMat = new THREE.MeshBasicMaterial({
      side: THREE.DoubleSide,
      depthTest: false,
      depthWrite: false,
      visible: false
    });
    const raycastPlane = new THREE.Mesh(raycastPlaneGeo, raycastPlaneMat);
    raycastPlane.position.set(0, 0, 0.5);
    mainScene.add(raycastPlane);

    const raycaster = new THREE.Raycaster();
    const mouseNorm = new THREE.Vector2(-9999, -9999);
    
    let targetGravity = 1.0;
    let currentGravity = 1.0;
    let pointerLagCounter = 0;
    const clock = new THREE.Clock();
    let isVisible = true;

    function handlePointerCoord(clientX: number, clientY: number) {
      mouseNorm.x = (clientX / width) * 2 - 1;
      mouseNorm.y = -(clientY / height) * 2 + 1;
    }

    function onMouseMove(e: MouseEvent) {
      handlePointerCoord(e.clientX, e.clientY);
    }

    function onTouchMove(e: TouchEvent) {
      if (e.touches.length > 0) {
        handlePointerCoord(e.touches[0].clientX, e.touches[0].clientY);
      }
    }

    function onMouseDown() {
      targetGravity = -1.0;
    }

    function onMouseUp() {
      targetGravity = 1.0;
    }

    function onTouchStart(e: TouchEvent) {
      targetGravity = -1.0;
      if (e.touches.length > 0) {
        handlePointerCoord(e.touches[0].clientX, e.touches[0].clientY);
      }
    }

    function onTouchEnd() {
      targetGravity = 1.0;
    }

    function onResize() {
      width = window.innerWidth;
      height = window.innerHeight;
      aspect = width / height;

      camera.left = (frustumSize * aspect) / -2;
      camera.right = (frustumSize * aspect) / 2;
      camera.top = frustumSize / 2;
      camera.bottom = frustumSize / -2;
      camera.position.z = width < 500 ? 76 : width < 980 ? 46 : 26;
      camera.updateProjectionMatrix();

      renderer.setSize(width, height);
    }

    function onVisibilityChange() {
      isVisible = !document.hidden;
    }

    window.addEventListener('mousemove', onMouseMove, { passive: true });
    window.addEventListener('mousedown', onMouseDown, { passive: true });
    window.addEventListener('mouseup', onMouseUp, { passive: true });
    window.addEventListener('touchmove', onTouchMove, { passive: true });
    window.addEventListener('touchstart', onTouchStart, { passive: true });
    window.addEventListener('touchend', onTouchEnd, { passive: true });
    window.addEventListener('resize', onResize, { passive: true });
    document.addEventListener('visibilitychange', onVisibilityChange);

    let animFrameId: number;

    function renderLoop() {
      if (!isVisible) {
        animFrameId = requestAnimationFrame(renderLoop);
        return;
      }

      const elapsedTime = clock.getElapsedTime();

      // Smooth gravity transition
      currentGravity += (targetGravity - currentGravity) * 0.10;

      // 1. Simulation Step: Velocities
      const tempVel = fboVel1;
      fboVel1 = fboVel2;
      fboVel2 = tempVel;

      simMaterial.vertexShader = quadVertexShader;
      simMaterial.fragmentShader = velocityFragmentShader;
      simMaterial.needsUpdate = true;

      renderer.setRenderTarget(fboVel1);
      renderer.clear();
      renderer.render(simScene, simCamera);
      simUniforms.uVelocitiesTexture.value = fboVel1.texture;

      // 2. Simulation Step: Positions
      const tempPos = fboPos1;
      fboPos1 = fboPos2;
      fboPos2 = tempPos;

      simMaterial.vertexShader = quadVertexShader;
      simMaterial.fragmentShader = positionFragmentShader;
      simMaterial.needsUpdate = true;

      renderer.setRenderTarget(fboPos1);
      renderer.clear();
      renderer.render(simScene, simCamera);
      simUniforms.uPositionsTexture.value = fboPos1.texture;

      // 3. Render Particles to Screen
      renderer.setRenderTarget(null);
      displayUniforms.uPositionsTexture.value = fboPos1.texture;
      displayUniforms.uGravity.value = currentGravity;
      displayUniforms.uHighlightColor.value = highlightColor;

      simUniforms.uTime.value = elapsedTime;
      simUniforms.uDeltaTime.value = 0.003;
      simUniforms.uGravity.value = currentGravity;

      // Raycast cursor onto 3D plane
      raycaster.setFromCamera(mouseNorm, camera);
      const intersects = raycaster.intersectObjects([raycastPlane]);
      if (intersects[0]) {
        simUniforms.uPointer.value = intersects[0].point.clone().sub(particlePoints.position);
      }

      if (pointerLagCounter <= 0) {
        simUniforms.uPointerStart.value = (simUniforms.uPointer.value as THREE.Vector3).clone();
        pointerLagCounter = 10;
      } else {
        pointerLagCounter -= 1;
      }

      renderer.render(mainScene, camera);
      animFrameId = requestAnimationFrame(renderLoop);
    }

    renderLoop();

    return () => {
      cancelAnimationFrame(animFrameId);
      window.removeEventListener('mousemove', onMouseMove);
      window.removeEventListener('mousedown', onMouseDown);
      window.removeEventListener('mouseup', onMouseUp);
      window.removeEventListener('touchmove', onTouchMove);
      window.removeEventListener('touchstart', onTouchStart);
      window.removeEventListener('touchend', onTouchEnd);
      window.removeEventListener('resize', onResize);
      document.removeEventListener('visibilitychange', onVisibilityChange);

      fboPos1.dispose();
      fboPos2.dispose();
      fboVel1.dispose();
      fboVel2.dispose();
      origPosTex.dispose();
      curPosTex.dispose();
      curVelTex.dispose();
      particleGeometry.dispose();
      pointsMaterial.dispose();
      quadGeo.dispose();
      simMaterial.dispose();
      raycastPlaneGeo.dispose();
      raycastPlaneMat.dispose();
      renderer.dispose();
    };
  });
</script>

<canvas
  bind:this={canvasEl}
  class="serotonin-background-canvas"
  aria-hidden="true"
></canvas>

<style>
  .serotonin-background-canvas {
    position: fixed;
    inset: 0;
    width: 100vw;
    height: 100vh;
    z-index: 0;
    pointer-events: none;
    transform: translate3d(0, 0, 0);
    will-change: transform;
    opacity: 0.75;
    transition: opacity 0.5s ease;
  }

  @media (prefers-reduced-motion: reduce) {
    .serotonin-background-canvas {
      display: none;
    }
  }
</style>
