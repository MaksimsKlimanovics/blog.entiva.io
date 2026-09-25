# Estimate Builder article: editorial source notes

Post: [_posts/2026-09-25-vie-estimate-builder-foundry-to-anthropic.md](../../_posts/2026-09-25-vie-estimate-builder-foundry-to-anthropic.md)

These notes are excluded from the Jekyll site by the existing docs/ exclusion.

Original draft: `D:/Garage Hive CodeBase/Garage_Hive_Base_App/docs/blog/vie-estimate-builder-foundry-to-anthropic.md`. Implementation paths below are relative to that Garage Hive repository.

Prepared from the local branch refs and working tree on 2026-09-25, the earlier Estimate Builder implementation recap, and local chat records including thread 01a0d3ac-d033-7183-bd4b-928164e4d637 and its earlier conversation material.

Primary implementation sources:
- app/src/Copilot/Estimate Builder/README.md
- docs/specs/2026-09-24-vie-estimate-builder-anthropic.md
- app/src/Copilot/Estimate Builder/GHVEstimateBuilderCopilot.Codeunit.al
- app/src/Copilot/Estimate Builder/GHVEBClaudeClient.Codeunit.al
- app/src/Copilot/Estimate Builder/GHVEBApplyProposal.Codeunit.al
- app/src/Copilot/Estimate Builder/GHVEBToolHistPackages.Codeunit.al
- app/src/Copilot/Estimate Builder/GHVEBToolCallHistory.Table.al
- app/src/Copilot/Estimate Builder/GHVEBADSearch.Codeunit.al
- app/src/Copilot/Estimate Builder/GHVEBADSearchCache.Codeunit.al
- app/src/Copilot/Estimate Builder/GHVEBSearchSynonyms.Codeunit.al
- app/src/Copilot/Estimate Builder/GHVEBToolADCategories.Codeunit.al
- app/src/Copilot/Estimate Builder/GHVEBToolADRepairTimes.Codeunit.al
- app/src/Autodata/API/ADAPILookupMgt.Codeunit.al
- app/Resources/GarageHiveEstimateBuilderPrompt.txt
- test/Test/EstimateBuilderTest.Codeunit.al
- test/Test/GHVEBADSearchTest.Codeunit.al

Publication boundaries: No comparative model-quality benchmark was found. The recorded Foundry errors establish observed behavior for the tested paths, not universal platform limitations. The GPT-5 compatibility note is scoped to System Application 28.3.52162.52222 and the observed custom-extension request; it does not claim every GPT-5 variant, later model, core version or built-in agent is unsupported. Microsoft documentation was checked when adding that note. The article deliberately omits deployment resource names, user/session identifiers, specific model defaults and credentials. No build, deployment or live runtime test was performed while drafting this article. Local remote-tracking refs were inspected; upstream refs were not refreshed. No repository code was changed by this documentation task.
