<template>
  <editor-content class="editor" :editor="editor" />
</template>

<script setup>
import { onBeforeUnmount } from 'vue'
import { EditorContent, useEditor } from '@tiptap/vue-3'
import StarterKit from '@tiptap/starter-kit'
import { Node, mergeAttributes } from '@tiptap/core'

// 自定义可编辑的字段节点
const InputField = Node.create({
  name: 'inputField',
  group: 'inline',
  inline: true,
  atom: false, // 允许内部编辑
  selectable: true,
  draggable: false,

  addAttributes() {
    return {
      key: { default: '' },
      placeholder: { default: '' },
    }
  },

  parseHTML() {
    return [{ tag: 'span[data-type="input-field"]' }]
  },

  renderHTML({ HTMLAttributes }) {
    return [
      'span',
      mergeAttributes(HTMLAttributes, {
        'data-type': 'input-field',
        'data-placeholder': HTMLAttributes.placeholder,
        class: 'input-field',
        contenteditable: 'true',
      }),
      0,
    ]
  },

  addCommands() {
    return {
      setInputField:
        (attrs) =>
        ({ commands }) => {
          return commands.insertContent({
            type: this.name,
            attrs,
          })
        },
    }
  },
})

// 模板部分：文本和字段占位符交替组成
const templateParts = [
  '你好，我是 ',
  { key: 'name', placeholder: '姓名' },
  '，今年 ',
  { key: 'age', placeholder: '年龄' },
  ' 岁了，在 ',
  { key: 'city', placeholder: '城市' },
  ' 工作。',
]

// 组合成 HTML 字符串，字段用自定义 span 表示
function buildInitialContent(parts) {
  let html = ''
  for (const part of parts) {
    if (typeof part === 'string') {
      html += part
    } else {
      html += `<span data-type="input-field" data-placeholder="${part.placeholder}" contenteditable="true"></span>`
    }
  }
  return html
}

// 创建编辑器实例
const editor = useEditor({
  extensions: [StarterKit, InputField],
  content: buildInitialContent(templateParts),
})

// 处理 Backspace 删除逻辑
function handleKeydown(event) {
  if (event.key !== 'Backspace') return
  if (!editor) return

  const { state } = editor
  const { selection } = state
  const { empty, $from } = selection

  if (!empty) return

  const nodeBefore = $from.nodeBefore

  // 光标前是 inputField 节点时，删除整个节点
  if (nodeBefore && nodeBefore.type.name === 'inputField') {
    event.preventDefault()
    const tr = state.tr.delete($from.pos - nodeBefore.nodeSize, $from.pos)
    editor.view.dispatch(tr)
    return
  }

  // 光标在 inputField 节点且内容为空时阻止删除
  if ($from.parent.type.name === 'inputField' && $from.parent.textContent.length === 0) {
    event.preventDefault()
  }
}

// 监听键盘事件
editor?.on('keydown', handleKeydown)

onBeforeUnmount(() => {
  editor?.destroy()
})
</script>

<style scoped>
.editor {
  border: 1px solid #ccc;
  min-height: 150px;
  padding: 10px;
  font-size: 16px;
  line-height: 1.5;
  white-space: pre-wrap;
}

.input-field {
  display: inline-block;
  min-width: 60px;
  padding: 0 4px;
  border-bottom: 1px dashed #888;
  background: #f3f3fc;
  outline: none;
  white-space: nowrap;
  cursor: text;
}

.input-field:empty::before {
  content: attr(data-placeholder);
  color: #aaa;
  pointer-events: none;
  user-select: none;
}
</style>