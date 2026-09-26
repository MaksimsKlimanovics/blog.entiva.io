---
layout: post
title: "Building an AI Estimate Builder in Business Central: From Foundry to Anthropic, and Why Search Mattered More Than the Model"
description: "Turning vehicle inspection findings into a reviewable estimate inside a Business Central GMS: GPT-5 compatibility, the move to Anthropic, and vehicle data provider search, synonyms and validation."
date: 2026-09-25
category: ai-copilot
tags: [AL, Copilot, Anthropic, Integrations]
---

Asking a language model to write a repair estimate is easy. It will happily produce one in seconds:

> Replace front brake discs and pads. 1.2 hours. Parts in stock.

Every word of that is plausible. None of it is verified. The labour time may belong to a different operation, the pads may not fit this vehicle, and "in stock" may mean "the model has never seen your stock table".

A workshop cannot send the customer plausible. It has to send an estimate built from the correct labour operation, parts that fit and are actually available, and prices calculated by the same business logic that invoices the job later.

This post is about building that — an **Estimate Builder** inside a Business Central-based Garage Management Software (GMS) — and about the less glamorous part: most of the problems I hit were not in the model at all.

## Who this is for

- **Business Central developers** adding Copilot or AI features to their own extensions, especially if you are calling the built-in `Azure OpenAI` codeunit and wondering why a newer model refuses to cooperate.
- **Architects and product owners** of vertical solutions (GMS, field service, distribution) who want AI that proposes work from real business data rather than inventing it.
- **Anyone integrating an LLM with a metered, external data provider**, where every lookup costs money and "just try again with different wording" has an invoice attached.

If you are looking for a model benchmark, this is not it. I have no comparative data on estimating quality, and I am not going to pretend otherwise.

## The problem: inspection findings do not turn into estimates by themselves

The workshop process looks like this. A technician inspects the vehicle and records findings. A service advisor then has to turn those findings into work the customer can understand and approve. That means:

- choosing applicable service packages,
- finding the right labour operations and repair times,
- identifying parts and checking they are available,
- explaining the proposed work in customer language.

All of the data needed already exists. The GMS holds the inspection findings, service packages, vehicle history and stock, and it integrates with a vehicle data provider for repair times and associated parts. The advisor is the integration layer, clicking between them by hand, for every vehicle, every day.

The obvious answer is "let the AI do it". The obvious problem is that the AI will do *something*, confidently, whether or not the underlying data supports it.

So my requirements were practical:

1. **Use the existing Business Central logic.** Prices and package insertion stay in the routines that already do them correctly.
2. **Explain the source of every proposed job.** A package, a provider operation, or an honest "I could not find this".
3. **Check parts availability** against real stock, not against optimism.
4. **Let the advisor review everything** before a single line is written to the estimate.

A plausible description is not enough if the labour operation is wrong or the part does not fit.

## What the Estimate Builder does

The Estimate Builder starts from the inspection findings and creates temporary proposal groups and lines. The advisor sees the source, linked findings, proposed work and rationale, reviews the customer wording, and chooses what to keep.

I kept the lookup order explicit, so the agent uses the existing business data consistently instead of improvising:

1. Search currently applicable service packages and inspect their contents.
2. If the direct search is inconclusive, inspect packages previously used for this vehicle, checking that a current version is still selectable.
3. Use the vehicle data provider for suitable labour operations and associated parts when packages do not cover the work.
4. Search local stock for parts needed by provider-based and manual proposals.
5. Leave unresolved work for the advisor, with an explanation of what is missing.

Existing GMS routines still calculate document prices and insert service packages. Historical package use is a useful clue, but it does not establish that the same package fits today's finding.

The shape of the solution is simple. Getting it to behave was not. I ran into problems at several levels: the core AI integration did not accept the selected model's request parameters, the automated vehicle lookup behaved differently from the interactive page, and search results quietly omitted data the agent needed. The rest of this post follows those problems in the order they bit.

> **Status, so nobody is misled:** the development snapshot is 25 September 2026. The Anthropic integration is committed on its feature branch; the latest synonym-aware provider search changes are still working-tree changes. This is an engineering account of the implementation, with live acceptance testing still to complete.

## Problem 1: the model call failed, and told me nothing

I started with Business Central's `Azure OpenAI` codeunit and `GenerateChatCompletion`. The GMS already used this approach for VI Estimate sales descriptions, so reusing it was a reasonable starting point. Microsoft documents this codeunit as the AL interface to Azure OpenAI. [Microsoft's Azure OpenAI codeunit reference](https://learn.microsoft.com/en-us/dynamics365/business-central/application/system-application/codeunit/system.ai.azure-openai).

The debugger stopped around generation. The operation response was admirably concise:

```text
GetStatusCode() = 0
GetError() = empty
```

Status zero, no error. Everything is fine, except nothing happened.

I tested the Foundry deployment separately with PowerShell. That request succeeded: the API key was accepted and the deployment was reachable from my machine. Case closed? Not quite. The PowerShell test used the Responses API; the AL code used Business Central's Chat Completions wrapper. I had verified the credentials and deployment from *my* machine, but still needed to verify the request *Business Central* was making from *its* server, including the endpoint and parameters.

I added a connection test on the setup page and captured more error detail. That exposed an underlying `ALCopilotFunctions.GenerateChatCompletion` exception with the message `authorization` and runtime code `DotNetInvoke:InvalidOperation`.

Later, the failure finally became a concrete HTTP 400:

```text
Unsupported parameter: 'max_tokens' is not supported with this model.
Use 'max_completion_tokens' instead.
```

The original code configured its output limit through `SetMaxTokens`, and the selected deployment rejected the resulting parameter. At last, a specific request-compatibility problem to investigate, rather than a vague "authorization".

### GPT-5 and newer models were not drop-in replacements through the tested core integration

The failure came through the `System.AI` integration in Microsoft's System Application, version `28.3.52162.52222`. That is the specific core-app path I mean here. In our test, its generated request used `max_tokens`, while the selected newer GPT deployment required `max_completion_tokens`. Microsoft documents this parameter difference for GPT-5 reasoning models; other request settings also depend on the model. [Microsoft's Chat Completions guidance](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/chatgpt).

For GPT-5 and later models requiring that contract, the tested wrapper request was incompatible. Changing the deployment name in setup did not make it compatible. Supporting the model required a compatible core implementation, or a direct client that could send the parameters the model accepted.

To be fair to Microsoft: this limitation is specific to the version and extension API path we tested. Business Central's built-in agents have their own model support, including documented GPT-5.3-chat use. That does not establish compatibility for a custom extension calling `GenerateChatCompletion` with our configuration. [Business Central's Copilot and agent configuration](https://learn.microsoft.com/en-us/dynamics365/business-central/enable-ai).

### Why I moved to Anthropic

For the Estimate Builder, I wanted control over the HTTP request, the tool exchange and error handling. I moved the generation path to Anthropic's Messages API while retaining the Business Central review experience and the GMS tools.

A direct Foundry client was another possible approach. The decision was about integration control, not about which model is smarter. Again: no benchmark, no claim.

## Keeping business tools separate from the provider

I added a dedicated Claude client and an explicit conversation loop in AL. The existing tool declarations still define the names, descriptions and argument schemas. The client converts those declarations into Anthropic tool definitions, so I do not maintain two versions of every business tool — one per provider, both slowly drifting apart.

The loop sends the system prompt and messages, receives the complete assistant content, executes requested business tools in AL, and returns results with the matching tool-use identifiers. This follows Anthropic's documented `tool_use` and `tool_result` exchange. [Anthropic's tool-call documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls).

A tool result needs its corresponding request in the conversation. In the original implementation, I enlarged the history window to keep the initial context and complete tool exchanges together. The direct client preserves the full assistant content and matching results explicitly.

The direct client also exposes HTTP status and provider error text — a refreshing change from status zero. The setup connection test uses that same client, which makes it a relevant diagnostic for the actual feature path rather than a test of something adjacent.

Business Central still provides the PromptDialog, capability activation, GMS entitlement checks and advisor review. Estimate Builder customer descriptions use the new connection, while the existing standalone VI Estimate Sales Descriptions feature keeps its original provider.

## Problem 2: the provider lookup worked in the page, but not through the tool

One early run produced a tyre service package and manual brake groups. Its explanation said the vehicle data provider lookup had failed. Yet opening Repair Times interactively in the web client produced a result for the same vehicle.

So the data was there. The agent just could not reach it.

The interactive workflow and the non-interactive lookup did not resolve vehicles the same way. The API lookup tried to resolve the vehicle through its registration, while the vehicle could already have a saved provider vehicle identifier.

I changed the lookup to use the linked vehicle's saved identifier first, then try non-interactive registration resolution. If neither works, the error tells the advisor to open Repair Times, select and confirm the vehicle, and retry.

This was a vehicle-resolution issue in the integration. Rewriting the prompt would have produced a more eloquent explanation of the same failed lookup.

## Problem 3: the Brakes category was missing

In another real VIE run, the provider's category response was truncated and did not include Brakes. The agent, reasonably, created manual groups for front and rear brake work — even though the provider contained suitable operations.

Before touching the prompt again, I checked exactly what the tool had returned. The category was simply not in the response. The model did not ignore Brakes; it was never told Brakes existed. I had to fix the tool's result format and give the agent a way to search and continue through the data.

The latest changes:

- Return a flat, compact category list with identifiers, names and short descriptions, without an arbitrary category-count cutoff.
- Accept `searchText` on both the category and operation tools.
- Support paging on operation results with `skip`, `top`, `truncated` and `nextSkip`, so the agent can continue beyond the first page.
- Include result counts, so the agent can distinguish an empty search from an incomplete one. If more operations exist, it must continue paging before deciding there is no suitable match.

## Problem 4: workshop vocabulary is not the provider's vocabulary

Consider an inspection finding:

```text
Front brake discs replacement
```

The relevant provider operation might be:

```text
Remove and install front brake disks
```

A literal phrase search misses that match. The words differ in order, spelling, number and action wording, while describing the same work. Any mechanic reads them as identical. A `LIKE` filter does not.

I first added workshop vocabulary and progressively broader searches to the prompt. That helped define the expected behaviour, but I wanted the tools themselves to handle wording consistently, not rely on the model remembering to be creative. The latest provider search changes implement the matching in AL:

- Fold case, accents and punctuation.
- Apply simple singularisation.
- Resolve multi-word synonyms before individual words.
- Remove configured filler terms.
- Map aliases to canonical terms.
- Require all query tokens for a full match; consider partial candidates only when no full match exists.
- Weight name matches above description matches, and return a score and match type.

The synonym setup distinguishes ordinary words, actions, positions and filler terms. Examples include `disc / disk / rotor`, replacement wording such as `renew / remove and install / R&I`, and position aliases such as `front axle / front` or `LH / left`.

These mappings are editable company data, with defaults seeded on install and upgrade. Validation rejects synonym chains, cycles and conflicting term types, and seeding preserves existing custom mappings. Adding an alias should not require an app deployment.

The mappings still need care: terms that are interchangeable in one context can describe different work in another. Matching must preserve the **component**, the **action** and the **position**:

- Inspecting a brake component does not cover replacing it.
- Rear work does not cover a front finding.
- A shared action word must not rescue an unrelated component.

The AL matcher rejects explicit action and position conflicts and labels partial results for further review. The prompt then asks the model to inspect the returned operations and confirm the intended work. A search score ranks candidates; it does not establish repair suitability or vehicle fitment.

And no, there is no embedding index or vector database in this search path. I used deterministic text processing and a configurable vocabulary, so I can inspect why an operation matched, reproduce its ranking, and test cases such as front versus rear or replacement versus inspection. When an advisor asks "why did it pick that?", "cosine similarity 0.83" is not an answer they will enjoy.

## Problem 5: every search costs money

Provider usage is metered. Changing search wording must not repeatedly fetch the same underlying data — otherwise the agent's persistence becomes a line item.

The latest implementation snapshots categories for the run's vehicle lookup, and operations for each fetched category. Filtering and paging operate on those snapshots. Asking for the next page or trying a broader term reuses the data already obtained.

The existing budget limits external lookup attempts, and failed and empty fetches are remembered to avoid repeated attempts. Stable ranking uses the original candidate order to break equal scores, keeping page boundaries predictable within the snapshot.

The prompt now explicitly requires narrowing and paging where applicable before falling back to manual work. It also asks the final explanation to identify the provider category and search attempted when a potentially matchable job remains manual.

## A text match still needs a stock and fitment check

Parts are searched by article or item reference first when an identifier is available. If that fails, or no identifier exists, the prompt requests progressively broader description searches: distinctive terms first, then fewer terms retaining the component, then a useful workshop synonym.

One honest caveat: the stock tool still uses literal field and word matching. The new canonical synonym engine applies to the provider category and operation tools; prompt-guided synonym searches help the stock path separately.

Candidates must have enough remaining availability at the VIE location. Existing estimate demand and included draft demand count against that availability, and quantities must be interpreted in the correct units. Package parts keep their configured identity rather than being silently substituted.

A similar description is only a candidate. Fitment still needs evidence, and creating an estimate does not reserve the inventory. Availability is checked again when the advisor keeps the proposal.

## I needed to see what the tools actually did

A final sentence such as "vehicle data provider lookup failed" is a summary, not a diagnosis. I added persistent tool-call history with the run, tool name, arguments, success flag, result summary, timestamp and metered-call information.

That lets me check whether a failure came from vehicle resolution, entitlements, an empty search, invalid arguments or an exhausted budget *before* deciding what to change — instead of rewriting the prompt and hoping.

The newest search telemetry records counts, paging and cache information without VIN, registration or customer data. Detailed tool history has a different purpose and can contain inspection or provider-error content, so it should not be mistaken for anonymous telemetry.

Optional web search is also bounded. The branch exposes an Anthropic server-side search tool with a use limit and a domain allowlist. The prompt treats web material as supporting research and prohibits sending vehicle identifiers to unrestricted web search. A public search result cannot establish a fitted part, an applicable labour time or an inspected fault. Dedicated licensed catalogue integrations remain a separate capability.

## The final write is separate, validated and boring — on purpose

Generated proposals and customer wording remain temporary until the advisor selects **Keep it**. The apply code rechecks document status, finding ownership, source fingerprints, package applicability and stock requirements, then writes accepted groups in one transaction.

If a finding has changed or stock is insufficient, the transaction fails instead of leaving a partially applied estimate. Keep performs no model calls. Customer authorisation and subsequent workshop processing remain with the advisor.

Discard leaves the estimate unchanged, although external lookup usage and integration-cache writes made during generation have already happened. The meter does not have an undo button.

Prompt configuration follows the same separation of responsibilities. Administrators can edit a company-specific UTF-8 prompt using the existing multiline editor, or restore the packaged default. Credentials are encrypted in company-scoped isolated storage. Prompt customisation survives upgrades, which also means existing custom prompts need deliberate review when new tool guidance is introduced.

## Where it stands

| Branch or stage | Recorded state | Significance |
| --- | --- | --- |
| `backup/vie-estimate-builder-cloud-base` | `7d4e0c619` | Preserves the first implementation, originally built on the Cloud-Version baseline. |
| Master-based feature | Base `cf5e60f3e`; feature `9fa879e8e` | Re-established the implementation on the intended master baseline. |
| Schema upgrade | `2610ac9a9` | Raised the app version to `28.3.0.5` after a new setup field existed in source but was missing from the installed runtime schema. |
| `feature/vie-estimate-builder-copilot` | `8f24aeaf0` | Retains the Azure OpenAI route and the diagnostics, vehicle-history and lookup improvements. |
| Anthropic branch preparation | `4ac2ff4fa` | Captures the diagnostics and lookup improvements on the path to the provider change. Its tree differs from the Copilot branch tip only by the bundled prototype ZIP. |
| `feature/vie-estimate-builder-anthropic` | `669f2c48f` | Adds the direct Messages API client, adapted tool loop, connection setup and Estimate Builder description generation. |
| Current working tree | Uncommitted; manifest `28.3.0.7` | Adds the provider synonym setup, normalised ranking, complete category results, cached paging, prompt updates and focused tests. |

Along the way I also hit a perfectly ordinary Business Central deployment issue: the new setup field compiled, but the installed schema did not contain it. I increased the app version for the upgrade. The build, the installed schema and the runtime behaviour are three different things, and all three need checking when delivering new fields.

The source includes tests for proposal validation and the provider contract, plus new cases for synonym matching, action and position mismatches, categories beyond the old limit, paging without gaps or duplicates, cache isolation and repeatable seeding. Earlier development recorded successful app and test compilation. Live provider, UI and end-to-end acceptance checks remain separate work; this post does not claim the latest changes have passed them.

## What I would tell someone building the same thing

- **Check what the tool returned before blaming the model.** Two of my "AI problems" were a vehicle-resolution bug and a truncated response.
- **Do not assume a newer model is a configuration change.** Through the tested `System.AI` path, GPT-5-class models needed request parameters the wrapper did not send.
- **Own the tool loop if you need to debug it.** A direct client with real status codes beats a wrapper that returns zero and silence.
- **Put domain vocabulary in the tools, not only in the prompt.** Deterministic, configurable matching is testable and explainable.
- **Cache metered data per run.** Retrying with different wording should not mean paying again.
- **Keep the write path free of AI.** Revalidate, write in one transaction, and let a human press the button.

For me, the useful engineering work is being able to trace a proposal all the way back: the inspection finding, the data returned by the tools, the matching operation, the selected part, and the checks performed before insertion. When a brake job becomes a manual group, I want to see why. When the agent selects a part, I want to know what supported that selection.

That is the standard I am working towards: a service advisor can review the proposed estimate, and a developer can explain how the system produced it. The remaining runtime checks need to confirm that the complete workflow behaves that way with real workshop data — not just plausibly.
