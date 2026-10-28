# Anthropic's official guidance on orchestrator-subagent systems and orchestrator context bloat

Scope: first-party Anthropic sources only (anthropic.com/engineering, claude.dev blog, code.claude.com docs, platform.claude.com API docs). Researched 2026-10-28. Claude Code / Agent SDK docs are living pages with no publication date; where a behaviour is version-gated the doc's version number is given so the reader can tell how recent it is. Quotes are marked with quotation marks and are under 25 words; everything else is paraphrase.

Sources used, with dates:

| Source | Date | URL |
| --- | --- | --- |
| "How we built our multi-agent research system" | 2025-06-13 | https://www.anthropic.com/engineering/multi-agent-research-system |
| "Effective context engineering for AI agents" | 2025-09-29 | https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents |
| "Building effective agents" | 2024-12-19 | https://www.anthropic.com/engineering/building-effective-agents |
| "Writing tools for agents" | 2025-09-11 | https://www.anthropic.com/engineering/writing-tools-for-agents |
| "Lessons from building Claude Code: Prompt caching is everything" (Thariq Shihipar) | 2026-04-30 | https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/ |
| Claude Code docs: Subagents | live, undated | https://code.claude.com/docs/en/sub-agents |
| Claude Code docs: Agent teams | live, undated | https://code.claude.com/docs/en/agent-teams |
| Claude Code docs: Dynamic workflows | live, undated | https://code.claude.com/docs/en/workflows |
| Claude Code docs: Manage costs | live, undated | https://code.claude.com/docs/en/costs |
| Claude Code docs: Best practices (the old anthropic.com/engineering/claude-code-best-practices URL now 308-redirects here) | live, undated | https://code.claude.com/docs/en/best-practices |
| Claude Code docs: Explore the context window | live, undated | https://code.claude.com/docs/en/context-window |
| Claude Code docs: How Claude Code uses prompt caching | live, undated | https://code.claude.com/docs/en/prompt-caching |
| Agent SDK docs: Subagents in the SDK | live, undated | https://code.claude.com/docs/en/agent-sdk/subagents |
| Claude API docs: Prompt caching | live, undated | https://platform.claude.com/docs/en/build-with-claude/prompt-caching |
| Claude API docs: Rate limits | live, undated | https://platform.claude.com/docs/en/api/rate-limits |

## Q1. The multi-agent research system post (June 2025): orchestrator-worker, condensed returns, the ~15x figure, long-running agents

### Takeaway
Anthropic's own research product is an orchestrator-worker system whose subagents each run in their own context window and hand the lead agent condensed findings or lightweight references to externally stored output, never raw transcripts; the post is explicit that this costs about 15x the tokens of a chat and is only worth it for high-value, parallelisable tasks, and that the lead agent persists its plan to memory because context past 200k tokens gets truncated.

### Cited Findings
- Architecture: the system uses an "orchestrator-worker pattern, where a lead agent coordinates the process while delegating to specialized subagents." The lead plans the work, then spawns parallel subagents for different aspects of the query. — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Separate contexts as compression: subagents run in parallel with their own context windows, and the post says this helps "facilitate compression" by condensing the most important tokens for the lead agent. Each subagent has its own tools, prompt and trajectory. — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- What comes back: subagents return condensed findings (the worked example is a subagent returning "a list of companies to the lead agent so it can compile a final answer"), not raw material. — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Reference passing (appendix): subagents can store outputs in external systems and "pass lightweight references back to the coordinator", so large outputs never travel through conversation history. — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Token cost: agents use about 4x the tokens of a chat interaction, and "multi-agent systems use about 15× more tokens than chats." — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Token spend predicts quality: on BrowseComp, "token usage by itself explains 80% of the variance" in performance. — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Headline result: a lead Opus 4 with Sonnet 4 subagents beat single-agent Opus 4 by 90.2% on an internal research eval. — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- When NOT to use multi-agent: "multi-agent systems require tasks where the value of the task is high enough" to justify the cost; domains needing shared context or many inter-agent dependencies are "not a good fit for multi-agent systems today"; coding has "fewer truly parallelizable tasks than research." — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Context limit handling: the lead agent saves its plan to Memory "to persist the context" because a context window over 200,000 tokens is truncated. For long conversations, agents summarise completed phases and store essential information externally. — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Fresh contexts for long runs: as limits approach, agents can "spawn fresh subagents with clean contexts while maintaining continuity through careful handoffs." — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Parallelism: the lead spins up "3-5 subagents in parallel rather than serially"; subagents use "3+ tools in parallel"; together this cut research time "by up to 90% for complex queries." — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Effort scaling rules embedded in the lead prompt: simple fact-finding = 1 agent with 3-10 tool calls; direct comparisons = 2-4 subagents with 10-15 calls each; complex research = more than 10 subagents with clearly divided responsibilities. — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Failure mode they hit: early versions were "spawning 50 subagents for simple queries." — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Known bottleneck is synchrony, not compression: "Currently, our lead agents execute subagents synchronously," so the lead cannot steer mid-flight and the system waits on the slowest subagent. The post frames compression into the lead as a benefit, not a bottleneck. — [Multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)

### Inferences
- The post's design has three separate mechanisms against lead-agent bloat: (1) subagents own the raw tokens, (2) returns are condensed or are references, (3) the lead externalises its plan. All three are needed; condensation alone does not stop a lead that spawns dozens of subagents from filling up with summaries.
- The "4x per agent, 15x per multi-agent system" figures imply the orchestrator itself is a small share of spend; the subagents' exploration dominates. Bloat in the orchestrator is primarily a latency/cache/quality problem (see Q7), not the main cost driver.

### Gaps
- The post does not give a target size for a subagent's return message; that number appears only in the context-engineering post (Q2).
- The post does not say how the lead's memory persistence is implemented beyond "Memory".

## Q2. "Effective context engineering for AI agents" (Sept 2025): compaction, note-taking, sub-agents as isolation, just-in-time retrieval, 1,000-2,000-token summaries

### Takeaway
This post names three long-horizon techniques (compaction, structured note-taking, sub-agent architectures), explains that sub-agents let one agent burn tens of thousands of tokens while returning only a 1,000-2,000-token distilled summary, and frames all of it around a finite "attention budget" that degrades as context grows ("context rot").

### Cited Findings
- Definition: context engineering is "the set of strategies for curating and maintaining the optimal set of tokens (information) during LLM inference." — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Context rot: as the window grows, "the model's ability to accurately recall information from that context decreases." LLMs have an "attention budget" that every token depletes, so context must be treated as a finite resource. — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Guiding principle: "find the smallest set of high-signal tokens that maximize the likelihood of your desired outcome." — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Just-in-time retrieval: agents should "maintain lightweight identifiers (file paths, stored queries, web links, etc.)" and load data at runtime via tools instead of pre-loading; a hybrid is suggested because runtime exploration is slower. Claude Code is cited as the example: "CLAUDE.md files are naively dropped into context up front," while glob/grep fetch files just-in-time. — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Compaction: "the practice of taking a conversation nearing the context window limit," summarising it, and restarting with the summary. Advice: maximise recall first, then precision. Keep "architectural decisions, unresolved bugs, and implementation details"; drop "redundant tool outputs or messages." Tool-result clearing is "one of the safest lightest touch forms of compaction." — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Claude Code's compaction mechanics: it compacts by "passing the message history to the model to summarize and compress the most critical details," then continues with the compressed context "plus the five most recently accessed files." — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Structured note-taking (agentic memory): the agent writes notes "persisted to memory outside of the context window" and pulls them back later. Examples: "Claude playing Pokémon" keeping tallies and maps, and an agent "maintaining a NOTES.md file." — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Sub-agent architectures: a subagent may explore "using tens of thousands of tokens or more" but returns "only a condensed, distilled summary", often 1,000-2,000 tokens; this "achieves a clear separation of concerns." — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Which technique when: compaction for tasks "requiring extensive back-and-forth"; note-taking for "iterative development with clear milestones"; multi-agent for complex research where "parallel exploration pays dividends." — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Tool design: tools should return "information that is token efficient"; avoid bloated, overlapping tool sets and curate "a minimal viable set of tools." — [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

### Inferences
- The 1,000-2,000-token return figure is the only first-party number for subagent report size. With a 20-way fan-out that is 20-40k tokens of summaries entering the orchestrator per round, which is the arithmetic behind the Claude Code warning in Q5 that many detailed results "can consume significant context".
- "Context rot" is Anthropic's stated reason that a bloated orchestrator gets worse, not just slower: recall over the accumulated summaries degrades.

### Gaps
- No first-party figure for how much quality degrades per unit of context length; "context rot" is described qualitatively.

## Q3. "Building effective agents" (Dec 2024): orchestrator-workers vs parallelization vs routing, and start simple

### Takeaway
The post's core advice is to find the simplest solution and add complexity only when needed, because agentic systems "trade latency and cost for better task performance"; orchestrator-workers is reserved for tasks where subtasks cannot be predicted up front, otherwise plain parallelization (sectioning or voting) or routing is enough.

### Cited Findings
- Simplicity first: "we recommend finding the simplest solution possible, and only increasing complexity when needed." — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- The trade: "Agentic systems often trade latency and cost for better task performance." — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Workflows vs agents: "Workflows are systems where LLMs and tools are orchestrated through predefined code paths"; "Agents, on the other hand, are systems where LLMs dynamically direct their own processes and tool usage". — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Routing: for "complex tasks where there are distinct categories that are better handled separately." — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Parallelization has two forms: sectioning, "Breaking a task into independent subtasks run in parallel," and voting, "Running the same task multiple times to get diverse outputs." Use when "multiple perspectives or attempts are needed for higher confidence results." — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Orchestrator-workers: "a central LLM dynamically breaks down tasks, delegates them to worker LLMs, and synthesizes their results." Use "where you can't predict the subtasks needed". The difference from parallelization is that "subtasks aren't pre-defined, but determined by the orchestrator based on the specific input". — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Frameworks: they "create extra layers of abstraction that can obscure the underlying prompts"; "developers start by using LLM APIs directly." — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Three principles: "Maintain simplicity in your agent's design"; "Prioritize transparency by explicitly showing the agent's planning steps"; "Carefully craft your agent-computer interface (ACI) through thorough tool documentation and testing." — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Tool docs: "A good tool definition often includes example usage, edge cases, input format requirements"; write it like "a great docstring for a junior developer on your team"; and "Poka-yoke your tools. Change the arguments so that it is harder to make mistakes." — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)

### Inferences
- Read together with Q5's workflows doc, Anthropic's current position is that if the subtasks are predictable (a file list, a set of research angles), a scripted fan-out (a "workflow" in the 2024 vocabulary) beats an LLM orchestrator precisely because the orchestrator's context is not used as the accumulator.

### Gaps
- This post predates the token-cost figures; it gives no numbers.

## Q4. "Writing tools for agents" (Sept 2025): tool-result size, pagination, truncation, response_format

### Takeaway
Tool results are the main input that bloats an agent; Anthropic's rule is to return only high-signal content, default to pagination/filtering/truncation (Claude Code caps tool responses at 25,000 tokens), offer a concise-vs-detailed response_format, and when truncating, tell the agent how to narrow its next call.

### Cited Findings
- Return "only high signal information back to agents"; prioritise contextual relevance over flexibility; avoid low-level identifiers like UUIDs and MIME types in favour of names. — [Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- For tools that can return a lot, implement "pagination, range selection, filtering, and/or truncation with sensible default parameter values". — [Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- Claude Code's default: "we restrict tool responses to 25,000 tokens by default." — [Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- response_format enum: concise vs detailed. In the Slack example the detailed response is 206 tokens and the concise one 72 tokens, "~⅓ of the tokens". — [Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- Truncation must steer: "If you choose to truncate responses, be sure to steer agents with helpful instructions." Encourage "making many small and targeted searches instead of a single, broad search". Errors should be actionable, not opaque codes. — [Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- Consolidate tools: one `schedule_event` instead of `list_users` + `list_events` + `create_event`; `search_logs` instead of `read_logs`; `get_customer_context` instead of three lookups. — [Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- Namespacing by service or resource helps the model pick among overlapping tools; prefix vs suffix naming had "non-trivial effects" on evals and the best choice varies by model. — [Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)

### Inferences
- The Agent tool that returns a subagent's report is itself a tool result; the same advice (return condensed, make the caller ask for more) applies to the subagent's final message, which is why the Claude Code docs tell users to "Ask for short summaries back" (Q5).

### Gaps
- The post gives no guidance specific to the size of an Agent/subagent tool result, only to tools in general.

## Q5. Claude Code docs: subagents, agent teams, workflows, costs, best practices, context window

### Takeaway
In Claude Code a subagent runs in its own context window, starts without the parent's history, and only its final message (plus a small metadata trailer) enters the parent; its transcript is stored in a separate file and never enters the parent context. The docs now say subagents are mainly for parallelism, isolating noisy output and independent review, warn that many detailed returns still bloat the main context, cap concurrency at 20, and point to scripted "dynamic workflows" (up to 1,000 agents per run, intermediate results in script variables) when a conversation cannot coordinate the fan-out.

### Cited Findings

Subagents (what enters the parent, and when to use them)
- "Each subagent runs in its own context window with a custom system prompt, specific tool access, and independent permissions." — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Starts clean: "It doesn't see your conversation history, the skills you've already invoked, or the files Claude has already read." A non-fork subagent gets its own system prompt, the task message, CLAUDE.md files, a git status snapshot and any preloaded skills. — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Only the summary returns: the subagent "does that work in its own context and returns only the summary." — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Transcript isolation: "when the main conversation compacts, subagent transcripts are unaffected. They're stored in separate files." Transcripts live at `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl`, deleted after `cleanupPeriodDays` (30 days default). — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Context-window page confirms the mechanism: a subagent handles research "in its own separate context window, so the large file reads stay out of yours. Only the summary and a small metadata trailer come back." — [Explore the context window](https://code.claude.com/docs/en/context-window)
- Stated benefit: "Preserve context by keeping exploration and implementation out of your main conversation." Use case "Isolate high-volume operations" (example: run tests, report only failures). — [Subagents](https://code.claude.com/docs/en/sub-agents)
- The warning: "Running many subagents that each return detailed results can consume significant context, and each subagent spends tokens of its own while it runs." — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Usage: subagent requests "count toward the same usage limits as your main conversation." — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Main thread vs subagent: use the main conversation when "The task needs frequent back-and-forth or iterative refinement" and when "Latency matters. A subagent that isn't a fork starts fresh and may need time to gather context." Use a subagent when "The task produces verbose output you don't need in your main context" or "You want to enforce specific tool restrictions or permissions." — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Foreground vs background: "Foreground subagents block the main conversation until complete." "Background subagents run concurrently while you continue working." "A background subagent's results reach Claude as a completion notification in a later turn." Background subagents get a smaller built-in tool set. — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Nesting (contradicts the older "subagents cannot spawn subagents" rule): "By default, a subagent can spawn subagents of its own, up to three layers below the main conversation." `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=1` turns nesting off. — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Concurrency cap: 20 concurrent subagents by default; beyond that spawning fails with `Concurrent subagent limit reached`; `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` changes it (v2.1.217+). "There's no limit on the total number of subagents Claude can spawn over a session." — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Subagents compact themselves: "Compaction triggers as the subagent's own context window fills"; `autoCompactWindow` frontmatter (100,000-1,000,000 tokens, v2.1.296+) makes one subagent compact earlier; the transcript logs `compact_boundary` with `preTokens`. — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Forks: a fork "inherits the entire conversation so far instead of starting fresh", but "The fork's own tool calls still stay out of your conversation and only its final result comes back, so your main context window stays clean." Because its prefix is identical to the parent, "a fork costs less than a fresh subagent for tasks that need the same context." A fork can't spawn forks. Fork mode is on by default from v2.1.302. — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Resume: "Each subagent invocation creates a new instance rather than continuing an earlier one"; resumed subagents "retain their full conversation history"; Claude resumes via `SendMessage`. — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Built-in Explore agent starts without CLAUDE.md or git snapshot "to keep research fast and inexpensive." — [Subagents](https://code.claude.com/docs/en/sub-agents)
- Delegation routing budget: keep the combined subagent descriptions within the "15,000-token description budget". — [Subagents](https://code.claude.com/docs/en/sub-agents)

Best practices page (what subagents cost, and what advice is now retired)
- Defaults: "To keep your main conversation focused while independent work happens in parallel, Claude delegates to subagents and checks in only when it has to." — [Best practices](https://code.claude.com/docs/en/best-practices)
- Subagents pay off for parallel work, "Keeping long output, like a full test run or a wide search, out of your main conversation", a reviewer that hasn't seen the code written, and fewer tools. — [Best practices](https://code.claude.com/docs/en/best-practices)
- Their cost: "What they cost you is mostly tokens, since every subagent sends its own requests"; "a fresh subagent first has to gather context your conversation already had, so it can end up slower than doing the task where you are." "You get parallelism or a fresh pair of eyes out of it, but not a smarter answer." — [Best practices](https://code.claude.com/docs/en/best-practices)
- To spend less: `model: haiku` for simple subagents; "Ask for short summaries back, so twenty results don't fill your context anyway."; check `/usage` per subagent type; restrict tools. — [Best practices](https://code.claude.com/docs/en/best-practices)
- Why usage climbs: "Every request sends the whole conversation, so a session that's collected three unrelated tasks re-reads all three on every turn of the fourth." Discounted while the cache is warm, "full price once it expires." — [Best practices](https://code.claude.com/docs/en/best-practices)
- Retired advice table: "The context window is your main constraint, manage it aggressively" is gone because "Most current models have a 1M-token window and compaction happens on its own, so running out is rare." "Use subagents to protect your context window" is gone because "Running out of context is rarely the problem anymore. Subagents are still great for parallel work, keeping noisy output out of your way, and independent review, but each of those costs extra tokens." "Delegate to a subagent for a better answer": "Handing the same task to a subagent doesn't make the answer any better." — [Best practices](https://code.claude.com/docs/en/best-practices)
- `/clear` "costs nothing"; `/compact` "is itself a big request, since it reads the whole conversation to summarize it, and even more so once the cache has expired." Also "Keep an eye on background work: a /loop, a goal's check after every turn, and Claude's own subagents all keep sending requests while you aren't typing." — [Best practices](https://code.claude.com/docs/en/best-practices)
- Idle compaction: "On the one-hour cache, a session past about 200K tokens that sits idle is compacted shortly before the cache expires". — [Best practices](https://code.claude.com/docs/en/best-practices)
- Parallel sessions: `claude -w <name>` starts a session in a worktree; `/batch` splits one mechanical change "into 5 to 30 units", each in its own worktree with its own PR; a second session "is also the easiest independent reviewer you can get." — [Best practices](https://code.claude.com/docs/en/best-practices)

Agent teams (lead + peer sessions)
- Status: "Agent teams are experimental and disabled by default" (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`). — [Agent teams](https://code.claude.com/docs/en/agent-teams)
- Cost comparison table: subagents are "Lower: results summarized back to main context"; teams are "Higher: each teammate is a separate Claude instance". "Agent teams add coordination overhead and use significantly more tokens than a single session." — [Agent teams](https://code.claude.com/docs/en/agent-teams)
- Lead context: teammates get the spawn prompt only; "The lead's conversation history does not carry over." When a teammate finishes it "automatically notifies the lead and includes its final answer in the notification." Messages are delivered automatically; "The lead doesn't need to poll for updates." — [Agent teams](https://code.claude.com/docs/en/agent-teams)
- Sizing: "Token costs scale linearly"; "Start with 3-5 teammates for most workflows"; "If you have 15 independent tasks, 3 teammates is a good starting point"; "Three focused teammates often outperform five scattered ones"; 5-6 tasks per teammate. — [Agent teams](https://code.claude.com/docs/en/agent-teams)
- Task sizing: "Too small: coordination overhead exceeds the benefit"; "Too large: teammates work too long without check-ins". — [Agent teams](https://code.claude.com/docs/en/agent-teams)
- The costs page adds a figure: "Agent teams use approximately 7x more tokens than standard sessions when teammates run in plan mode". — [Manage costs](https://code.claude.com/docs/en/costs)
- Limits: no nested teams, one team per session, lead is fixed, in-process teammates cannot run background subagents. — [Agent teams](https://code.claude.com/docs/en/agent-teams)

Dynamic workflows (orchestration outside the context window)
- Definition: "A dynamic workflow is a JavaScript script that orchestrates many subagents at once," executed by a runtime in the background. — [Dynamic workflows](https://code.claude.com/docs/en/workflows)
- The key architectural statement: with subagents, skills and teams "Claude is the orchestrator: it decides turn by turn what to spawn or assign next, and every result goes into a context window. A workflow script holds the loop, the branching, and the intermediate results itself, so Claude's context holds only the final answer." — [Dynamic workflows](https://code.claude.com/docs/en/workflows)
- Comparison table: where intermediate results live is "Claude's context window" for subagents and skills, "A shared task list" for teams, "Script variables" for workflows; scale is "A few delegated tasks per turn" for subagents vs "Dozens to hundreds of agents per run" for workflows. — [Dynamic workflows](https://code.claude.com/docs/en/workflows)
- Runtime isolation: "The workflow runtime executes the script in an isolated environment, separate from your conversation. Intermediate results stay in script variables instead of landing in Claude's context." — [Dynamic workflows](https://code.claude.com/docs/en/workflows)
- Limits: up to 16 concurrent agents by default (`CLAUDE_CODE_WORKFLOW_MAX_CONCURRENT_AGENTS`, 1-256, v2.1.269+); up to 4,096 items per `parallel()`/`pipeline()`; "1,000 agents total per run"; no mid-run user input. — [Dynamic workflows](https://code.claude.com/docs/en/workflows)
- Cost signals: a `Large workflow` warning when a run schedules more than 25 agents or "its projected token total passes 1.5 million". Size guideline: small = fewer than 5 agents, medium = fewer than 10, large = fewer than 50; default medium (small on Pro). Advice: "run the workflow on a small slice first". — [Dynamic workflows](https://code.claude.com/docs/en/workflows)
- Fan-out cache sharing: agents with the same model, effort, type, tools, schema and directory share a prefix; Claude Code holds all but the first for up to 5,000 ms (`CLAUDE_CODE_WORKFLOW_PREFIX_STAGGER_MS`) so siblings read the first agent's cache. — [Dynamic workflows](https://code.claude.com/docs/en/workflows)
- Usage-limit behaviour: when an agent hits the claude.ai usage limit the run pauses and resumes after reset (v2.1.271+), at most twice. — [Dynamic workflows](https://code.claude.com/docs/en/workflows)
- Agent SDK phrasing of the same point: "Subagents work well for a few delegated tasks per turn. For runs that coordinate dozens to hundreds of agents, use the Workflow tool". — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)

Costs page (general context hygiene)
- "Token costs scale with context size: the more context Claude processes, the more tokens you use." — [Manage costs](https://code.claude.com/docs/en/costs)
- "Claude Code sends your full conversation with every request, and each time Claude uses tools it sends another request carrying that batch of tool results." A one-line question in an all-day session "still draws usage for the whole conversation." — [Manage costs](https://code.claude.com/docs/en/costs)
- "Subagents and workflows: every subagent, and every agent a dynamic workflow spawns, sends its own requests on top of the main conversation's." — [Manage costs](https://code.claude.com/docs/en/costs)
- Delegate verbose ops: "the verbose output stays in the subagent's context while only a summary returns to your main conversation." — [Manage costs](https://code.claude.com/docs/en/costs)
- Hooks as pre-filters: a hook can grep a 10,000-line log for ERROR, "reducing context from tens of thousands of tokens to hundreds." — [Manage costs](https://code.claude.com/docs/en/costs)
- Keep CLAUDE.md under 200 lines; MCP tool definitions are deferred by default. — [Manage costs](https://code.claude.com/docs/en/costs)
- Enterprise averages: about $13 per developer per active day, $150-250 per month. — [Manage costs](https://code.claude.com/docs/en/costs)

Context window page (compaction details)
- After compaction Claude Code re-reads "up to five" recently modified files; a file over 5,000 tokens comes back as a path reference; invoked skill bodies are re-injected "capped at 5,000 tokens per skill and 25,000 tokens total". Background subagents keep running across compaction and Claude is reminded which are still running. — [Explore the context window](https://code.claude.com/docs/en/context-window)
- `/autocompact 500k` sets the auto-compact point; `/clear` between tasks because "Old conversation crowds out the files you need next and costs tokens on every message"; "Delegate large reads" to a subagent. 1M-token windows on Fable, Sonnet 5+, Haiku 5.5, Opus 4.6+, Sonnet 4.6. — [Explore the context window](https://code.claude.com/docs/en/context-window)

### Inferences
- Anthropic's own escalation ladder is: main thread -> subagent (few per turn, summaries back) -> agent team (3-5 peers, shared task list) -> workflow (dozens to hundreds, results never touch the conversation). An LLM orchestrator holding dozens of reports is, by their table, the wrong tier.
- The 20-concurrent-subagent cap and the "15,000-token description budget" are the two hard numbers that bound a Claude Code fan-out from a single session; the workflows runtime raises the ceiling to 16 concurrent (CPU-bound) and 1,000 total.
- The best-practices "retired advice" table shows Anthropic's emphasis has moved from "protect the context window" to "pay only for the context you need": the risk of a big orchestrator is now framed as cost and cache misses more than hitting the window.

### Gaps
- No first-party number for the size of the "small metadata trailer" appended to a subagent's result, or for the typical size of a subagent's final message in Claude Code.
- The subagents page was 116k characters; the middle section (roughly chars 100k-116k was read, 0-100k summarised) may contain an "API errors in subagents" and "output scanning" section that I only saw via the SDK page's cross-reference.

## Q6. Claude Agent SDK docs: subagents and context isolation

### Takeaway
The SDK page is the most explicit first-party statement of the isolation contract: intermediate tool calls and results stay inside the subagent and only its final message reaches the parent; the only thing passed down is the Agent tool's prompt string; depth (3), concurrency (20) and spend (`maxBudgetUsd`) are the three caps on a tree of agents.

### Cited Findings
- Purpose: "Use them to isolate context, run multiple analyses in parallel, and apply specialized instructions without adding to the main agent's prompt." — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- Context isolation: "intermediate tool calls and results stay inside the subagent; only its final message returns to the parent." "The parent receives a concise summary, not every file the subagent read." — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- Parallelization: "independent subtasks finish in the time of the slowest one rather than the sum of all of them." — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- What goes down: "The only content you pass from parent to subagent is the Agent tool's prompt string, so include any file paths, error messages, or decisions the subagent needs directly in that prompt." The subagent does not receive "The parent's conversation history or tool results" or "The parent's system prompt". — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- Parent may re-summarise: "The parent receives the subagent's final report, but may summarize it in its own response." — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- Output scanning (v2.1.210+): the final message is scanned for control-tag imitation, permission-config mentions and `Human:`/`Assistant:` turn markers; "it never removes or rewords the subagent's text." — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- Default is background: "a subagent your main agent spawns runs in the background," the Agent tool result has `status: "async_launched"`, and "the report arrives in a later turn." — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- Trees of agents: "a subagent can spawn subagents of its own, so one prompt can grow into a tree of agents." Caps: depth 3 (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`), concurrency 20 (`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`), spend `maxBudgetUsd` (no default). Requires SDK TS v0.3.219 / Py v0.2.127+. — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- `maxTurns` stops a subagent and marks output partial (v2.1.246+); `autoCompactWindow` per agent (TS SDK v0.3.296+); `omitClaudeMd` trims a subagent's startup context (TS SDK v0.3.271+). — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- Opus 5 "delegates to subagents more readily than earlier models," so the caps "matter most" there; the `claude_code` preset adds a line telling Claude not to call the Agent tool unless asked. — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- Subagent messages in the stream carry `parent_tool_use_id` (and `agent_id` in TS v0.3.292+), so the host can see subagent traffic even though the parent model does not. — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- An API error that ends a subagent early (e.g. a rate limit) "is never delivered as its result." — [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)

### Inferences
- Because the SDK streams subagent messages to the host application with `parent_tool_use_id`, an SDK orchestrator can log or persist full subagent transcripts externally without them ever entering the model's context, which is the "write to external systems, pass references" pattern from Q1 done at the harness level.

### Gaps
- The SDK page does not give a recommended size for the Agent tool prompt or for the subagent's final report.

## Q7. Prompt caching and rate limits: why a big, growing orchestrator context is slow or throttled

### Takeaway
Caching is prefix-based and exact: every turn re-sends the whole conversation, cache reads cost 0.1x (or less) and do not count toward input-token rate limits on most models, but any change ahead of the breakpoint (tools, system prompt, model, effort, compaction) forces the whole prefix to be reprocessed at full price and full latency; the longer the orchestrator's context, the more each miss costs, and Anthropic's Claude Code team says "a few percentage points of cache miss rate can dramatically affect cost and latency."

### Cited Findings

API prompt caching mechanics
- Prefix order: caching "references the entire prompt: tools, system, and messages (in that order), up to and including the block designated with cache_control." "The system automatically finds the longest prefix that a prior request already wrote to the cache." — [Prompt caching (API)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Exactness: "Cache hits require 100 percent identical prompt segments, including all text and images up to and including the block marked with cache control." — [Prompt caching (API)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Lifetime: "By default, the cache has a 5-minute lifetime. The cache is refreshed for no additional cost each time the cached content is used." A 1-hour TTL is available via `"ttl": "1h"`. The lifetime counts from the start of the request. — [Prompt caching (API)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Pricing: 5-minute cache writes "are 1.25 times the base input tokens price"; 1-hour writes "are 2 times the base input tokens price"; cache reads "are 0.1 times the base input tokens price", with per-model exceptions of 0.05x on Opus 5.5 / Sonnet 5.5 and 0.025x on Fable 5.1 / Mythos 5.1. — [Prompt caching (API)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Minimum cacheable prefix: 512 tokens on Fable 5.x, Mythos 5.x, Opus 5/5.5, Sonnet 5/5.5, Haiku 5.5; 1,024 on Opus 4.8, Sonnet 5, Sonnet 4.x; 2,048 on Opus 4.7 / Mythos Preview; 4,096 on Opus 4.5/4.6 and Haiku 4.5. — [Prompt caching (API)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Invalidation cascades: "Changes at each level invalidate that level and all subsequent levels." "Modifying tool definitions (names, descriptions, parameters) invalidates the entire cache." Thinking config and `output_config.effort` changes invalidate the message cache. — [Prompt caching (API)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Multi-turn rule: "Place cache_control on the last block whose prefix is identical across the requests you want to share a cache." Automatic caching puts the breakpoint on the last cacheable block; "The system checks at most 20 positions for each breakpoint". Up to 4 breakpoints. — [Prompt caching (API)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Usage accounting: `input_tokens` covers only tokens after the last breakpoint; total input = `cache_read_input_tokens + cache_creation_input_tokens + input_tokens`. — [Prompt caching (API)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

Rate limits
- Three limits per model class: RPM, ITPM, OTPM; a 429 says which was exceeded and may carry `retry-after`. Token bucket: "capacity is continuously replenished up to your maximum limit, rather than being reset at fixed intervals." Short bursts can trip per-second enforcement. — [Rate limits](https://platform.claude.com/docs/en/api/rate-limits)
- Cache-aware ITPM: "For most Claude models, only uncached input tokens count toward your ITPM rate limits." `input_tokens` and `cache_creation_input_tokens` count; `cache_read_input_tokens` "Do NOT count toward ITPM" (exception: Haiku 3.5). "ITPM rate limits are estimated at the beginning of each request". Worked example: 2M ITPM with an 80% cache hit rate = 10M total input tokens/min. — [Rate limits](https://platform.claude.com/docs/en/api/rate-limits)
- Acceleration limits: 429s also come from "a sharp increase in usage"; advice is to "increase your traffic gradually". — [Rate limits](https://platform.claude.com/docs/en/api/rate-limits)
- Standard tiers (ITPM / OTPM / RPM): Start tier Opus 5.5 / Sonnet 5.5 / Haiku 5.5 = 2,000,000 / 400,000 / 1,000; Fable 5.x = 500,000 / 100,000 / 1,000. Build: 5M / 1M / 5,000 (Fable 1.5M / 300k / 2,000). Scale: 10M / 2M / 10,000 (Fable 4M / 800k / 4,000). Monthly spend caps: Start $500, Build $1,000, Scale $200,000. — [Rate limits](https://platform.claude.com/docs/en/api/rate-limits)
- OTPM counts actual generated tokens; `max_tokens` is not counted. — [Rate limits](https://platform.claude.com/docs/en/api/rate-limits)
- Claude Code per-user TPM recommendation for teams: 200k-300k TPM and 5-7 RPM per user at 1-5 users, falling to 10k-15k TPM at 500+ users; limits are organisation-level. — [Manage costs](https://code.claude.com/docs/en/costs)

Claude Code's caching behaviour (why a growing orchestrator pays)
- "Claude Code re-sends the full context: the system prompt, your project context, every prior message and tool result, and your new message." "The match is exact, so a change anywhere in the prefix recomputes everything after it. There is no per-file or per-segment caching." — [Prompt caching (Claude Code)](https://code.claude.com/docs/en/prompt-caching)
- Layer order: system prompt (changes when tool definitions change), project context (changes at session start, `/clear`, `/compact`), conversation (every turn). "A change to the system prompt invalidates everything". — [Prompt caching (Claude Code)](https://code.claude.com/docs/en/prompt-caching)
- Invalidating actions: switching models ("Each model has its own cache"), changing effort (except on Opus 5.5 / Sonnet 5.5 / Haiku 5.5 / Fable 5.1 with API key or subscription), fast mode toggle, connecting/removing an MCP server when tools are not deferred, denying an entire tool, compaction, accumulating many images, upgrading Claude Code. — [Prompt caching (Claude Code)](https://code.claude.com/docs/en/prompt-caching)
- Cache-safe actions: spawning a subagent ("From the parent's side, the subagent's call and result append to the conversation, leaving the parent's prefix intact."), invoking skills, plan mode, `/rewind`, editing files, editing CLAUDE.md mid-session. — [Prompt caching (Claude Code)](https://code.claude.com/docs/en/prompt-caching)
- Subagents and cache: a subagent's "first request doesn't read the parent's cache, because the two prefixes differ, and it warms a cache of its own." Subagents fall outside the main TTL bucket "so they get five minutes even on a subscription". A fork "inherits the parent's system prompt, tools, and conversation history exactly, so its first request reads the parent's cache." — [Prompt caching (Claude Code)](https://code.claude.com/docs/en/prompt-caching)
- TTL buckets: main conversation gets one hour on a Claude subscription within plan usage, five minutes on API key / usage credits / cloud; "Everything else" (subagents, workflows, in-process teammates, forks, compaction) gets five minutes. `promptCacheTtl` and `subagentPromptCacheTtl` (v2.1.242+) override. — [Prompt caching (Claude Code)](https://code.claude.com/docs/en/prompt-caching)
- Compaction cost: the summarisation request "repeats your conversation's system prompt, tools, and the history being summarized"; warm, "a mid-session /compact then costs a fraction of what the context size suggests"; cold, it "reprocesses that history as uncached input." Idle compaction on the one-hour TTL can be turned off with `idleCompaction: false` (v2.1.290+). — [Prompt caching (Claude Code)](https://code.claude.com/docs/en/prompt-caching)
- Cache scope: two parallel sessions in the same directory "can read each other's cache"; different directories build different prefixes. — [Prompt caching (Claude Code)](https://code.claude.com/docs/en/prompt-caching)
- Observability: `/usage` shows a `Prompt cache (main)` line with hit share, misses and warm/cold (v2.1.251+), and the likely cause of the last miss (v2.1.260+). — [Manage costs](https://code.claude.com/docs/en/costs)

The Claude Code team's blog on caching (April 2026)
- Prompt caching "allows us to reuse computation from previous roundtrips and significantly decrease latency and cost," and a high hit rate "decreases costs and helps us create more generous rate limits." — [Prompt caching is everything](https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/)
- Design rule: "static content first, dynamic content last" (system prompt and tools, CLAUDE.md, session context, messages). Plan mode keeps "all tools in the request at all times" and uses EnterPlanMode/ExitPlanMode as tools; deferred tool stubs stay "in the same order"; updates arrive as a `<system-reminder>` in the next message rather than editing the system prompt. — [Prompt caching is everything](https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/)
- "Adding or removing a tool invalidates the cache for the entire conversation." "Any change anywhere in the prefix invalidates everything after it." — [Prompt caching is everything](https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/)
- The cost of a miss grows with context: a miss means paying "the full, uncached input rate for the entire conversation you're sending in," and the longer the conversation "the more expensive that one call becomes." Example: switching from Opus to Haiku "100k tokens into a conversation" costs more than staying on Opus. — [Prompt caching is everything](https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/)
- "A few percentage points of cache miss rate can dramatically affect cost and latency." The team alerts on hit rate and declares SEVs when it drops. — [Prompt caching is everything](https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/)
- Subagents as the cache-safe way to change model: use a subagent instead of switching the main model; Opus can write a hand-off message; Claude Code's Explore agents run Haiku this way. "Fork operations need to share the parent's prefix." Compaction uses "the exact same system prompt, user context, system context, and tool definitions as the parent conversation" and needs a "compaction buffer" of headroom. — [Prompt caching is everything](https://claude.dev/blog/lessons-from-building-claude-code-prompt-caching-is-everything/)

### Inferences
- Why a large orchestrator stalls, in Anthropic's own terms: each of its turns re-sends the whole accumulated context; while the cache is warm that is cheap (0.1x or less) and rate-limit-free, but every subagent report appended is new `input_tokens` that does count toward ITPM, and any invalidating event (tool-set change, compaction, model/effort change, a gap longer than the TTL) re-bills and re-processes the entire prefix at full price, with latency to match. The bigger the orchestrator's context, the bigger that penalty, which is the mechanism behind the blog's "few percentage points of miss rate" warning.
- Subagents get a 5-minute TTL by default even on a subscription, so a fan-out whose workers idle (waiting on tools, on the orchestrator, or on a rate-limit pause) more than five minutes between requests re-warms its caches at 1.25x write cost; `subagentPromptCacheTtl: 1h` is the documented fix.
- Cache-aware ITPM explains why a fan-out of many fresh subagents is more rate-limit-hungry than one long conversation: each subagent's first request is entirely uncached `cache_creation_input_tokens`, which do count toward ITPM, whereas the orchestrator's repeated prefix does not.

### Gaps
- Concurrency / in-flight request limits: the API rate-limits page lists RPM, ITPM, OTPM, acceleration limits and a batch-queue limit, but no explicit "maximum concurrent in-flight requests" figure. I found no first-party statement of a concurrency cap for Max plans either.
- Max plan 5-hour / weekly window: the official support article (support.claude.com/en/articles/11014257, both "about-claude-s-max-plan-usage" and "about-claude-max-plan-usage" slugs) returned HTTP 404 on 2026-10-28, so I could not quote it. The closest first-party statements I could fetch are: Claude Code's costs page, which says Team/Enterprise seat allowances reset "on a rolling five-hour window and a weekly window" and are shared with chat and Cowork, and that session/weekly limits are "shared across all models"; and the workflows page, which says a run pauses when it hits the claude.ai usage limit and that ultracode sessions reach "a session or weekly limit sooner". The 5x/20x multipliers and message-count estimates for Max are therefore unverified from a first-party page in this note.
- The prompt-caching API page was truncated at 100k of 162k characters; the sections on 1-hour cache specifics beyond the pricing table, and on cache storage/sharing, were not read.
