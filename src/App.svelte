<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three'; // Use an alias
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';


  let container;
  let currentCellType = 'epithelial';
  let isSimulating = false;

  let scene, camera, renderer, controls, petriDishBase, petriDishWall;
  let originalCellPositions = {};
  let animationFrameIdSim;
  let animationFrameIdRender;
  let simulationStartTime;

  // UI State
  let apBioAccordionOpen = false;


  const cellInfo = {
    epithelial: {
      name: 'Frog Epithelial Cell',
      shortName: 'Epithelial',
      color: '#D2B48C',
      description: `
        <p><strong>Origin:</strong> Derived from the embryonic skin tissue (ectoderm) of the African Clawed Frog, <em>Xenopus laevis</em>.</p>
        <p><strong>Function in Xenobots:</strong> These cells provide passive structural support, forming the main body or scaffold of the Xenobot. They are like the "bricks" that give the Xenobot its shape.</p>
        <p><strong>AP Bio Concepts:</strong> Cell differentiation, tissue formation (epithelial tissue), cell adhesion, extracellular matrix (though simplified here).</p>
      `
    },
    cardiomyocyte: {
      name: 'Frog Cardiomyocyte',
      shortName: 'Cardiomyocyte',
      color: '#FFB6C1',
      description: `
        <p><strong>Origin:</strong> Derived from embryonic heart muscle tissue (mesoderm) of the African Clawed Frog, <em>Xenopus laevis</em>.</p>
        <p><strong>Function in Xenobots:</strong> These cells contract and relax rhythmically, providing the motive force for the Xenobot. Their coordinated contractions allow the Xenobot to move or manipulate its environment.</p>
        <p><strong>AP Bio Concepts:</strong> Cell differentiation, specialized cell function (muscle contraction), cytoskeleton (actin & myosin filaments), cellular respiration (ATP for contraction), emergent properties when cells work together.</p>
      `
    },
    delete: {
      name: 'Delete Cell',
      shortName: 'Delete',
      color: '#FF4136',
      description: `
        <p><strong>Function:</strong> Select this tool to remove cells from your Xenobot design.</p>
        <p><strong>How to use:</strong> Select "Delete" then click on any existing cell to remove it from the structure.</p>
      `
    }
  };

  let showInfoModal = false;
  let modalContent = { title: '', description: '' };

  function openInfoModal(cellTypeKey) {
    if (cellInfo[cellTypeKey]) {
      modalContent.title = cellInfo[cellTypeKey].name;
      modalContent.description = cellInfo[cellTypeKey].description;
      showInfoModal = true;
    }
  }

  function closeInfoModal() {
    showInfoModal = false;
  }

  function enterSimulationMode() {
    if (isSimulating || !scene) return;
    isSimulating = true;
    simulationStartTime = Date.now();

    if (petriDishBase) petriDishBase.visible = false;
    if (petriDishWall) petriDishWall.visible = false;

    originalCellPositions = {};
    const allCells = scene.children.filter(child =>
        child.geometry instanceof THREE.BoxGeometry && child.userData.isCell
    );

    if (allCells.length === 0) {
        isSimulating = false;
        if (petriDishBase) petriDishBase.visible = true;
        if (petriDishWall) petriDishWall.visible = true;
        return;
    }
    
    allCells.forEach(cell => {
        originalCellPositions[cell.uuid] = cell.position.clone();
    });

    const faceConnections = new Map();
    const cellSize = 0.2;
    const tolerance = 0.01;

    const faceDirections = [
        new THREE.Vector3(1, 0, 0), new THREE.Vector3(-1, 0, 0),
        new THREE.Vector3(0, 1, 0), new THREE.Vector3(0, -1, 0),
        new THREE.Vector3(0, 0, 1), new THREE.Vector3(0, 0, -1)
    ];

    allCells.forEach(cell => faceConnections.set(cell.uuid, []));

    allCells.forEach(cell1 => {
        const pos1 = originalCellPositions[cell1.uuid];
        allCells.forEach(cell2 => {
            if (cell1 === cell2) return;
            const pos2 = originalCellPositions[cell2.uuid];
            faceDirections.forEach(faceDir => {
                const expectedPos = new THREE.Vector3().copy(pos1).add(faceDir.clone().multiplyScalar(cellSize));
                if (expectedPos.distanceTo(pos2) < tolerance) {
                    faceConnections.get(cell1.uuid).push({
                        cell: cell2,
                        cellId: cell2.uuid,
                        direction: faceDir.clone(),
                    });
                }
            });
        });
    });

    function animateXenobot() {
        if (!isSimulating) return;

        const currentTime = Date.now();
        const elapsedTime = (currentTime - simulationStartTime) / 1000;
        const oscillationFactor = 0.6 + 0.4 * Math.sin(Math.PI * 1.5 * elapsedTime);

        allCells.forEach(cell => {
            if (cell.userData.cellType === 'cardiomyocyte') {
                cell.scale.set(oscillationFactor, oscillationFactor, oscillationFactor);
            } else {
                cell.scale.set(1, 1, 1);
            }
        });

        const MAX_ITERATIONS = 5;
        for (let iteration = 0; iteration < MAX_ITERATIONS; iteration++) {
            const processed = new Set();
            const baseDishTopY = 0; // Top of the petri dish base is at y=0
            const anchorCells = allCells.filter(cell => {
                const connections = faceConnections.get(cell.uuid) || [];
                const originalPos = originalCellPositions[cell.uuid];
                return connections.length === 0 || (originalPos && Math.abs(originalPos.y - (baseDishTopY + cellSize / 2)) < tolerance);
            });
            
            anchorCells.forEach(cell => processed.add(cell.uuid));
            
            const queue = [...anchorCells.map(b => b.uuid)];
            let head = 0;
            while (head < queue.length) {
                const currentCellId = queue[head++];
                const currentCell = allCells.find(b => b.uuid === currentCellId);
                if (!currentCell) continue;

                const connections = faceConnections.get(currentCellId) || [];
                connections.forEach(conn => {
                    const connectedCell = conn.cell;
                    if (!connectedCell || processed.has(connectedCell.uuid)) return;

                    const faceDir = conn.direction;
                    const currentCellScale = currentCell.scale.x;
                    const connectedCellScale = connectedCell.scale.x;
                    const adjustedDistance = (cellSize / 2 * currentCellScale) + (cellSize / 2 * connectedCellScale);

                    connectedCell.position.copy(currentCell.position).add(faceDir.clone().multiplyScalar(adjustedDistance));
                    
                    queue.push(connectedCell.uuid);
                    processed.add(connectedCell.uuid);
                });
            }

            const unprocessed = allCells.filter(cell => !processed.has(cell.uuid));
            while (unprocessed.length > 0) {
                const startCell = unprocessed.shift();
                if (!startCell) continue;

                const componentQueue = [startCell.uuid];
                processed.add(startCell.uuid);
                
                let compHead = 0;
                while(compHead < componentQueue.length) {
                    const currentId = componentQueue[compHead++];
                    const currentCellInComponent = allCells.find(b => b.uuid === currentId);
                    if (!currentCellInComponent) continue;

                    const indexInUnprocessed = unprocessed.findIndex(b => b.uuid === currentId);
                    if (indexInUnprocessed >= 0) unprocessed.splice(indexInUnprocessed, 1);

                    const connections = faceConnections.get(currentId) || [];
                    connections.forEach(conn => {
                        const connectedCell = conn.cell;
                        if (!connectedCell || processed.has(connectedCell.uuid)) return;

                        const faceDir = conn.direction;
                        const currentCellScale = currentCellInComponent.scale.x;
                        const connectedCellScale = connectedCell.scale.x;
                        const adjustedDistance = (cellSize / 2 * currentCellScale) + (cellSize / 2 * connectedCellScale);

                        connectedCell.position.copy(currentCellInComponent.position).add(faceDir.clone().multiplyScalar(adjustedDistance));
                        
                        componentQueue.push(connectedCell.uuid);
                        processed.add(connectedCell.uuid);
                    });
                }
            }
            verifyAndCorrectCellConnections(allCells, cellSize, faceConnections);
        }
        animationFrameIdSim = requestAnimationFrame(animateXenobot);
    }

    function verifyAndCorrectCellConnections(cells, baseCellSize, connectionsMap) {
        cells.forEach(cell1 => {
            const connections = connectionsMap.get(cell1.uuid) || [];
            connections.forEach(conn => {
                const cell2 = conn.cell;
                if (!cell2) return;
                const direction = conn.direction.clone();
                const expectedDistance = (baseCellSize / 2 * cell1.scale.x) + (baseCellSize / 2 * cell2.scale.x);
                const actualDirectionVector = new THREE.Vector3().subVectors(cell2.position, cell1.position);
                const actualDistance = actualDirectionVector.length();

                if (Math.abs(actualDistance - expectedDistance) > 0.001) {
                    cell2.position.copy(cell1.position).add(direction.multiplyScalar(expectedDistance));
                }
            });
        });
    }
    animateXenobot();
  }


  function exitSimulationMode() {
    if (!isSimulating) return;
    isSimulating = false;

    if (animationFrameIdSim) {
      cancelAnimationFrame(animationFrameIdSim);
      animationFrameIdSim = null;
    }

    if (petriDishBase) petriDishBase.visible = true;
    if (petriDishWall) petriDishWall.visible = true;

    const allCells = scene.children.filter(child => child.userData.isCell);
    allCells.forEach(cell => {
        if (originalCellPositions[cell.uuid]) {
            cell.position.copy(originalCellPositions[cell.uuid]);
            cell.scale.set(1, 1, 1);
        }
    });
    originalCellPositions = {};
  }

  onMount(() => {
    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
    renderer = new THREE.WebGLRenderer({ antialias: true });

    scene.background = new THREE.Color(0xe8f0fe);

    renderer.setSize(window.innerWidth, window.innerHeight);
    if (container) {
        container.appendChild(renderer.domElement);
    } else {
        console.error("Container for renderer not found on mount.");
        return;
    }

    const petriDishMaterial = new THREE.MeshPhysicalMaterial({
      color: 0xb0c4de, transparent: true, opacity: 0.35,
      roughness: 0.05, metalness: 0.2, ior: 1.4,
      clearcoat: 0.8, clearcoatRoughness: 0.1, side: THREE.DoubleSide,
    });

    const petriDishRadius = 2;
    const petriDishBaseHeight = 0.15; // Keep this, it's for visual thickness
    const petriDishWallHeight = 0.5;
    const petriDishSegments = 64; // Increased for smoother appearance

    // Position petriDishBase so its TOP surface is at y = 0
    petriDishBase = new THREE.Mesh(new THREE.CylinderGeometry(petriDishRadius, petriDishRadius, petriDishBaseHeight, petriDishSegments), petriDishMaterial);
    petriDishBase.position.y = -petriDishBaseHeight / 2; // This makes its top surface at y = 0
    scene.add(petriDishBase);
    
    // Position wall to sit on top of the base's new position
    petriDishWall = new THREE.Mesh(new THREE.CylinderGeometry(petriDishRadius, petriDishRadius, petriDishWallHeight, petriDishSegments, 1, true), petriDishMaterial);
    // Wall's bottom should be at y=0 (top of base), so its center is at y = wallHeight/2
    petriDishWall.position.y = petriDishWallHeight / 2;
    scene.add(petriDishWall);

    scene.add(new THREE.AmbientLight(0xffffff, 0.8)); // Slightly brighter ambient
    const dirLight1 = new THREE.DirectionalLight(0xffffff, 0.7); // Slightly stronger
    dirLight1.position.set(8, 15, 10);
    scene.add(dirLight1);
    const dirLight2 = new THREE.DirectionalLight(0xffffff, 0.5);
    dirLight2.position.set(-8, 10, -10); // Adjusted y for fill
    scene.add(dirLight2);

    camera.position.set(4, 4, 4);

    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.15;
    controls.target.set(0, 0.5, 0); // Target slightly above dish surface

    const cellMaterials = {
      epithelial: new THREE.MeshStandardMaterial({ color: cellInfo.epithelial.color, roughness: 0.6, metalness: 0.1 }),
      cardiomyocyte: new THREE.MeshStandardMaterial({ color: cellInfo.cardiomyocyte.color, roughness: 0.5, metalness: 0.1 }),
      delete: new THREE.MeshStandardMaterial({ color: cellInfo.delete.color, roughness: 0.5, metalness: 0.1 })
    };

    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();
    const cellSize = 0.2;

    function cellExistsAt(position) {
      const tolerance = cellSize * 0.49; // Reduce tolerance slightly from 0.5
      return scene.children.some(child => {
        if (child.geometry instanceof THREE.BoxGeometry && child.userData.isCell) {
          return child.position.distanceTo(position) < tolerance;
        }
        return false;
      });
    }

    // In the onWindowClick event handler within onMount:
    const onWindowClick = (event) => {
      if (isSimulating || showInfoModal) {
        if (showInfoModal && event.target.classList.contains('modal-overlay')) {
          closeInfoModal();
        }
        return;
      }

      if (event.target.closest('.controls-panel')) return;

      mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

      raycaster.setFromCamera(mouse, camera);
      
      // For delete mode, we only want to check intersections with cells
      let intersectableObjects;
      if (currentCellType === 'delete') {
        intersectableObjects = scene.children.filter(c => c.userData.isCell && c.visible);
      } else {
        intersectableObjects = scene.children.filter(c => c === petriDishBase || (c.userData.isCell && c.visible));
      }
      
      const intersects = raycaster.intersectObjects(intersectableObjects, false);

      if (intersects.length > 0) {
        const intersect = intersects[0];
        
        // Handle delete action
        if (currentCellType === 'delete') {
          if (intersect.object.userData.isCell) {
            // Remove the cell from the scene
            scene.remove(intersect.object);
            return;
          }
        } else {
          // Original cell placement logic
          const newCellPosition = new THREE.Vector3();
          const normalMatrix = new THREE.Matrix3().getNormalMatrix(intersect.object.matrixWorld);
          const worldNormal = intersect.face.normal.clone().applyMatrix3(normalMatrix).normalize();
          
          if (intersect.object === petriDishBase) {
            // Placement on petri dish
            newCellPosition.copy(intersect.point);
            newCellPosition.y = cellSize/2; // Center Y position
            newCellPosition.x = Math.round(newCellPosition.x / cellSize) * cellSize;
            newCellPosition.z = Math.round(newCellPosition.z / cellSize) * cellSize;
          } else {
            // Placement on existing cell
            newCellPosition.copy(intersect.object.position);
            newCellPosition.add(worldNormal.multiplyScalar(cellSize));
          }

          // Remove Y-axis grid adjustments and only maintain valid Y positions
          newCellPosition.y = Math.max(cellSize/2, Math.round(newCellPosition.y / (cellSize/2)) * (cellSize/2));

          if (!cellExistsAt(newCellPosition)) {
            const cellGeometry = new THREE.BoxGeometry(cellSize, cellSize, cellSize);
            const cell = new THREE.Mesh(cellGeometry, cellMaterials[currentCellType]);
            cell.position.copy(newCellPosition);
            cell.userData = { isCell: true, cellType: currentCellType };
            scene.add(cell);
          }
        }
      }
    };
    
    window.addEventListener('click', onWindowClick);

    function render() {
      animationFrameIdRender = requestAnimationFrame(render);
      if (controls) controls.update();
      if (renderer && scene && camera) renderer.render(scene, camera);
    }
    render();

    const handleResize = () => { /* ... */ };
    window.addEventListener('resize', handleResize);
    const handleKeyDown = (event) => { /* ... */ };
    window.addEventListener('keydown', handleKeyDown);

    return () => { /* ... */ };
  });

</script>

<div class="app-container">
  <div bind:this={container} class="renderer-canvas"></div>

  <div class="title-bar">
    <h1>Xenobot Design Lab</h1>
    <p class="subtitle">Biology Today: Computerized Xenobot Creation</p>
  </div>

  <div class="controls-panel">
    <div class="panel-section">
      <h2>Build Your Xenobot</h2>
      <p class="instructions">
        Select a cell type, then click on the petri dish or an existing cell to place new cells.
        Use mouse to rotate, scroll to zoom, right-click & drag to pan.
        Select "Delete" to remove cells.
      </p>
    </div>

    <div class="panel-section cell-selector">
      <h3>Cell Types:</h3>
      <div class="cell-option">
        <button
          class="cell-button"
          class:selected={currentCellType === 'epithelial'}
          style="background-color: {cellInfo.epithelial.color}; color: white;"
          on:click={() => currentCellType = 'epithelial'}>
          {cellInfo.epithelial.shortName}
        </button>
        <button class="info-button" on:click={() => openInfoModal('epithelial')} title="Learn about Epithelial Cells">?</button>
      </div>
      <div class="cell-option">
        <button
          class="cell-button"
          class:selected={currentCellType === 'cardiomyocyte'}
          style="background-color: {cellInfo.cardiomyocyte.color}; color: white;"
          on:click={() => currentCellType = 'cardiomyocyte'}>
          {cellInfo.cardiomyocyte.shortName}
        </button>
        <button class="info-button" on:click={() => openInfoModal('cardiomyocyte')} title="Learn about Cardiomyocytes">?</button>
      </div>
      <div class="cell-option">
        <button
          class="cell-button"
          class:selected={currentCellType === 'delete'}
          style="background-color: {cellInfo.delete.color}; color: white;"
          on:click={() => currentCellType = 'delete'}>
          {cellInfo.delete.shortName}
        </button>
        <button class="info-button" on:click={() => openInfoModal('delete')} title="Learn about Delete Tool">?</button>
      </div>
      <p class="current-selection">Selected: <strong>{cellInfo[currentCellType].name}</strong></p>
    </div>

    <div class="panel-section simulation-controls">
      <h3>Simulation:</h3>
      <button class="action-button" on:click={() => isSimulating ? exitSimulationMode() : enterSimulationMode()}>
        {isSimulating ? 'Deactivate Xenobot (Space)' : 'Activate Xenobot (Space)'}
      </button>
      </div>

     <div class="panel-section ap-bio-info">
      <h3 class="accordion-header" on:click={() => apBioAccordionOpen = !apBioAccordionOpen}>
        AP Biology Connections <span class="accordion-icon">{apBioAccordionOpen ? '▼' : '►'}</span>
      </h3>
      {#if apBioAccordionOpen}
        <ul>
          <li><strong>Cellular Basis of Life:</strong> Xenobots are built from specialized frog cells.</li>
          <li><strong>Cell Differentiation:</strong> Epithelial (structure) and Cardiomyocyte (movement) cells have distinct functions.</li>
          <li><strong>Emergent Properties:</strong> Complex behaviors (like movement) arise from the interactions of these cells – the whole is greater than the sum of its parts.</li>
          <li><strong>Biotechnology & Synthetic Biology:</strong> This demonstrates creating novel biological systems.</li>
        </ul>
      {/if}
    </div>
  </div>

  {#if showInfoModal}
    <div class="modal-overlay" on:click|self={closeInfoModal}>
      <div class="modal-content">
        <button class="modal-close-button" on:click={closeInfoModal} aria-label="Close modal">&times;</button>
        <h3>{modalContent.title}</h3>
        {@html modalContent.description}
      </div>
    </div>
  {/if}
</div>
<style>
  :global(body) {
    margin: 0;
    overflow: hidden;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    color: #333;
    background-color: #f0f4f8;
  }

  .app-container {
    position: relative;
    width: 100vw;
    height: 100vh;
  }

  .renderer-canvas {
    width: 100%;
    height: 100%;
    display: block;
  }

  .title-bar {
    position: absolute;
    top: 15px;
    left: 50%;
    transform: translateX(-50%);
    background-color: rgba(255, 255, 255, 0.9);
    padding: 10px 25px;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    text-align: center;
    z-index: 10;
  }

  .title-bar h1 {
    margin: 0 0 5px 0;
    font-size: 1.6em;
    color: #2c3e50;
  }
  .title-bar .subtitle {
    margin: 0;
    font-size: 0.9em;
    color: #555;
    font-style: italic;
  }

  .controls-panel {
    position: absolute;
    top: 20px;
    left: 20px;
    width: 330px;
    max-height: calc(100vh - 40px);
    overflow-y: auto;
    background-color: rgba(255, 255, 255, 0.95);
    padding: 20px;
    border-radius: 10px;
    box-shadow: 0 4px 15px rgba(0,0,0,0.15);
    font-family: 'Roboto', 'Helvetica Neue', Arial, sans-serif;
    z-index: 10;
    color: #333;
  }
  
  .panel-section {
    margin-bottom: 20px;
    padding-bottom: 15px;
    border-bottom: 1px solid #e0e0e0;
  }
  .panel-section:last-child {
    border-bottom: none;
    margin-bottom: 0;
  }

  .panel-section h2 {
    font-size: 1.3em;
    margin-top: 0;
    margin-bottom: 10px;
    color: #3498db;
  }
  .panel-section h3 {
    font-size: 1.1em;
    margin-top: 0;
    margin-bottom: 8px;
    color: #2980b9;
  }
  .instructions, .current-selection {
    font-size: 0.9em;
    color: #555;
    line-height: 1.4;
  }
  .current-selection strong {
    color: #2c3e50;
  }

  .cell-selector .cell-option {
    display: flex;
    align-items: center; /* Vertically align items */
    margin-bottom: 10px;
  }

  .cell-button { /* Renamed from .cell-selector button for specificity */
    padding: 8px 12px; /* Adjusted padding */
    border: 2px solid transparent;
    border-radius: 6px;
    cursor: pointer;
    font-size: 0.95em;
    font-weight: 500;
    transition: all 0.2s ease;
    margin-right: 8px;
    min-width: 120px;
    text-align: center;
    height: 38px; /* Explicit height */
    box-sizing: border-box; /* Include padding and border in height */
  }

  
  .cell-button[style*="rgb(210, 180, 140)"]{
    background-color: #b89a74 !important;
  }
  .cell-button[style*="rgb(255, 182, 193)"]{
    background-color: #f08080 !important;
  }
  .cell-button.selected {
	font-weight: bold;
	box-shadow: 0 0 5px rgba(0, 0, 0, 0.2);
  }

  .info-button {
    background-color: #d4d3cf;
    color: #2c3e50;
    border: 0px solid #bdc3c7;
    border-radius: 4px; /* Square-ish */
    width: 38px;  /* Match height of cell-button */
    height: 38px; /* Match height of cell-button */
    padding: 0;
    font-size: 1.2em; /* Slightly larger '?' */
    font-weight: bold;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    line-height: 1;
    flex-shrink: 0;
    box-sizing: border-box; /* Include padding and border in height */
  }
  .info-button:hover {
    background-color: #cad1d3; /* Slightly different hover */
    border-color: #a2a9ac;
  }

  .action-button {
    background-color: #2ecc71;
    color: white;
    padding: 12px 18px;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-size: 1em;
    font-weight: 500;
    width: 100%;
    transition: background-color 0.2s ease;
    margin-bottom: 10px;
  }
  .action-button:hover {
    background-color: #27ae60;
  }

  .ap-bio-info .accordion-header {
    cursor: pointer;
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 5px 0; /* Add some padding */
  }
  .ap-bio-info .accordion-header:hover {
    color: #3498db;
  }
  .ap-bio-info .accordion-icon {
    font-size: 0.8em;
    transition: transform 0.2s ease-in-out;
  }
  /* Consider rotating icon if you want: 
    .ap-bio-info .accordion-header.open .accordion-icon { transform: rotate(90deg); } 
  */

  .ap-bio-info ul {
    list-style-type: disc;
    padding-left: 20px;
    font-size: 0.85em;
    line-height: 1.5;
    color: #444;
    margin-top: 8px; /* Space below header */
  }
  .ap-bio-info strong {
    color: #2980b9;
  }

  /* Modal styles (mostly unchanged) */
  .modal-overlay {
    position: fixed; top: 0; left: 0; width: 100%; height: 100%;
    background-color: rgba(0, 0, 0, 0.6); display: flex;
    align-items: center; justify-content: center; z-index: 1000;
    backdrop-filter: blur(3px);
  }
  .modal-content {
    background-color: #fff; padding: 25px 30px; border-radius: 10px;
    box-shadow: 0 5px 20px rgba(0,0,0,0.2); width: 90%; max-width: 550px;
    position: relative; color: #333; line-height: 1.6;
  }
  .modal-content h3 { margin-top: 0; color: #3498db; font-size: 1.5em; }
  .modal-content p { font-size: 0.95em; margin-bottom: 12px; }
  .modal-content strong { color: #2c3e50; }
  .modal-close-button {
    position: absolute; top: 10px; right: 15px; background: none;
    border: none; font-size: 2em; color: #aaa; cursor: pointer; line-height: 1;
  }
  .modal-close-button:hover { color: #777; }

</style>
