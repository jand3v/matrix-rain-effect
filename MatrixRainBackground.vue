<script setup>
import { onMounted, onBeforeUnmount, ref, watch } from "vue";

const props = defineProps({
  /**
   * Aktiviert Cyberpunk Neon Farben
   */
  cyberpunk: {
    type: Boolean,
    default: false,
  },

  /**
   * Aktiviert Mausinteraktion
   */
  mouseInteraction: {
    type: Boolean,
    default: false,
  },

  /**
   * Aktiviert echten Film-Look
   * (weiße führende Zeichen)
   */
  movieMode: {
    type: Boolean,
    default: true,
  },

  /**
   * Zeichengröße
   */
  fontSize: {
    type: Number,
    default: 16,
  },

  /**
   * Trail Stärke
   * Kleinere Werte = längere Trails
   */
  trailOpacity: {
    type: Number,
    default: 0.08,
  },

  /**
   * Geschwindigkeit
   */
  speed: {
    type: Number,
    default: 1,
  },
});

const canvasRef = ref(null);

let ctx;
let animationFrame;

let width;
let height;

let columns = [];
let mouseX = -9999;
let mouseY = -9999;

const chars =
  "アイウエオカキクケコサシスセソタチツテトナニヌネノABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";

const letters = chars.split("");

function createColumn(x) {
  return {
    x,
    y: Math.random() * -500,
    speed: (0.5 + Math.random() * 1.5) * props.speed,
    length: 8 + Math.floor(Math.random() * 25),
  };
}

function resizeCanvas() {
  const canvas = canvasRef.value;

  width = canvas.width = window.innerWidth;
  height = canvas.height = window.innerHeight;

  const totalColumns = Math.floor(width / props.fontSize);

  columns = [];

  for (let i = 0; i < totalColumns; i++) {
    columns.push(createColumn(i * props.fontSize));
  }
}

function getColors() {
  if (props.cyberpunk) {
    return {
      glow: "#ff00ff",
      primary: "#ff4dff",
      secondary: "#00f7ff",
      head: "#ffffff",
    };
  }

  return {
    glow: "#00ff99",
    primary: "#00ff99",
    secondary: "#00cc77",
    head: "#ffffff",
  };
}

function drawCharacter(char, x, y, opacity, isHead) {
  const colors = getColors();

  ctx.shadowColor = colors.glow;
  ctx.shadowBlur = isHead ? 18 : 8;

  if (isHead && props.movieMode) {
    ctx.fillStyle = `rgba(255,255,255,${opacity})`;
  } else {
    ctx.fillStyle = props.cyberpunk
      ? `rgba(255,0,255,${opacity})`
      : `rgba(0,255,140,${opacity})`;
  }

  ctx.fillText(char, x, y);
}

function applyMouseEffect(column) {
  if (!props.mouseInteraction) return 1;

  const dx = column.x - mouseX;
  const dy = column.y * props.fontSize - mouseY;

  const distance = Math.sqrt(dx * dx + dy * dy);

  const radius = 200;

  if (distance < radius) {
    return 2.5;
  }

  return 1;
}

function draw() {
  ctx.fillStyle = `rgba(5, 8, 13, ${props.trailOpacity})`;
  ctx.fillRect(0, 0, width, height);

  ctx.font = `${props.fontSize}px monospace`;
  ctx.textBaseline = "top";

  columns.forEach((column) => {
    const mouseBoost = applyMouseEffect(column);

    for (let i = 0; i < column.length; i++) {
      const char =
        letters[Math.floor(Math.random() * letters.length)];

      const x = column.x;
      const y = (column.y - i) * props.fontSize;

      if (y < 0 || y > height) continue;

      const opacity = 1 - i / column.length;

      const isHead = i === 0;

      drawCharacter(char, x, y, opacity, isHead);
    }

    column.y += column.speed * mouseBoost;

    if (column.y * props.fontSize > height + 200) {
      column.y = Math.random() * -100;
      column.speed =
        (0.5 + Math.random() * 1.5) * props.speed;
      column.length = 8 + Math.floor(Math.random() * 25);
    }
  });

  animationFrame = requestAnimationFrame(draw);
}

function handleMouseMove(e) {
  mouseX = e.clientX;
  mouseY = e.clientY;
}

onMounted(() => {
  const canvas = canvasRef.value;

  ctx = canvas.getContext("2d");

  resizeCanvas();

  draw();

  window.addEventListener("resize", resizeCanvas);

  if (props.mouseInteraction) {
    window.addEventListener("mousemove", handleMouseMove);
  }
});

watch(
  () => props.mouseInteraction,
  (enabled) => {
    if (enabled) {
      window.addEventListener("mousemove", handleMouseMove);
    } else {
      window.removeEventListener(
        "mousemove",
        handleMouseMove
      );
    }
  }
);

onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrame);

  window.removeEventListener("resize", resizeCanvas);

  window.removeEventListener(
    "mousemove",
    handleMouseMove
  );
});
</script>

<template>
  <canvas
    ref="canvasRef"
    class="fixed inset-0 w-full h-full pointer-events-none -z-10 bg-[#05080d]"
  />
</template>