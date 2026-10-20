<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Game Dò Mìn - Tăng Độ Khó & Âm Thanh</title>
    <style>
        :root {
            --bg-color: #1e1e24;
            --cell-size: 40px;
            --cell-bg: #434955;
            --cell-revealed: #a0aab2;
            --cell-border: #2e323b;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: var(--bg-color);
            color: #fff;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            margin: 0;
            user-select: none;
        }

        h1 {
            margin-bottom: 5px;
            color: #ff4a5a;
            text-shadow: 0 0 10px rgba(255, 74, 90, 0.5);
        }

        .stats-container {
            display: flex;
            gap: 20px;
            font-size: 1.1rem;
            margin-bottom: 15px;
            background: #2b2f3a;
            padding: 10px 20px;
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.3);
        }

        .stat-box span {
            font-weight: bold;
            color: #00ffcc;
        }

        #game-board {
            display: grid;
            background-color: var(--cell-border);
            padding: 6px;
            border-radius: 10px;
            box-shadow: 0 10px 25px rgba(0,0,0,0.5);
            gap: 2px;
            transition: all 0.3s ease;
        }

        .cell {
            width: var(--cell-size);
            height: var(--cell-size);
            background-color: var(--cell-bg);
            border-radius: 4px;
            display: flex;
            align-items: center;
            justify-content: center;
            font-weight: bold;
            font-size: 1.2rem;
            cursor: pointer;
            box-shadow: inset 0 3px 0 rgba(255,255,255,0.2), inset 0 -3px 0 rgba(0,0,0,0.2);
            transition: background-color 0.1s, transform 0.1s;
        }

        .cell:hover:not(.revealed) {
            background-color: #525969;
            transform: scale(1.02);
        }

        .cell.revealed {
            background-color: var(--cell-revealed);
            box-shadow: none;
            color: #1e1e24;
            border: 1px solid rgba(0,0,0,0.05);
        }

        .cell.flagged {
            background-color: #f39c12 !important;
        }

        .cell.mine {
            background-color: #e74c3c !important;
            animation: explode 0.5s ease-out;
        }

        /* Màu sắc cho các con số chỉ định lượng bom xung quanh */
        .n-1 { color: #1bc5bd; }
        .n-2 { color: #3699ff; }
        .n-3 { color: #f64e60; }
        .n-4 { color: #8950fc; }
        .n-5 { color: #ffa800; }
        .n-6, .n-7, .n-8 { color: #b5b5c3; }

        .btn {
            margin-top: 20px;
            padding: 10px 25px;
            font-size: 1rem;
            font-weight: bold;
            background-color: #ff4a5a;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            box-shadow: 0 4px 15px rgba(255, 74, 90, 0.4);
            transition: background 0.2s, transform 0.1s;
        }

        .btn:hover {
            background-color: #e03e4d;
            transform: translateY(-2px);
        }

        .btn:active {
            transform: translateY(0);
        }

        @keyframes explode {
            0% { transform: scale(1); background-color: #ff4a5a; }
            50% { transform: scale(1.2); background-color: #ffcc00; }
            100% { transform: scale(1); background-color: #e74c3c; }
        }
    </style>
</head>
<body>

    <h1>💣 DÒ MÌN SIÊU CẤP 💣</h1>
    
    <div class="stats-container">
        <div class="stat-box">Cấp độ: <span id="level">1</span></div>
        <div class="stat-box">Kích cỡ: <span id="grid-size">8x8</span></div>
        <div class="stat-box">Số bom: <span id="mine-count">10</span></div>
    </div>

    <div id="game-board"></div>

    <button class="btn" onclick="initGame()">Chơi Lại Cấp Độ Này</button>

    <script>
        // Cấu hình cấp độ ban đầu
        let currentLevel = 1;
        let rows = 8;
        let cols = 8;
        let mines = 10;

        let board = [];
        let gameOver = false;
        let revealedCount = 0;

        const boardElement = document.getElementById('game-board');

        // --- HỆ THỐNG ÂM THANH GIẢ LẬP (Web Audio API) ---
        const audioCtx = new (window.AudioContext || window.webkitAudioContext)();

        function playSound(type) {
            if (audioCtx.state === 'suspended') {
                audioCtx.resume();
            }
            
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.connect(gain);
            gain.connect(audioCtx.destination);

            const now = audioCtx.currentTime;

            if (type === 'click') {
                // Tiếng click nhẹ khi mở ô an toàn
                osc.type = 'sine';
                osc.frequency.setValueAtTime(400, now);
                osc.frequency.exponentialRampToValueAtTime(800, now + 0.05);
                gain.gain.setValueAtTime(0.1, now);
                gain.gain.linearRampToValueAtTime(0, now + 0.05);
                osc.start(now);
                osc.stop(now + 0.05);
            } else if (type === 'flag') {
                // Tiếng cắm cờ
                osc.type = 'triangle';
                osc.frequency.setValueAtTime(600, now);
                osc.frequency.setValueAtTime(300, now + 0.08);
                gain.gain.setValueAtTime(0.15, now);
                gain.gain.linearRampToValueAtTime(0, now + 0.15);
                osc.start(now);
                osc.stop(now + 0.15);
            } else if (type === 'explosion') {
                // Tiếng nổ bom lớn trầm
                osc.type = 'sawtooth';
                osc.frequency.setValueAtTime(120, now);
                osc.frequency.linearRampToValueAtTime(20, now + 0.5);
                gain.gain.setValueAtTime(0.4, now);
                gain.gain.linearRampToValueAtTime(0, now + 0.5);
                osc.start(now);
                osc.stop(now + 0.5);
            } else if (type === 'win') {
                // Giai điệu chiến thắng ngắn vui nhộn
                const notes = [523.25, 659.25, 783.99, 1046.50]; // Đồ - Mi - Sol - Đố
                notes.forEach((freq, index) => {
                    const noteOsc = audioCtx.createOscillator();
                    const noteGain = audioCtx.createGain();
                    noteOsc.connect(noteGain);
                    noteGain.connect(audioCtx.destination);
                    
                    noteOsc.frequency.setValueAtTime(freq, now + index * 0.1);
                    noteGain.gain.setValueAtTime(0.15, now + index * 0.1);
                    noteGain.gain.linearRampToValueAtTime(0, now + index * 0.1 + 0.2);
                    
                    noteOsc.start(now + index * 0.1);
                    noteOsc.stop(now + index * 0.1 + 0.2);
                });
            }
        }

        // --- KHỞI TẠO TRÒ CHƠI ---
        function initGame() {
            gameOver = false;
            revealedCount = 0;
            board = [];
            boardElement.innerHTML = '';

            // Cập nhật thông số hiển thị giao diện
            document.getElementById('level').innerText = currentLevel;
            document.getElementById('grid-size').innerText = `${rows}x${cols}`;
            document.getElementById('mine-count').innerText = mines;

            // Thiết lập grid CSS linh hoạt theo kích cỡ bảng
            boardElement.style.gridTemplateColumns = `repeat(${cols}, var(--cell-size))`;
            boardElement.style.gridTemplateRows = `repeat(${rows}, var(--cell-size))`;

            // 1. Tạo ma trận rỗng
            for (let r = 0; r < rows; r++) {
                board[r] = [];
                for (let c = 0; c < cols; c++) {
                    board[r][c] = {
                        row: r,
                        col: c,
                        isMine: false,
                        isRevealed: false,
                        isFlagged: false,
                        neighborMines: 0,
                        element: null
                    };
                }
            }

            // 2. Đặt bom ngẫu nhiên
            let minesPlaced = 0;
            while (minesPlaced < mines) {
                let r = Math.floor(Math.random() * rows);
                let c = Math.floor(Math.random() * cols);
                if (!board[r][c].isMine) {
                    board[r][c].isMine = true;
                    minesPlaced++;
                }
            }

            // 3. Tính toán số bom xung quanh mỗi ô
            for (let r = 0; r < rows; r++) {
                for (let c = 0; c < cols; c++) {
                    if (!board[r][c].isMine) {
                        board[r][c].neighborMines = countNeighborMines(r, c);
                    }
                }
            }

            // 4. Vẽ giao diện các ô vuông lên màn hình
            for (let r = 0; r < rows; r++) {
                for (let c = 0; c < cols; c++) {
                    const cellElement = document.createElement('div');
                    cellElement.classList.add('cell');
                    cellElement.dataset.row = r;
                    cellElement.dataset.col = c;

                    // Click chuột trái để mở ô
                    cellElement.addEventListener('click', () => handleCellClick(r, c));
                    
                    // Click chuột phải để cắm cờ
                    cellElement.addEventListener('contextmenu', (e) => {
                        e.preventDefault();
