<script lang="ts">
  type Role = 'user' | 'assistant' | 'system'
  type ChatMessage = { id: number; role: Role; content: string; thinking?: string; streaming?: boolean }
  type Status = 'idle' | 'checking' | 'connected' | 'error'
  type LoadedModel = { name: string; size: number; sizeVram: number; expiresAt: string }

  const SETTINGS_KEY = 'ollama-chat:settings'
  const SYSTEM_PROMPT = 'Be brief and precise. Answer directly, skip preamble and filler, and do not pad your response with unnecessary detail.'
  const VRAM_CAPACITY_BYTES = 16 * 1024 ** 3

  let storedSettings = { host: '', model: '', contextSize: '' }
  try {
    const raw = localStorage.getItem(SETTINGS_KEY)
    if (raw) storedSettings = { ...storedSettings, ...JSON.parse(raw) }
  } catch {
    // ignore malformed storage
  }

  let host = $state(storedSettings.host)
  let model = $state(storedSettings.model)
  let contextSize = $state(storedSettings.contextSize)
  let models = $state<string[]>([])
  let status = $state<Status>('idle')
  let statusMessage = $state('Not connected')
  let settingsOpen = $state(storedSettings.host === '')

  let messages = $state<ChatMessage[]>([])
  let input = $state('')
  let isStreaming = $state(false)
  let isUnloading = $state(false)
  let loadedModels = $state<LoadedModel[]>([])
  let nextId = 1
  let abortController: AbortController | null = null

  let messagesEl: HTMLElement | null = null
  let textareaEl: HTMLTextAreaElement | null = null
  let checkTimer: ReturnType<typeof setTimeout> | undefined

  function baseUrl(): string {
    let value = host.trim()
    if (!/^https?:\/\//i.test(value)) value = `http://${value}`
    return value.replace(/\/+$/, '')
  }

  function persistSettings() {
    localStorage.setItem(SETTINGS_KEY, JSON.stringify({ host, model, contextSize }))
  }

  async function checkConnection() {
    if (!host.trim()) {
      status = 'idle'
      statusMessage = 'Not connected'
      return
    }
    status = 'checking'
    statusMessage = 'Checking…'
    try {
      const controller = new AbortController()
      const timeout = setTimeout(() => controller.abort(), 5000)
      const res = await fetch(`${baseUrl()}/api/tags`, { signal: controller.signal })
      clearTimeout(timeout)
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      const data = await res.json()
      models = (data.models ?? []).map((m: { name: string }) => m.name)
      if (!model && models.length) model = models[0]
      status = 'connected'
      statusMessage = `Connected · ${models.length} model${models.length === 1 ? '' : 's'}`
      persistSettings()
      fetchLoadedModels()
    } catch (err) {
      status = 'error'
      statusMessage = err instanceof Error ? err.message : 'Connection failed'
      models = []
      loadedModels = []
    }
  }

  function onHostInput() {
    clearTimeout(checkTimer)
    checkTimer = setTimeout(checkConnection, 600)
  }

  function autoGrow() {
    if (!textareaEl) return
    textareaEl.style.height = 'auto'
    textareaEl.style.height = `${Math.min(textareaEl.scrollHeight, 200)}px`
  }

  function scrollToBottom() {
    requestAnimationFrame(() => {
      if (messagesEl) messagesEl.scrollTop = messagesEl.scrollHeight
    })
  }

  async function sendMessage() {
    const text = input.trim()
    if (!text || isStreaming) return
    if (status !== 'connected') {
      messages.push({ id: nextId++, role: 'system', content: 'Not connected to an Ollama server. Set the address above.' })
      scrollToBottom()
      return
    }
    if (!model) {
      messages.push({ id: nextId++, role: 'system', content: 'No model selected.' })
      scrollToBottom()
      return
    }

    messages.push({ id: nextId++, role: 'user', content: text })
    input = ''
    autoGrow()
    scrollToBottom()

    const assistantMsg: ChatMessage = $state({ id: nextId++, role: 'assistant', content: '', thinking: '', streaming: true })
    messages.push(assistantMsg)
    isStreaming = true
    abortController = new AbortController()

    try {
      const history = [
        { role: 'system', content: SYSTEM_PROMPT },
        ...messages
          .filter((m) => m.role === 'user' || m.role === 'assistant')
          .filter((m) => m !== assistantMsg)
          .map((m) => ({ role: m.role, content: m.content })),
      ]

      const numCtx = parseInt(contextSize, 10)
      const options = Number.isFinite(numCtx) && numCtx > 0 ? { num_ctx: numCtx } : undefined

      const res = await fetch(`${baseUrl()}/api/chat`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ model, messages: history, stream: true, options }),
        signal: abortController.signal,
      })

      if (!res.ok || !res.body) {
        const errText = await res.text().catch(() => '')
        throw new Error(errText || `HTTP ${res.status}`)
      }

      const reader = res.body.getReader()
      const decoder = new TextDecoder()
      let buffer = ''

      while (true) {
        const { value, done } = await reader.read()
        if (done) break
        buffer += decoder.decode(value, { stream: true })
        const lines = buffer.split('\n')
        buffer = lines.pop() ?? ''
        for (const line of lines) {
          if (!line.trim()) continue
          const chunk = JSON.parse(line)
          if (chunk.message?.thinking) {
            assistantMsg.thinking = (assistantMsg.thinking ?? '') + chunk.message.thinking
            scrollToBottom()
          }
          if (chunk.message?.content) {
            assistantMsg.content += chunk.message.content
            scrollToBottom()
          }
          if (chunk.error) throw new Error(chunk.error)
        }
      }
      if (buffer.trim()) {
        const chunk = JSON.parse(buffer)
        if (chunk.message?.thinking) assistantMsg.thinking = (assistantMsg.thinking ?? '') + chunk.message.thinking
        if (chunk.message?.content) assistantMsg.content += chunk.message.content
      }
    } catch (err) {
      if (!(err instanceof DOMException && err.name === 'AbortError')) {
        assistantMsg.content += `\n\n[error: ${err instanceof Error ? err.message : 'stream failed'}]`
      }
    } finally {
      assistantMsg.streaming = false
      isStreaming = false
      abortController = null
      scrollToBottom()
      fetchLoadedModels()
    }
  }

  function stopGeneration() {
    abortController?.abort()
  }

  async function unloadModel() {
    if (!model || isUnloading) return
    isUnloading = true
    try {
      const res = await fetch(`${baseUrl()}/api/generate`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ model, keep_alive: 0 }),
      })
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      messages.push({ id: nextId++, role: 'system', content: `Unloaded "${model}" from memory.` })
    } catch (err) {
      messages.push({ id: nextId++, role: 'system', content: `Failed to unload model: ${err instanceof Error ? err.message : 'unknown error'}` })
    } finally {
      isUnloading = false
      scrollToBottom()
      fetchLoadedModels()
    }
  }

  async function fetchLoadedModels() {
    if (status !== 'connected') {
      loadedModels = []
      return
    }
    try {
      const res = await fetch(`${baseUrl()}/api/ps`)
      if (!res.ok) throw new Error()
      const data = await res.json()
      loadedModels = (data.models ?? []).map((m: { name: string; size: number; size_vram: number; expires_at: string }) => ({
        name: m.name,
        size: m.size ?? 0,
        sizeVram: m.size_vram ?? 0,
        expiresAt: m.expires_at,
      }))
    } catch {
      loadedModels = []
    }
  }

  $effect(() => {
    fetchLoadedModels()
    const timer = setInterval(fetchLoadedModels, 4000)
    return () => clearInterval(timer)
  })

  function formatBytes(n: number): string {
    if (!n) return '0 B'
    const units = ['B', 'KiB', 'MiB', 'GiB', 'TiB']
    let value = n
    let i = 0
    while (value >= 1024 && i < units.length - 1) {
      value /= 1024
      i++
    }
    return `${value.toFixed(value < 10 ? 1 : 0)} ${units[i]}`
  }

  function formatCountdown(expiresAt: string): string {
    const ms = new Date(expiresAt).getTime() - Date.now()
    if (ms <= 0) return 'expiring'
    const totalSec = Math.floor(ms / 1000)
    const m = Math.floor(totalSec / 60)
    const s = totalSec % 60
    return `${m}m ${s.toString().padStart(2, '0')}s`
  }

  function meterBar(percent: number, width = 24): [string, string] {
    const filled = Math.round((Math.min(100, Math.max(0, percent)) / 100) * width)
    return ['█'.repeat(filled), '░'.repeat(width - filled)]
  }

  function clearChat() {
    if (messages.length && !confirm('Clear the conversation?')) return
    messages = []
  }

  function onKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault()
      sendMessage()
    }
  }
</script>

<div class="shell">
  <header class="topbar">
    <div class="brand">
      <span class="rings" class:spin={isStreaming} aria-hidden="true">
        <span class="ring r1"></span>
        <span class="ring r2"></span>
        <span class="ring r3"></span>
      </span>
      <h1>OLLAMA<span>_</span>CHAT</h1>
    </div>

    <button class="status-pill" class:connected={status === 'connected'} class:error={status === 'error'} onclick={() => (settingsOpen = !settingsOpen)}>
      <span class="dot"></span>
      {statusMessage}
    </button>
  </header>

  {#if settingsOpen}
    <section class="settings">
      <label class="field">
        <span>SERVER ADDRESS</span>
        <input
          type="text"
          placeholder="192.168.1.50:11434"
          bind:value={host}
          oninput={onHostInput}
        />
      </label>

      <label class="field model-field">
        <span>MODEL</span>
        {#if models.length}
          <select bind:value={model} onchange={persistSettings}>
            {#each models as m (m)}
              <option value={m}>{m}</option>
            {/each}
          </select>
        {:else}
          <input type="text" placeholder="e.g. llama3.1" bind:value={model} onchange={persistSettings} />
        {/if}
      </label>

      <label class="field context-field">
        <span>CONTEXT SIZE</span>
        <input
          type="number"
          min="1"
          step="1"
          placeholder="model default"
          bind:value={contextSize}
          onchange={persistSettings}
        />
      </label>

      <button class="ghost-btn" onclick={checkConnection}>↻ Refresh</button>
      <button
        class="ghost-btn danger"
        onclick={unloadModel}
        disabled={!model || status !== 'connected' || isUnloading}
        title="Unload the model from the Ollama server's memory"
      >
        {isUnloading ? 'Unloading…' : '⏻ Unload Model'}
      </button>
      <button class="ghost-btn" onclick={() => (settingsOpen = false)}>Done</button>
    </section>
  {/if}

  <section class="btop-panel">
    <span class="btop-title">OLLAMA · LOADED MODELS</span>
    {#if !loadedModels.length}
      <p class="btop-empty">No models currently loaded.</p>
    {/if}
    {#each loadedModels as m (m.name)}
      {@const percent = Math.round((m.sizeVram / VRAM_CAPACITY_BYTES) * 100)}
      {@const [filled, empty] = meterBar(percent)}
      <div class="btop-row">
        <div class="btop-row-head">
          <span class="btop-name">{m.name}</span>
          <span class="btop-expiry">keep-alive {formatCountdown(m.expiresAt)}</span>
        </div>
        <div class="btop-meter">
          <span class="meter-label">VRAM</span>
          <span class="meter-bar"><span class="meter-filled">{filled}</span><span class="meter-empty">{empty}</span></span>
          <span class="meter-pct">{percent}%</span>
          <span class="meter-size">{formatBytes(m.sizeVram)} / {formatBytes(VRAM_CAPACITY_BYTES)}</span>
        </div>
      </div>
    {/each}
  </section>

  <main class="messages" bind:this={messagesEl}>
    {#if !messages.length}
      <div class="empty">
        <span class="rings big" aria-hidden="true">
          <span class="ring r1"></span>
          <span class="ring r2"></span>
          <span class="ring r3"></span>
        </span>
        <p>Point this at your Ollama server and start talking.</p>
      </div>
    {/if}

    {#each messages as msg (msg.id)}
      <div class="msg {msg.role}">
        <div class="role">{msg.role === 'user' ? 'YOU' : msg.role === 'assistant' ? 'MODEL' : 'SYSTEM'}</div>
        {#if msg.thinking}
          <details class="thinking">
            <summary>
              <span class="chevron">▸</span>
              Thinking
              {#if msg.streaming && !msg.content}<span class="thinking-pulse"></span>{/if}
            </summary>
            <div class="thinking-body">{msg.thinking}</div>
          </details>
        {/if}
        {#if msg.content || !msg.thinking}
          <div class="bubble">
            {msg.content}{#if msg.streaming}<span class="cursor"></span>{/if}
          </div>
        {/if}
      </div>
    {/each}
  </main>

  <footer class="composer">
    <textarea
      bind:this={textareaEl}
      bind:value={input}
      oninput={autoGrow}
      onkeydown={onKeydown}
      placeholder="Type a message… (Enter to send, Shift+Enter for newline)"
      rows="1"
    ></textarea>
    {#if isStreaming}
      <button class="send-btn stop" onclick={stopGeneration} title="Stop generating">■</button>
    {:else}
      <button class="send-btn" onclick={sendMessage} disabled={!input.trim()} title="Send">➤</button>
    {/if}
    <button class="clear-btn" onclick={clearChat} title="Clear chat" disabled={!messages.length}>✕</button>
  </footer>
</div>

<style>
  .shell {
    display: flex;
    flex-direction: column;
    height: 100%;
    max-width: 880px;
    margin: 0 auto;
    border-left: 1px solid var(--border);
    border-right: 1px solid var(--border);
  }

  .topbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 16px 20px;
    border-bottom: 1px solid var(--border);
    background: var(--black-soft);
  }

  .brand {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .brand h1 {
    font-size: 16px;
    font-weight: 600;
    letter-spacing: 2px;
    margin: 0;
    color: var(--white);
  }
  .brand h1 span {
    color: var(--velvet-bright);
  }

  .rings {
    display: inline-block;
    position: relative;
    width: 26px;
    height: 26px;
    flex-shrink: 0;
  }
  .rings.big {
    width: 96px;
    height: 96px;
    margin: 0 auto 20px;
  }
  .rings .ring {
    position: absolute;
    inset: 0;
    border-radius: 50%;
    border: 1.5px solid var(--velvet-bright);
  }
  .rings .ring.r2 {
    inset: 22%;
    border-color: var(--white-dim);
    opacity: 0.7;
  }
  .rings .ring.r3 {
    inset: 44%;
    border-color: var(--velvet-bright);
  }
  .rings.spin {
    animation: spin 2.4s linear infinite;
  }
  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  .status-pill {
    display: flex;
    align-items: center;
    gap: 8px;
    background: var(--panel);
    border: 1px solid var(--border);
    color: var(--white-dim);
    font-size: 12px;
    letter-spacing: 0.5px;
    padding: 7px 12px;
    border-radius: 999px;
    cursor: pointer;
  }
  .status-pill .dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: var(--white-dim);
  }
  .status-pill.connected .dot {
    background: var(--velvet-bright);
    box-shadow: 0 0 8px var(--velvet-glow);
  }
  .status-pill.connected {
    color: var(--white);
  }
  .status-pill.error .dot {
    background: #ff5470;
    box-shadow: 0 0 8px rgba(255, 84, 112, 0.5);
  }
  .status-pill.error {
    color: var(--white);
  }

  .settings {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    align-items: end;
    padding: 16px 20px;
    background: var(--panel);
    border-bottom: 1px solid var(--border);
  }
  .field {
    display: flex;
    flex-direction: column;
    gap: 6px;
    flex: 1 1 200px;
  }
  .field span {
    font-size: 10px;
    letter-spacing: 1.5px;
    color: var(--white-dim);
  }
  .model-field {
    flex: 1 1 160px;
  }
  .context-field {
    flex: 1 1 130px;
  }
  .context-field input::-webkit-outer-spin-button,
  .context-field input::-webkit-inner-spin-button {
    -webkit-appearance: none;
    margin: 0;
  }
  .context-field input {
    appearance: textfield;
    -moz-appearance: textfield;
  }
  input,
  select {
    background: var(--black);
    border: 1px solid var(--border);
    color: var(--white);
    padding: 9px 10px;
    border-radius: 6px;
    font-size: 13px;
    outline: none;
  }
  input:focus,
  select:focus {
    border-color: var(--velvet-bright);
    box-shadow: 0 0 0 3px var(--velvet-glow);
  }
  .ghost-btn {
    background: transparent;
    border: 1px solid var(--border);
    color: var(--white-dim);
    padding: 9px 14px;
    border-radius: 6px;
    cursor: pointer;
    font-size: 12px;
  }
  .ghost-btn:hover:not(:disabled) {
    border-color: var(--velvet-bright);
    color: var(--white);
  }
  .ghost-btn:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
  .ghost-btn.danger:hover:not(:disabled) {
    border-color: #ff5470;
    color: #ff5470;
  }

  .btop-panel {
    position: relative;
    margin: 16px 20px 0;
    padding: 16px 16px 14px;
    background: var(--panel);
    border: 1px solid var(--border);
    border-radius: 8px;
  }
  .btop-title {
    position: absolute;
    top: -9px;
    left: 14px;
    padding: 0 8px;
    background: var(--panel);
    color: var(--velvet-bright);
    font-size: 10px;
    letter-spacing: 2px;
  }
  .btop-empty {
    margin: 0;
    color: var(--white-dim);
    font-size: 12px;
  }
  .btop-row + .btop-row {
    margin-top: 12px;
    padding-top: 12px;
    border-top: 1px dashed var(--border);
  }
  .btop-row-head {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    margin-bottom: 6px;
    gap: 12px;
  }
  .btop-name {
    font-size: 13px;
    font-weight: 600;
    color: var(--white);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  .btop-expiry {
    flex-shrink: 0;
    font-size: 11px;
    color: var(--white-dim);
  }
  .btop-meter {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 12px;
  }
  .meter-label {
    flex-shrink: 0;
    width: 36px;
    font-size: 10px;
    letter-spacing: 1px;
    color: var(--white-dim);
  }
  .meter-bar {
    flex: 1;
    overflow: hidden;
    white-space: nowrap;
    letter-spacing: -1px;
    line-height: 1;
  }
  .meter-filled {
    color: var(--velvet-bright);
  }
  .meter-empty {
    color: var(--border);
  }
  .meter-pct {
    flex-shrink: 0;
    width: 36px;
    text-align: right;
    color: var(--white);
  }
  .meter-size {
    flex-shrink: 0;
    min-width: 130px;
    text-align: right;
    font-size: 11px;
    color: var(--white-dim);
  }

  .messages {
    flex: 1;
    overflow-y: auto;
    padding: 24px 20px;
    display: flex;
    flex-direction: column;
    gap: 18px;
  }

  .empty {
    margin: auto;
    text-align: center;
    color: var(--white-dim);
    font-size: 13px;
  }

  .msg {
    display: flex;
    flex-direction: column;
    max-width: 78%;
  }
  .msg.user {
    align-self: flex-end;
    align-items: flex-end;
  }
  .msg.assistant,
  .msg.system {
    align-self: flex-start;
    align-items: flex-start;
  }
  .role {
    font-size: 10px;
    letter-spacing: 1.5px;
    color: var(--white-dim);
    margin-bottom: 6px;
  }
  .bubble {
    white-space: pre-wrap;
    word-break: break-word;
    padding: 12px 14px;
    border-radius: 10px;
    font-size: 14px;
    line-height: 1.55;
  }
  .msg.user .bubble {
    background: var(--white);
    color: var(--black);
    border-radius: 10px 10px 2px 10px;
  }
  .msg.assistant .bubble {
    background: var(--panel-raised);
    border: 1px solid var(--border);
    border-left: 2px solid var(--velvet-bright);
    border-radius: 10px 10px 10px 2px;
  }
  .msg.system .bubble {
    background: transparent;
    border: 1px dashed var(--border);
    color: var(--white-dim);
    font-size: 12px;
  }

  .thinking {
    width: 100%;
    margin-bottom: 6px;
  }
  .thinking summary {
    display: flex;
    align-items: center;
    gap: 6px;
    list-style: none;
    cursor: pointer;
    font-size: 11px;
    letter-spacing: 1px;
    color: var(--white-dim);
    padding: 6px 10px;
    border: 1px solid var(--border);
    border-radius: 8px;
    background: var(--panel);
    user-select: none;
  }
  .thinking summary::-webkit-details-marker {
    display: none;
  }
  .thinking summary:hover {
    color: var(--white);
    border-color: var(--velvet-bright);
  }
  .thinking .chevron {
    display: inline-block;
    transition: transform 0.15s;
    color: var(--velvet-bright);
  }
  .thinking[open] .chevron {
    transform: rotate(90deg);
  }
  .thinking-body {
    white-space: pre-wrap;
    word-break: break-word;
    font-size: 12.5px;
    line-height: 1.55;
    color: var(--white-dim);
    padding: 10px 12px;
    margin-top: 4px;
    border: 1px dashed var(--border);
    border-radius: 8px;
    background: var(--black-soft);
  }
  .thinking-pulse {
    display: inline-block;
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: var(--velvet-bright);
    animation: blink 1s step-start infinite;
  }

  .cursor {
    display: inline-block;
    width: 7px;
    height: 14px;
    margin-left: 2px;
    background: var(--velvet-bright);
    vertical-align: -2px;
    animation: blink 1s step-start infinite;
  }
  @keyframes blink {
    50% {
      opacity: 0;
    }
  }

  .composer {
    display: flex;
    gap: 10px;
    align-items: flex-end;
    padding: 16px 20px;
    border-top: 1px solid var(--border);
    background: var(--black-soft);
  }
  textarea {
    flex: 1;
    resize: none;
    background: var(--panel);
    border: 1px solid var(--border);
    color: var(--white);
    border-radius: 10px;
    padding: 12px 14px;
    font-size: 14px;
    max-height: 200px;
    outline: none;
  }
  textarea:focus {
    border-color: var(--velvet-bright);
    box-shadow: 0 0 0 3px var(--velvet-glow);
  }

  .send-btn,
  .clear-btn {
    width: 42px;
    height: 42px;
    border-radius: 50%;
    border: 1px solid var(--border);
    background: var(--panel);
    color: var(--white);
    cursor: pointer;
    font-size: 15px;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    transition: box-shadow 0.2s, border-color 0.2s;
  }
  .send-btn {
    background: var(--velvet);
    border-color: var(--velvet-bright);
  }
  .send-btn:hover:not(:disabled) {
    box-shadow: 0 0 0 5px var(--velvet-glow);
  }
  .send-btn:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
  .send-btn.stop {
    background: var(--black);
    color: var(--velvet-bright);
  }
  .clear-btn:hover:not(:disabled) {
    border-color: var(--velvet-bright);
    color: var(--velvet-bright);
  }
  .clear-btn:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }
</style>
