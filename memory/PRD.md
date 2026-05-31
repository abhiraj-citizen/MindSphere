# MindSphere — Product Requirements Doc (Working)

## Original Problem Statement (2026-05-30)
> "So like there is a simple bug in this app, when you login and stuff, and go on any dashboard anything, you have to scroll down to see the main thing, there is like a big gap.
> Second, take the Fix It Gemini's Voice mode backend and everything except the ai prompt, and ui, and just change the MindSphere's Voice mode with Fix it Gemini's."

## Architecture
- Backend: FastAPI + MongoDB (Motor) + Emergent LLM (GPT-4o) + Gemini Live (google-genai)
- Frontend: React (CRACO) + Tailwind + framer-motion + lucide-react
- Voice: WebSocket `/api/voice/ws` relaying to Gemini Live `gemini-3.1-flash-live-preview`
- Auth: JWT bearer, demo seed user

## User Personas
- Mental wellness seekers wanting journaling, mood tracking, AI companion (Lyra), real-time voice conversations, assessments, sleep, diet, meditation

## Session History
### 2026-05-31 — Feature: Interactive First-Run Tutorial
- **Added: Step-by-step tutorial for newly onboarded users**
  - Backend: added `tutorial_completed` flag to user model (default `False` on signup, `True` for demo user); endpoints `POST /api/users/tutorial-complete` and `POST /api/users/tutorial-reset`.
  - Frontend: new `TutorialOverlay.jsx` — lightweight custom guided tour (no external library). Spotlights real sidebar nav items using `data-testid` selectors, with a 4-panel dimmer + glowing ring around the target and a glass tooltip card showing title/body/Back/Next/Skip + animated progress dots. Supports keyboard nav (Esc/←/→/Enter).
  - 9 steps: Welcome → Dashboard → Mind Journal → Mood Tracker → Meet Lyra → Real-Time Voice → Mental Health Tools → Analytics & Insights → You're ready.
  - Auto-launches in `Dashboard.jsx` when `user.onboarded && user.tutorial_completed === false`. Uses a `useRef` guard to be StrictMode-safe (a `setTimeout` without a cleanup cancel).
  - Added a **Replay the tour** card in `Settings.jsx` — calls `/users/tutorial-reset`, refreshes auth, navigates to dashboard so the auto-trigger fires.
  - Verified end-to-end via Playwright: tour opens automatically post-onboarding, all 9 steps render in order, Back/Next/Skip work, server marks `tutorial_completed=true`, reload does NOT re-trigger, replay from Settings works.

### 2026-05-30 — Bug Fixes
- **Fixed: Dashboard "big gap" bug (P0)**
  - Root cause: `.glass { position: relative; }` in `index.css` overrode the Tailwind `fixed` class on `<Sidebar>` `<aside class="glass fixed ...">`. The sidebar rendered relatively in flow at 851px height, pushing `<main>` down by ~851px on every authenticated page.
  - Fix: Removed `position: relative` from `.glass` selector. Sidebar now correctly `position: fixed`; `mainTop: 0` on all pages (Dashboard, Journal, Mood, Exercise, Sleep, Voice verified).
- **Replaced: MindSphere Voice Mode backend + frontend with Fix-It Gemini's implementation (P0)**
  - Backend: Rewrote `@app.websocket("/api/voice/ws")` to use Fix-It Gemini's proven Live API relay pattern (single `stop_event`, dual pump tasks, `audio_in`/`audio_out`/`transcript`/`status`/`stop` message types, `gemini-3.1-flash-live-preview` model).
  - Preserved MindSphere's per-user mental-wellness system prompt via `build_voice_system_prompt(user_id)` — referencing journal/mood/onboarding context. (User explicitly asked to keep AI prompt.)
  - Frontend: Rewrote `Voice.jsx` with Fix-It Gemini's audio pipeline (single AudioContext, `/pcm-processor.js` worklet, scheduled BufferSource playback at 24kHz, exponential backoff reconnect, `audio_in`/`audio_out` protocol). Preserved MindSphere's UI styling — `AppShell`, `PageHeader`, mind-orb, glass cards, waveform, live captions, accent #10b981.
  - Added `/app/frontend/public/pcm-processor.js`.
  - Smoke-tested: WebSocket connects with `status: connected` and Gemini Live returns `audio_out` chunks for text prompts.

## Implementation Status
- [x] Dashboard scroll/gap bug fixed (all authenticated pages)
- [x] Voice Mode backend (Fix-It pattern, MindSphere prompt) wired
- [x] Voice Mode frontend (Fix-It audio pipeline, MindSphere UI) wired
- [x] `gemini-3.1-flash-live-preview` confirmed working via WS smoke test

## Known Things to Watch
- Voice mode requires the GEMINI_API_KEY env var (configured).
- Voice mode uses the Fix-It Gemini pattern WITHOUT explicit input/output transcription config; transcripts will only appear if Gemini returns `response.text` (model dependent).

## Next Action Items
- (Optional) Add input/output `audio_transcription` config back to enable live captions for both user and Lyra if user requests.
- (Optional) Add interrupt/barge-in signaling on the wire (currently UI-only flush).
