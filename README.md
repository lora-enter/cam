# cam
https://majestic-bienenstitch-44a7c4.netlify.app/
давай сделаем проект на джаваскрипте и на штмл. что бы через камеру компьютера он определял если один указательный палец показываю то я могу рисовать(белым) на изображении камеры. если как бы хватаю рисунок двумя пальцами то я смог его перетащить если двумя руками беру с разных концов то я смог или увеличивать либо уменьшать, а если я показываю полностью ладошку то  я могу стирать где провожу ладошкой и придумай название для этого проекта
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>AirCanvas</title>

  <link rel="stylesheet" href="style.css"/>

  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
</head>
<body>

  <video id="video" autoplay playsinline></video>

  <canvas id="drawCanvas"></canvas>

  <div class="info">
    <h1>AirCanvas</h1>
    <p>☝ Один палец — рисование</p>
    <p>✋ Ладонь — стирание</p>
    <p>🤏 Щипок — перетаскивание</p>
    <p>👐 Две руки — масштаб</p>
  </div>

  <script src="script.js"></script>

</body>
</html>

*{
  margin:0;
  padding:0;
  box-sizing:border-box;
}

body{
  overflow:hidden;
  background:black;
  font-family:Arial, sans-serif;
}

video{
  position:absolute;
  width:100vw;
  height:100vh;
  object-fit:cover;
  transform: scaleX(-1);
}

canvas{
  position:absolute;
  width:100vw;
  height:100vh;
  pointer-events:none;
}

.info{
  position:absolute;
  top:20px;
  left:20px;
  z-index:10;
  color:white;
  background:rgba(0,0,0,0.5);
  padding:15px;
  border-radius:15px;
  backdrop-filter: blur(10px);
}

.info h1{
  margin-bottom:10px;
}

const video = document.getElementById("video");
const canvas = document.getElementById("drawCanvas");
const ctx = canvas.getContext("2d");

canvas.width = window.innerWidth;
canvas.height = window.innerHeight;

let drawing = false;

let lastX = 0;
let lastY = 0;

let drawLayer = document.createElement("canvas");
drawLayer.width = canvas.width;
drawLayer.height = canvas.height;

let drawCtx = drawLayer.getContext("2d");

let offsetX = 0;
let offsetY = 0;

let scale = 1;

let dragging = false;

let startDragX = 0;
let startDragY = 0;

let initialDistance = null;

navigator.mediaDevices.getUserMedia({
  video: true
})
.then(stream => {
  video.srcObject = stream;
});

function distance(a, b){
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function drawLine(x1, y1, x2, y2){

  drawCtx.strokeStyle = "white";
  drawCtx.lineWidth = 5;
  drawCtx.lineCap = "round";

  drawCtx.beginPath();
  drawCtx.moveTo(x1, y1);
  drawCtx.lineTo(x2, y2);
  drawCtx.stroke();
}

function erase(x, y){

  drawCtx.save();

  drawCtx.globalCompositeOperation = "destination-out";

  drawCtx.beginPath();
  drawCtx.arc(x, y, 40, 0, Math.PI * 2);

  drawCtx.fill();

  drawCtx.restore();
}

function isFingerUp(hand){

  return hand[8].y < hand[6].y;
}

function isMiddleDown(hand){

  return hand[12].y > hand[10].y;
}

function isOpenPalm(hand){

  return (
    hand[8].y < hand[6].y &&
    hand[12].y < hand[10].y &&
    hand[16].y < hand[14].y &&
    hand[20].y < hand[18].y
  );
}

function pinch(hand){

  let thumb = hand[4];
  let index = hand[8];

  return distance(thumb, index) < 0.05;
}

const hands = new Hands({
  locateFile: (file) => {
    return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
  }
});

hands.setOptions({
  maxNumHands: 2,
  modelComplexity: 1,
  minDetectionConfidence: 0.7,
  minTrackingConfidence: 0.7
});

hands.onResults(results => {

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  ctx.save();

  ctx.translate(canvas.width / 2, canvas.height / 2);
  ctx.scale(scale, scale);
  ctx.translate(-canvas.width / 2 + offsetX, -canvas.height / 2 + offsetY);

  ctx.drawImage(drawLayer, 0, 0);

  ctx.restore();

  if(results.multiHandLandmarks){

    if(results.multiHandLandmarks.length === 1){

      let hand = results.multiHandLandmarks[0];

      let index = hand[8];

      let x = canvas.width - (index.x * canvas.width);
      let y = index.y * canvas.height;

      // DRAW MODE

      if(isFingerUp(hand) && isMiddleDown(hand)){

        if(!drawing){
          drawing = true;
          lastX = x;
          lastY = y;
        }

        drawLine(lastX, lastY, x, y);

        lastX = x;
        lastY = y;
      }
      else{
        drawing = false;
      }

      // ERASE MODE

      if(isOpenPalm(hand)){

        erase(x, y);
      }

      // DRAG MODE

      if(pinch(hand)){

        if(!dragging){

          dragging = true;

          startDragX = x;
          startDragY = y;
        }

        let dx = x - startDragX;
        let dy = y - startDragY;

        offsetX += dx;
        offsetY += dy;

        startDragX = x;
        startDragY = y;
      }
      else{
        dragging = false;
      }

    }

    // ZOOM WITH TWO HANDS

    if(results.multiHandLandmarks.length === 2){

      let hand1 = results.multiHandLandmarks[0];
      let hand2 = results.multiHandLandmarks[1];

      let p1 = hand1[8];
      let p2 = hand2[8];

      let dist = distance(p1, p2);

      if(initialDistance === null){

        initialDistance = dist;
      }

      let zoomFactor = dist / initialDistance;

      scale = Math.min(Math.max(zoomFactor, 0.5), 3);

    }
    else{
      initialDistance = null;
    }

  }

});

const camera = new Camera(video, {
  onFrame: async () => {
    await hands.send({ image: video });
  },
  width: 1280,
  height: 720
});

camera.start();