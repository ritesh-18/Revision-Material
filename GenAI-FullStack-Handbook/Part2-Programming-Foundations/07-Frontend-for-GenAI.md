# Chapter 7 — Frontend for GenAI

## 7.1 Concept Explanation

The frontend is where GenAI actually meets users, and the design decisions made here determine whether the product feels magical or broken. This chapter teaches enough frontend to build AI-first interfaces. We deliberately skip framework internals — your job is not to write a virtual DOM, it is to ship a streaming, responsive, accessible chat that handles failure gracefully.

The defining property of an AI frontend: latency is variable and partial responses are normal. A traditional form submits and renders an answer; an AI frontend renders answer fragments as they arrive, must handle mid-stream cancellation, must show intermediate states (tool calls, citations, plan steps), and must recover from network blips without losing context.

## 7.2 React Basics — Just Enough

React is the dominant view library for AI apps. The essentials:
- Components are functions that return UI based on props (inputs) and state.
- State changes trigger re-renders.
- Effects let you synchronize with external systems (network, timers, subscriptions).
- Refs hold mutable values that should not trigger re-renders (DOM handles, abort controllers).

For AI UIs, you spend most of your time managing streaming state: an array of messages, the current in-flight message being built up, and abort handles for cancellation. The rest is layout.

## 7.3 Next.js Basics

Next.js is React plus server-side rendering, routing, and an integrated backend (API routes, server actions). For AI products, Next.js shines because it lets you co-locate the streaming backend handler and the chat UI in one codebase and one deployment.

Two architectural choices matter:
- **Server components vs client components.** Server components run on the server and stream rendered HTML. Client components run in the browser and handle interactivity. AI chat UIs are almost entirely client components because they need stateful streaming behavior.
- **Edge runtime vs Node runtime.** Edge functions run on a global CDN, cold-start fast, but have fewer Node APIs. Useful for thin LLM proxies. Node runtime is needed for heavier work.

## 7.4 Streaming UI — The Heart of Chat UX

A chat UI must render tokens as they arrive. Three implementation layers:

1. **Network layer.** Open an SSE or fetch-with-streaming connection. Read chunks as they arrive.
2. **State layer.** Append each chunk to the current message's text in state.
3. **Render layer.** React re-renders the message component on each state update.

The trap: re-rendering the entire message list on every token is expensive once messages get long or numerous. The mitigation: render the streaming message as its own component that owns its text state, so the parent list does not re-render.

A second trap: scrolling. As text grows, the user wants the view pinned to the bottom unless they have scrolled up to read history. Implement "stick to bottom unless user scrolled" — a small piece of logic that, done wrong, makes the UI feel like it is fighting the user.

## 7.5 AI Chat Interface Patterns

The conversation list. Each message is a turn (user or assistant). The assistant turn may contain text, tool calls, citations, code blocks, images. Render each part appropriately.

The composer. A text area with send button, file attachments, voice input, model selector, optional system prompt control. Keyboard shortcut for send (typically Cmd/Ctrl+Enter). Disable send while a response is streaming, or implement "stop generating."

The action affordances. Copy message, regenerate, rate response, fork conversation, share. The patterns are converging across products — copy what works.

Empty state and onboarding. First-time users do not know what to type. Suggest prompts. Show example flows.

## 7.6 Markdown Rendering

LLMs love markdown. Headings, lists, tables, code blocks, links, bold, italic. Render it.

Use a markdown library (react-markdown plus remark plugins is the standard). Add syntax highlighting for code blocks. Render LaTeX for math if your domain needs it. Sanitize HTML aggressively — LLM output can contain malicious tags if not filtered, especially in agent flows where the model might be quoting user input.

Streaming markdown is tricky. A partially streamed code block may not yet have its closing fence, and the renderer may panic on the incomplete syntax. Defensive rendering: try to parse, fall back to plain text on failure, re-parse on each new chunk.

## 7.7 Token Streaming UX Details

- **Time-to-first-token (TTFT).** The most important perceived-latency metric. Aim for under one second; above two seconds users start tabbing away.
- **Smooth flow.** Tokens arriving in bursts feel choppy. A small animation delay (10–30ms) that smooths bursts can improve perceived quality.
- **Cancellation.** A stop button that aborts the request immediately, frees server resources, and locks in whatever was generated.
- **Resumability.** Network blip mid-stream — can the UI resume? Most apps do not bother, but enterprise apps with long completions should.

## 7.8 File Uploads

AI products increasingly accept files (PDFs, images, spreadsheets). The pipeline:
1. User selects file.
2. Client uploads to backend or pre-signed cloud storage URL.
3. Backend extracts text, embeddings, or runs vision on it.
4. UI shows progress and any extraction preview.
5. The file becomes part of conversation context.

Pre-signed URLs (S3, GCS) are the standard for any file over a few megabytes — they offload bandwidth from your backend. Show progress bars; large uploads on flaky connections will fail and users need to know.

## 7.9 Voice Interfaces

Voice introduces three pipelines:
- **Speech-to-text (STT).** Browser MediaRecorder API or a streaming STT service like Deepgram, Whisper, or AssemblyAI.
- **LLM** processes the transcribed text.
- **Text-to-speech (TTS).** Streaming TTS like ElevenLabs, OpenAI, or Cartesia plays audio as it generates.

For natural conversation, you need to handle interruption: the user starts speaking while the assistant is still talking. Detect voice activity, pause TTS, transcribe the new input. The whole loop must run under 300ms latency to feel real. WebRTC, WebSockets, and careful buffer management are required. This is genuinely hard engineering.

## 7.10 Realtime UX Patterns

- **Optimistic UI.** When the user sends a message, render it instantly before the server confirms.
- **Skeleton loaders.** While retrieval runs, show a placeholder so the UI does not appear frozen.
- **Tool call indicators.** When an agent is calling a tool, show what tool and what input — transparency builds trust.
- **Citation rendering.** Show source documents inline, with hover or click to expand.
- **Conversation forking.** Let users branch off any message to try a different path without losing the original.

## 7.11 Production Architecture

```
   Browser
     |
   Next.js (SSR + client) or React SPA + separate API
     |
     |--- streaming fetch / SSE / WebSocket ---|
     v                                          v
   Edge cache (static assets)         Backend (Chapter 6)
```

State management: for small apps, React's built-in `useState` and `useReducer` are sufficient. For larger apps, Zustand or Redux Toolkit. Avoid global state for streaming text — keep it scoped to the message component to avoid wholesale re-renders.

## 7.12 Tradeoffs

- **Next.js vs separate SPA + backend.** Next.js consolidates deployment and SSR. A separate React SPA + backend is cleaner architecturally but doubles the deployment surface.
- **SSE vs WebSocket.** SSE is simpler for one-way streaming. WebSocket is required for bidirectional flows like voice.
- **Build for "stop generating" vs not.** Stop generating costs effort but users expect it; skipping it makes the product feel cheap.

## 7.13 Scaling Challenges

Frontend scaling is mostly about CDN coverage and bundle size. Keep the bundle small; defer rarely-used components (file viewers, voice mode) behind dynamic imports. Use a CDN globally. For SSR-heavy AI apps, watch for slow page loads when the backend is also under load — the page itself can stall on a database query.

## 7.14 Security Concerns

- Sanitize all model output before rendering as HTML.
- Beware of indirect prompt injection through document uploads — content displayed to the user may carry hidden instructions.
- Cross-origin policies for API calls — typically same-origin for chat, with strict CORS for any external embedding.
- Auth tokens in browser storage — prefer HttpOnly cookies over localStorage to limit XSS impact.

## 7.15 Deployment Guide

Static assets to a CDN (Vercel, Cloudflare Pages, Netlify, S3 + CloudFront). API routes either on the same platform (Vercel functions) or behind your own backend. Set up environment-specific previews so PRs can be reviewed visually. Roll forward with feature flags so risky AI changes can be turned off without redeploy.

## 7.16 Monitoring Strategy

Real-user monitoring (RUM) is critical. Track TTFB, TTFT, total response time, message render time, error rates, and user actions (stop, regenerate, rate). Tools: Sentry for errors, PostHog or Mixpanel for product analytics, OpenTelemetry web SDKs for tracing into the backend.

## 7.17 Cost Optimization

Frontend cost is mostly CDN bandwidth. Compress assets. Use modern image formats. Lazy-load heavy components. The big AI-specific lever: client-side caching of past conversations so the user does not refetch them on each visit.

## 7.18 Interview Questions

- How do you implement streaming chat without re-rendering the whole list on every token?
- Describe the "stick to bottom" scroll behavior.
- How do you handle network drop mid-stream?
- Walk through markdown rendering during streaming.
- What changes when you add voice mode?

## 7.19 Hands-on Exercises

1. Sketch the React component tree for a chat UI: list, message, composer, sidebar.
2. Design a state shape that supports messages, in-flight streaming text, and abort controllers.
3. Plan the voice interruption flow on paper — what events fire, what state changes, where buffers live.

## 7.20 Common Mistakes

- Re-rendering the full message list on every token.
- Forgetting to clean up streams on component unmount.
- Rendering raw model markdown without sanitization.
- No stop button. No regenerate button. No copy button.
- Letting the composer's auto-focus steal focus while the user is reading.

## 7.21 Enterprise Best Practices

Build a single chat-UI component library and reuse it across products. Standardize on a markdown renderer with shared sanitization rules. Build instrumentation hooks into every action so product analytics work uniformly. Plan accessibility from the start (keyboard navigation, screen readers, contrast). Localize from day one if your audience is international.
