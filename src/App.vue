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

    <div class="fixed-actions">
      <button class="action-btn" @click="openPasteModal">
        📋 Paste JSON
      </button>
      <a href="https://github.com/0necloud/JSON-Viewer" target="_blank" rel="noopener noreferrer"
        class="action-btn github-btn">
        <svg viewBox="0 0 24 24" width="18" height="18" fill="currentColor">
          <path
            d="M12 0c-6.626 0-12 5.373-12 12 0 5.302 3.438 9.8 8.207 11.387.599.111.793-.261.793-.577v-2.234c-3.338.726-4.033-1.416-4.033-1.416-.546-1.387-1.333-1.756-1.333-1.756-1.089-.745.083-.729.083-.729 1.205.084 1.839 1.237 1.839 1.237 1.07 1.834 2.807 1.304 3.492.997.107-.775.418-1.305.762-1.604-2.665-.305-5.467-1.334-5.467-5.931 0-1.311.469-2.381 1.236-3.221-.124-.303-.535-1.524.117-3.176 0 0 1.008-.322 3.301 1.23.957-.266 1.983-.399 3.003-.404 1.02.005 2.047.138 3.006.404 2.291-1.552 3.297-1.23 3.297-1.23.653 1.653.242 2.874.118 3.176.77.84 1.235 1.911 1.235 3.221 0 4.609-2.807 5.624-5.479 5.921.43.372.823 1.102.823 2.222v3.293c0 .319.192.694.801.576 4.765-1.589 8.199-6.086 8.199-11.386 0-6.627-5.373-12-12-12z" />
        </svg>
        GitHub
      </a>
    </div>

    <div v-if="showModal" class="modal-overlay" @click.self="closePasteModal">
      <div class="modal-content">
        <h3 class="modal-title">Paste Custom JSON</h3>
        <textarea v-model="pastedText" class="paste-area"
          placeholder="{&#10;  &#34;key&#34;: &#34;value&#34;&#10;}"></textarea>
        <div v-if="modalError" class="error-msg modal-error">
          > ERROR: {{ modalError }}
        </div>
        <div class="modal-buttons">
          <button class="secondary-btn" @click="closePasteModal">Cancel</button>
          <button class="primary-btn" @click="applyPastedJson">Render JSON</button>
        </div>
      </div>
    </div>
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
const rawSearch = ref('');
const searchQuery = ref('');
const autoCollapse = ref(true);

const expandTrigger = ref(0);
const collapseTrigger = ref(0);

const expandAll = () => expandTrigger.value++;
const collapseAll = () => collapseTrigger.value++;

const showModal = ref(false);
const pastedText = ref('');
const modalError = ref(null);

const openPasteModal = () => {
  showModal.value = true;
  modalError.value = null;
};

const closePasteModal = () => {
  showModal.value = false;
  modalError.value = null;
  pastedText.value = '';
};

const applyPastedJson = () => {
  if (!pastedText.value.trim()) {
    modalError.value = 'JSON field is empty.';
    return;
  }

  try {
    const parsedData = JSON.parse(pastedText.value);

    jsonData.value = parsedData;
    error.value = null;
    apiUrl.value = '';

    // Clear the URL parameter
    const url = new URL(window.location);
    url.searchParams.delete('url');
    window.history.pushState({}, '', url);

    closePasteModal();
  } catch (err) {
    modalError.value = `Invalid JSON format (${err.message})`;
  }
};

// Watch the user's typing to coordinate the collapse -> search flow
watch(rawSearch, async (newVal) => {
  if (autoCollapse.value) {
    collapseAll();
    // Wait for Vue to process the collapse signal across all recursive components
    await nextTick();
  }
  // Apply the new search string to trigger the expansion
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

.fixed-actions {
  position: fixed;
  bottom: 24px;
  right: 24px;
  display: flex;
  gap: 12px;
  z-index: 100;
}

.action-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 16px;
  border-radius: 50px;
  background-color: #1e1e2e;
  color: #cdd6f4;
  border: 1px solid #313244;
  font-family: 'Inter', sans-serif;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  text-decoration: none;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
  transition: background-color 0.2s, transform 0.2s;
}

.action-btn:hover {
  background-color: #313244;
  transform: translateY(-2px);
}

.github-btn:hover {
  color: #89b4fa;
}

/* Modal Overlay */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(17, 17, 27, 0.8);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 200;
  padding: 20px;
}

.modal-content {
  background: #1e1e2e;
  border: 1px solid #313244;
  border-radius: 12px;
  width: 100%;
  max-width: 600px;
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
}

.modal-title {
  margin: 0;
  font-size: 18px;
  color: #cdd6f4;
}

.paste-area {
  width: 100%;
  min-height: 250px;
  background: #11111b;
  border: 1px solid #313244;
  border-radius: 8px;
  color: #a6e3a1;
  padding: 16px;
  font-family: 'Consolas', monospace;
  font-size: 14px;
  resize: vertical;
}

.paste-area:focus {
  outline: none;
  border-color: #89b4fa;
}

.modal-error {
  margin-bottom: 0;
}

.modal-buttons {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
}

.secondary-btn {
  padding: 0 20px;
  height: 42px;
  background-color: transparent;
  color: #a6adc8;
  border: 1px solid #45475a;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
}

.secondary-btn:hover {
  background-color: #313244;
  color: #cdd6f4;
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

  .fixed-actions {
    bottom: 8px;
    right: 16px;
    left: 16px;
    justify-content: space-between;
  }

  .action-btn {
    flex-grow: 1;
    justify-content: center;
  }

  .modal-content {
    padding: 16px;
  }
}
</style>