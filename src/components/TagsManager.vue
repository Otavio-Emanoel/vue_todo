<script setup lang="ts">
import { ref } from 'vue';

interface Tag {
  name: string;
  color: string; // Should be hex or oklch color string
}

const props = defineProps<{
  tags: Tag[];
}>();

const emit = defineEmits<{
  (e: 'update-tags', tags: Tag[]): void;
}>();

const showModal = ref(false);
const newTagName = ref('');
const newTagColor = ref('oklch(62% 0.17 145)'); // Default to emerald

// Preset colors (OKLCH values for beautiful, high-contrast text and backgrounds)
const colorPresets = [
  { name: 'Lavender', value: 'oklch(74% 0.16 284)' },
  { name: 'Emerald', value: 'oklch(68% 0.17 145)' },
  { name: 'Amber', value: 'oklch(78% 0.16 75)' },
  { name: 'Crimson', value: 'oklch(62% 0.21 24)' },
  { name: 'Sky', value: 'oklch(69% 0.15 210)' },
  { name: 'Pink', value: 'oklch(68% 0.19 330)' },
  { name: 'Slate', value: 'oklch(60% 0.05 250)' }
];

const openModal = () => {
  showModal.value = true;
};

const closeModal = () => {
  showModal.value = false;
  newTagName.value = '';
};

const addTag = () => {
  const trimmed = newTagName.value.trim();
  if (!trimmed) return;

  // Check if tag already exists
  if (props.tags.some(t => t.name.toLowerCase() === trimmed.toLowerCase())) {
    alert('This tag already exists!');
    return;
  }

  const updatedTags = [...props.tags, { name: trimmed, color: newTagColor.value }];
  emit('update-tags', updatedTags);
  newTagName.value = '';
};

const deleteTag = (tagName: string) => {
  // Prevent deleting all tags
  if (props.tags.length <= 1) {
    alert('You need to keep at least one category tag!');
    return;
  }
  const updatedTags = props.tags.filter(t => t.name !== tagName);
  emit('update-tags', updatedTags);
};
</script>

<template>
  <div class="tags-manager">
    <button class="secondary w-full" @click="openModal">
      <svg class="icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <path d="M12 5v14M5 12h14"/>
      </svg>
      Manage Categories
    </button>

    <!-- Modal Backdrop -->
    <Teleport to="body">
      <div v-if="showModal" class="modal-backdrop" @click.self="closeModal">
        <div class="glass-panel modal-content animate-pop-in">
          <div class="modal-header">
            <h3>Manage Categories</h3>
            <button class="close-btn" @click="closeModal" aria-label="Close Modal">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M18 6L6 18M6 6l12 12"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Existing Tags -->
            <div class="tags-list-container">
              <h4 class="section-title">Current Tags</h4>
              <div class="tags-grid">
                <div 
                  v-for="tag in props.tags" 
                  :key="tag.name" 
                  class="tag-item"
                  :style="{ 
                    backgroundColor: tag.color + '15', 
                    borderColor: tag.color + '40', 
                    color: tag.color 
                  }"
                >
                  <span class="dot" :style="{ backgroundColor: tag.color }"></span>
                  <span class="tag-name">{{ tag.name }}</span>
                  <button 
                    class="delete-tag-btn" 
                    @click="deleteTag(tag.name)" 
                    title="Delete Tag"
                    aria-label="Delete Tag"
                  >
                    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <path d="M18 6L6 18M6 6l12 12"/>
                    </svg>
                  </button>
                </div>
              </div>
            </div>

            <!-- Create Tag Form -->
            <form @submit.prevent="addTag" class="create-tag-form">
              <h4 class="section-title">Create New Tag</h4>
              <div class="form-group">
                <label for="tag-name-input">Tag Name</label>
                <input 
                  id="tag-name-input"
                  type="text" 
                  v-model="newTagName" 
                  placeholder="e.g. Work, Health, Shopping" 
                  maxlength="20"
                  required
                />
              </div>

              <div class="form-group">
                <label>Select Color</label>
                <div class="color-picker-grid">
                  <button 
                    v-for="preset in colorPresets" 
                    :key="preset.name"
                    type="button"
                    class="color-preset-btn"
                    :class="{ active: newTagColor === preset.value }"
                    :style="{ backgroundColor: preset.value }"
                    @click="newTagColor = preset.value"
                    :title="preset.name"
                  >
                    <svg v-if="newTagColor === preset.value" class="check-icon" viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="3">
                      <polyline points="20 6 9 17 4 12"/>
                    </svg>
                  </button>
                </div>
              </div>

              <button type="submit" class="primary w-full add-btn">
                Add Tag
              </button>
            </form>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<style scoped>
.w-full {
  width: 100%;
}

.modal-backdrop {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(8px);
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1.5rem;
}

.modal-content {
  width: 100%;
  max-width: 480px;
  background-color: var(--card-bg);
  border: 1px solid var(--border);
  box-shadow: var(--shadow);
  display: flex;
  flex-direction: column;
  max-height: 90vh;
  overflow: hidden;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.modal-header h3 {
  font-size: 1.25rem;
  font-weight: 600;
}

.close-btn {
  width: 32px;
  height: 32px;
  padding: 0;
  min-height: 0;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: var(--border);
  color: var(--text-heading);
}

.close-btn:hover {
  background-color: var(--accent);
  color: white;
}

.close-btn svg {
  width: 16px;
  height: 16px;
}

.modal-body {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  overflow-y: auto;
  padding-right: 4px;
}

.section-title {
  font-size: 0.875rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--text-muted);
  margin-bottom: 0.75rem;
}

.tags-list-container {
  border-bottom: 1px solid var(--border);
  padding-bottom: 1.5rem;
}

.tags-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.tag-item {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 6px 12px;
  border-radius: 9999px;
  font-size: 0.85rem;
  font-weight: 500;
  border: 1px solid transparent;
}

.dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

.delete-tag-btn {
  width: 16px;
  height: 16px;
  padding: 0;
  min-height: 0;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: currentColor;
  opacity: 0.6;
}

.delete-tag-btn:hover {
  opacity: 1;
  background-color: rgba(0, 0, 0, 0.1);
}

.delete-tag-btn svg {
  width: 12px;
  height: 12px;
}

.create-tag-form {
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

.color-picker-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.color-preset-btn {
  width: 32px;
  height: 32px;
  min-height: 0;
  padding: 0;
  border-radius: 50%;
  transition: transform 0.2s, outline 0.2s;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.color-preset-btn:hover {
  transform: scale(1.1);
}

.color-preset-btn.active {
  outline: 2px solid var(--text-heading);
  outline-offset: 2px;
}

.check-icon {
  width: 16px;
  height: 16px;
}

.add-btn {
  margin-top: 0.5rem;
}

.icon {
  width: 16px;
  height: 16px;
}
</style>
