<script setup lang="ts">
import { ref, watch, computed, onMounted, onUnmounted } from 'vue';
import VueJsonPretty from 'vue-json-pretty';
import 'vue-json-pretty/lib/styles.css';
import { open, save } from '@tauri-apps/plugin-dialog';
import { readTextFileLines, writeTextFile } from '@tauri-apps/plugin-fs';
import { listen } from "@tauri-apps/api/event";

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

const defaultInputFontSize = 1.2
const defaultOutputFontSize = 1.2
const errorMessage = ref('');
const showLineNumber = ref(true);
const showLength = ref(true);
const isEditable = ref(true);
const inputFontSize = ref(defaultInputFontSize);
const outputFontSize = ref(defaultOutputFontSize);
const isRightPanelVisible = ref(true);
const isLeftPanelVisible = ref(true);
const isFormatMode = ref(false);
const fileExtensions = ['json', 'txt'];
const displayData = ref({});
const textareaBackgroundColor = ref('#444');
const colorInput = ref<HTMLInputElement | null>(null);

let defaultPath = 'untitled.json'

const outputStyle = computed(() => ({
  fontSize: outputFontSize.value + 'em',
  '--vjs-padding-left': showLineNumber.value ? 'calc(3ch + 4px)' : '4px',
}));

try {
  displayData.value = JSON.parse(rawJsonInput.value);
} catch (error) {
  displayData.value = { error: "Invalid initial JSON" };
}

/** 
 * 切換輸入面板的顯示狀態 (左)
 */
function toggleLeftPanel() { isLeftPanelVisible.value = !isLeftPanelVisible.value; }

/** 
 * 切換輸出面板的顯示狀態 (右)
 */
function toggleRightPanel() { isRightPanelVisible.value = !isRightPanelVisible.value; }

/** 
 * 增加輸入面板字體大小 (左)
 */
function increaseFontSize() { inputFontSize.value += 0.1; }

/** 
 * 減少輸入面板字體大小 - 最小限制為 0.5em (左)
 */ 
function decreaseFontSize() {
  if (inputFontSize.value > 0.5) { inputFontSize.value -= 0.1; }
}

/** 
 * 增加輸出面板的字體大小 (右)
 */
function increaseOutputFontSize() {
  outputFontSize.value += 0.1;
}

/** 
 * 減少輸出面板的字體大小 - 最小限制為 0.5em (右) 
 */ 
function decreaseOutputFontSize() {
  if (outputFontSize.value > 0.5) { outputFontSize.value -= 0.1; }
}

/**
 * 回復面板預設字體大小
 */
function resetFontSize() {
  inputFontSize.value = defaultInputFontSize
  outputFontSize.value = defaultOutputFontSize
}

/** 
 * 切換格式化 / 壓縮模式 
 */
function toggleFormatMinify() {
  isFormatMode.value = !isFormatMode.value;
  _toggleFormat(isFormatMode.value);
}

/**
 * 取得設定好的背景色色碼
 */
function defaultBackgroundColor() {
  const savedColor = localStorage.getItem('textareaBackgroundColor');
  if (savedColor) { textareaBackgroundColor.value = savedColor; }
}

/**
 * 開啟調色盤
 */
function openColorPicker() {
  colorInput.value?.click();
}

// MARK: - async functions
/** 
 * 打開文件選擇對話框並讀取 JSON 文件
 */
async function openFile() {

  const filePath = await open({
    multiple: false,
    directory: false,
    filters: [{ name: 'JSON Files', extensions: fileExtensions }],
  });

  displayJSON(filePath as string);
}

/**
 * 打開文件保存對話框並保存 Markdown 文件
 */
async function saveFile() {

  try {
    const filePath = await save({
      filters: [{ name: 'Markdown Files', extensions: fileExtensions }],
      defaultPath: defaultPath
    });

    errorMessage.value = 'File saved successfully!';
    if (!filePath) { errorMessage.value = 'No file path selected.'; return; }
    if (typeof filePath !== 'string') { errorMessage.value = 'Invalid file path selected.'; return; }

    await writeTextFile(filePath, rawJsonInput.value);
    defaultPath = filePath as string;
  } catch (error) {
    errorMessage.value = `Error saving file: ${error}`;
    console.error('Error saving file:', error);
  }
}

/** 
 * 根據文件路徑讀取 JSON 文件並顯示內容
 * @param filePath - 要讀取的文件路徑
 */
async function displayJSON(filePath?: string) {

  if (!filePath) { errorMessage.value = 'file extension is null.'; return; }

  const fileExtension = filePath.split('.').pop();
  if (!fileExtensions.includes(fileExtension || '')) { errorMessage.value = `Unsupported file type: ${fileExtension}`; return; }

  const lines = await readTextFileLines(filePath);
  let parsed = '';

  for await (const line of lines) {
    let newLine = line
    if (line.endsWith('\0')) { newLine = line.slice(0, -1); };
    parsed += newLine + '\n';
  }

  rawJsonInput.value = parsed.trim();
  _toggleFormat(isFormatMode.value);
}

// MARK: - event handlers
/** 處理編輯事件
 * @param newData - 編輯後的新數據
 */
function handleEdit(newData: any) {
  rawJsonInput.value = JSON.stringify(newData, null, 2);
}

/** 處理原始 JSON 輸入變更
 * @param newInput - 新的原始 JSON 輸入
 */
function handleRawJsonInputChange(newInput: string) {
  try {
    const parsed = JSON.parse(newInput);
    displayData.value = parsed;
    errorMessage.value = '';
  } catch (error: any) {
    errorMessage.value = error.message;
  }
}

/**
 * 處理鍵盤事件以增減字體大小 (Command / Ctrl + '+' / '=' / '-')
 * @param event - 鍵盤事件
 */
function handleKeyboardEvent(event: KeyboardEvent) {

  if (event.metaKey || event.ctrlKey) {

    event.preventDefault()

    switch (event.key) {
      case '=': // Typically the key for '+' without Shift
      case '+': increaseFontSize(); increaseOutputFontSize(); break;
      case '-': decreaseFontSize(); decreaseOutputFontSize(); break;
      case '0': resetFontSize(); break;
      case 'o': openFile(); break;
      case 's': saveFile(); break;
    }
  }
}

/**
 * 處理文件拖放事件
 */
function handleFileDragDrop() {

  listen('tauri://drag-drop', (event: any) => {
    if (event?.payload?.paths.length > 0) {
      defaultPath = event.payload.paths[0];
      displayJSON(defaultPath);
    }
  });
}

// MARK: - private functions
/** 切換格式化或壓縮模式
 * @param isFormatMode - 是不是要格式化顯示
 */
function _toggleFormat(isFormatMode: boolean) {

  try {
    const parsed = JSON.parse(rawJsonInput.value);
    rawJsonInput.value = (!isFormatMode) ? JSON.stringify(parsed, null, 2) : JSON.stringify(parsed);
    errorMessage.value = '';
  } catch (error: any) {
    errorMessage.value = `Invalid JSON: ${error.message}`;
  }
}

// MARK: - 生命週期
onMounted(() => {
  window.addEventListener('keydown', handleKeyboardEvent);
  handleFileDragDrop();
  defaultBackgroundColor();
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyboardEvent);
});

watch(rawJsonInput, (newInput: string) => {
  handleRawJsonInputChange(newInput);
}, { immediate: false });

watch(textareaBackgroundColor, (newColor: any) => {
  localStorage.setItem('textareaBackgroundColor', newColor);
});

</script>

<template>
  <main class="container">
    <div class="panels">
      <div class="panel" :class="{ 'panel-hidden': !isLeftPanelVisible }">
        <div class="panel-header">
          <h2>Input JSON</h2>
          <div class="button-group">
            <button @click="openFile" title="Open file" class="open-button">Open</button>
            <button @click="saveFile" title="Save file" class="save-button">Save</button>
            <button @click="toggleFormatMinify" class="format-button">{{ isFormatMode ? 'Format' : 'Minify' }}</button>
            <button @click="toggleRightPanel" :title="isRightPanelVisible ? 'Hide Output Panel' : 'Show Output Panel'" class="toggle-panel-button">{{ isRightPanelVisible ? 'Hide' : 'Show' }}</button>
            <div style="position: relative;">
              <button @click="openColorPicker" title="Change background color">
                Color
                <input type="color" ref="colorInput" v-model="textareaBackgroundColor" style="position: absolute; opacity: 0; width: 0; height: 0; pointer-events: none;" />
              </button>
            </div>
            <button @click="decreaseFontSize" title="Decrease font size" class="font-size-button">-</button>
            <button @click="increaseFontSize" title="Increase font size" class="font-size-button">+</button>
          </div>
        </div>
        <textarea v-model="rawJsonInput" :style="{fontSize: inputFontSize + 'em', backgroundColor: textareaBackgroundColor}"></textarea>
        <div v-if="errorMessage" class="error-display">{{ errorMessage }}</div>
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
  margin: 0;
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

.open-button {
  background-color: #ff0000;
  color: white;
}

.open-button:hover {
  background-color: #c80000;
}

.save-button {
  background-color: #FF00FF; /* A standard brown */
  color: white;
}

.save-button:hover {
  background-color: #e9068f; /* A darker brown for hover */
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
  background-color: #444;
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