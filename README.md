# Multi-agent system sandbox — TypeScript

A small, deliberately readable multi-agent system built to be measured rather than
demoed: a support desk for a fictional logistics company, wired as a supervisor over
three workers, with evals, tracing and cost accounting attached to every run.

Ported one-to-one from `~/agent-eval-sandbox` (Python) — same architecture, same
dataset, same metrics — so the two languages can be compared on the same task.

## Quick start

```bash
npm install --legacy-peer-deps          # see "Known rough edges"
npm run chat -- "How much does domestic delivery of an 8 kg parcel cost?"
npm run trace                           # readable view of the latest trace
npm run run                             # the full experiment: real Claude + LLM judges
npm run report -- <experimentId>
```

Phoenix must be up — it is shared with the Python version:
`cd ~/agent-eval-sandbox && make up`.

Every run costs money: `ANTHROPIC_API_KEY` is required and there is no free mode.

## Structure

```
src/
  agents/       the registry — and the only place the TOPOLOGY is written down
    supervisor/ index.ts (spec) · prompt.ts
    retriever/  + tools/  search-docs · bm25 · hybrid · fusion · corpus · langchain
    calc/       + tools/  calc.ts
    api/        + tools/  get-shipment · get-customer · data.ts
    index.ts    registry · registry.ts  integrity rules · types.ts
  runtime/      the only folder that knows about LangGraph
    index.ts    asTool + assembling the agent tree
    run.ts      facade: question → RunResult
    recorder.ts · spans.ts · tools.ts
  kb/           the knowledge base the retriever searches
  config.ts     models, prices, Phoenix endpoint
evals/          dataset.ts · evaluators.ts · runExperiment.ts · report.ts
                retrieval-truth.ts · retrievalBench.ts
chat.ts · trace.ts · datasets.ts · promote.ts
```

An agent is a system prompt plus a `canCall` list. The topology is the content of
those lists, not the shape of the code: turning the supervisor into a mesh means
adding names to `canCall`, and nothing in `src/runtime/` changes.

## Metrics

Deterministic, computed in code: `trajectory_match`, `tool_selection_f1`, `no_loops`,
`keyword_check`, `cost_usd`, `latency_s`, `steps_count`.

LLM-as-judge (`claude-opus-5`): `correctness_judge`, `groundedness_judge`, `safety_judge`.

The rule is simple: whatever can be checked in code, check in code. A model judge
costs money, drifts, and needs calibrating itself. Efficiency metrics (`cost_usd`,
`latency_s`, `steps_count`) sit next to the quality ones on purpose — a prompt
"improvement" that doubles the bill without changing an answer is invisible otherwise.

## Where TypeScript came out better

**Zod as a single source of truth.** In Python, a tool's schema, its argument
validation and its type are three separate places (and validation is absent
altogether). Here one object gives all three:

```ts
const Input = z.object({
  query: z.string().describe("Search query"),
});
// → JSON schema for the model  (toJsonSchema)
// → runtime validation         (Input.parse)
// → static type                (z.infer)
```

**Types are actually checked.** `npm run typecheck` catches what passes silently in
Python: `noUncheckedIndexedAccess` forced every indexed access to be handled
explicitly, and that found two places where the Python version would simply have
crashed at runtime.

## Where it took effort

**The calculator.** In Python, `ast.parse` gives you safe expression parsing out of
the box. JS has nothing of the sort, and `eval()` would run arbitrary code from the
model's reply — so `src/agents/calc/tools/calc.ts` carries a hand-written recursive
descent parser (~70 lines against 15 in Python).

**Reading eval results.** The Python client returns them from `get_experiment()`.
The JS client does not, so `report.ts` goes to the REST endpoint
`/v1/experiments/{id}/json` instead.

**Spans from callbacks.** LangChain callbacks arrive from different async contexts
and the parent context does not travel with them, so spans built inside the callbacks
came out orphaned. `recorder.ts` therefore only collects data, and `spans.ts` builds
the whole tree afterwards in one synchronous pass.

**Hybrid search.** The library's `EnsembleRetriever` implements the same weighted RRF
formula but merges documents by `pageContent` — and the vector store's text carries a
`passage:` prefix that BM25's does not, so the fusion silently never happens. See the
comment at the top of `src/agents/retriever/tools/hybrid.ts`.

## Articles

A three-part series built on this sandbox — topology, retrieval, evaluation — in English
and Ukrainian: [`docs/articles/`](docs/articles/README.md).

## Known rough edges

`@arizeai/phoenix-client@7.8.0` keeps a stale optional peer on
`@anthropic-ai/sdk@^0.35.0`. We do not use that peer (the judge calls Anthropic
directly), so we install with `--legacy-peer-deps`.

## Language

Code, prompts, knowledge base and dataset are all in English. Earlier revisions were
in Ukrainian; any measurement taken before the switch describes the Ukrainian-language
system and is not directly comparable to a run of this revision.
