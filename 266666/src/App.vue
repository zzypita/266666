<template>
  <div class="study-timer">
    <div class="status-bar">
      <div class="stat-item">🍎 蘋果: {{ apples }}</div>
    </div>

    <div class="tree-area">
      <div v-if="activeInsect" class="insect-companion">
        <div class="dialogue-bubble">{{ insectQuote }}</div>
        <div class="insect-pixel" :title="activeInsect.name">{{ activeInsect.emoji }}</div>
      </div>

      <div class="tree">
        <!-- 使用本地 src/assets 中的圖片 -->
        <img src="./assets/download-removebg-preview.png" alt="Study Tree" class="tree-img">
        
        <div class="leaves-pixel">
            <div
              v-for="(leaf, idx) in leaves"
              :key="idx"
              class="leaf-pixel"
              :style="{
                left: leaf.left + 'px',
                top: (leaf.isFalling ? 350 : leaf.top) + 'px',
                opacity: leaf.isFalling ? 0 : 1,
                transform: `rotate(${leaf.rotate}deg)`,
                transition: leaf.isFalling ? 'top 2s cubic-bezier(0.6, 0.04, 0.98, 0.33), opacity 2s' : 'none'
              }"
            ></div>
        </div>
      </div>

      <!-- 將科目按鈕移至樹木旁邊 -->
      <div class="subjects">
        <template v-if="!subjectLocked">
          <button
            v-for="subject in subjects"
            :key="subject"
            class="subject-btn"
            @click="lockSubject(subject)"
          >
            {{ subject }}
          </button>
        </template>
        <template v-else>
          <span class="subject-btn active">{{ selectedSubject }}</span>
          <button class="change-btn" @click="unlockSubject">更換</button>
        </template>
      </div>
    </div>

    <div class="settings">
      <label>設定時間：</label>
      <button @click="decreaseTime" :disabled="timerRunning">-</button>
      <span>{{ studyMinutes }} 分鐘</span>
      <button @click="increaseTime" :disabled="timerRunning">+</button>
    </div>

    <div class="controls">
      <button @click="startTimer" :disabled="timerRunning || !subjectLocked">開始</button>
      <button @click="pauseTimer" :disabled="!timerRunning">暫停</button>
    </div>

    <div class="timer-display">
      <span>{{ displayTime }}</span>
    </div>

    <div class="shop">
      <h3>昆蟲商店</h3>
      <div class="shop-items">
        <div v-for="item in insectShop" :key="item.id" class="shop-card">
          <span class="shop-icon">{{ item.emoji }}</span>
          <span class="shop-name">{{ item.name }} ({{ item.price }}🍎)</span>
          <button 
            @click="buyInsect(item)" 
            :disabled="apples < item.price || ownedInsects.includes(item.id)"
          >
            {{ ownedInsects.includes(item.id) ? '已擁有' : '兌換' }}
          </button>
        </div>
      </div>
    </div>

    <div v-if="showPauseModal" class="modal-overlay">
      <div class="modal bee-modal">
        <div class="bee-pixel">🐝</div>
        <p>嗡嗡！要結束這次計時嗎？</p>
        <div class="modal-buttons">
          <button @click="handlePauseChoice(true)">是</button>
          <button @click="handlePauseChoice(false)">否</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'App',
  data() {
    return {
      subjects: ['國文', '英文', '數學', '公民', '地理', '歷史', '生物', '化學', '物理', '地科'],
      selectedSubject: '',
      subjectLocked: false,
      studyMinutes: 25,
      timer: null,
      timerRunning: false,
      timeLeft: 25 * 60,
      totalStartTime: 25 * 60,
      leaves: [],
      branchArea: { width: 180, height: 100 },
      apples: parseInt(localStorage.getItem('study_apples') || '0'),
      ownedInsects: JSON.parse(localStorage.getItem('study_insects') || '[]'),
      insectShop: [
        { id: 'butterfly', name: '蝴蝶', price: 50, emoji: '🦋' },
        { id: 'worm', name: '瓢蟲', price: 100, emoji: '🐞' },
        { id: 'beetle', name: '甲蟲', price: 150, emoji: '🪲' },
        { id: 'dragonfly', name: '蚱蜢', price: 200, emoji: '🦗' },
        { id: 'ant', name: '螞蟻', price: 250, emoji: '🐜' }
      ],
      showPauseModal: false,
      activeInsect: null,
      insectQuote: '加油！',
      quoteInterval: null
    };
  },
  computed: {
    displayTime() {
      const min = Math.floor(this.timeLeft / 60).toString().padStart(2, '0');
      const sec = (this.timeLeft % 60).toString().padStart(2, '0');
      return `${min}:${sec}`;
    },
    initialLeafCount() {
      return Math.floor(this.studyMinutes / 5);
    }
  },
  watch: {
    studyMinutes(newVal) {
      if (!this.timerRunning) {
        this.timeLeft = newVal * 60;
        this.totalStartTime = newVal * 60;
        this.generateLeaves();
      }
    },
    apples(newVal) {
      localStorage.setItem('study_apples', newVal);
    }
  },
  mounted() {
    this.generateLeaves();
  },
  methods: {
    lockSubject(subject) {
      this.selectedSubject = subject;
      this.subjectLocked = true;
    },
    unlockSubject() {
      this.subjectLocked = false;
      this.selectedSubject = '';
    },
    increaseTime() {
      if (this.studyMinutes < 180) {
        this.studyMinutes += 5;
      }
    },
    decreaseTime() {
      if (this.studyMinutes > 5) {
        this.studyMinutes -= 5;
      }
    },
    startTimer() {
      if (this.timerRunning || !this.subjectLocked) return;
      this.timerRunning = true;
      
      if (this.ownedInsects.length > 0) {
        const randId = this.ownedInsects[Math.floor(Math.random() * this.ownedInsects.length)];
        this.activeInsect = this.insectShop.find(i => i.id === randId);
        this.startQuotes();
      }

      this.timer = setInterval(() => {
        if (this.timeLeft > 0) {
          this.timeLeft--;
          this.updateLeavesFalling();
        } else {
          this.finishSession();
        }
      }, 1000);
    },
    updateLeavesFalling() {
      const elapsedSeconds = this.totalStartTime - this.timeLeft;
      const shouldHaveDropped = Math.floor(elapsedSeconds / 300);
      for (let i = 0; i < shouldHaveDropped && i < this.leaves.length; i++) {
        this.leaves[i].isFalling = true;
      }
    },
    pauseTimer() {
      if (!this.timerRunning) return;
      this.stopTimer();
      this.showPauseModal = true;
    },
    handlePauseChoice(shouldEnd) {
      this.showPauseModal = false;
      if (shouldEnd) {
        this.finishSession();
      } else {
        this.startTimer();
      }
    },
    finishSession() {
      const elapsedMinutes = Math.floor((this.totalStartTime - this.timeLeft) / 60);
      const earned = Math.floor(elapsedMinutes / 5);
      this.apples += earned;
      
      this.stopTimer();
      alert(`辛苦了！本次獲得 ${earned} 顆蘋果 🍎`);
      
      this.timeLeft = this.studyMinutes * 60;
      this.totalStartTime = this.timeLeft;
      this.activeInsect = null;
      this.generateLeaves();
    },
    stopTimer() {
      this.timerRunning = false;
      clearInterval(this.timer);
      this.timer = null;
      clearInterval(this.quoteInterval);
    },
    buyInsect(item) {
      if (this.apples >= item.price && !this.ownedInsects.includes(item.id)) {
        this.apples -= item.price;
        this.ownedInsects.push(item.id);
        localStorage.setItem('study_insects', JSON.stringify(this.ownedInsects));
      }
    },
    startQuotes() {
      const texts = ['加油', '辛苦了', '快結束了'];
      this.insectQuote = texts[Math.floor(Math.random() * texts.length)];
      this.quoteInterval = setInterval(() => {
        this.insectQuote = texts[Math.floor(Math.random() * texts.length)];
      }, 10000);
    },
    generateLeaves() {
      const count = this.initialLeafCount;
      const areaW = this.branchArea.width;
      const areaH = this.branchArea.height;
      const size = 24;
      const padding = 4;
      let newLeaves = [];
      for (let i = 0; i < count; i++) {
        let placed = false;
        let tries = 0;
        while (!placed && tries < 50) {
          const left = Math.random() * (areaW - size);
          const top = Math.random() * (areaH - size);
          const rotate = Math.floor(Math.random() * 360);
          let overlap = false;
          for (const leaf of newLeaves) {
            if (Math.abs(leaf.left - left) < size + padding && Math.abs(leaf.top - top) < size + padding) {
              overlap = true;
              break;
            }
          }
          if (!overlap) {
            newLeaves.push({ left, top, rotate, isFalling: false });
            placed = true;
          }
          tries++;
        }
      }
      this.leaves = newLeaves;
    }
  }
}
</script>

<style scoped>
.study-timer {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: 'Courier New', Courier, monospace;
  background-color: #f0f8ff;
  min-height: 100vh;
  padding: 20px;
}
.status-bar {
  width: 100%;
  max-width: 400px;
  display: flex;
  justify-content: flex-end;
  padding: 10px;
  font-weight: bold;
  color: #d84315;
}
.tree-area {
  position: relative;
  margin-top: 50px;
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.tree {
  position: relative;
  width: 320px;
  height: 400px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
.tree-img {
  width: 280px;
  height: auto;
  image-rendering: pixelated;
  z-index: 1;
}

.leaves-pixel {
  position: absolute;
  top: 55px; /* 調整至圖片樹冠位置，往下移動 10 單位 */
  left: 50%;
  transform: translateX(-50%);
  width: 180px;
  height: 100px;
  z-index: 3;
  pointer-events: none;
}
.leaf-pixel {
  position: absolute;
  width: 20px;
  height: 20px;
  background-image: url('./assets/download-removebg-preview (1).png');
  background-size: contain;
  background-repeat: no-repeat;
  image-rendering: auto; /* 寫實風格圖片使用 auto 渲染效果較佳 */
}

.subject-btn {
  font-size: 11px;
  padding: 4px 10px;
  background: #fffbe7;
  border: 1px solid #333;
  cursor: pointer;
}
.subject-btn.active { background: #ffd54f; font-weight: bold; }

.subjects {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-left: 40px; /* 與樹木保持距離 */
  z-index: 5;
  background: rgba(255, 255, 255, 0.6);
  padding: 10px;
  border-radius: 8px;
  border: 1px dashed #ccc;
  min-width: 80px;
}

.insect-companion {
  position: absolute;
  left: -80px;
  bottom: 50px;
  display: flex;
  flex-direction: column;
  align-items: center;
}
.insect-pixel { font-size: 40px; animation: bounce 2s infinite; }
@keyframes bounce { 0%, 100% { transform: translateY(0); } 50% { transform: translateY(-10px); } }

.dialogue-bubble {
  background: white;
  border: 2px solid #333;
  padding: 4px 8px;
  border-radius: 8px;
  font-size: 12px;
  margin-bottom: 5px;
}

.settings, .controls, .timer-display { margin: 10px 0; }
.timer-display { font-size: 40px; font-weight: bold; color: #333; }

.shop {
  margin-top: 20px;
  width: 320px;
  background: rgba(255,255,255,0.8);
  padding: 10px;
  border: 2px solid #333;
}
.shop-card {
  display: flex;
  justify-content: space-between;
  margin: 5px 0;
  align-items: center;
}

.modal-overlay {
  position: fixed;
  top:0; left:0; width:100%; height:100%;
  background: rgba(0,0,0,0.5);
  display: flex; justify-content: center; align-items: center;
  z-index: 100;
}
.modal { background: white; padding: 20px; border: 4px solid #333; text-align: center; }
.bee-pixel { font-size: 50px; }
</style>


