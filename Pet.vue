<template>
  <div class="pet pet-1" id="pet1" ref="petRef" :class="{ hidden: isHidden }">
    <div class="speech-bubble" id="bubble" ref="bubbleRef">{{ bubbleText }}</div>
    <img :src="petImage" id="img1" alt="pet" />
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import petImage from '@/../pet.png';

const petRef = ref(null);
const bubbleRef = ref(null);
const bubbleText = ref('');
const isHidden = ref(false);

let mouseX = 0;
let mouseY = 0;
let lastMouseX = 0;
let lastMouseY = 0;
let pet1 = null;
let animationId = null;
let bubbleTimer = null;
let hideTimer = null;
let lastMoveTime = 0;
let isSlidingIn = false;
let slideProgress = 0;

const STILL_THRESHOLD = 4000;
const SLIDE_DURATION = 800;


const name_words= '我是XXX';


function showBubble(text) {
  clearTimeout(bubbleTimer);
  bubbleText.value = text;
  setTimeout(() => {
    if (bubbleRef.value) {
      bubbleRef.value.classList.add('show');
    }
  }, 10);
  bubbleTimer = setTimeout(hideBubble, 2000);
}

function hideBubble() {
  if (bubbleRef.value) {
    bubbleRef.value.classList.remove('show');
  }
}

function updatePet(pet, targetX, targetY, el, imgEl) {
  const dist = Math.hypot(pet.x - targetX, pet.y - targetY);
  const farLimit = 500;
  const waitTime = 1.5 * 1000;
  const runSpeed = 6;
  const walkSpeed = 3;
  const stopDist = 150;
  const friction = 0.92;
  const inertiaFactor = 0.15;
  const lost_words= '糟了...跟丢了';
  const find_words= '原来你在这里';

  if (pet.isLost) {
    pet.vx *= friction;
    pet.vy *= friction;
    pet.x += pet.vx;
    pet.y += pet.vy;
    el.style.left = pet.x + 'px';
    el.style.top = pet.y + 'px';
    return;
  }

  if (dist > farLimit) {
    if (!pet.isWaiting && !pet.isRunning) {
      pet.isWaiting = true;
      clearTimeout(pet.waitTimer);
      pet.waitTimer = setTimeout(() => {
        pet.isWaiting = false;
        pet.isRunning = true;
        if (Date.now() - pet.lastBubbleTime > 3000) {
          showBubble(find_words);
          pet.lastBubbleTime = Date.now();
        }
      }, waitTime);
    }
  } else {
    if (pet.isWaiting) {
      clearTimeout(pet.waitTimer);
      pet.isWaiting = false;
    }
    if (pet.isRunning && dist < farLimit * 0.8) {
      pet.isRunning = false;
    }
  }

  if (!pet.isWaiting && dist > stopDist) {
    const speed = pet.isRunning ? runSpeed : walkSpeed;
    const angle = Math.atan2(targetY - pet.y, targetX - pet.x);

    const targetVx = Math.cos(angle) * speed;
    const targetVy = Math.sin(angle) * speed;

    pet.vx += (targetVx - pet.vx) * inertiaFactor;
    pet.vy += (targetVy - pet.vy) * inertiaFactor;

    const currentSpeed = Math.hypot(pet.vx, pet.vy);
    if (currentSpeed > speed) {
      pet.vx = (pet.vx / currentSpeed) * speed;
      pet.vy = (pet.vy / currentSpeed) * speed;
    }

    pet.x += pet.vx;
    pet.y += pet.vy;

    const currentDirection = Math.cos(angle) < 0 ? -1 : 1;
    if (currentDirection !== pet.lastDirection) {
      const now = Date.now();
      if (now - pet.lastFlipTime > 1000) {
        pet.flipCount = 0;
      }
      pet.flipCount++;
      pet.lastFlipTime = now;
      pet.lastDirection = currentDirection;

      if (pet.flipCount >= 3 && !pet.isLost) {
        pet.isLost = true;
        showBubble(lost_words);

        pet.lostTimer = setTimeout(() => {
          pet.isLost = false;
          pet.flipCount = 0;
        }, 2000);
      }
    }

    imgEl.style.transform = currentDirection < 0 ? 'scaleX(-1)' : 'scaleX(1)';
  } else if (dist <= stopDist) {
    pet.isRunning = false;
    pet.vx *= friction;
    pet.vy *= friction;
    pet.x += pet.vx;
    pet.y += pet.vy;

    const currentDirection = targetX < pet.x ? -1 : 1;
    if (currentDirection !== pet.lastDirection) {
      const now = Date.now();
      if (now - pet.lastFlipTime > 1000) {
        pet.flipCount = 0;
      }
      pet.flipCount++;
      pet.lastFlipTime = now;
      pet.lastDirection = currentDirection;

      if (pet.flipCount >= 3 && !pet.isLost) {
        pet.isLost = true;
        showBubble(lost_words);

        pet.lostTimer = setTimeout(() => {
          pet.isLost = false;
          pet.flipCount = 0;
        }, 2000);
      }
    }

    imgEl.style.transform = currentDirection < 0 ? 'scaleX(-1)' : 'scaleX(1)';
  }

  el.style.left = pet.x + 'px';
  el.style.top = pet.y + 'px';
}

function loop() {
  if (pet1) {
    checkMouseStationary();

    if (isSlidingIn) {
      slideProgress += 16 / SLIDE_DURATION;
      if (slideProgress >= 1) {
        slideProgress = 1;
        isSlidingIn = false;
      }
      const eased = 1 - Math.pow(1 - slideProgress, 3);
      const targetX = mouseX;
      const targetY = mouseY;
      pet1.x = -100 + (targetX + 100) * eased;
      pet1.y = targetY;
      if (petRef.value) {
        petRef.value.style.left = pet1.x + 'px';
        petRef.value.style.top = pet1.y + 'px';
      }
      const dist = Math.hypot(pet1.x - targetX, pet1.y - targetY);
      if (dist < 150) {
        isSlidingIn = false;
        pet1.isRunning = false;
      }
    } else if (!isHidden.value) {
      updatePet(pet1, mouseX, mouseY, petRef.value, document.getElementById('img1'));
    }
  }
  animationId = requestAnimationFrame(loop);
}

function handleMouseMove(e) {
  mouseX = e.clientX;
  mouseY = e.clientY;
}

function checkMouseStationary() {
  const now = Date.now();
  if (mouseX === lastMouseX && mouseY === lastMouseY) {
    if (now - lastMoveTime > STILL_THRESHOLD && !isHidden.value && !isSlidingIn) {
      isHidden.value = true;
    }
  } else {
    if (lastMoveTime > 0 && isHidden.value && !isSlidingIn) {
      isSlidingIn = true;
      slideProgress = 0;
      isHidden.value = false;
    }
    lastMoveTime = now;
    lastMouseX = mouseX;
    lastMouseY = mouseY;
  }
}

onMounted(() => {
  mouseX = window.innerWidth / 2;
  mouseY = window.innerHeight / 2;

  pet1 = {
    x: window.innerWidth - 100,
    y: 150,
    vx: 0,
    vy: 0,
    waitTimer: null,
    isWaiting: false,
    isRunning: false,
    isLost: false,
    lostTimer: null,
    lastBubbleTime: 0,
    bubbleTimer: null,
    lastFlipTime: 0,
    flipCount: 0,
    flipTrackTimer: null,
    lastDirection: 1
  };

  document.addEventListener('mousemove', handleMouseMove);
  loop();

  setTimeout(() => {
    showBubble(name_words);
  }, 500);
});

onUnmounted(() => {
  document.removeEventListener('mousemove', handleMouseMove);
  if (animationId) {
    cancelAnimationFrame(animationId);
  }
  clearTimeout(bubbleTimer);
  clearTimeout(hideTimer);
});
</script>

<style scoped>
.pet {
  position: fixed;
  pointer-events: none;
  z-index: 99999;
  transform: translate(-50%, -50%);
  will-change: left, top;
  transition: opacity 1s ease;
}

.pet.hidden {
  opacity: 0;
}

.pet-1 {
  width: 150px;
  height: 150px;
}

.pet img {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.speech-bubble {
  position: absolute;
  top: -45px;
  left: 50%;
  color: black;
  transform: translateX(-50%);
  background: #fff;
  padding: 6px 12px;
  border-radius: 12px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.15);
  font-size: 14px;
  white-space: nowrap;
  opacity: 0;
  transition: opacity 0.3s ease;
  z-index: 99999;
}

.speech-bubble.show {
  opacity: 1;
}

.speech-bubble::after {
  content: '';
  position: absolute;
  bottom: -6px;
  left: 50%;
  transform: translateX(-50%);
  border-left: 6px solid transparent;
  border-right: 6px solid transparent;
  border-top: 6px solid #fff;
}
</style>
