<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Skeletal Centipede Follower</title>
  <style>
    html,body{margin:0;height:100%;overflow:hidden;background:#111;}
    canvas{display:block;touch-action:none;}
  </style>
</head>
<body>
<canvas id="c"></canvas>

<script>
(() => {
  const canvas = document.getElementById("c");
  const ctx = canvas.getContext("2d");

  function resize(){
    const dpr = Math.max(1, devicePixelRatio||1);
    canvas.width  = Math.floor(innerWidth * dpr);
    canvas.height = Math.floor(innerHeight * dpr);
    canvas.style.width  = innerWidth + "px";
    canvas.style.height = innerHeight + "px";
    ctx.setTransform(dpr,0,0,dpr,0,0);
  }
  addEventListener("resize", resize);
  resize();

  // 목표
  const target = { x: innerWidth/2, y: innerHeight/2 };
  addEventListener("pointermove", e => { target.x=e.clientX; target.y=e.clientY; }, {passive:true});
  addEventListener("pointerdown", e => { target.x=e.clientX; target.y=e.clientY; }, {passive:true});

  // ===== 이동(자기 속도 Arrive) =====
  const MAX_SPEED = 100;
  const MAX_ACCEL = 750;
  const ARRIVE_RADIUS = 220;
  const STOP_RADIUS = 6;
  const headVel = { x: 0, y: 0 };

  // ===== 몸통(척추) =====
  const N = 85;            // 마디 많게(이미지 느낌 핵심)
  const SPACING = 7.5;     // 촘촘히
  const FADE = 26;       // 잔상 길게(낮을수록 길어짐)

  const pts = Array.from({length:N}, (_,i)=>({
    x: target.x - i*SPACING,
    y: target.y
  }));

  // ===== 라인아트 스타일 =====
  const LINE_ALPHA = 0.22;     // 전체 선 투명도
  const SPINE_W = 1.2;         // 척추 굵기
  const RIB_W = 0.9;           // 갈비뼈 굵기
  const LEG_W = 0.9;           // 다리 굵기
  const LEG_LEN = 36;          // 다리 길이(길게)
  const LEG_JOINT1 = 0.55;     // 다리 관절 비율
  const LEG_WAVE = 0.9;        // 다리 꿈틀
  const LEG_WAVE_SPEED = 5.2;  // 다리 파동 속도

  // 보조 함수
  function clamp(v,a,b){ return Math.max(a, Math.min(b, v)); }

  function step(){
    const t = performance.now()*0.001;

    // 잔상
    ctx.fillStyle = `rgba(17,17,17,${FADE})`;
    ctx.fillRect(0,0,innerWidth,innerHeight);

    // ---- 머리 Arrive ----
    const hx = pts[0].x, hy = pts[0].y;
    let dx = target.x - hx, dy = target.y - hy;
    const dist = Math.hypot(dx, dy) || 0;

    let desiredSpeed = MAX_SPEED;
    if (dist < ARRIVE_RADIUS) desiredSpeed = MAX_SPEED * (dist / ARRIVE_RADIUS);
    if (dist < STOP_RADIUS) desiredSpeed = 0;

    let desiredVx = 0, desiredVy = 0;
    if (dist > 1e-6 && desiredSpeed > 0){
      desiredVx = (dx/dist) * desiredSpeed;
      desiredVy = (dy/dist) * desiredSpeed;
    }

    let ax = desiredVx - headVel.x;
    let ay = desiredVy - headVel.y;
    const aLen = Math.hypot(ax, ay) || 0;
    const maxA = MAX_ACCEL * (1/60);
    if (aLen > maxA){
      const s = maxA / aLen;
      ax *= s; ay *= s;
    }

    headVel.x += ax;
    headVel.y += ay;

    const vLen = Math.hypot(headVel.x, headVel.y) || 0;
    if (vLen > MAX_SPEED){
      const s = MAX_SPEED / vLen;
      headVel.x *= s; headVel.y *= s;
    }

    pts[0].x += headVel.x * (1/60);
    pts[0].y += headVel.y * (1/60);

    // ---- 몸통 간격 유지 ----
    for(let i=1;i<N;i++){
      const prev = pts[i-1], p = pts[i];
      const vx = p.x - prev.x;
      const vy = p.y - prev.y;
      const d = Math.hypot(vx,vy) || 1;
      const tx = prev.x + (vx/d)*SPACING;
      const ty = prev.y + (vy/d)*SPACING;
      p.x += (tx - p.x) * 0.55;
      p.y += (ty - p.y) * 0.55;
    }

    // ---- 방향(진행방향) ----
    let dirA = Math.atan2(headVel.y, headVel.x);
    if (Math.hypot(headVel.x, headVel.y) < 1e-3) {
      dirA = Math.atan2(pts[0].y - pts[1].y, pts[0].x - pts[1].x);
    }
    const dirX = Math.cos(dirA), dirY = Math.sin(dirA);
    const nX = -dirY, nY = dirX;

    // 공통 색
    const col = `rgba(220,220,220,${LINE_ALPHA})`;
    ctx.strokeStyle = col;
    ctx.fillStyle = col;

    // ---- 척추(스파인) ----
    ctx.lineWidth = SPINE_W;
    ctx.beginPath();
    ctx.moveTo(pts[0].x, pts[0].y);
    for (let i=1; i<N-1; i++){
      const cx = (pts[i].x + pts[i+1].x)/2;
      const cy = (pts[i].y + pts[i+1].y)/2;
      ctx.quadraticCurveTo(pts[i].x, pts[i].y, cx, cy);
    }
    ctx.stroke();

    // ---- 갈비뼈 + 다리(촘촘히) ----
    // 여러 마디에 얇은 선을 좌우로 뻗으면 저 이미지 느낌이 확 남
    for (let i=6; i<N-6; i++){
      const p = pts[i];
      const k = i/(N-1);

      // 마디별 진행 방향(몸 굽힘 반영)
      const prev = pts[i-1], next = pts[i+1];
      const a = Math.atan2(next.y - prev.y, next.x - prev.x);
      const nx = -Math.sin(a);
      const ny =  Math.cos(a);

      // 갈비뼈 길이: 머리쪽 길고 꼬리쪽 짧게
      const ribLen = (1-k) * 22 + 6;

      ctx.lineWidth = RIB_W;
      ctx.beginPath();
      ctx.moveTo(p.x - nx * ribLen, p.y - ny * ribLen);
      ctx.lineTo(p.x + nx * ribLen, p.y + ny * ribLen);
      ctx.stroke();

      // 다리는 너무 다 달면 과하니까 2~3마디마다
      if (i % 3 !== 0) continue;

      // 다리 길이: 앞쪽 약간 더 길게
      const L = LEG_LEN * (0.95 + (1-k)*0.25);

      // 다리 “파동”으로 살아있는 느낌(스텝락 대신 라인아트)
      const wave = Math.sin(t*LEG_WAVE_SPEED - i*0.35) * LEG_WAVE;

      // 좌/우 다리 (2관절처럼 꺾어서 그리기)
      for (const side of [-1, +1]){
        const hipX = p.x + nx * side * (ribLen*0.55);
        const hipY = p.y + ny * side * (ribLen*0.55);

        // 다리 방향: 바깥쪽 + 약간 앞쪽
        const outX = nx * side;
        const outY = ny * side;

        const fwdX = Math.cos(a);
        const fwdY = Math.sin(a);

        const kneeX = hipX + (outX * L * LEG_JOINT1) + (fwdX * 10) + outX * wave * 4;
        const kneeY = hipY + (outY * L * LEG_JOINT1) + (fwdY * 10) + outY * wave * 4;

        const footX = hipX + (outX * L) + (fwdX * 18) + outX * wave * 7;
        const footY = hipY + (outY * L) + (fwdY * 18) + outY * wave * 7;

        ctx.lineWidth = LEG_W;
        ctx.beginPath();
        ctx.moveTo(hipX, hipY);
        ctx.lineTo(kneeX, kneeY);
        ctx.lineTo(footX, footY);
        ctx.stroke();

        // 발 끝 작은 갈퀴 느낌(짧은 3가닥)
        const claw = 6;
        const ca = Math.atan2(footY - kneeY, footX - kneeX);
        for (let s=-1; s<=1; s++){
          const aa = ca + s*0.35;
          ctx.beginPath();
          ctx.moveTo(footX, footY);
          ctx.lineTo(footX + Math.cos(aa)*claw, footY + Math.sin(aa)*claw);
          ctx.stroke();
        }
      }
    }

    // 머리쪽 작은 “턱/집게” 느낌(선 두 개)
    const head = pts[0];
    ctx.lineWidth = 1.0;
    ctx.beginPath();
    ctx.moveTo(head.x + nX*6, head.y + nY*6);
    ctx.lineTo(head.x + dirX*16 + nX*14, head.y + dirY*16 + nY*14);
    ctx.moveTo(head.x - nX*6, head.y - nY*6);
    ctx.lineTo(head.x + dirX*16 - nX*14, head.y + dirY*16 - nY*14);
    ctx.stroke();

    requestAnimationFrame(step);
  }

  // 초기 배경
  ctx.fillStyle="#111";
  ctx.fillRect(0,0,innerWidth,innerHeight);
  step();
})();
</script>
</body>
</html>
