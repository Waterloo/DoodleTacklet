# Restore the Vue component
<script setup lang="ts">
import { ref, onMounted, onUnmounted, nextTick } from "vue";
import tackpad from 'tackpad-widget-ts-sdk'
import { fabric } from "fabric";
import {
  Pencil,
  Eraser,
  Minus,
  Square,
  Circle,
  Triangle,
  Text,
  Undo,
  Redo,
  Trash2,
} from "lucide-vue-next";

const canvas = ref<fabric.Canvas | null>(null);
const activeTool = ref("brush");
const activeColor = ref("#000000");
const brushSize = ref(2);
const showColorPicker = ref(false);
const isDrawing = ref(false);
const origX = ref(0);
const origY = ref(0);
const undoStack = ref<string[]>([]);
const redoStack = ref<string[]>([]);
const activeShape = ref<fabric.Object | null>(null);
const canUndo = ref(false);
const canRedo = ref(false);

const tools = [
  { id: "brush", icon: Pencil },
  { id: "eraser", icon: Eraser },
  { id: "line", icon: Minus },
  { id: "rect", icon: Square },
  { id: "circle", icon: Circle },
  { id: "triangle", icon: Triangle },
  { id: "text", icon: Text },
];

const colors = ref([
  { color: "#000000" },
  { color: "#EBCFFF" },
  { color: "#CCEBFF" },
  { color: "#EBFFDC" },
  { color: "#FFDCA8" },
  { color: "#FFD4C4" },
  { color: "#FF9E99" },
  { color: "#FFE3EF" },
]);

const toolsVisibility = ref(true);

onMounted(async () => {

  window.tackpad = tackpad;
  nextTick(async () => {
  tackpad.on('selected', () => {
    toolsVisibility.value = true;
  });
  tackpad.on('deselected', () => {
    toolsVisibility.value = false;
  });
  await tackpad.connect()
  console.log("Initializing canvas")
  initCanvas();
  window.addEventListener("resize", resizeCanvas);
  document.addEventListener("click", (e) => {
    const target = e.target as HTMLElement;
    if (!target.closest(".color-picker-wrapper")) {
      showColorPicker.value = false;
    }
  });
  });
 
});

onUnmounted(() => {
  window.removeEventListener("resize", resizeCanvas);
});

async function initCanvas() {
  const container = document.querySelector(".canvas-container");
  if (!container) return;

  const containerWidth = container.clientWidth;
  const containerHeight = container.clientHeight;

  canvas.value = new fabric.Canvas("drawing-canvas", {
    width: containerWidth,
    height: containerHeight,
    isDrawingMode: true,
    backgroundColor: "#fff",
  });

  if (!canvas.value) return;

  canvas.value.freeDrawingBrush.color = activeColor.value;
  canvas.value.freeDrawingBrush.width = brushSize.value;

  const initalState =await tackpad.getData()
  console.log({initalState})
  if(Object.keys(initalState).length > 0) {
    try {canvas.value.loadFromJSON(
      initalState,
      () => {
        canvas.value?.renderAll();
      }
    );
  } catch (error) {
    console.error("Failed to load initial state:", error);
  }
  }

  canvas.value.on("mouse:down", onMouseDown);
  canvas.value.on("mouse:move", onMouseMove);
  canvas.value.on("mouse:up", onMouseUp);
  canvas.value.on("object:modified", () => saveState());
  canvas.value.on("object:added", () => saveState());
  canvas.value.on("object:removed", () => saveState());

  // Initialize with empty canvas state
  saveState();
}

function resizeCanvas() {
  if (!canvas.value) return;

  const container = document.querySelector(".canvas-container");
  if (!container) return;

  const containerWidth = container.clientWidth;
  const containerHeight = container.clientHeight;

  canvas.value.setDimensions({
    width: containerWidth,
    height: containerHeight,
  });

  canvas.value.renderAll();
}

function selectTool(tool: string) {
  activeTool.value = tool;
  if (!canvas.value) return;

  switch (tool) {
    case "brush":
      canvas.value.isDrawingMode = true;
      canvas.value.freeDrawingBrush.color = activeColor.value;
      canvas.value.freeDrawingBrush.width = brushSize.value;
      break;
    case "eraser":
      canvas.value.isDrawingMode = true;
      canvas.value.freeDrawingBrush.color = "#ffffff";
      canvas.value.freeDrawingBrush.width = brushSize.value * 2;
      break;
    default:
      canvas.value.isDrawingMode = false;
      break;
  }
}

function selectColor(color: string) {
  activeColor.value = color;

  // Add new color to the beginning of the array and remove the last color, if color already exists move color to the beginning
  const index = colors.value.findIndex((c) => c.color === color);
  if (index !== -1) {
    colors.value.splice(index, 1);
  }
  colors.value.unshift({ color });
  if (colors.value.length > 8) {
    colors.value.pop();
  }

  if (canvas.value && activeTool.value === "brush") {
    canvas.value.freeDrawingBrush.color = color;
  }
}

function selectSize(size: number) {
  brushSize.value = size;
  if (!canvas.value) return;

  if (activeTool.value === "brush") {
    canvas.value.freeDrawingBrush.width = size;
  } else if (activeTool.value === "eraser") {
    canvas.value.freeDrawingBrush.width = size * 2;
  }
}

function clearCanvas() {
  if (!canvas.value || !confirm("Are you sure you want to clear the canvas?"))
    return;

  canvas.value.clear();
  canvas.value.backgroundColor = "#fff";
  canvas.value.renderAll();
  saveState();
}

function saveState() {
  if (!canvas.value) return;

  if (undoStack.value.length >= 30) {
    undoStack.value.shift();
  }

  const data = canvas.value.toJSON(['selectable', 'evented'])
  const state = JSON.stringify(data);
  
  // Don't save if the state is the same as the last one
  if (undoStack.value.length > 0 && undoStack.value[undoStack.value.length - 1] === state) {
    return;
  }

  redoStack.value = [];
  undoStack.value.push(state);
  tackpad.setData(data);
  updateUndoRedoButtons();
}

function undo() {
  if (!canvas.value || undoStack.value.length <= 1) return;

  const currentState = JSON.stringify(canvas.value.toJSON(['selectable', 'evented']));
  redoStack.value.push(currentState);
  
  undoStack.value.pop(); // Remove current state
  canvas.value.loadFromJSON(
    undoStack.value[undoStack.value.length - 1],
    () => {
      canvas.value?.renderAll();
      updateUndoRedoButtons();
    }
  );
}

function redo() {
  if (!canvas.value || redoStack.value.length === 0) return;

  const state = redoStack.value.pop()!;
  undoStack.value.push(JSON.stringify(canvas.value.toJSON(['selectable', 'evented'])));
  
  canvas.value.loadFromJSON(state, () => {
    canvas.value?.renderAll();
    updateUndoRedoButtons();
  });
}

function updateUndoRedoButtons() {
  canUndo.value = undoStack.value.length > 1;
  canRedo.value = redoStack.value.length > 0;
}

function onMouseDown(o: fabric.IEvent) {
  if (!canvas.value || canvas.value.isDrawingMode) return;

  isDrawing.value = true;
  const pointer = canvas.value.getPointer(o.e);
  origX.value = pointer.x;
  origY.value = pointer.y;

  if (activeTool.value === "text") {
    addText(pointer);
    isDrawing.value = false;
    return;
  }
}

function onMouseMove(o: fabric.IEvent) {
  if (!isDrawing.value || !canvas.value) return;

  const pointer = canvas.value.getPointer(o.e);

  switch (activeTool.value) {
    case "line":
      drawLine(pointer);
      break;
    case "rect":
      drawRect(pointer);
      break;
    case "circle":
      drawCircle(pointer);
      break;
    case "triangle":
      drawTriangle(pointer);
      break;
  }
}

function onMouseUp() {
  isDrawing.value = false;
  if (!canvas.value || canvas.value.isDrawingMode) return;

  saveState();
}

function drawLine(pointer: { x: number; y: number }) {
  if (!canvas.value) return;

  if (activeShape.value) {
    canvas.value.remove(activeShape.value);
  }

  activeShape.value = new fabric.Line(
    [origX.value, origY.value, pointer.x, pointer.y],
    {
      stroke: activeColor.value,
      strokeWidth: brushSize.value,
      selectable: false,
    }
  );

  canvas.value.add(activeShape.value);
  canvas.value.renderAll();
}

function drawRect(pointer: { x: number; y: number }) {
  if (!canvas.value) return;

  if (activeShape.value) {
    canvas.value.remove(activeShape.value);
  }

  const width = pointer.x - origX.value;
  const height = pointer.y - origY.value;

  activeShape.value = new fabric.Rect({
    left: origX.value,
    top: origY.value,
    width: width,
    height: height,
    stroke: activeColor.value,
    strokeWidth: brushSize.value,
    fill: "transparent",
    selectable: false,
  });

  canvas.value.add(activeShape.value);
  canvas.value.renderAll();
}

function drawCircle(pointer: { x: number; y: number }) {
  if (!canvas.value) return;

  if (activeShape.value) {
    canvas.value.remove(activeShape.value);
  }

  const radius =
    Math.sqrt(
      Math.pow(pointer.x - origX.value, 2) +
        Math.pow(pointer.y - origY.value, 2)
    ) / 2;
  const centerX = (origX.value + pointer.x) / 2;
  const centerY = (origY.value + pointer.y) / 2;

  activeShape.value = new fabric.Circle({
    left: centerX - radius,
    top: centerY - radius,
    radius: radius,
    stroke: activeColor.value,
    strokeWidth: brushSize.value,
    fill: "transparent",
    selectable: false,
  });

  canvas.value.add(activeShape.value);
  canvas.value.renderAll();
}

function drawTriangle(pointer: { x: number; y: number }) {
  if (!canvas.value) return;

  if (activeShape.value) {
    canvas.value.remove(activeShape.value);
  }

  const width = pointer.x - origX.value;
  const height = pointer.y - origY.value;

  activeShape.value = new fabric.Triangle({
    left: origX.value,
    top: origY.value,
    width: width,
    height: height,
    stroke: activeColor.value,
    strokeWidth: brushSize.value,
    fill: "transparent",
    selectable: false,
  });

  canvas.value.add(activeShape.value);
  canvas.value.renderAll();
}

function addText(pointer: { x: number; y: number }) {
  if (!canvas.value) return;

  const text = new fabric.IText("Text", {
    left: pointer.x,
    top: pointer.y,
    fontFamily: "Arial",
    fontSize: brushSize.value * 5,
    fill: activeColor.value,
    editable: true,
  });

  canvas.value.add(text);
  canvas.value.setActiveObject(text);
  text.enterEditing();
  canvas.value.renderAll();

  text.on("editing:exited", function () {
    if (!canvas.value) return;

    if (text.text!.trim() === "") {
      canvas.value.remove(text);
    }
    saveState();
  });
}
</script>

<template>
  <div class="widget-container">

    <transition name="reduce-height">
    <div class="header" v-show="toolsVisibility">
      <div class="header-left">
        <div class="title">Drawing Board</div>
      </div>
      <div class="header-right">
        <div class="history-buttons">
          <button class="history-btn" :disabled="!canUndo" @click="undo">
            <Undo class="w-4 h-4" stroke-width="1.5" />
          </button>
          <button class="history-btn" :disabled="!canRedo" @click="redo">
            <Redo class="w-4 h-4" stroke-width="1.5" />
          </button>
        </div>
        <button class="btn" @click="clearCanvas">
          <Trash2 class="w-5 h-5" stroke-width="1.5" />
        </button>
      </div>
    </div>
  </transition>
    <div class="canvas-container" :style="{ paddingTop:toolsVisibility ? '0' : '12px' }" style="transition: padding 0.55s ease-in-out;">
      <canvas id="drawing-canvas"></canvas>
    </div>
  <transition name="reduce-height">
    <div class="toolbar" v-show="toolsVisibility">
      <div class="tool-group">
        <div class="tools">
          <div
            v-for="tool in tools"
            :key="tool.id"
            class="tool"
            :class="{ active: activeTool === tool.id }"
            @click="selectTool(tool.id)"
          >
            <component :is="tool.icon" class="w-5 h-5" stroke-width="1.5" />
          </div>
        </div>
      </div>
      <div class="toolbar-group">
        <div class="color-picker-wrapper">
          <div
            class="active-color"
            :style="{ backgroundColor: activeColor }"
            @click="showColorPicker = !showColorPicker"
          ></div>
          <div v-if="showColorPicker" class="color-picker-popup" @click.stop>
            <div class="color-grid">
              <div
                v-for="(color, index) in colors"
                :key="index"
                class="color-option"
                :style="{ backgroundColor: color.color }"
                @click="
                  selectColor(color.color);
                  showColorPicker = false;
                "
              ></div>
            </div>
            <input
              type="color"
              :value="activeColor"
              @blur="e => selectColor((e.target as HTMLInputElement).value)"
              class="custom-color-input"
            />
          </div>
        </div>
        <div class="brush-size-controls">
          <input
            type="range"
            :value="brushSize"
            min="1"
            max="50"
            @input="e => selectSize(Number((e.target as HTMLInputElement).value))"
            class="brush-size-slider"
          />
          <input
            type="number"
            :value="brushSize"
            min="1"
            max="50"
            @input="e => selectSize(Number((e.target as HTMLInputElement).value))"
            class="brush-size-input"
          />
        </div>
      </div>
    </div>
  </transition>
  </div>
</template>

<style scoped>
/* Base styles */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
    Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
}

.widget-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  max-width: 100%;
  margin: 0 auto;
  background-color: white;
  border-radius: 12px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.08);
  overflow: hidden;
  justify-content: space-between;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px;
  border-bottom: 1px solid #f0f0f0;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.logo {
  width: 32px;
  height: 32px;
  border-radius: 8px;
  background-color: #000;
  display: flex;
  align-items: center;
  justify-content: center;
}

.logo svg {
  width: 20px;
  height: 20px;
  fill: white;
}

.title {
  font-weight: 600;
  font-size: 16px;
  color: #111;
}

.header-right {
  display: flex;
  gap: 12px;
}

.btn {
  background: none;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  border-radius: 8px;
  transition: background 0.2s;
}

.btn:hover {
  background-color: #f5f5f7;
}

.btn.active {
  background-color: #f0f0f0;
}

.btn svg {
  width: 20px;
  height: 20px;
}

.canvas-container {
  flex: 1;
  position: relative;
  overflow: hidden;
  flex-grow: 1;
}

canvas {
  display: block;
}

.toolbar {
  display: flex;
  justify-content: space-between;
  padding: 12px 16px;
  border-top: 1px solid #f0f0f0;
  flex-wrap: wrap;
  gap: 1rem;
}

.tool-group {
  display: flex;
  gap: 8px;
}

.tools {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  justify-content: center;
}

.tool {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  background: transparent;
}

.tool:hover {
  background: #f3f4f6;
}

.tool svg {
  width: 20px;
  height: 20px;
  stroke: #374151;
  stroke-width: 1.25;
  fill: none;
}

.tool.active {
  border: 2px solid blueviolet;
  padding: 4px;
  border-color: #d1d5db;
}

.tool.active svg {
  stroke: #111827;
}

.header-left .logo svg {
  stroke: #374151;
  stroke-width: 1.25;
  fill: none;
}

.header-right .btn svg {
  stroke: #374151;
  stroke-width: 1.25;
  fill: none;
}

.undo-redo .tool svg {
  stroke: #374151;
  stroke-width: 1.25;
  fill: none;
}

.undo-redo .tool:disabled svg {
  stroke: #9ca3af;
}

.toolbar-group {
  display: flex;
  gap: 16px;
  align-items: center;
}

.color-picker-wrapper {
  position: relative;
}

.active-color {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  border: 1px solid #e0e0e0;
  cursor: pointer;
  transition: transform 0.2s;
}

.active-color:hover {
  transform: scale(1.05);
}

.color-picker-popup {
  position: absolute;
  bottom: 100%;
  left: 0;
  margin-bottom: 8px;
  padding: 12px;
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  z-index: 1000;
}

.color-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
  margin-bottom: 8px;
}

.color-option {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  border: 1px solid #e0e0e0;
  cursor: pointer;
  transition: transform 0.2s;
}

.color-option:hover {
  transform: scale(1.1);
}

.custom-color-input {
  width: 100%;
  height: 24px;
  padding: 0;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  cursor: pointer;
}

.brush-size-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}

.brush-size-slider {
  width: 120px;
  height: 36px;
  padding: 0 8px;
}

.brush-size-input {
  width: 60px;
  height: 36px;
  padding: 0 8px;
  text-align: center;
  border-radius: 6px;
  border: 1px solid #e0e0e0;
  font-size: 14px;
}

.brush-size-input:hover,
.brush-size-input:focus {
  border-color: #999;
}

/* Custom range slider styles */
.brush-size-slider {
  -webkit-appearance: none;
  background: transparent;
}

.brush-size-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  height: 16px;
  width: 16px;
  border-radius: 50%;
  background: #1890ff;
  cursor: pointer;
  margin-top: -6px;
}

.brush-size-slider::-webkit-slider-runnable-track {
  width: 100%;
  height: 4px;
  background: #e0e0e0;
  border-radius: 2px;
}

.brush-size-slider:focus {
  outline: none;
}

.history-buttons {
  display: flex;
  gap: 8px;
}

.history-btn {
  background-color: #f5f5f7;
  border: none;
  border-radius: 8px;
  padding: 8px 12px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}

.history-btn:hover {
  background-color: #e5e5e7;
}

.history-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.history-btn svg {
  width: 16px;
  height: 16px;
  fill: none;
  stroke-width: 1.5;
}

/* Media queries for responsiveness */
/* @media (max-width: 600px) {
  .header {
    padding: 12px;
  }

  .logo {
    width: 28px;
    height: 28px;
  }

  .title {
    font-size: 14px;
  }

  .btn {
    width: 32px;
    height: 32px;
  }

  .toolbar,
  .bottom-toolbar {
    padding: 8px 12px;
  }

  .tool {
    width: 36px;
    height: 36px;
  }

  .tool svg {
    width: 18px;
    height: 18px;
  }

  .brush-size-slider {
    width: 80px;
  }
}

@media (max-width: 400px) {
  .tools {
    gap: 4px;
  }

  .tool {
    width: 32px;
    height: 32px;
  }

  .toolbar-group {
    gap: 8px;
  }

  .brush-size-slider {
    width: 60px;
  }
}

@media (max-height: 500px) {
  .header {
    padding: 8px 12px;
  }

  .logo {
    width: 24px;
    height: 24px;
  }
} */

.reduce-height-enter-active,
.reduce-height-leave-active {
  transition: all 0.5s ease;
  overflow: hidden;
}

.reduce-height-enter-from,
.reduce-height-leave-to {
  opacity: 0;
  height: 0;
  margin: 0;
  padding: 0;
}

.reduce-height-enter-to,
.reduce-height-leave-from {
  opacity: 1;
  height: auto;
}
</style>
