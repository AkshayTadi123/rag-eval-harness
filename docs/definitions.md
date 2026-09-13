# Definitions

A glossary of the information retrieval terms used throughout this repository. Terms are
grouped by what they describe rather than alphabetically, because most of them only make
sense in relation to each other.

---

## 1. The data model

Retrieval evaluation was standardized by the TREC conferences in the early 1990s, and
essentially every IR experiment is built from the same four objects.

### Corpus
The collection of documents being searched. A mapping `doc_id -> document`. At small
scale (thousands to tens of thousands of documents) a corpus fits entirely in memory, and
brute-force similarity over a numpy matrix is a complete solution.

### Document
One retrievable unit of text. In BEIR datasets a document usually has a `title` and a
`text` field; whether you concatenate them before indexing is a real decision that
changes your scores, so it must be recorded rather than assumed.

### doc_id
The stable string identifier for a document. **Always a string, never an int.** Type
drift between corpus IDs and qrels IDs (`"1388"` vs `1388`) is one of the most common
silent failures in an eval harness — see *ID mismatch* below.

### Query
The question being asked, as raw text, with its own `query_id`. Query style varies by
dataset: SciFact queries are scientific claims, others are natural questions.

### Qrels — *query relevance judgments*
The ground truth: `query_id -> {doc_id: relevance_grade}`. A human annotator looked at a
document in the context of a query and graded it. This is the only source of truth about
correctness in an evaluation, so parsing it correctly matters more than anything
downstream of it.

Two properties matter enormously:

- **Qrels are incomplete.** Nobody judges every document against every query. Only a
  handful per query are ever looked at, and every unjudged document is *assumed
  non-relevant* by the metrics. So a retriever that surfaces a genuinely useful document
  no annotator happened to see is scored as if it failed. This is a known bias in the
  field, not a bug — but it caps how much meaning a small score difference can carry.
- **Grades can be graded, not binary.** SciFact is effectively binary (relevant / not).
  NFCorpus uses grades like 1 and 2 to mean "relevant" and "highly relevant". Metrics
  differ in how they convert grades into numbers.

### Relevance grade
The integer in the qrels. `0` (or absent) = not relevant. `1` = relevant. `2+` = more
relevant, in datasets that make that distinction.

### Run
A system's output: `query_id -> [(doc_id, score), ...]`, sorted best-first. This is the
object that gets scored.

### top-k
How many documents the retriever is asked to return. `k` is a parameter of the
measurement, not of the retriever: the same system evaluated at `k=10` and `k=1000`
answers two different questions — "is the answer at the top?" versus "did I find it at
all?"

### Rank vs. score
**Score** is the retriever's own number (a BM25 score, a cosine similarity) and its scale
is arbitrary — BM25 scores and cosine similarities are not comparable quantities.
**Rank** is the position in the sorted list, starting at 1. Metrics care almost entirely
about rank, which is why rank-based fusion (see *RRF*) is safer than combining raw scores.

---

## 2. The metrics

`ir_measures` and `pytrec_eval` both wrap `trec_eval` — the same C program used to score
published papers — which is why results computed through them are directly comparable to
the literature.

### Recall@k
Of all documents that *are* relevant for this query, what fraction appear in the top k?

```
Recall@k = |relevant ∩ top-k| / |relevant|
```

Answers "did I find the material at all." This is the right metric for a **first-stage**
retriever whose output will be reranked later: if the answer isn't in the candidate set,
no reranker can rescue it.

### Precision@k
Of the k documents returned, what fraction are relevant? The mirror of recall. Less
informative when ordering matters, since it ignores position within the k and is heavily
influenced by how many relevant documents happen to exist for that query.

### MRR — Mean Reciprocal Rank
`1 / (rank of the first relevant document)`, averaged over queries. First relevant hit at
rank 1 → `1.0`; rank 4 → `0.25`; nothing found → `0`. Cares *only* about the single best
hit, which makes it the right metric when a user will realistically read one result, and
the wrong metric when they need several.

### NDCG@k — Normalized Discounted Cumulative Gain
The field's default. Three ideas stacked:

1. **Gain** — each document contributes according to its relevance grade, so a grade-2
   document is worth more than a grade-1.
2. **Discounted** — each document's gain is divided by `log2(rank + 1)`, so a hit at
   rank 1 counts far more than the same hit at rank 10. This is what makes it
   position-aware.
3. **Normalized** — divide by the DCG of the *perfect* ranking for that query (the
   **IDCG**, ideal DCG). So `1.0` always means "unimprovable", and scores are comparable
   across queries that have different numbers of relevant documents.

**Worked example.** One query, two relevant documents (both grade 1). The system returns
10 documents, with the relevant ones at ranks 2 and 5.

```
DCG  = 1/log2(2+1) + 1/log2(5+1)   = 0.6309 + 0.3869 = 1.0178
IDCG = 1/log2(1+1) + 1/log2(2+1)   = 1.0000 + 0.6309 = 1.6309   (perfect: ranks 1 and 2)

NDCG@10 = 1.0178 / 1.6309 = 0.624
```

For that same ranking, `Recall@10 = 2/2 = 1.0` and `MRR = 1/2 = 0.5`. Three metrics,
three different verdicts on one identical result — which is why reporting several beats
picking a favourite.

### Why NDCG is not worth reimplementing
Implementations legitimately differ on (a) the **gain function** — linear `grade` vs.
exponential `2^grade − 1`; (b) **tie handling** when documents share a score; and (c) what
to do with **documents in the run that appear in no qrels**. A hand-rolled version
produces plausible numbers that match nothing published, which removes your ability to
validate against known results.

Note that for *binary* relevance the two gain functions coincide (`2^1 − 1 = 1`), so on a
dataset like SciFact the choice is invisible — and on graded data like NFCorpus it is not.
A discrepancy that hides on one dataset and appears on the next is the expensive kind.

### Per-query scores
Every metric is computed **per query first**, then averaged. The per-query vector is the
valuable output: it is what significance testing operates on, and it is what lets failure
modes be read directly ("dense retrieval wins on these 12 queries, and here is what they
have in common"). A mean alone throws that away.

---

## 3. Failure modes of an evaluation harness

### Relevance leak
Any code path that lets a retriever see the qrels, directly or indirectly — for example
building candidate lists only from judged documents. Symptom: *everything* scores
suspiciously well and systems become indistinguishable. The worst failure mode, because it
moves every number in the same direction, so nothing looks broken.

The structural defence is to pass `retrieve()` the query **text** and never the
`query_id`. With no identifier in hand, a retriever cannot look anything up: the leak is
made unrepresentable by the type signature rather than forbidden by a comment.

### ID mismatch
Corpus `doc_id`s and qrels `doc_id`s fail to match — int vs. string, stray whitespace, a
differing prefix. Symptom: every query scores `0.0`, and the retriever gets debugged for a
day when the retriever was always fine.

### Inverted ranking
Sorting ascending by score, so the best document lands last. Symptom: scores near zero but
not exactly zero.

### Random baseline
A retriever that ignores the query and returns a seeded shuffle. It is a test **with a
known answer**: it cannot possibly be good, so if it looks good there is a leak.

The subtlety is that "near zero" is a weak assertion. On a 5,183-document corpus, a random
`NDCG@10` is ≈ `0.000` — but an *ID mismatch also* produces `0.000`, so the test catches
leaks and misses mismatches. A deep-`k` recall fixes this by predicting a specific
non-trivial value: random `Recall@1000` over 5,183 documents should land near
`1000 / 5183 ≈ 0.19`, and only correctly wired plumbing produces that.

### Oracle retriever
A deliberately cheating retriever that ranks the known-relevant documents first, scoring
`NDCG@10 ≈ 1.0`. It proves the metric is *capable* of emitting a high number, which a
random baseline cannot tell you. It is the one component permitted to touch qrels, and it
belongs in a test rather than alongside real retrievers.

Random and oracle together give a **two-sided** check: random pins the floor, oracle pins
the ceiling.

---

## 4. Harness structure

### Run file / TREC run format
A run persists to disk as whitespace-delimited text, one line per retrieved document:

```
query_id  Q0  doc_id  rank  score  tag
```

(`Q0` is a vestigial field from 1990s TREC; it is always the literal string `Q0`.)

### Separating run generation from scoring
Retrieval writes a run file; scoring reads it. Keeping these as distinct steps buys three
things:

- Re-scoring with a new metric costs no retrieval time — no re-indexing, no re-embedding.
- Run files are diffable, so "did my change alter the ranking?" becomes `diff`.
- Results can be cross-checked against the standalone `trec_eval` binary, independently of
  any Python.

### Retriever interface
A single contract shared by every strategy, so adding one costs nothing:

```python
def retrieve(self, query: str, k: int) -> list[tuple[str, float]]
```

Building an index (tokenizing, embedding) is a separate, cached, one-time step; querying
is then cheap.

---

## 5. Retrieval strategies

### BM25
The standard lexical scoring function: term frequency, damped so repetition has
diminishing returns, weighted by how rare each term is across the corpus, and normalized
by document length. Matches literal words, so it is strong on exact terminology and blind
to paraphrase.

### Lexical vs. dense retrieval
Lexical retrieval matches words; dense retrieval matches positions in an embedding space,
so it can connect a query to a document that shares no vocabulary with it. They fail in
different ways, which is the reason to compare them rather than assume one wins.

### Bi-encoder
Encodes queries and documents *separately*, so document vectors can be precomputed once
and a query becomes a single vector comparison against the whole corpus. Fast, and less
accurate than a cross-encoder.

### Cross-encoder
Encodes a query and a document *together* in one forward pass, producing a direct
relevance score. Substantially more accurate, and far too slow to run over an entire
corpus — hence its use as a reranker over a shortlist of candidates.

### Reranking
Taking the top candidates from a cheap first-stage retriever and reordering them with a
more expensive, more accurate model. Recall of the first stage sets the ceiling: the
reranker can only reorder what it is given.

### RRF — Reciprocal Rank Fusion
Combines several rankings by summing `1 / (k + rank)` for each document across systems,
with `k ≈ 60` conventionally. It uses ranks rather than scores precisely because BM25 and
cosine scores are on incomparable scales, which makes it robust without any score
normalization or tuning.

### Learning to rank
Training a model (often gradient-boosted trees, e.g. LambdaMART) to combine retrieval
signals — BM25 score, dense similarity, cross-encoder score, document length, term
overlap — into a single ranking, instead of fusing them with a fixed rule like RRF.

### Chunking
Splitting long documents into smaller retrievable passages. Chunk size trades retrieval
precision against context completeness, and it is the parameter most often tuned by
intuition rather than measurement.

---

## 6. Statistical treatment

### Paired significance test
Compares two systems on the *same* queries using their per-query score vectors — a paired
t-test or a paired bootstrap. Pairing matters because query difficulty varies far more
than systems do, and comparing unpaired means buries the signal in that variance. With a
few hundred queries, a 1–2 point NDCG difference is frequently noise.

### Bootstrap confidence interval
Resample the set of queries many times with replacement, recompute the mean each time, and
report the spread. Gives a range rather than a single point estimate, and makes no
assumption that the per-query scores are normally distributed.
