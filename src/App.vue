<script setup lang="ts">
import { ref, watch, computed, onMounted, onUnmounted } from 'vue';
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
const fontSize = ref(1.15); // in em
const outputFontSize = ref(1.3); // in em
const isRightPanelVisible = ref(true);
const isLeftPanelVisible = ref(true);

// To store the last valid parsed JSON object
const displayData = ref({});

// Computed property for dynamic styles on the output panel
const outputStyle = computed(() => ({
  fontSize: outputFontSize.value + 'em',
  '--vjs-padding-left': showLineNumber.value ? 'calc(3ch + 4px)' : '4px',
}));

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

function toggleRightPanel() {
  isRightPanelVisible.value = !isRightPanelVisible.value;
}

function toggleLeftPanel() {
  isLeftPanelVisible.value = !isLeftPanelVisible.value;
}

function increaseFontSize() {
  fontSize.value += 0.1;
}

function decreaseFontSize() {
  if (fontSize.value > 0.5) { // Prevent it from getting too small
    fontSize.value -= 0.1;
  }
}

function increaseOutputFontSize() {
  outputFontSize.value += 0.1;
}

function decreaseOutputFontSize() {
  if (outputFontSize.value > 0.5) {
    outputFontSize.value -= 0.1;
  }
}

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

// Keyboard shortcuts handler
const handleKeydown = (event: KeyboardEvent) => {
  // Check for Command (metaKey on macOS) or Ctrl (ctrlKey on other OSes)
  if (event.metaKey || event.ctrlKey) {
    switch (event.key) {
      case '=': // Typically the key for '+' without Shift
      case '+':
        event.preventDefault();
        increaseFontSize();
        increaseOutputFontSize();
        break;
      case '-':
        event.preventDefault();
        decreaseFontSize();
        decreaseOutputFontSize();
        break;
    }
  }
};

// Set up and tear down the global event listener
onMounted(() => {
  window.addEventListener('keydown', handleKeydown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown);
});

</script>

<template>
  <main class="container">
    <div class="panels">
      <div class="panel" :class="{ 'panel-hidden': !isLeftPanelVisible }">
        <div class="panel-header">
          <h2>Input JSON</h2>
          <div class="button-group">
            <button @click="formatJson" class="format-button">Format</button>
            <button @click="minifyJson" class="format-button">Minify</button>
            <button @click="toggleRightPanel" :title="isRightPanelVisible ? 'Hide Output Panel' : 'Show Output Panel'" class="toggle-panel-button">{{ isRightPanelVisible ? 'Hide' : 'Show' }}</button>
            <button @click="decreaseFontSize" title="Decrease font size" class="font-size-button">-</button>
            <button @click="increaseFontSize" title="Increase font size" class="font-size-button">+</button>
          </div>
        </div>
        <textarea v-model="rawJsonInput" :style="{ fontSize: fontSize + 'em' }"></textarea>
        <div v-if="error" class="error-display">{{ error }}</div>
      </div>
      <div class="panel" :class="{ 'panel-hidden': !isRightPanelVisible }">
        <div class="panel-header">
          <h2>Formatted Output</h2>
          <div class="button-group">
            <button @click="showLineNumber = !showLineNumber" :class="{ active: showLineNumber }">Line No.</button>
            <button @click="showLength = !showLength" :class="{ active: showLength }">Length</button>
            <button @click="isEditable = !isEditable" :class="{ active: isEditable }">Editable</button>
            <button @click="toggleLeftPanel" :title="isLeftPanelVisible ? 'Hide Input Panel' : 'Show Input Panel'" class="toggle-panel-button">{{ isLeftPanelVisible ? 'Hide' : 'Show' }}</button>
            <button @click="decreaseOutputFontSize" title="Decrease font size" class="font-size-button">-</button>
            <button @click="increaseOutputFontSize" title="Increase font size" class="font-size-button">+</button>
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
          :style="outputStyle"
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
  height: 100%;
  margin: 0 -1rem; /* Absorb child margins */
}

.panel {
  flex: 1 1 50%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  transition: all 0.4s ease-in-out;
  margin: 0 1rem; /* Create gap */
}

.panel-hidden {
  flex: 0 0 0;
  min-width: 0;
  opacity: 0;
  padding: 0;
  border: 0;
  margin: 0;
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

.format-button {
  background-color: #007bff;
  color: white;
}

.format-button:hover {
  background-color: #0056b3;
}

.font-size-button {
  background-color: #d9534f;
  color: white;
}

.font-size-button:hover {
  background-color: #c9302c;
}

.toggle-panel-button {
  background-color: #20c997;
  color: white;
}

.toggle-panel-button:hover {
  background-color: #1baa80;
}

textarea {
  flex-grow: 1;
  border-radius: 8px;
  border: 1px solid #444;
  padding: 0.8em;
  font-family: Menlo, Monaco, 'Courier New', monospace;
  resize: none;
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
  padding-left: var(--vjs-padding-left, 4px) !important;
  font-family: Menlo, Monaco, 'Courier New', monospace; /* Set monospace font */
}

.panel :deep(.vjs-tree .vjs-tree-node) {
  line-height: 1.4;
}
</style>