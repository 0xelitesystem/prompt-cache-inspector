# Prompt Cache Inspector

Paste two consecutive Anthropic request bodies and find the exact byte where your cached prefix broke, whether your breakpoints are legal, and whether caching pays for itself at your request rate.

## Live demo

https://0xelitesystem.github.io/prompt-cache-inspector/

## Features

Prompt caching fails silently. There is no error and no warning - `cache_read_input_tokens` just sits at zero. This tool localizes that failure from artifacts you already have: two logged request bodies. It instruments nothing and calls nothing.

- **Render-order walk, not a JSON diff.** Both bodies are walked into an ordered list of cache blocks in the documented render order: every entry of `tools` in array order, then `system`, then every content block of every message. This is the difference that matters. A pretty-printed diff of two bodies can look completely harmless while the render order shows the prefix dying at block 0.
- **Exact divergence location.** The longest common block prefix, then the longest common byte prefix inside the first differing block. You get the block index, its segment and type, the byte offset, and a context window either side with the divergence point marked.
- **A diagnosis engine with named causes.** Timestamp or date, UUID or request id, tool reorder, tool added/removed/edited, key reorder from a nondeterministic serializer, model change, whitespace only, session or user id, or a plain `UNCLASSIFIED` with the raw bytes shown. It never claims a cause it cannot show evidence for.
- **Breakpoint legality audit.** Per marker: block index, estimated prefix tokens as a band, a PASS / FAIL / BORDERLINE verdict against the model's minimum, whether the breakpoint sits before or after the divergence (one that sits after can never produce a read), the count against the documented maximum of 4, and the mixed-TTL ordering rule.
- **20-block lookback check.** Counts backward from the breakpoint, counting the breakpoint itself as position 1, and shows the arithmetic when the prior entry falls outside: `breakpoint at block 35; nearest prior entry at block 9; window covers blocks 35 down to 16; entry is 7 positions outside`.
- **A TTL clock built on the rule everyone gets wrong.** Lifetime runs from the **start** of the request that writes or reads the entry, not from the end of its response. When a request streams for longer than the TTL, the entry is dead before its own response finishes and no follow-up can ever hit it on that tier. That case gets its own panel and its own loud verdict.
- **Economics with the algebra shown.** No headline savings percentage anywhere. It prints the formula, solves for the break-even request count from the multipliers in the rates block, and tabulates net cost at N = 1, 2, 3, 5, 10, 50.
- **ITPM side effect.** Cache reads do not count toward input-tokens-per-minute on most models, so a hit rate is a throughput multiplier as well as a discount. Includes the documented exception, and the `input_tokens` field trap that makes teams under-count their own prompts.
- **Silent-invalidator static scan** over `tools` and `system` - everything ahead of the messages segment - so the tool is still useful when you only have one request body.
- **Single request mode**, a **Load sample** that fires four separate traps in one click, dark and light themes persisted across visits, keyboard access, copy buttons, and no external dependencies of any kind.

## How it works

Everything runs in your browser. There is no backend, no API key, no network request, and no telemetry.

The one invariant the whole tool is built on: **prompt caching is a prefix match, and the render order is `tools`, then `system`, then `messages`.** A single byte change anywhere in the prefix invalidates every breakpoint at or after that position. That is why a tool reorder is so destructive: tools render first, so reordering the array changes the prefix hash from block 0 and takes an untouched system prompt down with it.

Serialization is deterministic and order-preserving. Object keys are emitted in the order they appear in your JSON and are **never** sorted, because a nondeterministic serializer on your side is itself a real invalidator the tool needs to be able to name. `cache_control` markers are treated as caching directives rather than prompt content, so they are excluded from the compared bytes and tracked separately.

The page is split into two visually distinct zones so a stale vendor number can never make a correct diagnosis look wrong:

- **Computed from your input** - the render walk, the divergence block and byte offset, breakpoint positions and counts, the lookback arithmetic, the TTL arithmetic. Byte-exact, no vendor constant involved.
- **Computed from the rates table** - floor verdicts, prices, break-even, TTL durations, the breakpoint maximum, the lookback size. All of it reads from one editable JSON block on the page, stamped with the date it was confirmed and linked to its source.

### What it computes exactly vs what it estimates

Exact: the render order, the block index and byte offset of the first divergence, breakpoint positions and counts, and the lookback and TTL arithmetic once you supply the window size and tier duration.

Estimated: **every token count on the page.** They come from a character heuristic - characters divided by a band of 3.2 to 4.4 characters per token - and always render as a range, never as a point value. Any prefix whose band straddles the model's minimum renders BORDERLINE rather than as a pass or a fail; confirm those with the `count_tokens` endpoint. OpenAI tokenizers such as `tiktoken` are **wrong for Claude** and this tool does not use one, nor does it ship a Claude tokenizer.

The tool never contacts an API and measures nothing about your actual traffic. It cannot tell you your real hit rate. It names the exact fields to read instead.

### The four traps, and the sample that fires all of them

Press **Load sample** for a synthetic two-turn coding-agent pair, built so that a single click demonstrates:

1. **Tool reorder.** The same four tools in a different order between N and N+1. A JSON diff shows nothing meaningful; the render-order analysis shows the prefix dying at block 0 and taking a system prompt that never changed a character with it.
2. **A second, independent invalidator.** A `Current date: ...` line inside the system prompt, plus a session id and a turn counter. The tool reports the *first* divergence and lists the rest, so fixing one does not just move the failure a few blocks along.
3. **The lookback miss.** Request N+1 appends twelve tool call and result pairs - 24 tool_use and tool_result blocks - plus a summary and the new user turn, pushing its breakpoint to block 35 while the entry request N wrote sits at block 9. That is seven positions outside a window reaching down to block 16. This is what silently kills agentic turns.
4. **The floor that does not track tier or recency.** The sample prefix is sized so it **passes** on Opus 4.8 and Sonnet 4.5 and **fails** on Opus 4.7 and Haiku 4.5. Change the model selector and nothing else, and the verdict flips. Lead with the consequence rather than the adjective: a roughly 1,500-token prefix caches on Opus 4.8 and Sonnet 4.5 and silently does not on Opus 4.7 or Haiku 4.5. Within the Opus line the minimums decrease cleanly as versions advance; what does not hold is any relationship *across* families, where the newer Haiku 4.5 sits at 4,096 while the older Sonnet 4.5 sits at 1,024.

The TTL inputs are prefilled with request N streaming for 380 seconds and N+1 starting 60 seconds after N completes, so the 5-minute entry was already dead before N's own response finished. Switching the tier to 1h fixes it and visibly changes the break-even count, which is the honest trade rather than a free upgrade.

The sample is labelled synthetic in the interface. It is illustrative, not a benchmark.

## Measuring your real hit rate

This tool estimates. The API measures. Read these from the `usage` object on the response, or from the `message_start` event when streaming:

| Field | What it is |
| --- | --- |
| `cache_creation_input_tokens` | Tokens written to the cache when creating a new entry |
| `cache_read_input_tokens` | Tokens served from the cache on this request |
| `input_tokens` | **Only** the tokens after the last cache breakpoint |
| `cache_creation.ephemeral_5m_input_tokens` | Per-tier breakdown of the write, 5-minute portion |
| `cache_creation.ephemeral_1h_input_tokens` | Per-tier breakdown of the write, 1-hour portion |

```
total_input_tokens = cache_read_input_tokens + cache_creation_input_tokens + input_tokens
```

If `cache_read_input_tokens` is zero across repeated requests with a stable prefix, a silent invalidator is at work - that is the case this tool exists to localize. If both `cache_creation_input_tokens` and `cache_read_input_tokens` are zero, the prompt was not cached at all, most often because the prefix fell under the model's minimum. Neither case returns an error.

Reading `input_tokens` alone will badly under-count your prompt, and the better your caching gets, the more wrong that number becomes.

## Scope

This models **Anthropic's documented prefix caching only.** Other providers implement caching differently and are out of scope; none of the render order, minimums, multipliers or lookback arithmetic here transfers to them. The tool detects several non-Anthropic body shapes and says so rather than producing a confident wrong answer.

## Facts verified against

Confirmed on 2026-08-07 against these primary sources, and stamped with that date on the page:

- Write multipliers (1.25x for the 5-minute TTL, 2x for the 1-hour TTL) and the 0.1x read multiplier - https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- The full minimum cacheable prefix table, and that those minimums apply on every platform where each model is available - same source
- The maximum of 4 explicit breakpoints, and that automatic caching on top of 4 existing explicit breakpoints returns a 400 - same source
- The 20-block lookback window, checking at most 20 positions per breakpoint and counting the breakpoint itself as the first - same source
- The invalidation hierarchy table, including that the thinking-parameters and effort-setting rows are **model-specific** in the tools and system columns rather than preserved - same source
- That a cache entry only becomes available after the first response begins - same source
- That the lifetime is measured from the start of the request that writes or reads the entry, not from the end of its response - same source
- The `usage` field names and the `cache_creation` per-tier breakdown - same source
- The mixed-TTL ordering rule: both tiers may appear in one request, but entries with a longer TTL must come before shorter ones - same source
- That `cache_read_input_tokens` do not count toward ITPM for most models, the one documented model that does count them (Claude Haiku 3.5, carrying the dagger marker in the rate-limit tables), and that `input_tokens` means only the tokens after the last cache breakpoint - https://platform.claude.com/docs/en/api/rate-limits
- The worked ITPM example the tool's formula reproduces: a 2,000,000 ITPM limit at an 80% hit rate is 10,000,000 effective input tokens per minute - same source
- Base input price per million tokens for every model in the rates block - https://platform.claude.com/docs/en/about-claude/models/overview

Two notes on the rates block. Haiku 3.5 is retired on the Claude API and still served on Amazon Bedrock and Google Cloud; it is kept because it is the one documented ITPM exception, and its price row carries that caveat. The documented minimum-prefix table also covers Mythos Preview (2,048) and Opus 4.1, Opus 4 and Sonnet 4 (1,024); those are listed under `models_documented_elsewhere` rather than in the model selector, because this tool ships no verified price for them.

Every one of those values lives in the editable rates block on the page rather than in the code, so you can correct a stale row yourself and the answers update with it. Prices and floors change; the byte-level diagnosis does not depend on any of them.

## Privacy

Everything runs locally in your browser. No request bodies, timings, rates or results leave the page. There is no backend, no analytics, no cookies, no local storage of your input, and no external dependencies - a single self-contained HTML file with all CSS and JavaScript inline. The only thing persisted is your light or dark theme preference.

Request bodies routinely contain proprietary system prompts and customer data. That is precisely why this tool has no network path at all.

## License

MIT. See [LICENSE](LICENSE).

## More

- https://0xelitesystem.github.io/
- https://elitesystem.ai
