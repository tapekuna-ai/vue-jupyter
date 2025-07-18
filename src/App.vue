<!--
# Copyright (c) 2025 Thomas Liebig, Tapekuna UG
# Licensed under the MIT License. See LICENSE file for details.
-->
<template>
  <div class="p-4">
    <h1>Jupyter Frontend in Vue.js</h1>

    <!-- Connection Panel -->
    <div v-if="!connected" class="mb-6 border p-4 bg-gray-50">
      <h2>Connect to Jupyter Server</h2>
      <input
        v-model="serverInput"
        placeholder="Enter Jupyter URL or token"
        class="border p-2 w-full mb-2"
      />
      <button @click="connectToServer">Connect</button>

      <div v-if="savedConnection" class="mt-4">
        <p>Last connection:</p>
        <button @click="useSavedConnection">
          {{ savedConnection.baseUrl }} (token: {{ savedConnection.token.slice(0, 6) }}…)
        </button>
        <button class="ml-2 text-sm text-red-600" @click="clearSavedConnection">
          Forget this connection
        </button>
      </div>

      <p v-if="error" class="text-red-600 mt-2">{{ error }}</p>
    </div>

    <!-- Notebook Loader -->
    <div v-else>
      <button @click="listNotebooks">List Notebooks</button>
      <ul>
        <li v-for="file in notebooks" :key="file.path">
          <button @click="loadNotebook(file.path)">
            {{ file.name }}
          </button>
        </li>
      </ul>

      <!-- Notebook Editor -->
      <div v-if="cells.length" class="mt-4">
        <h2>{{ currentNotebook }}</h2>
        <button @click="addCell('code')">+ Add Code Cell</button>
        <button @click="addCell('markdown')">+ Add Markdown Cell</button>
        <button @click="saveNotebook">💾 Save Notebook</button>

        <div v-for="(cell, idx) in cells" :key="idx" class="mb-4 border p-2 bg-gray-50">
          <div class="flex items-center space-x-2">
            <select v-model="cell.type">
              <option value="code">Code</option>
              <option value="markdown">Markdown</option>
            </select>
            <button @click="moveCell(idx, -1)" :disabled="idx === 0">⬆️</button>
            <button @click="moveCell(idx, 1)" :disabled="idx === cells.length - 1">⬇️</button>
            <button @click="deleteCell(idx)">🗑 Delete</button>
          </div>

          <!-- Editable source -->
          <textarea v-if="!cell.rendered" v-model="cell.source" rows="4" cols="60"></textarea>

          <!-- Rendered Markdown (with math) -->
          <div
            v-else-if="cell.type === 'markdown'"
            v-html="renderMarkdown(cell.source)"
            class="p-2 bg-white markdown"
          ></div>

          <!-- Run button for code cells -->
          <div v-if="cell.type === 'code'" class="mt-2">
            <button @click="runCell(idx)">▶ Run</button>
          </div>

          <!-- Outputs -->
          <div v-if="cell.outputs.length" class="mt-2 p-2 bg-white border">
            <div v-for="(out, oidx) in cell.outputs" :key="oidx">
              <div v-if="out.type === 'text'">
                <pre>{{ out.data }}</pre>
              </div>
              <div v-else-if="out.type === 'html'" v-html="out.data"></div>
              <div v-else-if="out.type === 'image'">
                <img :src="`data:image/png;base64,${out.data}`" />
              </div>
              <div v-else-if="out.type === 'svg'">
                <div v-html="atob(out.data)"></div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue'
import { marked } from 'marked'
import katex from 'katex'
import * as services from '@jupyterlab/services'
import 'katex/dist/katex.min.css'

// Connection state
let serverSettings = null
let contentsManager = null
let session = null

const connected = ref(false)
const serverInput = ref('')
const error = ref('')
const savedConnection = ref(null)
const notebooks = ref([])
const cells = ref([])
const currentNotebook = ref('')

// Restore last connection
onMounted(() => {
  const saved = localStorage.getItem('jupyter-connection')
  if (saved) {
    try {
      savedConnection.value = JSON.parse(saved)
    } catch {
      savedConnection.value = null
    }
  }
})

// Connect with new input
async function connectToServer() {
  try {
    let url = serverInput.value.trim()
    let baseUrl = 'http://localhost:8888'
    let token = ''

    if (url.startsWith('http')) {
      const u = new URL(url)
      baseUrl = `${u.protocol}//${u.host}`
      token = u.searchParams.get('token') || ''
    } else {
      token = url
    }

    serverSettings = services.ServerConnection.makeSettings({
      baseUrl,
      token,
      wsUrl: baseUrl.replace('http', 'ws'),
    })
    contentsManager = new services.ContentsManager({ serverSettings })

    await contentsManager.get('', { content: false })

    localStorage.setItem('jupyter-connection', JSON.stringify({ baseUrl, token }))
    savedConnection.value = { baseUrl, token }

    connected.value = true
    error.value = ''
  } catch (err) {
    error.value = 'Failed to connect to Jupyter server.'
    console.error(err)
  }
}

// Use saved connection
async function useSavedConnection() {
  if (!savedConnection.value) return
  try {
    serverSettings = services.ServerConnection.makeSettings({
      baseUrl: savedConnection.value.baseUrl,
      token: savedConnection.value.token,
      wsUrl: savedConnection.value.baseUrl.replace('http', 'ws'),
    })
    contentsManager = new services.ContentsManager({ serverSettings })

    await contentsManager.get('', { content: false })
    connected.value = true
    error.value = ''
  } catch (err) {
    error.value = 'Failed to reconnect.'
    console.error(err)
  }
}

// Forget saved connection
function clearSavedConnection() {
  localStorage.removeItem('jupyter-connection')
  savedConnection.value = null
}

// Render Markdown + math
function renderMarkdown(src) {
  const rawHtml = marked(src || '')
  // Render inline math ($...$) and block math ($$...$$)
  return rawHtml
    .replace(/\$\$(.+?)\$\$/g, (_, eq) => katex.renderToString(eq, { displayMode: true }))
    .replace(/\$(.+?)\$/g, (_, eq) => katex.renderToString(eq, { displayMode: false }))
}

// Notebook loading
async function listNotebooks() {
  const root = await contentsManager.get('', { content: true })
  notebooks.value = root.content.filter((f) => f.type === 'notebook')
}

async function loadNotebook(path) {
  const nb = await contentsManager.get(path, { content: true })
  currentNotebook.value = path

  cells.value = nb.content.cells.map((c) => ({
    type: c.cell_type,
    source: Array.isArray(c.source) ? c.source.join('') : c.source,
    outputs: [],
    rendered: c.cell_type === 'markdown',
  }))

  const kernelManager = new services.KernelManager({ serverSettings })
  const sessionManager = new services.SessionManager({ serverSettings, kernelManager })
  await sessionManager.ready

  let existing = null
  for await (const running of sessionManager.running()) {
    if (running.path === path) {
      existing = running
      break
    }
  }

  if (existing) {
    session = await sessionManager.connectTo({ model: existing })
  } else {
    session = await sessionManager.startNew({
      path,
      name: path,
      type: 'notebook',
      kernel: { name: 'python3' },
    })
  }
}

// Cell editing
function addCell(type = 'code') {
  cells.value.push({ type, source: '', outputs: [], rendered: type === 'markdown' })
}
function deleteCell(idx) {
  cells.value.splice(idx, 1)
}
function moveCell(idx, dir) {
  const newIdx = idx + dir
  if (newIdx < 0 || newIdx >= cells.value.length) return
  const [cell] = cells.value.splice(idx, 1)
  cells.value.splice(newIdx, 0, cell)
}

// Run a code cell (capture PNG, SVG, HTML, text)
async function runCell(idx) {
  if (!session || cells.value[idx].type !== 'code') return
  cells.value[idx].outputs = []

  const future = session.kernel.requestExecute({ code: cells.value[idx].source })
  future.onIOPub = (msg) => {
    const { msg_type } = msg.header
    const content = msg.content

    if (msg_type === 'stream') {
      cells.value[idx].outputs.push({ type: 'text', data: content.text })
    } else if (msg_type === 'display_data' || msg_type === 'execute_result') {
      const data = content.data
      if (data['image/svg+xml']) {
        // Encode SVG as Base64 for embedding
        const svgBase64 = btoa(data['image/svg+xml'])
        cells.value[idx].outputs.push({ type: 'svg', data: svgBase64 })
      } else if (data['image/png']) {
        cells.value[idx].outputs.push({ type: 'image', data: data['image/png'] })
      } else if (data['text/html']) {
        cells.value[idx].outputs.push({ type: 'html', data: data['text/html'] })
      } else if (data['text/plain']) {
        cells.value[idx].outputs.push({ type: 'text', data: data['text/plain'] })
      }
    } else if (msg_type === 'error') {
      cells.value[idx].outputs.push({
        type: 'text',
        data: `Error: ${content.ename}: ${content.evalue}\n${content.traceback.join('\n')}`,
      })
    }
  }
}

// Save notebook
async function saveNotebook() {
  if (!currentNotebook.value) return
  const nb = {
    cells: cells.value.map((c) => ({
      cell_type: c.type,
      metadata: {},
      source: [c.source],
      outputs: c.type === 'code' ? [] : undefined,
      execution_count: c.type === 'code' ? null : undefined,
    })),
    metadata: { language_info: { name: 'python' } },
    nbformat: 4,
    nbformat_minor: 5,
  }
  await contentsManager.save(currentNotebook.value, {
    type: 'notebook',
    format: 'json',
    content: nb,
  })
  alert('Notebook saved!')
}
</script>

<style>
body {
  font-family: sans-serif;
}
textarea {
  width: 100%;
  margin-top: 8px;
}
pre {
  background: #f0f0f0;
  padding: 5px;
  white-space: pre-wrap;
}
button {
  margin-right: 5px;
}
.markdown p {
  margin: 0.5em 0;
}
</style>
