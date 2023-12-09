<template>
  <div id="container">
    <div class="buttons">
      <el-button @click="clear">清空</el-button>

      <el-radio-group v-model="state.type" size="large" @change="handleTypeChange">
        <el-radio-button label="draw"/>
        <el-radio-button label="clean"/>
      </el-radio-group>
    </div>

    <div class="box">
      <canvas id="oriCanvas"
              width="800"
              height="400" style=" z-index:4;"></canvas>

      <canvas
          style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index:3;opacity: 0.5;"
          id="canvas"
          width="800"
          height="400"
          @mousedown="handleMouseDown"
          @mousemove="handleMousemove"
          @mouseup="handleMouseup"
          @mouseout="handleMouseout">
      </canvas>

      <canvas
          style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index:1;display: none"
          id="maskCanvas"
          width="800"
          height="400"
      >
      </canvas>
    </div>
  </div>
</template>

<script setup lang="ts">
import {onMounted, reactive, ref} from "vue";


// 橡皮宽度
const eraserWidth = ref(16);

interface IState {
  type: string;
  isDraw: boolean;
  canvasDom?: HTMLCanvasElement | null;
  ctx?: CanvasRenderingContext2D | null;
  beginX: number;
  beginY: number;
  color: string;
  imageData: ImageData | null;
}

const state = reactive<IState>({
  type: "draw",
  isDraw: false,
  beginX: 0,
  beginY: 0,
  color: "#000",
  imageData: null,
});

let isDrawing = false;
let pointsRef: IObject[] = []
let beginPointRef: IObject

let canvas: HTMLCanvasElement;
let oriCanvas: HTMLCanvasElement;
let maskCanvas: HTMLCanvasElement;


let ctx: CanvasRenderingContext2D;
let oriCtx: CanvasRenderingContext2D;
let maskCtx: CanvasRenderingContext2D;

let ctxGroup: CanvasRenderingContext2D[];

onMounted(() => {
  canvas = document.getElementById("canvas") as HTMLCanvasElement;
  oriCanvas = document.getElementById("oriCanvas") as HTMLCanvasElement;
  maskCanvas = document.getElementById("maskCanvas") as HTMLCanvasElement;

  ctx = canvas.getContext("2d") as CanvasRenderingContext2D;
  oriCtx = oriCanvas.getContext("2d") as CanvasRenderingContext2D;
  maskCtx = maskCanvas.getContext("2d") as CanvasRenderingContext2D;

  const image = new Image();
  image.src = "https://yunchengzhuanyong.oss-cn-hangzhou.aliyuncs.com/xhh_ai/ai-server/20231206/aigc0ef64dc238ae17440e7bc62df1bef39e.png";

  image.onload = () => {
    if (!oriCanvas) return
    oriCtx.drawImage(image, 0, 0, 800, 400)
  }

  ctx.strokeStyle = 'rgb(241,110,110)'

  // 创建一个黑色背景
  maskCtx.strokeStyle = "black"; // 设置填充颜色为黑色
  maskCtx.fillRect(0, 0, canvas?.width as number, canvas?.height as number); // 填充整个Canvas
  maskCtx.strokeStyle = "white"; // 设置填充颜色为黑色


  ctxGroup = [ctx, maskCtx]
  setCtxStyle()
});

// 设置 ctx 公共样式
const setCtxStyle = () => {
  ctxGroup.forEach(ctx => {
    ctx.lineWidth = 15; // 设置线宽
    ctx.lineCap = 'round'; // 设置线帽为圆形
    ctx.lineJoin = 'round'; // 设置线帽为圆形
  });
};


interface IObject {
  x: number;
  y: number;
}

// 绘制路径
function handleMousemove(e: MouseEvent) {

  if (!isDrawing) return;
  const x = e.offsetX;
  const y = e.offsetY;
  pointsRef.push({x, y});

  if (pointsRef.length > 3) {
    const lastTwoPoints = pointsRef.slice(-2);
    const controlPoint = lastTwoPoints[0];
    const endPoint = {
      x: (lastTwoPoints[0].x + lastTwoPoints[1].x) / 2,
      y: (lastTwoPoints[0].y + lastTwoPoints[1].y) / 2,
    };
    drawLine(beginPointRef, controlPoint, endPoint);
    beginPointRef = endPoint;
  }
}

function drawLine(beginPoint: IObject, controlPoint: IObject, endPoint: IObject) {
  ctxGroup.forEach(ctx => {
    ctx.beginPath();
    ctx.moveTo(beginPoint.x, beginPoint.y);

    if (state.type == "clean") {
      ctx.clearRect(
          beginPoint.x - eraserWidth.value / 2,
          beginPoint.y - eraserWidth.value,
          eraserWidth.value,
          eraserWidth.value
      );
    } else {
      ctx.quadraticCurveTo(controlPoint.x, controlPoint.y, endPoint.x, endPoint.y);
      ctx.stroke();
    }


  });
}


const handleMouseDown = (e: MouseEvent) => {
  isDrawing = true;
  const x = e.offsetX;
  const y = e.offsetY;
  ctxGroup.forEach(ctx => {
    ctx.moveTo(x, y);
  });

  pointsRef.push({x, y});
  beginPointRef = {x, y};
};

const handleMouseup = () => {
  isDrawing = false;
  ctxGroup.forEach(ctx => {
    ctx.closePath();
  });

  const maskDataURL = maskCanvas.toDataURL();

// 输出图像数据URL
  console.log(maskDataURL);
};


const handleMouseout = () => {
  isDrawing = false;
};

const clear = () => {
  // 清空蒙版
  ctx.clearRect(0, 0, canvas?.width as number, canvas?.height as number);
  maskCtx.strokeStyle = "black"; // 设置填充颜色为黑色
  maskCtx.fillRect(0, 0, canvas?.width as number, canvas?.height as number);
  maskCtx.strokeStyle = "white"; // 设置填充颜色为黑色
};

let handleTypeChange =(type: string) => {
   console.log(type)
  switch (type) {
    case "draw":
      canvas.style.cursor = `url(${getImageByPath(
          "./assets/icon/pen.svg"
      )}) 6 26, auto`;
      break;
    case "clean":
      canvas.style.cursor = `url(${getImageByPath(
          "./assets/icon/eraser.svg"
      )}) 6 26, auto`;
      break;
    default:
      canvas.style.cursor = "default";
      break;
  }
};

const getImageByPath = (imgPath:string) => {
  return new URL(
      imgPath, import.meta.url
  ).href
}

</script>

<style>
.box {
  position: relative;
}

#container {
  display: flex;
  justify-content: center;
  align-items: flex-start;
}


.buttons {
  margin-bottom: 10px;
  display: flex;
}

#canvas {
  border: 1px solid black;
}
</style>