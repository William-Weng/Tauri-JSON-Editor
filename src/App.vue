<script setup lang="ts">
import { ref, watch } from 'vue';
import VueJsonPretty from 'vue-json-pretty';
import 'vue-json-pretty/lib/styles.css';

const rawJsonInput = ref(`{
  "name": "Tauri-JSON-Editor",
  "version": "1.0.0",
  "isAwesome": true,
  "features": [
    "Real-time JSON validation",
    "Pretty formatting"
  ],
  "details": {
    "level": 1,
    "active": null
  }
}`);

const error = ref('');
const showLineNumber = ref(true);
const showLength = ref(true);
const isEditable = ref(true);

// To store the last valid parsed JSON object
const displayData = ref({});

// Initialize with the default valid JSON
try {
  displayData.value = JSON.parse(rawJsonInput.value);
} catch (e) {
  // Initial content might be invalid in some cases
  displayData.value = { error: "Invalid initial JSON" };
}

// Watch for input changes to validate and update
watch(rawJsonInput, (newInput) => {
  try {
    const parsed = JSON.parse(newInput);
    displayData.value = parsed; // On success, update the data to be displayed
    error.value = ''; // and clear any existing error
  } catch (e: any) {
    error.value = e.message; // On failure, set the error message
    // Importantly, we DO NOT modify displayData, so it keeps showing the last valid state
  }
}, { immediate: false }); // immediate:false because we already initialized it

function formatJson() {
  try {
    const parsed = JSON.parse(rawJsonInput.value);
    rawJsonInput.value = JSON.stringify(parsed, null, 2);
    error.value = ''; // Clear error on successful format
  } catch (e: any) {
    error.value = `Invalid JSON: ${e.message}`;
  }
}

function minifyJson() {
  try {
    const parsed = JSON.parse(rawJsonInput.value);
    rawJsonInput.value = JSON.stringify(parsed);
    error.value = ''; // Clear error on successful minify
  } catch (e: any) {
    error.value = `Invalid JSON: ${e.message}`;
  }
}

// eslint-disable-next-line @typescript-eslint/no-explicit-any
function handleEdit(newData: any) {
  // When data is edited in vue-json-pretty, it emits the new data.
  // We update our source of truth (rawJsonInput) to reflect this change.
  rawJsonInput.value = JSON.stringify(newData, null, 2);
}

</script>

<template>
  <main class="container">
    <div class="panels">
      <div class="panel">
        <div class="panel-header">
          <h2>Input JSON</h2>
          <div class="button-group format-buttons">
            <button @click="formatJson">Format</button>
            <button @click="minifyJson">Minify</button>
          </div>
        </div>
        <textarea v-model="rawJsonInput"></textarea>
        <div v-if="error" class="error-display">{{ error }}</div>
      </div>
      <div class="panel">
        <div class="panel-header">
          <h2>Formatted Output</h2>
          <div class="button-group">
            <button @click="showLineNumber = !showLineNumber" :class="{ active: showLineNumber }">Line No.</button>
            <button @click="showLength = !showLength" :class="{ active: showLength }">Length</button>
            <button @click="isEditable = !isEditable" :class="{ active: isEditable }">Editable</button>
          </div>
        </div>
        <vue-json-pretty
          theme="dark"
          :data="displayData"
          :deep="3"
          show-line
          :show-line-number="showLineNumber"
          :show-length="showLength"
          :editable="isEditable"
          @update:data="handleEdit"
        />
      </div>
    </div>
  </main>
</template>

<style scoped>
:global(body) {
  background-color: #1a1a1a;
  color: #e0e0e0;
}

.container {
  padding: 2rem;
  height: 100vh;
  box-sizing: border-box;
}

h1, h2 {
  color: #f5f5f5;
}

h1 {
  margin-top: 0;
  text-align: center;
}

.panels {
  display: flex;
  gap: 2rem;
  height: 100%;
}

.panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

h2 {
  margin-top: 0;
  font-size: 1.2rem;
  margin-bottom: 0;
}

.button-group {
  display: flex;
  gap: 0.5rem;
}

button {
  padding: 0.5em 1em;
  border: 1px solid #444;
  background-color: #3a3a3a;
  color: #e0e0e0;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background-color 0.2s, border-color 0.2s;
}

button:hover {
  background-color: #4a4a4a;
  border-color: #666;
}

button:active {
  background-color: #5a5a5a;
}

button.active {
  background-color: #5c7e10;
  border-color: #7db117;
  color: #fff;
}

button.active:hover {
  background-color: #6f9a14;
}

.format-buttons button {
  background-color: #007bff;
  color: white;
}

.format-buttons button:hover {
  background-color: #0056b3;
}

textarea {
  flex-grow: 1;
  border-radius: 8px;
  border: 1px solid #444;
  padding: 0.8em;
  font-family: Menlo, Monaco, 'Courier New', monospace;
  resize: none;
  font-size: 1.15em;
  line-height: 1.5;
  background-color: #2a2a2a;
  color: #e0e0e0;
}

textarea:focus {
  outline: none;
  border-color: #5c7e10;
}

.error-display {
  color: #ff8a80;
  margin-top: 1rem;
  font-family: monospace;
  background-color: #4d2a2a;
  padding: 0.5rem;
  border-radius: 4px;
  border: 1px solid #ff5252;
}

/* Style for the vue-json-pretty component */
.panel :deep(.vjs-tree) {
  height: 100%;
  overflow-y: auto;
  border: 1px solid #444;
  border-radius: 8px;
  padding: 1em;
  font-size: 1.3em; /* Match textarea font size */
  font-family: Menlo, Monaco, 'Courier New', monospace; /* Set monospace font */
}

.panel :deep(.vjs-tree .vjs-tree-node) {
  line-height: 1.4;
}
</style>
