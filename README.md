<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>SkyMultiplier - Crash Game Demo</title>
  <style>
    * { box-sizing: border-box; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
    body {
      background-color: #0e1015;
      color: #ffffff;
      display: flex;
      flex-direction: column;
      align-items: center;
      margin: 0;
      padding: 20px;
    }
    .header {
      width: 100%;
      max-width: 600px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      background: #1b1e28;
      padding: 15px 20px;
      border-radius: 10px;
      margin-bottom: 20px;
    }
    .balance { font-size: 1.2rem; font-weight: bold; color: #28a745; }
    .game-screen {
      width: 100%;
      max-width: 600px;
      height: 300px;
      background: radial-gradient(circle at bottom left, #2c1623, #14161f);
      border: 2px solid #2a2e3d;
      border-radius: 12px;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      position: relative;
      overflow: hidden;
      margin-bottom: 20px;
    }
    .multiplier {
      font-size: 4rem;
      font-weight: 800;
      z-index: 2;
    }
    .crashed { color: #e53935; }
    .status-text {
      font-size: 1.1rem;
      color: #a0a5b5;
      margin-top: 10px;
      z-index: 2;
    }
    .plane {
      font-size: 2.5rem;
      position: absolute;
      bottom: 20px;
      left: 20px;
      transition: transform 0.1s linear;
    }
    .controls {
      width: 100%;
      max-width: 600px;
      background: #1b1e28;
      padding: 20px;
      border-radius: 12px;
      display: flex;
      gap: 15px;
      align-items: center;
      justify-content: space-between;
    }
    input[type="number"] {
      width: 140px;
      padding: 12px;
      font-size: 1.1rem;
      border-radius: 8px;
      border: 1px solid #3a3f52;
      background: #0e1015;
      color: white;
    }
    button {
      flex: 1;
      padding: 15px;
      font-size: 1.2rem;
      font-weight: bold;
      border: none;
      border-radius: 8px;
      cursor: pointer;
      color: white;
      transition: background 0.2s;
    }
    .btn-bet { background-color: #28a745; }
    .btn-bet:hover { background-color: #218838; }
    .btn-cashout { background-color: #ff9800; color: #000; }
    .btn-cashout:hover { background-color: #e68900; }
    .btn-disabled { background-color: #44495a; cursor: not-allowed; color: #888; }
    .history {
      width: 100%;
      max-width: 600px;
      display: flex;
      gap: 8px;
      margin-bottom: 15px;
      overflow-x: auto;
    }
    .badge {
      padding: 5px 10px;
      border-radius: 15px;
      font-size: 0.85rem;
      font-weight: bold;
      background: #232734;
    }
    .badge.high { color: #9c27b0; }
    .badge.mid { color: #2196f3; }
  </style>
</head>
<body>

  <div class="header">
    <div><strong>SkyMultiplier</strong> (Demo)</div>
    <div class="balance">KES <span id="balanceDisplay">1000.00</span></div>
  </div>

  <div class="history" id="historyBar"></div>

  <div class="game-screen">
    <div class="plane" id="planeIcon">✈️</div>
    <div class="multiplier" id="multiplierDisplay">1.00x</div>
    <div class="status-text" id="statusText">Place your bet for the next round</div>
  </div>

  <div class="controls">
    <div>
      <label for="betInput" style="display:block; font-size:0.8rem; color:#aaa; margin-bottom:4px;">Bet Amount (KES)</label>
      <input type="number" id="betInput" value="50" min="10" max="10000">
    </div>
    <button id="actionBtn" class="btn-bet">PLACE BET</button>
  </div>

  <script>
    let balance = 1000.00;
    let currentBet = 0;
    let multiplier = 1.00;
    let crashPoint = 1.00;
    let gameState = 'WAITING'; // WAITING, FLYING, CRASHED
    let gameInterval;
    let hasCashedOut = false;

    const balanceDisplay = document.getElementById('balanceDisplay');
    const multiplierDisplay = document.getElementById('multiplierDisplay');
    const statusText = document.getElementById('statusText');
    const actionBtn = document.getElementById('actionBtn');
    const betInput = document.getElementById('betInput');
    const planeIcon = document.getElementById('planeIcon');
    const historyBar = document.getElementById('historyBar');

    function updateBalance() {
      balanceDisplay.textContent = balance.toFixed(2);
    }

    // Generates a crash point with a built-in house edge
    function generateCrashPoint() {
      const r = Math.random();
      if (r < 0.05) return 1.00; // 5% instant crash
      const crash = 0.96 / (1 - r); // 4% house edge
      return Math.max(1.01, Math.min(crash, 100)).toFixed(2);
    }

    function addHistory(val) {
      const badge = document.createElement('span');
      badge.className = 'badge ' + (val >= 10 ? 'high' : val >= 2 ? 'mid' : '');
      badge.textContent = val + 'x';
      historyBar.prepend(badge);
      if (historyBar.children.length > 8) {
        historyBar.removeChild(historyBar.lastChild);
      }
    }

    function startWaitingPhase() {
      gameState = 'WAITING';
      multiplier = 1.00;
      hasCashedOut = false;
      multiplierDisplay.textContent = '1.00x';
      multiplierDisplay.classList.remove('crashed');
      planeIcon.style.transform = 'translate(0px, 0px)';

      if (currentBet > 0) {
        statusText.textContent = `Bet of KES ${currentBet} locked! Taking off soon...`;
        actionBtn.textContent = 'WAITING...';
        actionBtn.className = 'btn-disabled';
      } else {
        statusText.textContent = 'Place your bet before takeoff (5s)';
        actionBtn.textContent = 'PLACE BET';
        actionBtn.className = 'btn-bet';
      }

      setTimeout(startFlight, 5000);
    }

    function startFlight() {
      gameState = 'FLYING';
      crashPoint = parseFloat(generateCrashPoint());
      statusText.textContent = 'Flying...';

      if (currentBet > 0) {
        actionBtn.textContent = `CASH OUT (KES ${(currentBet * multiplier).toFixed(2)})`;
        actionBtn.className = 'btn-cashout';
      } else {
        actionBtn.textContent = 'ROUND IN PROGRESS';
        actionBtn.className = 'btn-disabled';
      }

      let xPos = 0;
      let yPos = 0;

      gameInterval = setInterval(() => {
        multiplier += 0.01 * Math.pow(multiplier, 0.5);
        multiplierDisplay.textContent = multiplier.toFixed(2) + 'x';

        // Move plane visually
        xPos = Math.min(xPos + 2, 480);
        yPos = Math.min(yPos + 1, 200);
        planeIcon.style.transform = `translate(${xPos}px, -${yPos}px)`;

        if (currentBet > 0 && !hasCashedOut) {
          actionBtn.textContent = `CASH OUT (KES ${(currentBet * multiplier).toFixed(2)})`;
        }

        if (multiplier >= crashPoint) {
          endFlight();
        }
      }, 60);
    }

    function endFlight() {
      clearInterval(gameInterval);
      gameState = 'CRASHED';
      multiplierDisplay.textContent = crashPoint.toFixed(2) + 'x';
      multiplierDisplay.classList.add('crashed');
      statusText.textContent = 'FLEW AWAY!';
      addHistory(crashPoint.toFixed(2));

      currentBet = 0;
      actionBtn.textContent = 'ROUND ENDED';
      actionBtn.className = 'btn-disabled';

      setTimeout(startWaitingPhase, 3000);
    }

    actionBtn.addEventListener('click', () => {
      if (gameState === 'WAITING' && currentBet === 0) {
        const betAmount = parseFloat(betInput.value);
        if (isNaN(betAmount) || betAmount <= 0 || betAmount > balance) {
          alert('Invalid bet amount or insufficient balance!');
          return;
        }
        balance -= betAmount;
        currentBet = betAmount;
        updateBalance();
        statusText.textContent = `Bet of KES ${currentBet} placed!`;
        actionBtn.textContent = 'BET PLACED';
        actionBtn.className = 'btn-disabled';
      } 
      else if (gameState === 'FLYING' && currentBet > 0 && !hasCashedOut) {
        hasCashedOut = true;
        const winnings = currentBet * multiplier;
        balance += winnings;
        updateBalance();
        statusText.textContent = `Cashed out at ${multiplier.toFixed(2)}x! Won KES ${winnings.toFixed(2)}`;
        actionBtn.textContent = 'CASHED OUT';
        actionBtn.className = 'btn-disabled';
        currentBet = 0;
      }
    });

    // Initialize game loop
    startWaitingPhase();
  </script>
</body>
</html>
