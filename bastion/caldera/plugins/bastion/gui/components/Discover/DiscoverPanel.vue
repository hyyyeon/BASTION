<!-- DiscoverPanel.vue: Kibana Discover 레이아웃과 중앙 상태 관리 컴포넌트 -->
<template>
  <section class="discover-panel">
    <header class="discover-header">
      <div class="title-group">
        <p class="eyebrow">Bastion</p>
        <h2 class="title">Discover</h2>
        <p class="subtitle">Search and inspect indexed events</p>
      </div>
    </header>

    <div class="control-bar">
      <IndexSelector
        class="control-item index"
        :indices="indexOptions"
        :selected="selectedIndex"
        :disabled="!indices.length"
        @update:selected="handleIndexChange"
      />
      <SearchBar
        class="control-item search"
        :value="kql"
        :loading="isSearching"
        @update:value="handleKqlChange"
        @submit="handleSubmit"
      />
      <TimeRangePicker
        class="control-item time"
        :value="timeRange"
        @update:value="handleTimeRangeChange"
      />
    </div>

    <div class="discover-body">
      <aside :class="['sidebar', { collapsed: !showSidebar }]">
        <FieldSidebar
          :fields="availableFields"
          :selected="visibleColumns"
          :show-document="showDocument"
          :open="showSidebar"
          @toggle="toggleSidebar"
          @toggle-field="toggleFieldColumn"
          @toggle-document="toggleDocument"
        />
      </aside>

      <main class="results">
        <ResultTable
          :results="pagedResults"
          :loading="isSearching"
          :current-index="selectedIndex"
          :page="page"
          :page-size="pageSize"
          :total="totalRows"
          @update:page="handlePage"
          @update:page-size="handlePageSize"
        />
      </main>
    </div>
  </section>
</template>

<script setup>
import { ref, computed, reactive, onMounted, inject } from 'vue';

import IndexSelector from './IndexSelector.vue';
import SearchBar from './SearchBar.vue';
import TimeRangePicker from './TimeRangePicker.vue';
import FieldSidebar from './FieldSidebar.vue';
import ResultTable from './ResultTable.vue';

const $api = inject('$api');

// ─ 상태 ─
const indices = ref([]);
const selectedIndex = ref('');
const kql = ref('');
// 기본 시간 범위: now-7d ~ now (Kibana Discover 유사)
const timeRange = reactive({
  from: 'now-7d',
  to: 'now'
});

const fieldFilters = ref([]);
const showFilters = ref(true);

// Document(전체 로그) 토글
const showDocument = ref(true);
// 사용자가 선택한 추가 컬럼
const visibleColumns = ref([]);

const page = ref(1);
const pageSize = ref(100); // Kibana 기본 100 rows와 유사

// ─ 인덱스 옵션(Data View 느낌) ─
const indexOptions = computed(() => {
  const list = indices.value || [];
  const hasWazuhAlerts = list.some((name) => typeof name === 'string' && name.startsWith('wazuh-alerts-'));

  const opts = [];
  if (hasWazuhAlerts) {
    opts.push({ value: 'wazuh-alerts-*', label: 'wazuh-alerts-*', hint: 'Data view (pattern)' });
  }
  for (const name of list) {
    opts.push({ value: name, label: name, hint: '' });
  }
  return opts;
});

// ─ 결과/필드/컬럼 ─
const results = ref({ total: 0, columns: [], rows: [] });

const availableFields = computed(() => {
  const cols = results.value?.columns || [];
  return cols.filter(Boolean);
});

const totalRows = computed(() => {
  const t = results.value?.total;
  if (typeof t === 'number') return t;
  if (t && typeof t === 'object' && typeof t.value === 'number') return t.value;
  return (results.value?.rows || []).length;
});

const tableResults = computed(() => {
  const base = [];
  if (availableFields.value.includes('@timestamp')) base.push('@timestamp');
  const extras = visibleColumns.value.filter((c) => c !== '@timestamp');
  const extrasUnique = Array.from(new Set(extras));
  const doc = showDocument.value ? ['__document__'] : [];
  return {
    total: totalRows.value,
    columns: [...base, ...extrasUnique, ...doc],
    rows: results.value?.rows || []
  };
});

// 백엔드가 size/offset으로 페이징하므로 slice 금지
const pagedResults = computed(() => tableResults.value);

const isSearching = ref(false);
const showSidebar = ref(true);

// ─ API ─
async function searchLogs({ index, kql, timeRange }) {
  if (!$api) {
    console.warn('[Discover] $api 주입 실패');
    return { total: 0, columns: [], rows: [] };
  }
  try {
    const { data } = await $api.post('/api/discover/search', {
      index,
      from: timeRange.from,
      to: timeRange.to,
      query: kql || '*',
      size: pageSize.value,
      offset: (page.value - 1) * pageSize.value
    });
    return data;
  } catch (e) {
    console.error('[Discover] 검색 실패', e);
    return { total: 0, columns: [], rows: [] };
  }
}

const runSearch = async () => {
  if (!selectedIndex.value) {
    results.value = { total: 0, columns: [], rows: [] };
    return;
  }
  isSearching.value = true;
  try {
    const data = await searchLogs({
      index: selectedIndex.value,
      kql: kql.value,
      timeRange: { ...timeRange }
    });
    results.value = data;
    // 선택 컬럼이 결과에 없으면 제거
    const set = new Set(availableFields.value);
    visibleColumns.value = (visibleColumns.value || []).filter((c) => set.has(c));
    if (!showDocument.value && visibleColumns.value.length === 0 && availableFields.value.length) {
      visibleColumns.value = [availableFields.value[0]];
    }
  } finally {
    isSearching.value = false;
  }
};

const loadIndices = async () => {
  if (!$api) return;
  try {
    const { data } = await $api.get('/api/discover/indices');
    const list = (data || []).filter(Boolean);
    const userIndices = list.filter((i) => i && !i.startsWith('.'));
    const systemIndices = list.filter((i) => i && i.startsWith('.'));
    indices.value = [...userIndices, ...systemIndices];

    if (!selectedIndex.value && indices.value.length > 0) {
      const hasWazuhAlerts = indices.value.some((name) => typeof name === 'string' && name.startsWith('wazuh-alerts-'));
      selectedIndex.value = hasWazuhAlerts ? 'wazuh-alerts-*' : indices.value[0];
    }
    if (selectedIndex.value) {
      await runSearch();
    }
  } catch (e) {
    console.error('[Discover] 인덱스 로드 실패', e);
  }
};

// ─ 토글/이벤트 ─
const toggleFieldColumn = (field) => {
  if (!field || field === '__document__') return;
  const idx = visibleColumns.value.indexOf(field);
  if (idx === -1) visibleColumns.value.push(field);
  else visibleColumns.value.splice(idx, 1);
  if (!showDocument.value && visibleColumns.value.length === 0 && availableFields.value.length) {
    visibleColumns.value = [availableFields.value[0]];
  }
};

const toggleDocument = () => {
  showDocument.value = !showDocument.value;
  if (!showDocument.value && visibleColumns.value.length === 0 && availableFields.value.length) {
    const fallback = ['message', 'full_log', 'data.full_log', 'rule.description']
      .find((f) => availableFields.value.includes(f));
    if (fallback) visibleColumns.value = [fallback];
    else {
      const first = availableFields.value.find((f) => f !== '@timestamp') || availableFields.value[0];
      visibleColumns.value = first ? [first] : [];
    }
  }
};

const handleIndexChange = (value) => {
  selectedIndex.value = value;
  page.value = 1;
  runSearch();
};

const handleKqlChange = (value) => { kql.value = value; };

const handleSubmit = () => {
  page.value = 1;
  runSearch();
};

const handleTimeRangeChange = (value) => {
  timeRange.from = value.from;
  timeRange.to = value.to;
  page.value = 1;
  runSearch();
};

const toggleSidebar = () => { showSidebar.value = !showSidebar.value; };

const handlePage = (val) => {
  page.value = val;
  runSearch();
};

const handlePageSize = (val) => {
  pageSize.value = val;
  page.value = 1;
  runSearch();
};

onMounted(() => {
  loadIndices();
});
</script>

<style scoped>
.discover-panel {
  background: #0f172a;
  border: 1px solid #1f2937;
  border-radius: 10px;
  padding: 18px;
  color: #e5e7eb;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
  display: flex;
  flex-direction: column;
  height: 100%;
  min-height: 0;
}

.discover-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1.25rem;
  border-bottom: 1px solid #1f2937;
  padding-bottom: 0.85rem;
  margin-bottom: 0.5rem;
}

.title-group .eyebrow {
  font-size: 0.75rem;
  letter-spacing: 0.1em;
  color: #94a3b8;
  text-transform: uppercase;
  margin-bottom: 0.25rem;
}

.title-group .title {
  font-size: 1.35rem;
  font-weight: 700;
  margin: 0;
}

.title-group .subtitle {
  margin: 0.15rem 0 0;
  color: #cbd5e1;
  font-size: 0.95rem;
}

.control-bar {
  display: flex;
  gap: 0.6rem;
  align-items: stretch;
  justify-content: flex-start;
  margin-bottom: 0.75rem;
  flex-wrap: wrap;
}

.control-item.index { min-width: 220px; }
.control-item.search { flex: 1; min-width: 320px; }
.control-item.time { min-width: 260px; }

.discover-body {
  display: flex;
  gap: 1rem;
  flex: 1;
  min-height: 0;
}

.sidebar {
  flex: 0 0 320px;
  min-width: 260px;
  min-height: 0;
}

.sidebar.collapsed {
  flex: 0 0 80px;
  min-width: 80px;
}

.results {
  flex: 1;
  min-width: 0;
  min-height: 0;
  display: flex;
  flex-direction: column;
}

@media (max-width: 960px) {
  .discover-header { flex-direction: column; align-items: flex-start; }
  .discover-body { flex-direction: column; }
  .sidebar { flex: 1; }
}
</style>
