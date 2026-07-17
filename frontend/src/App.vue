<script setup>
import { ref, reactive, onMounted, watch } from 'vue'

const models = ref([])
const selectedModel = ref('')
const userInput = ref('')
const messages = ref([])
const isStreaming = ref(false)

const theme = ref(localStorage.getItem('theme') || 'dark')

function applyTheme() {
  document.documentElement.setAttribute('data-theme', theme.value)
  localStorage.setItem('theme', theme.value)
}

function toggleTheme() {
  theme.value = theme.value === 'dark' ? 'light' : 'dark'
}

watch(theme, applyTheme)

const API_BASE = 'http://localhost:8000'

onMounted(async () => {
  applyTheme() // apply saved/default theme as soon as the app loads

  const res = await fetch(`${API_BASE}/api/models`)
  models.value = await res.json()
  if (models.value.length > 0) {
    selectedModel.value = models.value[0]
  }
})

async function sendMessage() {
  const text = userInput.value.trim()
  if (!text || isStreaming.value) return

  messages.value.push({ role: 'user', content: text })
  userInput.value = ''

  const assistantMessage = reactive({ role: 'assistant', content: '' })
  messages.value.push(assistantMessage)

  isStreaming.value = true

  try {
    const response = await fetch(`${API_BASE}/api/chat`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: selectedModel.value,
        messages: messages.value
          .slice(0, -1)
          .map(m => ({ role: m.role, content: m.content })),
      }),
    })

    const reader = response.body.getReader()
    const decoder = new TextDecoder()

    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      assistantMessage.content += decoder.decode(value, { stream: true })
    }
  } catch (err) {
    assistantMessage.content = 'Error: could not reach the backend.'
    console.error(err)
  } finally {
    isStreaming.value = false
  }
}

// Handle Enter key for sending messages and Shift+Enter for new lines
function handleComposerKey(event) {
  if (event.key === 'Enter' && !event.shiftKey) {
    event.preventDefault()
    sendMessage()
  }
}


</script>

<template>
  <div class="app">
    <header class="app-header">
      <h1>Ollama Chat</h1>
      <button class="theme-toggle" @click="toggleTheme">
        {{ theme === 'dark' ? '☀️ Light' : '🌙 Dark' }}
      </button>
    </header>

    <select class="model-select" v-model="selectedModel">
      <option v-for="m in models" :key="m" :value="m">{{ m }}</option>
    </select>

    <div class="chat-window">
      <div
        v-for="(msg, i) in messages"
        :key="i"
        class="message"
        :class="msg.role === 'user' ? 'message--user' : 'message--assistant'">
        <span class="message-role">{{ msg.role === 'user' ? 'You' : selectedModel }}</span>
        <p class="message-content">{{ msg.content }}</p>
      </div>
    </div>

    <form class="chat-form" @submit.prevent="sendMessage">
      <textarea
        v-model="userInput"
        type="text"
        placeholder="Ask something..."
        @keydown="handleComposerKey"
        @input="autoResizeComposer"
        class="chat-input"
        :disabled="isStreaming"></textarea>
      <button type="submit" class="send-button" :disabled="isStreaming">
        {{ isStreaming ? '...' : 'Send' }}
      </button>
    </form>
  </div>
</template>

<style scoped>
.app {
  max-width: 1100px;
  margin: 40px auto;
  padding: 0 16px;
}

.app-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.theme-toggle {
  background: var(--color-surface-alt);
  color: var(--color-text);
  border: 1px solid var(--color-border);
  border-radius: 20px;
  padding: 6px 14px;
  cursor: pointer;
  font-size: 0.9rem;
}

.model-select {
  background: var(--color-surface);
  color: var(--color-text);
  border: 1px solid var(--color-border);
  border-radius: 6px;
  padding: 6px 10px;
  margin-bottom: 16px;
}

.chat-window {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: 12px;
  padding: 16px;
  min-height: 600px;
  max-height: 700px;
  overflow-y: auto;
}

.message {
  margin-bottom: 14px;
}

.message-role {
  font-size: 0.8rem;
  font-weight: 600;
  color: var(--color-accent);
}

.message--user .message-role {
  color: var(--color-text-secondary);
}

.message-content {
  white-space: pre-wrap;
  margin: 4px 0 0;
  line-height: 1.5;
}

.chat-form {
  display: flex;
  gap: 8px;
  margin-top: 16px;
}

.chat-input {
  flex: 1;
  background: var(--color-surface);
  color: var(--color-text);
  border: 1px solid var(--color-border);
  border-radius: 8px;
  padding: 10px 12px;
  font-size: 1rem;
  min-height: 50px;
  max-height: 80px;
  overflow-y: auto;
}

.chat-input:focus {
  outline: none;
  border-color: var(--color-accent);
}

.send-button {
  background: var(--color-accent);
  color: var(--color-accent-text);
  border: none;
  border-radius: 8px;
  padding: 10px 18px;
  font-weight: 600;
  cursor: pointer;
}

.send-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
</style>