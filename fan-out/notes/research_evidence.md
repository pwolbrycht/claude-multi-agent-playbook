# Evidence base: large orchestrator context vs thin coordinators with summarized / file-based handoffs

Researched 2026-10-28. Scope: measured evidence (peer-reviewed where possible; preprints and vendor blogs marked) on long-context degradation, multi-agent failure modes, hierarchical summarization / external memory, orchestrator-worker vs single long-context agent, cost/latency scaling of re-sent context, structured handoffs, and concurrency/queueing. Every finding is sourced inline; numbers I could not verify against a primary source are in Gaps.

## Q1. Long-context degradation: does accuracy fall with input length even inside the window?

### Takeaway
Yes, consistently and across model generations. Every benchmark that controls for task difficulty (Lost in the Middle 2023, RULER 2024, LongICLBench 2024, NoLiMa 2025, Chroma Context Rot 2025, Du et al. 2025, Wang et al. 2026) finds accuracy falling as input grows, well before the advertised window, and the drop persists even when the model provably retrieves the right evidence. The effect is driven by distractors, low lexical overlap and sheer length, not just "the middle".

### Cited Findings
- **Lost in the Middle** (Liu, Lin, Hewitt, Paranjape, Bevilacqua, Petroni, Liang; Stanford/UW/Meta; arXiv July 2023, TACL 2023). GPT-3.5-Turbo multi-document QA with 20 docs: 75.8% when the answer is first, 53.8% at position 9 (middle), 63.2% at position 19; 30-doc setting: 73.4% / 50.5% / 63.7%. Worst-case positions fall below the closed-book score (56.1%). Going from 20 to 50 retrieved documents improves accuracy only ~1.5% (GPT-3.5) / ~1% (Claude-1.3) while increasing length, latency and cost. — [arXiv 2307.03172](https://arxiv.org/abs/2307.03172); numbers from [HTML v3](https://arxiv.org/html/2307.03172v3)
- **RULER** (Hsieh et al., NVIDIA; arXiv April 2024, COLM 2024). 13 tasks, 17 models. "Despite achieving nearly perfect accuracy in the vanilla NIAH test, all models exhibit large performance drops as the context length increases." Nearly all claimed 32K+ windows; only about half held performance at 32K. Effective length defined by an 85.6% threshold (Llama2-7B at 4K); e.g. a Llama-3-8B-1048K variant claims 1M but is effective to 16K. — [arXiv 2404.06654](https://arxiv.org/abs/2404.06654v2); [benchmark table summary](https://benchmarklist.com/benchmarks/ruler/) (secondary)
- **LongICLBench** (Li et al., TIGER-AI-Lab; arXiv April 2024, TMLR 2025). Six extreme-label classification sets (28-174 labels, 2K-50K-token demonstrations), 13-15 long-context LLMs. Models degrade as demonstrations lengthen; on the hardest set (Discovery) all LLMs score near zero except Gemini-1.5-Pro at 14%, vs 87% for a fine-tuned BERT; models favour labels presented near the end. — [arXiv 2404.02060](https://arxiv.org/abs/2404.02060v3); [GitHub](https://github.com/TIGER-AI-Lab/LongICLBench)
- **NoLiMa** (Modarressi et al., LMU/Adobe; arXiv Feb 2025, ICML 2025, peer-reviewed). Removes lexical overlap between question and needle. At 32K, 10 of the 12-13 models drop below 50% of their short-context baseline; GPT-4o falls from 99.3% to 69.7%. Authors attribute it to attention difficulty when literal matches are absent. — [arXiv 2502.05167](https://arxiv.org/abs/2502.05167); [PMLR v267](https://proceedings.mlr.press/v267/modarressi25a.html)
- **Context Rot** (Hong, Troynikov, Huber; Chroma technical report, 14 July 2025; not peer-reviewed). 18 models (GPT-4.1, Claude 4, Gemini 2.5, Qwen3 families). Findings: lower needle-question semantic similarity degrades faster with length; a single distractor hurts and four hurt more; shuffled haystacks beat logically ordered ones across all 18 models; LongMemEval 306 prompts at ~113k tokens (full) vs ~300 tokens (focused): all models significantly better on focused input, Claude family largest gap (mostly abstentions); even repeated-word replication degrades up to 10,000 words. The report gives few exact percentage drops; it reports refusal rates (Claude Opus 4 2.89%, GPT-4.1 2.55%, Qwen3-8B 4.21%, GPT-3.5 Turbo 60.29%). — [Chroma report](https://www.trychroma.com/research/context-rot)
- A secondary write-up of the Chroma data states GPT-4-1106 dropped from 96.6% to 81.2% between 4K and 128K; I could not confirm this in the report itself. — [morphllm summary](https://www.morphllm.com/context-rot) (secondary, unverified)
- **Context Length Alone Hurts LLM Performance Despite Perfect Retrieval** (Du et al.; arXiv Oct 2025, Findings of EMNLP 2025). Five models, math/QA/coding: performance "still degrades substantially (13.9%--85%)" as input grows even when all relevant information is retrieved and inputs are within claimed context; persists when irrelevant tokens are replaced with whitespace or masked. Mitigation (recite retrieved evidence before solving) gives up to +4% for GPT-4o on RULER. — [arXiv 2510.05381](https://arxiv.org/abs/2510.05381)
- **LLMs Get Lost in Multi-Turn Conversation** (Laban, Hayashi, Zhou, Neville; Microsoft/Salesforce; arXiv May 2025, preprint). 200,000+ simulated conversations, six generation tasks: average 39% drop multi-turn vs single-turn, decomposed into a minor aptitude loss and a large reliability loss; models "take a wrong turn... and do not recover". Relevant because an orchestrator's growing turn history is exactly this regime. — [arXiv 2505.06120](https://arxiv.org/abs/2505.06120)
- **2026 update: Intelligence Degradation in Long-Context LLMs** (Wang, Min, Zou; arXiv 7 Jan 2026, preprint, one model). Qwen2.5-7B on 1,000 samples spanning 5-95% of the window: F1 falls from 0.55-0.56 to 0.30 (45.5% degradation) at a critical threshold of 40-50% of the maximum context, i.e. collapse rather than gradual decay. — [arXiv 2601.15300](https://arxiv.org/abs/2601.15300)
- **2026 update: RULER++** (arXiv Feb 2026, preprint) adds effective-context-length curves and evaluates 24 models with 128K-1M windows. — [arXiv 2602.20459](https://arxiv.org/pdf/2602.20459) (not read in full)
- A Feb 2026 preprint on automated bug fixing reportedly found sharp degradation even with the relevant files injected directly (Qwen3-Coder ~7% resolve rate at 64k). — [arXiv 2602.16069 via secondary summary](https://www.morphllm.com/context-rot) (unverified against primary)

### Inferences
- The relevant design variable is not window size but "how much irrelevant or weakly related material sits in front of the model". Chroma's distractor and LongMemEval results and Du et al.'s whitespace experiment both point at length itself, not just retrieval failure.
- For an orchestrator, every worker transcript appended to its context is a distractor set for every later decision. The ~113k-vs-300-token LongMemEval gap is the closest direct analogue of "full worker transcripts vs condensed worker summaries".
- The 2026 threshold paper suggests practical budgets around 40-50% of the window for small open models; frontier models degrade more gradually (NoLiMa, Chroma) but still measurably.

### Gaps
- Chroma's report does not publish per-model accuracy curves as numbers; exact drop sizes require their released data.
- No 2026 peer-reviewed study specifically measures *agent* task success vs orchestrator context length; the 2026 evidence is preprints on single models.
- The 13.9%-85% range in Du et al. is not broken down by model/task in the abstract.

## Q2. Multi-agent failure taxonomies (MAST): which failures relate to information loss, misalignment, orchestrator overload, and what fixes were tested?

### Takeaway
MAST (Cemri et al., Berkeley; arXiv March 2025, NeurIPS 2025) finds 41-86.7% failure rates across seven frameworks and attributes most failures to system design and inter-agent coordination, not raw model ability. Structural (topology) fixes beat prompt fixes, but even the best interventions gave only +9 to +16 points, which the authors call not substantial. Google/MIT's Dec 2025 scaling study adds that uncoordinated agents amplify errors 17.2x vs 4.4x under a central orchestrator.

### Cited Findings
- **Why Do Multi-Agent LLM Systems Fail?** (Cemri, Pan, Yang et al.; UC Berkeley + Intesa Sanpaolo; arXiv 2503.13657, March 2025; NeurIPS 2025). 150+ traces (later 1,600+ in MAST-Data), seven frameworks, Cohen's kappa 0.88; LLM judge (o1) 94% accuracy, kappa 0.77. Failure rates 41%-86.7% across systems. — [arXiv abs v3](https://arxiv.org/abs/2503.13657v3); [HTML v3](https://arxiv.org/html/2503.13657v3); [NeurIPS](https://nips.cc/virtual/2025/poster/121528)
- Taxonomy and prevalence: FC1 System design: disobey task spec 11.8%, disobey role spec 1.5%, step repetition 15.7%, loss of conversation history 2.8%, unaware of termination 12.4%. FC2 Inter-agent misalignment: conversation reset 2.2%, fail to ask for clarification 6.8%, task derailment 7.4%, information withholding 0.85%, ignored other agent's input 1.9%, reasoning-action mismatch 13.2%. FC3 Task verification: premature termination 6.2%, no/incomplete verification 8.2%, incorrect verification 9.1%. — [arXiv HTML v3](https://arxiv.org/html/2503.13657v3)
- Information withholding (FM-2.4) "appears almost only in failed runs"; example: a HyperAgent Navigator solved internally without telling the Planner. — [arXiv HTML v3](https://arxiv.org/html/2503.13657v3)
- Interventions on ChatDev (GPT-4o): giving the CEO the final say +9.4% success; improved role specification +9.4%; adding a high-level task-objective verification step +15.6% on ProgramDev. AG2/MathChat on GSM-Plus: GPT-4 baseline 84.75 -> improved prompt 89.75 (significant) vs new topology 85.50 (p=0.4, n.s.); GPT-4o baseline 84.25 -> prompt 89.00 and topology 88.83 (both p=0.03). Authors: "topology-based changes are more effective than prompt-based changes for both systems", but the ChatDev gains "do not constitute substantial improvements". — [arXiv HTML v3](https://arxiv.org/html/2503.13657v3)
- MAST recommends memory/state management (citing MemGPT and TapeAgents' replayable log), standardized communication protocols, confidence-thresholded action, and topology redesign for verification. — [arXiv HTML v3](https://arxiv.org/html/2503.13657v3)
- **Towards a Science of Scaling Agent Systems** (Kim, Liu et al.; Google Research / DeepMind / MIT; arXiv 2512.08296, Dec 2025, preprint; Google blog 28 Jan 2026). 180 configurations, 5 architectures (single, independent, centralized, decentralized, hybrid), 3 model families. Independent multi-agent amplified errors 17.2x; centralized contained it to 4.4x. Centralized coordination +80.9% on Finance-Agent (parallelizable); every multi-agent variant degraded sequential PlanCraft by 39-70%. "Tool-coordination trade-off": with many tools (16+) coordination cost rises disproportionately. Model predicts the best architecture for 87% of held-out configs (R^2 0.513). — [Google Research blog](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/); [arXiv 2512.08296](https://arxiv.org/html/2512.08296v2); [MIT Media Lab page](https://www.media.mit.edu/projects/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/overview/)
- **Cognition, "Don't Build Multi-Agents"** (Walden Yan, 12 June 2025; engineering essay, no measurements). Two principles: share full agent traces, not just messages; actions carry implicit decisions and conflicting decisions produce bad results. Flappy Bird example: two subagents produced mismatched styles even with the original task shared. Proposes a dedicated (sometimes fine-tuned, smaller) compressor model for history. — [cognition.com](https://cognition.com/blog/dont-build-multi-agents); April 2026 follow-up says the finding still holds for parallel-writer swarms and most working deployments use read-only subagents — [Multi-Agents: What's Actually Working](https://cognition.com/blog/multi-agents-working)
- LangChain's synthesis (Harrison Chase, 16 June 2025): read-only subagents parallelize; parallel writes with conflicting decisions are far worse; Anthropic's early vague subagent briefs caused duplicated work (two subagents both on 2025 supply chains). Vendor post. — [langchain.com](https://www.langchain.com/blog/how-and-when-to-build-multi-agent-systems)

### Inferences
- MAST's directly "information-loss" modes (history loss 2.8%, withholding 0.85%, ignored input 1.9%, reset 2.2%) are individually rare, but the big buckets (step repetition 15.7%, reasoning-action mismatch 13.2%, unaware of termination 12.4%, task-spec disobedience 11.8%) are what an overloaded or under-briefed coordinator produces. The paper does not itself attribute them to context size.
- The 17.2x vs 4.4x error amplification is the strongest quantitative argument for a *central* coordinator with verification, and the 39-70% sequential-task degradation is the strongest argument against fanning out sequential work at all.
- No study isolates "orchestrator context size" as a variable; the evidence for thin coordinators is indirect (Q1 degradation + Q2 coordination costs).

### Gaps
- MAST does not report per-framework failure rates in the text I read (they are in a figure).
- The "45% single-agent baseline threshold" quoted by a Towards Data Science article is not in the Google blog; unverified.
- MAST's interventions were prompt and topology changes; none tested "summarize vs pass full transcripts".

## Q3. Hierarchical / map-reduce summarization and scratchpad / external memory: what do they show about offloading state?

### Takeaway
Offloading state out of the working context is the best-evidenced single technique: MemGPT lifts GPT-4 deep-memory retrieval from 32.1% to 92.5%, OpenAI's recursive book summarization set SOTA on BookSum/NarrativeQA in 2021, and Anthropic's production system reports that subagents writing to external storage and passing lightweight references reduces information loss. Reflexion/ReAct/Generative Agents establish the pattern but their numbers were not re-verified this session.

### Cited Findings
- **Recursively Summarizing Books with Human Feedback** (Wu, Ouyang, Ziegler, Stiennon, Lowe, Leike, Christiano; OpenAI; arXiv Sept 2021). Recursive task decomposition: summarize small sections, then summaries of summaries. Matched human summary quality in ~5% of books; SOTA on BookSum; zero-shot QA on the summaries reached SOTA on NarrativeQA. Labelers could supervise without reading the whole book. — [arXiv 2109.10862](https://arxiv.org/abs/2109.10862)
- **MemGPT** (Packer, Wooders, Lin, Fang, Patil, Stoica, Gonzalez; Berkeley; arXiv Oct 2023, v2 Feb 2024; preprint, now Letta). OS-style virtual context management, tiered memory. Deep Memory Retrieval accuracy: GPT-3.5 38.7% -> 66.9% with MemGPT; GPT-4 32.1% -> 92.5%; GPT-4 Turbo 35.3% -> 93.4%. Multi-document QA: MemGPT unaffected by document count while fixed-context GPT-4 degrades; nested key-value retrieval: GPT-4 baseline hits 0% by 3 nesting levels, MemGPT+GPT-4 unaffected. — [arXiv 2310.08560](https://arxiv.org/abs/2310.08560); [HTML v2](https://arxiv.org/html/2310.08560v2)
- **Anthropic multi-agent research system** (June 2025, engineering post, internal evals). Lead agent saves its plan to Memory because context over 200,000 tokens is truncated; subagents act as "intelligent filters" returning condensed findings; subagents store work in external systems and pass "lightweight references back to the coordinator", which "reduces information loss and token overhead"; completed phases are summarized and essentials stored in external memory before spawning fresh subagents. — [anthropic.com](https://www.anthropic.com/engineering/multi-agent-research-system)
- MAST names memory/state management (MemGPT, TapeAgents' replayable log) as the mechanism to "enhance context understanding and reduce ambiguity in communication". — [arXiv 2503.13657 HTML v3](https://arxiv.org/html/2503.13657v3)
- Cognition reports using a smaller fine-tuned compressor model for history in some domains, with the caveat that it is hard to do well and "still has limits". — [cognition.com](https://cognition.com/blog/dont-build-multi-agents)
- Du et al. 2025: making the model "recite" retrieved evidence before answering (a form of in-context summarization) gives up to +4% on RULER for GPT-4o. — [arXiv 2510.05381](https://arxiv.org/abs/2510.05381)
- Pattern papers (URLs verified, numbers from my prior knowledge, not re-fetched this session): ReAct (Yao et al., Oct 2022) interleaves reasoning and acting — [arXiv 2210.03629](https://arxiv.org/abs/2210.03629); Reflexion (Shinn et al., March 2023) stores verbal reflections in episodic memory, reported 91% pass@1 on HumanEval vs GPT-4's 80% — [arXiv 2303.11366](https://arxiv.org/abs/2303.11366); Generative Agents (Park et al., April 2023) memory stream with retrieval + reflection, ablations show removing memory/reflection degrades believability — [arXiv 2304.03442](https://arxiv.org/abs/2304.03442).

### Inferences
- The MemGPT numbers are the cleanest demonstration that the *same* model with the *same* facts performs far better when facts are paged in on demand than when they sit in a long context.
- Anthropic's "lightweight references" is a file-based handoff in all but name; it is production evidence but not a controlled comparison.

### Gaps
- No controlled A/B of "orchestrator receives full worker transcript" vs "orchestrator receives summary + file pointer" was found in the literature.
- Reflexion / ReAct / Generative Agents figures were not re-verified in this session.
- Wu et al. 2021 is pre-LLM-agent era and measures summary quality, not downstream agent task success.

## Q4. Orchestrator-worker vs single long-context agent: independent benchmarks

### Takeaway
Evidence is genuinely contested. Anthropic's 90.2% gain is internal and uncontrolled for compute; Tran & Kiela (April 2026) and Google/MIT (Dec 2025) show single agents match or beat multi-agent under equal token budgets on sequential/multi-hop tasks, while parallelizable breadth tasks (BrowseComp, finance analysis) show consistent but modest multi-agent gains (+2 to +5 points on BrowseComp in 2026 vendor reports, +81% on Finance-Agent). "More agents" sampling-and-voting gains are real but are test-time-compute gains, not coordination gains.

### Cited Findings
- **Anthropic** (June 2025): Opus 4 lead + Sonnet 4 subagents beat single Opus 4 by 90.2% on an internal research eval; on BrowseComp three factors explained 95% of variance, token usage alone 80%; agents use ~4x chat tokens, multi-agent ~15x; parallel subagents cut research time by up to 90% on complex queries. — [anthropic.com](https://www.anthropic.com/engineering/multi-agent-research-system)
- **Single-Agent LLMs Outperform Multi-Agent Systems on Multi-Hop Reasoning Under Equal Thinking Token Budgets** (Tran & Kiela; arXiv 2 April 2026, preprint). Data Processing Inequality argument: single agents are more information-efficient; under matched reasoning-token budgets single agents matched or beat several MAS architectures on Qwen3, DeepSeek-R1-Distill-Llama and Gemini 2.5; API budget-control artifacts (notably Gemini 2.5) inflate apparent MAS gains. — [arXiv 2604.02460](https://arxiv.org/abs/2604.02460)
- **Google/MIT scaling study** (Dec 2025): centralized +80.9% on parallelizable Finance-Agent, -39% to -70% on sequential PlanCraft; independent agents 17.2x error amplification. — [Google Research blog](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/)
- **More Agents Is All You Need** (Li et al., Tencent; arXiv Feb 2024, TMLR 2024, peer-reviewed). Sampling-and-voting with up to 40 agents: Llama2-13B GSM8K 0.35 -> 0.59 (beats single Llama2-70B at 0.54); GPT-3.5 GSM8K 0.73 -> 0.85, MATH 0.29 -> 0.39, MMLU 0.59 -> 0.70, HumanEval 0.67 -> 0.73. Gains larger on harder tasks and weaker models; taper at very high difficulty; token usage grows proportionally with agents. — [arXiv 2402.05120](https://arxiv.org/html/2402.05120v2); [TMLR](https://mlanthology.org/tmlr/2024/li2024tmlr-more/)
- **BrowseComp single vs multi, same model** (vendor-reported, 2026, collated by a third-party leaderboard): Claude Opus 4.8 84.3% single vs 88.5% multi; Claude Sonnet 5 84.7% vs 86.6%; Claude Mythos 5 88.0% vs 93.3% (async multi-agent); GPT-5.6 Sol 90.4% vs 92.2% with four parallel agents; Kimi K2.5 74.9% (context-managed single) vs 78.4% (Agent Swarm). — [steel.dev leaderboard](https://leaderboard.steel.dev/leaderboards/browsecomp) (aggregator; each row cites a vendor announcement or system card)
- **GAIA**: MiroFlow (arXiv Feb 2026) reports multi-agent generally higher but single-agent superior on GAIA-Val, attributing it to GAIA's "strongly sequential task structure" where splitting "risks information loss". COMPASS reports Gemini 2.5 Pro 58.6 -> 65.5 on GAIA with an agent-as-a-tool setup; OxyGent ablation 36.21 -> 42.19 adding multi-agent orchestration. All preprints, self-reported. — [MiroFlow 2602.22808](https://arxiv.org/pdf/2602.22808); [COMPASS 2510.08790](https://arxiv.org/pdf/2510.08790); [OxyGent 2604.25602](https://arxiv.org/pdf/2604.25602)
- **Can Small Agents Collaborate to Beat a Single LLM?** (arXiv Jan 2026, ICLR 2026): an orchestrated 8B multi-agent system matched a 32B single agent; without thinking, results were mixed. — [arXiv 2601.11327](https://arxiv.org/pdf/2601.11327)
- Model dependence: a geometry-reasoning study found multi-agent helps open-source models but Gemini-2.0-Flash did better single-agent. — [arXiv 2512.16698](https://arxiv.org/abs/2512.16698v1)
- Cognition (June 2025): production multi-agent parallelism judged fragile for coding in 2025; by April 2026 most working deployments limit subagents to read-only search. — [cognition.com](https://cognition.com/blog/dont-build-multi-agents); [follow-up](https://cognition.com/blog/multi-agents-working)

### Inferences
- The consistent pattern across Anthropic, Google/MIT, Tran & Kiela and the GAIA/BrowseComp results: fan-out wins on breadth-first, parallelizable, read-heavy work and loses (or merely matches at higher cost) on sequential, write-heavy or multi-hop work.
- Anthropic's own variance analysis (token usage explains 80%) is consistent with Tran & Kiela's claim that much of the multi-agent gain is extra compute, not architecture.
- The "more agents" voting results support parallel *redundant* workers for verification, not deep orchestration hierarchies.

### Gaps
- No independent replication of the 90.2% figure; Anthropic's eval is internal.
- SWE-bench: no like-for-like single vs multi-agent entry found in this search.
- "Agents of Change", MetaGPT, ChatDev and AgentVerse papers were not fetched; MAST's traces show ChatDev/MetaGPT failure rates in the 41-86.7% band but per-framework figures were not extracted.

## Q5. Cost and latency scaling of re-sending large contexts; prompt-caching economics

### Takeaway
Measured prefill time grows super-linearly with prompt length (ChatGLM-6B: 1.27 s at 32K, 2.92 s at 64K, 7.76 s at 128K; attention's share rising from 32% to 57%), so an agentic loop that re-sends its full history pays a growing per-turn latency and a roughly quadratic cumulative token bill. Prompt caching converts most of that to 0.1x (or 0.025-0.05x on newest Claude models) reads at 1.25x write cost, but only for an exactly identical prefix.

### Cited Findings
- **SampleAttention** (arXiv June 2024, preprint): measured TTFT for ChatGLM-6B with full attention 1273.4 ms at 32K, 2917 ms at 64K, 7756.5 ms at 128K; full attention's share of prefill 32.2% at 32K -> 56.8% at 128K. — [arXiv 2406.15486](https://arxiv.org/pdf/2406.15486)
- **vAttention** (Microsoft Research, arXiv May 2024): Yi-6B prefill ~10.6 s at 64K and ~37.9 s at 128K (about 30 s of it attention); "throughput for longer contexts is lower due to the quadratic complexity of prefill attention"; at short contexts linear layers dominate. — [arXiv 2405.04437](https://arxiv.org/pdf/2405.04437)
- Vendor explainers agree prefill cost rises with prompt length and that FlashAttention/chunked prefill make practical scaling closer to linear at moderate lengths; a leaderboard post gives H100 TTFT ~680-740 ms for 7B and 3-3.3 s for 70B models without stating prompt length (unverified). — [Together AI](https://docs.together.ai/learn/ttft-and-tps); [generalcompute](https://www.generalcompute.com/blog/time-to-first-token-leaderboard-every-major-open-model-ranked?agent=true)
- **Anthropic prompt caching** (official docs, read 2026-10-28): 5-minute cache write 1.25x base input price, 1-hour write 2x, cache read 0.1x (0.025x on Claude Fable 5.1 / Mythos 5.1, 0.05x on Opus 5.5 / Sonnet 5.5). Minimum cacheable prefix 512-4,096 tokens depending on model. Cache hits need 100% identical segments; the hierarchy is tools -> system -> messages, and a change at one level invalidates that level and everything after; changing tool definitions invalidates everything; changing thinking parameters invalidates the message cache. Parallel requests must wait for the first response before they can hit the cache. — [platform.claude.com](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Lost in the Middle: 20 -> 50 retrieved docs gives ~1-1.5% accuracy for a large increase in length, latency and cost. — [arXiv 2307.03172](https://arxiv.org/html/2307.03172v3)
- Anthropic: multi-agent uses ~15x chat tokens; "More Agents": token usage grows proportionally with agent count. — [anthropic.com](https://www.anthropic.com/engineering/multi-agent-research-system); [arXiv 2402.05120](https://arxiv.org/html/2402.05120v2)

### Inferences
- Quadratic-loop argument (derived, not from a paper): if turn t re-sends a prefix of ~t*k tokens, cumulative input over T turns is ~k*T^2/2 uncached; with a hit rate near 1 it is ~0.1*k*T^2/2 + write costs, i.e. still quadratic in T but 10x cheaper, and per-turn prefill latency still grows with the prefix. A thin coordinator with short summaries lowers k directly, which caching cannot do.
- Cache economics punish anything that mutates the prefix: tool-set changes, thinking-setting changes, or re-ordering content. Appending worker outputs at the end is cache-friendly; editing the plan near the top is not.
- The "parallel requests must wait for the first response" rule means a fan-out of N workers sharing a big prefix should issue one warm-up call first or each worker pays the write cost.

### Gaps
- No vendor publishes a clean TTFT-vs-prompt-length curve for frontier hosted models; the measured curves above are 6B open models on specific kernels.
- The "up to 90% cost / 85% latency" caching claim often quoted was not present on the docs page I read; treat as unverified.
- No paper found that empirically measures the quadratic cumulative cost of an agentic loop; it is an arithmetic consequence of per-turn re-sending.

## Q6. Structured / schema-constrained handoffs between agents: do they reduce error propagation?

### Takeaway
Evidence is thin and two-sided. The only controlled study (Tam et al., EMNLP 2024 Industry) shows strict JSON-mode output *hurts* reasoning (GPT-3.5 GSM8K 76.6% -> 49.3%; Claude 3 Haiku 86.5% -> 23.4%) while helping some classification tasks; the fix is "reason in prose, then convert". No peer-reviewed study measures schema-gated handoffs reducing cascade errors in multi-agent systems; MAST recommends "standardized communication protocols" without testing them.

### Cited Findings
- **Let Me Speak Freely?** (Tam, Wu, Tsai, Lin, Lee, Chen; Appier/NTU; arXiv Aug 2024, EMNLP 2024 Industry Track). GSM8K zero-shot, text vs JSON-mode: GPT-3.5-Turbo 76.60 vs 49.25; Claude 3 Haiku 86.51 vs 23.44; Gemini 1.5 Flash 89.33 vs 89.21; LLaMA 3 8B 74.73 vs 48.90. "Stricter format constraints generally lead to greater performance degradation in reasoning tasks." Classification (DDXPlus) JSON *beat* text: Gemini 41.59 -> 60.36, GPT-3.5 44.07 -> 55.51. NL-to-format conversion stays near NL accuracy. — [arXiv 2408.02442](https://arxiv.org/html/2408.02442v1); [ACL Anthology](https://preview.aclanthology.org/credits/2024.emnlp-industry.91)
- MAST lists "standardized communication protocols" and confidence-thresholded action among recommended structural fixes, untested in the paper. — [arXiv 2503.13657 HTML v3](https://arxiv.org/html/2503.13657v3)
- GETReason (arXiv May 2025, application paper) claims staged JSON outputs limit cascading errors; qualitative, with ablations only on dropping image input. — [arXiv 2505.21863](https://arxiv.org/pdf/2505.21863)
- Anthropic's lead agent must give each subagent an objective, output format, tool guidance and task boundaries; vague briefs caused duplicated work and gaps. — [anthropic.com](https://www.anthropic.com/engineering/multi-agent-research-system); [langchain.com summary](https://www.langchain.com/blog/how-and-when-to-build-multi-agent-systems)
- A practitioner post claims schema gates raised end-to-end accuracy "from 10% to 70%"; untraceable, not reliable. — [zylos.ai](https://zylos.ai/research/2026-06-21-structured-output-validation-multi-agent-workflows) (unverified)

### Inferences
- Structured handoffs are best justified as *parse reliability* and *brief completeness* (objective, format, boundaries), not as a proven accuracy lever. The reasoning should happen in free text and the schema applied at the boundary.
- Schema-conformance guarantees (OpenAI Structured Outputs, Claude tool use) prevent parse failures, not semantic errors; MAST's verification modes (17.5% combined) are the ones a schema does not touch.

### Gaps
- No controlled multi-agent experiment comparing free-text vs schema handoffs on cascade error rate was found.

## Q7. Concurrency, rate limits and queueing (Little's law) for parallel LLM calls

### Takeaway
Only engineering blogs apply Little's law (L = lambda * W) to LLM fan-out; no academic or vendor-official treatment was found. The consistent practical rules: in-flight requests = arrival rate x mean latency, so latency spikes alone can trip concurrency/RPM limits; size concurrency ~2x the Little's-law steady state because latency variance is high; M/M/1 fits poorly because service times are not memoryless.

### Cited Findings
- Little's law applied to LLM pipelines: in-flight requests tied to throughput x response time; recommend sizing concurrency at ~2x the steady-state L; M/M/1 "fits poorly" because LLM service times are not memoryless. — [tianpan.co, April 2026](https://tianpan.co/blog/2026-04-16-backpressure-llm-pipelines-queue-theory) (blog)
- A slowdown can push a client into concurrency or RPM limits with no change in arrival rate; "the limit is a symptom of latency degradation, not its cause". — [tianpan.co, April 2026](https://tianpan.co/blog/2026-04-17-llm-rate-limits-distributed-systems-starvation) (blog)
- Concurrency caps complement per-window rate limits for slow calls; LangChain exposes maxConcurrency. — [gravitee.io](https://gravitee.io/corpus/gen-1738/operations-research/llm-traffic-queueing-models.html); [LangChain docs](https://js.langchain.com/docs/modules/model_io/llms/dealing_with_rate_limits)
- Anthropic caching docs: parallel requests meant to share a cache must wait for the first response to begin, or they each write. — [platform.claude.com](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Anthropic: multi-agent at ~15x chat tokens; "More Agents" debate variant capped at 10 agents "due to communication overhead". — [anthropic.com](https://www.anthropic.com/engineering/multi-agent-research-system); [arXiv 2402.05120](https://arxiv.org/html/2402.05120v2)

### Inferences
- With W growing per turn (Q5 prefill scaling), a fan-out of N workers each re-sending large contexts raises both lambda (more calls) and W (longer calls), so L grows faster than N; this is the queueing-theory version of "thin coordinators and short worker contexts keep you under rate limits".

### Gaps
- No peer-reviewed or vendor-official queueing analysis of LLM API fan-out was found; all sources are practitioner blogs and heuristics (2x margin, 80%-of-RPM cap).
- Anthropic/OpenAI official docs on concurrency limits were not retrieved in this search.
