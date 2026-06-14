# AI Voice Sales Agent · "Катя"

> **Case study** · Real-time in-browser AI voice sales agent · custom STT→LLM→TTS pipeline over WebSocket · lead dev in a small team

[![Status](https://img.shields.io/badge/status-MVP%2Fbeta-eab308)]()
[![Role](https://img.shields.io/badge/role-lead%20dev-blue)]()
[![Stack](https://img.shields.io/badge/stack-FastAPI%20%2B%20React%2019-009688?logo=fastapi&logoColor=white)]()
[![AI](https://img.shields.io/badge/AI-Deepgram%20%2B%203%20LLMs%20%2B%20Fish.audio-1f6feb)]()

**Code:** private — access on request

---

## TL;DR

A web funnel that turns Instagram-reel leads into qualified sales calls **with no human in the loop**: visitor lands in a chat, AI extracts business data in ~4 minutes, then a **live demo call from "Катя" (the AI voice agent)** plays in the browser via a custom **STT → LLM → TTS WebSocket pipeline**. Visitor hangs up, sees an offer, pays 10 000 ₽ to provision their own agent. Lead dev on a small team — designed the voice pipeline, multi-model LLM routing, and most of the latency-optimization work.

## Why this exists

Voice AI sales agents at vendor level (Vapi, Bland, Retell) ship great demos but lock you into their pricing, models, and prosody. The client wanted a **white-label voice agent business** — sell prebuilt sales agents to small businesses on a flat fee — and that meant the *demo* on the funnel had to feel as good as a vendor demo *without* the vendor SDK in the critical path. So I rebuilt the voice pipeline from scratch over a single WebSocket, with model selection per use-case and explicit latency budgets.

## What I built

### Real-time voice pipeline (no SDK)

- **Single WebSocket** `/ws/voice` carries mic frames up, typed JSON + binary PCM down
- **STT — Deepgram nova-3** streaming, RU, LINEAR16 @ 16 kHz, `interim_results=True`, `endpointing=300ms`, `utterance_end_ms=1000ms`. Generator yields `{text, final, confidence}` and closes on `speech_final` or `UtteranceEnd`
- **LLM — OpenRouter** with **three models routed per use-case** via `config.py`:

| Use case | Model | Why |
|---|---|---|
| Chat funnel | `google/gemini-2.5-flash` | 50+ turns, cheap, fast enough |
| In-call turns | `google/gemini-3.1-flash-lite` (was Claude Haiku 4.5) | **TTFT ~0.5 s** vs Haiku's ~1.2 s — voice is TTFT-bound |
| Niche briefing | `anthropic/claude-sonnet-4-5` | One-shot, breadth + factual; cached to disk |

- **TTS — Fish.audio s2-pro** WebSocket, raw PCM 16 kHz 16-bit mono; **emotion tags** (`[дружелюбно]`, `[воодушевлённо]`) auto-prepended for sales prosody; `temperature=0.5`, `prosody.speed=0.95`
- **Streaming end-to-end** — LLM tokens fan straight through `synthesize_streaming_from_iter` into TTS without phrase buffering
- **Barge-in** — frontend `RMS_THRESHOLD=0.015` over 3 frames triggers stop-speaking + STT swap mid-utterance

### Latency engineering

Production-style numbers from the live pipeline:

- **STT final → first PCM ≈ 1.0–1.3 s** (target 800 ms — headroom for pre-warm)
- **Parallelized Fish.audio handshake || LLM call** — opened TTS WS in a background task while LLM draining started, saved **~547 ms per turn** (1851 ms → 1304 ms)
- **Direct LLM→TTS token fan-through** — removed phrase buffer, AI turn 7.5 s → 5.7 s
- **Persistent briefing cache** — `backend/.briefing-cache.json` survives restarts; second call on the same niche is instant
- **TTS streaming start in parallel with LLM TTFT** — Fish WS ~580 ms handshake hides inside Gemini's TTFT

### Funnel + state machine

- `chat → extracting → calling → offer` (Zustand `useFunnelStore`)
- Handoff trigger is literal — the chat LLM ends with `"Звоню?"`, frontend looks for that substring to switch screens
- `chat_sessions` in Supabase, single table, JSONB messages, joined by client-side `session_token` UUID
- Streaming chat via `fetch` + `ReadableStream` (POST), Supabase persistence in FastAPI `BackgroundTask` so the user-facing stream isn't blocked

### Migrations done in-flight

- **Vapi → custom pipeline** — removed `@vapi-ai/web`, owned the WS contract
- **Google STT → Deepgram nova-3** — same one-utterance-per-call shape preserved 1:1, router untouched
- **Azure TTS → Fish.audio s2-pro** — better RU prosody + lower cost + barge-in actually works
- **Single LLM → 3-model routing** — chat / call / briefing each get the right trade-off
- **Haiku-4.5 → Gemini 3.1 Flash-lite** for in-call — TTFT halved at the cost of slight QA drop, the right call for voice

## Outcome

| | |
|---|---|
| Status | MVP/beta — funnel + voice demo live for internal use |
| Role | Lead dev in a small team — owned voice pipeline, multi-model routing, latency work |
| Pipeline | Vendor-SDK-free; ~$X per demo call vs vendor-fee equivalent |
| Compliance | Explicit consent gate for cross-border PII (152-ФЗ §12 ч.4) — bundled into chat onboarding |

## Stack

### Frontend
- React 19 + TypeScript + Vite 8
- Tailwind CSS v4 (config in CSS, no `tailwind.config.js`)
- Framer Motion (cubic-bezier `[0.32, 0.72, 0, 1]` as the project's standard ease)
- Zustand state, `react-router-dom` v7
- Native `WebSocket` — no voice SDK
- `@supabase/supabase-js` direct from frontend (anon key, RLS-guarded)
- `geist` font, `@phosphor-icons/react`

### Backend
- FastAPI + uvicorn, `httpx` for outbound
- `supabase` Python client (lazy-init, no-op when env unset — so the backend boots without Supabase configured)
- Streaming response wraps `BackgroundTask` for post-stream persistence
- Pydantic threat-budget — messages capped at 50/req, 2000 chars each (threat mitigation T-01-02)

### Vendors
- **STT** — Deepgram (`nova-3`, ru)
- **LLM** — OpenRouter (3 models, swap by env var)
- **TTS** — Fish.audio (`s2-pro`, voice ID from dashboard)
- **DB** — Supabase
- **Hosting** — Vercel (frontend), VPS (backend)
- **Notifications** — Telegram bot to operator

## Architecture

```
Browser
  │   – chat (streaming POST + ReadableStream)
  │   – mic (AudioWorklet, 16 kHz PCM, 64 ms frames)
  ▼
FastAPI
  ├── /api/chat/message       ──► OpenRouter (Gemini stream)
  │                            └─► BackgroundTask → Supabase
  ├── /api/chat/extract       ──► OpenRouter (JSON mode) → CompanyData
  ├── /api/generate-prompt    ──► OpenRouter → voice prompt + first message
  └── WS /ws/voice
        │
        ├── Deepgram STT (nova-3, ru, 16 kHz)
        │       │  speech_final / UtteranceEnd
        │       ▼
        ├── OpenRouter (CALL_LLM_MODEL, token stream)
        │       │
        │       ▼  (direct fan-through)
        ├── Fish.audio TTS (s2-pro, PCM 16 kHz streaming)
        │       │  with [дружелюбно] / [воодушевлённо] emotion tags
        │       ▼
        └── PCM binary frames → StreamingAudioPlayer (AudioBufferSourceNode, gapless)
  │
  ▼
Supabase  (chat_sessions: session_token UNIQUE, messages JSONB, state)
```

## What I learned

- **Voice is TTFT-bound, not throughput-bound** — once you're streaming, the listener tolerates moderate token rate but hates the initial silence. Picking Gemini 3.1 Flash-lite over Claude Haiku 4.5 for in-call turns halved TTFT for a barely-perceptible QA drop. The right trade for a sales demo
- **Multi-model routing > "pick the best model"** — chat / call / briefing have orthogonal constraints. Splitting them via three env vars cost nothing and made every model swap a one-line change
- **Parallelize what doesn't depend** — Fish WS handshake ran serially after LLM TTFT until I drained the LLM into a queue from a background task and opened TTS in parallel. ~547 ms / turn for almost no code
- **Custom voice pipeline beats vendor SDKs *once* you accept WebSocket plumbing** — Vapi/Bland get you demoing in an afternoon but make you allergic to vendor fees as you scale. Going custom paid off after the first real client conversation about pricing
- **Briefing-cache pattern generalizes** — a "one-shot, expensive, breadth-required" call (niche briefing) is the right place for a hard-coded persistent cache. We pay once per niche, ever
- **Emotion tags are a real prosody control surface** — pre-prepending `[дружелюбно]` to every Fish.audio s2-pro turn stabilized sales-tone delivery; per-phrase tags from the LLM let the model vary mood ("[спокойно]", "[воодушевлённо]") within a call
- **Streaming + barge-in changes the unit of work** — you can't pre-compose audio; everything is realtime. RMS-threshold barge-in detection on the client + graceful TTS cancellation on the server are what makes the call feel alive vs robotic

## Code access

Source code is in a private repository. Happy to share read access with hiring teams on request — ping me on **[Telegram @santicc712](https://t.me/santicc712)**.

---

<details>
<summary><b>🇷🇺 По-русски</b></summary>

<br/>

**Case study** · In-browser AI voice sales agent в реальном времени · кастомный STT→LLM→TTS пайплайн через WebSocket · lead dev в маленькой команде

### TL;DR

Веб-воронка превращает лидов с Instagram-рилсов в квалифицированные продажные звонки **без человека в петле**: посетитель попадает в чат, AI вытягивает данные о бизнесе за ~4 минуты, затем **живой демо-звонок от "Кати" (AI-голосового агента)** играет в браузере через кастомный **STT → LLM → TTS WebSocket пайплайн**. Бросает трубку, видит оффер, платит 10 000 ₽ за свой агент. Lead dev в маленькой команде — спроектировал голосовой пайплайн, multi-model LLM-роутинг и большую часть работы по латентности.

### Зачем

Vendor voice-AI агенты (Vapi, Bland, Retell) дают красивые демо, но лочат в свои цены, модели и просодию. Заказчик хотел **white-label voice-agent бизнес** — продавать преднастроенных продажных агентов малому бизнесу — и для этого *демо* в воронке должно ощущаться как vendor-демо *без* vendor SDK в критическом пути. Поэтому я переписал голосовой пайплайн с нуля через один WebSocket, с выбором модели под задачу и явными бюджетами латентности.

### Что сделал

#### Real-time голосовой пайплайн (без SDK)

- **Один WebSocket** `/ws/voice` тащит вверх кадры микрофона, обратно типизированный JSON + binary PCM
- **STT — Deepgram nova-3** streaming, RU, LINEAR16 @ 16 kHz
- **LLM — OpenRouter с тремя моделями под use-case:**

| Use-case | Модель | Почему |
|---|---|---|
| Chat-воронка | `google/gemini-2.5-flash` | 50+ ходов, дёшево, быстро |
| In-call turns | `google/gemini-3.1-flash-lite` | **TTFT ~0.5 с** vs Haiku ~1.2 с — voice TTFT-bound |
| Briefing | `anthropic/claude-sonnet-4-5` | One-shot, breadth + factual, кеш на диск |

- **TTS — Fish.audio s2-pro** WebSocket, raw PCM 16 kHz mono; **emotion-теги** (`[дружелюбно]`, `[воодушевлённо]`)
- **Streaming end-to-end** — LLM-токены идут напрямую в TTS без phrase buffer
- **Barge-in** — RMS-threshold на фронте → stop-speaking + STT swap

#### Инжиниринг латентности

- **STT final → first PCM ≈ 1.0–1.3 с** (цель 800 мс)
- **Параллелизация Fish handshake || LLM** — сэкономила **~547 мс на ход** (1851 → 1304)
- **Direct LLM→TTS token fan-through** — AI ход 7.5 с → 5.7 с
- **Persistent briefing cache** — второй звонок на ту же нишу мгновенный

#### Воронка + state machine

- `chat → extracting → calling → offer` через Zustand
- Хендофф-триггер буквальный: чат-LLM заканчивает фразой `"Звоню?"`
- Single table `chat_sessions` в Supabase, JSONB messages

#### Миграции в полёте

- **Vapi → custom pipeline** — `@vapi-ai/web` выкинут
- **Google STT → Deepgram nova-3** — контракт 1:1 сохранён
- **Azure TTS → Fish.audio s2-pro** — лучше RU-просодия + дешевле + barge-in работает
- **Один LLM → 3-модели routing** — chat/call/briefing получили правильный trade-off
- **Haiku-4.5 → Gemini 3.1 Flash-lite** в звонке — TTFT вдвое, минимальное падение QA

### Результат

- MVP/beta — воронка + голосовой демо живут для внутреннего использования
- Lead dev — voice pipeline, multi-model routing, latency
- Pipeline без vendor SDK
- 152-ФЗ ч.4 §12 compliance — явный consent gate в onboarding чата

### Стек

**Frontend:** React 19 + TS + Vite 8, Tailwind v4, Framer Motion, Zustand, native WebSocket (без SDK), Supabase JS, geist.

**Backend:** FastAPI + uvicorn, httpx, Supabase Python (lazy), streaming responses + BackgroundTask, Pydantic threat-budget.

**Vendors:** Deepgram (STT, nova-3), OpenRouter (3 модели), Fish.audio (TTS, s2-pro), Supabase, Vercel + VPS, Telegram бот для оператора.

### Что вынес

- **Voice TTFT-bound, не throughput-bound** — слушатель терпит средний rate, но не терпит начальной тишины. Гемини flash-lite vs Haiku 4.5 в звонке — правильный трейд для продажного демо
- **Multi-model routing > «выбрать лучшую модель»** — у chat / call / briefing разные ограничения. Три env-vars — и каждая замена модели в одну строку
- **Параллелизуй то, что не зависит** — Fish WS handshake висел в очереди за LLM TTFT, пока не вынес LLM-drain в background task. ~547 мс за бесплатно
- **Custom voice pipeline бьёт vendor SDK** *после* того как примешь WebSocket-plumbing. Vapi/Bland дают демо за вечер, но при росте пользователей делают аллергию на vendor fees
- **Briefing-cache паттерн обобщается** — «one-shot, дорого, нужна широта» — это место для жёсткого persistent-кеша
- **Emotion-теги — реальный контрольный рычаг просодии** — auto-prefix `[дружелюбно]` стабилизирует sales-тон; per-phrase теги от LLM меняют настроение в звонке
- **Streaming + barge-in меняют единицу работы** — нельзя предсобрать аудио, всё реалтайм. RMS-barge-in на клиенте + graceful TTS cancel на сервере — то что отличает живой звонок от робота

### Доступ к коду

Исходники в приватном репо. Готов дать read-доступ командам найма — пингуй в **[Telegram @santicc712](https://t.me/santicc712)**.

</details>
