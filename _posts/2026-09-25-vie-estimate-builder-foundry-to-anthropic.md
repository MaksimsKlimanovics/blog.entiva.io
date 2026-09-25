---
layout: post
title: "Building an AI Estimate Builder in Business Central: From Foundry to Anthropic, and Why Search Mattered"
description: "Building Garage Hive's Estimate Builder: GPT-5 compatibility, the move to Anthropic, and Autodata search, synonyms and validation in Business Central."
date: 2026-09-25
category: ai-copilot
tags: [AL, Copilot, Anthropic, Integrations]
---

I wanted to reduce the work between a completed vehicle inspection and an estimate ready for the service advisor to review. Garage Hive already holds the findings, service packages, vehicle history and stock data, and integrates with Autodata for repair times. The Estimate Builder needed to connect those pieces into a usable workflow.

My requirements were practical: use the existing Business Central logic, explain where each proposed job came from, check parts availability, and let the advisor review everything before writing it to the estimate. A plausible description is not enough if the labour operation is wrong or the part does not fit.

During development, I ran into problems at several levels: the core AI integration did not accept the selected model's request parameters, the automated vehicle lookup behaved differently from the interactive page, and search results omitted data the agent needed. This is how the implementation evolved, including why I moved the Estimate Builder from the Business Central Azure OpenAI route targeting Microsoft Foundry to Anthropic's Messages API.

The development snapshot is 25 September 2026. The Anthropic integration is committed on its feature branch; the latest synonym-aware Autodata search changes are still working-tree changes. This is an engineering account of the implementation, with live acceptance testing still to complete.

## I started with the workshop process

A technician records inspection findings. A service advisor then has to turn those findings into work the customer can understand and approve. That means choosing applicable service packages, finding labour operations, identifying parts, checking availability, and explaining the proposed work.

The Estimate Builder starts from those findings and creates temporary proposal groups and lines. The advisor sees the source, linked findings, proposed work and rationale, reviews the customer wording, and chooses what to keep.

I kept the lookup order explicit so that the agent uses the existing business data consistently:

1. Search currently applicable service packages and inspect their contents.
2. If the direct search is inconclusive, inspect packages previously used for this vehicle, checking that a current version is still selectable.
3. Use Autodata for suitable labour operations and associated parts when packages do not cover the work.
4. Search local stock for parts needed by Autodata and manual proposals.
5. Leave unresolved work for the advisor, with an explanation of what is missing.

Existing Garage Hive routines still calculate document prices and insert service packages. Historical package use provides a useful clue, but does not establish that the same package fits today's finding.

## First, I needed to find out why the model call failed

I initially used Business Central's `Azure OpenAI` codeunit and `GenerateChatCompletion`. Garage Hive already used this approach for VI Estimate sales descriptions, so reusing it was a reasonable starting point. Microsoft documents this codeunit as the AL interface to Azure OpenAI. [Microsoft's Azure OpenAI codeunit reference](https://learn.microsoft.com/en-us/dynamics365/business-central/application/system-application/codeunit/system.ai.azure-openai).

The debugger stopped around generation, but the operation response gave me very little to investigate:

```text
GetStatusCode() = 0
GetError() = empty
```

I tested the Foundry deployment separately with PowerShell. That request succeeded: the API key was accepted and the deployment was accessible from my machine.

The PowerShell test used the Responses API. The AL code used Business Central's Chat Completions wrapper. I had verified the credentials and deployment from my machine, but still needed to verify the request Business Central was making from its server, including the endpoint and parameters.

I added a connection test on the setup page and captured more error details. That exposed an underlying `ALCopilotFunctions.GenerateChatCompletion` exception with the message `authorization` and runtime code `DotNetInvoke:InvalidOperation`.

Later, the failure became a concrete HTTP 400 response:

```text
Unsupported parameter: 'max_tokens' is not supported with this model.
Use 'max_completion_tokens' instead.
```

The original code configured its output limit through `SetMaxTokens`. The selected deployment rejected the resulting parameter. At that point, there was a specific request-compatibility problem to investigate, beyond the earlier authorization error.

## GPT-5 and newer models were not drop-in replacements through the tested Business Central core integration

The failure came through the `System.AI` integration in Microsoft's System Application, version `28.3.52162.52222`. That is the specific core-app path I mean here. In our test, its generated request used `max_tokens`, while the selected newer GPT deployment required `max_completion_tokens`. Microsoft documents this parameter difference for GPT-5 reasoning models; other request settings also depend on the model. [Microsoft's Chat Completions guidance](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/chatgpt).

For GPT-5 and later models requiring that contract, the tested wrapper request was incompatible. Changing the deployment name in setup did not make it compatible. Supporting the model required a compatible core implementation or a direct client that could send the parameters the model accepted.

This limitation is specific to the version and extension API path we tested. Microsoft's built-in Business Central agents have their own model support, including documented GPT-5.3-chat use. That does not establish compatibility for a custom extension calling `GenerateChatCompletion` with our configuration. [Business Central's Copilot and agent configuration](https://learn.microsoft.com/en-us/dynamics365/business-central/enable-ai).

For the Estimate Builder, I wanted control over the HTTP request, tool exchange and error handling. I moved the generation path to Anthropic's Messages API while retaining the Business Central review experience and Garage Hive tools. A direct Foundry client was another possible approach. The decision here was about integration control; I had no comparative model benchmark to claim an estimating-quality advantage.

## I kept the business tools separate from the provider

I added a dedicated Claude client and an explicit conversation loop in AL. The existing tool declarations still define the names, descriptions and argument schemas. The client converts those declarations into Anthropic tool definitions, so I do not have to maintain two versions of every business tool.

The loop sends the system prompt and messages, receives the complete assistant content, executes requested business tools in AL, and returns results with the matching tool-use identifiers. This follows Anthropic's documented `tool_use` and `tool_result` exchange. [Anthropic's tool-call documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls).

A tool result needs its corresponding request in the conversation. In the original implementation, I enlarged the history window to keep the initial context and complete tool exchanges together. The direct client preserves the full assistant content and matching results explicitly.

The direct client now exposes HTTP status and provider error text. The setup connection test uses that same client, which makes it a more relevant diagnostic for the actual feature path.

Business Central still provides the PromptDialog, capability activation, Garage Hive entitlement checks and advisor review. Estimate Builder customer descriptions use the new connection, while the existing standalone VI Estimate Sales Descriptions feature retains its original provider.

## Next, Autodata worked in the page but failed through the tool

One early run produced a tyre service package and manual brake groups. Its explanation said Autodata lookup had failed. Yet opening Repair Times interactively in the web client produced a result for the vehicle.

The interactive workflow and the non-interactive lookup did not have identical vehicle-resolution behavior. The API lookup tried to resolve the vehicle through its registration, while the vehicle could already have a saved Autodata vehicle identifier, or MID.

I changed the lookup to use the linked vehicle's saved MID first, then try non-interactive registration resolution. If neither works, the error tells the advisor to open Repair Times, select and confirm the vehicle, and retry.

This was a vehicle-resolution issue in the integration. Changing the model prompt would have left the failing lookup in place.

## The next problem was a missing Brakes category

In another real VIE run, the Autodata category response was truncated and did not include Brakes. The agent created manual groups for front and rear brake work even though Autodata contained suitable operations.

Before changing the prompt again, I needed to check exactly what the tool had returned. The relevant category was missing from that response. I had to fix the tool's result format and give the agent a way to search and continue through the data.

The latest changes return a flat, compact category list with identifiers, names and short descriptions, without an arbitrary category-count cutoff. Category and operation tools accept `searchText`. Operation results support paging with `skip`, `top`, `truncated` and `nextSkip`, so the agent has a way to continue beyond the first page.

The response includes result counts so the agent can distinguish an empty search from an incomplete result. If more operations exist, it needs to continue paging before deciding there is no suitable match.

## Workshop vocabulary became part of the search implementation

Consider an inspection finding expressed as:

```text
Front brake discs replacement
```

The relevant operation might be:

```text
Remove and install front brake disks
```

A literal phrase search can miss that match. The words differ in order, spelling, number and action wording, while describing the same intended work in this example.

I first added workshop vocabulary and progressively broader searches to the prompt. That helped define the expected behaviour, but I also wanted the tools to handle the same wording consistently. The latest Autodata changes implement that matching in AL:

- Fold case, accents and punctuation.
- Apply simple singularisation.
- Resolve multi-word synonyms before individual words.
- Remove configured filler terms.
- Map aliases to canonical terms.
- Require all query tokens for a full match; consider partial candidates when no full match exists.
- Weight name matches above description matches and return a score and match type.

The synonym setup distinguishes ordinary words, actions, positions and filler terms. Examples include `disc / disk / rotor`, replacement wording such as `renew / remove and install / R&I`, and position aliases such as `front axle / front` or `LH / left`.

These mappings are editable company data, with defaults seeded on install and upgrade. Validation rejects synonym chains, cycles and conflicting term types, and seeding preserves existing custom mappings.

I wanted workshop vocabulary to be maintainable through setup. Adding an alias should not require an app deployment. The mappings still need care: terms that are interchangeable in one context can describe different work in another.

In particular, matching must preserve the component, action and position. Inspecting a brake component does not cover replacing it. Rear work does not cover a front finding. A shared action word must not rescue an unrelated component.

The AL matcher rejects explicit action and position conflicts and labels partial results for further review. The prompt then asks the model to inspect the returned operations and confirm the intended work. A search score ranks candidates; it does not establish repair suitability or vehicle fitment.

For this implementation, I used deterministic text processing and a configurable vocabulary. I can inspect why an operation matched, reproduce its ranking, and test examples such as front versus rear or replacement versus inspection. There is no embedding index or vector database in this search path.

## Repeated searches should reuse the data already fetched

Autodata usage is metered, so changing search wording must not repeatedly fetch the same underlying data.

The latest implementation snapshots categories for the run's vehicle lookup and operations for each fetched category. Filtering and paging operate on those snapshots. Asking for the next page or trying a broader term reuses the data already obtained.

The existing budget limits external lookup attempts, while failed and empty fetches are remembered to avoid repeated attempts. Stable ranking uses the original candidate order to break equal scores, keeping page boundaries predictable within the snapshot.

The prompt now explicitly requires narrowing and paging where applicable before falling back to manual work. It also asks the final explanation to identify the Autodata category and search attempted when a potentially matchable job remains manual.

## A text match still needs a stock and fitment check

Parts are searched by article or item reference first when an identifier is available. If that fails, or no identifier exists, the prompt requests progressively broader description searches: distinctive terms first, then fewer terms retaining the component, then a useful workshop synonym.

The current stock tool still uses literal field and word matching. The new canonical synonym engine applies to the Autodata category and operation tools; prompt-guided synonym searches help the stock path separately.

Candidates must have enough remaining availability at the VIE location. Existing estimate demand and included draft demand count against that availability, and quantities must be interpreted in the correct units. Package parts keep their configured identity rather than being silently substituted.

A similar description is only a candidate. Fitment still needs evidence, and creating an estimate does not reserve the inventory. Availability is checked again when the advisor keeps the proposal.

## I needed to see what the tools actually did

A final sentence such as “Autodata lookup failed” did not give me enough information to debug the run. I added persistent tool-call history with the run, tool name, arguments, success flag, result summary, timestamp and metered-call information.

I can then check whether the failure came from vehicle resolution, entitlements, an empty search, invalid arguments or an exhausted budget before deciding what to change.

The newest search telemetry records counts, paging and cache information without VIN, registration or customer data. Detailed tool history has a different purpose and can contain inspection or provider-error content, so it should not be mistaken for anonymous telemetry.

Optional web search is also bounded. The branch exposes an Anthropic server-side search tool with a use limit and domain allowlist. The prompt treats web material as supporting research and prohibits sending vehicle identifiers to unrestricted web search. A public search result cannot establish a fitted part, an applicable labour time or an inspected fault. Dedicated licensed catalogue integrations remain a separate capability.

## I kept the final write in a separate, validated transaction

Generated proposals and customer wording remain temporary until the advisor selects **Keep it**. The apply code rechecks document status, finding ownership, source fingerprints, package applicability and stock requirements, then writes accepted groups in one transaction.

If a finding has changed or stock is insufficient, the transaction fails instead of leaving a partially applied estimate. Keep performs no model calls. Customer authorization and subsequent workshop processing remain with the advisor.

Discard leaves the estimate unchanged, although external lookup usage and integration-cache writes made during generation have already occurred.

Prompt configuration follows the same separation of responsibilities. Administrators can edit a company-specific UTF-8 prompt using the existing multiline editor, or restore the packaged default. Credentials are encrypted in company-scoped isolated storage. Prompt customization survives upgrades, which also means existing custom prompts need deliberate review when new tool guidance is introduced.

## The branch history records the evolution

| Branch or stage | Recorded state | Significance |
| --- | --- | --- |
| `backup/vie-estimate-builder-cloud-base` | `7d4e0c619` | Preserves the first implementation, originally built on the Cloud-Version baseline. |
| Master-based feature | Base `cf5e60f3e`; feature `9fa879e8e` | Re-established the implementation on the intended master baseline. |
| Schema upgrade | `2610ac9a9` | Raised the app version to `28.3.0.5` after a new setup field existed in source but was missing from the installed runtime schema. |
| `feature/vie-estimate-builder-copilot` | `8f24aeaf0` | Retains the Azure OpenAI route and the diagnostics, vehicle-history and lookup improvements. |
| Anthropic branch preparation | `4ac2ff4fa` | Captures the diagnostics and lookup improvements on the path to the provider change. Its tree differs from the Copilot branch tip only by the bundled prototype ZIP. |
| `feature/vie-estimate-builder-anthropic` | `669f2c48f` | Adds the direct Messages API client, adapted tool loop, connection setup and Estimate Builder description generation. |
| Current working tree | Uncommitted; manifest `28.3.0.7` | Adds the Autodata synonym setup, normalized ranking, complete category results, cached paging, prompt updates and focused tests. |

I also hit a normal Business Central deployment issue: the new setup field compiled, but the installed schema did not contain it. I increased the app version for the upgrade. The build, installed schema and runtime behaviour all need checking when delivering new fields.

The source includes tests for proposal validation and the provider contract, plus new cases for synonym matching, action and position mismatches, categories beyond the old limit, paging without gaps or duplicates, cache isolation and repeatable seeding. Earlier development recorded successful app and test compilation. Live provider, UI and end-to-end acceptance checks remain separate work; this post does not claim the latest changes have passed them.

For me, the useful engineering work is being able to trace a proposal all the way back: the inspection finding, the data returned by the tools, the matching operation, the selected part, and the checks performed before insertion. When a brake job becomes a manual group, I want to see why. When the agent selects a part, I want to know what supported that selection.

That is the standard I am working towards with this feature: a service advisor can review the proposed estimate, and a developer can explain how the system produced it. The remaining runtime checks need to confirm that the complete workflow behaves that way with real workshop data.
