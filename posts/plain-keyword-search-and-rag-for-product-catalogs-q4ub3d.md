# Plain Keyword Search and RAG for Product Catalogs Explained

Index cost changes this decision. For a customer-support team searching a product catalog, ship plain keyword retrieval first, then add reranking only when relevance tests show a specific vocabulary gap. Add RAG after the workflow actually needs a generated answer grounded in retrieved records. **RAG is an answer architecture, not a synonym for better search.**

TL;DR: the simpler Node.js release is a small lexical index over approved catalog fields, with deterministic filters and observable ranking. It has fewer moving parts, keeps exact identifiers strong, and avoids building a second representation of every item before the team knows that representation helps.

## Should a product catalog use RAG or plain keyword search?

Before: a support agent enters `acme red charger 65w`. The service normalizes the query, applies availability or region filters, searches names, SKUs, aliases, and descriptions, then returns ranked product records. Every result can point back to an indexed field. This is retrieval.

After: retrieval still happens, but another stage may rerank candidates by relevance. A RAG flow goes farther: it supplies retrieved records as context to a generator, which produces an answer. The original RAG paper describes combining parametric model memory with non-parametric memory retrieved from an external corpus for knowledge-intensive tasks [1]. That extra generation step can be useful when an agent needs a synthesized response, but it does not remove the need for retrieval, filtering, or source records.

Picture the pipeline in words: query -> normalize -> filter -> retrieve -> optionally rerank -> optionally generate -> cite the catalog record. Each arrow adds latency, failure handling, logs, and a contract to test. Shorter wins until evidence says otherwise.

The choice has real limitations and trade-offs:

| Approach | Good fit | Poor fit | Extra operating surface |
| --- | --- | --- | --- |
| Plain keyword search | SKUs, known names, filters, exact terms | Unmodeled synonyms and broad intent | Lexical index and refresh jobs |
| Retrieval plus reranking | Relevant candidates exist but arrive in the wrong order | First-stage retrieval misses the item | Reranker, evaluation, and latency budget |
| RAG | A support reply must synthesize approved records | The caller only needs ranked product records | Retrieval, generation, grounding checks, and citations |

Plain search is the wrong choice when judged queries are dominated by conceptual language that aliases cannot cover. RAG is the wrong choice when the application only needs a product ID and ranked records; generation adds another failure boundary without solving missing candidates. A reranker sits between them, but its limitation is blunt: it can only reorder what it receives.

The smallest useful implementation can still be tiny.

Keep the first contract boring. The example below searches an in-memory catalog so the ranking behavior is visible; the same interface can sit in front of a durable lexical index later. Exact SKU matches lead, then token overlap breaks ties. It also records enough data to explain a bad result without logging generated prose or hiding the score.

```ts
type Product = {
  id: string;
  sku: string;
  name: string;
  aliases: string[];
  active: boolean;
};

type SearchHit = Product & { score: number };

const normalize = (value: string): string[] =>
  value.toLowerCase().match(/[a-z0-9]+/g) ?? [];

export function searchCatalog(
  query: string,
  products: Product[],
  limit = 10,
): SearchHit[] {
  const queryTokens = new Set(normalize(query));
  const normalizedQuery = [...queryTokens].join(" ");

  return products
    .filter((product) => product.active)
    .map((product) => {
      const sku = product.sku.toLowerCase();
      const fields = [product.name, product.sku, ...product.aliases];
      const fieldTokens = new Set(fields.flatMap(normalize));
      const overlap = [...queryTokens].filter((token) => fieldTokens.has(token)).length;
      const exactSkuBoost = normalizedQuery === sku ? 100 : 0;

      return { ...product, score: exactSkuBoost + overlap };
    })
    .filter((product) => product.score > 0)
    .sort((a, b) => b.score - a.score || a.id.localeCompare(b.id))
    .slice(0, limit);
}
```

This is intentionally a baseline, not a production indexing engine. Once the catalog no longer fits the process memory, move the same fields and scoring intent into a persistent index. Preserve the contract: filters happen before the final ranking, stable ties stay stable, and the response contains record IDs. The concrete pitfall is token normalization: punctuation in an SKU can split a value that should remain exact, so exact identifiers need their own normalized field and tests rather than blind reuse of the description tokenizer. The `100` boost above only makes that ordering rule visible; tune production scoring against judgments instead of treating the number as universal.

Test it with judged queries from the support queue. A compact set should include exact SKUs, product names, aliases, misspellings, discontinued items, and queries with no valid result. Track recall at a fixed candidate count, the position of the first relevant result, empty-result rate, and p95 latency. Also log index version, query class, candidate count, and selected IDs. Those dimensions turn “search feels worse” into a comparison the team can reproduce.

## Where does the simple approach fail?

Lexical retrieval struggles when customers and catalog editors use different words. “Laptop power brick” may need to find an item described only as an “AC adapter.” Curated aliases can close common gaps cheaply. As the vocabulary broadens, reranking a bounded lexical candidate set can capture semantic relevance without paying to semantically index every field and every inactive record.

Do not send the entire catalog into a reranker. Retrieve perhaps a configured candidate count, measure whether the relevant item survives that first stage, and rerank only those candidates. The exact count belongs in an evaluation, not in a universal rule. A reranker cannot rescue a relevant product that retrieval never returned.

Freshness is another trap. Product availability, region, and lifecycle state are structured facts. Keep them as filters sourced from the catalog system; do not ask a generator to infer them from descriptive text. Reindex when searchable fields change, record the index version, and alert on indexing lag. If an update fails, the service should retain the last complete index rather than expose a half-built one.

No-result behavior matters too. Return an explicit empty result and enough diagnostic context for the calling application to choose its next step. Fabricating a plausible product is not retrieval.

Stop there.

No. Reranking changes the order of retrieved records. RAG adds generation over retrieved context. They solve different parts of the support workflow and can be deployed separately.

That distinction keeps the decision testable. If judged search queries show that lexical candidates contain the right product but rank it too low, evaluate a reranker. Compare relevance gain against added latency, operational load, and the cost of maintaining any additional representation. If the right product is absent from the candidate set, fix retrieval, fields, aliases, filters, or index freshness first.

Add generation only when the output needs synthesis, such as drafting a support response from several approved records. Then require record citations, define behavior when retrieval is empty, and evaluate whether each answer is supported by its context. Retrieval quality remains a dependency. So does catalog freshness.

## What about index cost at catalog scale?

Count representations, not feature labels. A lexical design stores searchable fields and its index structures. A semantic retrieval design also needs derived representations for the chosen records or fields, plus a refresh path when those inputs change. A hybrid system maintains both. The cost question is therefore driven by catalog size, indexed field volume, update frequency, retention of old versions, and the number of environments.

Use a before-and-after capacity sheet with five rows: source records, lexical index, derived representations, temporary rebuild space, and replicas. Measure those values on a representative catalog snapshot. Estimates based only on item count miss long descriptions, variant explosions, and rebuild overlap.

**Choose the least elaborate pipeline that passes the relevance target.** For most first releases, that means lexical search with explicit fields, aliases, filters, and a judged query set. Introduce reranking when the measurements isolate a ranking problem. Introduce RAG when users need grounded synthesis rather than a list of products. That sequence keeps every added index and runtime stage tied to an observed failure.

## Sources and References

1. Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
