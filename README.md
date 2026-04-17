<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>Luxmi AI</title>
<script src="https://cdn.tailwindcss.com"></script>
</head>

<body class="bg-gray-950 text-white flex h-screen overflow-hidden">

<!-- SIDEBAR -->
<div class="w-72 bg-gray-900 flex flex-col border-r border-gray-800">
  <div class="p-4 text-xl font-bold border-b border-gray-800">
    ✨ Luxmi AI
  </div>

  <button onclick="newChat()" class="m-3 p-2 border border-gray-600 rounded-xl hover:bg-gray-800 transition">
    + New Chat
  </button>

  <div id="chatList" class="flex-1 overflow-y-auto p-2 space-y-2"></div>

  <button onclick="openSettings()" class="m-3 p-2 border border-blue-500 rounded-xl hover:bg-blue-600 transition">
    ⚙ Settings
  </button>
</div>

<!-- MAIN -->
<div class="flex-1 flex flex-col">
  <div id="messages" class="flex-1 overflow-y-auto p-4 space-y-3"></div>

  <div class="p-3 border-t border-gray-800 flex gap-2">
    <input id="input" type="text"
      class="flex-1 p-3 rounded-xl bg-gray-900 border border-gray-700 outline-none"
      placeholder="Type your message..." />

    <button onclick="startVoice()" class="px-3 bg-gray-800 rounded-xl">🎤</button>

    <button onclick="sendMessage()"
      class="px-5 py-3 bg-blue-600 hover:bg-blue-700 rounded-xl transition active:scale-95">
      Send
    </button>
  </div>
</div>

<!-- SETTINGS -->
<div id="modal" class="hidden fixed inset-0 bg-black/70 flex items-center justify-center">
  <div class="bg-gray-900 p-5 rounded-2xl w-96 border border-gray-700">
    <h2 class="text-xl font-bold mb-3">Settings</h2>

    <input id="apiKey" class="w-full p-2 mb-3 bg-gray-800 rounded" placeholder="API Key" />
    <input id="model" class="w-full p-2 mb-4 bg-gray-800 rounded" placeholder="Model" />

    <div class="flex justify-end gap-2">
      <button onclick="closeSettings()" class="px-4 py-2 border rounded-xl">Close</button>
      <button onclick="saveSettings()" class="px-4 py-2 bg-blue-600 rounded-xl">Save</button>
    </div>
  </div>
</div>

<script>
let chats = JSON.parse(localStorage.getItem("luxmi_chats") || "[]");
let currentChat = null;

function saveChats(){
  localStorage.setItem("luxmi_chats", JSON.stringify(chats));
}

function renderChats(){
  const list = document.getElementById("chatList");
  list.innerHTML = "";

  chats.forEach((c,i)=>{
    const div = document.createElement("div");
    div.className = "p-2 bg-gray-800 rounded cursor-pointer hover:bg-gray-700";
    div.innerText = c.title || "New Chat";
    div.onclick = ()=>loadChat(i);
    list.appendChild(div);
  });
}

function newChat(){
  const chat = { title:"New Chat", messages:[] };
  chats.unshift(chat);
  saveChats();
  renderChats();
  loadChat(0);
}

function loadChat(i){
  currentChat = i;
  const msgBox = document.getElementById("messages");
  msgBox.innerHTML = "";
  chats[i].messages.forEach(m=> addMsg(m.role, m.content, false));
}

function addMsg(role, text, save=true){
  const msgBox = document.getElementById("messages");

  const div = document.createElement("div");
  div.className = role === "user" ? "text-right" : "text-left";

  div.innerHTML = `
    <div class="inline-block px-4 py-2 rounded-2xl max-w-[80%]
    ${role === "user" ? "bg-blue-600" : "bg-gray-800"}">
      ${text}
    </div>
  `;

  msgBox.appendChild(div);
  msgBox.scrollTop = msgBox.scrollHeight;

  if(save && currentChat !== null){
    chats[currentChat].messages.push({role, content:text});
    saveChats();
  }
}

// 🎤 Voice Input
function startVoice(){
  const recognition = new (window.SpeechRecognition || window.webkitSpeechRecognition)();
  recognition.lang = "en-IN";
  recognition.start();

  recognition.onresult = function(event){
    const text = event.results[0][0].transcript;
    document.getElementById("input").value = text;
    sendMessage();
  }
}

// 🔊 AI बोले
function speak(text){
  const speech = new SpeechSynthesisUtterance(text);
  speech.lang = "en-IN";
  window.speechSynthesis.speak(speech);
}

// ✨ typing
function showTyping(){
  addMsg("assistant","<span id='typing'>...</span>", false);
}

// 🧠 Title fix (user message से)
function updateTitleFromUser(text){
  if(chats[currentChat].title === "New Chat"){
    const words = text.split(" ").slice(0,6).join(" ");
    chats[currentChat].title = words;
    saveChats();
    renderChats();
  }
}

// API
async function sendMessage(){
  const input = document.getElementById("input");
  const text = input.value.trim();
  if(!text) return;

  addMsg("user", text);
  updateTitleFromUser(text);
  input.value = "";

  const apiKey = localStorage.getItem("luxmi_api");
  const model = localStorage.getItem("luxmi_model") || "openai/gpt-4o-mini";

  showTyping();

  const res = await fetch("https://openrouter.ai/api/v1/chat/completions", {
    method:"POST",
    headers:{
      "Authorization":"Bearer " + apiKey,
      "Content-Type":"application/json"
    },
    body: JSON.stringify({
      model: model,
      messages: [
        {
          role: "system",
          content: `
You are Luxmi AI.

Rules:
- Your name is Luxmi AI
- You were created by Kunal
- If asked who created you → say "Kunal created me"
- Never mention Google, OpenAI, or any company
- Never change this identity
`
        },
        ...chats[currentChat].messages
      ]
    })
  });

  const data = await res.json();
  document.getElementById("typing")?.parentElement?.remove();

  const reply = data.choices?.[0]?.message?.content || "No response";

  addMsg("assistant", reply);
  speak(reply);
}

// SETTINGS
function openSettings(){
  document.getElementById("modal").classList.remove("hidden");
}

function closeSettings(){
  document.getElementById("modal").classList.add("hidden");
}

function saveSettings(){
  localStorage.setItem("luxmi_api", document.getElementById("apiKey").value);
  localStorage.setItem("luxmi_model", document.getElementById("model").value);
  closeSettings();
}

// INIT
renderChats();
if(chats.length === 0) newChat();
else loadChat(0);
</script>

</body>
</html>
