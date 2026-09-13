# eTax — AI-Assisted Tax Platform

*The digital taxation hub.* An end-to-end AI-powered tax platform designed to
simplify and modernize how taxpayers interact with tax services — secure
identity, conversational access to authorized records, and fraud-risk
assessment, all in one workflow.

[![Watch the eTax demo](https://img.youtube.com/vi/u1ywiZGVsxU/hqdefault.jpg)](https://youtu.be/u1ywiZGVsxU)

▶️ **[Watch the 3:35 demo video](https://youtu.be/u1ywiZGVsxU)** — sign-up, face
verification, fraud-risk assessment, and the chat agent, end to end.

---

## The problem

Filing taxes the traditional way is a hassle: long queues → stacks of
paperwork → hours lost per visit → room for human error. A reliable process
should leave no room for mistakes — but the manual version rarely does.

## The solution

eTax isn't a chatbot bolted onto a database. It's an **intent router with
judgment**: it decides what kind of help a request needs first, and only then
checks who's asking and what they're allowed to see — at every step along the
way, enforced by the backend and the database, not just the UI.

- 🤖 **AI Tax Agent** — built with **LangGraph** for agent orchestration,
  routing every message to the right workflow (fraud assessment, authorized
  data lookup, or a plain answer) and pausing/resuming multi-step flows
  (e.g. a fraud review interrupted mid-way) without losing state.
- 🕵️ **Fraud detection ML model** — an XGBoost model trained on real tax
  data (a meaningfully imbalanced ~10.7% fraud rate) scores every reviewed
  record. Evaluation is **recall-first**: a fraud case slipping through
  undetected is treated as worse than an unnecessary manual review.
- 🔐 **Secure SQL Agent** — the LLM writes SQL, but only against
  per-company **database views**, behind forced **Row-Level Security**,
  scoped **grants**, and **query validation** (a real SQL parse tree, not a
  string check) — so it can generate a query but can never see or return
  data the requesting user isn't authorized for.
- 🪪 **Face recognition with anti-spoofing** — ArcFace embeddings for
  identity, with passive liveness detection so a photo, screen replay, or
  spoof attempt never reaches the matching step.
- 🎙️ **Speech-to-text / text-to-speech**, supporting **Egyptian Arabic and
  English** — including switching languages mid-conversation.
- 🐳 **FastAPI backend**, fully **containerized with Docker** — the whole
  stack (Postgres+pgvector, backend, frontend) comes up with one command.

## How it works

```
Sign up (+ a claimed tax record) → face enrollment (liveness-gated)
   → every login re-verifies your face → chat

Chat message → intent router decides:
   ├─ fraud assessment  → pulls YOUR linked record → XGBoost scores it → risk %
   ├─ database query    → LLM writes SQL scoped to companies you actually own
   │                       → executed under Postgres RLS as an unprivileged role
   └─ everything else    → answered from curated templates, no LLM guesswork
```

**Where AI ends and code begins** (a deliberate design boundary, not an
accident):

| The LLM handles | Code + the database decide |
|---|---|
| Interpreting a message, classifying intent | User permissions |
| Extracting values, phrasing an answer | Missing-value / interrupted-workflow handling |
| Generating scoped SQL | The fraud prediction itself |
| Summarizing results | What data is actually returned |

The LLM never invents a fact: fraud risk always comes from the trained model,
and database answers are grounded only in rows Postgres actually returned.

For the full technical deep-dive (schemas, the RLS/view security design,
provider fallback logic, the LangGraph state machine, etc.), see
**[CLAUDE.md](CLAUDE.md)**.

---

## Try it yourself

### 1. Configure your environment

```bash
cp .env.example .env
```

⚠️ **You must create this `.env` file yourself before running anything** — it
isn't committed (it's git-ignored on purpose, since it holds real secrets).
Open it and fill in at least:

- `JWT_SECRET_KEY` — any long random string
- `COHERE_API_KEY`, `GROQ_API_KEY`, `GEMINI_API_KEY` — needed for the chatbot
  (speech-to-text, LLM calls, text-to-speech). `.env.example` documents how
  to add extra fallback Gemini keys if you have more than one.
- `ELEVENLABS_TTS_KEY` — optional (Gemini/edge-tts cover text-to-speech if
  this is left empty)

### 2. Run it

```bash
docker compose up --build
```

- Frontend: http://localhost:5173
- Backend / Swagger docs: http://localhost:8000/docs
- pgAdmin: http://localhost:5050 (`admin@etax.com` / `admin123`)

The first build takes a while — it bakes the InsightFace (`buffalo_l`) and
MiniFASNetV2 liveness models into the backend image.

### 3. Try face recognition + the fraud model

Every account is linked to one real record from the training dataset via a
one-time **9-digit claim code**, entered at signup — this is what the fraud
model actually scores, and it's never typed or pasted into the chat, only
confirmed. Use this pre-seeded demo code to try it yourself:

```
Claim code: 309295275
```

1. Go to **http://localhost:5173 → Create account** — any name, username,
   email, and password you like — and enter the claim code above as your
   *tax record code*.
2. Enroll your face when prompted (needs camera access — browsers only
   allow this on `localhost` or HTTPS, which is why running it locally on
   `localhost:5173` just works).
3. Log back in — you'll be asked to verify your face against that same
   enrollment before you can chat.
4. In the chat, ask it to assess your fraud risk. It pulls the real record
   behind your claim code and runs the trained model live — no fake data.

A claim code can only be linked to one account. If `309295275` has already
been claimed by an earlier test run, reseed the database (drop the Docker
volumes and `docker compose up --build` again) or use a different unclaimed
code from `tax.fraud_records`.

### Don't want to run it locally?

The **[demo video](https://youtu.be/u1ywiZGVsxU)** above walks through this
exact flow — sign-up, face verification, and a live fraud assessment — end
to end, with nothing to install.

If you'd rather poke at the running app without installing Docker on your
own machine, this should also work in **GitHub Codespaces**: open a
Codespace on this repo, run the same two commands from step 1–2 inside it,
then open the forwarded `5173` port. Codespaces serves forwarded ports over
HTTPS, which satisfies the same camera-access requirement `localhost` does.
This hasn't been verified against the free-tier machine size — the backend
image is sizable (it bakes in the face-recognition models), so a larger
Codespace machine type may be needed.

---

## Project structure

```
backend/                 FastAPI API — auth, face enrollment/verification, chat agent, PostgreSQL+pgvector
frontend/                React (Vite) — the eTax design system + product pages
docs/                    Deep-dive docs: chatbot flow, codebase review, debugging guide, presentation script
ml_artifacts/            Source fraud-model artifacts (dataset, feature importance, encoders) —
                          the copies actually used at runtime live in backend/app/chat/fraud/{models,data}/
system_design_UI.zip     Source design system the frontend was ported from
Presentation.pptx        The slide deck this project was demoed from
docker-compose.yml       db + backend + frontend + pgAdmin
.env.example             Template for every required API key/config — copy to .env and fill in
CLAUDE.md                Full technical architecture reference
```

## Security design (at a glance)

1. **Secure views** — every query runs against a Postgres view created with
   `security_invoker = true`, filtered by the session's own
   `app.current_user_id` — never the raw tables.
2. **Forced Row-Level Security** on `company_owners`, `transactions`, and
   `items` — identity is enforced by the database session itself, not by
   trusting an ID the LLM or the request happened to mention.
3. **Role segregation** — all LLM-generated SQL executes as `app_agent`, an
   unprivileged, non-superuser role with no `BYPASSRLS`, tightly scoped
   `SELECT` grants, and zero access to the authentication schema.

See **[CLAUDE.md](CLAUDE.md)**'s "Ownership-aware SQL security" section for
the complete design, including the planning/authorization pipeline that runs
*before* any SQL is ever generated.

## Documentation

- **[CLAUDE.md](CLAUDE.md)** — full architecture reference (schemas, the
  LangGraph agent, provider fallback, the security pipeline)
- **[docs/CHATBOT_DETAILED_FLOW.md](docs/CHATBOT_DETAILED_FLOW.md)** — the
  chatbot's turn-by-turn flow
- **[docs/CODEBASE_REVIEW.md](docs/CODEBASE_REVIEW.md)** /
  **[docs/DEBUG_GUIDE.md](docs/DEBUG_GUIDE.md)** — codebase walkthrough and
  troubleshooting notes
- **[docs/PRESENTATION_SCRIPT.md](docs/PRESENTATION_SCRIPT.md)** /
  **Presentation.pptx** — the narrated deck this project was demoed from
