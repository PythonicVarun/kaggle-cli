---
name: research-latest-benchmarks
description: >
  Researches the latest model launches from OpenAI, Anthropic, and Google to
  identify which benchmarks each lab cited on their official model cards and
  press releases. Produces a cross-model benchmark comparison table showing
  which evaluations each lab considers important. Uses a multi-subagent pipeline
  with built-in validation. Keywords: benchmarks, model card, model launch,
  frontier models, evaluation, comparison, press release, GPT, Claude, Gemini.
metadata:
  author: nicholaskang
  version: "1.0"
---

# Research latest benchmarks

## Overview

This skill orchestrates a multi-phase research pipeline to answer the question: **"What benchmarks are frontier AI labs citing on their latest model launches, and what does that tell us about what the industry considers important?"**

The pipeline discovers the latest models from OpenAI, Anthropic, and Google, researches each model's official press release and model card in parallel, synthesizes the findings into a unified comparison table, and then runs a validation pass to catch any missing benchmarks or errors.

The output is a markdown document with:
- A cross-model benchmark comparison table (benchmark × model with check/cross)
- Benchmark descriptions and categories
- Reference links to the official announcements
- Key findings about emerging industry evaluation standards


## When to use this skill

Activate this skill when the user asks to:
- Find out what benchmarks the latest models are citing
- Compare benchmark citations across frontier model launches
- Understand which evaluations the industry considers important
- Update an existing benchmark comparison table with new model launches
- Research a specific model launch for its benchmark citations


## Phase 1: Discover latest models

Before researching benchmarks, you MUST first identify the latest models from each lab. Do NOT assume model names or versions — they change with every launch cycle.

Spin up a single research subagent to discover the current latest models.

### Subagent 0: Model discovery

Role: "Latest Model Discovery"
Type: research

Prompt template:
```
Search the web for the LATEST model releases from each of these three AI labs. I need the most recent generally available (GA) model from each — not previews or limited-access variants unless they are the only option.

1. OpenAI — what is the latest GPT model? (e.g., GPT-5.5, GPT-6, etc.)
2. Anthropic — what is the latest Claude model? (e.g., Claude Opus 4.7, Claude 5, etc.)
3. Google DeepMind — what is the latest Gemini model? (e.g., Gemini 3.5 Flash, Gemini 4, etc.)

For each model, report:
- Full model name and version
- Release date
- URL of the official announcement or press release (look for openai.com, anthropic.com/news, blog.google, or deepmind.google)
- One-line summary of positioning (e.g., "optimized for agentic workflows and coding")

IMPORTANT: Only report models that have been publicly released and have official announcements. Do not include rumors, leaked names, or unconfirmed models.
```

After receiving the model discovery results, confirm the three model names with the user before proceeding. Present them as:
- OpenAI: [model name] (released [date])
- Anthropic: [model name] (released [date])
- Google: [model name] (released [date])

Ask: "These are the latest models I found. Should I proceed with researching their benchmark citations, or do you want to adjust the list?"

CRITICAL: Wait for user confirmation before proceeding to Phase 2. The user may want to substitute a model, add a fourth lab, or focus on only one model.


## Phase 2: Benchmark research — deploy 3 research subagents in parallel

After the user confirms the model list, spin up exactly 3 research subagents using `invoke_subagent`. All three run in parallel.

### Subagent 1: OpenAI benchmark researcher

Role: "[Model Name] Benchmark Researcher"
Type: research

Prompt template:
```
Search the web for the official press release, model card, system card, and technical announcement for [FULL MODEL NAME] from OpenAI (released [DATE]).

Your task is to identify EVERY benchmark cited in the official announcement or model card. For each benchmark found, provide:
1. Benchmark name (exact name as cited)
2. Version number if specified
3. Score achieved (if reported)
4. A brief description of what the benchmark measures (1-2 sentences)

Where to look:
- openai.com/index/ — official blog posts and announcements
- openai.com system cards
- Third-party coverage from reputable sources (techcrunch.com, theverge.com, arstechnica.com) that reference the official model card

IMPORTANT:
- Only include benchmarks that were explicitly cited by OpenAI in their official materials
- Do not include benchmarks from third-party evaluations unless OpenAI itself referenced them
- Note the exact URL of the official announcement you found
```

### Subagent 2: Anthropic benchmark researcher

Role: "[Model Name] Benchmark Researcher"
Type: research

Prompt template:
```
Search the web for the official press release, model card, and technical announcement for [FULL MODEL NAME] from Anthropic (released [DATE]).

Your task is to identify EVERY benchmark cited in the official announcement or model card. For each benchmark found, provide:
1. Benchmark name (exact name as cited)
2. Version number if specified
3. Score achieved (if reported)
4. A brief description of what the benchmark measures (1-2 sentences)

Where to look:
- anthropic.com/news — official announcements
- anthropic.com/research — technical reports
- claude.com/docs — model documentation
- Third-party coverage from reputable sources that reference the official model card

IMPORTANT:
- Only include benchmarks that were explicitly cited by Anthropic in their official materials
- Do not include benchmarks from third-party evaluations unless Anthropic itself referenced them
- Note the exact URL of the official announcement you found
```

### Subagent 3: Google benchmark researcher

Role: "[Model Name] Benchmark Researcher"
Type: research

Prompt template:
```
Search the web for the official press release, model card, and technical announcement for [FULL MODEL NAME] from Google DeepMind (released [DATE]).

Your task is to identify EVERY benchmark cited in the official announcement or model card. For each benchmark found, provide:
1. Benchmark name (exact name as cited)
2. Version number if specified
3. Score achieved (if reported)
4. A brief description of what the benchmark measures (1-2 sentences)

Where to look:
- blog.google/technology/ — official Google blog
- deepmind.google — DeepMind announcements and model cards
- ai.google.dev — developer documentation
- Third-party coverage from reputable sources that reference the official model card

IMPORTANT:
- Only include benchmarks that were explicitly cited by Google in their official materials
- Do not include benchmarks from third-party evaluations unless Google itself referenced them
- Note the exact URL of the official announcement you found
```


## Phase 3: Synthesis — build the comparison table

After ALL three research subagents complete, synthesize the findings into a single unified document. Follow these steps exactly:

### Step 1: Build the master benchmark list

Create a deduplicated list of every benchmark cited by any of the three models. For each benchmark, note:
- Canonical name (normalize across labs — e.g., if one lab says "GPQA" and another says "GPQA Diamond," note the difference)
- Version differences (e.g., Terminal-Bench 2.0 vs 2.1)
- Brief description of what it measures

### Step 2: Build the comparison table

Create a markdown table with the following columns:
- Benchmark
- Description
- [OpenAI Model Name] (✓ or x, with version notes in parentheses)
- [Anthropic Model Name] (✓ or x, with version notes in parentheses)
- [Google Model Name] (✓ or x, with version notes in parentheses)

Sort the table with benchmarks cited by ALL three labs at the top, then by two labs, then by one lab only.

### Step 3: Write the reference links section

Include direct links to each model's official announcement or press release, sourced from the research subagents.

### Step 4: Write key findings

Analyze the table and write 3-5 key findings about emerging industry evaluation standards. Focus on:
- Which benchmarks have consensus across labs (cited by all 3)
- Which benchmarks are unique to one lab (potentially proprietary or niche)
- What categories of evaluation dominate (agentic, coding, reasoning, multimodal, etc.)
- Which traditional benchmarks are no longer cited (e.g., MMLU, GSM8K)
- Any notable shifts from the previous generation of model launches

### Step 5: Write detailed benchmark profiles

For each benchmark in the table, write a brief profile with:
- Category (e.g., "Agentic software engineering," "Expert-level reasoning")
- What it measures (1-2 sentences)
- Version notes if applicable


## Phase 4: Validation — deploy verification subagent

After building the comparison table, spin up a verification subagent to independently cross-check the findings.

### Subagent 4: Benchmark citation verifier

Role: "Benchmark Citation Verifier"
Type: research

Prompt template:
```
I need to verify the benchmark citations for three recent model launches. Please search the web for the actual official press releases and model cards for each of the following models, and confirm which benchmarks each one cited:

1. [OPENAI MODEL NAME] (launched [DATE]) — I have the following benchmarks attributed to it: [LIST]. Please confirm or deny each one by checking the actual official announcement or system card.

2. [ANTHROPIC MODEL NAME] (launched [DATE]) — I have: [LIST]. Please confirm or deny each.

3. [GOOGLE MODEL NAME] (launched [DATE]) — I have: [LIST]. Please confirm or deny each.

Also check: are there any benchmarks cited by these models that I'm MISSING from the lists above?

Please search for the actual official announcements (openai.com, anthropic.com, blog.google, deepmind.google) to verify. Report back with a clear confirmation/denial for each benchmark per model, and flag any discrepancies or missing benchmarks.
```

After receiving the verification results:
1. Fix any errors in the table (wrong attributions, incorrect benchmark names)
2. Add any missing benchmarks that the verifier discovered
3. Note any benchmarks that could not be independently confirmed


## Phase 5: Output — present to user

Create the final document as a markdown file in the user's workspace. Use the filename pattern: `benchmark_comparison_latest_models.md`

If a previous version of the file exists, overwrite it with the updated data.

The document structure should be:

```
# Benchmark citation analysis of latest model launches ([Model 1], [Model 2], [Model 3])

Authors: Antigravity
Date: [current date]

---

## Executive summary
[2-3 sentences summarizing the key takeaway]

---

## Reference links
[Direct links to each model's official announcement]

---

## Cross-model benchmark comparison table
[The full markdown table]

---

## Key findings and emerging industry standards
[3-5 analytical findings]

---

## Detailed benchmark profiles
[One subsection per benchmark with category + description]
```

After creating the file, present the comparison table inline in your response so the user can see it immediately without opening the file.

IMPORTANT: The final document should follow the user's formatting preferences:
- Use standard markdown (this is a technical reference doc, not a Google Docs copy-paste)
- Sentence case for all headings
- No trailing periods on bullet points
- No question marks in headings


## Workflow summary

```
Phase 1: Discover latest models (1 subagent)
    ↓
User confirmation gate — confirm model list
    ↓
Phase 2: Benchmark research (3 subagents in parallel)
    ├── OpenAI Model Researcher
    ├── Anthropic Model Researcher
    └── Google Model Researcher
    ↓
Phase 3: Synthesis — build comparison table and analysis
    ↓
Phase 4: Validation (1 subagent)
    ↓
Phase 5: Output — present final document
```


## Subagent summary

| # | Role | Type | Phase | Purpose |
|---|------|------|-------|---------|
| 0 | Latest Model Discovery | research | 1 | Identify the current latest models from each lab |
| 1 | OpenAI Benchmark Researcher | research | 2 | Find all benchmarks cited in OpenAI's model card |
| 2 | Anthropic Benchmark Researcher | research | 2 | Find all benchmarks cited in Anthropic's model card |
| 3 | Google Benchmark Researcher | research | 2 | Find all benchmarks cited in Google's model card |
| 4 | Benchmark Citation Verifier | research | 4 | Cross-check all findings for accuracy and completeness |


## Important rules

1. ALWAYS discover the latest models first — never hardcode model names like "GPT-5.5" or "Opus 4.7" because they will be outdated on the next run
2. ALWAYS run the validation subagent — the initial research subagents may miss benchmarks or misattribute them
3. Sort the comparison table by consensus (all 3 labs → 2 labs → 1 lab) so the most important benchmarks appear first
4. Include version numbers in the table when benchmarks differ across labs (e.g., Terminal-Bench 2.0 vs 2.1)
5. Use ✓ and x in the table cells — not checkmarks/crosses or yes/no
6. Link to official press releases, not third-party coverage, in the reference links section
7. The user's formatting preferences apply to ALL output
8. If the user asks to add additional labs (e.g., Meta/Llama, xAI/Grok), extend the pipeline by adding additional research subagents in Phase 2 and additional columns to the table
9. If a benchmark is cited by a model but with a different version or variant name than other models, note the variant in parentheses in the table cell (e.g., "✓ (Standard)" or "✓ (v2)")
10. Always present the table inline in your response in addition to saving the file — the user should see the results immediately
