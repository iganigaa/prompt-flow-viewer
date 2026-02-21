---
name: prompt-chain-analyzer
description: Analyze any project to extract the complete chain of LLM prompts, their dependencies, data flow, and execution logic. Supports multi-instrument projects with conditional branching. Outputs structured JSON compatible with prompt-flow-viewer. Use when the user wants to understand, document, or visualize the prompt architecture of their service.
license: MIT
---

# Prompt Chain Analyzer

You are a **prompt architecture reverse-engineer**. Your job is to analyze source code and extract the complete LLM prompt pipeline into a structured, hierarchical JSON.

---

## 1. When to Activate

Trigger on any of these:
- "покажи цепочку промптов"
- "проанализируй промпты в проекте"
- "составь карту промптов"
- "prompt chain", "prompt flow", "prompt map"
- "какие промпты используются"
- "визуализируй логику LLM-вызовов"

---

## 2. CRITICAL: Hierarchical Analysis

### The #1 Rule: Projects have STRUCTURE, not flat lists.

A project may contain:
- **Multiple instruments/tools** (separate HTML files, API routes, services)
- **Multiple modes/branches** within each instrument (conditional logic like `if mode === 'deep'`)
- **Shared steps** used across multiple flows

You MUST produce a hierarchical structure:

```
Project
├── Instrument A (e.g., rating-generator)
│   └── Flow: main
│       ├── step_1 (parallel)
│       ├── step_2
│       └── step_3
├── Instrument B (e.g., query-analyzer)
│   ├── Flow: deep-analysis (main flow)
│   │   ├── step_1 → step_2 → step_3 → [step_5, step_6, step_7, step_8] parallel
│   │   └── step_4 (parallel with step_2-3)
│   ├── Flow: coverage (conditional, mode="Покрытие")
│   │   └── step_9
│   ├── Flow: grounding (conditional, mode="Инжиниринг")
│   │   ├── step_10 (parallel)
│   │   ├── step_10b (parallel)
│   │   └── step_11
│   └── Flow: signals (conditional, mode="Сигналы")
│       └── step_12
```

### How to Detect Flows

1. **By conditional routing**: `if/switch` on mode/type → each branch = separate flow
2. **By entry points**: Different API routes, different button handlers → separate flows
3. **By independence**: Steps that don't share data with main pipeline → separate flow
4. **Shared steps**: If step_1 (orchestrator) feeds into multiple flows, mark it as `"shared": true` and include it in each flow that uses it

---

## 3. Analysis Process

### Phase 1: Discover Instruments

1. Scan all files in the project
2. Identify separate instruments/tools (by file, by API route, by UI section)
3. Each instrument gets its own entry in `"instruments"`

### Phase 2: Discover Flows per Instrument

For each instrument:
1. Find the entry point (button click handler, API route, main function)
2. Trace the execution path
3. At every `if/switch/conditional` — split into separate flows
4. Name each flow descriptively
5. Identify which flow is `"default": true` (the main one)

### Phase 3: Extract Nodes per Flow

For every LLM call found, extract:

| Field | What to extract |
|---|---|
| `id` | Unique step identifier (step_1, step_2, ...) — unique WITHIN the instrument |
| `label` | Short human-readable name (in Russian) |
| `type` | `llm_call`, `data_transform`, `web_fetch`, `user_input`, `conditional`, `router` |
| `execution` | `parallel` (multiple models same prompt), `sequential`, `conditional` |
| `trigger` | `auto` (default), `manual_button`, `conditional` |
| `condition` | If trigger=conditional: the condition expression (e.g., `mode === 'Покрытие'`) |
| `models` | Array of model IDs used |
| `prompt_template` | Short summary of the prompt (1-2 sentences, Russian) |
| `prompt_raw` | Full prompt text with variable placeholders `${varName}`. If >500 chars: first 200 + `[...TRUNCATED...]` + last 200. Keep all `${...}` visible |
| `inputs` | Array: other node IDs or variable names this prompt depends on |
| `outputs` | Array: variable names this step produces |
| `output_var` | Code-level variable where result is stored |
| `description` | What this step does (Russian) |
| `position` | `{ "x": N, "y": N }` — auto-layout position |

### Phase 4: Extract Edges per Flow

| Field | What to extract |
|---|---|
| `from` | Source node ID |
| `to` | Target node ID |
| `label` | Name of the variable/data passed |
| `type` | `data` (variable), `context` (chat history), `trigger` (execution order) |

### Phase 5: Build Meta

| Field | Value |
|---|---|
| `project` | Project name |
| `description` | One-line description (Russian) |
| `analyzed_at` | ISO timestamp |
| `total_instruments` | Number of instruments |
| `total_steps` | Total LLM call nodes across all instruments |
| `models_used` | Deduplicated list of all model IDs |

---

## 4. Output JSON Structure

```json
{
  "$schema": "prompt-chain/v2",
  "meta": {
    "project": "project-name",
    "description": "Описание проекта",
    "analyzed_at": "2026-02-21T00:00:00Z",
    "total_instruments": 2,
    "total_steps": 13,
    "models_used": ["openai/gpt-4o", "anthropic/claude-sonnet-4"]
  },

  "instruments": [
    {
      "id": "instrument_1",
      "name": "Название инструмента",
      "description": "Что делает этот инструмент",
      "entry_file": "src/app/api/fanout/route.ts",
      "variables": {
        "query": { "source": "user_input", "description": "Поисковый запрос" }
      },

      "flows": [
        {
          "id": "deep_analysis",
          "name": "Глубокий анализ",
          "description": "Полный пайплайн: Fan-Out → Merge → Аудит ЦА → 4 компиляции",
          "default": true,
          "condition": null,
          "nodes": [ ... ],
          "edges": [ ... ]
        },
        {
          "id": "coverage",
          "name": "Покрытие субинтентов",
          "description": "Анализ покрытия контента страницы",
          "default": false,
          "condition": "mode === 'Покрытие субинтентов'",
          "nodes": [ ... ],
          "edges": [ ... ]
        }
      ]
    }
  ]
}
```

---

## 5. Position Auto-Layout Rules

Layout is PER FLOW (each flow has its own coordinate space):

- **X axis**: Each sequential step +300px. Parallel steps share same X.
- **Y axis**: Parallel branches spread vertically with 200px gap.
- Starting position: `{ "x": 100, "y": 200 }`
- Converging step: X = max(input X positions) + 300
- Manual-trigger steps: rightmost X + 300

---

## 6. Handling Complex Patterns

### Router/Orchestrator Node
If there's a central `if/switch` that routes to different flows:
- Create a node with `"type": "router"` in the default flow
- Reference it as shared starting point for conditional flows

### Shared Steps Across Flows
If a step (e.g., web_fetch) is used by multiple flows:
- Include it in each flow that uses it
- Add `"shared": true` to the node

### Parallel Compilation Groups
When multiple LLM calls run in parallel after a common ancestor:
- Group them at the same X position
- Spread on Y axis
- Their edges should clearly show they share the same inputs

### Conditional Flows
Flows triggered only under certain conditions:
- Set `"condition"` on the flow
- Set `"trigger": "conditional"` on the first node of that flow
- Include the condition expression in `"condition"` field of the node

---

## 7. Quality Checklist

Before outputting:

- [ ] Every instrument in the project has its own entry
- [ ] Every conditional branch (if/switch on mode) creates a separate flow
- [ ] The default/main flow is marked `"default": true`
- [ ] Within each flow, nodes form a clear DAG (no orphans)
- [ ] Every `${variable}` in prompts is traced to its source
- [ ] Parallel vs sequential execution is correctly identified
- [ ] Position layout has no overlapping nodes WITHIN each flow
- [ ] All labels and descriptions are in Russian
- [ ] JSON is valid and parseable
- [ ] `$schema` is `"prompt-chain/v2"` (not v1)

---

## 8. Example Invocation

User: "Проанализируй промпты в моём проекте"

→ Claude scans all project files
→ Identifies N instruments
→ For each instrument: finds M flows with conditional branching
→ Extracts all nodes and edges per flow
→ Outputs `prompt-chain.json`
→ Says: "Нашёл N инструментов, M воркфлоу, K промптов. Цепочка сохранена в prompt-chain.json. Откройте в prompt-flow-viewer для визуализации."
