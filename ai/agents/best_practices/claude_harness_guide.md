# Harnessing Claude’s intelligence
https://claude.com/blog/harnessing-claudes-intelligence

## 1. Use what Claude knows

- ⬜ Prefer tools that Claude already understands deeply (e.g., bash, text editor)
- ⬜ Avoid designing bespoke agent tools when general tools suffice
- ⬜ Leverage tool patterns Claude has demonstrated strong performance with
- ⬜ Build higher-level abstractions by composing known tools rather than replacing them
- ⬜ Reuse existing Claude-supported primitives (tools, skills, memory) rather than duplicating logic

---

## 2. Ask “what can I stop doing?”

### Let Claude orchestrate its own actions

- ⬜ Avoid routing every tool result back through the context window
- ⬜ Remove harness-level assumptions about how tool outputs must be processed
- ⬜ Give Claude a code execution environment (e.g., bash, REPL)
- ⬜ Allow Claude to decide which tool outputs to filter, pass through, or ignore
- ⬜ Minimize token usage by letting tools communicate directly where possible
- ⬜ Shift orchestration logic from the harness to Claude-written code

---

### Let Claude manage its own context

- ⬜ Reduce reliance on large, hand-crafted system prompts
- ⬜ Avoid pre-loading rarely used task instructions into context
- ⬜ Provide access to skills with lightweight frontmatter descriptions
- ⬜ Allow Claude to selectively load full skill content when needed
- ⬜ Use context editing to remove stale or irrelevant context
- ⬜ Allow Claude to fork work into subagents when tasks should be isolated

---

### Let Claude persist its own context

- ⬜ Avoid hard-coding external memory retrieval logic where possible
- ⬜ Enable compaction so Claude can summarize and carry forward key information
- ⬜ Allow Claude to decide what information should be remembered
- ⬜ Provide a memory folder Claude can write to and read from
- ⬜ Let Claude structure persistent memory into files and directories
- ⬜ Review and prune memory mechanisms as Claude’s native capabilities improve

---

## 3. Set boundaries carefully

### Design context to maximize cache hits

- ⬜ Structure prompts so repeated portions remain identical between turns
- ⬜ Introduce clear prompt cache breakpoints
- ⬜ Place stable instructions and tool definitions before cache boundaries
- ⬜ Minimize changes to early prompt segments to increase cache reuse
- ⬜ Optimize prompts to benefit from discounted cached-token pricing

---

### Use declarative tools for UX, observability, or security boundaries

- ⬜ Promote high-risk or irreversible actions into dedicated tools
- ⬜ Gate sensitive actions behind explicit tool-level checks
- ⬜ Require user confirmation for actions that cross security boundaries
- ⬜ Add staleness or safety checks directly into write or edit tools
- ⬜ Use typed tool arguments for logging, auditing, and replay
- ⬜ Render tools in the UI when user interaction or clarity is required
- ⬜ Periodically reassess whether actions should remain tools or return to general execution
- ⬜ Use secondary model review only where users explicitly trust broader autonomy

---

## Looking forward

- ⬜ Re-test assumptions about Claude’s limitations after each major model release
- ⬜ Remove harness logic added to compensate for behaviors that no longer exist
- ⬜ Audit agent harnesses for “dead weight” introduced by older models
- ⬜ Continuously ask: *what can I stop doing?*
- ⬜ Prune constraints that bottleneck Claude’s improving capabilities
