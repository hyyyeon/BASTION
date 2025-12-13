<!-- IndexSelector.vue: 인덱스 선택 토글 드롭다운 (Kibana Data View 유사 UX) -->
<template>
  <div class="index-selector">
    <label class="label">Index</label>
    <div class="trigger" :class="{ disabled }" @click="toggle">
      <span class="value">{{ displayValue }}</span>
      <span class="chevron" :class="{ open: isOpen && !disabled }">▼</span>
    </div>

    <div v-if="isOpen && !disabled" class="dropdown">
      <button
        v-for="opt in normalizedOptions"
        :key="opt.value"
        class="dropdown-item"
        :class="{ active: opt.value === props.selected }"
        type="button"
        @click="selectIndex(opt.value)"
      >
        <span class="check">{{ opt.value === props.selected ? '✓' : '' }}</span>
        <span class="item-text">
          <span class="item-label">{{ opt.label }}</span>
          <span v-if="opt.hint" class="item-hint">{{ opt.hint }}</span>
        </span>
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, computed } from 'vue';

const props = defineProps({
  indices: { type: Array, default: () => [] }, // 문자열 배열 또는 { value,label,hint }
  selected: { type: String, default: '' },
  disabled: { type: Boolean, default: false }
});

const emit = defineEmits(['update:selected']);
const isOpen = ref(false);

const normalizedOptions = computed(() => {
  const list = props.indices || [];
  return list
    .map((item) => {
      if (!item) return null;
      if (typeof item === 'string') return { value: item, label: item, hint: '' };
      const value = item.value ?? '';
      if (!value) return null;
      return { value, label: item.label ?? value, hint: item.hint ?? '' };
    })
    .filter(Boolean);
});

const displayValue = computed(() => {
  if (props.selected) return props.selected;
  if (!normalizedOptions.value.length) return '인덱스 없음';
  return '선택 없음';
});

const toggle = () => {
  if (props.disabled) return;
  isOpen.value = !isOpen.value;
};

const selectIndex = (value) => {
  emit('update:selected', value);
  isOpen.value = false;
};

const handleClickOutside = (event) => {
  if (!event.target.closest('.index-selector')) isOpen.value = false;
};

onMounted(() => window.addEventListener('click', handleClickOutside));
onBeforeUnmount(() => window.removeEventListener('click', handleClickOutside));
</script>

<style scoped>
.index-selector {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  min-width: 220px;
}

.label {
  color: #cbd5e1;
  font-size: 0.85rem;
}

.trigger {
  background: #0b1221;
  color: #e5e7eb;
  border: 1px solid #1f2937;
  border-radius: 8px;
  padding: 0.55rem 0.65rem;
  min-height: 42px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  cursor: pointer;
}

.trigger.disabled { cursor: not-allowed; opacity: 0.6; }
.trigger:hover { border-color: #3273dc; }

.value { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

.chevron { transition: transform 0.15s ease; font-size: 0.8rem; }
.chevron.open { transform: rotate(180deg); }

.dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background: #0b1221;
  border: 1px solid #1f2937;
  border-radius: 10px;
  margin-top: 0.3rem;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  z-index: 10;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  max-height: 320px;
  overflow-y: auto;
}

.dropdown-item {
  width: 100%;
  text-align: left;
  padding: 0.55rem 0.7rem;
  background: transparent;
  border: none;
  color: #e5e7eb;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.dropdown-item:hover {
  background: #0f172a;
  color: #bfdbfe;
}

.dropdown-item.active {
  background: rgba(37, 99, 235, 0.12);
  border-left: 3px solid #3273dc;
}

.check {
  width: 18px;
  display: inline-flex;
  justify-content: center;
  font-weight: 800;
}

.item-text { display: flex; flex-direction: column; gap: 0.1rem; }
.item-label { font-weight: 600; }
.item-hint { color: #94a3b8; font-size: 0.82rem; }
</style>
