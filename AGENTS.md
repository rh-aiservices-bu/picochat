# Repository Guidelines

## Project Overview
picochat is a deliberately minimal, single-file web app: paste a base URL, API key, model id, context window size, and max tokens, then chat with any OpenAI-compatible `/chat/completions` endpoint. No build, no dependencies, no framework — the entire app is one static `index.html` with inline CSS and JS.

## Architecture & Data Flow
Static HTML page, no backend of its own. All logic lives in one inline `<script>` in `index.html`:

1. **Config** — 5 form fields (Base URL, API Key, Selected Model, Model context window, Max Tokens). Every keystroke persists all five to `localStorage` under key `picochat.cfg.v1`; fields rehydrate on load. The API key never leaves the browser (sent only as `Authorization: Bearer` to the user-chosen base URL).
2. **Send** — on submit, the new user message is appended to `msgs` first, then the full history is passed through `fit()` (so the current prompt is always in the request), then POSTed to `{base}/chat/completions` with `stream: true`, `max_tokens` (only when set), and Bearer auth.
3. **Stream** — `streamSSE()` reads the `res.body` ReadableStream, splits SSE lines, parses each `data:` JSON, and appends `choices[0].delta.content` deltas to a live DOM bubble until `data: [DONE]`.
4. **Render** — messages are DOM divs (`.msg.user` / `.msg.assistant` / `.msg.error`) appended to `#chat`; Reset clears the DOM and `msgs` (config is untouched).

Key behavior to preserve when editing:

- **Stateless API contract** — the server is never assumed to keep state; every request resends the entire message history.
- **Context-window fitting** — `fit()` uses a heuristic token estimate (`4 + ceil(chars/4)` per message), reserves `max_tokens` of budget, and drops the *oldest* messages first, always keeping at least the newest. If the context-window field is empty, no truncation happens.
- **Error path** — non-2xx, network, or malformed-stream failures show a multi-line `.msg.error` bubble: the error message plus a diagnostics block (request URL, model, history vs. sent counts, context window, max_tokens, key prefix, and status/content-type when a response arrived; a network/CORS hint for `TypeError: Failed to fetch`). Every request and failure is also logged to the console (`[picochat]` prefix, with `err.stack`). The full API key is never shown — only `first8…last4` (or `(short key)` if ≤12 chars). The user message stays in `msgs` so a retry re-sends it, and the failed assistant bubble is removed.
- **Base URL convention** — the app appends exactly `/chat/completions` (trailing slashes stripped). Users paste the base *including* `/v1` (e.g. `https://api.openai.com/v1`).

## Key Directories
Flat repo, no subdirectories:

- `index.html` — the entire application (markup, styles, logic).

## Development Commands
There is no build system.

```bash
# Serve and open (any static server works):
python3 -m http.server 8080
# → http://localhost:8080/

# Or just open the file:
open index.html   # macOS; file:// works fine, no fetch cross-origin involved
```

## Code Conventions & Common Patterns
- **Single file** — keep the app in `index.html`; do not introduce modules, bundlers, or dependencies. The file must remain usable by double-clicking.
- **Vanilla JS, no framework** — element access via the `$` id shorthand (`$('chat')`), DOM built with `createElement`, no innerHTML with user data.
- **Config object pattern** — `cfgEls` maps setting keys to inputs; adding a new setting means one entry in `cfgEls` plus matching `<label>` markup. Persistence is automatic for anything in `cfgEls`.
- **Async** — `async/await` only; streaming via `fetch` + `ReadableStream.getReader()` (not `EventSource`, because the request is a POST).
- **Error handling** — one `try/catch` around the request in the submit handler; collect request diagnostics in a local `dbg` object (no secrets), surface a human-readable multi-line message in an error bubble, and mirror it to `console.error` with the stack. Never show the full API key anywhere.
- **CSS** — minimal utility-ish classes, system font stack, `100dvh` flex column layout (config grid → scrollable chat → input form). Mobile: config collapses to one column under 700px.
- **Token math** — if you change `estTok`/`fit`, keep the invariant: never drop the newest message, and reserve the `max_tokens` budget before fitting history.

## Important Files
- `index.html` — entry point and only source file. Key internals: `cfgEls` (settings), `msgs` (chat state), `fit()` (context truncation), `streamSSE()` (SSE parser), the `#send` submit handler (request pipeline).

## Runtime/Tooling Preferences
- **No runtime required** — pure static file; works from `file://` or any static server.
- **No package manager** — there is no `package.json`; do not create one.
- **Any OpenAI-compatible API** — OpenAI, proxies, local models (e.g. llama.cpp/Ollama with `/v1`); the page works against whichever base URL the user enters.

## Testing & QA
No test framework and no test files — this is a one-file app verified manually.

- **Manual smoke test** — serve the page, point it at any live endpoint, send a prompt, confirm streaming, multi-turn history, and that config survives a page reload.
- **Mocking an endpoint** — to test offline, run any tiny server that serves `index.html` at `/` and returns SSE chunks at `POST /v1/chat/completions`:

  ```
  data: {"choices":[{"delta":{"content":"hello "}}]}

  data: {"choices":[{"delta":{"content":"world"}}]}

  data: [DONE]
  ```

  (A throwaway Node mock was used during development and deleted; recreate ad hoc when needed. It caches `index.html` at startup — **restart it after editing the app**, or it serves a stale copy.)
- **Behaviors worth checking after any change** — streaming renders incrementally; context-window truncation drops oldest messages first (set a tiny window and send long messages repeatedly); bad credentials show an error bubble and the next send retries the queued message; Reset clears chat only, not config; localStorage round-trips all five settings.
