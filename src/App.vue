<template>
  <div class="json-viewer-app">
    <header class="top-bar">
      <div class="url-input-group">
        <input v-model="apiUrl" type="text" placeholder="Enter JSON API URL..." @keyup.enter="fetchJson" />
        <button class="primary-btn" @click="fetchJson" :disabled="loading">
          {{ loading ? 'Fetching...' : 'Load' }}
        </button>
      </div>

      <div class="toolbar" v-if="jsonData">
        <input v-model="rawSearch" type="text" placeholder="Search keys or values..." class="search-input" />

        <label class="toggle-container" title="Collapse tree before applying new search">
          <input type="checkbox" v-model="autoCollapse" />
          <span class="slider"></span>
          <span class="toggle-label">Auto-Collapse</span>
        </label>

        <button class="icon-btn" @click="expandAll" title="Expand All">[+]</button>
        <button class="icon-btn" @click="collapseAll" title="Collapse All">[-]</button>
      </div>
    </header>

    <div v-if="error" class="error-msg">
      > ERROR: {{ error }}
    </div>

    <main class="viewer-container" v-if="jsonData">
      <JsonNode :data="jsonData" :name="'Root'" :depth="0" :search="searchQuery" :expandSignal="expandTrigger"
        :collapseSignal="collapseTrigger" />
    </main>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick } from 'vue';
import JsonNode from './components/JsonNode.vue';

const apiUrl = ref('');
const jsonData = ref(null);
const loading = ref(false);
const error = ref(null);

// Search and Toggle States
const rawSearch = ref('');     // What the user types
const searchQuery = ref('');   // What is passed to the nodes
const autoCollapse = ref(true); // Toggle state

const expandTrigger = ref(0);
const collapseTrigger = ref(0);

const expandAll = () => expandTrigger.value++;
const collapseAll = () => collapseTrigger.value++;

// Watch the user's typing to coordinate the collapse -> search flow
watch(rawSearch, async (newVal) => {
  if (autoCollapse.value) {
    collapseAll();
    // Wait for Vue to process the collapse signal across all recursive components
    await nextTick();
  }
  // Now apply the new search string to trigger the expansion
  searchQuery.value = newVal;
});

const fetchJson = async () => {
  if (!apiUrl.value) return;

  loading.value = true;
  error.value = null;
  jsonData.value = null;
  rawSearch.value = '';

  try {
    const response = await fetch(apiUrl.value);
    if (!response.ok) throw new Error(`HTTP ${response.status}: Failed to fetch`);
    jsonData.value = await response.json();

    const url = new URL(window.location);
    url.searchParams.set('url', apiUrl.value);
    window.history.pushState({}, '', url);
  } catch (err) {
    error.value = err.message;
  } finally {
    loading.value = false;
  }
};

onMounted(() => {
  document.body.style.backgroundColor = '#11111b';
  document.body.style.color = '#cdd6f4';
  document.body.style.margin = '0';

  const params = new URLSearchParams(window.location.search);
  const queryUrl = params.get('url');
  if (queryUrl) {
    apiUrl.value = queryUrl;
    fetchJson();
  }
});
</script>

<style scoped>
* {
  box-sizing: border-box;
}

/* Previous Styles Remain */
.json-viewer-app {
  max-width: 1000px;
  margin: 0 auto;
  padding: 30px 20px;
  font-family: 'Inter', sans-serif;
}

.top-bar {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 24px;
}

.url-input-group {
  display: flex;
  gap: 10px;
}

input[type="text"] {
  flex-grow: 1;
  padding: 12px 16px;
  background: #1e1e2e;
  border: 1px solid #313244;
  color: #cdd6f4;
  border-radius: 6px;
  font-family: monospace;
}

input[type="text"]:focus {
  outline: none;
  border-color: #89b4fa;
}

.primary-btn {
  padding: 0 24px;
  background-color: #89b4fa;
  color: #11111b;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
}

.primary-btn:disabled {
  background-color: #45475a;
  color: #a6adc8;
}

.toolbar {
  display: flex;
  align-items: center;
  gap: 12px;
}

.search-input {
  flex-grow: 1;
  background: #181825;
  padding: 8px 12px;
}

.icon-btn {
  background: #1e1e2e;
  color: #a6adc8;
  border: 1px solid #313244;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-family: monospace;
}

.icon-btn:hover {
  background: #313244;
  color: #cdd6f4;
}

.error-msg {
  color: #f38ba8;
  padding: 16px;
  background: rgba(243, 139, 168, 0.1);
  border: 1px solid #f38ba8;
  border-radius: 6px;
  margin-bottom: 20px;
  font-family: monospace;
}

.viewer-container {
  background: #181825;
  border: 1px solid #313244;
  border-radius: 8px;
  padding: 16px;
  overflow-x: auto;
}

.viewer-container>.json-node {
  min-width: 600px;
}

/* --- New Toggle Switch Styles --- */
.toggle-container {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  color: #a6adc8;
  font-size: 13px;
  font-family: monospace;
  user-select: none;
  margin: 0 8px;
}

.toggle-container input {
  display: none;
}

.slider {
  width: 36px;
  height: 20px;
  background-color: #313244;
  border-radius: 20px;
  position: relative;
  transition: background-color 0.3s;
}

.slider::before {
  content: "";
  position: absolute;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background-color: #a6adc8;
  top: 3px;
  left: 3px;
  transition: transform 0.3s, background-color 0.3s;
}

.toggle-container input:checked+.slider {
  background-color: #89b4fa;
}

.toggle-container input:checked+.slider::before {
  transform: translateX(16px);
  background-color: #11111b;
}

.toggle-label {
  margin-top: 1px;
}

/* --- Mobile Responsiveness --- */
@media (max-width: 600px) {
  .json-viewer-app {
    padding: 16px 12px;
  }

  .url-input-group {
    flex-direction: column;
  }

  .primary-btn {
    width: 100%;
    padding: 14px;
  }

  .toolbar {
    flex-wrap: wrap;
    gap: 10px;
  }

  .search-input {
    width: 100%;
    flex-grow: 1;
  }

  .toggle-container {
    margin: 0;
    flex-grow: 1;
  }

  .icon-btn {
    flex-grow: 1;
    text-align: center;
    padding: 12px;
  }
}
</style>