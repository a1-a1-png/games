<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=320, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>幸运刮刮卡</title>
  <style>
    body {
      background: #f7f7f7;
      min-height: 100vh;
      margin: 0;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      font-family: '微软雅黑', Arial, sans-serif;
    }
    .container {
      background: #fff;
      border-radius: 16px;
      box-shadow: 0 4px 24px rgba(0,0,0,0.08);
      padding: 32px 24px 24px 24px;
      display: flex;
      flex-direction: column;
      align-items: center;
      min-width: 320px;
      max-width: 90vw;
    }
    h1 {
      margin-bottom: 18px;
      color: #ff9800;
      font-size: 2rem;
    }
    #scratchCard {
      border-radius: 12px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.08);
      margin-bottom: 16px;
      touch-action: none;
      background: #fffbe6;
      display: block;
    }
    .tip {
      color: #888;
      font-size: 15px;
      margin-bottom: 8px;
    }
    .count {
      color: #ff9800;
      font-size: 16px;
      margin-bottom: 8px;
    }
    .btn-reset {
      margin-top: 10px;
      padding: 6px 18px;
      border: none;
      border-radius: 6px;
      background: #ff9800;
      color: #fff;
      font-size: 15px;
      cursor: pointer;
      transition: background 0.2s;
    }
    .btn-reset:active {
      background: #e68900;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>幸运刮刮卡</h1>
    <div class="count" id="countInfo"></div>
    <canvas id="scratchCard" width="300" height="120"></canvas>
    <div class="tip" id="tip">用鼠标或手指刮开灰色区域查看中奖信息</div>
    <button class="btn-reset" id="resetBtn" style="display:none;">再来一次</button>
  </div>
  <script>
    const MAX_TIMES = 5;
    const canvas = document.getElementById('scratchCard');
    const ctx = canvas.getContext('2d');
    const tip = document.getElementById('tip');
    const countInfo = document.getElementById('countInfo');
    const resetBtn = document.getElementById('resetBtn');
    function getTimes() {
      return parseInt(localStorage.getItem('scratch_times') || '0', 10);
    }
    // 设置已刮次数
    function setTimes(n) {
      localStorage.setItem('scratch_times', n);
    }
    // 初始化中奖结果
    let isWin = false;
    let finished = false;
    function randomWin() {
      // 40%概率中奖
      return Math.random() < 0.4;
    }
    function drawPrize() {
      ctx.globalCompositeOperation = 'source-over'; // 关键，防止中奖信息被“挖空”
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = '#fffbe6';
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.font = 'bold 28px 微软雅黑';
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      ctx.fillStyle = isWin ? '#ff9800' : '#888';
      ctx.fillText(isWin ? '恭喜你，中奖了！🎉' : '很遗憾，未中奖', canvas.width / 2, canvas.height / 2);
    }
    function drawMask() {
      ctx.globalCompositeOperation = 'source-over';
      ctx.fillStyle = '#bbb';
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.font = '20px 微软雅黑';
      ctx.fillStyle = '#888';
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      ctx.fillText('刮一刮', canvas.width / 2, canvas.height / 2);
    }
    function updateCountInfo() {
      const times = getTimes();
      countInfo.textContent = `今日剩余刮卡次数：${Math.max(0, MAX_TIMES - times)} / ${MAX_TIMES}`;
    }
    function initCard() {
      finished = false;
      isWin = randomWin();
      drawPrize();
      drawMask();
      tip.textContent = '用鼠标或手指刮开灰色区域查看中奖信息';
      resetBtn.style.display = 'none';
    }
    // 事件相关
    let isDrawing = false;
    function getPos(e) {
      let x, y;
      if (e.touches) {
        const rect = canvas.getBoundingClientRect();
        x = e.touches[0].clientX - rect.left;
        y = e.touches[0].clientY - rect.top;
      } else {
        x = e.offsetX;
        y = e.offsetY;
      }
      return { x, y };
    }
    function scratch(e) {
      if (!isDrawing || finished) return;
      ctx.globalCompositeOperation = 'destination-out';
      const { x, y } = getPos(e);
      ctx.beginPath();
      ctx.arc(x, y, 18, 0, Math.PI * 2);
      ctx.fill();
      if (e.preventDefault) e.preventDefault();
    }
    // 检查刮开面积
    function checkClear() {
      if (finished) return;
      const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
      let cleared = 0;
      for (let i = 0; i < imgData.data.length; i += 4) {
        if (imgData.data[i + 3] === 0) cleared++;
      }
      if (cleared > imgData.data.length / 8) { // 刮开超过1/4
        finished = true;
        setTimeout(() => {
          drawPrize();
          tip.textContent = isWin ? '恭喜中奖！点击“再来一次”继续' : '未中奖，点击“再来一次”试试手气';
          let times = getTimes() + 1;
          setTimes(times);
          updateCountInfo();
          if (times < MAX_TIMES) {
            resetBtn.style.display = '';
          } else {
            tip.textContent = '今日刮卡次数已用完，欢迎明天再来！';
            resetBtn.style.display = 'none';
            // 灰色遮罩覆盖
            ctx.globalCompositeOperation = 'source-over';
            ctx.fillStyle = 'rgba(187,187,187,0.7)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
          }
        }, 300);
      }
    }
    // 事件绑定
    canvas.addEventListener('mousedown', e => {
      if (finished) return;
      if (getTimes() >= MAX_TIMES) return;
      isDrawing = true;
      scratch(e);
    });
    canvas.addEventListener('mousemove', scratch);
    canvas.addEventListener('mouseup', () => {
      isDrawing = false;
      checkClear();
    });
    canvas.addEventListener('mouseleave', () => {
      isDrawing = false;
    });
    canvas.addEventListener('touchstart', e => {
      if (finished) return;
      if (getTimes() >= MAX_TIMES) return;
      isDrawing = true;
      scratch(e);
      e.preventDefault();
    });
    canvas.addEventListener('touchmove', e => {
      scratch(e);
      e.preventDefault();
    });
    canvas.addEventListener('touchend', () => {
      isDrawing = false;
      checkClear();
    });
    // 再来一次
    resetBtn.addEventListener('click', () => {
      if (getTimes() >= MAX_TIMES) {
        tip.textContent = '今日刮卡次数已用完，欢迎明天再来！';
        resetBtn.style.display = 'none';
        return;
      }
      initCard();
    });
    // 初始化
    function main() {
      updateCountInfo();
      if (getTimes() >= MAX_TIMES) {
        drawPrize();
        ctx.globalCompositeOperation = 'source-over';
        ctx.fillStyle = 'rgba(187,187,187,0.7)';
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        tip.textContent = '今日刮卡次数已用完，欢迎明天再来！';
        resetBtn.style.display = 'none';
      } else {
        initCard();
      }
    }
    main();
  </script>
</body>
</html>
