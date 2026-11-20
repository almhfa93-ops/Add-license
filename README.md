<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>هجولة سعودية</title>
    <style>
        * {
            box-sizing: border-box;
            user-select: none;
            -webkit-user-select: none;
            touch-action: none;
        }
        body, html {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #d4a359;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        canvas {
            display: block;
        }
        #ui {
            position: absolute;
            top: 15px;
            left: 15px;
            right: 15px;
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            pointer-events: none;
            color: white;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
        }
        .score-box {
            background: rgba(0, 0, 0, 0.6);
            padding: 10px 18px;
            border-radius: 12px;
            border: 2px solid #f39c12;
        }
        .score-title { font-size: 14px; color: #f1c40f; }
        .score-val { font-size: 26px; font-weight: bold; }
        #drift-badge {
            position: absolute;
            top: 90px;
            left: 50%;
            transform: translateX(-50%);
            font-size: 24px;
            font-weight: bold;
            color: #ff4757;
            text-shadow: 2px 2px 0 #000;
            opacity: 0;
            transition: opacity 0.2s;
            pointer-events: none;
        }
        .top-btns {
            pointer-events: auto;
            display: flex;
            gap: 8px;
        }
        .top-btn {
            background: rgba(0,0,0,0.7);
            color: white;
            border: 1px solid #fff;
            padding: 8px 12px;
            border-radius: 8px;
            font-size: 14px;
            cursor: pointer;
        }
        #controls {
            position: absolute;
            bottom: 20px;
            left: 20px;
            right: 20px;
            display: flex;
            justify-content: space-between;
            align-items: flex-end;
            pointer-events: none;
        }
        .control-group {
            display: flex;
            gap: 15px;
            pointer-events: auto;
        }
        .btn {
            width: 70px;
            height: 70px;
            background: rgba(255, 255, 255, 0.2);
            border: 2px solid rgba(255, 255, 255, 0.6);
            border-radius: 50%;
            color: white;
            font-size: 24px;
            font-weight: bold;
            display: flex;
            align-items: center;
            justify-content: center;
            backdrop-filter: blur(4px);
        }
        .btn:active, .btn.active {
            background: rgba(243, 156, 18, 0.7);
            transform: scale(0.95);
        }
        .pedal {
            width: 65px;
            height: 95px;
            border-radius: 15px;
            font-size: 16px;
        }
        #btn-gas { background: rgba(46, 204, 113, 0.3); border-color: #2ecc71; }
        #btn-gas.active { background: rgba(46, 204, 113, 0.8); }
        #btn-brake { background: rgba(231, 76, 60, 0.3); border-color: #e74c3c; }
        #btn-brake.active { background: rgba(231, 76, 60, 0.8); }
    </style>
</head>
<body>

    <canvas id="gameCanvas"></canvas>

    <div id="ui">
        <div class="score-box">
            <div class="score-title">نقاط الهجولة</div>
            <div class="score-val" id="score">0</div>
        </div>
        <div class="top-btns">
            <button class="top-btn" onclick="changeColor()">🎨 لون الموتر</button>
            <button class="top-btn" onclick="resetCar()">🔄 تعديل</button>
        </div>
    </div>

    <div id="drift-badge">🔥 تفحيط! +<span id="drift-current">0</span></div>

    <div id="controls">
        <div class="control-group">
            <div class="btn" id="btn-left">◀</div>
            <div class="btn" id="btn-right">▶</div>
        </div>
        <div class="control-group">
            <div class="btn pedal" id="btn-brake">فرامل<br>جلنط</div>
            <div class="btn pedal" id="btn-gas">بنزين</div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        function resize() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', resize);
        resize();

        const colors = ['#ffffff', '#111111', '#c0392b', '#2980b9', '#f1c40f', '#7f8c8d'];
        let colorIdx = 0;

        const car = {
            x: 0,
            y: 0,
            width: 34,
            height: 64,
            angle: -Math.PI / 2,
            vx: 0,
            vy: 0,
            speed: 0,
            maxSpeed: 14,
            accel: 0.35,
            friction: 0.97,
            turnSpeed: 0.065,
            driftFactor: 0.94,
            color: colors[0]
        };

        let score = 0;
        let currentDriftScore = 0;
        let skidMarks = [];
        let smoke = [];
        let palms = [];

        for (let i = 0; i < 60; i++) {
            let px = (Math.random() - 0.5) * 4000;
            if (Math.abs(px) < 280) px += (px > 0 ? 350 : -350);
            palms.push({
                x: px,
                y: (Math.random() - 0.5) * 6000,
                size: 25 + Math.random() * 15
            });
        }

        const keys = { left: false, right: false, gas: false, brake: false };

        function bindTouch(id, key) {
            const el = document.getElementById(id);
            el.addEventListener('touchstart', (e) => { e.preventDefault(); keys[key] = true; el.classList.add('active'); });
            el.addEventListener('touchend', (e) => { e.preventDefault(); keys[key] = false; el.classList.remove('active'); });
            el.addEventListener('mousedown', () => { keys[key] = true; el.classList.add('active'); });
            el.addEventListener('mouseup', () => { keys[key] = false; el.classList.remove('active'); });
        }

        bindTouch('btn-left', 'left');
        bindTouch('btn-right', 'right');
        bindTouch('btn-gas', 'gas');
        bindTouch('btn-brake', 'brake');

        window.addEventListener('keydown', (e) => {
            if (e.key === 'ArrowLeft' || e.key === 'a') keys.left = true;
            if (e.key === 'ArrowRight' || e.key === 'd') keys.right = true;
            
