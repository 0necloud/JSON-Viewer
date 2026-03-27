<template>
    <div class="json-node" :style="{ backgroundColor: depthColor }">
        <div class="node-header" @click="toggleCollapse" :class="{ 'clickable': isComplex }">
            <span v-if="isComplex" class="toggle-icon">
                {{ isCollapsed ? '▶' : '▼' }}
            </span>

            <span v-if="name !== null" class="node-key" :class="{ 'highlight': isKeyMatch }">
                {{ name }}:
            </span>

            <span v-if="!isComplex" class="node-value" :class="dataType">
                {{ formattedValue }}
            </span>
            <span v-else-if="isCollapsed" class="summary-text">
                {{ isArray ? `[ ${data.length} items ]` : '{ ... }' }}
            </span>
        </div>

        <div v-show="!isCollapsed" v-if="isComplex" class="node-children">
            <template v-if="isArray">
                <div v-for="(item, index) in data" :key="index" class="list-item">
                    <JsonNode :data="item" :depth="depth + 1" :search="search" :expandSignal="expandSignal"
                        :collapseSignal="collapseSignal" />
                </div>
            </template>
            <template v-else>
                <JsonNode v-for="(val, key) in data" :key="key" :name="key" :data="val" :depth="depth + 1"
                    :search="search" :expandSignal="expandSignal" :collapseSignal="collapseSignal" />
            </template>
        </div>
    </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue';

const props = defineProps({
    data: { required: true },
    name: { type: [String, Number], default: null },
    depth: { type: Number, default: 0 },
    search: { type: String, default: '' },
    expandSignal: { type: Number, default: 0 },
    collapseSignal: { type: Number, default: 0 }
});

// Start collapsed by default unless it's the root (depth 0)
const isCollapsed = ref(props.depth > 0);

const toggleCollapse = () => {
    if (isComplex.value) {
        isCollapsed.value = !isCollapsed.value;
    }
};

const dataType = computed(() => {
    if (props.data === null) return 'null';
    if (Array.isArray(props.data)) return 'array';
    return typeof props.data;
});

const isArray = computed(() => dataType.value === 'array');
const isComplex = computed(() => dataType.value === 'object' || dataType.value === 'array');

const formattedValue = computed(() => {
    if (dataType.value === 'string') return `"${props.data}"`;
    if (dataType.value === 'null') return 'null';
    return props.data;
});

// Search and Expand Logic
const isKeyMatch = computed(() => {
    if (!props.search) return false;
    return String(props.name).toLowerCase().includes(props.search.toLowerCase());
});

watch(() => props.search, (newVal) => {
    if (!newVal || !isComplex.value) return;
    // If the search term is anywhere inside this object/array, open it
    if (JSON.stringify(props.data).toLowerCase().includes(newVal.toLowerCase())) {
        isCollapsed.value = false;
    }
});

// Listen to global expand/collapse buttons
watch(() => props.expandSignal, () => { if (isComplex.value) isCollapsed.value = false; });
watch(() => props.collapseSignal, () => { if (isComplex.value) isCollapsed.value = true; });

// Dark mode compatible nested colors
const depthColor = computed(() => {
    const hue = (props.depth * 40) % 360;
    return `hsla(${hue}, 40%, 25%, 0.15)`;
});
</script>

<style scoped>
.json-node {
    margin: 4px 0;
    padding: 6px 8px;
    border-radius: 6px;
    font-family: 'Consolas', 'Monaco', monospace;
    font-size: 14px;
    border: 1px solid rgba(255, 255, 255, 0.05);
    color: #cdd6f4;
}

.node-header {
    display: flex;
    align-items: flex-start;
    gap: 8px;
}

.clickable {
    cursor: pointer;
    user-select: none;
}

.clickable:hover {
    background-color: rgba(255, 255, 255, 0.05);
    border-radius: 4px;
}

.toggle-icon {
    font-size: 11px;
    color: #6c7086;
    margin-top: 3px;
    min-width: 12px;
}

.node-key {
    font-weight: bold;
    color: #89b4fa;
}

.highlight {
    background-color: #f38ba8;
    color: #1e1e2e;
    padding: 0 4px;
    border-radius: 3px;
}

.summary-text {
    color: #6c7086;
    font-style: italic;
}

.node-children {
    margin-top: 4px;
    padding-left: 14px;
    border-left: 1px solid rgba(255, 255, 255, 0.1);
}

.list-item {
    margin-bottom: 4px;
}

/* Syntax Highlighting & Line Break Support */
.string {
    color: #a6e3a1;
    white-space: pre-wrap;
    word-break: break-word;
}

.number {
    color: #fab387;
}

.boolean {
    color: #cba6f7;
    font-weight: bold;
}

.null {
    color: #f38ba8;
    font-style: italic;
}
</style>