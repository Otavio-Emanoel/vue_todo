<script setup lang="ts">
import { ref, watch } from 'vue';

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
  tags: Tag[];
  taskToEdit: Task | null;
}>();

const emit = defineEmits<{
  (e: 'submit-task', taskData: Omit<Task, 'id' | 'completed' | 'subtasks' | 'createdAt'> & { id?: string }): void;
  (e: 'cancel'): void;
}>();

const title = ref('');
const description = ref('');
const priority = ref<'low' | 'medium' | 'high'>('medium');
const selectedTags = ref<string[]>([]);
const dueDate = ref('');

// Validation States
const titleTouched = ref(false);
const titleError = ref(false);

const handleTitleBlur = () => {
  titleTouched.value = true;
  titleError.value = !title.value.trim();
};

const handleTitleInput = () => {
  if (titleTouched.value) {
    titleError.value = !title.value.trim();
  }
};

// Sync fields if editing a task
watch(() => props.taskToEdit, (task) => {
  if (task) {
    title.value = task.title;
    description.value = task.description;
    priority.value = task.priority;
    selectedTags.value = [...task.tags];
    dueDate.value = task.dueDate;
  } else {
    resetForm();
  }
}, { immediate: true });

function resetForm() {
  title.value = '';
  description.value = '';
  priority.value = 'medium';
  selectedTags.value = [];
  dueDate.value = '';
  titleTouched.value = false;
  titleError.value = false;
}

const toggleTag = (tagName: string) => {
  const index = selectedTags.value.indexOf(tagName);
  if (index === -1) {
    selectedTags.value.push(tagName);
  } else {
    selectedTags.value.splice(index, 1);
  }
};

const submitForm = () => {
  titleTouched.value = true;
  if (!title.value.trim()) {
    titleError.value = true;
    return;
  }

  emit('submit-task', {
    ...(props.taskToEdit ? { id: props.taskToEdit.id } : {}),
    title: title.value.trim(),
    description: description.value.trim(),
    priority: priority.value,
    tags: [...selectedTags.value],
    dueDate: dueDate.value
  });

  resetForm();
};
</script>

<template>
  <div class="task-form glass-panel">
    <h3 class="form-title">{{ props.taskToEdit ? 'Edit Task' : 'Create New Task' }}</h3>
    
    <form @submit.prevent="submitForm" novalidate>
      <!-- Task Title -->
      <div class="form-group" :class="{ 'has-error': titleError }">
        <label for="task-title">Title <span class="required">*</span></label>
        <input 
          type="text" 
          id="task-title" 
          v-model="title" 
          placeholder="e.g. Design app UI dashboard"
          @blur="handleTitleBlur"
          @input="handleTitleInput"
          required
        />
        <span v-if="titleError" class="error-msg" role="alert">Task title is required</span>
      </div>

      <!-- Description -->
      <div class="form-group">
        <label for="task-desc">Description</label>
        <textarea 
          id="task-desc" 
          v-model="description" 
          placeholder="Add details about this task..."
          rows="3"
        ></textarea>
      </div>

      <!-- Priority -->
      <div class="form-group">
        <label>Priority</label>
        <div class="priority-selector">
          <button 
            type="button" 
            class="priority-btn low" 
            :class="{ active: priority === 'low' }"
            @click="priority = 'low'"
          >
            Low
          </button>
          <button 
            type="button" 
            class="priority-btn medium" 
            :class="{ active: priority === 'medium' }"
            @click="priority = 'medium'"
          >
            Medium
          </button>
          <button 
            type="button" 
            class="priority-btn high" 
            :class="{ active: priority === 'high' }"
            @click="priority = 'high'"
          >
            High
          </button>
        </div>
      </div>

      <!-- Tag Multi-select -->
      <div class="form-group">
        <label>Categories</label>
        <div class="tags-picker">
          <button
            v-for="tag in props.tags"
            :key="tag.name"
            type="button"
            class="tag-picker-chip"
            :class="{ active: selectedTags.includes(tag.name) }"
            :style="{ 
              '--tag-color': tag.color,
              borderColor: selectedTags.includes(tag.name) ? tag.color : 'var(--border)',
              backgroundColor: selectedTags.includes(tag.name) ? tag.color + '15' : 'transparent',
              color: selectedTags.includes(tag.name) ? tag.color : 'var(--text)'
            }"
            @click="toggleTag(tag.name)"
          >
            <span class="dot" :style="{ backgroundColor: tag.color }"></span>
            {{ tag.name }}
          </button>
        </div>
      </div>

      <!-- Due Date -->
      <div class="form-group">
        <label for="task-due">Due Date</label>
        <input 
          type="date" 
          id="task-due" 
          v-model="dueDate"
        />
      </div>

      <!-- Actions -->
      <div class="form-actions">
        <button 
          v-if="props.taskToEdit" 
          type="button" 
          class="secondary cancel-btn" 
          @click="emit('cancel')"
        >
          Cancel
        </button>
        <button type="submit" class="primary submit-btn">
          {{ props.taskToEdit ? 'Save Changes' : 'Create Task' }}
        </button>
      </div>
    </form>
  </div>
</template>

<style scoped>
.task-form {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.form-title {
  font-size: 1.2rem;
  font-weight: 600;
  border-bottom: 1px solid var(--border);
  padding-bottom: 0.75rem;
}

form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-group label {
  font-size: 0.875rem;
  font-weight: 500;
  color: var(--text-heading);
}

.required {
  color: var(--danger);
}

textarea {
  resize: vertical;
}

/* Priority buttons styling */
.priority-selector {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
}

.priority-btn {
  padding: 8px 12px;
  border: 1px solid var(--border);
  border-radius: 8px;
  font-size: 0.85rem;
  font-weight: 600;
  text-align: center;
  background-color: var(--card-bg);
  color: var(--text-muted);
}

.priority-btn.low.active {
  background-color: oklch(from var(--success) l c h / 0.1);
  border-color: var(--success);
  color: var(--success);
}

.priority-btn.medium.active {
  background-color: oklch(from var(--warning) l c h / 0.1);
  border-color: var(--warning);
  color: var(--warning);
}

.priority-btn.high.active {
  background-color: oklch(from var(--danger) l c h / 0.1);
  border-color: var(--danger);
  color: var(--danger);
}

/* Tags Picker */
.tags-picker {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.tag-picker-chip {
  padding: 6px 12px;
  min-height: 0;
  border-radius: 999px;
  font-size: 0.8rem;
  font-weight: 500;
  border: 1px solid var(--border);
  display: inline-flex;
  align-items: center;
  gap: 6px;
  transition: all 0.2s ease;
}

.tag-picker-chip.active {
  font-weight: 600;
}

.tag-picker-chip .dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
}

/* Validation styles */
.has-error input {
  border-color: var(--danger);
  box-shadow: 0 0 0 3px oklch(from var(--danger) l c h / 0.15);
}

.error-msg {
  font-size: 0.75rem;
  color: var(--danger);
  font-weight: 500;
}

.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 0.5rem;
}

.submit-btn {
  flex-grow: 1;
}

.cancel-btn {
  flex-grow: 1;
}
</style>
