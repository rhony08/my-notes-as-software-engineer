# Chunking Strategies for Better Retrieval

You built a RAG pipeline. You embedded the docs, wired up the vector search, and asked a question. The answer came back... kind of right, but sliced in the middle of a sentence, with the second half of the explanation missing.

The retrieval didn't fail because your embeddings were bad or your vector DB was slow. It failed because of how you cut the documents up before embedding them. **Chunking is the step nobody tunes, and it's the one that decides whether retrieval works at all.**

Here's the uncomfortable truth: a brilliant embedding model fed badly-chunked text loses to a mediocre model fed well-chunked text. Every time.

## Why Chunking Is the Whole Game

Embeddings compress meaning into a fixed vector. That compression has a budget. Cram a 3,000-word document into one vector and the model has to average together a dozen distinct ideas — the result is a "smeared" point that's vaguely near everything and precisely near nothing.

Split too small and you get the opposite problem: `"It returns a 404."` is a chunk with no subject, no context, no referent. Embed it and you get a vector that matches almost any sentence about HTTP status codes, and answers zero questions.

### What Bad Chunks Actually Cost You

- **Cut mid-sentence** → the retrieved context starts with `"and therefore the retry should..."` and the model has no idea what "therefore" refers to.
- **Cut mid-idea** → half the reasoning is in chunk 4, the other half in chunk 5. The retriever fetches one, the answer is wrong, confidently.
- **One giant chunk** → the relevant sentence is buried in 2,000 tokens of noise. The LLM may ignore it or get distracted.
- **Context lost on split** → a table's header row is in a different chunk than its rows.

The fix isn't a magic chunk size. It's cutting on the boundaries that carry meaning.

## The Chunking Spectrum

There's no single right answer, but there's a clear ladder from dumb to smart. Start simple; move up only when your evals say you need to.

| Strategy | How it splits | Good for | Watch out for |
|----------|--------------|----------|---------------|
| **Fixed size** | Every N characters/tokens | Uniform logs, quick prototypes | Cuts mid-sentence, mid-idea |
| **Recursive** | Splits on `\n\n`, then `\n`, then `. `, then hard cut | Most prose/docs — the sane default | Still ignores structure like tables |
| **Document-aware** | Uses Markdown headers, HTML tags, code fences | Structured docs, source code | Needs a parser per format |
| **Semantic** | Splits where sentence embeddings diverge | Transcripts, dense narrative | Slower, costlier, harder to debug |
| **Parent-child** | Index small, retrieve big | Precision retrieval, rich context | Extra storage, more plumbing |

### Fixed-Size: Fine Until It Isn't

The naive approach — `text[i:i+1000]` — is a legitimate starting point. It's fast, predictable, and dead simple to reason about. For a pile of uniform log lines or a quick prototype, it's genuinely enough.

Where it breaks: the moment your content has structure. A fixed window doesn't know that a `####` starts a new section, so it'll happily glue the end of one topic to the start of another and embed the mash-up.

```python
# ❌ Naive fixed split — severs sentences and ideas
def chunk(text, size=1000):
    return [text[i:i+size] for i in range(0, len(text), size)]

# The last chunk of every doc ends mid-word. The first chunk of the next
# starts with a dangling pronoun. Retrieval quality suffers silently.
```

### Recursive Splitting: The Default You Should Reach For

Recursive character splitting is what most frameworks (LangChain's `RecursiveCharacterTextSplitter`, LlamaIndex's node parsers) do by default, and for good reason. It tries the *biggest* boundary first and only falls back to smaller ones if a piece is still too large:

1. Split on double newlines (paragraphs)
2. If a piece is still too long, split on single newlines
3. Still too long? Split on sentence boundaries (`. `, `? `, `! `)
4. Last resort: hard cut at the character limit

This preserves the natural units of text — paragraphs stay whole when they fit, and only oversized ones get subdivided.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,        # target size in characters
    chunk_overlap=100,     # keep a little shared context between neighbors
    separators=["\n\n", "\n", ". ", " ", ""],  # biggest boundary first
)
chunks = splitter.split_text(document)
```

Notice the ordering of `separators`. That list *is* the strategy. Put `"\n\n"` first and you respect paragraphs; put `" "` first and you shred the doc into word soup.

### The Overlap Dial

`chunk_overlap` is the insurance policy against the split happening at a bad spot. By repeating the last ~100 characters into the next chunk, you make sure an idea that straddles the boundary appears *whole* in at least one chunk.

The trade-off is obvious once you look at your bill: 100 chars of overlap on 800-char chunks is ~12% more vectors, ~12% more storage, ~12% more embedding cost. Use enough to cover a sentence or two. Don't use 50% "just to be safe" — you'll double your index for marginal gain.

### Document-Aware Splitting: Use the Structure You Already Have

Your documents are already chunked — by the author. Markdown headers, HTML sections, and code fence boundaries are *exactly* the semantic units you're trying to recover. Throw them away and re-cut blindly, and you're discarding free signal.

```python
# ✅ Split on Markdown headers — the author already told you where ideas end
import re

def split_by_headers(md):
    # Keep the header with the text it introduces, so chunks are self-describing
    sections = re.split(r'\n(?=#{1,3} )', md)
    return [s.strip() for s in sections if s.strip()]

# Each chunk now starts with its own heading, e.g.
# "## Retry Policy\n\nRetries use exponential backoff starting at..."
```

The payoff is subtle but huge: the chunk begins with its own title, so the embedded vector carries the *topic heading* as well as the body. A query about "retry policy" now matches the chunk far more reliably.

The same logic applies to code (split on function/class boundaries, never mid-function) and HTML (split on `<section>`/`<article>` tags, not character counts).

### The Structure Trap: Tables and Lists

Here's one that bites everyone eventually. A Markdown table split across two chunks turns into meaningless rows:

```markdown
<!-- ❌ Split here, in the middle of a table -->
| Plan | Requests/min | Burst |
|------|-------------|-------|
<!-- chunk boundary -->
| Free | 60 | 10 |
| Pro  | 600 | 100 |
```

The second chunk is a table with no header row. The model can't tell which column is which. The fix: treat a whole table (or list, or code block) as an atomic unit — if it fits, never split inside it. Detect fences and pipe tables in your splitter and back off to the previous boundary.

### Semantic Chunking: When Structure Isn't Enough

Sometimes there's no markup to lean on — raw transcripts, scraped prose, chat logs. Semantic chunking embeds each sentence, then starts a new chunk wherever the similarity between consecutive sentences drops below a threshold. The idea: a chunk ends when the *topic* changes, not when a character counter ticks over.

It works, and it can measurably improve retrieval on messy text. But be honest about the cost: you're embedding the document once just to decide how to chunk it, then embedding it again for the index. It's slower, more expensive, and the threshold is a new hyperparameter you now have to tune. Reach for it only after simpler methods plateau.

### Parent-Child: Index Small, Retrieve Big

The best of both worlds, and worth knowing even if you don't use it on day one. You embed *small* chunks (sentences or single paragraphs) because they give precise, unambiguous matches. But you store a pointer to the *parent* chunk (the whole section) and hand *that* to the LLM at generation time.

The retriever gets surgical precision; the generator gets enough context to actually answer. It's the standard trick for "my retrieval is accurate but my answers are thin" — and it's a direct consequence of accepting that **the chunk you search with and the chunk you generate with don't have to be the same size.**

```python
# Store two linked records per node
child = {"id": "c1", "text": "Retries use exponential backoff...", "parent_id": "p1"}
parent = {"id": "p1", "text": "<entire 'Retry Policy' section, ~1500 tokens>"}

# Search matches the child, but you fetch the parent and pass THAT to the LLM
```

## Sizing: The Numbers That Matter

Chunk size in characters vs tokens — pick one and stick to it, because the two scales drift with language and content type. Rough rule: ~4 characters ≈ 1 token for English prose.

| Content type | Suggested chunk size | Rationale |
|--------------|---------------------|-----------|
| Dense docs / API reference | 400–600 tokens | Short, self-contained facts |
| General prose / articles | 600–1000 tokens | Fits a few paragraphs of reasoning |
| Narrative / transcripts | 800–1200 tokens | Ideas take longer to complete |
| Code | Split by function/class | Never split mid-function |

These are starting points, not laws. The only way to find *your* number is to test it.

## How to Actually Know It's Working

You can't eyeball chunk quality. Build a tiny eval and let it decide:

1. Collect 20–30 real questions with known answers.
2. For each, check whether the *right* content shows up in the top-k retrieved chunks.
3. Track **retrieval recall** (did we find it?) and **precision** (was the chunk mostly signal, not noise?).
4. Change one thing at a time — chunk size, then overlap, then strategy — and re-run.

If recall is bad, your chunks are probably too big (signal drowned) or too small (idea split). If answers are thin despite good retrieval, look at parent-child. If a specific doc type keeps failing, it's probably a structure your splitter is ignoring.

## Takeaways

- **Chunking decides retrieval quality more than your embedding model does.** Tune it before you tune anything else.
- **Start with recursive splitting** (`\n\n` → `\n` → sentence → char), chunk size ~600–1000 tokens, overlap ~10–15%.
- **Use the structure your documents already have** — Markdown headers, HTML sections, code boundaries. It's free signal.
- **Never split inside a table, list, or code block.** Make them atomic.
- **Retrieve small, generate big** with parent-child when precise matches give you thin answers.
- **Set up a retrieval eval** and change one variable at a time. Chunking is empirical — the right answer is the one your eval likes.
