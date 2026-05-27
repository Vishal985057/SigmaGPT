# SigmaGPT 🤖

> A full-stack ChatGPT clone with persistent multi-thread conversations, word-by-word typing animation, Markdown + syntax-highlighted code rendering, and OpenAI GPT-4o-mini integration — built with React 19, Vite 7, Express 5, and MongoDB.

[![React](https://img.shields.io/badge/React-19.1-61DAFB?logo=react&logoColor=white)](https://reactjs.org/)
[![Vite](https://img.shields.io/badge/Vite-7.0-646CFF?logo=vite&logoColor=white)](https://vitejs.dev/)
[![Node.js](https://img.shields.io/badge/Node.js-Express_5-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![MongoDB](https://img.shields.io/badge/MongoDB-Mongoose_8-47A248?logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![OpenAI](https://img.shields.io/badge/OpenAI-GPT--4o--mini-412991?logo=openai&logoColor=white)](https://openai.com/)
[![License: ISC](https://img.shields.io/badge/License-ISC-blue.svg)](https://opensource.org/licenses/ISC)

---

## Table of Contents

- [Overview](#overview)
- [Live Demo](#live-demo)
- [Features](#features)
- [System Architecture](#system-architecture)
- [Tech Stack](#tech-stack)
- [Data Model](#data-model)
- [API Reference](#api-reference)
- [Component Breakdown](#component-breakdown)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Key Engineering Decisions](#key-engineering-decisions)
- [Future Improvements](#future-improvements)

---

## Overview

SigmaGPT is a production-style ChatGPT clone that replicates the core UX of OpenAI's chat interface. Users can hold multi-turn conversations with GPT-4o-mini, switch between named conversation threads, delete threads, and watch responses render word-by-word with a typing animation — all backed by persistent storage in MongoDB so conversation history survives page refreshes.

The project is split into a **React 19 + Vite** frontend and an **Express 5 + MongoDB** backend, with a clean REST API bridging the two. Global UI state (active thread, prompt, reply, chat history) is managed via React Context, keeping all components in sync without prop drilling.

---

## Live Demo

> _Deployment link here (e.g., frontend on Vercel, backend on Render)_

---

## Features

### Chat Experience
- **Multi-turn conversations** — Each chat session is a named thread. The first message of a thread becomes its title, just like ChatGPT
- **Persistent history** — All threads and messages are stored in MongoDB; refreshing the page restores the full conversation
- **Word-by-word typing animation** — GPT responses animate in word-by-word (interval: 40ms/word) using `setInterval` on the split reply string, simulating a streaming effect without a true SSE/WebSocket stream
- **Markdown rendering** — Assistant replies are rendered through `react-markdown` with `rehype-highlight`, giving properly formatted headers, lists, bold, and code blocks with GitHub Dark syntax highlighting
- **Enter to send** — Input field listens for `keyDown` Enter events, triggering the same send handler as the submit button
- **Loading spinner** — A `ScaleLoader` from `react-spinners` appears while awaiting the API response

### Sidebar
- **Thread list** — All threads fetched from the backend on mount and whenever the active thread changes, sorted by most recently updated
- **Active thread highlight** — The currently selected thread is visually highlighted with a CSS class
- **New chat** — Creates a fresh UUID v1 thread ID, resets all state, and starts a blank conversation
- **Delete thread** — Sends a `DELETE` request to the API and immediately removes the thread from state (optimistic UI update). If the deleted thread was active, automatically creates a new chat
- **Event bubbling prevention** — The delete icon uses `e.stopPropagation()` to prevent the click from also triggering thread selection

### UI / UX
- Clean two-panel layout: fixed sidebar + scrollable chat window
- Profile dropdown menu with Settings, Upgrade Plan, and Logout options
- Responsive input box pinned to the bottom of the chat window

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Browser (React 19 + Vite)                 │
│                                                              │
│   ┌───────────┐          ┌──────────────────────────────┐   │
│   │  Sidebar  │          │        ChatWindow             │   │
│   │           │          │   ┌──────────────────────┐   │   │
│   │ Thread    │          │   │  Chat (message list) │   │   │
│   │ History   │  Context │   │  Typing animation    │   │   │
│   │ New Chat  │◄────────►│   │  Markdown renderer   │   │   │
│   │ Delete    │  (State) │   └──────────────────────┘   │   │
│   └───────────┘          │   Input Box + Send Button     │   │
│                          └──────────────────────────────┘   │
└───────────────────────────────┬──────────────────────────────┘
                                │ fetch() REST API calls
                                │
┌───────────────────────────────▼──────────────────────────────┐
│              Express 5 + Node.js  (Port 8080)                │
│                                                              │
│   POST /api/chat          →  chatRoutes → OpenAI API         │
│   GET  /api/thread        →  Thread.find().sort({updatedAt}) │
│   GET  /api/thread/:id    →  Thread.findOne() → messages[]   │
│   DELETE /api/thread/:id  →  Thread.findOneAndDelete()       │
└───────────────────────────────┬──────────────────────────────┘
                                │
          ┌─────────────────────┤
          │                     │
┌─────────▼────────┐  ┌─────────▼──────────────────┐
│   MongoDB Atlas  │  │       OpenAI API             │
│  threads coll.   │  │   POST /v1/chat/completions  │
│  (messages[])    │  │   model: gpt-4o-mini         │
└──────────────────┘  └────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | React 19 + Vite 7 | SPA with latest concurrent features |
| Global State | React Context API | Share thread/chat state across components |
| Markdown | react-markdown + rehype-highlight | Render GPT markdown with code syntax highlighting |
| Syntax Theme | highlight.js github-dark | GitHub dark code block styling |
| UUID | uuid v11 (v1) | Time-based unique thread IDs |
| Loading UX | react-spinners (ScaleLoader) | Visual feedback during API calls |
| Backend | Express 5 | REST API server (latest major version) |
| AI | OpenAI SDK + GPT-4o-mini | Language model for chat completions |
| Database | MongoDB + Mongoose 8 | Persistent thread + message storage |
| Config | dotenv | API key and URI management |

---

## Data Model

### Thread (with embedded Messages)

SigmaGPT uses a **single-collection embedded document pattern** — messages are stored directly inside their thread document rather than in a separate collection. This means fetching a thread's full history requires exactly one database query with no joins.

```js
// MessageSchema (embedded)
{
  role:      String (enum: ["user", "assistant"], required),
  content:   String (required),
  timestamp: Date (default: Date.now)
}

// ThreadSchema
{
  threadId:  String (required, unique),   // UUID v1 generated client-side
  title:     String (default: "New Chat"), // Set to first user message on creation
  messages:  [MessageSchema],              // Full conversation history
  createdAt: Date (default: Date.now),
  updatedAt: Date                          // Updated on every new message; used for sidebar sort
}
```

**Why embedded documents?**
A chat thread and its messages are always accessed together — you never need messages without their thread or a thread without its messages. Embedding eliminates the need for a `populate()` call and keeps the read path to a single indexed lookup by `threadId`.

---

## API Reference

Base URL: `http://localhost:8080/api`

| Method | Endpoint | Request Body | Response | Description |
|--------|----------|---|---|---|
| POST | `/chat` | `{ threadId, message }` | `{ reply }` | Send a message; creates thread if new; appends user + assistant messages; returns GPT reply |
| GET | `/thread` | — | `Array<Thread>` | All threads sorted by `updatedAt` descending |
| GET | `/thread/:threadId` | — | `Array<Message>` | All messages for a specific thread |
| DELETE | `/thread/:threadId` | — | `{ success }` | Permanently delete a thread and all its messages |

### Chat Flow — `POST /api/chat`

```
1. Receive { threadId, message }
2. Thread.findOne({ threadId })
   ├── Not found → create new Thread with title = message
   └── Found     → push { role: "user", content: message }
3. Call getOpenAIAPIResponse(message) → GPT-4o-mini
4. Push { role: "assistant", content: reply }
5. thread.updatedAt = new Date()
6. thread.save()
7. res.json({ reply })
```

---

## Component Breakdown

### `App.jsx` — State Root
Owns all shared state and provides it through `MyContext.Provider` to the entire component tree:

```js
// State owned at App level:
prompt        // current input value
reply         // latest GPT response
currThreadId  // UUID of the active conversation
prevChats     // messages[] for the current thread
newChat       // boolean: show "Start a New Chat!" heading
allThreads    // [{ threadId, title }] for sidebar list
```

### `Sidebar.jsx` — Thread Management
- Fetches `GET /api/thread` in a `useEffect` that re-runs whenever `currThreadId` changes, keeping the sidebar in sync after every new conversation is created
- `createNewChat()` generates a fresh UUID v1, resets all state, and sets `newChat: true`
- `changeThread(newThreadId)` fetches the thread's messages from `GET /api/thread/:id` and loads them into `prevChats`
- `deleteThread(threadId)` uses `setAllThreads(prev => prev.filter(...))` for instant optimistic removal before the API call resolves

### `Chat.jsx` — Message Renderer + Typing Animation
The typing animation is the most nuanced component. It distinguishes between two rendering modes:

- **Loading previous thread** (`reply === null`): renders all messages as static `ReactMarkdown`, bypassing the animation entirely
- **New reply arrived** (`reply !== null`): splits the reply string into words, then uses `setInterval` (40ms) to progressively reveal words via `content.slice(0, idx+1).join(" ")`. The cleanup function (`return () => clearInterval(interval)`) prevents memory leaks on unmount or re-render.

All non-latest messages are rendered as static `ReactMarkdown` with `rehype-highlight`. The latest assistant message switches between `key="typing"` and `key="non-typing"` to force React to remount the component when switching modes.

### `ChatWindow.jsx` — Input + API Dispatch
- `getReply()` posts to `POST /api/chat` with the current `prompt` and `currThreadId`
- A `useEffect` on `[reply]` appends both the user message and assistant reply to `prevChats` after the response arrives, keeping the message list consistent without an additional API call
- `setPrompt("")` runs inside the same effect, clearing the input field only after the reply is stored

---

## Project Structure

```
SigmaGPT/
│
├── Backend/
│   ├── server.js              # Express app: CORS, JSON parsing, routes, MongoDB connect
│   ├── package.json
│   │
│   ├── models/
│   │   └── Thread.js          # Mongoose schema: Thread with embedded MessageSchema
│   │
│   ├── routes/
│   │   └── chat.js            # GET/POST /thread, DELETE /thread/:id, POST /chat
│   │
│   └── utils/
│       └── openai.js          # fetch() wrapper for OpenAI /v1/chat/completions
│
└── Frontend/
    ├── index.html
    ├── vite.config.js
    ├── package.json
    └── src/
        ├── main.jsx           # React root
        ├── App.jsx            # State root + MyContext.Provider
        ├── MyContext.jsx      # createContext() — shared state contract
        │
        ├── Sidebar.jsx        # Thread list, new chat, delete thread
        ├── Sidebar.css
        │
        ├── ChatWindow.jsx     # Navbar, Chat embed, input box, API dispatch
        ├── ChatWindow.css
        │
        ├── Chat.jsx           # Message renderer + word-by-word typing animation
        ├── Chat.css
        │
        └── assets/
            └── blacklogo.png  # SigmaGPT logo
```

---

## Getting Started

### Prerequisites

- Node.js ≥ 18
- MongoDB (local or Atlas)
- OpenAI API key with access to `gpt-4o-mini`

### 1. Clone the Repository

```bash
git clone https://github.com/Vishal985057/SigmaGPT.git
cd SigmaGPT/SigmaGPT-main
```

### 2. Start the Backend

```bash
cd Backend
npm install
# Create .env file (see Environment Variables below)
node server.js
# Server runs on http://localhost:8080
```

### 3. Start the Frontend

```bash
cd Frontend
npm install
npm run dev
# App runs on http://localhost:5173
```

---

## Environment Variables

Create a `.env` file inside `Backend/`:

```env
# MongoDB
MONGODB_URI=mongodb+srv://<user>:<password>@cluster.mongodb.net/sigmagpt

# OpenAI
OPENAI_API_KEY=sk-...your-openai-api-key...
```

---

## Key Engineering Decisions

**Why embedded documents for messages instead of a separate collection?**
Chat messages have no independent existence outside their thread — they are always fetched together. Embedding messages in the thread document reduces reads to a single indexed query (`Thread.findOne({ threadId })`), avoids `$lookup`/`populate()` overhead, and keeps write operations atomic (one `thread.save()` persists both the new message and the updated `updatedAt` timestamp together).

**Why UUID v1 (time-based) instead of v4 (random) for thread IDs?**
UUID v1 encodes the generation timestamp in the ID itself. This gives threads a natural chronological sort order by ID without needing a separate `createdAt` index, though the sidebar currently sorts by `updatedAt` for recency. v1 also reduces the (already negligible) collision risk for high-frequency thread creation.

**Why simulate typing word-by-word instead of true streaming?**
True streaming requires Server-Sent Events (SSE) or WebSocket, which adds significant backend complexity — you'd need to pipe the OpenAI stream through Express to the client. The word-splitting `setInterval` approach replicates the visual experience with zero additional infrastructure while still delivering the full reply to the user in a readable, engaging way. Streaming is the natural next upgrade.

**Why use `fetch()` directly in `openai.js` instead of the OpenAI SDK?**
The `openai` npm package is listed as a dependency but the current implementation uses a raw `fetch()` call to `/v1/chat/completions`. This keeps the integration transparent — every request/response parameter is visible in the code — and avoids SDK abstractions that can obscure what's actually being sent to the API. It also makes it easier to swap in a different provider (e.g., Anthropic, Gemini) by changing one fetch call.

**Why `e.stopPropagation()` on the delete icon?**
Each thread `<li>` has an `onClick` to select the thread. The delete icon is nested inside that `<li>`. Without `stopPropagation()`, clicking delete would fire both the delete handler and the thread-selection handler — causing a race condition where the app tries to load a thread that's being deleted.

---

## Future Improvements

- [ ] Replace the word-split animation with true SSE streaming from OpenAI (`stream: true`) for genuine token-by-token output
- [ ] Add user authentication so each user has their own private thread history
- [ ] Implement thread renaming (double-click to edit title inline)
- [ ] Add conversation export (download thread as `.md` or `.txt`)
- [ ] Support image input via GPT-4o's vision capabilities
- [ ] Add system prompt customization (e.g., "You are a coding assistant")
- [ ] Implement search across all threads and messages
- [ ] Write component tests with React Testing Library and Vitest

---

_Built with React 19, Vite 7, Express 5, MongoDB, and OpenAI GPT-4o-mini._
