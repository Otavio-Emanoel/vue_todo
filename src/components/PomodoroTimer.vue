<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted } from 'vue';

interface Task {
  id: string;
  title: string;
  completed: boolean;
}

interface FocusSessionLog {
  id: string;
  taskTitle: string | null;
  timestamp: string;
  duration: number;
}

const props = defineProps<{
  activeTask: Task | null;
}>();

const emit = defineEmits<{
  (e: 'clear-active-task'): void;
}>();

// Timer Modes
type TimerMode = 'pomodoro' | 'shortBreak' | 'longBreak';

const MODE_TIMES: Record<TimerMode, number> = {
  pomodoro: 25 * 60,
  shortBreak: 5 * 60,
  longBreak: 15 * 60
};

const currentMode = ref<TimerMode>('pomodoro');
const timeLeft = ref(MODE_TIMES.pomodoro);
const isRunning = ref(false);
let intervalId: number | null = null;

const isMuted = ref(localStorage.getItem('pomodoro-muted') === 'true');
const focusHistory = ref<FocusSessionLog[]>([]);

onMounted(() => {
  const savedHistory = localStorage.getItem('pomodoro-history');
  if (savedHistory) {
    focusHistory.value = JSON.parse(savedHistory);
  }
});

watch(isMuted, (val) => {
  localStorage.setItem('pomodoro-muted', String(val));
});

watch(focusHistory, (newHistory) => {
  localStorage.setItem('pomodoro-history', JSON.stringify(newHistory));
}, { deep: true });

const formatHistoryDate = (isoStr: string) => {
  const d = new Date(isoStr);
  return d.toLocaleTimeString(undefined, { hour: '2-digit', minute: '2-digit' });
};

// Audio notification using Web Audio API to avoid external file dependencies
const playNotificationSound = () => {
  if (isMuted.value) return;
  try {
    const AudioContextClass = window.AudioContext || (window as any).webkitAudioContext;
    if (!AudioContextClass) return;
    const ctx = new AudioContextClass();
    
    // Play a pleasant "ding-dong" chime
    const playTone = (freq: number, startTime: number, duration: number) => {
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      osc.connect(gain);
      gain.connect(ctx.destination);
      
      osc.type = 'sine';
      osc.frequency.setValueAtTime(freq, startTime);
      
      gain.gain.setValueAtTime(0, startTime);
      gain.gain.linearRampToValueAtTime(0.4, startTime + 0.05);
      gain.gain.exponentialRampToValueAtTime(0.001, startTime + duration);
      
      osc.start(startTime);
      osc.stop(startTime + duration);
    };

    const now = ctx.currentTime;
    playTone(523.25, now, 0.4);      // C5
    playTone(659.25, now + 0.2, 0.5);  // E5
  } catch (e) {
    console.warn('AudioContext not allowed or not supported:', e);
  }
};

const setMode = (mode: TimerMode) => {
  currentMode.value = mode;
  timeLeft.value = MODE_TIMES[mode];
  pauseTimer();
};

const toggleTimer = () => {
  if (isRunning.value) {
    pauseTimer();
  } else {
    startTimer();
  }
};

const startTimer = () => {
  if (isRunning.value) return;
  isRunning.value = true;
  intervalId = window.setInterval(() => {
    if (timeLeft.value > 0) {
      timeLeft.value--;
    } else {
      timerFinished();
    }
  }, 1000);
};

const pauseTimer = () => {
  isRunning.value = false;
  if (intervalId !== null) {
    clearInterval(intervalId);
    intervalId = null;
  }
};

const resetTimer = () => {
  timeLeft.value = MODE_TIMES[currentMode.value];
  pauseTimer();
};

const timerFinished = () => {
  pauseTimer();
  playNotificationSound();

  if (currentMode.value === 'pomodoro') {
    const newLog: FocusSessionLog = {
      id: 'session-' + Date.now(),
      taskTitle: props.activeTask ? props.activeTask.title : null,
      timestamp: new Date().toISOString(),
      duration: Math.round(MODE_TIMES.pomodoro / 60)
    };
    focusHistory.value.unshift(newLog);
    if (focusHistory.value.length > 10) {
      focusHistory.value = focusHistory.value.slice(0, 10);
    }
  }

  alert(`Timer finished! Time for a ${currentMode.value === 'pomodoro' ? 'break' : 'focus session'}!`);
  // Automatically switch mode
  if (currentMode.value === 'pomodoro') {
    setMode('shortBreak');
  } else {
    setMode('pomodoro');
  }
};

const formatTime = (seconds: number) => {
  const m = Math.floor(seconds / 60).toString().padStart(2, '0');
  const s = (seconds % 60).toString().padStart(2, '0');
  return `${m}:${s}`;
};

// SVG progress calculation
const progressPercent = computed(() => {
  const total = MODE_TIMES[currentMode.value];
  return ((total - timeLeft.value) / total) * 100;
});

const strokeDashoffset = computed(() => {
  const radius = 90;
  const circumference = 2 * Math.PI * radius;
  return circumference - (progressPercent.value / 100) * circumference;
});

// Update Document Title with current time
watch([timeLeft, isRunning, currentMode], () => {
  const emoji = currentMode.value === 'pomodoro' ? '⏱️' : '☕';
  const statusStr = isRunning.value ? '' : '⏸️ ';
  document.title = `${statusStr}${formatTime(timeLeft.value)} ${emoji} Sleek Task Flow`;
});

// If the active task changes, we might want to reset the timer to help them focus
watch(() => props.activeTask, (newTask) => {
  if (newTask) {
    setMode('pomodoro');
  }
});

onUnmounted(() => {
  pauseTimer();
  document.title = 'Sleek Task Flow';
});
</script>

<template>
  <div class="pomodoro-timer glass-panel" :class="{ 'focus-active': activeTask }">
    <div class="timer-header">
      <h3>Focus Space</h3>
      <span class="mode-tag" :class="currentMode">
        {{ currentMode === 'pomodoro' ? 'Focus Session' : currentMode === 'shortBreak' ? 'Short Break' : 'Long Break' }}
      </span>
    </div>

    <!-- Active Task Focus Alert -->
    <div v-if="activeTask" class="active-focus-task animate-pop-in">
      <div class="task-info">
        <span class="focus-label">Focusing on:</span>
        <span class="task-title">{{ activeTask.title }}</span>
      </div>
      <button class="clear-focus-btn" @click="emit('clear-active-task')" title="Stop Focusing">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M18 6L6 18M6 6l12 12"/>
        </svg>
      </button>
    </div>

    <!-- Visual Timer Progress Ring -->
    <div class="timer-visual-container">
      <svg class="progress-ring" width="200" height="200">
        <circle
          class="progress-ring-bg"
          stroke="var(--border)"
          stroke-width="8"
          fill="transparent"
          r="90"
          cx="100"
          cy="100"
        />
        <circle
          class="progress-ring-circle"
          :class="[currentMode, { 'pulse': isRunning }]"
          stroke-width="8"
          stroke-linecap="round"
          fill="transparent"
          r="90"
          cx="100"
          cy="100"
          :stroke-dasharray="2 * Math.PI * 90"
          :stroke-dashoffset="strokeDashoffset"
        />
      </svg>
      <div class="timer-digits">
        <span class="time-display">{{ formatTime(timeLeft) }}</span>
      </div>
    </div>

    <!-- Mode Selector Buttons -->
    <div class="mode-selectors">
      <button 
        class="mode-btn" 
        :class="{ active: currentMode === 'pomodoro' }" 
        @click="setMode('pomodoro')"
      >
        Focus
      </button>
      <button 
        class="mode-btn" 
        :class="{ active: currentMode === 'shortBreak' }" 
        @click="setMode('shortBreak')"
      >
        Short Break
      </button>
      <button 
        class="mode-btn" 
        :class="{ active: currentMode === 'longBreak' }" 
        @click="setMode('longBreak')"
      >
        Long Break
      </button>
    </div>

    <!-- Timer Controls -->
    <div class="timer-controls">
      <button class="primary play-pause-btn" :class="{ running: isRunning }" @click="toggleTimer">
        <svg v-if="!isRunning" viewBox="0 0 24 24" fill="currentColor" class="control-icon">
          <path d="M8 5v14l11-7z"/>
        </svg>
        <svg v-else viewBox="0 0 24 24" fill="currentColor" class="control-icon">
          <path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/>
        </svg>
        {{ isRunning ? 'Pause' : 'Start' }}
      </button>
      <button class="secondary reset-btn" @click="resetTimer">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" class="control-icon-stroke">
          <path d="M21.5 2v6h-6M21.34 15.57a10 10 0 11-.57-8.38l5.67-5.67"/>
        </svg>
        Reset
      </button>
      <button 
        class="secondary mute-btn" 
        @click="isMuted = !isMuted"
        :title="isMuted ? 'Unmute alerts' : 'Mute alerts'"
        :aria-label="isMuted ? 'Unmute alerts' : 'Mute alerts'"
      >
        <svg v-if="!isMuted" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" class="control-icon-stroke">
          <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"/>
          <path d="M19.07 4.93a10 10 0 0 1 0 14.14M15.54 8.46a5 5 0 0 1 0 7.07"/>
        </svg>
        <svg v-else viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" class="control-icon-stroke">
          <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"/>
          <line x1="23" y1="9" x2="17" y2="15"/>
          <line x1="17" y1="9" x2="23" y2="15"/>
        </svg>
      </button>
    </div>

    <!-- Focus History Log -->
    <div class="focus-history-container">
      <h4 class="history-title">Session History</h4>
      <div v-if="focusHistory.length === 0" class="empty-history">
        <p>No focus sessions completed today. Start working!</p>
      </div>
      <div v-else class="history-list">
        <div v-for="session in focusHistory" :key="session.id" class="history-item">
          <div class="history-info">
            <span class="history-task">{{ session.taskTitle || 'General Focus Session' }}</span>
            <span class="history-time">{{ formatHistoryDate(session.timestamp) }}</span>
          </div>
          <span class="history-duration">+{{ session.duration }}m</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.pomodoro-timer {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.5rem;
  position: relative;
  overflow: hidden;
}

.timer-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.timer-header h3 {
  font-size: 1.1rem;
  font-weight: 600;
}

.mode-tag {
  font-size: 0.7rem;
  padding: 4px 8px;
  border-radius: 999px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.mode-tag.pomodoro {
  background-color: oklch(from var(--accent) l c h / 0.15);
  color: var(--accent);
}

.mode-tag.shortBreak, .mode-tag.longBreak {
  background-color: oklch(from var(--success) l c h / 0.15);
  color: var(--success);
}

.active-focus-task {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  background-color: oklch(from var(--accent) l c h / 0.08);
  border: 1px solid oklch(from var(--accent) l c h / 0.25);
  border-radius: 10px;
  padding: 8px 12px;
}

.task-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.focus-label {
  font-size: 0.75rem;
  color: var(--text-muted);
  font-weight: 500;
}

.task-title {
  font-size: 0.9rem;
  font-weight: 600;
  color: var(--text-heading);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.clear-focus-btn {
  width: 24px;
  height: 24px;
  min-height: 0;
  padding: 0;
  border-radius: 50%;
  color: var(--text-muted);
}

.clear-focus-btn:hover {
  background-color: rgba(0, 0, 0, 0.08);
  color: var(--danger);
}

.clear-focus-btn svg {
  width: 14px;
  height: 14px;
}

.timer-visual-container {
  position: relative;
  width: 200px;
  height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.progress-ring {
  position: absolute;
  top: 0;
  left: 0;
  transform: rotate(-90deg);
}

.progress-ring-circle.pomodoro {
  stroke: var(--accent);
}

.progress-ring-circle.shortBreak, .progress-ring-circle.longBreak {
  stroke: var(--success);
}

.progress-ring-circle.pulse {
  animation: pulseOpacity 2s infinite;
}

@keyframes pulseOpacity {
  0% { opacity: 1; }
  50% { opacity: 0.6; }
  100% { opacity: 1; }
}

.timer-digits {
  z-index: 2;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.time-display {
  font-family: var(--font-mono);
  font-size: 2.25rem;
  font-weight: 600;
  color: var(--text-heading);
  letter-spacing: -0.02em;
}

.mode-selectors {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 6px;
  width: 100%;
}

.mode-btn {
  font-size: 0.75rem;
  padding: 6px 4px;
  min-height: 0;
  border-radius: 6px;
  background-color: transparent;
  color: var(--text-muted);
  border: 1px solid transparent;
}

.mode-btn.active {
  background-color: var(--card-bg);
  border-color: var(--border);
  color: var(--text-heading);
  font-weight: 600;
}

.timer-controls {
  display: flex;
  gap: 12px;
  width: 100%;
}

.play-pause-btn {
  flex-grow: 2;
  min-height: 42px;
}

.play-pause-btn.running {
  background-color: var(--danger);
  box-shadow: 0 4px 14px oklch(from var(--danger) l c h / 0.35);
}

.reset-btn {
  flex-grow: 1;
  min-height: 42px;
}

.control-icon {
  width: 16px;
  height: 16px;
}

.control-icon-stroke {
  width: 16px;
  height: 16px;
}

.mute-btn {
  flex-shrink: 0;
  width: 42px;
  height: 42px;
  min-height: 0;
  padding: 0;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* History Log Styles */
.focus-history-container {
  width: 100%;
  border-top: 1px solid var(--border);
  padding-top: 1.25rem;
  margin-top: 0.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.history-title {
  font-size: 0.9rem;
  font-weight: 700;
  color: var(--text-heading);
}

.empty-history {
  font-size: 0.8rem;
  color: var(--text-muted);
  text-align: center;
  padding: 1rem 0;
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  max-height: 180px;
  overflow-y: auto;
  padding-right: 4px;
}

.history-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 12px;
  border-radius: 8px;
  background-color: oklch(from var(--text) l c h / 0.02);
  border: 1px solid var(--border);
}

.history-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.history-task {
  font-size: 0.825rem;
  font-weight: 600;
  color: var(--text-heading);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.history-time {
  font-size: 0.7rem;
  color: var(--text-muted);
}

.history-duration {
  font-size: 0.75rem;
  font-weight: 700;
  color: var(--success);
  flex-shrink: 0;
}

.pomodoro-timer.focus-active {
  animation: timerPulseBorder 4s infinite;
}

@keyframes timerPulseBorder {
  0% { border-color: var(--border); }
  50% { border-color: oklch(from var(--accent) l c h / 0.5); }
  100% { border-color: var(--border); }
}
</style>
