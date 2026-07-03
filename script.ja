/* =========================================================
   TriguAI — application logic
   - Menyimpan seluruh percakapan ke localStorage
   - Memanggil Gemini API langsung dari browser
   - Pencarian percakapan berdasarkan judul (substring match)
========================================================= */

const STORAGE_KEY = "triguai_conversations_v1";
const API_KEY_STORAGE = "triguai_gemini_api_key";
const MODEL_STORAGE = "triguai_gemini_model";

const state = {
  conversations: [],   // [{id, title, messages:[{role,text}], createdAt, updatedAt}]
  activeId: null,
  apiKey: "",
  model: "gemini-2.0-flash",
};

// ---------- DOM refs ----------
const sidebar = document.getElementById("sidebar");
const historyList = document.getElementById("historyList");
const searchInput = document.getElementById("searchInput");
const newChatBtn = document.getElementById("newChatBtn");
const deleteChatBtn = document.getElementById("deleteChatBtn");
const chatTitle = document.getElementById("chatTitle");
const chatArea = document.getElementById("chatArea");
const emptyState = document.getElementById("emptyState");
const messagesEl = document.getElementById("messages");
const composerForm = document.getElementById("composerForm");
const promptInput = document.getElementById("promptInput");
const sendBtn = document.getElementById("sendBtn");
const menuToggle = document.getElementById("menuToggle");

const modalOverlay = document.getElementById("modalOverlay");
const settingsBtn = document.getElementById("settingsBtn");
const closeModalBtn = document.getElementById("closeModalBtn");
const apiKeyInput = document.getElementById("apiKeyInput");
const modelSelect = document.getElementById("modelSelect");
const saveApiKeyBtn = document.getElementById("saveApiKeyBtn");

// ---------- Storage helpers ----------
function loadConversations() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    return raw ? JSON.parse(raw) : [];
  } catch (e) {
    console.error("Gagal membaca localStorage:", e);
    return [];
  }
}

function saveConversations() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state.conversations));
}

function loadSettings() {
  state.apiKey = localStorage.getItem(API_KEY_STORAGE) || "";
  state.model = localStorage.getItem(MODEL_STORAGE) || "gemini-2.0-flash";
}

// ---------- Utilities ----------
function uid() {
  return Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
}

function escapeHtml(str) {
  return str
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;");
}

// Ubah teks biasa + code block sederhana (```) menjadi HTML aman
function renderBubbleContent(text) {
  const escaped = escapeHtml(text);
  const withCode = escaped.replace(/```([\s\S]*?)```/g, (_, code) => {
    return `<pre><code>${code.trim()}</code></pre>`;
  });
  const withInlineCode = withCode.replace(/`([^`]+)`/g, "<code>$1</code>");
  const withParagraphs = withInlineCode
    .split(/\n{2,}/)
    .map(p => `<p>${p.replace(/\n/g, "<br>")}</p>`)
    .join("");
  return withParagraphs;
}

function formatRelativeTime(ts) {
  const diff = Date.now() - ts;
  const min = Math.floor(diff / 60000);
  if (min < 1) return "Baru saja";
  if (min < 60) return `${min} menit lalu`;
  const hr = Math.floor(min / 60);
  if (hr < 24) return `${hr} jam lalu`;
  const day = Math.floor(hr / 24);
  if (day < 7) return `${day} hari lalu`;
  return new Date(ts).toLocaleDateString("id-ID");
}

// ---------- Conversation management ----------
function getActiveConversation() {
  return state.conversations.find(c => c.id === state.activeId) || null;
}

function createConversation() {
  const convo = {
    id: uid(),
    title: "Percakapan Baru",
    messages: [],
    createdAt: Date.now(),
    updatedAt: Date.now(),
  };
  state.conversations.unshift(convo);
  state.activeId = convo.id;
  saveConversations();
  renderHistory();
  renderActiveChat();
}

function deleteActiveConversation() {
  const convo = getActiveConversation();
  if (!convo) return;
  if (!confirm(`Hapus percakapan "${convo.title}"?`)) return;
  state.conversations = state.conversations.filter(c => c.id !== convo.id);
  state.activeId = state.conversations[0]?.id || null;
  saveConversations();
  renderHistory();
  renderActiveChat();
}

function setActiveConversation(id) {
  state.activeId = id;
  renderHistory();
  renderActiveChat();
  if (window.innerWidth <= 860) sidebar.classList.remove("open");
}

function deriveTitle(firstMessage) {
  const clean = firstMessage.trim().replace(/\s+/g, " ");
  return clean.length > 42 ? clean.slice(0, 42) + "…" : clean || "Percakapan Baru";
}

// ---------- Rendering: history / search ----------
function renderHistory() {
  const query = searchInput.value.trim().toLowerCase();
  const filtered = query
    ? state.conversations.filter(c => c.title.toLowerCase().includes(query))
    : state.conversations;

  historyList.innerHTML = "";

  if (filtered.length === 0) {
    const empty = document.createElement("div");
    empty.className = "history-empty";
    empty.textContent = query ? "Tidak ada percakapan cocok." : "Belum ada percakapan.";
    historyList.appendChild(empty);
    return;
  }

  filtered.forEach((c, idx) => {
    const item = document.createElement("div");
    item.className = "history-item" + (c.id === state.activeId ? " active" : "");
    item.style.animationDelay = `${idx * 0.02}s`;

    const titleEl = document.createElement("div");
    titleEl.className = "h-title";
    titleEl.innerHTML = highlightMatch(c.title, query);

    const metaEl = document.createElement("div");
    metaEl.className = "h-meta";
    metaEl.textContent = formatRelativeTime(c.updatedAt);

    item.appendChild(titleEl);
    item.appendChild(metaEl);
    item.addEventListener("click", () => setActiveConversation(c.id));
    historyList.appendChild(item);
  });
}

function highlightMatch(title, query) {
  const safeTitle = escapeHtml(title);
  if (!query) return safeTitle;
  const idx = safeTitle.toLowerCase().indexOf(query.toLowerCase());
  if (idx === -1) return safeTitle;
  return (
    safeTitle.slice(0, idx) +
    "<mark>" + safeTitle.slice(idx, idx + query.length) + "</mark>" +
    safeTitle.slice(idx + query.length)
  );
}

// ---------- Rendering: chat area ----------
function renderActiveChat() {
  const convo = getActiveConversation();

  if (!convo) {
    chatTitle.textContent = "Percakapan Baru";
    emptyState.style.display = "flex";
    messagesEl.innerHTML = "";
    return;
  }

  chatTitle.textContent = convo.title;

  if (convo.messages.length === 0) {
    emptyState.style.display = "flex";
    messagesEl.innerHTML = "";
    return;
  }

  emptyState.style.display = "none";
  messagesEl.innerHTML = "";
  convo.messages.forEach(m => appendMessageToDOM(m.role, m.text, false));
  scrollChatToBottom();
}

function appendMessageToDOM(role, text, animate = true) {
  const msg = document.createElement("div");
  msg.className = `msg ${role}`;
  if (!animate) msg.style.animation = "none";

  const avatar = document.createElement("div");
  avatar.className = "avatar";
  if (role === "bot") msg.appendChild(avatar);

  const bubble = document.createElement("div");
  bubble.className = "bubble";
  bubble.innerHTML = renderBubbleContent(text);

  msg.appendChild(bubble);
  messagesEl.appendChild(msg);
}

function scrollChatToBottom() {
  requestAnimationFrame(() => {
    chatArea.scrollTop = chatArea.scrollHeight;
  });
}

function showTypingIndicator() {
  const msg = document.createElement("div");
  msg.className = "msg bot";
  msg.id = "typingIndicator";

  const avatar = document.createElement("div");
  avatar.className = "avatar";

  const bubble = document.createElement("div");
  bubble.className = "bubble";
  bubble.innerHTML = `<div class="typing"><span></span><span></span><span></span></div>`;

  msg.appendChild(avatar);
  msg.appendChild(bubble);
  messagesEl.appendChild(msg);
  scrollChatToBottom();
}

function removeTypingIndicator() {
  document.getElementById("typingIndicator")?.remove();
}

// ---------- Gemini API ----------
async function callGeminiAPI(convo) {
  if (!state.apiKey) {
    openModal();
    throw new Error("API key belum diatur.");
  }

  const url = `https://generativelanguage.googleapis.com/v1beta/models/${state.model}:generateContent?key=${state.apiKey}`;

  const contents = convo.messages.map(m => ({
    role: m.role === "user" ? "user" : "model",
    parts: [{ text: m.text }],
  }));

  const res = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ contents }),
  });

  if (!res.ok) {
    const errBody = await res.json().catch(() => ({}));
    const message = errBody?.error?.message || `Permintaan gagal (status ${res.status}).`;
    throw new Error(message);
  }

  const data = await res.json();
  const text = data?.candidates?.[0]?.content?.parts?.map(p => p.text).join("") || "";
  return text || "Maaf, tidak ada respons yang bisa ditampilkan.";
}

// ---------- Message send flow ----------
async function handleSend(e) {
  e.preventDefault();
  const text = promptInput.value.trim();
  if (!text) return;

  let convo = getActiveConversation();
  if (!convo) {
    createConversation();
    convo = getActiveConversation();
  }

  const isFirstMessage = convo.messages.length === 0;
  convo.messages.push({ role: "user", text });
  if (isFirstMessage) {
    convo.title = deriveTitle(text);
    chatTitle.textContent = convo.title;
  }
  convo.updatedAt = Date.now();
  saveConversations();

  emptyState.style.display = "none";
  appendMessageToDOM("user", text);
  scrollChatToBottom();

  promptInput.value = "";
  autoResizeTextarea();
  renderHistory();

  sendBtn.disabled = true;
  showTypingIndicator();

  try {
    const replyText = await callGeminiAPI(convo);
    removeTypingIndicator();
    convo.messages.push({ role: "bot", text: replyText });
    convo.updatedAt = Date.now();
    saveConversations();
    appendMessageToDOM("bot", replyText);
    scrollChatToBottom();
    renderHistory();
  } catch (err) {
    removeTypingIndicator();
    const message = err?.message || "Terjadi kesalahan saat menghubungi Gemini API.";
    appendMessageToDOM("bot", `Terjadi kesalahan: ${message}`);
    scrollChatToBottom();
  } finally {
    sendBtn.disabled = false;
  }
}

function autoResizeTextarea() {
  promptInput.style.height = "auto";
  promptInput.style.height = Math.min(promptInput.scrollHeight, 160) + "px";
}

// ---------- Modal ----------
function openModal() {
  apiKeyInput.value = state.apiKey;
  modelSelect.value = state.model;
  modalOverlay.classList.add("open");
}
function closeModal() {
  modalOverlay.classList.remove("open");
}
function saveApiSettings() {
  state.apiKey = apiKeyInput.value.trim();
  state.model = modelSelect.value;
  localStorage.setItem(API_KEY_STORAGE, state.apiKey);
  localStorage.setItem(MODEL_STORAGE, state.model);
  closeModal();
}

// ---------- Event listeners ----------
newChatBtn.addEventListener("click", createConversation);
deleteChatBtn.addEventListener("click", deleteActiveConversation);
searchInput.addEventListener("input", renderHistory);
composerForm.addEventListener("submit", handleSend);
promptInput.addEventListener("input", autoResizeTextarea);
promptInput.addEventListener("keydown", (e) => {
  if (e.key === "Enter" && !e.shiftKey) {
    e.preventDefault();
    composerForm.requestSubmit();
  }
});

settingsBtn.addEventListener("click", openModal);
closeModalBtn.addEventListener("click", closeModal);
saveApiKeyBtn.addEventListener("click", saveApiSettings);
modalOverlay.addEventListener("click", (e) => {
  if (e.target === modalOverlay) closeModal();
});

menuToggle.addEventListener("click", () => sidebar.classList.toggle("open"));

// ---------- Init ----------
function init() {
  loadSettings();
  state.conversations = loadConversations();
  state.activeId = state.conversations[0]?.id || null;

  if (!state.apiKey) {
    // Tampilkan modal pengaturan otomatis jika belum ada API key
    setTimeout(openModal, 400);
  }

  renderHistory();
  renderActiveChat();
}

init();
