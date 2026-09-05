# Vemo — Build Specification

**Status:** Active
**Purpose:** Single source of truth for building Vemo
**Platform:** Windows
**Architecture:** Lightweight local-first personal AI companion
**Priority:** Finish a powerful working Vemo before adding infrastructure or configuration systems

---

# 1. What Vemo Is

Vemo is a lightweight, local-first personal AI companion for Windows.

Vemo is **the system**, not a particular AI model.

Models, tools, memory, skills, and integrations are components used by Vemo.

The fundamental goal is:

> Build one small application that can intelligently use local models, stronger cloud models, memory, tools, research, and the user's computer to solve problems.

Vemo should feel like a persistent personal companion that happens to be highly capable.

It should be:

* natural
* competent
* somewhat adorable
* playful during casual conversation
* focused during serious work
* persistent across conversations
* independent of any particular AI provider

Changing the underlying model must not destroy Vemo's:

* personality
* memory
* preferences
* conversation history
* skills
* configuration

---

# 2. Core Development Philosophy

## The most important rule

**Do not build infrastructure before it is needed.**

Vemo is currently a personal application, not a platform.

Prefer:

```text
simple function
```

over:

```text
manager
registry
factory
provider framework
plugin framework
configuration system
```

Only introduce an abstraction when the application actually has multiple implementations that require it.

---

# 3. What We Are NOT Building Right Now

Do NOT build these merely because the specification mentions them:

* plugin marketplace
* connector marketplace
* provider management UI
* model management UI
* settings framework
* permission management UI
* skill marketplace
* skill installer
* generic configuration editor
* multi-user support
* accounts
* authentication
* cloud synchronization
* server infrastructure
* PostgreSQL
* Supabase
* Redis
* separate memory server
* separate orchestration server
* microservices
* continuous AI inference
* continuous screen monitoring
* always-listening microphone
* background model inference
* complicated agent frameworks
* multi-agent infrastructure
* model benchmarking system
* elaborate mood engine
* elaborate confidence UI
* elaborate activity dashboard
* complicated backup system
* unnecessary abstractions

These can be added later if Vemo genuinely needs them.

---

# 4. Vemo MVP

The first real Vemo should be able to:

* run as a Windows application
* remain effectively idle when unused
* wake with a keyboard shortcut
* provide a full chat interface
* maintain conversation context
* persist conversation history
* search previous conversations
* automatically extract useful memories
* retrieve relevant old memories
* maintain evolving preferences
* use a local model as the primary orchestrator
* use stronger cloud models when useful
* support an OpenAI-compatible cloud endpoint
* perform web research
* inspect webpages
* perform quick/normal/deep research
* take screenshots on request
* send screenshots to a vision-capable model
* search local files
* read local files
* run controlled terminal commands
* read Obsidian
* write to Obsidian when explicitly instructed
* use markdown-based skills
* perform difficult multi-step problem solving
* use model fallbacks
* operate with near-zero idle resource usage

---

# 5. Initial Architecture

Vemo should initially be one application.

Conceptually:

```text
                    VEMO.EXE
                       │
                       ▼
                  ┌─────────┐
                  │   UI    │
                  └────┬────┘
                       │
                       ▼
                handleMessage()
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Memory        Router        Tools
       SQLite       Local LLM       │
          │            │            ├── Web
          │            │            ├── Files
          │            │            ├── Terminal
          │            │            ├── Screenshot
          │            │            └── Obsidian
          │            │
          │            ▼
          │        Cloud LLM
          │            │
          └────────────┼────────────┘
                       ▼
                    Answer
```

There should not be multiple services unless there is a concrete reason.

---

# 6. Suggested Project Structure

Keep the project small.

```text
vemo/
├── src/
│   ├── app/
│   │   └── App.tsx
│   │
│   ├── core/
│   │   ├── vemo.ts
│   │   ├── prompt.ts
│   │   ├── router.ts
│   │   └── types.ts
│   │
│   ├── models/
│   │   ├── ollama.ts
│   │   └── cloud.ts
│   │
│   ├── memory/
│   │   └── memory.ts
│   │
│   ├── tools/
│   │   ├── web.ts
│   │   ├── files.ts
│   │   ├── terminal.ts
│   │   ├── screenshot.ts
│   │   └── obsidian.ts
│   │
│   └── main.ts
│
├── skills/
│   ├── coding.md
│   ├── deep-research.md
│   ├── hardware.md
│   └── writing.md
│
├── data/
│   └── vemo.db
│
├── package.json
└── VEMO_BUILD_SPEC.md
```

The exact structure can change if implementation requires it.

The principle is more important than the exact filenames:

> Keep related functionality together and avoid unnecessary layers.

---

# 7. Configuration

For now, configuration lives in code.

Example:

```ts
export const CONFIG = {
  localModel: "qwen3:4b",

  ollamaUrl: "http://127.0.0.1:11434",

  cloud: {
    provider: "openai-compatible",
    baseUrl: "",
    model: "",
    apiKey: ""
  },

  obsidianVault: "",

  hotkey: "Alt+Space",

  warmTimeMs: 30_000,

  memoryResults: 8
};
```

There does NOT need to be a settings UI.

There does NOT need to be a configuration database.

Changing a constant and rebuilding the application is acceptable during development.

Secrets should eventually be stored securely rather than committed to source control.

---

# 8. Runtime

The central runtime is Vemo.

A simplified flow:

```text
User request
     ↓
Load recent conversation
     ↓
Retrieve relevant memory
     ↓
Local model understands request
     ↓
Determine complexity
     ↓
Determine required tools
     ↓
Determine whether stronger model is needed
     ↓
Execute tools
     ↓
Check results
     ↓
Use specialist/cloud model when appropriate
     ↓
Synthesize final response
     ↓
Save conversation
     ↓
Extract useful memory
```

The runtime should be deterministic where possible and model-driven where intelligence is actually needed.

---

# 9. Core Message Pipeline

The central function should conceptually resemble:

```ts
async function handleMessage(text: string) {
  const history = await conversations.recent();
  const memories = await memory.search(text);

  const decision = await localModel.route({
    text,
    history,
    memories
  });

  const toolResults = await executeTools(decision.tools);

  const answer = decision.useCloudModel
    ? await cloudModel.generate({
        text,
        history,
        memories,
        toolResults
      })
    : await localModel.generate({
        text,
        history,
        memories,
        toolResults
      });

  await conversations.save(text, answer);

  extractMemoryInBackground(text, answer);

  return answer;
}
```

The real implementation can differ.

The important thing is that Vemo owns this pipeline.

---

# 10. Local Model

The local model is Vemo's primary orchestrator.

Responsibilities:

* understand requests
* handle ordinary conversation
* inspect context
* retrieve memory
* determine task complexity
* decide whether tools are required
* decide whether a stronger model is warranted
* select skills
* help plan tasks
* synthesize ordinary results

Ollama is the initial local model backend.

The local model should not need to remain running when Vemo is idle.

---

# 11. Cloud Models

Cloud models are optional specialists.

Initially support one OpenAI-compatible interface.

Conceptually:

```ts
interface Model {
  generate(input: ModelInput): Promise<ModelOutput>;
}
```

Initial implementations:

```text
OllamaModel
OpenAICompatibleModel
```

Do not build a large provider abstraction system.

The OpenAI-compatible endpoint should make it possible to use different providers without writing a custom integration for every provider.

---

# 12. Model Routing

Routing should initially be simple.

Examples:

```text
"How are you?"
→ local model

"What is 15% of 200?"
→ local model

"Find current GPU prices in Pakistan."
→ local model + web

"Look at my screen."
→ screenshot + vision model

"Fix this complicated architecture."
→ stronger coding/reasoning model

"Deeply investigate this."
→ deep research workflow
```

Routing may use deterministic checks plus the local model.

Example decision:

```json
{
  "complexity": "high",
  "tools": ["web"],
  "specialist": "reasoning"
}
```

Do not build a sophisticated routing framework unless simple routing proves insufficient.

---

# 13. Fallbacks

Cloud model failures should not destroy the task.

Basic fallback logic is sufficient:

```ts
try {
  return await primaryModel.generate(request);
} catch {
  return await fallbackModel.generate(request);
}
```

The fallback system should remain small.

---

# 14. Memory

Memory is one of Vemo's core capabilities.

Memory belongs to Vemo, not to a model.

Use SQLite.

Do not introduce:

* PostgreSQL
* Supabase
* Redis
* a separate memory service
* a vector database server

unless scale actually requires them.

---

# 15. Memory Database

Initial tables:

```text
conversations
messages
memories
preferences
```

It is acceptable to combine memories and preferences initially if that simplifies implementation.

A memory can contain:

```text
id
type
category
fact
confidence
created_at
updated_at
status
embedding
```

Example:

```json
{
  "type": "preference",
  "category": "music",
  "fact": "Strongly prefers high-energy emotionally intense songs.",
  "confidence": 0.94,
  "status": "active"
}
```

---

# 16. AI-Efficient Memory

Do not store memories as essays.

Prefer compact structured facts.

Example:

```text
ENTITY: user
CATEGORY: hardware.preference
FACT: prioritizes performance/value over aesthetics
CONFIDENCE: 0.94
FIRST_OBSERVED: 2026-09-05
LAST_CONFIRMED: 2026-09-05
STATUS: active
```

Memory should be optimized for retrieval and model consumption.

---

# 17. Automatic Memory

Vemo should automatically identify useful long-term information.

Example:

```text
User repeatedly rejects slow music.
        ↓
Vemo detects pattern.
        ↓
Preference created.
        ↓
Confidence increases with additional evidence.
```

Memory extraction should happen after the main response whenever possible so it does not unnecessarily slow down the user's interaction.

---

# 18. Memory Evolution

Do not simply delete old information.

Example:

```text
Old memory
    ↓
new contradictory evidence
    ↓
confidence decreases
    ↓
new memory becomes active
    ↓
old memory becomes superseded
```

Old memories remain available as historical context when appropriate.

---

# 19. Memory Retrieval

Memory retrieval should be automatic.

The user should not need to say:

> "Remember that we talked about this six months ago."

Vemo should retrieve relevant historical information automatically.

Initially, a relatively small memory database can use simple embedding similarity or even efficient brute-force similarity.

Do not optimize for millions of records before Vemo has millions of records.

---

# 20. Conversation History

Conversation history is separate from long-term memory.

History contains raw conversations.

It should support:

* saving messages
* reopening conversations
* searching conversations
* deleting conversations

Deleting conversation history should NOT automatically delete extracted memories.

---

# 21. Preferences

Preferences are simply persistent user-specific information.

Initially they can be represented as memory objects.

Example:

```text
Music
- likes high-energy tracks
- likes emotionally intense songs
- dislikes generic slow recommendations

Hardware
- prioritizes performance/value
- does not care much about flashy aesthetics
```

A dedicated preference UI is not required initially.

---

# 22. Skills

Skills should initially be simple Markdown files.

Example:

```text
skills/
├── coding.md
├── deep-research.md
├── hardware.md
└── writing.md
```

A skill contains instructions for performing a type of work.

Example:

```text
# Hardware Research

Prioritize:
- current pricing
- performance per rupee
- compatibility
- Pakistani availability
- used-market value
```

When a skill is relevant, Vemo loads the file into context.

No skill marketplace.
No skill installer.
No complicated skill engine.

User-created skills can simply be new `.md` files.

---

# 23. Tools

Tools should initially be ordinary functions.

Conceptually:

```ts
const tools = {
  webSearch,
  openWebpage,
  searchFiles,
  readFile,
  runTerminal,
  screenshot,
  readObsidian,
  writeObsidian
};
```

Do not build a complicated tool registry.

Tools should be permissioned where necessary.

---

# 24. Web

Vemo should recognize when fresh information is necessary.

Example:

> "Find the best GPU under Rs. 50k."

Vemo should understand that:

* current information matters
* location matters
* Pakistani sources matter
* pricing matters
* user's hardware context may matter
* this is a recommendation task
* source quality matters

Basic research tools:

```text
search
open page
extract page content
compare information
verify claims
```

---

# 25. Research

Use one research system with a depth parameter.

```ts
research(query, depth)
```

Depth:

```text
quick
normal
deep
```

## Quick

```text
search
↓
inspect strongest result
↓
answer
```

## Normal

```text
search multiple sources
↓
inspect relevant sources
↓
compare
↓
answer
```

## Deep

```text
research
↓
compare sources
↓
identify disagreements
↓
research unresolved questions
↓
use stronger model
↓
critique
↓
verify
↓
synthesize
```

Vemo should infer research depth from the request when possible.

---

# 26. Maximum-Effort Problem Solving

This is a major Vemo capability.

For difficult tasks:

```text
Understand problem
        ↓
Retrieve memory
        ↓
Research
        ↓
Generate candidate solutions
        ↓
Ask stronger model(s) to reason
        ↓
Look for contradictions
        ↓
Critique leading solution
        ↓
Research unresolved issues
        ↓
Synthesize
        ↓
Assess confidence
        ↓
Final answer
```

The user should be able to simply say:

> "Figure this out."

Vemo decides whether the problem deserves maximum effort.

The user can explicitly request:

> "Deeply investigate this."

---

# 27. Confidence

Internally, Vemo should consider:

* evidence quality
* freshness
* source agreement
* model agreement
* ambiguity
* completeness

User-facing confidence should be natural.

Do not expose meaningless decimal scores unless useful.

Prefer:

```text
High confidence
Moderate confidence
Low confidence
```

---

# 28. Screenshot Understanding

No continuous screen monitoring.

When the user says:

> "Look at my screen."

Vemo should:

```text
capture screenshot
       ↓
send screenshot to vision model
       ↓
retain result as current task context
       ↓
continue conversation
```

The screenshot should be captured on demand.

Nothing should continuously inspect the screen.

---

# 29. File Access

Initial filesystem capabilities:

* search files
* inspect filenames
* read files
* inspect metadata where useful

MVP should primarily be read-oriented.

Example:

> "Find the config file I changed last month."

Vemo can search the filesystem and inspect likely candidates.

---

# 30. Terminal

Controlled terminal access is required for useful technical tasks.

Examples:

> "Check what's using port 5173."

> "Run the tests."

> "Execute this script."

Terminal commands should be classified as safe/risky where appropriate.

Destructive or dangerous commands should require confirmation.

---

# 31. Obsidian

Obsidian is an important connector.

Vemo can:

* search notes
* read notes
* create notes
* edit notes
* append content
* create folders
* rename/move notes
* create WikiLinks

Important rule:

> Vemo must never automatically write to Obsidian without an explicit user instruction.

Reading/searching Obsidian may happen automatically when relevant.

Examples:

```text
"Search my Obsidian notes for Vemo architecture."

"Create a note about this research."

"Add these findings to my PC upgrades note."
```

Initially, the Obsidian vault path can simply be hardcoded in configuration.

---

# 32. Permissions

Do not build a permission management UI initially.

Use a simple internal permission policy.

Example:

```ts
const PERMISSIONS = {
  web: "allow",
  filesRead: "allow",
  filesWrite: "deny",
  terminalSafe: "allow",
  terminalRisky: "confirm",
  obsidianRead: "allow",
  obsidianWrite: "explicit"
};
```

Possible states:

```text
allow
confirm
deny
explicit
```

The important requirement is that dangerous operations cannot silently execute.

---

# 33. Kill Switch

Vemo must have a way to immediately stop active work.

At minimum:

```text
Stop current task
```

The implementation can initially cancel active operations using an `AbortController` or equivalent mechanism.

Do not build an elaborate activity management system just for this.

---

# 34. Idle Behavior

When Vemo is not being used:

```text
CPU → effectively negligible
GPU → effectively zero
LLM inference → off
microphone processing → off
screen monitoring → off
network activity → none unless required
```

The application should primarily wait for user input.

---

# 35. Warm Period

After completing a task, Vemo remains ready for follow-up work for approximately:

```text
30 seconds
```

After that, return to idle.

The warm period should not mean continuous model inference.

---

# 36. Voice

Voice should be an input/output layer around the existing Vemo runtime.

The architecture should eventually be:

```text
hotkey
↓
speech-to-text
↓
handleMessage()
↓
response
↓
text-to-speech
```

Voice should not be part of the core intelligence architecture.

Do not block the core text experience on voice.

---

# 37. UI

Keep the UI minimal.

Primary interface:

```text
┌───────────────────────────────────┐
│ Vemo                              │
│                                   │
│ Conversation                      │
│                                   │
│                                   │
│                                   │
│───────────────────────────────────│
│ Ask Vemo...                   ↑   │
└───────────────────────────────────┘
```

The UI needs to provide:

* conversation
* message input
* response streaming if practical
* task/activity indication
* stop/cancel control
* compact mode
* full-screen mode

Do not build large management dashboards.

---

# 38. Settings

There is intentionally no settings UI during the initial build.

Configuration is code.

Examples:

```text
model
API endpoint
API key
Obsidian path
hotkey
warm timeout
memory limits
```

can all be changed in code.

A settings UI can be added after the core system is working.

---

# 39. Backup

Backup is not a priority for the first working build.

Eventually a backup should preserve:

```text
personality
memories
preferences
conversation history
model configuration
routing
tools
permissions
skills
Obsidian configuration
settings
```

Do not include raw model binaries by default.

Do not store API keys in normal backups.

---

# 40. Personality

Vemo has a persistent identity independent of the model.

Default personality:

* natural
* curious
* playful
* somewhat adorable
* companion-like
* expressive without being theatrical
* competent
* honest
* focused during serious work
* casual during casual conversation

Personality is supplied through Vemo's system prompt/context.

Changing the model must not change the personality.

---

# 41. Mood

Mood is optional and subtle.

Potential dimensions:

```text
Energy
Warmth
Curiosity
Seriousness
Playfulness
```

Mood should affect expression, not intelligence.

Vemo should never become less capable because of mood.

Do not build a complicated emotional simulation.

---

# 42. Model Independence

This is one of Vemo's fundamental principles.

The model is replaceable.

Vemo owns:

```text
identity
memory
preferences
history
skills
tools
routing
system behavior
```

The model provides intelligence.

Therefore:

```text
Model A
   ↓
Vemo
   ↓
same identity

Model B
   ↓
Vemo
   ↓
same identity
```

---

# 43. Security

The application runs on the user's own Windows machine.

Important principles:

* never execute dangerous commands without appropriate confirmation
* never silently delete user data
* never silently write to Obsidian
* never expose API keys to the model unnecessarily
* keep secrets outside source control
* provide a kill switch
* validate tool arguments
* constrain file operations where practical

Security should be practical rather than an enormous enterprise framework.

---

# 44. Development Order

Build Vemo in this order.

## Step 1 — Skeleton

Get the Windows application launching.

Implement:

```text
UI
main process
basic runtime
```

No AI complexity yet.

---

## Step 2 — Local Chat

Connect Ollama.

Implement:

```text
input
→ Ollama
→ response
```

Make this reliable first.

---

## Step 3 — Conversation

Add SQLite.

Implement:

```text
conversations
messages
```

Persist conversations.

---

## Step 4 — Context

Give the local model:

```text
recent conversation
system personality
```

Implement context management.

---

## Step 5 — Memory

Add:

```text
memory extraction
memory storage
memory retrieval
```

Test it heavily.

---

## Step 6 — Routing

Allow the local model to determine:

```text
simple task
tool task
cloud task
research task
deep task
```

Keep the routing implementation small.

---

## Step 7 — Cloud Model

Add one OpenAI-compatible cloud model.

Implement:

```text
local → cloud when necessary
```

Add fallback.

---

## Step 8 — Web

Implement:

```text
search
open
extract
research
```

Then add:

```text
quick
normal
deep
```

---

## Step 9 — Tools

Add:

```text
files
terminal
screenshot
```

One tool at a time.

---

## Step 10 — Obsidian

Add:

```text
search
read
create
edit
append
```

Ensure writes require explicit user intent.

---

## Step 11 — Skills

Load Markdown skill files dynamically.

---

## Step 12 — Maximum Effort

Combine:

```text
memory
research
strong models
criticism
verification
synthesis
```

into the deep problem-solving workflow.

---

## Step 13 — Voice

Only after the text runtime is reliable:

```text
hotkey
→ STT
→ Vemo
→ TTS
```

---

## Step 14 — Packaging

Produce a lightweight Windows installer/executable.

---

# 45. Testing Philosophy

Test the actual behavior instead of building test infrastructure for its own sake.

Important tests:

### Conversation

```text
send message
→ receives response
→ conversation persists
```

### Memory

```text
tell Vemo preference
→ later ask related question
→ relevant memory retrieved
```

### Model switching

```text
use local model
→ switch cloud model
→ personality/memory/history remain
```

### Routing

```text
simple question
→ local model

current research
→ web

difficult problem
→ stronger model
```

### Tools

```text
file search works
terminal works
screenshot works
Obsidian works
```

### Safety

```text
dangerous command
→ confirmation

Obsidian write without explicit instruction
→ blocked
```

---

# 46. Performance Requirements

Performance is a core requirement.

When idle:

```text
No model inference
No continuous microphone processing
No continuous screen capture
No unnecessary network requests
```

The application should consume minimal CPU and RAM.

When active, performance can temporarily increase because useful work is occurring.

The goal is not zero resource usage.

The goal is:

> zero meaningful resource usage when Vemo is doing nothing.

---

# 47. Anti-Bloat Rules

Before adding code, ask:

1. Does Vemo actually need this?
2. Can a simple function solve it?
3. Can an existing component solve it?
4. Is this required for the current MVP?
5. Does this add complexity that will make debugging harder?

If the answer to #1 or #4 is no:

**Do not build it yet.**

---

# 48. Abstraction Rule

Start concrete.

Bad:

```text
AbstractToolFactory
ToolRegistryProvider
ToolExecutionManager
ToolCapabilityResolver
```

Good:

```ts
async function searchFiles(query: string) {}
```

When there are actually multiple implementations and a real need for abstraction:

```text
then refactor
```

Do not architect imaginary future requirements.

---

# 49. What "Powerful" Means

Vemo does not become powerful by having thousands of classes.

Vemo becomes powerful by combining:

```text
good local model
+
strong cloud model
+
good routing
+
memory
+
web
+
filesystem
+
terminal
+
vision
+
Obsidian
+
skills
+
research
+
verification
```

The application around these components should remain small.

---

# 50. Definition of Done

Vemo's first real release is complete when this workflow works:

```text
Launch Vemo
     ↓
Press hotkey
     ↓
Ask something
     ↓
Vemo understands it
     ↓
Vemo remembers relevant context
     ↓
Vemo chooses local/cloud intelligence
     ↓
Vemo uses tools when necessary
     ↓
Vemo researches when necessary
     ↓
Vemo verifies difficult work
     ↓
Vemo answers naturally
     ↓
Vemo saves the conversation
     ↓
Vemo learns useful long-term information
     ↓
Vemo becomes idle
```

The application should feel like one coherent assistant rather than a collection of disconnected features.

---

# 51. Final Architectural Principle

The most important sentence in this document:

> **Build the smallest system capable of producing the Vemo experience.**

Do not build a platform.

Do not build infrastructure for hypothetical users.

Do not build configuration screens for things that can currently be constants.

Do not build abstractions for imaginary implementations.

Do not build future features during MVP.

Build Vemo itself.

---

# 52. Current Priority

The immediate priority is:

```text
WORKING VEMO
```

not:

```text
perfect architecture
perfect settings
perfect abstraction
perfect UI
perfect plugin system
perfect provider system
```

The implementation should move forward incrementally.

When something breaks:

1. inspect the actual code
2. identify the root cause
3. fix the smallest necessary surface
4. build
5. test
6. continue

Do not rewrite working systems without a concrete reason.

---

# 53. Development Context

Vemo is a personal Windows application.

The repository is public during development.

When inspecting the current project implementation, use the current repository/code rather than relying on an old remembered version.

The repository:

```text
https://github.com/zuixhh/Vemo
```

The current code is authoritative over this document for implementation details.

This document is authoritative for the intended architecture and scope.

If the repository and this document disagree:

* current working code describes reality
* this document describes intended direction
* do not blindly rewrite the project
* identify the discrepancy and make the smallest change necessary

---

# 54. Rule for Future Conversations

When this document is provided at the beginning of a conversation, assume:

* Vemo is the project being built
* this is the current architectural direction
* the goal is a lightweight but highly capable Windows application
* unnecessary infrastructure should be rejected
* implementation should prioritize working software
* code should be inspected before proposing large changes
* existing working code should be preserved
* features should be implemented incrementally
* the assistant should not expand the MVP without explicit agreement

The ultimate goal is simple:

> **Vemo should be a tiny application around a powerful intelligence loop.**

> **Build the system once. Change the models forever.**
