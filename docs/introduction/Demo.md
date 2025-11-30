
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Warehouse Safety & Hazard Identification (A‑Frame)</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <!-- A-Frame core -->
  https://aframe.io/releases/1.5.0/aframe.min.js</script>
  <!-- Optional: cursor components for better UX -->
  https://cdn.jsdelivr.net/gh/supermedium/aframe-environment-component/dist/aframe-environment-component.min.js</script>
  <style>
    body { margin: 0; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; }
    /* Simple HUD */
    #hud {
      position: absolute; left: 12px; top: 12px; z-index: 10;
      background: rgba(0,0,0,0.55); color: #fff; padding: 10px 12px; border-radius: 8px;
      max-width: 360px; font-size: 14px; line-height: 1.35;
    }
    #hud strong { color: #ffd166; }
    #controls {
      position: absolute; right: 12px; top: 12px; z-index: 10;
      display: flex; gap: 8px; flex-wrap: wrap;
    }
    .btn {
      background: #1f6feb; color: white; border: none; border-radius: 6px;
      padding: 8px 10px; font-size: 13px; cursor: pointer;
    }
    .btn.secondary { background: #5965f2; }
    .btn.warn { background: #ef476f; }
    .toast {
      position: absolute; left: 50%; transform: translateX(-50%);
      bottom: 20px; z-index: 10; padding: 10px 14px; border-radius: 8px;
      color: #fff; background: rgba(30,30,30,0.85); min-width: 240px; text-align: center;
      opacity: 0; transition: opacity 0.2s;
    }
    .toast.show { opacity: 1; }
    #modal {
      position: absolute; inset: 0; display: none; z-index: 20;
      background: rgba(0,0,0,0.6); align-items: center; justify-content: center;
    }
    #modal .card {
      background: #111; color: #fff; padding: 18px; border-radius: 10px;
      width: min(90vw, 520px);
    }
    #modal h3 { margin-top: 0; }
    #modal .row { margin: 8px 0; }
    #modal .actions { display: flex; gap: 8px; justify-content: flex-end; margin-top: 12px; }
    /* Mobile note */
    #mobileNote {
      position: absolute; left: 50%; transform: translateX(-50%); top: 60px; z-index: 9;
      background: rgba(0,0,0,0.45); color: #fff; padding: 6px 9px; border-radius: 6px;
      font-size: 12px;
    }
  </style>
</head>
<body>
  <!-- Heads-up display -->
  <div id="hud">
    <div><strong>Scenario:</strong> Warehouse Safety & Hazard ID</div>
    <div><strong>Goal:</strong> Identify <span id="goalCount">5</span> hazards across scenes.</div>
    <div><strong>Score:</strong> <span id="score">0</span> / <span id="maxScore">5</span></div>
    <div><strong>Scene:</strong> <span id="sceneName">Warehouse Floor</span></div>
    <div style="margin-top:6px">Click hotspots to classify hazards. Complete the quiz when ready.</div>
  </div>

  <!-- Scene controls -->
  <div id="controls">
    <button class="btn" data-scene="floor">Warehouse Floor</button>
    <button class="btn" data-scene="aisles">Aisles</button>
    <button class="btn" data-scene="bay">Loading Bay</button>
    <button class="btn secondary" id="openQuiz">Quiz (True/False)</button>
    <button class="btn warn" id="resetBtn">Reset</button>
  </div>

  <div id="mobileNote">Tip: On mobile, tap the center dot to interact.</div>

  <!-- Feedback toast -->
  <div id="toast" class="toast"></div>

  <!-- Quiz Modal -->
  <div id="modal">
    <div class="card">
      <h3>Quick Quiz (True / False)</h3>
      <div class="row">1) Wearing PPE is mandatory in the loading bay. <em>(True/False)</em></div>
      <div class="actions">
        <button class="btn" data-q="1" data-answer="true">True</button>
        <button class="btn" data-q="1" data-answer="false">False</button>
      </div>
      <div class="row">2) Blocking emergency exits is acceptable during peak hours. <em>(True/False)</em></div>
      <div class="actions">
        <button class="btn" data-q="2" data-answer="true">True</button>
        <button class="btn" data-q="2" data-answer="false">False</button>
      </div>
      <div class="actions">
        <button class="btn secondary" id="closeQuiz">Close</button>
      </div>
      <div id="quizResult" class="row" style="margin-top:10px;"></div>
    </div>
  </div>

  <!-- A-Frame Scene -->
  <a-scene renderer="colorManagement: true" background="color: #000">
    <!-- Camera + Cursor (good for both desktop and mobile) -->
    <a-entity id="cameraRig">
      <a-entity camera look-controls position="0 1.6 0">
        <a-entity 
          cursor="fuse:false; rayOrigin: mouse" 
          raycaster="objects: .clickable"
          position="0 0 -1"
          geometry="primitive: ring; radiusInner: 0.01; radiusOuter: 0.015"
          material="color: #fff; shader: flat">
        </a-entity>
      </a-entity>
    </a-entity>

    <!-- 360° Sky: swap src via JS depending on scene -->
    #imgFloor</a-sky>

    <!-- Assets (replace with your images) -->
    <a-assets>
      <!-- Use your own high-res equirectangular images -->
      assets/warehouse-floor-360.jpg
      assets/warehouse-aisles-360.jpg
      assets/loading-bay-360.jpg

      <!-- Simple icons/badges for hotspots -->
      assets/icon-hazard.png
      assets/icon-info.png
    </a-assets>

    <!-- Hotspot containers per scene -->
    <a-entity id="scene-floor"></a-entity>
    <a-entity id="scene-aisles" visible="false"></a-entity>
    <a-entity id="scene-bay" visible="false"></a-entity>
  </a-scene>

  <script>
    // ------- State -------
    const state = {
      currentScene: 'floor',
      score: 0,
      maxScore: 5,
      hazardsFound: new Set(),     // track by unique hotspot IDs
      quizAnswers: {1:null, 2:null},
      quizCorrect: {1: true, 2: false}, // Q1 True, Q2 False
    };

    // ------- DOM helpers -------
    const sceneNameEl = document.getElementById('sceneName');
    const scoreEl = document.getElementById('score');
    const maxScoreEl = document.getElementById('maxScore');
    const toastEl = document.getElementById('toast');
    const skyEl = document.getElementById('sky');
    const floorContainer = document.getElementById('scene-floor');
    const aislesContainer = document.getElementById('scene-aisles');
    const bayContainer = document.getElementById('scene-bay');
    maxScoreEl.textContent = state.maxScore;

    function setToast(msg, ok=true) {
      toastEl.textContent = msg;
      toastEl.style.background = ok ? 'rgba(33, 111, 67, 0.9)' : 'rgba(150, 40, 40, 0.9)';
      toastEl.classList.add('show');
      setTimeout(() => toastEl.classList.remove('show'), 1600);
    }

    function updateHUD() {
      scoreEl.textContent = state.score;
      const sceneNames = {floor: 'Warehouse Floor', aisles: 'Aisles', bay: 'Loading Bay'};
      sceneNameEl.textContent = sceneNames[state.currentScene];
    }

    function resetAll() {
      state.currentScene = 'floor';
      state.score = 0;
      state.hazardsFound.clear();
      state.quizAnswers = {1:null, 2:null};
      loadScene('floor');
      setToast('Reset complete', true);
      updateHUD();
    }

    // ------- Scene switching -------
    function loadScene(key) {
      // Swap sky texture
      const skyMap = {floor: '#imgFloor', aisles: '#imgAisles', bay: '#imgBay'};
      skyEl.setAttribute('src', skyMap[key]);

      // Toggle containers
      floorContainer.setAttribute('visible', key === 'floor');
      aislesContainer.setAttribute('visible', key === 'aisles');
      bayContainer.setAttribute('visible', key === 'bay');

      state.currentScene = key;
      updateHUD();
    }

    // ------- Hotspot factory -------
    /**
     * Create a flat hotspot image that the user can click. Mount into parent.
     * @param {Object} opts - {id, parent, position, rotation, label, isHazard}
     */
    function createHotspot(opts) {
      const {id, parent, position, rotation, label, isHazard=false} = opts;

      const icon = isHazard ? '#iconHazard' : '#iconInfo';
      const el = document.createElement('a-image');
      el.setAttribute('src', icon);
      el.setAttribute('class', 'clickable');
      el.setAttribute('position', position || '0 1.6 -3');
      el.setAttribute('rotation', rotation || '0 0 0');
      el.setAttribute('scale', '0.6 0.6 0.6');
      el.setAttribute('transparent', true);
      el.setAttribute('alphaTest', 0.5);

      // Hover pulse
      el.setAttribute('animation__hover', 'property: scale; to: 0.72 0.72 0.72; dir: alternate; dur: 550; loop: true; startEvents: mouseenter; pauseEvents: mouseleave');

      // Tooltip text (simple floating label)
      const text = document.createElement('a-entity');
      text.setAttribute('text', `value: ${label}; align: center; color: #fff; width: 3`);
      text.setAttribute('position', '0 0.35 0');
      el.appendChild(text);

      // Click behavior
      el.addEventListener('click', () => {
        const already = state.hazardsFound.has(id);
        if (isHazard) {
          if (!already) {
            state.hazardsFound.add(id);
            state.score = Math.min(state.maxScore, state.hazardsFound.size);
            updateHUD();
            setToast('Correct: Hazard identified ✔️', true);
          } else {
            setToast('Already counted ✔️', true);
          }
        } else {
          setToast('Incorrect: This is not a hazard ❌', false);
        }
      });

      parent.appendChild(el);
      return el;
    }

    // ------- Populate scenes with hotspots -------
    function populateScenes() {
      // Clear existing (if any)
      floorContainer.innerHTML = '';
      aislesContainer.innerHTML = '';
      bayContainer.innerHTML = '';

      // Scene: Warehouse Floor (360 image positions approximated)
      createHotspot({
        id: 'floor-slip',
        parent: floorContainer,
        position: '-2 1.4 -3',
        rotation: '0 15 0',
        label: 'Slip hazard (spill)',
        isHazard: true
      });
      createHotspot({
        id: 'floor-info-pallets',
        parent: floorContainer,
        position: '1.5 1.6 -2.6',
        rotation: '0 -10 0',
        label: 'Stacking info',
        isHazard: false
      });

      // Scene: Aisles
      createHotspot({
        id: 'aisle-cables',
        parent: aislesContainer,
        position: '-1.2 1.3 -2.2',
        rotation: '0 0 0',
        label: 'Loose cables',
        isHazard: true
      });
      createHotspot({
        id: 'aisle-blocked-exit',
        parent: aislesContainer,
        position: '2.1 1.5 -3.0',
        rotation: '0 12 0',
        label: 'Blocked exit',
        isHazard: true
      });
      createHotspot({
        id: 'aisle-info-signage',
        parent: aislesContainer,
        position: '0.8 1.2 -2.8',
        rotation: '0 -5 0',
        label: 'Signage info',
        isHazard: false
      });

      // Scene: Loading Bay
      createHotspot({
        id: 'bay-missing-ppe',
        parent: bayContainer,
        position: '-1.6 1.55 -2.4',
        rotation: '0 0 0',
        label: 'Missing PPE',
        isHazard: true
      });
      createHotspot({
        id: 'bay-slip',
        parent: bayContainer,
        position: '1.8 1.3 -3.2',
        rotation: '0 0 0',
        label: 'Oil spill',
        isHazard: true
      });
      createHotspot({
        id: 'bay-info-loading',
        parent: bayContainer,
        position: '0.2 1.1 -2.0',
        rotation: '0 0 0',
        label: 'Loading procedure',
        isHazard: false
      });
    }

    // ------- Init -------
    populateScenes();
    loadScene('floor');
    updateHUD();

    // ------- UI event wiring -------
    // Scene buttons
    document.querySelectorAll('[data-scene]').forEach(btn => {
      btn.addEventListener('click', (e) => {
        const key = e.target.getAttribute('data-scene');
        loadScene(key);
      });
    });

    // Reset
    document.getElementById('resetBtn').addEventListener('click', resetAll);

    // Quiz modal
    const modal = document.getElementById('modal');
    const quizResultEl = document.getElementById('quizResult');
    document.getElementById('openQuiz').addEventListener('click', () => {
      modal.style.display = 'flex';
      quizResultEl.textContent = '';
    });
    document.getElementById('closeQuiz').addEventListener('click', () => {
      modal.style.display = 'none';
    });

    // Quiz button answers
    modal.querySelectorAll('.card .actions .btn[data-q]').forEach(btn => {
      btn.addEventListener('click', (e) => {
        const q = e.target.getAttribute('data-q');
        const ans = e.target.getAttribute('data-answer') === 'true';
        state.quizAnswers[q] = ans;

        // Immediate feedback for each question
        const correct = state.quizCorrect[q] === ans;
        setToast(correct ? `Q${q}: Correct ✔️` : `Q${q}: Incorrect ❌`, correct);

        // If both answered, show summary
        if (state.quizAnswers[1] !== null && state.quizAnswers[2] !== null) {
          const total = (state.quizCorrect[1] === state.quizAnswers[1]) + (state.quizCorrect[2] === state.quizAnswers[2]);
          quizResultEl.textContent = `Quiz complete: ${total} / 2 correct.`;
        }
      });
    });

    // Optional: hide mobile tip after 5s
    setTimeout(() => {
      const note = document.getElementById('mobileNote');
      if (note) note.style.display = 'none';
    }, 5000);
  </script>
</body>
</html>
