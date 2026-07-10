<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue';
import Dashboard from './components/Dashboard.vue';
import TaskItem from './components/TaskItem.vue';
import TaskForm from './components/TaskForm.vue';
import PomodoroTimer from './components/PomodoroTimer.vue';
import TagsManager from './components/TagsManager.vue';

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

// ----------------------------------------------------
// Theme Management (2-State Control: System vs. Pinned)
// ----------------------------------------------------
const systemDarkMedia = window.matchMedia('(prefers-color-scheme: dark)');
const isPinnedTheme = ref(localStorage.getItem('todo-theme-pinned') === 'true');
const savedTheme = localStorage.getItem('todo-theme') || 'system';
const theme = ref<'system' | 'light' | 'dark'>(savedTheme as any);

const updateHtmlTheme = (currentTheme: 'system' | 'light' | 'dark') => {
  let activeTheme: 'light' | 'dark';
  if (currentTheme === 'system') {
    activeTheme = systemDarkMedia.matches ? 'dark' : 'light';
  } else {
    activeTheme = currentTheme;
  }

  if (activeTheme === 'dark') {
    document.documentElement.style.colorScheme = 'dark';
    document.documentElement.classList.add('dark-theme');
    document.documentElement.classList.remove('light-theme');
  } else {
    document.documentElement.style.colorScheme = 'light';
    document.documentElement.classList.add('light-theme');
    document.documentElement.classList.remove('dark-theme');
  }
};

const toggleTheme = () => {
  const systemTheme = systemDarkMedia.matches ? 'dark' : 'light';
  if (theme.value === 'system') {
    // Switch to opposite of system and pin it
    theme.value = systemTheme === 'dark' ? 'light' : 'dark';
    localStorage.setItem('todo-theme', theme.value);
    localStorage.setItem('todo-theme-pinned', 'true');
    isPinnedTheme.value = true;
  } else {
    // Reset to system and unpin
    theme.value = 'system';
    localStorage.setItem('todo-theme', 'system');
    localStorage.setItem('todo-theme-pinned', 'false');
    isPinnedTheme.value = false;
  }
  updateHtmlTheme(theme.value);
};

// Listen to OS theme changes
systemDarkMedia.addEventListener('change', () => {
  if (theme.value === 'system') {
    updateHtmlTheme('system');
  }
});

// ----------------------------------------------------
// State Definition
// ----------------------------------------------------
const defaultTags: Tag[] = [
  { name: 'Work', color: 'oklch(74% 0.16 284)' },       // Lavender
  { name: 'Personal', color: 'oklch(68% 0.17 145)' },   // Emerald
  { name: 'Shopping', color: 'oklch(78% 0.16 75)' },    // Amber
  { name: 'Health', color: 'oklch(62% 0.21 24)' }       // Crimson
];

// Helper to get formatted dates
const getRelativeDate = (offsetDays: number): string => {
  const d = new Date();
  d.setDate(d.getDate() + offsetDays);
  const year = d.getFullYear();
  const month = String(d.getMonth() + 1).padStart(2, '0');
  const day = String(d.getDate()).padStart(2, '0');
  return `${year}-${month}-${day}`;
};

const defaultTasks: Task[] = [
  {
    id: 'task-1',
    title: 'Design premium dashboard UI layout',
    description: 'Sketch layout concepts, configure OKLCH design variables, and implement glassmorphic overlay sheets.',
    priority: 'high',
    tags: ['Work'],
    dueDate: getRelativeDate(1), // Tomorrow
    completed: false,
    subtasks: [
      { id: 'sub-1-1', title: 'Select custom Outfit Google Font', completed: true },
      { id: 'sub-1-2', title: 'Code custom checkbox styles', completed: false },
      { id: 'sub-1-3', title: 'Draw custom circular SVG progress widget', completed: false }
    ],
    createdAt: new Date(Date.now() - 4 * 3600000).toISOString()
  },
  {
    id: 'task-2',
    title: 'Review modern CSS best practices guide',
    description: 'Familiarize with the new light-dark() CSS color utility, container queries, and accessibility targets.',
    priority: 'medium',
    tags: ['Work', 'Personal'],
    dueDate: getRelativeDate(0), // Today
    completed: true,
    subtasks: [],
    createdAt: new Date(Date.now() - 24 * 3600000).toISOString()
  },
  {
    id: 'task-3',
    title: 'Restock organic greens and healthy snacks',
    description: 'Purchase avocados, baby spinach, strawberries, almonds, oats, and 85% dark cacao bar.',
    priority: 'low',
    tags: ['Shopping', 'Health'],
    dueDate: getRelativeDate(-2), // Overdue (2 days ago)
    completed: false,
    subtasks: [
      { id: 'sub-3-1', title: 'Check local farmer market stock', completed: false }
    ],
    createdAt: new Date(Date.now() - 48 * 3600000).toISOString()
  }
];

const tasks = ref<Task[]>([]);
const tags = ref<Tag[]>([]);
const activeFocusTaskId = ref<string | null>(null);
const activePage = ref<'tasks' | 'dashboard' | 'pomodoro' | 'tags'>('tasks');
const pendingTasksList = computed(() => tasks.value.filter(t => !t.completed));

// Form Control States
const showForm = ref(false);
const taskToEdit = ref<Task | null>(null);

// Filter States
const statusFilter = ref<'all' | 'active' | 'completed'>('all');
const priorityFilter = ref<'all' | 'high' | 'medium' | 'low'>('all');
const tagFilter = ref<string>('all');
const searchQuery = ref('');
const sortBy = ref('createdAt-desc');

// ----------------------------------------------------
// Lifecycles and Local Storage synchronization
// ----------------------------------------------------
onMounted(() => {
  // Theme load
  updateHtmlTheme(theme.value);

  // Data load
  const savedTasks = localStorage.getItem('todo-tasks');
  const savedTags = localStorage.getItem('todo-tags');
  const savedActiveFocus = localStorage.getItem('todo-active-focus-task-id');

  if (savedTasks) {
    tasks.value = JSON.parse(savedTasks);
  } else {
    tasks.value = defaultTasks;
  }

  if (savedTags) {
    tags.value = JSON.parse(savedTags);
  } else {
    tags.value = defaultTags;
  }

  if (savedActiveFocus) {
    activeFocusTaskId.value = savedActiveFocus;
  }
});

watch(tasks, (newTasks) => {
  localStorage.setItem('todo-tasks', JSON.stringify(newTasks));
}, { deep: true });

watch(tags, (newTags) => {
  localStorage.setItem('todo-tags', JSON.stringify(newTags));
}, { deep: true });

watch(activeFocusTaskId, (newVal) => {
  if (newVal) {
    localStorage.setItem('todo-active-focus-task-id', newVal);
  } else {
    localStorage.removeItem('todo-active-focus-task-id');
  }
});

// Computed Active Task
const activeFocusTask = computed(() => {
  if (!activeFocusTaskId.value) return null;
  return tasks.value.find(t => t.id === activeFocusTaskId.value) || null;
});

// ----------------------------------------------------
// Task Methods
// ----------------------------------------------------
const handleAddTaskClick = () => {
  taskToEdit.value = null;
  showForm.value = !showForm.value;
};

const handleEditTaskClick = (task: Task) => {
  taskToEdit.value = task;
  showForm.value = true;
};

const cancelForm = () => {
  showForm.value = false;
  taskToEdit.value = null;
};

const handleTaskSubmit = (taskData: Omit<Task, 'id' | 'completed' | 'subtasks' | 'createdAt'> & { id?: string }) => {
  if (taskData.id) {
    // Edit existing task
    const taskIdx = tasks.value.findIndex(t => t.id === taskData.id);
    if (taskIdx !== -1) {
      tasks.value[taskIdx] = {
        ...tasks.value[taskIdx],
        title: taskData.title,
        description: taskData.description,
        priority: taskData.priority,
        tags: taskData.tags,
        dueDate: taskData.dueDate
      };
    }
  } else {
    // Create new task
    const newTask: Task = {
      id: 'task-' + Date.now(),
      title: taskData.title,
      description: taskData.description,
      priority: taskData.priority,
      tags: taskData.tags,
      dueDate: taskData.dueDate,
      completed: false,
      subtasks: [],
      createdAt: new Date().toISOString()
    };
    tasks.value.unshift(newTask);
  }

  showForm.value = false;
  taskToEdit.value = null;
};

const toggleComplete = (id: string) => {
  const task = tasks.value.find(t => t.id === id);
  if (task) {
    task.completed = !task.completed;
  }
};

const deleteTask = (id: string) => {
  if (confirm('Are you sure you want to delete this task?')) {
    tasks.value = tasks.value.filter(t => t.id !== id);
    if (activeFocusTaskId.value === id) {
      activeFocusTaskId.value = null;
    }
  }
};

const handleFocusTask = (task: Task) => {
  activeFocusTaskId.value = task.id;
};

const handleClearFocus = () => {
  activeFocusTaskId.value = null;
};

const updateTags = (newTags: Tag[]) => {
  tags.value = newTags;
  // Prune deleted tags from tasks
  const tagNames = new Set(newTags.map(t => t.name));
  tasks.value.forEach(task => {
    task.tags = task.tags.filter(t => tagNames.has(t));
  });
};

// ----------------------------------------------------
// Subtasks Methods
// ----------------------------------------------------
const toggleSubtask = (taskId: string, subtaskId: string) => {
  const task = tasks.value.find(t => t.id === taskId);
  if (task) {
    const sub = task.subtasks.find(s => s.id === subtaskId);
    if (sub) {
      sub.completed = !sub.completed;
    }
  }
};

const addSubtask = (taskId: string, title: string) => {
  const task = tasks.value.find(t => t.id === taskId);
  if (task) {
    task.subtasks.push({
      id: 'sub-' + Date.now(),
      title,
      completed: false
    });
  }
};

const deleteSubtask = (taskId: string, subtaskId: string) => {
  const task = tasks.value.find(t => t.id === taskId);
  if (task) {
    task.subtasks = task.subtasks.filter(s => s.id !== subtaskId);
  }
};

// ----------------------------------------------------
// Dashboard & Stats Computations
// ----------------------------------------------------
const totalTasksCount = computed(() => tasks.value.length);
const completedTasksCount = computed(() => tasks.value.filter(t => t.completed).length);
const pendingTasksCount = computed(() => tasks.value.filter(t => !t.completed).length);
const highPriorityTasksCount = computed(() => tasks.value.filter(t => t.priority === 'high' && !t.completed).length);

// ----------------------------------------------------
// Filter & Sort Logic
// ----------------------------------------------------
const filteredTasks = computed(() => {
  let list = [...tasks.value];

  // 1. Completion Filter
  if (statusFilter.value === 'active') {
    list = list.filter(t => !t.completed);
  } else if (statusFilter.value === 'completed') {
    list = list.filter(t => t.completed);
  }

  // 2. Priority Filter
  if (priorityFilter.value !== 'all') {
    list = list.filter(t => t.priority === priorityFilter.value);
  }

  // 3. Category Tag Filter
  if (tagFilter.value !== 'all') {
    list = list.filter(t => t.tags.includes(tagFilter.value));
  }

  // 4. Search Filter
  const q = searchQuery.value.trim().toLowerCase();
  if (q) {
    list = list.filter(t => 
      t.title.toLowerCase().includes(q) || 
      t.description.toLowerCase().includes(q)
    );
  }

  // 5. Sorting
  list.sort((a, b) => {
    switch (sortBy.value) {
      case 'title-asc':
        return a.title.localeCompare(b.title);
      case 'createdAt-desc':
        return new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime();
      case 'createdAt-asc':
        return new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime();
      case 'dueDate-asc':
        if (!a.dueDate) return 1;
        if (!b.dueDate) return -1;
        return a.dueDate.localeCompare(b.dueDate);
      case 'dueDate-desc':
        if (!a.dueDate) return 1;
        if (!b.dueDate) return -1;
        return b.dueDate.localeCompare(a.dueDate);
      case 'priority-desc': {
        const priorityWeight = { high: 3, medium: 2, low: 1 };
        return priorityWeight[b.priority] - priorityWeight[a.priority];
      }
      case 'priority-asc': {
        const priorityWeight = { high: 3, medium: 2, low: 1 };
        return priorityWeight[a.priority] - priorityWeight[b.priority];
      }
      default:
        return 0;
    }
  });

  return list;
});
</script>

<template>
  <div class="glass-container main-wrapper">
    <!-- Header Area -->
    <header class="app-header glass-panel animate-pop-in">
      <div class="logo-wrapper">
        <svg class="app-logo" viewBox="0 0 24 24" fill="none" stroke="var(--accent)" stroke-width="2.5">
          <path d="M12 2L2 7l10 5 10-5-10-5zM2 17l10 5 10-5M2 12l10 5 10-5"/>
        </svg>
        <div class="title-meta">
          <h1>Task Flow</h1>
          <p class="subtitle">Streamline your thoughts. Elevate your focus.</p>
        </div>
      </div>

      <div class="header-controls">
        <!-- Pinned Theme Toggle Button -->
        <button 
          class="theme-toggle-btn secondary" 
          @click="toggleTheme"
          :title="isPinnedTheme ? 'Pinned Theme: Reset to System' : 'System Theme: Pin Choice'"
        >
          <svg v-if="theme === 'dark' || (theme === 'system' && systemDarkMedia.matches)" class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M21 12.79A9 9 0 1111.21 3 7 7 0 0021 12.79z"/>
          </svg>
          <svg v-else class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="5"/>
            <line x1="12" y1="1" x2="12" y2="3"/>
            <line x1="12" y1="21" x2="12" y2="23"/>
            <line x1="4.22" y1="4.22" x2="5.64" y2="5.64"/>
            <line x1="18.36" y1="18.36" x2="19.78" y2="19.78"/>
            <line x1="1" y1="12" x2="3" y2="12"/>
            <line x1="21" y1="12" x2="23" y2="12"/>
            <line x1="4.22" y1="19.78" x2="5.64" y2="18.36"/>
            <line x1="18.36" y1="5.64" x2="19.78" y2="4.22"/>
          </svg>
          <span class="theme-mode-text">{{ isPinnedTheme ? 'Pinned' : 'Auto' }}</span>
        </button>

        <button 
          v-if="activePage === 'tasks'"
          class="primary add-task-trigger" 
          @click="handleAddTaskClick"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" class="icon">
            <line x1="12" y1="5" x2="12" y2="19"/>
            <line x1="5" y1="12" x2="14" y2="12"/>
          </svg>
          New Task
        </button>
      </div>
    </header>

    <!-- Navigation Tabs Bar -->
    <nav class="app-nav glass-panel animate-pop-in">
      <button 
        class="nav-tab-btn" 
        :class="{ active: activePage === 'tasks' }" 
        @click="activePage = 'tasks'"
      >
        <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M9 11l3 3L22 4"/>
          <path d="M21 12v7a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11"/>
        </svg>
        Tasks
      </button>
      <button 
        class="nav-tab-btn" 
        :class="{ active: activePage === 'dashboard' }" 
        @click="activePage = 'dashboard'"
      >
        <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M18 20V10M12 20V4M6 20v-6"/>
        </svg>
        Dashboard
      </button>
      <button 
        class="nav-tab-btn" 
        :class="{ active: activePage === 'pomodoro' }" 
        @click="activePage = 'pomodoro'"
      >
        <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <circle cx="12" cy="12" r="10"/>
          <polyline points="12 6 12 12 16 14"/>
        </svg>
        Focus Timer
      </button>
      <button 
        class="nav-tab-btn" 
        :class="{ active: activePage === 'tags' }" 
        @click="activePage = 'tags'"
      >
        <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M20.59 13.41l-7.17 7.17a2 2 0 0 1-2.83 0L2 12V2h10l8.59 8.59a2 2 0 0 1 0 2.82z"/>
          <line x1="7" y1="7" x2="7.01" y2="7"/>
        </svg>
        Categories
      </button>
    </nav>

    <!-- Content Workspace Layout -->
    <div class="glass-layout-tabs">
      <div class="page-content-wrapper">
        <!-- 1. Tasks Page -->
        <main v-if="activePage === 'tasks'" class="main-col flex-col animate-pop-in">
          <!-- Task Form (Collapsible Drawer) -->
          <Transition name="form-slide">
            <TaskForm 
              v-if="showForm"
              :tags="tags"
              :task-to-edit="taskToEdit"
              @submit-task="handleTaskSubmit"
              @cancel="cancelForm"
            />
          </Transition>

          <!-- Search & Filters -->
          <Dashboard 
            :total-tasks="totalTasksCount"
            :completed-tasks="completedTasksCount"
            :pending-tasks="pendingTasksCount"
            :high-priority-tasks="highPriorityTasksCount"
            
            v-model:statusFilter="statusFilter"
            v-model:priorityFilter="priorityFilter"
            v-model:tagFilter="tagFilter"
            v-model:searchQuery="searchQuery"
            v-model:sortBy="sortBy"
            
            :tags="tags"
            :show-filters-only="true"
          />

          <!-- Tasks Feed list -->
          <div class="tasks-feed-container">
            <TransitionGroup name="task-list" tag="div" class="tasks-transition-group">
              <TaskItem 
                v-for="task in filteredTasks"
                :key="task.id"
                :task="task"
                :tags="tags"
                :is-focused="activeFocusTaskId === task.id"
                @toggle-complete="toggleComplete"
                @edit-task="handleEditTaskClick"
                @delete-task="deleteTask"
                @focus-task="handleFocusTask"
                @toggle-subtask="toggleSubtask"
                @add-subtask="addSubtask"
                @delete-subtask="deleteSubtask"
              />
            </TransitionGroup>

            <!-- Empty list state -->
            <div v-if="filteredTasks.length === 0" class="empty-state glass-panel animate-pop-in">
              <svg class="empty-icon" viewBox="0 0 24 24" fill="none" stroke="var(--text-muted)" stroke-width="1.5">
                <circle cx="12" cy="12" r="10"/>
                <path d="M8 12h8"/>
              </svg>
              <h4>No tasks found</h4>
              <p>Try clearing filters or add a brand-new task to get started.</p>
              <button 
                v-if="statusFilter !== 'all' || priorityFilter !== 'all' || tagFilter !== 'all' || searchQuery"
                class="secondary" 
                @click="statusFilter = 'all'; priorityFilter = 'all'; tagFilter = 'all'; searchQuery = ''"
              >
                Reset Filters
              </button>
            </div>
          </div>
        </main>

        <!-- 2. Dashboard Page -->
        <div v-else-if="activePage === 'dashboard'" class="main-col flex-col animate-pop-in">
          <Dashboard 
            :total-tasks="totalTasksCount"
            :completed-tasks="completedTasksCount"
            :pending-tasks="pendingTasksCount"
            :high-priority-tasks="highPriorityTasksCount"
            
            v-model:statusFilter="statusFilter"
            v-model:priorityFilter="priorityFilter"
            v-model:tagFilter="tagFilter"
            v-model:searchQuery="searchQuery"
            v-model:sortBy="sortBy"
            
            :tags="tags"
            :show-stats-only="true"
          />
        </div>

        <!-- 3. Pomodoro Focus Timer Page -->
        <div v-else-if="activePage === 'pomodoro'" class="pomodoro-page-layout animate-pop-in">
          <div class="pomodoro-grid">
            <PomodoroTimer 
              :active-task="activeFocusTask" 
              @clear-active-task="handleClearFocus"
            />
            
            <!-- Quick-select Focus List -->
            <div class="pending-focus-panel glass-panel">
              <h3 class="panel-title">Focus on a Pending Task</h3>
              <p class="panel-subtitle">Select a task from your list to activate focused tracking.</p>
              <div v-if="pendingTasksList.length === 0" class="empty-pending-focus">
                <svg class="empty-focus-icon" viewBox="0 0 24 24" fill="none" stroke="var(--text-muted)" stroke-width="1.5">
                  <path d="M12 2v20M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"/>
                </svg>
                <p>All tasks are completed! Excellent work.</p>
              </div>
              <div v-else class="focus-tasks-list">
                <div 
                  v-for="task in pendingTasksList" 
                  :key="task.id" 
                  class="focus-task-item glass-panel"
                  :class="{ active: activeFocusTaskId === task.id }"
                >
                  <div class="focus-task-info">
                    <span class="focus-task-title-text">{{ task.title }}</span>
                    <span class="focus-task-priority" :class="task.priority">{{ task.priority }} priority</span>
                  </div>
                  <button 
                    class="focus-btn-action"
                    :class="activeFocusTaskId === task.id ? 'danger' : 'primary'"
                    @click="activeFocusTaskId === task.id ? handleClearFocus() : handleFocusTask(task)"
                  >
                    {{ activeFocusTaskId === task.id ? 'Unfocus' : 'Focus' }}
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 4. Categories Management Page -->
        <div v-else-if="activePage === 'tags'" class="tags-page-layout animate-pop-in">
          <div class="glass-panel">
            <TagsManager 
              :tags="tags" 
              :inline="true"
              @update-tags="updateTags"
            />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
/* Global App Override overrides for clean layout */
.app-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.25rem 2rem;
  flex-wrap: wrap;
  gap: 1.5rem;
}

.logo-wrapper {
  display: flex;
  align-items: center;
  gap: 12px;
}

.app-logo {
  width: 32px;
  height: 32px;
  flex-shrink: 0;
}

.title-meta h1 {
  font-size: 1.5rem;
  font-weight: 700;
  letter-spacing: -0.02em;
}

.subtitle {
  font-size: 0.8rem;
  color: var(--text-muted);
}

.header-controls {
  display: flex;
  align-items: center;
  gap: 10px;
}

.theme-toggle-btn {
  min-height: 40px;
}

.theme-mode-text {
  font-size: 0.75rem;
  text-transform: uppercase;
  font-weight: 600;
  letter-spacing: 0.05em;
  color: var(--text-muted);
}

.sidebar-actions-panel {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.sidebar-title {
  font-size: 0.95rem;
  font-weight: 600;
  color: var(--text-heading);
}

.sidebar-desc {
  font-size: 0.8rem;
  color: var(--text-muted);
  margin-bottom: 0.5rem;
}

.tasks-feed-container {
  position: relative;
  min-height: 200px;
}

/* Empty State */
.empty-state {
  text-align: center;
  padding: 3rem 2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
}

.empty-icon {
  width: 48px;
  height: 48px;
}

.empty-state h4 {
  font-size: 1.1rem;
  font-weight: 600;
}

.empty-state p {
  font-size: 0.875rem;
  color: var(--text-muted);
}

/* ----------------------------------------------------
   Transitions Animations Classes
   ---------------------------------------------------- */

/* Task Form slide transition */
.form-slide-enter-active,
.form-slide-leave-active {
  transition: all 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.form-slide-enter-from,
.form-slide-leave-to {
  opacity: 0;
  transform: translateY(-20px);
}

/* List Transitions */
.task-list-enter-active,
.task-list-leave-active {
  transition: all 0.4s cubic-bezier(0.3, 0.85, 0.4, 1.1);
}

.task-list-enter-from {
  opacity: 0;
  transform: translateY(20px);
}

.task-list-leave-to {
  opacity: 0;
  transform: translateX(40px);
}

.task-list-leave-active {
  position: absolute;
  width: 100%;
}

.task-list-move {
  transition: transform 0.4s cubic-bezier(0.3, 0.85, 0.4, 1.1);
}

/* Navigation Tabs styling */
.app-nav {
  display: flex;
  gap: 8px;
  padding: 0.5rem;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}

.nav-tab-btn {
  flex: 1;
  min-width: 120px;
  min-height: 44px;
  background-color: transparent;
  color: var(--text-muted);
  border: 1px solid transparent;
  border-radius: 12px;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  font-weight: 600;
  font-size: 0.9rem;
}

.nav-tab-btn:hover {
  background-color: oklch(from var(--text) l c h / 0.05);
  color: var(--text-heading);
}

.nav-tab-btn.active {
  background-color: var(--card-bg);
  border-color: var(--border);
  color: var(--accent);
  box-shadow: var(--shadow);
}

.nav-tab-btn .icon {
  width: 18px;
  height: 18px;
}

/* Tabs layout wrapper */
.glass-layout-tabs {
  width: 100%;
}

.page-content-wrapper {
  width: 100%;
}

/* Pomodoro page styles */
.pomodoro-page-layout {
  width: 100%;
}

.pomodoro-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 2rem;
}

@media (min-width: 800px) {
  .pomodoro-grid {
    grid-template-columns: 380px 1fr;
  }
}

.pending-focus-panel {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.panel-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: var(--text-heading);
}

.panel-subtitle {
  font-size: 0.85rem;
  color: var(--text-muted);
  margin-bottom: 0.5rem;
}

.focus-tasks-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  max-height: 450px;
  overflow-y: auto;
  padding-right: 4px;
}

.focus-task-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  gap: 1rem;
  border-radius: 12px;
  transition: all 0.25s ease;
}

.focus-task-item.active {
  border-color: var(--accent);
  box-shadow: 0 0 10px oklch(from var(--accent) l c h / 0.1);
}

.focus-task-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-width: 0;
}

.focus-task-title-text {
  font-weight: 600;
  font-size: 0.95rem;
  color: var(--text-heading);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.focus-task-priority {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.focus-task-priority.high {
  color: var(--danger);
}

.focus-task-priority.medium {
  color: var(--warning);
}

.focus-task-priority.low {
  color: var(--success);
}

.focus-btn-action {
  flex-shrink: 0;
  min-height: 36px;
  padding-inline: 12px;
  font-size: 0.85rem;
  font-weight: 600;
}

.empty-pending-focus {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 3rem 1rem;
  text-align: center;
  color: var(--text-muted);
  gap: 0.75rem;
}

.empty-focus-icon {
  width: 48px;
  height: 48px;
}

/* Category Tags Manager inline styles */
.tags-manager-inline {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.flex-col-inline {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

@media (max-width: 768px) {
  .app-header {
    flex-direction: column;
    align-items: stretch;
    padding: 1rem;
  }
  
  .logo-wrapper {
    justify-content: center;
  }
  
  .title-meta {
    text-align: center;
  }
  
  .header-controls {
    justify-content: center;
  }
}
</style>
