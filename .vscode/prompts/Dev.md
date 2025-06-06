# Instructions

I want you to create a fully playable game in a Canvas/Artifact using this template as a starting point. This template will include the essential setup for a 3D scene, camera, renderer, basic lighting, and a simple object, along with an animation loop.

# Requirements

1. The entire game must fit in a single file.
2. You must use `Three.js` within a `!DOCTYPE html` file.
3. Use WASD for movement.
4. Do not download any assets or textures.
5. Install required libraries via CDN.

# Gameplay Specifications

I want you to recreate this Threejs car driving game, but instead of threejs, you must use PlayCanvas WebGL engine via CDN.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Simple 3D Car Driving Game - Grid World + Drift + Nitro</title>
  <style>
    body {
      margin: 0;
      overflow: hidden; /* Prevent scrollbars */
      font-family: sans-serif;
      background-color: #000; /* Ensure no white flash */
    }
    canvas {
      display: block; /* Remove default inline spacing */
      width: 100%;   /* Make canvas fill width */
      height: 100%;  /* Make canvas fill height */
    }
    #info {
      position: absolute;
      top: 10px;
      left: 10px;
      background: rgba(0,0,0,0.7);
      color: white;
      padding: 8px 12px;
      border-radius: 5px;
      font-size: 14px;
      z-index: 10; /* Ensure info is above the canvas */
    }
     #speedometer {
        position: absolute;
        bottom: 10px;
        left: 10px;
        background: rgba(0,0,0,0.7);
        color: white;
        padding: 8px 12px;
        border-radius: 5px;
        font-size: 16px;
        z-index: 10; /* Ensure speedometer is above the canvas */
        min-width: 100px; /* Give it some base width */
        text-align: center;
     }
     /* --- Nitro Bar Styling --- */
     #nitro-ui {
        position: absolute;
        bottom: 10px;
        right: 10px; /* Position on the right */
        background: rgba(0,0,0,0.7);
        color: white;
        padding: 8px 12px;
        border-radius: 5px;
        font-size: 14px;
        z-index: 10;
        width: 150px; /* Width of the Nitro UI container */
        text-align: left;
     }
     #nitro-bar-container {
        width: 100%;
        height: 10px;
        background-color: #555; /* Dark background for the bar */
        border-radius: 3px;
        overflow: hidden; /* Hide overflow for the inner bar */
        margin-top: 4px;
     }
     #nitro-bar {
        width: 100%; /* Start full */
        height: 100%;
        background-color: #00ccff; /* Bright blue for Nitro */
        border-radius: 3px;
        transition: width 0.1s linear; /* Smooth transition for bar width */
     }
  </style>
</head>
<body>
  <div id="info">WASD: Drive | Space: Drift | LShift: Nitro</div> <div id="speedometer">Speed: 0</div>
  <div id="nitro-ui">
      Nitro:
      <div id="nitro-bar-container">
          <div id="nitro-bar"></div>
      </div>
  </div>

  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

  <script>
    // --- Global Variables ---
    let scene, camera, renderer, car;
    let clock = new THREE.Clock();
    let speedometerElement, nitroBarElement; // UI Elements
    let carBodyMesh;

    // Movement state object
    const keysPressed = {
        w: false, a: false, s: false, d: false,
        ' ': false, // Space for Drift
        shift: false // Use generic 'shift' for either Shift key
    };

    // --- World Parameters ---
    const worldSize = 1000;
    const roadWidth = 10;
    const roadSegments = 10;
    const roadSpacing = worldSize / roadSegments;

    // --- Car Physics Parameters ---
    let currentSpeed = 0;
    const baseMaxSpeed = 100; // Renamed for clarity
    const maxReverseSpeed = -15;
    const baseAcceleration = 15; // Renamed for clarity
    const deceleration = 20;
    const brakeDeceleration = 35;
    const baseTurnSpeed = Math.PI * 0.8; // Renamed for clarity
    const turnSpeedFactor = 0.5;

    // --- Drift Parameters ---
    let isDrifting = false;
    const driftTurnMultiplier = 1.8;
    const driftTiltAngle = Math.PI / 18;
    let currentTilt = 0;
    const tiltSpeed = Math.PI / 4;

    // --- Nitro Parameters ---
    let isBoosting = false;
    const maxNitroAmount = 100;
    let nitroAmount = maxNitroAmount; // Start with full Nitro
    const nitroConsumptionRate = 25; // Units per second
    const nitroRechargeRate = 8; // Units per second when not boosting
    const nitroAccelerationMultiplier = 3.0; // Boost acceleration significantly
    const nitroMaxSpeedMultiplier = 1.6; // Boost max speed
    const nitroTurnReduction = 0.4; // Reduce turning ability significantly during boost
    const baseCameraFOV = 75; // Store default FOV
    const boostCameraFOV = 85; // Increased FOV during boost
    let currentCameraFOV = baseCameraFOV; // For smooth FOV transition
    const fovChangeSpeed = 30; // Speed of FOV change (degrees per second)

    // Camera settings
    const cameraOffset = new THREE.Vector3(0, 8, -15);

    // --- Initialization Function ---
    function init() {
      // Get UI elements
      speedometerElement = document.getElementById('speedometer');
      nitroBarElement = document.getElementById('nitro-bar'); // Get the inner bar element

      // Scene setup
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x87CEEB);
      scene.fog = new THREE.Fog(0x87CEEB, worldSize * 0.2, worldSize * 0.8);

      // Camera setup
      camera = new THREE.PerspectiveCamera(
        baseCameraFOV, // Use base FOV initially
        window.innerWidth / window.innerHeight,
        0.1,
        worldSize * 1.5
      );
      camera.position.set(0, 10, 15);
      camera.lookAt(0, 0, 0);

      // Renderer setup
      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.shadowMap.enabled = true;
      renderer.shadowMap.type = THREE.PCFSoftShadowMap;
      document.body.appendChild(renderer.domElement);

      // Lighting setup
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
      scene.add(ambientLight);
      const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
      directionalLight.position.set(worldSize * 0.2, worldSize * 0.3, worldSize * 0.15);
      directionalLight.castShadow = true;
      directionalLight.shadow.mapSize.width = 1024;
      directionalLight.shadow.mapSize.height = 1024;
      const shadowCamSize = 100;
      directionalLight.shadow.camera.near = 0.5;
      directionalLight.shadow.camera.far = worldSize * 0.5;
      directionalLight.shadow.camera.left = -shadowCamSize;
      directionalLight.shadow.camera.right = shadowCamSize;
      directionalLight.shadow.camera.top = shadowCamSize;
      directionalLight.shadow.camera.bottom = -shadowCamSize;
      scene.add(directionalLight);

      // Ground plane setup
      const groundGeometry = new THREE.PlaneGeometry(worldSize, worldSize);
      const groundMaterial = new THREE.MeshStandardMaterial({
          color: 0x556B2F, roughness: 0.9, metalness: 0.1 });
      const ground = new THREE.Mesh(groundGeometry, groundMaterial);
      ground.rotation.x = -Math.PI / 2;
      ground.receiveShadow = true;
      scene.add(ground);

      // Create the Road Grid
      createRoadGrid();

      // Create the Car model
      createCar();

      // Create Scenery elements
      createScenery();

      // Event Listeners
      window.addEventListener('keydown', handleKeyDown);
      window.addEventListener('keyup', handleKeyUp);
      window.addEventListener('resize', onWindowResize);

      // Start the main animation loop
      window.onload = function() {
           animate();
      }
    }

    // --- Object Creation Functions --- (createRoadGrid, createCar, createScenery remain largely unchanged)

    function createRoadGrid() { // Unchanged
        const roadMaterial = new THREE.MeshStandardMaterial({ color: 0x444444, roughness: 0.8 });
        const markingMaterial = new THREE.MeshStandardMaterial({ color: 0xFFFFFF });
        const markingGeometry = new THREE.BoxGeometry(0.2, 0.02, 2);
        const markingSpacing = 5;
        for (let i = 0; i <= roadSegments; i++) {
            const positionOffset = -worldSize / 2 + i * roadSpacing;
            const vRoadGeometry = new THREE.PlaneGeometry(roadWidth, worldSize);
            const vRoad = new THREE.Mesh(vRoadGeometry, roadMaterial);
            vRoad.rotation.x = -Math.PI / 2; vRoad.position.set(positionOffset, 0.01, 0); vRoad.receiveShadow = true; scene.add(vRoad);
            const hRoadGeometry = new THREE.PlaneGeometry(worldSize, roadWidth);
            const hRoad = new THREE.Mesh(hRoadGeometry, roadMaterial);
            hRoad.rotation.x = -Math.PI / 2; hRoad.position.set(0, 0.01, positionOffset); hRoad.receiveShadow = true; scene.add(hRoad);
            for (let z = -worldSize / 2 + roadSpacing / 2; z < worldSize / 2 - roadSpacing / 2; z += roadSpacing) {
                for (let dashZ = z - roadSpacing / 2 + markingSpacing / 2; dashZ < z + roadSpacing / 2; dashZ += markingSpacing) {
                     if (Math.abs(dashZ) < worldSize / 2 - markingSpacing / 2) { const vMarking = new THREE.Mesh(markingGeometry, markingMaterial); vMarking.position.set(positionOffset, 0.02, dashZ); vMarking.receiveShadow = true; scene.add(vMarking); }
                } }
            for (let x = -worldSize / 2 + roadSpacing / 2; x < worldSize / 2 - roadSpacing / 2; x += roadSpacing) {
                 for (let dashX = x - roadSpacing / 2 + markingSpacing / 2; dashX < x + roadSpacing / 2; dashX += markingSpacing) {
                    if (Math.abs(dashX) < worldSize / 2 - markingSpacing / 2) { const hMarking = new THREE.Mesh(markingGeometry, markingMaterial); hMarking.rotation.y = Math.PI / 2; hMarking.position.set(dashX, 0.02, positionOffset); hMarking.receiveShadow = true; scene.add(hMarking); }
                 } } }
    }

    function createCar() { // Unchanged
      car = new THREE.Group(); car.position.set(0, 0.5, 0);
      const bodyGeometry = new THREE.BoxGeometry(2, 1, 4);
      const bodyMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000, roughness: 0.5, metalness: 0.3 });
      carBodyMesh = new THREE.Mesh(bodyGeometry, bodyMaterial); carBodyMesh.castShadow = true; carBodyMesh.receiveShadow = true; car.add(carBodyMesh);
      const cabinGeometry = new THREE.BoxGeometry(1.8, 0.8, 2);
      const cabinMaterial = new THREE.MeshStandardMaterial({ color: 0xaaaaaa, roughness: 0.6, metalness: 0.2 });
      const carCabin = new THREE.Mesh(cabinGeometry, cabinMaterial); carCabin.position.set(0, 0.9, -0.5); carCabin.castShadow = true; carCabin.receiveShadow = true; carBodyMesh.add(carCabin);
      const wheelGeometry = new THREE.CylinderGeometry(0.4, 0.4, 0.5, 16);
      const wheelMaterial = new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.8 });
      const wheelPositions = [ { x: 1.1, y: -0.1, z: 1.3 }, { x: -1.1, y: -0.1, z: 1.3 }, { x: 1.1, y: -0.1, z: -1.3 }, { x: -1.1, y: -0.1, z: -1.3 } ];
      wheelPositions.forEach(pos => { const wheel = new THREE.Mesh(wheelGeometry, wheelMaterial); wheel.position.set(pos.x, pos.y, pos.z); wheel.rotation.z = Math.PI / 2; wheel.castShadow = true; car.add(wheel); });
      scene.add(car);
    }

    function createScenery() { // Unchanged
        const buildingMaterial = new THREE.MeshStandardMaterial({ color: 0xcccccc, roughness: 0.8, metalness: 0.1 }); const trunkMaterial = new THREE.MeshStandardMaterial({ color: 0x8B4513, roughness: 0.9 }); const leavesMaterial = new THREE.MeshStandardMaterial({ color: 0x228B22, roughness: 0.8 }); const trunkGeom = new THREE.CylinderGeometry(0.3, 0.4, 3, 8); const leavesGeom = new THREE.ConeGeometry(1.5, 4, 8); const numBuildings = 150; const numTrees = 250; const halfWorld = worldSize / 2; const halfRoadWidth = roadWidth / 2; const blockInnerSize = roadSpacing - roadWidth;
        function isOnRoad(x, z) { for (let i = 0; i <= roadSegments; i++) { const roadCenter = -halfWorld + i * roadSpacing; if (Math.abs(x - roadCenter) < halfRoadWidth || Math.abs(z - roadCenter) < halfRoadWidth) { return true; } } return false; }
        for (let i = 0; i < numBuildings; i++) { let placed = false; while (!placed) { const blockX = Math.floor(Math.random() * roadSegments); const blockZ = Math.floor(Math.random() * roadSegments); const blockCenterX = -halfWorld + blockX * roadSpacing + roadSpacing / 2; const blockCenterZ = -halfWorld + blockZ * roadSpacing + roadSpacing / 2; const x = blockCenterX + (Math.random() - 0.5) * blockInnerSize; const z = blockCenterZ + (Math.random() - 0.5) * blockInnerSize; if (!isOnRoad(x, z)) { const h = Math.random() * 20 + 10; const w = Math.random() * 8 + 4; const d = Math.random() * 8 + 4; const buildingGeom = new THREE.BoxGeometry(w, h, d); const building = new THREE.Mesh(buildingGeom, buildingMaterial); building.position.set(x, h / 2, z); building.castShadow = true; building.receiveShadow = true; scene.add(building); placed = true; } } }
        for (let i = 0; i < numTrees; i++) { let placed = false; while (!placed) { const blockX = Math.floor(Math.random() * roadSegments); const blockZ = Math.floor(Math.random() * roadSegments); const blockCenterX = -halfWorld + blockX * roadSpacing + roadSpacing / 2; const blockCenterZ = -halfWorld + blockZ * roadSpacing + roadSpacing / 2; const x = blockCenterX + (Math.random() - 0.5) * blockInnerSize; const z = blockCenterZ + (Math.random() - 0.5) * blockInnerSize; if (!isOnRoad(x, z)) { const tree = new THREE.Group(); const trunk = new THREE.Mesh(trunkGeom, trunkMaterial); trunk.position.y = 1.5; trunk.castShadow = true; trunk.receiveShadow = true; tree.add(trunk); const leaves = new THREE.Mesh(leavesGeom, leavesMaterial); leaves.position.y = 3 + 2; leaves.castShadow = true; leaves.receiveShadow = true; tree.add(leaves); tree.position.set(x, 0, z); scene.add(tree); placed = true; } } }
    }

    // --- Event Handlers --- (Updated for Shift)

    function handleKeyDown(event) {
      const key = event.key.toLowerCase();
      // Use event.code for specific keys like ShiftLeft/ShiftRight if needed
      // For simplicity, treat any Shift as 'shift'
      if (key === 'shift') {
          keysPressed['shift'] = true;
      } else if (key in keysPressed) {
        keysPressed[key] = true;
      }
    }

    function handleKeyUp(event) {
      const key = event.key.toLowerCase();
       if (key === 'shift') {
          keysPressed['shift'] = false;
      } else if (key in keysPressed) {
        keysPressed[key] = false;
      }
    }

    function onWindowResize() {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    }

    // --- Animation Loop --- (Modified for Nitro)
    function animate() {
      requestAnimationFrame(animate);

      const deltaTime = clock.getDelta();

      // --- Nitro Management ---
      // Recharge Nitro if not boosting and not full
      if (!isBoosting && nitroAmount < maxNitroAmount) {
          nitroAmount += nitroRechargeRate * deltaTime;
          nitroAmount = Math.min(nitroAmount, maxNitroAmount); // Clamp at max
      }

      // Check for Boost Activation
      if (keysPressed['shift'] && nitroAmount > 0) {
          isBoosting = true;
          // Consume Nitro
          nitroAmount -= nitroConsumptionRate * deltaTime;
          nitroAmount = Math.max(nitroAmount, 0); // Clamp at min
          if (nitroAmount === 0) {
              isBoosting = false; // Stop boosting if Nitro runs out
          }
      } else {
          isBoosting = false;
      }

      // --- Update Drift State --- (Deactivate drift if boosting - optional choice)
      // if (isBoosting) {
      //     isDrifting = false; // Prevent drifting while boosting
      // } else if (keysPressed[' '] && currentSpeed > baseMaxSpeed * 0.3) {
      //     isDrifting = true;
      // } else {
      //     isDrifting = false;
      // }
      // --- OR Allow drift and boost simultaneously (current implementation) ---
       if (keysPressed[' '] && currentSpeed > baseMaxSpeed * 0.3 && !isBoosting) { // Only allow drift if NOT boosting
           isDrifting = true;
       } else {
           isDrifting = false;
       }


      // --- Define current physics parameters based on boost state ---
      let currentAcceleration = baseAcceleration;
      let currentMaxSpeed = baseMaxSpeed;
      let currentTurnSpeed = baseTurnSpeed;

      if (isBoosting) {
          currentAcceleration *= nitroAccelerationMultiplier;
          currentMaxSpeed *= nitroMaxSpeedMultiplier;
          currentTurnSpeed *= nitroTurnReduction; // Reduce turning ability
      }

      // --- Update Car Speed ---
      let appliedAcceleration = 0;
      if (keysPressed.w) {
          appliedAcceleration = currentAcceleration; // Use potentially boosted acceleration
      } else if (keysPressed.s) {
          // Braking/Reverse logic remains the same relative to base speed/deceleration
          if (currentSpeed > 0.1) { appliedAcceleration = -brakeDeceleration; }
          else { appliedAcceleration = -baseAcceleration; } // Use base acceleration for reverse
      } else {
          // Natural deceleration
          if (Math.abs(currentSpeed) > 0.1) { appliedAcceleration = -Math.sign(currentSpeed) * deceleration; }
          else { currentSpeed = 0; }
      }
      currentSpeed += appliedAcceleration * deltaTime;
      // Clamp speed using potentially boosted max speed
      currentSpeed = Math.max(maxReverseSpeed, Math.min(currentMaxSpeed, currentSpeed));

       if (!keysPressed.w && !keysPressed.s && Math.abs(currentSpeed) < 0.1) {
           currentSpeed = 0;
       }

      // --- Update Car Position and Rotation (With Drift/Boost Logic) ---
      const moveDistance = currentSpeed * deltaTime;
      const forward = new THREE.Vector3();
      car.getWorldDirection(forward);
      const right = new THREE.Vector3();
      right.crossVectors(car.up, forward).normalize();

      // Turning
      if (Math.abs(currentSpeed) > 0.1) {
          // Use the potentially boost-reduced turn speed
          let actualTurnRate = currentTurnSpeed * Math.pow(Math.abs(currentSpeed / currentMaxSpeed), turnSpeedFactor);
          let turnDirection = 0;

          if (keysPressed.a) turnDirection = 1;
          if (keysPressed.d) turnDirection = -1;

          // Apply drift modifications if drifting (and not boosting)
          if (isDrifting && turnDirection !== 0) {
              actualTurnRate *= driftTurnMultiplier; // Turn faster for drift
              const slideForce = right.clone().multiplyScalar(currentSpeed * 0.1 * -turnDirection * deltaTime);
              car.position.add(slideForce);
          }

          const turnAngle = actualTurnRate * turnDirection * deltaTime;
          car.rotation.y += turnAngle;
      }

      // Move the car forward
      car.position.addScaledVector(forward, moveDistance);


      // --- Visual Drift Tilt ---
      let targetTilt = 0;
      // Apply tilt only if drifting (and not boosting, based on current logic)
      if (isDrifting && (keysPressed.a || keysPressed.d)) {
          targetTilt = keysPressed.a ? driftTiltAngle : -driftTiltAngle;
      }
      const tiltDelta = targetTilt - currentTilt;
      const tiltStep = Math.sign(tiltDelta) * Math.min(Math.abs(tiltDelta), tiltSpeed * deltaTime);
      currentTilt += tiltStep;
      if (carBodyMesh) { carBodyMesh.rotation.z = currentTilt; }


      // --- World Boundaries Wrap ---
      const halfWorld = worldSize / 2;
      if (car.position.x > halfWorld) car.position.x = -halfWorld;
      if (car.position.x < -halfWorld) car.position.x = halfWorld;
      if (car.position.z > halfWorld) car.position.z = -halfWorld;
      if (car.position.z < -halfWorld) car.position.z = halfWorld;


      // --- Update Camera ---
      // Smoothly adjust FOV based on boost state
      let targetFOV = isBoosting ? boostCameraFOV : baseCameraFOV;
      const fovDelta = targetFOV - currentCameraFOV;
      const fovStep = Math.sign(fovDelta) * Math.min(Math.abs(fovDelta), fovChangeSpeed * deltaTime);
      currentCameraFOV += fovStep;
      if (camera.fov !== currentCameraFOV) {
          camera.fov = currentCameraFOV;
          camera.updateProjectionMatrix(); // IMPORTANT: Update projection matrix after FOV change
      }

      // Update camera position and lookAt (lerp for smoothness)
      const targetCameraPosition = new THREE.Vector3();
      targetCameraPosition.copy(cameraOffset);
      targetCameraPosition.applyQuaternion(car.quaternion);
      targetCameraPosition.add(car.position);
      camera.position.lerp(targetCameraPosition, 0.1); // Keep lerp for smooth follow

      const lookAtTarget = new THREE.Vector3();
      lookAtTarget.copy(car.position).addScaledVector(forward, 5.0);
      lookAtTarget.y += 1.0;
      camera.lookAt(lookAtTarget);

      // --- Update UI ---
      // Speedometer
      if (speedometerElement) {
          speedometerElement.innerText = `Speed: ${currentSpeed.toFixed(1)}`;
      }
      // Nitro Bar
      if (nitroBarElement) {
          const nitroPercent = (nitroAmount / maxNitroAmount) * 100;
          nitroBarElement.style.width = `${nitroPercent}%`;
      }

      // Render the scene
      renderer.render(scene, camera);
    }

    // --- Start ---
    if (typeof THREE === 'undefined') {
        console.error('Three.js not loaded!');
        document.getElementById('info').innerText = 'Error: Three.js failed to load.';
    } else {
         if (document.readyState === 'loading') {
            document.addEventListener('DOMContentLoaded', init);
        } else {
            init();
        }
    }

  </script>
</body>
</html>
```