<script setup lang="ts">
import { computed } from 'vue';

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

const props = withDefaults(defineProps<{
  tasks: Task[];
  totalTasks: number;
  completedTasks: number;
  pendingTasks: number;
  highPriorityTasks: number;
  
  statusFilter: 'all' | 'active' | 'completed';
  priorityFilter: 'all' | 'high' | 'medium' | 'low';
  tagFilter: string; // 'all' or tag name
  searchQuery: string;
  sortBy: string;
  
  tags: Tag[];

  showStatsOnly?: boolean;
  showFiltersOnly?: boolean;
}>(), {
  showStatsOnly: false,
  showFiltersOnly: false
});

const emit = defineEmits<{
  (e: 'update:statusFilter', val: 'all' | 'active' | 'completed'): void;
  (e: 'update:priorityFilter', val: 'all' | 'high' | 'medium' | 'low'): void;
  (e: 'update:tagFilter', val: string): void;
  (e: 'update:searchQuery', val: string): void;
  (e: 'update:sortBy', val: string): void;
}>();

// Completion Percentage
const completionRate = computed(() => {
  if (props.totalTasks === 0) return 0;
  return Math.round((props.completedTasks / props.totalTasks) * 100);
});

// SVG Circular Progress Ring calculations
const strokeDasharray = 2 * Math.PI * 34; // Radius is 34, circumference ~ 213.6
const strokeDashoffset = computed(() => {
  return strokeDasharray - (completionRate.value / 100) * strokeDasharray;
});

// Detailed Task Analytics
const highPriorityTasksList = computed(() => props.tasks.filter(t => t.priority === 'high'));
const mediumPriorityTasksList = computed(() => props.tasks.filter(t => t.priority === 'medium'));
const lowPriorityTasksList = computed(() => props.tasks.filter(t => t.priority === 'low'));

const highPriorityCompleted = computed(() => highPriorityTasksList.value.filter(t => t.completed).length);
const mediumPriorityCompleted = computed(() => mediumPriorityTasksList.value.filter(t => t.completed).length);
const lowPriorityCompleted = computed(() => lowPriorityTasksList.value.filter(t => t.completed).length);

const highPriorityPercent = computed(() => {
  const total = highPriorityTasksList.value.length;
  return total ? Math.round((highPriorityCompleted.value / total) * 100) : 0;
});
const mediumPriorityPercent = computed(() => {
  const total = mediumPriorityTasksList.value.length;
  return total ? Math.round((mediumPriorityCompleted.value / total) * 100) : 0;
});
const lowPriorityPercent = computed(() => {
  const total = lowPriorityTasksList.value.length;
  return total ? Math.round((lowPriorityCompleted.value / total) * 100) : 0;
});

// Category/tag progress stats
const categoryStats = computed(() => {
  return props.tags.map(tag => {
    const tagTasks = props.tasks.filter(t => t.tags.includes(tag.name));
    const completed = tagTasks.filter(t => t.completed).length;
    const total = tagTasks.length;
    const percent = total ? Math.round((completed / total) * 100) : 0;
    return {
      name: tag.name,
      color: tag.color,
      completed,
      total,
      percent
    };
  });
});
</script>

<template>
  <div class="dashboard flex-col">
    <!-- Stats Row -->
    <div v-if="!props.showFiltersOnly" class="stats-panel glass-panel">
      <!-- Circular Progress Ring Widget -->
      <div class="progress-widget">
        <svg class="progress-ring" viewBox="0 0 80 80">
          <circle
            class="progress-ring-bg"
            stroke="var(--border)"
            stroke-width="6"
            fill="transparent"
            r="34"
            cx="40"
            cy="40"
          />
          <circle
            class="progress-ring-circle"
            stroke="var(--accent)"
            stroke-width="6"
            stroke-linecap="round"
            fill="transparent"
            r="34"
            cx="40"
            cy="40"
            :stroke-dasharray="strokeDasharray"
            :stroke-dashoffset="strokeDashoffset"
          />
        </svg>
        <div class="progress-content">
          <span class="pct">{{ completionRate }}%</span>
          <span class="lbl">Done</span>
        </div>
      </div>

      <!-- Quick Stats Cards -->
      <div class="stats-cards-grid">
        <div class="stat-card">
          <span class="stat-num">{{ props.totalTasks }}</span>
          <span class="stat-lbl">Total Tasks</span>
        </div>
        <div class="stat-card">
          <span class="stat-num">{{ props.pendingTasks }}</span>
          <span class="stat-lbl">Pending</span>
        </div>
        <div class="stat-card urgent">
          <span class="stat-num text-danger">{{ props.highPriorityTasks }}</span>
          <span class="stat-lbl">High Priority</span>
        </div>
      </div>
    </div>

    <!-- Search & Sort Row -->
    <div v-if="!props.showStatsOnly" class="search-sort-row">
      <div class="search-container">
        <svg class="search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <circle cx="11" cy="11" r="8"/>
          <line x1="21" y1="21" x2="16.65" y2="16.65"/>
        </svg>
        <input 
          type="text" 
          :value="props.searchQuery"
          @input="emit('update:searchQuery', ($event.target as HTMLInputElement).value)"
          placeholder="Search tasks..." 
          aria-label="Search tasks"
        />
        <button 
          v-if="props.searchQuery" 
          class="clear-search-btn" 
          @click="emit('update:searchQuery', '')"
          aria-label="Clear search"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M18 6L6 18M6 6l12 12"/>
          </svg>
        </button>
      </div>

      <div class="sort-container">
        <label for="sort-select" class="visually-hidden">Sort By</label>
        <select 
          id="sort-select"
          :value="props.sortBy"
          @change="emit('update:sortBy', ($event.target as HTMLSelectElement).value)"
        >
          <option value="createdAt-desc">Date Added (Newest)</option>
          <option value="createdAt-asc">Date Added (Oldest)</option>
          <option value="dueDate-asc">Due Date (Earliest)</option>
          <option value="dueDate-desc">Due Date (Latest)</option>
          <option value="priority-desc">Priority (High to Low)</option>
          <option value="priority-asc">Priority (Low to High)</option>
          <option value="title-asc">Title (A-Z)</option>
        </select>
      </div>
    </div>

    <!-- Filter Bar Panel -->
    <div v-if="!props.showStatsOnly" class="filter-panel glass-panel">
      <!-- Status Tabs -->
      <div class="filter-group">
        <span class="group-label">Status</span>
        <div class="tab-filters">
          <button 
            class="tab-btn" 
            :class="{ active: props.statusFilter === 'all' }"
            @click="emit('update:statusFilter', 'all')"
          >
            All
          </button>
          <button 
            class="tab-btn" 
            :class="{ active: props.statusFilter === 'active' }"
            @click="emit('update:statusFilter', 'active')"
          >
            Active
          </button>
          <button 
            class="tab-btn" 
            :class="{ active: props.statusFilter === 'completed' }"
            @click="emit('update:statusFilter', 'completed')"
          >
            Completed
          </button>
        </div>
      </div>

      <!-- Priority Selector -->
      <div class="filter-group">
        <span class="group-label">Priority</span>
        <div class="priority-filters">
          <button 
            class="p-chip" 
            :class="{ active: props.priorityFilter === 'all' }"
            @click="emit('update:priorityFilter', 'all')"
          >
            All
          </button>
          <button 
            class="p-chip low" 
            :class="{ active: props.priorityFilter === 'low' }"
            @click="emit('update:priorityFilter', 'low')"
          >
            Low
          </button>
          <button 
            class="p-chip medium" 
            :class="{ active: props.priorityFilter === 'medium' }"
            @click="emit('update:priorityFilter', 'medium')"
          >
            Medium
          </button>
          <button 
            class="p-chip high" 
            :class="{ active: props.priorityFilter === 'high' }"
            @click="emit('update:priorityFilter', 'high')"
          >
            High
          </button>
        </div>
      </div>

      <!-- Tags Filters -->
      <div class="filter-group">
        <span class="group-label">Category</span>
        <div class="tag-filters">
          <button 
            class="tag-filter-chip"
            :class="{ active: props.tagFilter === 'all' }"
            @click="emit('update:tagFilter', 'all')"
          >
            All
          </button>
          <button 
            v-for="tag in props.tags"
            :key="tag.name"
            class="tag-filter-chip custom-tag"
            :class="{ active: props.tagFilter === tag.name }"
            :style="{ 
              '--tag-color': tag.color,
              borderColor: props.tagFilter === tag.name ? tag.color : 'var(--border)',
              backgroundColor: props.tagFilter === tag.name ? tag.color + '15' : 'transparent',
              color: props.tagFilter === tag.name ? tag.color : 'var(--text)'
            }"
            @click="emit('update:tagFilter', props.tagFilter === tag.name ? 'all' : tag.name)"
          >
            <span class="dot" :style="{ backgroundColor: tag.color }"></span>
            {{ tag.name }}
          </button>
        </div>
      </div>
    </div>
    <!-- Detailed Analytics Grid -->
    <div v-if="props.showStatsOnly" class="analytics-grid">
      <!-- Priority Distribution Panel -->
      <div class="glass-panel analytics-card">
        <h4 class="card-title">Priority Completion</h4>
        <div class="priority-chart">
          <!-- High priority bar -->
          <div class="priority-chart-row">
            <div class="priority-info">
              <span class="p-label high">High</span>
              <span class="p-counts">{{ highPriorityCompleted }}/{{ highPriorityTasksList.length }}</span>
            </div>
            <div class="bar-track">
              <div class="bar-fill bg-danger" :style="{ width: highPriorityPercent + '%' }"></div>
            </div>
            <span class="p-percent">{{ highPriorityPercent }}%</span>
          </div>

          <!-- Medium priority bar -->
          <div class="priority-chart-row">
            <div class="priority-info">
              <span class="p-label medium">Medium</span>
              <span class="p-counts">{{ mediumPriorityCompleted }}/{{ mediumPriorityTasksList.length }}</span>
            </div>
            <div class="bar-track">
              <div class="bar-fill bg-warning" :style="{ width: mediumPriorityPercent + '%' }"></div>
            </div>
            <span class="p-percent">{{ mediumPriorityPercent }}%</span>
          </div>

          <!-- Low priority bar -->
          <div class="priority-chart-row">
            <div class="priority-info">
              <span class="p-label low">Low</span>
              <span class="p-counts">{{ lowPriorityCompleted }}/{{ lowPriorityTasksList.length }}</span>
            </div>
            <div class="bar-track">
              <div class="bar-fill bg-success" :style="{ width: lowPriorityPercent + '%' }"></div>
            </div>
            <span class="p-percent">{{ lowPriorityPercent }}%</span>
          </div>
        </div>
      </div>

      <!-- Categories Progress Panel -->
      <div class="glass-panel analytics-card">
        <h4 class="card-title">Category Progress</h4>
        <div v-if="categoryStats.length === 0 || !categoryStats.some(s => s.total > 0)" class="empty-category-stats">
          <p>No active tasks in any categories.</p>
        </div>
        <div v-else class="categories-progress-list">
          <div v-for="stat in categoryStats" :key="stat.name">
            <div v-if="stat.total > 0" class="category-stat-row">
              <div class="cat-info">
                <span class="cat-name-chip" :style="{ color: stat.color, backgroundColor: stat.color + '15', borderColor: stat.color + '30' }">
                  <span class="dot" :style="{ backgroundColor: stat.color }"></span>
                  {{ stat.name }}
                </span>
                <span class="cat-counts">{{ stat.completed }}/{{ stat.total }} done</span>
              </div>
              <div class="bar-track">
                <div class="bar-fill" :style="{ width: stat.percent + '%', backgroundColor: stat.color }"></div>
              </div>
              <span class="cat-percent">{{ stat.percent }}%</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.flex-col {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

/* Stats panel */
.stats-panel {
  display: flex;
  align-items: center;
  gap: 2rem;
  padding: 1.25rem 1.5rem;
}

.progress-widget {
  position: relative;
  width: 76px;
  height: 76px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}

.progress-ring {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transform: rotate(-90deg);
}

.progress-ring-circle {
  transition: stroke-dashoffset 0.4s ease;
}

.progress-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  line-height: 1;
}

.progress-content .pct {
  font-family: var(--font-mono);
  font-size: 1.15rem;
  font-weight: 700;
  color: var(--text-heading);
}

.progress-content .lbl {
  font-size: 0.65rem;
  color: var(--text-muted);
  text-transform: uppercase;
  font-weight: 600;
  letter-spacing: 0.05em;
  margin-top: 2px;
}

.stats-cards-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1rem;
  flex-grow: 1;
}

.stat-card {
  display: flex;
  flex-direction: column;
  justify-content: center;
  background-color: oklch(from var(--text) l c h / 0.03);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 10px 14px;
  text-align: center;
}

.stat-num {
  font-size: 1.5rem;
  font-weight: 700;
  color: var(--text-heading);
  line-height: 1.2;
}

.stat-lbl {
  font-size: 0.75rem;
  color: var(--text-muted);
  font-weight: 500;
  margin-top: 2px;
}

.text-danger {
  color: var(--danger);
}

/* Search and Sort row */
.search-sort-row {
  display: flex;
  gap: 12px;
}

.search-container {
  position: relative;
  flex-grow: 1;
}

.search-icon {
  position: absolute;
  left: 14px;
  top: 50%;
  transform: translateY(-50%);
  width: 18px;
  height: 18px;
  color: var(--text-muted);
  pointer-events: none;
}

.search-container input {
  padding-left: 44px;
  padding-right: 40px;
  height: 44px;
}

.clear-search-btn {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  width: 24px;
  height: 24px;
  min-height: 0;
  padding: 0;
  border-radius: 50%;
  color: var(--text-muted);
  display: flex;
  align-items: center;
  justify-content: center;
}

.clear-search-btn:hover {
  background-color: var(--border);
  color: var(--text-heading);
}

.clear-search-btn svg {
  width: 14px;
  height: 14px;
}

.sort-container select {
  height: 44px;
  padding-right: 28px;
  min-width: 180px;
}

/* Filter panel */
.filter-panel {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  padding: 1.25rem 1.5rem;
}

.filter-group {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.group-label {
  font-size: 0.8rem;
  font-weight: 600;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.05em;
  width: 80px;
  flex-shrink: 0;
}

/* Tab filters for status */
.tab-filters {
  display: flex;
  background-color: oklch(from var(--text) l c h / 0.04);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 3px;
}

.tab-btn {
  padding: 6px 16px;
  min-height: 0;
  font-size: 0.85rem;
  font-weight: 500;
  color: var(--text-muted);
  border-radius: 6px;
  transition: all 0.15s ease;
}

.tab-btn.active {
  background-color: var(--card-bg);
  color: var(--text-heading);
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
  font-weight: 600;
}

/* Priority chips */
.priority-filters {
  display: flex;
  gap: 8px;
}

.p-chip {
  padding: 6px 12px;
  min-height: 0;
  border-radius: 6px;
  font-size: 0.8rem;
  font-weight: 500;
  border: 1px solid var(--border);
  background-color: transparent;
  color: var(--text-muted);
}

.p-chip.active {
  background-color: var(--border);
  color: var(--text-heading);
  font-weight: 600;
}

.p-chip.low.active {
  background-color: oklch(from var(--success) l c h / 0.1);
  border-color: var(--success);
  color: var(--success);
}

.p-chip.medium.active {
  background-color: oklch(from var(--warning) l c h / 0.1);
  border-color: var(--warning);
  color: var(--warning);
}

.p-chip.high.active {
  background-color: oklch(from var(--danger) l c h / 0.1);
  border-color: var(--danger);
  color: var(--danger);
}

/* Tag filter chips */
.tag-filters {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.tag-filter-chip {
  padding: 5px 12px;
  min-height: 0;
  border-radius: 999px;
  font-size: 0.75rem;
  font-weight: 500;
  border: 1px solid var(--border);
  background-color: transparent;
  color: var(--text-muted);
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.tag-filter-chip.active {
  background-color: var(--border);
  color: var(--text-heading);
  font-weight: 600;
}

.tag-filter-chip.custom-tag {
  border-color: var(--border);
}

.tag-filter-chip.custom-tag.active {
  font-weight: 600;
}

.tag-filter-chip .dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
}

/* Analytics Grid & Cards */
.analytics-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1.25rem;
  margin-top: 0.5rem;
}

@media (min-width: 800px) {
  .analytics-grid {
    grid-template-columns: 1fr 1fr;
  }
}

.analytics-card {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
  padding: 1.5rem;
}

.card-title {
  font-size: 1.05rem;
  font-weight: 700;
  color: var(--text-heading);
  border-bottom: 1px solid var(--border);
  padding-bottom: 0.75rem;
}

/* Charts */
.priority-chart, .categories-progress-list {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.priority-chart-row, .category-stat-row {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  position: relative;
}

.priority-info, .cat-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.85rem;
  font-weight: 600;
  padding-right: 50px;
}

.p-label.high {
  color: var(--danger);
}

.p-label.medium {
  color: var(--warning);
}

.p-label.low {
  color: var(--success);
}

.p-counts, .cat-counts {
  color: var(--text-muted);
  font-size: 0.8rem;
}

.bar-track {
  width: 100%;
  height: 8px;
  background-color: var(--border);
  border-radius: 99px;
  overflow: hidden;
}

.bar-fill {
  height: 100%;
  border-radius: 99px;
  transition: width 0.4s ease;
}

.bar-fill.bg-danger {
  background-color: var(--danger);
}

.bar-fill.bg-warning {
  background-color: var(--warning);
}

.bar-fill.bg-success {
  background-color: var(--success);
}

.p-percent, .cat-percent {
  position: absolute;
  right: 0;
  top: 0;
  font-size: 0.85rem;
  font-weight: 700;
  color: var(--text-heading);
}

/* Category chip inside stats */
.cat-name-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 3px 8px;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.03em;
  border: 1px solid transparent;
}

.cat-name-chip .dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
}

.empty-category-stats {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2rem;
  text-align: center;
  color: var(--text-muted);
  font-size: 0.85rem;
}

@media (max-width: 768px) {
  .stats-panel {
    flex-direction: column;
    gap: 1.25rem;
    align-items: stretch;
  }
  
  .progress-widget {
    align-self: center;
  }
  
  .stats-cards-grid {
    gap: 0.5rem;
  }
  
  .stat-card {
    padding: 6px;
  }
  
  .search-sort-row {
    flex-direction: column;
  }
  
  .sort-container select {
    width: 100%;
  }
  
  .filter-group {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
  }
  
  .group-label {
    width: auto;
  }
}
</style>
