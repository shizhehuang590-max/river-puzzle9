<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>8人過河謎題 - 真實倒帶回溯</title>
    <style>
        body { font-family: 'Segoe UI', Tahoma, sans-serif; background-color: #f0f8ff; text-align: center; padding: 20px; }
        #game-board {
            display: flex; justify-content: space-between; align-items: center;
            background-color: #fff; border: 3px solid #ccc; border-radius: 10px; padding: 20px; margin: 20px auto; max-width: 800px;
            transition: all 0.3s ease;
        }
        .bank { width: 30%; min-height: 150px; background-color: #e8f5e9; border: 2px dashed #4caf50; border-radius: 10px; padding: 10px; font-size: 2rem; display: flex; flex-wrap: wrap; align-content: flex-start; gap: 5px; }
        #river { width: 30%; height: 150px; background-color: #bbdefb; position: relative; border-radius: 10px; overflow: hidden; }
        #boat { font-size: 3rem; position: absolute; top: 50%; transform: translateY(-50%); transition: left 0.8s ease-in-out; }
        .boat-left { left: 10px; } .boat-right { left: calc(100% - 60px); }
        button { padding: 10px 20px; font-size: 1.1rem; margin: 0 10px; cursor: pointer; background-color: #2196f3; color: white; border: none; border-radius: 5px; min-width: 120px; }
        button:hover { background-color: #1976d2; }
        #status-text { font-size: 1.5rem; font-weight: bold; margin: 20px auto; min-height: 60px; max-width: 700px; transition: color 0.3s; }
        .error-state { border-color: #e74c3c !important; box-shadow: 0 0 20px rgba(231, 76, 60, 0.6) !important; }
        .backtrack-state { border-color: #f39c12 !important; box-shadow: 0 0 15px rgba(243, 156, 18, 0.4) !important; }
    </style>
</head>
<body>
    <h1>🧠 8人過河謎題：真實倒帶推演</h1>
    <div id="status-text">準備開始推演...</div>
    <div id="game-board">
        <div id="left-bank" class="bank"></div>
        <div id="river"><div id="boat" class="boat-left">🛶</div></div>
        <div id="right-bank" class="bank"></div>
    </div>
    <div id="controls">
        <button id="play-btn" onclick="togglePlay()">▶️ 自動推演</button>
        <span id="step-counter" style="font-size: 1.2rem; font-weight: bold; margin: 0 15px;">節點: 0</span>
    </div>

    <script>
        const states = [
            { desc: "【初始狀態】所有人準備過河", L: "👨 👩 👦 👦 👧 👧 💂 🐕", R: "", boat: "left" },
            // 錯誤路徑 1 (深入 3 步後發現錯誤)
            { desc: "❌ 錯誤嘗試 1：讓僕人和狗先過河 (💂🐕 ➡️)", L: "👨 👩 👦 👦 👧 👧", R: "💂 🐕", boat: "right" },
            { desc: "❌ 錯誤嘗試 1：僕人獨自返回 (⬅️ 💂)", L: "👨 👩 👦 👦 👧 👧 💂", R: "🐕", boat: "left" },
            { desc: "🚨 致命錯誤：爸爸帶兒子過河！狗沒人管會咬人！", L: "👩 👦 👧 👧 💂", R: "👨 👦 🐕", boat: "right", error: true },
            // 一步一步倒帶回溯
            { desc: "⏪ 任務失敗：退回上一步 (⬅️ 👨👦 退回)", L: "👨 👩 👦 👦 👧 👧 💂", R: "🐕", boat: "left", backtrack: true },
            { desc: "⏪ 發現此路徑不通，繼續退回 (💂 ➡️ 退回)", L: "👨 👩 👦 👦 👧 👧", R: "💂 🐕", boat: "right", backtrack: true },
            { desc: "⏪ 完全放棄此路線，退回起點 (⬅️ 💂🐕 退回)", L: "👨 👩 👦 👦 👧 👧 💂 🐕", R: "", boat: "left", backtrack: true },
            
            // 錯誤路徑 2 (走 1 步就發現錯誤)
            { desc: "🚨 ❌ 錯誤嘗試 2：媽媽帶女兒先過河！觸發爸爸打女兒！", L: "👨 👦 👦 👧 💂 🐕", R: "👩 👧", boat: "right", error: true },
            { desc: "⏪ 任務失敗：退回上一步 (⬅️ 👩👧 退回)", L: "👨 👩 👦 👦 👧 👧 💂 🐕", R: "", boat: "left", backtrack: true },
            
            // 正確解法
            { desc: "✅ 開始正確解法：步驟 1：👨爸爸和👦兒子 過河", L: "👩 👦 👧 👧 💂 🐕", R: "👨 👦", boat: "right" },
            { desc: "✅ 步驟 2：👨爸爸 返回", L: "👨 👩 👦 👧 👧 💂 🐕", R: "👦", boat: "left" },
            { desc: "✅ 步驟 3：👨爸爸和👦兒子 過河", L: "👩 👧 👧 💂 🐕", R: "👨 👦 👦", boat: "right" },
            { desc: "✅ 步驟 4：👨爸爸 返回", L: "👨 👩 👧 👧 💂 🐕", R: "👦 👦", boat: "left" },
            { desc: "✅ 步驟 5：👨爸爸和👩媽媽 過河", L: "👧 👧 💂 🐕", R: "👨 👩 👦 👦", boat: "right" },
            { desc: "✅ 步驟 6：👩媽媽 返回", L: "👩 👧 👧 💂 🐕", R: "👨 👦 👦", boat: "left" },
            { desc: "✅ 步驟 7：👩媽媽和👧女兒 過河", L: "👧 💂 🐕", R: "👨 👩 👦 👦 👧", boat: "right" },
            { desc: "✅ 步驟 8：👨爸爸和👩媽媽 一起返回 (防家暴關鍵！)", L: "👨 👩 👧 💂 🐕", R: "👦 👦 👧", boat: "left" },
            { desc: "✅ 步驟 9：👩媽媽和👧女兒 過河", L: "👨 💂 🐕", R: "👩 👦 👦 👧 👧", boat: "right" },
            { desc: "✅ 步驟 10：👩媽媽 返回", L: "👨 👩 💂 🐕", R: "👦 👦 👧 👧", boat: "left" },
            { desc: "✅ 步驟 11：👨爸爸和👩媽媽 過河", L: "💂 🐕", R: "👨 👩 👦 👦 👧 👧", boat: "right" },
            { desc: "✅ 步驟 12：👨爸爸 返回，準備交接給僕人", L: "👨 💂 🐕", R: "👩 👦 👦 👧 👧", boat: "left" },
            { desc: "✅ 步驟 13：💂僕人和🐕狗 過河！", L: "👨", R: "👩 👦 👦 👧 👧 💂 🐕", boat: "right" },
            { desc: "✅ 步驟 14：👩媽媽 返回接最後一人", L: "👨 👩", R: "👦 👦 👧 👧 💂 🐕", boat: "left" },
            { desc: "🎉 步驟 15：👨爸爸和👩媽媽 過河，全員完美通關！", L: "", R: "👨 👩 👦 👦 👧 👧 💂 🐕", boat: "right" }
        ];

        let currentStep = 0; let isPlaying = false; let playInterval;

        function updateBoard() {
            const state = states[currentStep];
            document.getElementById("left-bank").innerText = state.L;
            document.getElementById("right-bank").innerText = state.R;
            
            const statusText = document.getElementById("status-text");
            const board = document.getElementById("game-board");
            statusText.innerText = state.desc;
            document.getElementById("step-counter").innerText = `節點: ${currentStep} / ${states.length - 1}`;
            document.getElementById("boat").className = state.boat === "left" ? "boat-left" : "boat-right";

            board.classList.remove("error-state", "backtrack-state");
            if (state.error) {
                board.classList.add("error-state"); statusText.style.color = "#c0392b";
            } else if (state.backtrack) {
                board.classList.add("backtrack-state"); statusText.style.color = "#d35400";
            } else {
                statusText.style.color = "#333";
            }

            if (currentStep === states.length - 1 && isPlaying) togglePlay();
        }

        function togglePlay() {
            const playBtn = document.getElementById("play-btn");
            if (isPlaying) {
                clearInterval(playInterval); isPlaying = false;
                playBtn.innerText = "▶️ 繼續推演"; playBtn.style.backgroundColor = "#2196f3";
            } else {
                if (currentStep === states.length - 1) currentStep = 0;
                isPlaying = true; playBtn.innerText = "⏸️ 暫停"; playBtn.style.backgroundColor = "#f44336";
                updateBoard();
                playInterval = setInterval(() => { if (currentStep < states.length - 1) { currentStep++; updateBoard(); } }, 2500);
            }
        }
        updateBoard();
    </script>
</body>
</html>
