<template>
  <div 
    class="sentence" 
    ref="editableRef"
    contenteditable="true"
    @keydown.capture="onOuterKeydown"
  >
    <template v-for="(part, index) in parts" :key="index">
      <template v-if="typeof part === 'object'">
        <span 
          class="field-wrapper" 
          :data-key="part.key" 
          contenteditable="false"
        >
          <!-- 时间选择 -->
          <input
            v-if="fieldConfig[part.key]?.type === 'time'"
            contenteditable="false"
            type="time"
            :data-placeholder="placeholders[part.key]"
            v-model="model[part.key]"
            @change="onFieldInput(part.key, $event)"
          />
          <!-- 日期选择 -->
          <input
            v-else-if="fieldConfig[part.key]?.type === 'date'"
            contenteditable="false"
            type="date"
            :data-placeholder="placeholders[part.key]"
            v-model="model[part.key]"
            @change="onFieldInput(part.key, $event)"
          />
          <!-- 下拉框 -->
          <select
            v-else-if="fieldConfig[part.key]?.type === 'select'"
            contenteditable="false"
            :data-placeholder="placeholders[part.key]"
            v-model="model[part.key]"
            @change="onFieldInput(part.key, $event)"
          >
            <option 
              v-for="opt in fieldConfig[part.key].options" 
              :key="opt" 
              :value="opt"
            >{{ opt }}</option>
          </select>
          <!-- 默认可编辑文本 -->
          <span
            v-else
            class="field"
            :data-placeholder="placeholders[part.key]"
            contenteditable="true"
            spellcheck="false"
            @input="onFieldInput(part.key, $event)"
            @keydown="onFieldKeydown(part.key, $event)"
          >{{ model[part.key] }}</span>
        </span>
      </template>
      <template v-else>{{ part }}</template>
    </template>
  </div>
  
  <!-- 新增：输出文本框与确定按钮 -->
  <div class="controls">
    <input type="text" v-model="outputText" readonly />
    <button @click="confirmText">确定</button>
  </div>
</template>

<script setup>
import { ref, reactive, nextTick } from "vue";

const template = "帮我写一个自我介绍，字数 [wordCount]。现在是[appointment]。你好，我是 [name]，一直到[date]为止已经[age] 岁了，在 [city] 工作";

function parseTemplate(tpl) {
  const regex = /\[([^\]]+)\]/g;
  let result = [];
  let lastIndex = 0;
  let match;
  while ((match = regex.exec(tpl)) !== null) {
    if (match.index > lastIndex) {
      result.push(tpl.slice(lastIndex, match.index));
    }
    result.push({ key: match[1] });
    lastIndex = regex.lastIndex;
  }
  if (lastIndex < tpl.length) result.push(tpl.slice(lastIndex));
  return result;
}

const parts = parseTemplate(template);

const model = reactive({
  name: "",
  age: "",
  city: "",
  appointment: "",
  wordCount: "500",
  date: ""
});

// 新增：字段类型配置，支持 text、time、select（下拉）
const fieldConfig = {
  name:        { type: 'text' },
  age:         { type: 'text' },
  city:        { type: 'select', options: ['北京', '上海', '广州'] },
  appointment: { type: 'time' },  // 示例字段：时间选择
  wordCount:   { type: 'text' },
  date:        { type: 'date' }
};

const placeholders = {
  name: "[姓名]",
  age: "[年龄]",
  city: "[城市]",
  appointment:"[时间]",
  wordCount: "[字数]",
  date: "[日期]"
};

// 新增：撤销栈
const undoStack = [];

// 新增：跟踪待删除的字段
let pendingDeleteWrapper = null;

// 新增：根据 key 创建字段包装器
function createFieldWrapper(key) {
  const wrap = document.createElement("span");
  wrap.className = "field-wrapper";
  wrap.dataset.key = key;
  wrap.contentEditable = "false";

  const cfg = fieldConfig[key] || { type: 'text' };
  let fld;

  if (cfg.type === 'time') {
    // 时间选择
    fld = document.createElement("input");
    fld.type = "time";
    fld.value = model[key] || "";
    fld.dataset.placeholder = placeholders[key];
    fld.addEventListener("change", e => onFieldInput(key, e));
  }
  else if (cfg.type === 'date') {
    // 日期选择
    fld = document.createElement("input");
    fld.type = "date";
    fld.value = model[key] || "";
    fld.dataset.placeholder = placeholders[key];
    fld.addEventListener("change", e => onFieldInput(key, e));
  }
  else if (cfg.type === 'select') {
    // 下拉框
    fld = document.createElement("select");
    cfg.options.forEach(opt => {
      const o = document.createElement("option");
      o.value = o.textContent = opt;
      fld.appendChild(o);
    });
    fld.value = model[key] || "";
    fld.dataset.placeholder = placeholders[key];
    fld.addEventListener("change", e => onFieldInput(key, e));
  }
  else {
    // 默认可编辑文本
    fld = document.createElement("span");
    fld.className = "field";
    fld.dataset.placeholder = placeholders[key];
    fld.contentEditable = "true";
    fld.spellcheck = false;
    fld.textContent = model[key] || "";
    fld.addEventListener("input", e => onFieldInput(key, e));
    fld.addEventListener("keydown", e => onFieldKeydown(key, e));
  }

  wrap.appendChild(fld);
  return wrap;
}

const editableRef = ref();

// 修改：Backspace 连续两次删除字段（首次标记，二次删除）
function onOuterKeydown(e) {
  // Ctrl+Z → 撤销
  if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === 'z') {
    e.preventDefault();
    undoLastDeletion();
    return;
  }

  if (e.key === 'Backspace') {
    const sel = window.getSelection();
    if (!sel.rangeCount) return;
    const range = sel.getRangeAt(0);
    const { startContainer, startOffset } = range;

    // 普通文本内部且偏移 > 0：正常删除
    if (startContainer.nodeType === Node.TEXT_NODE && startOffset > 0) {
      pendingDeleteWrapper = null;
      return;
    }

    // 光标前无文本，则检查前一个节点是否字段包裹
    const prev = startContainer.previousSibling;
    if (prev?.classList.contains('field-wrapper')) {
      e.preventDefault();
      // 连续两次才删除字段
      if (pendingDeleteWrapper === prev) {
        undoStack.push({ wrapper: prev, nextSibling: prev.nextSibling });
        prev.remove();
        pendingDeleteWrapper = null;
      } else {
        pendingDeleteWrapper = prev;
      }
      return;
    }

    pendingDeleteWrapper = null;
    return;
  }

  // Delete 逻辑
  if (e.key === 'Delete') {
    const sel = window.getSelection();
    if (!sel.rangeCount) return;
    const range = sel.getRangeAt(0);
    const next = getNextNode(range.startContainer, range.startOffset);
    if (next?.classList.contains('field-wrapper')) {
      e.preventDefault();
      if (pendingDeleteWrapper === next) {
        undoStack.push({ wrapper: next, nextSibling: next.nextSibling });
        next.remove();
        pendingDeleteWrapper = null;
      } else {
        pendingDeleteWrapper = next;
        moveCursorAfterWrapper(next);
      }
    } else {
      pendingDeleteWrapper = null;
    }
    return;
  }
}

// 新增：删除字段后，将光标定位到指定节点前
function setCursorBefore(node) {
  const container = editableRef.value;
  const range = document.createRange();
  const sel = window.getSelection();
  if (node) {
    // 如果是文本节点，则在开头位置
    if (node.nodeType === Node.TEXT_NODE) {
      range.setStart(node, 0);
    } else {
      range.setStartBefore(node);
    }
  } else {
    // 到 container 末尾
    range.selectNodeContents(container);
    range.collapse(false);
  }
  sel.removeAllRanges();
  sel.addRange(range);
}

// 修改：撤销时直接复用入栈的 wrapper
function undoLastDeletion() {
  const action = undoStack.pop();
  if (!action) return;
  const container = editableRef.value;
  container.insertBefore(action.wrapper, action.nextSibling);
}

// 重建整个内容
function regenerateContent() {
  const container = editableRef.value;
  // 清空并重建
  container.innerHTML = "";
  parts.forEach(part => {
    if (typeof part === "object") {
      const key = part.key;
      const wrap = createFieldWrapper(key);
      container.appendChild(wrap);
    } else {
      container.appendChild(document.createTextNode(part));
    }
  });
}

// 获取光标前一个节点
function getPreviousNode(container, offset) {
  if (offset > 0 && container.nodeType === Node.TEXT_NODE) {
    return container.parentNode.childNodes[offset - 1];
  }
  return container.previousSibling;
}

// 新增：获取光标后的节点
function getNextNode(container, offset) {
  if (container.nodeType === Node.TEXT_NODE) {
    if (offset < container.textContent.length) return null;
    return container.parentNode.childNodes[offset];
  }
  return container.nextSibling;
}

// 修改：字段内部也捕获 Ctrl+Z，转至单次撤销；仅在 Backspace 时阻止删除并移动光标出字段
function onFieldKeydown(key, e) {
  // 拦截 Ctrl+Z
  if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === "z") {
    e.preventDefault();
    undoLastDeletion();
    return;
  }
  // 阻止字段内部删除自身
  if (e.key === "Backspace") {
    e.stopPropagation();
    const sel = window.getSelection();
    if (sel.anchorOffset === 0) {
      e.preventDefault();
      // 光标跳出字段外，移动到字段前
      moveCursorBeforeWrapper(e.target.closest(".field-wrapper"));
    }
  }
}

// 新增：将光标定位到字段包装器之前
function moveCursorBeforeWrapper(wrapper) {
  const container = editableRef.value;
  const range = document.createRange();
  const sel = window.getSelection();
  const prev = wrapper.previousSibling;
  if (prev && prev.nodeType === Node.TEXT_NODE) {
    range.setStart(prev, prev.textContent.length);
  } else {
    range.setStartBefore(wrapper);
  }
  range.collapse(true);
  sel.removeAllRanges();
  sel.addRange(range);
}

// 新增：将光标定位到字段包装器之后
function moveCursorAfterWrapper(wrapper) {
  const container = editableRef.value;
  const range = document.createRange();
  const sel = window.getSelection();
  const next = wrapper.nextSibling;
  if (next && next.nodeType === Node.TEXT_NODE) {
    range.setStart(next, 0);
  } else if (next) {
    range.setStartBefore(next);
  } else {
    range.selectNodeContents(container);
    range.collapse(false);
  }
  sel.removeAllRanges();
  sel.addRange(range);
}

// 修改：适配 input/select 与 span
function onFieldInput(key, e) {
  const el = e.target;
  model[key] = el.value !== undefined ? el.value : el.innerText;
}

// 设置光标到元素尾部
function setCursorToEnd(el) {
  const range = document.createRange();
  range.selectNodeContents(el);
  range.collapse(false);
  const sel = window.getSelection();
  sel.removeAllRanges();
  sel.addRange(range);
}

const outputText = ref("");

// 修改：遍历 DOM 节点，空字段用 placeholder，已删除字段不再输出
function confirmText() {
  const container = editableRef.value;
  let text = "";
  container.childNodes.forEach(node => {
    if (node.nodeType === Node.TEXT_NODE) {
      text += node.textContent;
    }
    else if (
      node.nodeType === Node.ELEMENT_NODE &&
      node.classList.contains("field-wrapper")
    ) {
      const el = node.firstElementChild;
      let val = el.value !== undefined ? el.value : el.innerText;
      if (!val) {
        val = el.dataset.placeholder || "";
      }
      text += val;
    }
  });
  outputText.value = text;
}
</script>

<style scoped>
.sentence {
  border: 1px solid #ccc;
  padding: 12px;
  white-space: pre-wrap;
  line-height: 1.8;
  min-height: 2em;
}

.field-wrapper {
  border: none;
  border-radius: 10px;
  color: #06f;
  color: var(--s-color-brand-primary-default, #06f);
  font-weight: 600;
  line-height: 150%;
  padding: 2px 6px;
  word-break: break-word;
}

/* 时间选择与下拉也应用同样的视觉风格 */
.field-wrapper input,
.field-wrapper select {
  background: rgba(0, 102, 255, .06);
  background: var(--s-color-brand-primary-transparent-1, rgba(0, 102, 255, .06));
  border: none;
  border-radius: 10px;
  color: #06f;
  color: var(--s-color-brand-primary-default, #06f);
  font-weight: 600;
  line-height: 150%;
  padding: 2px 6px;
  word-break: break-word;
}

.field {
  background: var(--s-color-brand-primary-transparent-1, rgba(0, 102, 255, .06));
  border: none;
  border-radius: 10px;
  color: #06f;
  color: var(#0057ff, #06f);
  font-weight: 600;
  line-height: 150%;
  padding: 2px 6px;
  word-break: break-word;
}

.field:empty::before {
  content: attr(data-placeholder);
  color: rgb( 152, 205, 253);
}
.field:hover{
  cursor : text 
}
.field:focus {
  cursor : text 
}

.controls {
  margin-top: 12px;
}
.controls input {
  width: 100%;
  padding: 6px;
  box-sizing: border-box;
}
.controls button {
  margin-top: 6px;
  padding: 6px 12px;
}
</style>
