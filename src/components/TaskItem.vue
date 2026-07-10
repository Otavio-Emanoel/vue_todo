<script setup lang="ts">
import { ref, computed } from 'vue';

interface Tag {
  name: string;
  color: string;
}

interface SubTask {
  id: string;
  title: string;
  completed: boolean;
}

interface Task {
  id: string;
  title: string;
  description: string;
  priority: 'low' | 'medium' | 'high';
  tags: string[];
  dueDate: string;
  completed: boolean;
  subtasks: SubTask[];
  createdAt: string;
}

const props = defineProps<{
  task: Task;
  tags: Tag[];
  isFocused: boolean;
}>();

const emit = defineEmits<{
  (e: 'toggle-complete', id: string): void;
  (e: 'edit-task', task: Task): void;
  (e: 'delete-task', id: string): void;
  (e: 'focus-task', task: Task): void;
  
  // Subtask operations
  (e: 'toggle-subtask', taskId: string, subtaskId: string): void;
  (e: 'add-subtask', taskId: string, subtaskTitle: string): void;
  (e: 'delete-subtask', taskId: string, subtaskId: string): void;
}>();

const isExpanded = ref(false);
const newSubtaskTitle = ref('');

// Get tag color
const getTagColor = (tagName: string) => {
  const found = props.tags.find(t => t.name === tagName);
  return found ? found.color : 'oklch(60% 0.05 250)';
};

// Date computations
const isOverdue = computed(() => {
  if (!props.task.dueDate || props.task.completed) return false;
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  // Split due date to avoid timezone issues
  const parts = props.task.dueDate.split('-');
  const due = new Date(Number(parts[0]), Number(parts[1]) - 1, Number(parts[2]));
  return due < today;
});

const isDueToday = computed(() => {
  if (!props.task.dueDate) return false;
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  const parts = props.task.dueDate.split('-');
  const due = new Date(Number(parts[0]), Number(parts[1]) - 1, Number(parts[2]));
  return due.getTime() === today.getTime();
});

const formattedDate = computed(() => {
  if (!props.task.dueDate) return '';
  const parts = props.task.dueDate.split('-');
  const date = new Date(Number(parts[0]), Number(parts[1]) - 1, Number(parts[2]));
  return date.toLocaleDateString(undefined, { month: 'short', day: 'numeric', year: 'numeric' });
});

// Subtask counts
const completedSubtasksCount = computed(() => {
  return props.task.subtasks.filter(st => st.completed).length;
});

const totalSubtasksCount = computed(() => {
  return props.task.subtasks.length;
});

const subtaskProgressPercent = computed(() => {
  if (totalSubtasksCount.value === 0) return 0;
  return Math.round((completedSubtasksCount.value / totalSubtasksCount.value) * 100);
});

// Add subtask inline handler
const handleAddSubtask = () => {
  const trimmed = newSubtaskTitle.value.trim();
  if (!trimmed) return;
  emit('add-subtask', props.task.id, trimmed);
  newSubtaskTitle.value = '';
};
</script>

<template>
  <div 
    class="task-card glass-panel" 
    :class="{ 
      'is-completed': props.task.completed, 
      'is-focused': props.isFocused,
      'expanded': isExpanded
    }"
  >
    <!-- Task Header/Primary Row -->
    <div class="task-primary-row">
      <!-- Complete Checkbox -->
      <div class="checkbox-container">
        <label :for="'check-' + props.task.id" class="visually-hidden">Complete task</label>
        <input 
          type="checkbox" 
          :id="'check-' + props.task.id"
          :checked="props.task.completed"
          @change="emit('toggle-complete', props.task.id)"
        />
      </div>

      <!-- Task Details Content -->
      <div class="task-details-summary" @click="isExpanded = !isExpanded">
        <div class="title-row">
          <span class="task-title-text">{{ props.task.title }}</span>
          <span 
            v-if="props.task.priority !== 'medium'" 
            class="priority-dot-indicator" 
            :class="props.task.priority"
            :title="'Priority: ' + props.task.priority"
          ></span>
        </div>

        <div class="metadata-row">
          <!-- Due Date -->
          <span 
            v-if="props.task.dueDate" 
            class="due-date-badge" 
            :class="{ overdue: isOverdue, today: isDueToday }"
          >
            <svg class="meta-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <rect x="3" y="4" width="18" height="18" rx="2" ry="2"/>
              <line x1="16" y1="2" x2="16" y2="6"/>
              <line x1="8" y1="2" x2="8" y2="6"/>
              <line x1="3" y1="10" x2="21" y2="10"/>
            </svg>
            {{ isDueToday ? 'Today' : formattedDate }}
            <span v-if="isOverdue" class="overdue-text">(Overdue)</span>
          </span>

          <!-- Subtasks Progress Preview -->
          <span v-if="totalSubtasksCount > 0" class="subtask-badge">
            <svg class="meta-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M9 11l3 3L22 4"/>
              <path d="M21 12v7a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11"/>
            </svg>
            {{ completedSubtasksCount }}/{{ totalSubtasksCount }} Subtasks
          </span>

          <!-- Category Tags -->
          <div class="tag-badges-container">
            <span 
              v-for="tagName in props.task.tags" 
              :key="tagName" 
              class="tag-pill"
              :style="{ 
                backgroundColor: getTagColor(tagName) + '15',
                borderColor: getTagColor(tagName) + '30',
                color: getTagColor(tagName)
              }"
            >
              {{ tagName }}
            </span>
          </div>
        </div>
      </div>

      <!-- Action Panel -->
      <div class="task-action-buttons">
        <button 
          class="action-icon-btn focus-btn" 
          :class="{ active: props.isFocused }"
          @click.stop="emit('focus-task', props.task)" 
          title="Focus on this task"
          aria-label="Focus on this task"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="10"/>
            <circle cx="12" cy="12" r="3"/>
          </svg>
        </button>
        <button 
          class="action-icon-btn edit-btn" 
          @click.stop="emit('edit-task', props.task)" 
          title="Edit Task"
          aria-label="Edit Task"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"/>
            <path d="M18.5 2.5a2.121 2.121 0 1 1 3 3L12 15l-4 1 1-4 9.5-9.5z"/>
          </svg>
        </button>
        <button 
          class="action-icon-btn delete-btn" 
          @click.stop="emit('delete-task', props.task.id)" 
          title="Delete Task"
          aria-label="Delete Task"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <polyline points="3 6 5 6 21 6"/>
            <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/>
            <line x1="10" y1="11" x2="10" y2="17"/>
            <line x1="14" y1="11" x2="14" y2="17"/>
          </svg>
        </button>
        <button 
          class="action-icon-btn expand-toggle-btn" 
          @click.stop="isExpanded = !isExpanded"
          :title="isExpanded ? 'Collapse' : 'Expand'"
          :aria-label="isExpanded ? 'Collapse' : 'Expand'"
        >
          <svg 
            viewBox="0 0 24 24" 
            fill="none" 
            stroke="currentColor" 
            stroke-width="2"
            :class="{ 'rotate-180': isExpanded }"
          >
            <polyline points="6 9 12 15 18 9"/>
          </svg>
        </button>
      </div>
    </div>

    <!-- Collapsible Secondary Content Panel -->
    <div v-show="isExpanded" class="task-secondary-row animate-pop-in">
      <!-- Description Section -->
      <div v-if="props.task.description" class="task-description">
        <h5>Notes</h5>
        <p>{{ props.task.description }}</p>
      </div>

      <!-- Subtasks Checklist Section -->
      <div class="subtasks-section">
        <div class="subtasks-header">
          <h5>Subtasks</h5>
          <span v-if="totalSubtasksCount > 0" class="subtask-progress-lbl">
            {{ completedSubtasksCount }} of {{ totalSubtasksCount }} completed ({{ subtaskProgressPercent }}%)
          </span>
        </div>

        <!-- Subtask Progress Bar -->
        <div v-if="totalSubtasksCount > 0" class="subtask-progress-bar-container">
          <div class="subtask-progress-bar-fill" :style="{ width: subtaskProgressPercent + '%' }"></div>
        </div>

        <!-- Subtask list -->
        <div class="subtasks-list">
          <div 
            v-for="subtask in props.task.subtasks" 
            :key="subtask.id" 
            class="subtask-item"
            :class="{ 'subtask-completed': subtask.completed }"
          >
            <label class="subtask-label">
              <input 
                type="checkbox" 
                :checked="subtask.completed"
                @change="emit('toggle-subtask', props.task.id, subtask.id)"
              />
              <span class="subtask-title-text">{{ subtask.title }}</span>
            </label>
            <button 
              class="delete-subtask-btn" 
              @click="emit('delete-subtask', props.task.id, subtask.id)"
              title="Delete Subtask"
              aria-label="Delete Subtask"
            >
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M18 6L6 18M6 6l12 12"/>
              </svg>
            </button>
          </div>
        </div>

        <!-- Quick Subtask Creator -->
        <form @submit.prevent="handleAddSubtask" class="add-subtask-form">
          <input 
            type="text" 
            v-model="newSubtaskTitle"
            placeholder="Add a step to this task..."
            aria-label="Add subtask text"
          />
          <button type="submit" class="secondary subtask-add-btn">
            Add
          </button>
        </form>
      </div>
    </div>
  </div>
</template>

<style scoped>
.task-card {
  margin-bottom: 0.75rem;
  padding: 1rem 1.25rem;
  border-radius: 12px;
  transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}

.task-card:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow);
}

.task-card.is-completed {
  opacity: 0.65;
  background-color: oklch(from var(--card-bg) l c h / 0.4);
}

.task-card.is-completed .task-title-text {
  text-decoration: line-through;
  color: var(--text-muted);
}

.task-card.is-focused {
  border-color: var(--accent);
  box-shadow: 0 0 15px oklch(from var(--accent) l c h / 0.15);
}

.task-primary-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.checkbox-container {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.task-details-summary {
  flex-grow: 1;
  min-width: 0;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.title-row {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
}

.task-title-text {
  font-size: 1rem;
  font-weight: 500;
  color: var(--text-heading);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.priority-dot-indicator {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex-shrink: 0;
}

.priority-dot-indicator.high {
  background-color: var(--danger);
  box-shadow: 0 0 6px var(--danger);
}

.priority-dot-indicator.low {
  background-color: var(--success);
}

.metadata-row {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 10px;
}

.meta-icon {
  width: 12px;
  height: 12px;
  color: currentColor;
}

.due-date-badge, .subtask-badge {
  font-size: 0.75rem;
  color: var(--text-muted);
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-weight: 500;
}

.due-date-badge.today {
  color: var(--warning);
}

.due-date-badge.overdue {
  color: var(--danger);
  font-weight: 600;
}

.overdue-text {
  font-size: 0.7rem;
}

.tag-badges-container {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
}

.tag-pill {
  font-size: 0.65rem;
  font-weight: 600;
  padding: 2px 6px;
  border-radius: 4px;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  border: 1px solid transparent;
}

/* Action controls */
.task-action-buttons {
  display: flex;
  align-items: center;
  gap: 4px;
  flex-shrink: 0;
}

.action-icon-btn {
  width: 32px;
  height: 32px;
  min-height: 0;
  padding: 0;
  border-radius: 8px;
  color: var(--text-muted);
  display: flex;
  align-items: center;
  justify-content: center;
}

.action-icon-btn:hover {
  background-color: var(--border);
  color: var(--text-heading);
}

.action-icon-btn svg {
  width: 16px;
  height: 16px;
  pointer-events: none;
}

.focus-btn:hover, .focus-btn.active {
  color: var(--accent);
  background-color: oklch(from var(--accent) l c h / 0.1);
}

.delete-btn:hover {
  color: var(--danger);
  background-color: oklch(from var(--danger) l c h / 0.1);
}

.expand-toggle-btn svg {
  transition: transform 0.2s ease;
}

.expand-toggle-btn svg.rotate-180 {
  transform: rotate(180deg);
}

/* Secondary collapsible row */
.task-secondary-row {
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid var(--border);
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.task-description h5, .subtasks-section h5 {
  font-size: 0.8rem;
  font-weight: 600;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 4px;
}

.task-description p {
  font-size: 0.9rem;
  color: var(--text);
  white-space: pre-wrap;
}

.subtasks-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 4px;
}

.subtask-progress-lbl {
  font-size: 0.75rem;
  color: var(--text-muted);
  font-weight: 500;
}

/* Subtask progress bar */
.subtask-progress-bar-container {
  width: 100%;
  height: 4px;
  background-color: var(--border);
  border-radius: 99px;
  overflow: hidden;
  margin-bottom: 0.75rem;
}

.subtask-progress-bar-fill {
  height: 100%;
  background-color: var(--success);
  transition: width 0.3s ease;
}

.subtasks-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 0.75rem;
}

.subtask-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 6px 8px;
  border-radius: 6px;
  background-color: oklch(from var(--text) l c h / 0.02);
}

.subtask-item:hover {
  background-color: oklch(from var(--text) l c h / 0.04);
}

.subtask-label {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-grow: 1;
  cursor: pointer;
  font-size: 0.85rem;
}

.subtask-title-text {
  color: var(--text);
}

.subtask-completed .subtask-title-text {
  text-decoration: line-through;
  color: var(--text-muted);
}

.subtask-item input[type="checkbox"] {
  width: 14px;
  height: 14px;
}

.delete-subtask-btn {
  width: 20px;
  height: 20px;
  min-height: 0;
  padding: 0;
  border-radius: 4px;
  color: var(--text-muted);
}

.delete-subtask-btn:hover {
  background-color: oklch(from var(--danger) l c h / 0.1);
  color: var(--danger);
}

.delete-subtask-btn svg {
  width: 12px;
  height: 12px;
}

/* Inline subtask creator */
.add-subtask-form {
  display: flex;
  gap: 8px;
}

.add-subtask-form input {
  flex-grow: 1;
  height: 34px;
  padding: 4px 10px;
  font-size: 0.85rem;
}

.subtask-add-btn {
  height: 34px;
  min-height: 0;
  padding-inline: 12px;
  font-size: 0.85rem;
}

@media (max-width: 768px) {
  .task-primary-row {
    flex-wrap: wrap;
  }
  
  .task-action-buttons {
    width: 100%;
    justify-content: flex-end;
    border-top: 1px solid var(--border);
    padding-top: 8px;
    margin-top: 4px;
  }
}
</style>
