# How Ponytail shapes a Cursor prompt

This is the Cursor path only: user-level hooks in `~/.cursor/hooks.json`, plus the optional always-on rule `.cursor/rules/ponytail.mdc`. Other agents use different loaders; see [agent-portability.md](agent-portability.md) and [cursor-hooks.md](cursor-hooks.md).

Ponytail does not rewrite your prompt. It adds instruction context before the model answers, and those instructions change what the model chooses to build and how it replies.

## End-to-end pipeline

```mermaid
flowchart TD
  A[Open a new Agent chat] --> B{sessionStart hook}
  B --> C{Workspace has<br/>.cursor/rules/ponytail.mdc?}
  C -->|Yes| D[Inject one-line notice:<br/>rule owns the ruleset]
  C -->|No| E[Resolve default level<br/>env → config.json → full]
  E --> F{Level is off?}
  F -->|Yes| G[No injection]
  F -->|No| H[Inject filtered ruleset<br/>from skills/ponytail/SKILL.md]
  D --> I[Chat ready]
  G --> I
  H --> I

  I --> J[You send a message]
  J --> K{beforeSubmitPrompt hook}
  K --> L{Message is a<br/>ponytail command?}
  L -->|No| M[Hook prints nothing]
  L -->|Yes and rule present| N[Notice only; level unchanged]
  L -->|Yes and hooks own mode| O[Update mode flag<br/>inject new level or OFF]
  M --> P[Cursor builds the model request]
  N --> P
  O --> P

  P --> Q[System: session context<br/>+ workspace rules<br/>+ this turn's hook context]
  Q --> R[User message]
  R --> S[Model answers under<br/>whatever ruleset is in context]
  S --> T[Closing lines:<br/>what was skipped / risk]
```

## 1. Chat start — `sessionStart`

Runs once when the conversation is created. Its `additional_context` is stored on the chat and sent with every later request.

```mermaid
flowchart LR
  subgraph sources [Default level]
    E1[PONYTAIL_DEFAULT_MODE] --> E2["%APPDATA%/ponytail/config.json"]
    E2 --> E3[full]
  end

  subgraph out [What the model gets]
    R[Always-on rule notice]
    F[PONYTAIL MODE ACTIVE — level: …<br/>+ filtered SKILL.md]
    Z[Nothing]
  end

  sources --> Decide{Rule file present?}
  Decide -->|Yes| R
  Decide -->|No, level off| Z
  Decide -->|No, lite/full/ultra| F
```

| Outcome | Injected text |
|---------|----------------|
| Rule file in workspace | Notice that the rule owns behavior; mode commands will not switch levels |
| Hooks own mode, level `off` | Empty |
| Hooks own mode, `lite` / `full` / `ultra` | Header + `SKILL.md` with only that level’s table row kept |

## 2. Each message — `beforeSubmitPrompt`

Runs on every send. Ordinary coding prompts pass through unchanged. Mode commands are plain messages; Cursor has no built-in `/ponytail` command.

```mermaid
flowchart TD
  M[Your message] --> P{Parse}
  P -->|/ponytail lite\|full\|ultra| S[Set live level<br/>inject MODE CHANGED + ruleset]
  P -->|/ponytail off<br/>stop ponytail<br/>normal mode| O[Clear flag<br/>inject MODE OFF]
  P -->|/ponytail| R[Report current level]
  P -->|/ponytail default &lt;level&gt;| D[Write config.json default]
  P -->|Anything else| X[No hook output]

  S --> Note{Rule file present?}
  O --> Note
  R --> Note
  D --> Note
  Note -->|Yes| N[Replace with coexistence notice;<br/>do not change the flag]
  Note -->|No| Apply[Deliver hook JSON to Cursor]
  X --> Model[Model request continues]
  N --> Model
  Apply --> Model
```

`MODE OFF` does not erase the ruleset already stored from `sessionStart`. The notice tells the model to stop applying it for the rest of the chat.

## 3. What reaches the model on a normal coding turn

```mermaid
flowchart TB
  subgraph context [Request context]
    SC[sessionStart additional_context<br/>or empty]
    WR[Workspace rules<br/>e.g. ponytail.mdc if present]
    HC[This turn's beforeSubmitPrompt<br/>context — usually empty]
    UM[Your user message]
  end

  SC --> Model
  WR --> Model
  HC --> Model
  UM --> Model
  Model[Model] --> Reply[Reply shaped by the ruleset]
```

In this ponytail checkout, `WR` already contains the compact ruleset, so hooks stay quiet. In a normal project with only the user-level hooks installed, `SC` carries the filtered skill and `WR` has no ponytail rule.

## 4. How the ruleset changes the answer

Once the ruleset is in context, the model is steered before it writes code. This is instruction-following, not a separate compiler pass.

```mermaid
flowchart TD
  Task[Task + code it touches] --> Scope[List callers, tests,<br/>fixtures, config, exports]
  Scope --> Need{Does this need<br/>to exist?}
  Need -->|No| Skip[Skip it; name it later]
  Need -->|Yes| Reuse{Already in the repo,<br/>stdlib, or a dependency?}
  Reuse -->|Yes| UseIt[Use that]
  Reuse -->|No| Smallest[Smallest complete change]
  UseIt --> Finish[Update every place<br/>the change must reach]
  Smallest --> Finish
  Finish --> Guard{Trust boundary / data loss<br/>/ security / a11y / asked-for?}
  Guard -->|Must keep| Keep[Do not cut]
  Guard -->|Extra| Drop[Leave out]
  Keep --> Reply[Ship the diff]
  Drop --> Reply
  Skip --> Reply
  Reply --> Close[1–2 lines: skipped or<br/>not checked + risk]
```

### Levels

| Level | Extra pressure on the same pipeline |
|-------|-------------------------------------|
| **lite** | Build what was asked; name a smaller option in one line and let you pick |
| **full** | Default: smallest complete change, check where logic needs one, closing skip/risk lines |
| **ultra** | Before building, push back on any part the need does not justify |

### What “skipped” means here

Not a task queue. Two things land in the closing lines:

1. **Work left out on purpose** — features, options, wrappers, config, and flexibility nobody asked for.
2. **Checks not run** — tests not executed, files not opened, installs not verified.

Never listed as skippable: callers/tests/fixtures the change breaks, validation at trust boundaries, error handling that prevents data loss, security, accessibility, hardware calibration, and anything you asked for.

A known limit that still ships goes in code as `shortcut: <limit>, <when to upgrade>` (debt for `/ponytail-debt`), not only in the reply footer.

## 5. Rule vs hooks (pick one)

```mermaid
flowchart LR
  subgraph rule [Always-on rule]
    MDC[".cursor/rules/ponytail.mdc"]
    Fixed[Fixed full-style ruleset<br/>every prompt]
  end

  subgraph hooks [User hooks]
    HJ["~/.cursor/hooks.json"]
    Levels[lite / full / ultra / off<br/>switchable per chat]
  end

  MDC --> Fixed
  HJ --> Levels
  Both{Both present in<br/>this workspace?}
  Both -->|Yes| Fixed
  Both -->|No, hooks only| Levels
```

They are alternatives, not layers. Delete the rule in a workspace when you want `/ponytail lite` and friends to work there.

## 6. Gaps in the Cursor path

```mermaid
flowchart TD
  Chat[Parent Agent chat] -->|gets ruleset| OK[sessionStart / rule]
  Chat -->|spawns| Sub[Subagent]
  Sub -->|no ruleset| Gap1[subagentStart cannot inject context]
  Cloud[Cloud agent] -->|no sessionStart| Gap2[Startup injection missing;<br/>/ponytail on submit still works]
```

## Quick check in a project

1. Prefer a project **without** `.cursor/rules/ponytail.mdc`.
2. Open a **new** Agent chat.
3. Ask: *Quote the first line of any ponytail context you were given.*
4. Expect something like `PONYTAIL MODE ACTIVE — level: full` (or your configured default).
5. Send `/ponytail lite`, then ask again; expect `PONYTAIL MODE CHANGED — level: lite`.

In the Hooks output channel under Customize, each `sessionStart` / `beforeSubmitPrompt` run shows whether the scripts printed JSON or stayed silent.
