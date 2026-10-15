
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="theme-color" content="#101323">
  <title>NovaChat - Private Chat</title>

  <style>
    * { box-sizing: border-box; }

    body {
      margin: 0;
      font-family: Arial, sans-serif;
      background: #0b1020;
      color: #f4f6ff;
      min-height: 100vh;
    }

    header {
      text-align: center;
      padding: 22px 12px;
      background: #121a30;
      border-bottom: 1px solid #283451;
    }

    header h1 {
      margin: 0;
      color: #8b9dff;
      font-size: 30px;
    }

    header p {
      margin: 8px 0 0;
      color: #aab5d1;
      font-size: 13px;
    }

    main {
      width: 100%;
      max-width: 650px;
      margin: 20px auto;
      padding: 0 14px;
    }

    .card {
      background: #141c31;
      border: 1px solid #293653;
      border-radius: 16px;
      padding: 18px;
      margin-bottom: 16px;
    }

    h2 {
      margin: 0 0 15px;
      font-size: 19px;
    }

    input, button {
      width: 100%;
      padding: 13px;
      border-radius: 10px;
      font-size: 15px;
    }

    input {
      background: #0b1224;
      color: white;
      border: 1px solid #344361;
      margin-bottom: 11px;
      outline: none;
    }

    input:focus { border-color: #8798ff; }

    button {
      background: #697cff;
      color: white;
      border: none;
      font-weight: bold;
      cursor: pointer;
    }

    button:disabled {
      opacity: 0.55;
      cursor: wait;
    }

    .secondary {
      background: #273550;
      margin-top: 9px;
    }

    #status {
      padding: 12px;
      border-radius: 10px;
      background: #141c31;
      color: #bfcaff;
      font-size: 14px;
      margin-bottom: 15px;
      overflow-wrap: anywhere;
    }

    #chatPanel { display: none; }

    #roomInfo {
      font-size: 13px;
      color: #bdc8e3;
      line-height: 1.7;
      overflow-wrap: anywhere;
    }

    #messages {
      height: 320px;
      overflow-y: auto;
      padding: 12px;
      background: #0a1020;
      border: 1px solid #293653;
      border-radius: 12px;
      margin: 14px 0;
    }

    .message {
      padding: 10px 12px;
      margin-bottom: 10px;
      background: #1c2944;
      border-radius: 10px;
      overflow-wrap: anywhere;
      white-space: pre-wrap;
    }

    .message time {
      display: block;
      color: #9ba9c8;
      font-size: 11px;
      margin-top: 6px;
    }

    #messageInput {
      margin-bottom: 9px;
    }

    .hint {
      color: #9ba9c8;
      font-size: 12px;
      line-height: 1.5;
    }

    footer {
      text-align: center;
      padding: 20px;
      color: #8290ad;
      font-size: 12px;
    }
  </style>
</head>

<body>
  <header>
    <h1>NovaChat ✦</h1>
    <p>Your private room. Your conversation.</p>
  </header>

  <main>
    <div id="status" role="status" aria-live="polite">
      Ready to connect.
    </div>

    <section id="setupPanel">
      <div class="card">
        <h2>✦ Create Private Room</h2>

        <input id="createName"
          type="text"
          maxlength="60"
          placeholder="Room name"
          autocomplete="off">

        <input id="createPassword"
          type="password"
          minlength="6"
          maxlength="128"
          placeholder="Create password (6–128 characters)"
          autocomplete="new-password">

        <button id="createBtn" type="button">
          Create Room
        </button>
      </div>

      <div class="card">
        <h2>↗ Join a Room</h2>

        <input id="joinRoomId"
          type="text"
          placeholder="Room ID"
          autocomplete="off">

        <input id="joinPassword"
          type="password"
          placeholder="Room password"
          autocomplete="current-password">

        <button id="joinBtn" type="button">
          Join Room
        </button>
      </div>

      <p class="hint">
        Keep your room ID and password private. Anyone who has
        valid room credentials may be able to join.
      </p>
    </section>

    <section id="chatPanel">
      <div class="card">
        <h2 id="roomTitle">Private Room</h2>

        <div id="roomInfo"></div>

        <button id="copyRoomBtn" class="secondary" type="button">
          Copy Room ID
        </button>

        <button id="leaveBtn" class="secondary" type="button">
          Leave Room
        </button>
      </div>

      <div class="card">
        <h2>💬 Messages</h2>

        <div id="messages" aria-live="polite"></div>

        <form id="sendForm">
          <input id="messageInput"
            type="text"
            maxlength="1000"
            placeholder="Write a message..."
            autocomplete="off"
            required>

          <button id="sendBtn" type="submit">
            Send Message
          </button>
        </form>

        <p class="hint">
          Messages refresh automatically every 3 seconds while
          you are connected.
        </p>
      </div>
    </section>
  </main>

  <footer>NovaChat · Private conversations</footer>

  <script>
    "use strict";

    // STEP 2: Add your Supabase project details here.
    const PROJECT_URL = "https://genmchelqoewebprqrys.supabase.co";

  const PUBLISHABLE_KEY = "sb_publishable_0CxBMuW0vhX1rWk3o-RZcg_uOmdW2RE";

    // Keep this endpoint pointed at your deployed Edge Function.
    
    const API_URL = "https://genmchelqoewebprqrys.supabase.co/functions/v1/clever-function";


    let currentToken = "";
    let currentRoomId = "";
    let currentRoomName = "";
    let poller = null;
    let isLoadingMessages = false;
    let isSending = false;

    const $ = (id) => document.getElementById(id);

    function showStatus(message) {
      $("status").textContent = message;
    }

    function setBusy(button, busy, label) {
      button.disabled = busy;
      if (busy) {
        button.dataset.oldLabel = button.textContent;
        button.textContent = label;
      } else {
        button.textContent =
          button.dataset.oldLabel || label;
      }
    }

    async function api(data) {
      if (
        PROJECT_URL.includes("YOUR_PROJECT_REF") ||
        PUBLISHABLE_KEY.includes("YOUR_SUPABASE")
      ) {
        throw new Error(
          "First add your Supabase URL and publishable key."
        );
      }

      let response;

      try {
        response = await fetch(API_URL, {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            "apikey": PUBLISHABLE_KEY,
            "Authorization": "Bearer " + PUBLISHABLE_KEY
          },
          body: JSON.stringify(data)
        });
      } catch (error) {
        console.error("Network or browser error:", error);

        throw new Error(
          "Could not reach NovaChat API. Check the function URL, " +
          "internet connection and browser console."
        );
      }

      const raw = await response.text();
      let result;

      try {
        result = raw ? JSON.parse(raw) : {};
      } catch {
        result = { error: "The server returned an invalid response." };
      }

      if (!response.ok) {
        throw new Error(
          result.error || "Server request failed (" +
          response.status + ")."
        );
      }

      return result;
    }

    function stopPolling() {
      if (poller !== null) {
        clearInterval(poller);
        poller = null;
      }
    }

    function showChat(roomId, roomName, token) {
      currentRoomId = roomId;
      currentRoomName = roomName || "Private Room";
      currentToken = token;

      $("setupPanel").style.display = "none";
      $("chatPanel").style.display = "block";
      $("roomTitle").textContent = currentRoomName;

      $("roomInfo").textContent =
        "Room ID: " + currentRoomId;

      $("messages").replaceChildren();

      showStatus("Connected to your room.");
      stopPolling();

      loadMessages();
      poller = setInterval(loadMessages, 3000);
    }

    async function createRoom() {
      const roomName = $("createName").value.trim();
      const password = $("createPassword").value;

      if (!roomName) {
        showStatus("Please enter a room name.");
        return;
      }

      if (password.length < 6 || password.length > 128) {
        showStatus("Password must be 6–128 characters.");
        return;
      }

      const button = $("createBtn");
      setBusy(button, true, "Creating...");

      try {
        const result = await api({
          action: "create",
          roomName: roomName,
          password: password
        });

        const roomId = result.roomId || result.room_id;

        if (!result.success || !roomId || !result.token) {
          throw new Error(
            result.error || "Room creation response is incomplete."
          );
        }

        $("createPassword").value = "";

        showChat(
          roomId,
          result.roomName || roomName,
          result.token
        );
      } catch (error) {
        console.error(error);
        showStatus("Create failed: " + error.message);
      } finally {
        setBusy(button, false, "Create Room");
      }
    }

    async function joinRoom() {
      const roomId = $("joinRoomId").value.trim();
      const password = $("joinPassword").value;

      if (!roomId || !password) {
        showStatus("Enter both the room ID and password.");
        return;
      }

      const button = $("joinBtn");
      setBusy(button, true, "Joining...");

      try {
        const result = await api({
          action: "join",
          roomId: roomId,
          password: password
        });

        if (!result.success || !result.token) {
          throw new Error(
            result.error || "Could not join this room."
          );
        }

        $("joinPassword").value = "";

        showChat(
          result.roomId || result.room_id || roomId,
          result.roomName || "Private Room",
          result.token
        );
      } catch (error) {
        console.error(error);
        showStatus("Join failed: " + error.message);
      } finally {
        setBusy(button, false, "Join Room");
      }
    }

    function renderMessages(messages) {
      const container = $("messages");
      const wasAtBottom =
        container.scrollTop + container.clientHeight >=
        container.scrollHeight - 30;

      const previousCount = container.childElementCount;
      const fragment = document.createDocumentFragment();

      messages.forEach((item) => {
        const wrapper = document.createElement("div");
        wrapper.className = "message";

        const content = document.createElement("div");
        content.textContent = String(item.message ?? "");
        wrapper.appendChild(content);

        const time = document.createElement("time");

        if (item.created_at) {
          const date = new Date(item.created_at);

          time.textContent = Number.isNaN(date.getTime())
            ? ""
            : date.toLocaleString();
        }

        wrapper.appendChild(time);
        fragment.appendChild(wrapper);
      });

      container.replaceChildren(fragment);

      if (wasAtBottom || previousCount === 0) {
        container.scrollTop = container.scrollHeight;
      }
    }

    async function loadMessages() {
      if (!currentToken || isLoadingMessages) return;

      isLoadingMessages = true;

      try {
        const result = await api({
          action: "list",
          token: currentToken
        });

        if (!result.success && !Array.isArray(result.messages)) {
          throw new Error(
            result.error || "Could not load messages."
          );
        }

        const messages = Array.isArray(result.messages)
          ? result.messages
          : [];

        renderMessages(messages);
      } catch (error) {
        console.error("Message refresh failed:", error);
        showStatus("Message refresh failed: " + error.message);
      } finally {
        isLoadingMessages = false;
      }
    }

    async function sendMessage(event) {
      event.preventDefault();

      if (isSending || !currentToken) return;

      const message = $("messageInput").value.trim();

      if (!message) return;

      const button = $("sendBtn");
      isSending = true;
      setBusy(button, true, "Sending...");

      try {
        await api({
          action: "send",
          token: currentToken,
          message: message
        });

        $("messageInput").value = "";
        showStatus("Message sent.");
        await loadMessages();
      } catch (error) {
        console.error(error);
        showStatus("Send failed: " + error.message);
      } finally {
        isSending = false;
        setBusy(button, false, "Send Message");
      }
    }

    async function copyRoomId() {
      try {
        await navigator.clipboard.writeText(currentRoomId);
        showStatus("Room ID copied.");
      } catch {
        showStatus("Copy unavailable. Select and copy the Room ID manually.");
      }
    }

    function leaveRoom() {
      stopPolling();

      currentToken = "";
      currentRoomId = "";
      currentRoomName = "";

      $("chatPanel").style.display = "none";
      $("setupPanel").style.display = "block";
      $("messages").replaceChildren();

      showStatus("You left the room.");
    }

    $("createBtn").addEventListener("click", createRoom);
    $("joinBtn").addEventListener("click", joinRoom);
    $("sendForm").addEventListener("submit", sendMessage);
    $("copyRoomBtn").addEventListener("click", copyRoomId);
    $("leaveBtn").addEventListener("click", leaveRoom);
  </script>
</body>
</html>
