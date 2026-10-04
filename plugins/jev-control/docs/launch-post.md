# Launch write-up (copy and adapt)

Three versions: a long post, a short post, and one-liners. Numbers are from [results.md](results.md); keep the honest
framing, it is the credible part.

---

## Long post

### Your agent's big model is doing small jobs. I gave it a reflex layer.

I have been running [Hermes Agent](https://github.com/NousResearch/hermes-agent) locally with a 27B model. Watching its
traces, I noticed how much of the work is not thinking at all. It is tiny pick-one decisions, made over and over:

- *Is this web search about to send a password to a search engine?*
- *Is this note worth saving to long-term memory, or is it just today's task?*
- *Which of these eight search results actually matter?*
- *Is this `rm -rf` safe to run?*

Each of those goes to the big model: seconds each, sometimes minutes when the server is busy, and every call is tokens
you pay for or wait on. So I built **jevcontrol-hermes**, a Hermes plugin that hands these decisions to a small
**decision model** ([spark-s1](https://huggingface.co/abhishek085/spark-s1-4b-v6), 4B). It answers a multiple-choice
question in about 80 ms and tells you how sure it is, so the plugin acts when the answer is clear and gives the decision back
to Hermes when it is not. The big model keeps the thinking and the writing.

It uses only Hermes' documented extension points. No core changes. Everything is off until you turn it on, and if the
decision server dies, Hermes behaves as if the plugin were not installed.

**What it does today**

- 🛡️ **Privacy guard**: blocks searches, URLs, browser scripts and network commands that would leak credentials, personal
  data or private local content. In my test set it caught 18 of 19 leaks and blocked none of 17 normal calls, about 80 ms each.
- 🧠 **Memory gatekeeper**: keeps long-term memory for durable facts and refuses procedures, task notes and orders to the
  agent, telling the agent where they belong. It kept 10 of 10 real facts and blocked 17 of 17 misfiled writes.
- ⚡ **Approval guard**: answers Hermes' smart-approval question for dangerous-looking shell commands. On 54 held-out
  commands it let through no unsafe command (same as Qwen3.8-27B), asked me about unclear ones more often, and answered in 88 ms
  instead of about 2.5 s, with no 90-second timeouts.
- 🔎 **Search picker** and 🗜️ **context compressor**: drop unrelated search results, and drop old tool output the agent no
  longer needs, with no LLM-written summary (about 260 ms per compaction in a real session).

Then I ran it live, with everything on, and watched the traces in Langfuse. The privacy guard stopped my agent from searching for
a password I had pasted. The memory gate refused a deploy procedure and accepted my preference for tabular answers. An
`rm -rf` on a temp folder was approved in 75 ms.

**What did not work (and why I am telling you)**

My first idea was the obvious one: let the small model take the routine first steps so the big model is called less. It
works, in that one call is skipped per task. But that call is the cheapest one in the loop, so the speed-up was not
statistically significant (about 1.6 s per run on a quiet server). I kept it in the plugin, off by default and labelled
experimental. The compressor is the same story so far: it works mechanically, but my live comparison against Hermes'
built-in compressor was inconclusive because tasks that size rarely need compaction. The strong results are the safety
features, where a fast, cautious decision is exactly what you want.

Along the way, testing found real problems: a server bug (HTTP 500 on any text containing the word "content"), and a Hermes quirk
where `auxiliary.approval.base_url` is silently ignored unless `provider: custom` is set. Both are documented.

**Try it**

Hermes does not list this plugin officially yet, so for now the way to use it is: fork Hermes, drop the plugin in, and
tune it for your use case.

```bash
git clone https://github.com/<you>/hermes-agent && cd hermes-agent
git clone https://github.com/abhishek085/jevcontrol-hermes plugins/jev-control
hermes plugins enable jev-control
hermes jev-control setup --preset observe --spark-url http://localhost:8400/v1 --api decide | sh
hermes jev-control doctor
```

Start in `observe` mode (log only) and read `hermes jev-control report` after a day of normal use. You need a decision
model server: a Jev-style `/v1/decide` API, or any OpenAI-compatible server that returns logprobs.

**Caveats, plainly:** small test sets that I labelled myself; one main model and one decision model; the decision
model can be confidently wrong, which is why every blocking feature has a log-only mode and a threshold.

**Thanks** to Nous Research for Hermes Agent and its plugin system, to the Langfuse team, to the Qwen and Gemma teams, and to
everyone behind vLLM, llama.cpp and mlx-lm. spark-s1 and JevControl are part of the Nokast open-source AI community; *Jev*
and *System One* are TypeSafe AI's names, and open-spark-Jev is an independent implementation inspired by them.

Repo: https://github.com/abhishek085/jevcontrol-hermes — issues, test cases and results from other setups are very welcome.

---

## Short post (LinkedIn / X thread opener)

Your agent's 27B model is answering questions a 4B model can answer in 80 ms.

I built **jevcontrol-hermes**, a plugin for Hermes Agent that gives it a reflex layer: a small decision model that handles
the tiny pick-one decisions in an agent loop.

🛡️ Privacy guard: stops a search or command from leaking credentials or personal data (caught 18/19, 0/17 false blocks)
🧠 Memory gatekeeper: keeps long-term memory clean (17/17 misfiled writes blocked, 10/10 real facts kept)
⚡ Approval guard: same safety as a 27B model on shell-command approvals, ~30× faster (88 ms vs 2.5 s)
🔎 Search picker + 🗜️ context compressor: leaner context, no LLM-written summary

No Hermes core changes. Off by default. Fails open. One command (`hermes jev-control report`) shows what it did.

Honest bit: my first idea, letting the small model skip routine LLM calls, gave no significant speed-up, so it's
experimental. Safety is where it shines.

Fork Hermes, drop it in, tune it for your use case → https://github.com/abhishek085/jevcontrol-hermes

Built on Hermes Agent (@NousResearch), spark-s1 / JevControl, Langfuse.

---

## One-liners

- "A reflex layer for Hermes Agent: a 4B decision model handles the small decisions so the big model only thinks."
- "Privacy guard, memory gatekeeper and approval guard for Hermes, in ~80 ms."
- "I measured it, including the parts that did not work. Tool-step routing: no significant speed-up. Safety checks: strong."
- "Fork Hermes, add the plugin, tune it for your agent. Off by default, fails open."

## Suggested assets

- Terminal screenshot of `hermes jev-control report` after a day of use.
- A Langfuse trace showing a `BLOCKED by the jev-control privacy guard` tool result.
- The before/after of an approval verdict: 2.5 s (27B) vs 88 ms (spark-s1).
