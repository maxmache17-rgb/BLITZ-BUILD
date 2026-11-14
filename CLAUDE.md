# KLEER — AI Cup 2026, Track A (Productivity & Work)

Two AI agents sit between a manager and an intern. The first turns a rough
request into a clear mission flashcard. The second answers the intern's
questions from the brief and passes on to the manager only what is new.

One-liner: Clear briefs for interns, fewer interruptions for managers.

## Today (Sat 14 Nov 2026, all times GMT)

| Time  | Goal |
|-------|------|
| 12:00 | Core loop works end to end on the published URL |
| 12:30 | If it does not, matching and the to-do list become static |
| 14:00 | Feature freeze, record the backup demo video |
| 15:00 | Submit on Commfy (hard deadline 15:30) |
| 16:00 | Shortlist judging: 2-min pitch and demo, 1-min Q&A |

Two things must work live: the brief (rough task, 3 to 5 questions, flashcard)
and the question filter (answer or pass on). Everything else is a simple version.

## Where things live

- **App:** Lovable project "KLEER" (link at the bottom of this file). Only the
  Build Lead prompts Lovable or edits app code. Lovable keeps its own GitHub
  repo; it cannot attach to this one.
- **This repo:** everything that is not the app.
  - `prompts/` — one file per agent prompt: `clarify.md`, `flashcard.md`, `match.md`, `answer.md`
  - `data/` — the made-up team of 6 and the three test tasks
  - `eval/` — intern questions labelled `answer` or `pass_on`, used to test the filter
  - `pitch/` — Commfy answers, 2-minute script, judge Q&A, interview numbers

## Who owns what

| Role | Owns | Does not touch |
|------|------|----------------|
| Build Lead | Lovable project, database, both screens | Prompt wording |
| AI & Voice Lead | `prompts/`, `eval/`, ElevenLabs, TypeSafe check | App code |
| Product & Domain Lead | `data/`, `pitch/`, the 10 interviews | App code, prompts |

Prompt changes travel one way: the AI & Voice Lead edits a file in `prompts/`,
then the Build Lead pastes it into the matching file in `src/agents/`. Do not change a
shape below without telling the other two.

## Rules

- Made-up, public or consented data only. No real names, calendars or HR data.
- The filter passes on when in doubt. A wrong answer costs more than an
  unnecessary interruption.
- Every agent answer shows where it came from.
- No API keys or redeem codes in this repo.

## Data shapes

Field names are fixed. The app's tables and the prompts both use them.

```ts
type TeamMember = {
  id: string;
  name: string;
  role: string;              // "Summer intern", "Junior analyst"
  skills: string[];
  free_hours: number;        // free hours this week
  preferences: string;       // one sentence
};

type Clarification = { question: string; answer: string };

type Flashcard = {
  goal: string;
  deliverable: string;
  deadline: string;          // readable, with a real date: "Fri 20 Nov, 17:00"
  context: string;
  done_looks_like: string;
  who_to_ask: string;
};

type Mission = {
  id: string;
  rough_task: string;
  status: "draft" | "sent";
  clarifications: Clarification[];
  flashcard: Flashcard | null;
  first_leads: string[];                         // 3 starting points
  time_plan: { step: string; hours: number }[];
  todos: { title: string; priority: number; done: boolean }[];
  match_ranking: { member_id: string; score: number; reason: string }[];
  assignee_id: string | null;
  created_at: string;
  sent_at: string | null;
};

type Question = {
  id: string;
  mission_id: string;
  asked_by: string;                              // team member id
  question: string;
  status: "answered_by_agent" | "passed_to_manager" | "answered_by_manager";
  answer: string | null;
  source: "flashcard" | "clarification" | "manager_answer" | null;
  source_detail: string | null;                  // field name or quoted line
  created_at: string;
  answered_at: string | null;
};
```

## The four agent prompts

Each one is a server function in the Lovable app at
`src/agents/<name>.functions.ts`, with its wording in a constant at the top of
the file (for example `ANSWER_SYSTEM_PROMPT`). The shared model call is in
`src/agents/llm.server.ts` and uses Lovable AI. All return JSON only.

| Function | Input | Output |
|----------|-------|--------|
| `clarify` | `{ rough_task }` | `{ questions: { question, why }[] }` — 3 to 5, the ones an intern would ask |
| `flashcard` | `{ rough_task, clarifications, today }` | `{ flashcard, first_leads, time_plan, todos }` |
| `match` | `{ flashcard, team: TeamMember[] }` | `{ ranking: { member_id, score, reason }[] }` — all 6, score 0 to 100, one reason each |
| `answer` | `{ question, flashcard, clarifications, past_answers: { question, answer }[] }` | `{ decision: "answer" \| "pass_on", answer, source, source_detail }` |

`answer` rules: answer only when the supplied material states it; never use
general knowledge to fill a gap about this task or this manager; otherwise
`pass_on`. `past_answers` is every question the manager has answered, across all
missions, so the next intern gets it without asking.

## State of the app (10:45, Sat 14 Nov)

Working and tested once in Lovable's test browser: brief, flashcard with a real
date, ranking of the six, send, live arrival on the intern screen, answer from
the flashcard, pass on to the manager, live reply, reuse of the manager's reply
for a reworded question, saved to-dos, counter, Reset demo.

Not done or not checked:
- Not published yet. The preview link needs a Lovable login.
- Never run on two separate laptops.
- A mission is saved only on "Approve and send"; there are no drafts.
- The `answer` wording was loosened after its test to reuse the manager's
  replies "even when worded differently". That change is untested, so the
  `eval/` set should include near-miss questions that must still be passed on.

## Demo script (2 minutes, we drive, two laptops side by side)

1. Manager types "benchmark our competitors for Friday".
2. Answers three questions, sees the flashcard and the suggested person, approves.
3. Intern asks a question the brief covers: instant answer with its source.
4. Intern asks one it does not cover: it appears on the manager's screen.
5. Manager answers. Intern asks it again: instant answer from the saved reply.
6. Point at the counter: questions asked against questions that reached the manager.

## Links

- Lovable project (editor): https://lovable.dev/projects/1d4c1201-5b4a-49b1-85e8-89bf48f0ffdc
- Preview: https://id-preview--1d4c1201-5b4a-49b1-85e8-89bf48f0ffdc.lovable.app
- Published URL: _added once published_
