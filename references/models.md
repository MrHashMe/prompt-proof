# Model facts (checked 2026-11-20 against vendor documentation)

A dated cache, taken from documentation text and not from live requests. State as current only what is written here or on a vendor page read in this session. When this file is more than about 60 days old, the target is not listed, or a wrong parameter would break production, read the vendor's current page first if you can browse (links at the end); otherwise mark the fact `[UNVERIFIED]`. Parameter names are for each vendor's own API. A gateway, an SDK or a consumer app may rename or hide them.

## True of all three families

- Thinking depth is an API setting, not prompt text, and none of these models has a setting that turns thinking off.
- The output cap covers thinking as well as the visible answer, so a cap sized for the answer alone truncates it or returns nothing visible.
- Omit `temperature`, `top_p` and `top_k`: they are rejected, unsupported or ignored.
- Claude and Gemini do not accept a prefilled final assistant turn. Get machine-readable output through the structured-output feature.
- Each API caches a shared prompt prefix, and editing earlier turns loses the cache and the saved reasoning, or fails. Keep stable content first and put changes in the newest turn. Do not pad a short prompt to reach a caching threshold.

## Claude Fable 5.2 and Mythos 5.2

`claude-fable-5-2`, `claude-mythos-5-2` (Mythos for verified organizations only).

- Context 1M tokens, output up to 128k. `max_tokens` caps thinking and the reply together.
- `output_config.effort`: `low`, `medium`, `high`, `xhigh`, `max`. The API default is `high`; Claude Code defaults this model to `medium`. Start at `high` and test the neighbouring levels on your own cases.
- Adaptive thinking is always on. `thinking: {type: "disabled"}` and `budget_tokens` return 400. In the vendor's testing, text telling the model not to deliberate did not stop it thinking at `medium` and above, so delete such lines and lower the effort.
- A prefilled assistant turn returns 400. Forced `tool_choice` (`any`, or a named tool) returns 400: use `auto` with an explicit instruction and `strict: true` tools, or JSON outputs through `output_config.format`. Organizations with customer-managed encryption keys have neither strict tools nor JSON outputs, and validate in code.
- Changing `system`, `tools` or earlier messages mid-conversation and then sending thinking blocks back fails the prefix check with 400. Freeze `system` and `tools` at session start, keep the history append-only, and make mid-session changes with a `role: "system"` message. Per-message effort (beta) is set the same way; changing the top-level value can invalidate the cache.
- Prompt caching needs a prefix of at least 512 tokens.
- A response can carry `thinking` blocks before `text`, so code selects blocks by type. Updates between tool calls arrive as `thinking` blocks, which are empty unless `thinking.display` is set.
- At `xhigh` and `max` the model may start subagents to check its own work, which costs tokens.
- Prompt wording: the vendor's guide for this model backs the general rules in `SKILL.md`: a finish line so that it does not stop while work is owed, the approval condition in place of a ban because "never" is taken as absolute, stated code ownership, one-line progress updates, costed subagent briefs, and a final report for a reader with little context. It adds that an approval should say exactly what it covers, not give a one-word go-ahead.

## GPT-6.2 Astra and Sol

`gpt-6.2-astra`, `gpt-6.2-sol`.

- Context 1,050,000 tokens, input up to 922,000, output up to 128,000. Above 272,000 input tokens the price rises: input and cached input double, output costs 1.5 times.
- `reasoning.effort` (`reasoning_effort` on Chat Completions): Astra accepts `low`, `medium`, `high`, `xhigh`, `max`, and both models reject `none`. Sol's own list was not found, so check its model page. Start at `medium`.
- `max_output_tokens` covers reasoning, visible and formatting tokens. Reaching it returns `status: incomplete`, possibly with no visible text, so leave at least 25,000 tokens of room at first.
- `temperature`, `top_p`, `logprobs` and `top_logprobs` are unsupported.
- Tool calling needs the Responses API. Chat Completions works without tools.
- Top-level `instructions` are appended after `input`, so reusable instructions go in a leading `developer` message.
- Editing the prefix, some model changes and end-user changes drop the saved reasoning items. To change `reasoning.effort` or `text.format` between responses, send a `configuration_update` item, which keeps the cached prefix, and send a fresh one after compaction.
- `prompt_cache_options.ttl` replaces `prompt_cache_retention`. Flex processing is not offered.
- Prompt wording: Astra is more concise and direct by default, so revisit "be concise" lines when porting to it. The vendor's guide otherwise backs the general rules in `SKILL.md`: room for reasonable assumptions, the outcome and the decisions that need the user, a completeness check, a final answer that stands alone, and the codebase's conventions.

## Gemini 4 Carbon, Argon, Neon and Hydrogen

`gemini-4-carbon`, `gemini-4-argon`, `gemini-4-neon`, `gemini-4-hydrogen`.

- The vendor's positioning: Carbon is the strongest reasoner; Argon is faster and cheaper per turn than Carbon; Neon is frontier-level and fast at whole tasks; Hydrogen is for speed in everyday coding and agents.
- `thinking_level`: Carbon and Argon take `LOW`, `MEDIUM`, `HIGH` (default `MEDIUM`; `XHIGH` does not exist there). Neon and Hydrogen take `LOW`, `MEDIUM`, `HIGH`, `XHIGH` (default `HIGH`). Values are uppercase on `GenerateContent` and lowercase on the Interactions API. `thinking_budget` and `MINIMAL` return 400.
- `temperature`, `top_p` and `top_k` are ignored. `frequency_penalty`, `presence_penalty` and `candidate_count` above 1 return 400.
- Input and output share one limit of 1,048,576 tokens, thinking and function calls included.
- One foreground response segment holds at most 65,536 output tokens, thinking included; on Carbon and Argon a non-streaming call holds 32,768. For more, set `max_output_tokens` above that figure (up to 1,048,576; only a larger value returns 400) and collect the segments: a current Google Gen AI SDK follows the `continuation_token` by itself, raw REST needs a loop, and the Interactions API can run the request with `background=true` and `store=true`. This needs `v1beta1`. Older SDK versions and the OpenAI compatibility layer cannot retrieve the continuation, and a continuation segment cannot change `model`, `system_instruction` or `tools`. A larger cap gives the model room to reason. It does not make answers longer, and the model does not follow length targets.
- Caching is implicit only, on shared prefixes of 4,096 tokens or more. Explicit cache control is announced for Hydrogen and later models from 2027-01-12.
- Structured output: `response_mime_type` with `response_schema`. History cannot end with a `model` turn.
- Tool calls: return every `thought_signature` in its original part. In the active turn a missing signature returns 400. With parallel calls only the first `functionCall` part carries one, and copying it to the others also returns 400.
- Video: `media_processing` defaults to `"static"`. Set `"agentic"` for agentic video understanding.

## Porting between them

| | Claude Fable 5.2 | GPT-6.2 | Gemini 4 |
|---|---|---|---|
| Depth setting | `output_config.effort` | `reasoning.effort` | `thinking_level` |
| Where to start | `high` (the default) | `medium` | the default: `MEDIUM` on Carbon and Argon, `HIGH` on Neon and Hydrogen |
| Highest level | `max` | `max` | `HIGH` on Carbon and Argon, `XHIGH` on Neon and Hydrogen |
| Reusable instructions | `system` | leading `developer` message | `system_instruction` |
| Structured output | `output_config.format`, `strict: true` tools | `text.format` | `response_mime_type` and `response_schema` |
| Output cap | `max_tokens`, up to 128k | `max_output_tokens`, up to 128,000 | `max_output_tokens`, 65,536 or 32,768 per segment |
| Caching minimum | 512 tokens | not checked | 4,096 tokens |

Levels with the same name do not behave the same across vendors. Pick the nearest level, then re-test on the target.

## Not confirmed

- The API's default effort for GPT-6.2, Sol's list of levels, and OpenAI's caching minimum.
- The status code Gemini returns for a prefilled turn, and what `v1` returns for a long output.
- Live behaviour: none of the above was tested with a request.

## Any other target

Older or smaller models, open-weight models, coding tools and image, video or voice generators are not cached here. The general rules in `SKILL.md` apply, and the target's facts come from its own current documentation or are marked `[UNVERIFIED]`.

## Where to re-check

- Claude: platform.claude.com/docs/en/models/fable-5-2/whats-new-fable-5-2, platform.claude.com/docs/en/models/fable-5-2/migration-guide, platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5-2
- OpenAI: developers.openai.com/api/docs/guides/latest-model, developers.openai.com/api/docs/guides/reasoning, developers.openai.com/api/docs/models
- Gemini: docs.cloud.google.com/gemini-enterprise-agent-platform/models/start/get-started-with-gemini-4, the per-model guides under docs.cloud.google.com/gemini-enterprise-agent-platform/models/guides/, aistudio.google.com/docs/long-decode
