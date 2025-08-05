Below is a **self-contained, engineering-ready implementation spec** for the Growth99 “conversational website builder.” It assumes **Next.js (App Router) + TypeScript** on the frontend and a **FastAPI + LangGraph** backend. It reproduces the essential mechanics of Dyad’s open-source approach (streamed file operations via XML-like tags, a virtual filesystem overlay, TypeScript problem loop, live preview with a shim, and component selection), but **does not require Dyad’s code**.

> **Important:** Where this spec quotes exact snippets from Dyad (to nail critical parsing or injection details), they are clearly labeled **“Copied verbatim from Dyad”** and cited inline so your team can implement them precisely.

---

## 1) What we’re building

A **conversational web app** where a designer/developer chats with an AI agent:

> “Create a home page with hero, pricing, and contact. Use brand color `#0ea5e9`. Add a booking widget. Make it mobile-first.”

The system **streams** code changes as structured “write/rename/delete” operations, applies them to a **virtual filesystem (VFS)**, type-checks and auto-fixes issues, then **renders live** in a preview iframe. A **component picker** lets the user click any element in the preview to instruct the next agent step to focus on that component/file.

---

## 2) Tech stack & high-level architecture

**Frontend**

* **Next.js (App Router)**, TypeScript, Tailwind, shadcn/ui
* Chat UI (SSE streaming), file tree & diff, problems panel
* Preview iframe (two modes: **Sandpack/Vite** or **Next Dev-Proxy**)
* Component picker overlay (DOM tagging + `postMessage` bridge)

**Backend**

* **FastAPI** HTTP API (+ SSE endpoint)
* **LangGraph** multi-agent orchestration (Planner → Writer → Typecheck → Fixer → RuntimeFixer → Committer)
* **VFS service** (overlay of writes/deletes/renames)
* **TypeScript check service** (Node worker/container running `tsc --noEmit`)
* **Preview services**: Sandpack (in-browser) and a **Node Dev-Proxy** that injects a shim into Next’s HTML

**State**

* Postgres (projects, files, versions, chats, messages)
* Redis (ephemeral stream state)
* S3 (exports/snapshots), optional Git push on commit

**Diagram**

```
[Next.js (chat, files, problems, preview)]
         │   ▲
   SSE   │   │  postMessage (errors, selected component)
         ▼   │
    [FastAPI + LangGraph Orchestrator]
      ├─ VFS (overlay apply + snapshot)
      ├─ TSC service (Node worker)
      └─ Preview services
           ├─ Sandpack/Vite (client sandbox)
           └─ Next Dev-Proxy (inject shim + component selector)
```

---

## 3) Language, libraries & filesystem conventions (for agents)

**All code generation must adhere to:**

* **Language**: TypeScript/TSX
* **Framework**: **Next.js App Router** (`app/` directory)
* **Styling/UI**: Tailwind CSS + **shadcn/ui** components
* **Imports**: ES Modules; **relative** paths inside the project
* **FS Layout** (examples):

  ```
  app/layout.tsx
  app/page.tsx                       # homepage
  app/pricing/page.tsx
  app/contact/page.tsx
  components/ui/*                    # shadcn re-exports
  components/sections/Hero.tsx
  lib/theme.ts
  styles/globals.css
  ```

**Strict output protocol for agents**: **Dyad-style tags** (documented below). Do **not** emit shell commands; to request packages, use `<dyad-add-dependency ...>` tags.

---

## 4) Output protocol (Dyad-style tags)

Agents **stream text** that includes structured tags. The server (or client) **parses completed tags** and applies them to the VFS. You must support at least these tags:

* **Write** a file:

  ````xml
  <dyad-write path="app/page.tsx" description="Add marketing hero">
  ```tsx
  // TSX content (code fences optional)
  ````

  </dyad-write>
  ```
* **Rename** a file:

  ```xml
  <dyad-rename from="components/Hreo.tsx" to="components/Hero.tsx"></dyad-rename>
  ```
* **Delete** a file:

  ```xml
  <dyad-delete path="components/OldBanner.tsx"></dyad-delete>
  ```
* **Add dependency** (optional, for package install orchestration):

  ```xml
  <dyad-add-dependency packages="lucide-react class-variance-authority"></dyad-add-dependency>
  ```

### 4.1 Exact parsing rules (copy & implement)

**Write tag regex + code-fence trimming** — **Copied verbatim from Dyad**
Use these expressions & trimming steps **exactly** to avoid common streaming pitfalls:

````ts
// Extracts <dyad-write ...> ... </dyad-write> with attributes
const dyadWriteRegex = /<dyad-write([^>]*)>([\s\S]*?)<\/dyad-write>/gi;
const pathRegex = /path="([^"]+)"/;
const descriptionRegex = /description="([^"]+)"/;

// Trim optional ``` fences from content
const contentLines = content.split("\n");
if (contentLines[0]?.startsWith("```")) { contentLines.shift(); }
if (contentLines[contentLines.length - 1]?.startsWith("```")) { contentLines.pop(); }
content = contentLines.join("\n");
````

(From Dyad’s `dyad_tag_parser.ts`. The **regex and fence-trimming** lines above must be implemented **as is**.)

**Rename tag regex** — **Copied verbatim from Dyad**

```ts
const dyadRenameRegex =
  /<dyad-rename from="([^"]+)" to="([^"]+)"[^>]*>([\s\S]*?)<\/dyad-rename>/g;
```



**Delete tag regex** — **Copied verbatim from Dyad**

```ts
const dyadDeleteRegex =
  /<dyad-delete path="([^"]+)"[^>]*>([\s\S]*?)<\/dyad-delete>/g;
```



**Add-dependency tag regex** — **Copied verbatim from Dyad**

```ts
const dyadAddDependencyRegex =
  /<dyad-add-dependency packages="([^"]+)">[^<]*<\/dyad-add-dependency>/g;
```



> **Note:** Normalize paths to a safe, project-relative format after extraction (Dyad calls `normalizePath(...)` before pushing tags).

---

## 5) Virtual Filesystem (VFS) overlay

Maintain an **overlay** of pending changes on top of the canonical project tree:

* `writes: Map<normalizedPath, content>`
* `deletes: Set<normalizedPath>`
* `renames: Map<from, to>` (apply as delete+write/move)
* `closedWrites: Set<path>` (optional guard for idempotency)

**Apply changes** (delete → rename → write):

> **Copied verbatim from Dyad (method structure & order):**
>
> ```
> // applyResponseChanges({ deletePaths, renameTags, writeTags })
> // 1) deletions → 2) renames → 3) writes
> ```
>
> (Structure and order gleaned from Dyad’s `applyResponseChanges` implementation; mimic this behavior to avoid inconsistencies.)

**Write file** — mark in `virtualFiles` and remove from `deletedFiles`.
**Delete file** — mark in `deletedFiles` and remove any virtual content.
**Rename** — move content if present; otherwise read from base FS and re-insert; mark old as deleted.
(**Mechanics match Dyad’s Base VFS; replicate semantics.**)

Provide helper APIs:

* `fileExists(path)`: consider deletes and overlay before base FS
* `readFile(path)`: prefer overlay content unless deleted
* `getVirtualFiles()` / `getDeletedFiles()` — return **relative** paths/content for export and preview rebuilds

---

## 6) Streaming protocol (SSE)

**Endpoint**: `GET /chats/{chat_id}/stream` → **Server-Sent Events**.

**Events**

* `tokens`: `{ text: string }` – raw assistant text for UI
* `patch`: `{ ops: Array<Write|Rename|Delete> }` – only when a tag **closes**
* `problems`: `ProblemReport` (after a tsc pass)
* `runtime_error`: runtime error payload (from preview shim)
* `done`: no payload

Process model output **incrementally**; buffer and parse tags; **emit a patch** only when a tag closes. (Dyad logs chunks as they arrive; you can follow the same read/decode loop for streaming bodies).

---

## 7) TypeScript problem loop (tsc → “concise fix” prompt → rewrite)

**Service**: A Node worker/container that accepts:

```ts
{
  virtualChanges: { deletePaths: string[], renameTags: {from,to}[], writeTags: {path, content, description?}[] },
  appPath: string,
  tsBuildInfoCacheDir?: string
}
```

Runs `tsc --noEmit` (or TypeScript API) against **materialized base + overlay** and returns:

```ts
type Problem = { file: string, line: number, column: number, message: string, code: number, snippet?: string }
type ProblemReport = { problems: Problem[] }
```

> **Controller shape (generateProblemReport)** — **Copied verbatim from Dyad (call graph & inputs)**
> Use the pattern where you **extract tags**, build `virtualChanges`, and post to a worker.
> (Re-create the structure below; names can differ, behavior should match.)

**Concise Fix Prompt** (format)
When problems exist, produce a short list for the Fixer agent, e.g.:

```
Fix these 3 TypeScript compile-time error(s):

1) src/components/Button.tsx:15:23 - Property 'onClick' does not exist on type 'ButtonProps'. (TS2339)
```

```tsx
SNIPPET
```

2. …

Please fix all errors in a concise way.

````

(Tests in Dyad show this concise formatting; implement equivalently.)

**Auto-fix loop**  
After each Writer pass → run TSC → if `problems.length > 0`, append a structured **problem report tag** to the assistant’s last output (optional, for traceability) and route to Fixer for another **tagged** patch. Limit to 2–3 attempts per turn (Dyad uses a cap of 2 in tests).

---

## 8) Preview services

### 8.1 Sandpack/Vite mode (default)
- Use VFS **snapshot** as the source for a **browser sandbox** (great for marketing pages and client-only code).
- Inject a minimal **shim** script into `index.html` that:
  - wraps `window.onerror` / `unhandledrejection`
  - posts `{ type: "window-error" | "unhandled-rejection", ... }` to `window.parent`
- Also inject the **component selector client** (see §9).

### 8.2 Next Dev-Proxy mode (SSR/RouteHandlers needed)
- Boot an ephemeral `next dev` (or `pnpm dev`) container from a **materialized snapshot** of the project.
- Front it with a **Node proxy** that **injects scripts** into HTML responses:
  - `stacktrace.js` (optional), **dyad-shim**, **component selector client**
- Only inject into root documents (e.g., `/` or `index.html`) to avoid double-injection.

> **HTML injection conditions & behavior** — **Copied verbatim from Dyad**  
> - Decide **when** to inject:  
>   `return pathname.endsWith("index.html") || pathname === "/";`:contentReference[oaicite:31]{index=31}
> - Avoid double-injection by detecting legacy shims, then **push `<script>` tags** into `<head>` (or prepend):  
>   (See the simplified flow below; replicate the control flow.):contentReference[oaicite:32]{index=32}:contentReference[oaicite:33]{index=33}:contentReference[oaicite:34]{index=34}

> **Resource loading (proxy worker)** — **Copied verbatim from Dyad**  
> Read `stacktrace.min.js`, `dyad-shim.js`, and `dyad-component-selector-client.js` and hold as strings to inject:contentReference[oaicite:37]{index=37}.

---

## 9) Component selection (picker overlay)

**Goal:** Let the user click any component in the preview; the iframe posts a message to the parent with a **stable component identifier** (e.g., `data-dyad-id="app/components/sections/Hero.tsx:Hero"` & `data-dyad-name="Hero"`). The next agent turn uses this to **focus edits** on the corresponding file/component.

**Client script requirements**
- Highlight hovered elements carrying `data-dyad-id`
- On click, `postMessage({ type: "dyad-component-selected", id, name })` to `window.parent`  
  (**Copied verbatim postMessage payload & types**)

**Parent frame (Next.js UI)**
- Listen for:
  - `"dyad-component-selector-initialized"` to confirm tags exist in the DOM
  - `"dyad-component-selected"` with `{ id, name }` → map to file path and set `selectedComponent` context for the next turn

> **Activation/deactivation messages** — **Copied verbatim from Dyad**  
> The parent sends `"activate-dyad-component-selector"` / `"deactivate-dyad-component-selector"` to the iframe; the injected client listens and toggles accordingly.

---

## 10) Backend: FastAPI surface

**Core endpoints**
- `POST /projects` → create project from template
- `GET /projects/{id}/files` → list (base + overlay)
- `POST /chats/{id}/stream` → **SSE** stream of tokens + `patch` + `problems`
- `POST /vfs/apply` → (optional if client parses) apply parsed ops server-side
- `POST /tscheck/run` → run `tsc` on (**base + overlay**)
- `POST /runtime-error` → forward iframe runtime errors into the graph
- `POST /commit` → write overlay to canonical files (+ optional Git)

**SSE handler sketch**
- Run the **LangGraph** turn
- While writing, **buffer** assistant text
- When a tag **closes**, parse → `applyResponseChanges` → emit `patch`
- After Writer pass, run TSC → emit `problems` → maybe loop to Fixer

---

## 11) Multi-agent graph (LangGraph)

**State**
```py
class BuildState(BaseModel):
    project_id: str
    turn_id: str
    user_prompt: str
    selected_component: Optional[str] = None  # e.g., "components/sections/Hero.tsx:Hero"
    vfs_overlay: VfsOverlay
    problems: list[Problem] = []
    runtime_errors: list[RuntimeError] = []
    plan: Optional[Plan] = None
    files_changed: list[FileChange] = []
    done: bool = False
````

**Nodes**

1. **Intake/Clarify** → extract pages, sections, brand tokens, constraints
2. **Planner** → produce file plan for **Next.js App Router** (directories & components)
3. **Writer** → emits **only** the tags in §4; never shell commands
4. **Typecheck** → call TSC service; returns `ProblemReport`
5. **Fixer** → if problems, generate **concise fix prompt** (see §7) and loop to Writer (cap 2–3)
6. **RuntimeFixer** (optional) → react to runtime errors from shim
7. **Committer** → persist overlay and versioning; optional Git push

**Edges**

```
Intake → Planner → Writer → Typecheck → (Fixer ↔ Writer ↔ Typecheck)* → RuntimeFixer? → Committer
```

**Agent tools (Python wrappers)**

* `fs.write(path, content)` / `fs.rename(from,to)` / `fs.delete(path)` → **MUST** serialize as `<dyad-*>` tags (transport invariant)
* `ts.check()` → returns `ProblemReport`
* `preview.focus(component_id)` → sets `selected_component` for next Writer pass

---

## 12) Frontend app (Next.js) details

**Chat UI**

* Calls `POST /chats/{id}/stream` and consumes **SSE** events
* Renders assistant tokens (nice, but **authoritative state** comes from `patch` events)
* Status pills: “writing files… running TypeScript checks…”

**File tree & code view**

* Show base files + overlay markers (pending writes, renames, deletes)
* Diff viewer for the last changed file(s)

**Problems panel**

* Render `ProblemReport` (file/line/code/message/snippet)
* “Fix with AI” button → triggers Fixer node (next graph step)

**Preview iframe**

* **Sandpack/Vite** by default
* Switch to **Dev-Proxy** (SSR) on demand
* Install message listeners for:

  * `"dyad-component-selector-initialized"` / `"dyad-component-selected"`
  * `"window-error"`, `"unhandled-rejection"`, `"iframe-sourcemapped-error"` → show to user & forward to backend

---

## 13) Data model (Postgres)

* `projects(id, owner_id, name, template, created_at)`
* `versions(id, project_id, name, created_at, base_version_id)`
* `files(id, project_id, version_id, path, content, hash, is_deleted, created_at, updated_at)`
* `chats(id, project_id, created_at)`
* `messages(id, chat_id, role, content, created_at)`
* `vfs_sessions(id, project_id, version_id, writes JSONB, deletes JSONB, renames JSONB, closed_writes JSONB)`
* `previews(id, project_id, mode enum['sandpack','devproxy'], status, url, created_at)`

---

## 14) Templates

Ship **Next.js (App Router) + Tailwind + shadcn** starter(s) the agents can expand:

* `app/layout.tsx` with `<ThemeProvider>`
* `app/page.tsx` with a minimal hero section
* `components/ui/*` re-exports (shadcn)
* `styles/globals.css` with Tailwind base

(Keep templates light; the Writer will scaffold sections on request.)

---

## 15) Security & ops

* **Sandbox** Node execution (TSC + Dev-Proxy) in containers; never execute untrusted code in the main web origin.
* **Path safety**: enforce a strict **safe join** within a project root; reject paths that escape the root (tests in Dyad’s suite show the importance of explicit guard rails).
* **Idempotency**: applying the same **closed** tag twice must be safe (track `closedWrites`).
* **Observability**: log every parsed tag, TSC error, and runtime error with `chat_id`/`turn_id`.
* **Cost control**: prune chat history; cap auto-fix attempts; summarize old turns.

---

## 16) System prompts (ready to use)

**Global system prompt (excerpt)**

> You are a code-writing agent for a Next.js App Router project using TypeScript, Tailwind, and shadcn/ui.
> **Only** modify the codebase via the following XML-like tags in your response:
>
> * `<dyad-write path="..."> ... code ... </dyad-write>`
> * `<dyad-rename from="..." to="..."></dyad-rename>`
> * `<dyad-delete path="..."></dyad-delete>`
> * `<dyad-add-dependency packages="..."></dyad-add-dependency>`
>
> Rules:
>
> 1. Code must compile with TypeScript and follow Next.js App Router conventions.
> 2. Use relative imports; do not call shell or package managers; request deps only via `<dyad-add-dependency>`.
> 3. Put **only** valid TS/TSX/CSS/JSON, etc., inside `<dyad-write>`. If you include \`\`\` fences, they will be removed.
> 4. Prefer small, targeted edits.
> 5. If the user has clicked a component, prioritize that file/component first.

**Fixer prompt (generated from ProblemReport)**

* Use the **concise** formatting described in §7, then:
  “Please fix all errors in a concise way, emitting only `<dyad-*>` tags.”

---

## 17) Milestones

**Week 1 — Foundations**

* VFS overlay + `applyResponseChanges` behavior (delete → rename → write)
* SSE streaming endpoint + server-side tag parser (regex + fence trimming)
* Minimal Writer that writes one file via `<dyad-write>`

**Week 2 — TypeScript & Problems**

* TSC worker container + `ProblemReport`
* Problems panel in UI; **auto-fix loop** (cap 2)

**Week 3 — Preview & Picker**

* Sandpack preview + shim for runtime errors
* Next Dev-Proxy with **HTML injection** (shim + selector)
* Component selector end-to-end (`activate`/`deactivate`, `component-selected`)

**Week 4 — Multi-agent & Ops**

* Planner/Writer/Fixer flow complete
* Commit + versioning + optional Git
* Logging, quotas, hardening (`safeJoin`)

---

## 18) Minimal reference implementations (snippets you can lift)

### 18.1 **Tag parsing (write, rename, delete, add-dependency)**

(**Copied verbatim from Dyad** — see §4.1 with citations.)
Implement these regexes and code-fence trimming exactly.

* Write: regex + fence removal
* Rename: regex
* Delete: regex
* Add dependency: regex

### 18.2 **VFS change application (order & semantics)**

Follow the **delete → rename → write** order and the `writeFile`, `deleteFile`, `renameFile` behaviors exactly (logic summarized in §5).
(Structure and call order drawn from Dyad’s VFS `applyResponseChanges` and helpers.)

### 18.3 **TSC worker controller shape**

Build the `generateProblemReport` controller that **extracts tags** and posts `virtualChanges` to a worker.
(**Copied verbatim pattern**)

### 18.4 **Dev-Proxy injection conditions**

Inject only on `index.html` or `/`, avoid double-injection, and push scripts into `<head>`.
(**Copied verbatim conditions & control flow**)

### 18.5 **Component selector messages**

Iframe to parent on selection:

```js
window.parent.postMessage({ type: "dyad-component-selected", id, name }, "*");
```

(**Copied verbatim**)

Parent to iframe to toggle picker:

```js
{ type: "activate-dyad-component-selector" } / { type: "deactivate-dyad-component-selector" }
```

(**Copied verbatim**)

---

## 19) API shape (FastAPI)

**POST /projects**

* body: `{ name, template }` → returns `{ project_id }` (materialize starter)

**GET /projects/{id}/files**

* returns base + overlay view

**POST /chats/{id}/stream** (SSE)

* body: `{ user_prompt, selected_component? }`
* stream events: `tokens`, `patch`, `problems`, `runtime_error`, `done`

**POST /tscheck/run**

* body: `{ project_id, virtualChanges }`
* returns: `ProblemReport`

**POST /runtime-error**

* body: `{ project_id, payload }` (mirrors shim events)

**POST /commit**

* body: `{ project_id, message? }` → writes overlay to canonical files (+ optional git ops)

---

## 20) Implementation guidance & gotchas

* **Parsing in the server**: Prefer server-side tag parsing (SSE transports both raw text and `patch` events). Keep the client parser only for UX hints.
* **Never apply half-open tags**: Only emit a `patch` when a tag **closes**.
* **Package requests**: Collect all `<dyad-add-dependency>` into a queue for a **post-run** install step (CI/build), not during chat.
* **SSR vs Sandpack**: Start on Sandpack; switch to Dev-Proxy when SSR/RouteHandlers are detected or requested.
* **AST mapping** *(optional)*: To improve “focus this component,” add a simple AST pass to map component names to file paths based on `data-dyad-id` or source maps.
* **Safe paths**: Enforce a strict base directory; reject paths with `..` that escape it (see §15).

---

## 21) Ready-to-paste examples for agents

**Create a Pricing page**

````xml
<dyad-write path="app/pricing/page.tsx" description="3-tier pricing with CTA">
```tsx
import { Button } from "@/components/ui/button";

export default function Page() {
  return (
    <main className="max-w-6xl mx-auto py-16">
      <section className="grid gap-8 md:grid-cols-3">
        {/* ...tiers... */}
      </section>
      <div className="mt-12 text-center">
        <Button>Start free</Button>
      </div>
    </main>
  );
}
````

</dyad-write>
```

**Fix a typo by renaming:**

```xml
<dyad-rename from="components/Hreo.tsx" to="components/Hero.tsx"></dyad-rename>
```

**Remove an obsolete component:**

```xml
<dyad-delete path="components/OldBanner.tsx"></dyad-delete>
```

**Request UI deps:**

```xml
<dyad-add-dependency packages="lucide-react class-variance-authority"></dyad-add-dependency>
```

---

### Final word

This spec is **implementation-complete** without Dyad’s code. Where exactness matters (tag parsing, injection conditions, selector messaging), we included **verbatim** Dyad snippets with citations so you can reproduce the behavior precisely.

