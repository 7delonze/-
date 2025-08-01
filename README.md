<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <title>학교 건물 강화하기</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      text-align: center;
      padding: 30px;
      background-color: #eef6ff;
    }
    #reset-btn {
      position: fixed;
      top: 20px;
      right: 20px;
      padding: 10px 15px;
      background-color: crimson;
      color: white;
      border: none;
      border-radius: 8px;
      font-weight: bold;
      cursor: pointer;
      z-index: 100;
    }
    #shop {
      margin-top: 30px;
      border: 2px solid #ccc;
      border-radius: 10px;
      padding: 15px;
      display: inline-block;
      background: #fff;
    }
    #shop h2 {
      margin-top: 0;
    }
    #shop button {
      margin: 5px;
      background-color: darkblue;
      color: white;
      padding: 6px 12px;
      border: none;
      border-radius: 5px;
      cursor: pointer;
    }
    #inventory {
      margin-top: 20px;
      border: 2px solid #ccc;
      border-radius: 10px;
      padding: 10px;
      display: inline-block;
      background: #fff;
      min-width: 260px;
      text-align: left;
      font-size: 18px;
    }
    #inventory h2 {
      margin-top: 0;
    }
    #inventory button {
      margin-left: 10px;
      background-color: darkgreen;
      color: white;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      padding: 3px 7px;
      font-size: 14px;
    }
    #building {
      width: 300px;
      margin: 20px;
      transition: transform 0.3s ease;
    }
    #effect {
      position: absolute;
      top: 180px;
      left: 50%;
      transform: translateX(-50%);
      width: 150px;
      height: 150px;
      display: none;
      z-index: 10;
    }
    button.upgrade, button.sell {
      padding: 12px 25px;
      font-size: 18px;
      margin: 10px 10px 0 10px;
      background-color: #4CAF50;
      color: white;
      border: none;
      border-radius: 8px;
      cursor: pointer;
    }
    #money, #level, #rate, #cost {
      font-size: 20px;
      font-weight: bold;
      margin: 10px 0;
    }
    #message {
      margin-top: 20px;
      font-weight: bold;
      font-size: 24px;
      transition: transform 0.3s ease;
      min-height: 28px;
    }
    .animate {
      transform: scale(1.5);
    }
    #disabled-message {
      color: darkred;
      font-size: 22px;
      font-weight: bold;
      margin-top: 15px;
      min-height: 26px;
    }
  </style>
</head>
<body>

  <button id="reset-btn" onclick="resetGame()">🔄 리셋</button>

  <h1>학교 건물 강화하기</h1>
  <p id="money">현재 자산: 7,770,000원</p>
  <p id="level">현재 강화 단계: 1</p>
  <p id="rate">강화 확률: 95%</p>
  <p id="cost">강화 비용: 10,000원 / 판매 금액: 5,000원</p>

  <div style="position: relative;">
    <img id="building" src="school_lv1.png" alt="학교 건물" />
    <img id="effect" src="" alt="이펙트" />
  </div>

  <button class="upgrade" id="upgrade-btn" onclick="upgrade()">강화하기</button>
  <button class="sell" onclick="sellBuilding()">건물 판매</button>
  <p id="message"></p>
  <p id="disabled-message"></p>

  <div id="shop">
    <h2>🏪 상점</h2>
    <p><button onclick="buyStart8()">8강부터 시작권 (800,000원)</button></p>
    <p><button onclick="buyRestore()">강화실패 복구권 (700,000원)</button></p>
  </div>

  <div id="inventory">
    <h2>🎒 인벤토리</h2>
    <ul id="inventory-list" style="list-style:none; padding-left:0;">
      <li>8강부터 시작권: <span id="start8-count">0</span>개 
        <button onclick="useStart8()">[사용하기]</button>
      </li>
      <li>강화 실패 복구권: <span id="restore-count">0</span>개
        <button onclick="useRestore()">[사용하기]</button>
      </li>
    </ul>
  </div>

  <audio id="explosion-sound" src="explosion.mp3" preload="auto"></audio>

  <script>
    const INITIAL_MONEY = 7770000;
    let money = INITIAL_MONEY;
    let level = 1;
    let isDisabled = false;
    let lastLevel = 1;

    // 인벤토리 아이템 개수
    let restoreTicketCount = 0;
    let startFrom8Count = 0;

    const maxLevel = 15;
    const upgradeBtn = document.getElementById("upgrade-btn");
    const moneyEl = document.getElementById("money");
    const levelEl = document.getElementById("level");
    const rateEl = document.getElementById("rate");
    const costEl = document.getElementById("cost");
    const msgEl = document.getElementById("message");
    const disabledMsg = document.getElementById("disabled-message");
    const building = document.getElementById("building");
    const effect = document.getElementById("effect");
    const explosionSound = document.getElementById("explosion-sound");

    // 단계별 강화 비용(원), 0단계 dummy
    const upgradeCost = [
      0, 10000, 20000, 30000, 40000, 50000,
      60000, 80000, 90000, 100000, 110000,
      120000, 130000, 140000, 150000
    ];
    // 단계별 판매 금액(원), 0단계 dummy
    const sellPrice = [
      0, 5000, 12000, 20000, 28000, 35000,
      42000, 100000, 120000, 140000, 160000,
      180000, 210000, 250000, 300000
    ];

    function getSuccessRate(lv) {
      if (lv === 1) return 0.95;
      if (lv >= 2 && lv <= 5) return 1 - lv * 0.05;
      if (lv >= 6 && lv <= 10) return 0.65;
      return 0.5;
    }

    function displayRate(rate) {
      return Math.round(rate * 100) + "%";
    }

    function updateUI() {
      moneyEl.innerText = `현재 자산: ${money.toLocaleString()}원`;
      levelEl.innerText = `현재 강화 단계: ${level}`;
      rateEl.innerText = `강화 확률: ${displayRate(getSuccessRate(level))}`;
      const cost = upgradeCost[level] || 0;
      const sell = sellPrice[level - 1] || 0;
      costEl.innerText = `강화 비용: ${cost.toLocaleString()}원 / 판매 금액: ${sell.toLocaleString()}원`;
      building.src = `school_lv${level}.png`;
    }

    function updateInventoryUI() {
      document.getElementById("restore-count").innerText = restoreTicketCount;
      document.getElementById("start8-count").innerText = startFrom8Count;
    }

    function animateMessage(text, success) {
      msgEl.innerText = text;
      msgEl.classList.add("animate");
      msgEl.style.color = success ? "green" : "red";
      setTimeout(() => msgEl.classList.remove("animate"), 500);
    }

    function upgrade() {
      if (isDisabled || level >= maxLevel) {
        animateMessage("더 이상 강화할 수 없습니다!", false);
        return;
      }

      const cost = upgradeCost[level];
      if (money < cost) {
        animateMessage("💸 돈이 부족합니다!", false);
        return;
      }

      money -= cost;
      lastLevel = level;
      const success = Math.random() < getSuccessRate(level);

      if (success) {
        level++;
        if (level === maxLevel) {
          effect.src = "fireworks.gif";
          effect.style.display = "block";
          animateMessage("🎉 축하합니다! 최고 단계를 달성했습니다!", true);
        } else {
          effect.src = "success_effect.gif";
          effect.style.display = "block";
          animateMessage("✨ 강화 성공!", true);
        }
      } else {
        if (restoreTicketCount > 0) {
          restoreTicketCount--;
          level = lastLevel;
          animateMessage("🛡 복구권 사용! 실패 전 단계로 복구!", true);
          updateInventoryUI();
        } else {
          level = Math.max(1, level - 2);
          explosionSound.currentTime = 0;
          explosionSound.play();
          effect.src = "explosion.gif";
          effect.style.display = "block";
          animateMessage("💥 강화 실패!", false);

          // 10% 확률로 영구 강화 불가
          if (Math.random() < 0.1) {
            isDisabled = true;
            upgradeBtn.disabled = true;
            disabledMsg.innerText = "⚠ 강화 불가! 더 이상 강화할 수 없습니다.";
          }
        }
      }

      updateUI();
      setTimeout(() => (effect.style.display = "none"), 1000);
    }

    function sellBuilding() {
      const sell = sellPrice[level - 1];
      money += sell;
      animateMessage(`💰 ${sell.toLocaleString()}원에 건물을 판매했습니다!`, true);
      level = 1;
      updateUI();
    }

    function buyStart8() {
      if (money >= 800000) {
        money -= 800000;
        startFrom8Count++;
        animateMessage("🧱 8강부터 시작권 구입 완료!", true);
        updateUI();
        updateInventoryUI();
      } else {
        animateMessage("❌ 돈이 부족합니다!", false);
      }
    }

    function buyRestore() {
      if (money >= 700000) {
        money -= 700000;
        restoreTicketCount++;
        animateMessage("🛡 복구권 구입 완료!", true);
        updateUI();
        updateInventoryUI();
      } else {
        animateMessage("❌ 돈이 부족합니다!", false);
      }
    }

    // 인벤토리에서 8강부터 시작권 사용
    function useStart8() {
      if (startFrom8Count > 0) {
        if (level >= 8) {
          animateMessage("이미 8강 이상입니다!", false);
          return;
        }
        startFrom8Count--;
        level = 8;
        animateMessage("🧱 8강부터 시작권을 사용하여 8단계로 이동했습니다!", true);
        updateInventoryUI();
        updateUI();
      } else {
        animateMessage("8강부터 시작권이 없습니다!", false);
      }
    }

    // 인벤토리에서 복구권 사용 (현재 단계 1단계 복구)
    function useRestore() {
      if (restoreTicketCount > 0) {
        if (level <= 1) {
          animateMessage("더 이상 복구할 단계가 없습니다!", false);
          return;
        }
        restoreTicketCount--;
        level = Math.min(maxLevel, level + 1);
        animateMessage("🛡 강화복구권을 사용하여 1단계 복구했습니다!", true);
        updateInventoryUI();
        updateUI();
      } else {
        animateMessage("복구권이 없습니다!", false);
      }
    }

    function resetGame() {
      if (startFrom8Count > 0) {
        level = 8;
        startFrom8Count--;
        updateInventoryUI();
      } else {
        level = 1;
      }
      money = INITIAL_MONEY;
      isDisabled = false;
      upgradeBtn.disabled = false;
      msgEl.innerText = "";
      disabledMsg.innerText = "";
      effect.style.display = "none";
      updateUI();
    }

    // 초기 UI 세팅
    updateUI();
    updateInventoryUI();
  </script>
</body>
</html>

