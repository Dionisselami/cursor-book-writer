# cursor-book-writer

**Write a book in Cursor.** A rule file, an MCP config, and a workflow that turns Cursor's agent
from a code assistant that can also write prose into a disciplined book-writing engine — via the
[Proseify](https://proseify.xyz) MCP server.

[![Listed on mcpservers.org](https://mcpservers.org/badge.svg)](https://mcpservers.org/servers/dionisselami/proseify-mcp) [![MCP Registry](https://img.shields.io/badge/MCP_registry-io.github.Dionisselami%2Fproseify--mcp-4a3f35)](https://registry.modelcontextprotocol.io/v0/servers?search=proseify)

---

## Why a rule file matters more than the MCP connection

Connecting an MCP server in Cursor takes thirty seconds. The problem is what happens next: the
agent calls `plan_book`, receives a good outline, and then writes chapter one in its own default
voice — because nothing told it that the genre recipe read *before* prose is the entire point.

`.cursor/rules/book-writing.mdc` fixes that. It is frontmatter-scoped (`globs` on your manuscript
paths) with `alwaysApply: false`, so it loads when you are inside the book and keeps its context
cost out of your code sessions.

```text
premise  →  genre recipe  →  corpus passages  →  outline  →  chapter drafts  →  evaluate_book  →  revision
```

Never let the model start at "chapter drafts". The ordering is the whole product: a genre
recipe read *before* prose beats any amount of "write like Stephen King" prompting, because it
hands the model concrete numbers — pacing beats, dialogue ratio, sentence-length profile — that
its defaults would otherwise flatten to its own house style.

---

## Quick start

### 1. Get a Proseify key

https://proseify.xyz — sign in, pick a plan (from $19/mo, or the one-time Founding Lifetime tier),
key issued on payment.

### 2. Put the key in your environment

```bash
export PROSEIFY_API_KEY="sk-..."        # macOS / Linux, in your shell profile
[Environment]::SetEnvironmentVariable("PROSEIFY_API_KEY","sk-...","User")   # Windows PowerShell
```

Then restart Cursor: apps read their environment at launch, and a key set afterwards will not be
seen.

### 3. Connect the MCP server

Cursor → Settings → MCP → **Add new MCP server**, or edit `.cursor/mcp.json` directly (already
written in this repo):

```json
{
  "mcpServers": {
    "proseify": {
      "url": "https://mcp.proseify.xyz/mcp",
      "headers": { "Authorization": "Bearer ${env:PROSEIFY_API_KEY}" }
    }
  }
}
```

The server should show green in the MCP panel with seven tools. A red dot is almost always the
environment variable, not the URL.

### 3. Install the writing skill

```bash
npx skills add https://proseify.xyz --skill anti-prose-slop -y
```

The MCP server gives the agent the *tools*. The `anti-prose-slop` skill gives it the *method* —
genre recipe first, corpus grounding, chapter discipline, and the per-chapter edit pass. Install
both or you get tool access and the same generic prose you had before.

### 5. Open a book folder and give it a premise

```bash
mkdir -p my-book/chapters && cd my-book && cursor .
```

> A literary mystery set in a flooded city: a translator realises the documents she is paid to
> translate describe crimes that have not happened yet. 24 chapters, 40,000 words.

---

## What's in here

| File | Purpose |
|------|---------|
| `.cursor/mcp.json` | Proseify MCP server, key read from the environment |
| `.cursor/rules/book-writing.mdc` | The book-writing rule, scoped to manuscript globs |
| `LICENSE` | MIT |

### The five-step loop

1. **Pick the genre before any prose exists.** `list_genres`, then `get_genre_recipe` for the
   closest fit. Blends are allowed — name both and let the recipe argue for one.
2. **Ground the register.** `search_corpus` (or `get_style_references`) for 3–5 model passages
   in the target genre. Read them. This is the calibration step, not decoration.
3. **`plan_book` once.** Hold the outline. Do not re-plan mid-draft; if the outline is wrong,
   fix it deliberately and say so.
4. **Draft chapter by chapter**, pausing after each for the per-chapter edit pass below.
5. **`evaluate_book` before you call it done.** Fix the chapters it flags and re-run the gate.

### The per-chapter edit pass

The failure mode of AI prose is not grammar — it is sameness. Every chapter gets struck against
this list before it counts as drafted:

- `seemed to`, `began to`, `started to`, `could feel` — delete or recast
- filter words: *felt, noticed, watched, saw, heard* — put the reader in the perception instead
- stacked adverbs on dialogue tags; `said` is not a problem, `said loudly, angrily` is
- sentence openings repeated across the chapter (and the weather-opening default)
- three-item lists used as rhythm filler
- dialogue that exists to explain the plot to the reader
- simile endings: the last line becoming a metaphor for the chapter

### If you only read one paragraph

The agent's own model does all the writing — Proseify calls no LLM and stores no manuscript.
There is no token bill from us: it is a corpus, a set of genre recipes, and a pipeline.


---

## Cursor notes and gotchas

- **`alwaysApply: false` plus globs is deliberate.** A book rule that loads in every TypeScript
  session spends context on work that has nothing to do with prose.
- **Cursor's MCP panel reports connection state, not auth success.** If the tools are missing behind
  a green light, check the environment of the running app rather than the JSON.
- **If your Cursor build does not interpolate `${env:PROSEIFY_API_KEY}`,** replace the header value
  with your literal key, add `.cursor/mcp.json` to `.gitignore` before you save it, and never
  paste the file into a chat. The env-var form is the default here for that reason.
- **Restart Cursor after editing `.cursor/mcp.json`.** Hot reload is unreliable, and a stale tool
  list looks exactly like an auth failure.
- **Keep the rule short.** Long rules get truncated in long agent sessions, and the one thing that
  must never be lost is the *order*: recipe → corpus → plan → draft → gate.
- **Write chapters to disk as you go.** Cursor's context does not survive a large manuscript;
  `chapters/NN-title.md` does, and it makes the book reviewable in the editor beside the agent.
- **A 40,000-word book is not one prompt.** Draft chapter by chapter with a stated budget; the
  one-shot `write_book` call is for novellas and short books.

| Tool | What it does |
|------|--------------|
| `list_genres` | The 11 genres the corpus covers (theatre, horror, romance, adventure, literary, mystery, gothic, sci-fi, fantasy, comedy, children's) |
| `get_genre_recipe` | Pacing beats, dialogue ratio, sentence-length profile and stylistic anchors derived from that tradition |
| `search_corpus` | FTS5 full-text search across 2,500+ chapters of public-domain classics (quoted phrases, AND/OR/NOT, wildcards) |
| `get_style_references` | Model passages from books in the target genre — the register you are aiming at |
| `plan_book` | A full chapter-by-chapter outline from a one-line premise, each beat carrying a corpus style reference |
| `write_book` | The one-shot flow: plan → draft → evaluate, in a single session |
| `evaluate_book` | Scores a draft against genre benchmarks (structure, pacing, chapter coverage, word budget) so weak chapters get revised |

## The corpus

80+ books and 2,500+ chapters of public-domain literature — Project Gutenberg and similar
sources — every chapter verified against its own file header at download time, so nothing in
the library has a copyright question hanging over it. Works from Austen, Stevenson, Hugo,
Conrad, Verne, the Brontës, Shelley and the gothic masters, grouped by genre and indexed for
full-text search.

Commercial use of what you write on top of it is safe. Full-text searchable chapter by chapter,
not a scrape of titles.

## Pricing

| Plan | Price | Rate limit |
|------|-------|-----------|
| Quill (Starter) | $19/mo | 120 req/min |
| Fable (Pro) | $49/mo | 400 req/min |
| Opus (Studio) | $99/mo | unlimited (fair use) |
| Founding Lifetime | $149 one-time | 400 req/min, all genres |

Sign in at https://proseify.xyz, pick a plan, and the key is issued the moment the purchase
clears. Cancel from https://proseify.xyz/account; 14-day refund window
(https://proseify.xyz/refunds).

## Other clients

Same Proseify server, same workflow, different config file. The cookbook is the method itself.

- [claude-book-writer — write a book with Claude Code](https://github.com/Dionisselami/claude-book-writer)
- [chatgpt-book-writer — write a book on your ChatGPT plan](https://github.com/Dionisselami/chatgpt-book-writer)
- [gemini-cli-book-writer — write a book with Gemini CLI](https://github.com/Dionisselami/gemini-cli-book-writer)
- [copilot-book-writer — write a book in VS Code with Copilot](https://github.com/Dionisselami/copilot-book-writer)
- [windsurf-book-writer — write a book in Windsurf](https://github.com/Dionisselami/windsurf-book-writer)
- [claude-book-cookbook — recipes for writing a whole book with Claude](https://github.com/Dionisselami/claude-book-cookbook)

## License

MIT for everything in this repository. The corpus texts themselves are public domain.

## Disclaimer

Unofficial. This repository is not affiliated with, endorsed by, or sponsored by Anysphere or Cursor. Client names appear only to describe which configuration file and transport the instructions are for. Proseify is an independent product — https://proseify.xyz.

Keep your Proseify key in local agent config or an environment variable. Never commit it, print it, log it, or paste it into a repository file.

## Links

- **Sign up / key issuance:** https://proseify.xyz
- **Filled-in config for your key:** https://proseify.xyz/agent
- **FAQ (ownership, KDP, what an MCP server is):** https://proseify.xyz/faq
- **Server repo & MCP registry entry:** https://github.com/Dionisselami/proseify-mcp
- **Support:** support@proseify.xyz
