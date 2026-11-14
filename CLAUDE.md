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
  // To-do hours add up to the time plan's total. Progress shown to the manager
  // = hours of done to-dos / total hours (3 of 10 hours ticked = 30%).
  todos: { title: string; priority: number; hours: number; done: boolean }[];
  // The screens show the confidence badge only; score just orders the list.
  match_ranking: {
    member_id: string;
    confidence: "high" | "moderate" | "low";
    score: number;
    reason: string;
  }[];
  assignee_id: string | null;
  created_at: string;
  sent_at: string | null;
};

type Question = {
  id: string;
  mission_id: string;
  asked_by: string;                              // team member id
  question: string;
  status:
    | "checking_brief"               // intern's agent is reading the brief
    | "answered_by_agent"            // intern's agent answered from the brief
    | "asking_manager_agent"         // handed over to the manager's agent
    | "answered_by_manager_agent"    // manager's agent answered from company knowledge
    | "passed_to_manager"            // neither agent knew
    | "answered_by_manager";
  answer: string | null;
  source: "flashcard" | "clarification" | "company_knowledge" | "manager_answer" | null;
  source_detail: string | null;                  // flashcard field or knowledge entry title
  knowledge_id: string | null;                   // the entry the manager's agent cited
  trail: { from: Party; to: Party; text: string; at: string }[];
  created_at: string;
  answered_at: string | null;
};

type Party = "intern" | "intern_agent" | "manager_agent" | "manager";

// Made-up company knowledge. A real version would index Drive, email and chat
// with consent. The manager's own replies are saved here too.
type Knowledge = {
  id: string;
  title: string;
  source_type: "drive_file" | "email" | "chat" | "handbook" | "manager_answer";
  location: string;                              // "Drive / Brand / ..."
  author: string;
  doc_date: string;
  content: string;
  mission_id: string | null;                     // set on the manager's replies
};
```

## The agent prompts

Each one is a server function in the Lovable app at
`src/agents/<name>.functions.ts`, with its wording in a constant at the top of
the file (for example `ANSWER_SYSTEM_PROMPT`). The shared model call is in
`src/agents/llm.server.ts` and uses Lovable AI. All return JSON only.

| Function | Input | Output |
|----------|-------|--------|
| `clarify` | `{ rough_task, knowledge }` | `{ questions: { question, why, options: [string, string, string] }[] }` — 3 to 5 questions, each with three answers specific to the company; the manager can select several and add "Other…" text, saved joined with "; ". Never asks a formatting question |
| `flashcard` | `{ rough_task, clarifications, today, knowledge }` | `{ flashcard, first_leads, time_plan, todos }` — knowledge is used for `first_leads` only |
| `match` | `{ flashcard, team: TeamMember[] }` | `{ ranking: { member_id, confidence, score, reason }[] }` — all 6, best first, confidence high, moderate or low, one reason each |
| `answer` (intern's agent) | `{ question, flashcard, clarifications }` | `{ decision: "answer" \| "pass_on", answer, source, source_detail }` |
| `managerAgent` (manager's agent) | `{ handover, flashcard, knowledge: Knowledge[] }` | `{ decision: "answer" \| "escalate", answer, knowledge_id, note }` |

How a question travels:

1. The intern's agent reads only the brief (flashcard and briefing answers). If
   the brief states the answer, it answers.
2. Otherwise it writes a handover to the manager's agent: the question restated
   so it stands alone, the mission goal, and what it checked.
3. The manager's agent reads the company knowledge. If one entry states the
   answer, it answers and cites that entry. The manager is not asked.
4. Otherwise it escalates to the manager with a one-line note.
5. The manager's reply is saved as a `Knowledge` entry, so the manager's agent
   answers it next time.

Both agents follow the same rule: answer only when the supplied material states
it, never fill a gap from general knowledge, and pass on when in doubt.

One exception, decided by the team: formatting questions (template, layout,
slide count, fonts, colours, charts, file type, naming, number and date formats)
never reach the manager. The manager's agent answers them from the brand
guidelines and the previous decks in the knowledge base. If no entry states it
exactly, it gives the closest precedent from the most recent deck of the same
type and names that deck. `clarify` never asks the manager a formatting question.

## State of the app (11:45, Sat 14 Nov)

Working, tested once in Lovable's test browser and confirmed in the database:
- Brief with multi-select answer options drawn from company knowledge, plus
  "Other…"; no formatting questions asked.
- Flashcard with a real date; first leads that name company files, including
  the previous deck to copy the format from.
- Ranking of the six with High, Moderate or Low confidence badges, send, live
  arrival on the intern screen.
- "Plan and to-dos" with hours per to-do; hours-weighted progress shown live on
  the intern screen and under "Sent missions" on the manager screen.
- Formatting questions answered by the manager's agent from previous decks,
  including one no entry states; none reached the manager.
- Question routing: brief, then manager's agent with company knowledge, then manager.
- The manager's reply saved as knowledge and reused for a reworded question.
- The agent exchange shown live on both screens; four-part counter; Reset demo.

The 11:15 run of the demo script gave: asked 5, from the brief 1, by the
manager's agent 3, reached the manager 1. The 11:40 run (five formatting
questions and one scope question) gave: asked 6, by the manager's agent 5,
reached the manager 1; ticking a 2-hour to-do of 7 hours showed 29%.

Not done or not checked:
- Not published yet. The preview link needs a Lovable login.
- Never run on two separate laptops.
- Reuse of a manager's reply across two different missions.
- Formatting questions on a task that is not a deck (for example a written
  meeting briefing). The knowledge base only describes decks.
- About 8 of the 20 knowledge entries have been asked about. The `eval/` set
  should cover the rest, plus near-miss questions the manager's agent must
  escalate (for example a competitor trial with a personal email address).
- "Who to ask" on the flashcard reads "Not specified" unless the manager says.
- A mission is saved only on "Approve and send"; there are no drafts.

## Demo script (2 minutes, we drive, two laptops side by side)

1. Manager types "benchmark our competitors for Friday".
2. Clicks an option for each question (no typing), sees the flashcard and the
   suggested person, approves.
3. Intern asks "When is this due?": answered from the brief.
4. Intern asks "Which slide template should I use?": the intern's agent asks the
   manager's agent, which answers from Brand guidelines 2026. The manager is not asked.
5. Intern asks "Should the benchmark cover the US market too, or only the UK?":
   neither agent knows, so it appears on the manager's screen. Manager answers.
6. Intern asks "Do I need to look at competitors in America?": the manager's
   agent answers from the saved reply.
7. Point at the counter: asked, answered from the brief, answered by the
   manager's agent, reached the manager.

The made-up company is Kettlebridge (rota software for independent restaurants),
the manager is Dana Whitfield, and the 20 knowledge entries (16 general, 4 on
previous decks and house style) are listed under "Company knowledge" on the
manager screen.

## Links

- Lovable project (editor): https://lovable.dev/projects/1d4c1201-5b4a-49b1-85e8-89bf48f0ffdc
- Preview: https://id-preview--1d4c1201-5b4a-49b1-85e8-89bf48f0ffdc.lovable.app
- Published URL: _added once published_
