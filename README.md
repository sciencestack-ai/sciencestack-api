# ScienceStack API v1 (Beta)

Access arXiv papers as structured, queryable data.

This API is designed for AI agents, research tooling, and programmatic analysis. 

ScienceStack parses arXiv papers directly from their LaTeX source — not PDFs, not OCR.
The result is a lossless, hierarchical representation of the paper that can be queried
by section, equation, figure, table, or theorem.


|  | ScienceStack | PDF + OCR |
|--|:------------:|:---------:|
| Parsed from LaTeX source (no OCR) | ✓ | ✗ |
| Pre-parsed, low-latency access | ✓ | ✗ |
| Structured AST (stable node IDs) | ✓ | ✗ |
| Clean LaTeX equations | ✓ | ✗ |
| AI summaries (paper + section) | ✓ | ✗ |
| Direct figure URLs + dimensions | ✓ | ✗ |
| Tables with structured data | ✓ | ✗ |
| Export to Markdown / LaTeX / text | ✓ | ✗ |
| Math environments as first-class objects | ✓ | ✗ |
| Resolved citations (Semantic Scholar) | ✓ | ✗ |
| Section hierarchy with depth | ✓ | ✗ |
| Algorithm and code blocks | ✓ | ✗ |

**Coverage:** All papers in [sciencestack.ai/explore](https://sciencestack.ai/explore) are available in the API

**Base URL:** `https://sciencestack.ai/api/v1`

## Authentication

All API requests require an API key passed via the `x-api-key` header:

```bash
curl "https://sciencestack.ai/api/v1/papers/1706.03762" \
  -H "x-api-key: sk_live_your_api_key_here"
```

### Getting an API Key

1. Sign in to [sciencestack.ai](https://sciencestack.ai)
2. Go to [Settings → API](https://sciencestack.ai/settings/api)
3. Create a new key and copy it (only shown once)

### Rate Limits

API access is limited by **unique papers per month**:

- Once you access a paper, you get **unlimited requests to that paper for the rest of the month**
- Accessing a new paper counts toward your monthly limit
- Limits reset on the 1st of each month

See [pricing](https://sciencestack.ai/pricing) for current tier limits.

### Your Own Papers

API keys can access:
- All public papers in the ScienceStack library
- Your own uploaded private papers

## Why This API?

PDFs are not machine-readable. Extracting equations from a PDF gives you broken Unicode like `1 X (ℓ⋆ ) A (u, v)`. This API returns clean LaTeX: `r^{\text{att}}_{i\rightarrow j} = \frac{1}{HW}\sum_{u,v}A^{(\ell^\star)}_{i\rightarrow j}(u,v)`.

**What you get:**
- **Clean LaTeX** - Every equation as source, not OCR garbage
- **Direct image URLs** - Figures ready for vision models, no PDF extraction
- **Semantic structure** - TOC, sections, theorems, proofs all queryable by ID
- **AI summaries** - Pre-computed paper and section summaries
- **One-call overview** - Everything an agent needs in a single request
- **Citation enrichment** - arXiv IDs, DOIs, Semantic Scholar links
- **Format conversion** - Full paper as Markdown, LaTeX, or plain text

## Quick Start

```bash
# Set your API key
export SCIENCESTACK_API_KEY="sk_live_your_key_here"

# Search for papers
curl "https://sciencestack.ai/api/v1/search?q=attention+mechanisms" \
  -H "x-api-key: $SCIENCESTACK_API_KEY"

# Get everything an agent needs in one call
curl "https://sciencestack.ai/api/v1/papers/1706.03762/overview" \
  -H "x-api-key: $SCIENCESTACK_API_KEY"

# Get paper metadata
curl "https://sciencestack.ai/api/v1/papers/1706.03762" \
  -H "x-api-key: $SCIENCESTACK_API_KEY"

# Get all equations with clean LaTeX
curl "https://sciencestack.ai/api/v1/papers/1706.03762/equations" \
  -H "x-api-key: $SCIENCESTACK_API_KEY"

# Get all figures with image URLs
curl "https://sciencestack.ai/api/v1/papers/1706.03762/figures" \
  -H "x-api-key: $SCIENCESTACK_API_KEY"

# Get all tables
curl "https://sciencestack.ai/api/v1/papers/1706.03762/tables" \
  -H "x-api-key: $SCIENCESTACK_API_KEY"

# Get all theorems/lemmas/proofs
curl "https://sciencestack.ai/api/v1/papers/1706.03762/theorems" \
  -H "x-api-key: $SCIENCESTACK_API_KEY"

# Get a specific section by nodeId
curl "https://sciencestack.ai/api/v1/papers/1706.03762/nodes?nodeId=sec:3.2" \
  -H "x-api-key: $SCIENCESTACK_API_KEY"
```

## Use Cases

### Agent Walkthrough: Reading a Paper

How an LLM agent should read a paper using this API:

```
0. GET /search?q=attention+mechanisms
   → Find papers by topic, author, or arXiv ID
   → Returns paper IDs to explore further

1. GET /papers/{arxivId}/overview
   → Cache this response
   → Provides TOC, AI summary, section summaries, and lists of all figures,
     tables, equations, and theorems
   → Use this as the index for all further navigation

2. Skim overview.sectionSummaries
   → Identify which sections are relevant to the task
   → Note section nodeIds (e.g. sec:3.2)

3. GET /papers/{id}/nodes?nodeId=sec:3.2
   → Fetch full content of the selected section
   → Use &context=N if nearby paragraphs are needed

4. If an equation is referenced (e.g. "see Eq. 7"):
   GET /papers/{id}/nodes?nodeId=eq:7&context=3
   → Retrieve clean LaTeX plus surrounding explanation

5. If a figure is referenced:
   - overview.figures has nodeId, number, caption (metadata only)
   - GET /papers/{id}/figures for full data with imageUrl
   - Or GET /papers/{id}/nodes?nodeId=fig:3 for node structure
   → Feed imageUrl from /figures to a vision model

6. If a theorem, lemma, or proof is referenced:
   GET /papers/{id}/nodes?nodeId=sec:4:12
   → Retrieve the full theorem/proof block

7. Use typed endpoints when bulk access is needed:
   - /equations → analyze all equations
   - /figures   → analyze all visuals
   - /tables    → extract tabular data
   - /theorems  → reason over formal results

8. For citations, check overview or GET /papers/{id}/references
   → Get enriched metadata (Semantic Scholar, DOI, arXiv links)
```

Key principles:
- **Search first**: Use `/search` to find papers, not browse
- **Start broad, drill down**: overview → section → specific node
- **Full-text search**: Use `/content?format=markdown` for whole-paper search, not node-by-node
- **Equations in text**: `format=text` preserves LaTeX inline (`$d_k$`). Use `/equations` if you need them separately.

### 1. AI Research Assistant
```bash
# Step 1: Get paper overview (TOC, summaries, structure)
curl ".../papers/2512.04012/overview"

# Step 2: Identify relevant section from TOC
# Response includes: {"nodeId": "sec:4", "title": "Methods", ...}

# Step 3: Fetch specific section content
curl ".../papers/2512.04012/nodes?nodeId=sec:4"

# Step 4: If equation referenced, get clean LaTeX
curl ".../papers/2512.04012/nodes?nodeId=eq:7&context=3"
```

### 2. Math/Equation Extraction
```bash
# Get all numbered equations with LaTeX source
curl ".../papers/1706.03762/equations"
# Returns: [{"nodeId": "eq:1", "number": "1", "latex": "\\mathrm{Attention}(Q,K,V)=..."}, ...]

# Get equation with surrounding context (for explanation)
curl ".../papers/1706.03762/nodes?nodeId=eq:1&context=3"
```

### 3. Figure Analysis
```bash
# Get all figures with direct image URLs
curl ".../papers/2512.04012/figures"
# Returns: [{"nodeId": "fig:1", "imageUrl": "https://...", "caption": "...", "width": 800, "height": 600}]

# Feed image URL directly to vision model - no PDF page extraction needed
```

### 4. Theorem/Proof Navigation
```bash
# Get paper overview - includes all math environments
curl ".../papers/1509.05363/overview"
# Response includes mathEnvs: [{"nodeId": "sec:4:12", "type": "theorem", "number": "3.1", "title": "Main Result"}]

# Fetch specific theorem content
curl ".../papers/1509.05363/nodes?nodeId=sec:4:12"
```

### 5. Citation Graph Building
```bash
# Get references with Semantic Scholar enrichment
curl ".../papers/1706.03762/references"
# Returns: arxivId, doi, Semantic Scholar paperId for each reference
```

### 6. LLM Context Injection
```bash
# Get full paper as markdown (best for LLMs)
curl ".../papers/1706.03762/content?format=markdown"

# Or get specific section only (saves tokens)
curl ".../papers/1706.03762/nodes?nodeId=sec:3&descendants=true&format=text"
```

### Node IDs

All content elements are addressable via stable `nodeId`s.

Format:
- `sec:{number}` – section (e.g. `sec:3.2`)
- `eq:{number}` – equation
- `fig:{number}` – figure
- `tab:{number}` – table
- `lem:{number}` – lemma
- `thm:{number}` – theorem
- etc.

All `nodeId`s can be fetched via:
```
GET /papers/{id}/nodes?nodeId={nodeId}
```

**Note:** Only structural nodes (sections, equations, figures, tables, theorems, etc.) are queryable by `nodeId`. Inline text fragments (e.g., `sec:1:4`) are embedded within their parent node's content and cannot be fetched individually. Use the parent section's `nodeId` to retrieve all content within it.

This is a low-level, power-user endpoint. Most agents should prefer typed endpoints (`/equations`, `/figures`, `/overview`) when possible.


## Endpoints

### Search

#### `GET /search?q={query}`

Search papers across the catalog. This is the primary discovery endpoint.

| Param | Type | Description |
|-------|------|-------------|
| `q` | string | Search query (required, min 2 chars) |
| `limit` | int | Max results (default: 20, max: 100) |
| `offset` | int | Pagination offset |

**Search behavior** (in order of precedence):
1. **Exact arXiv ID**: `1706.03762` or `1706.03762v7` → instant match
2. **Full-text search**: Searches title and abstract with stemming
3. **Author name**: Falls back to author search if no title/abstract matches
4. **Fuzzy match**: Last resort for typos (similarity > 0.2)

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Attention Is All You Need",
      "arxivId": "1706.03762v7",
      "authors": [{"name": "Ashish Vaswani"}],
      "published": "2017-06-12T00:00:00Z",
      "citationCount": 120000,
      "primaryCategory": "cs.CL"
    }
  ],
  "pagination": {"offset": 0, "limit": 20, "hasMore": true},
  "_version": "1.0.0"
}
```

**Examples:**
```bash
# Search by topic
curl ".../search?q=transformer+architecture"

# Search by author
curl ".../search?q=Vaswani"

# Search by arXiv ID
curl ".../search?q=1706.03762"

# Paginate results
curl ".../search?q=attention&limit=50&offset=100"
```

---

### Papers

#### `GET /papers/{id}`

Get paper metadata. Accepts arXiv ID (e.g., `1706.03762`) or internal UUID.

**Response:**
```json
{
  "data": {
    "id": "uuid",
    "title": "Attention Is All You Need",
    "arxivId": "1706.03762v7",
    "authors": [{"name": "Ashish Vaswani"}],
    "summary": "The dominant sequence transduction models...",
    "primaryCategory": "cs.CL",
    "published": "2017-06-12T00:00:00Z",
    "citationCount": 120000
  },
  "_version": "1.0.0"
}
```

#### `GET /papers/{id}/content`

Get full paper content in a single call. Ideal for LLM context injection.

| Param | Type | Description |
|-------|------|-------------|
| `format` | string | `markdown` (default), `latex`, or `text` |
| `styles` | boolean | Include text styling (bold, italic). Default: `false` (cleaner for LLMs) |

**Response:**
```json
{
  "data": {
    "id": "uuid",
    "title": "Attention Is All You Need",
    "arxivId": "1706.03762v7",
    "format": "markdown",
    "content": "# Attention Is All You Need\n\n## Abstract\n\n..."
  },
  "_version": "1.0.0"
}
```

#### `GET /papers/{id}/overview`

Get everything an agent needs in one call: metadata, TOC, summaries, figures, tables, equations, and theorems.

**Response:**
```json
{
  "data": {
    "id": "uuid",
    "title": "Attention Is All You Need",
    "arxivId": "1706.03762",
    "abstract": "The dominant sequence transduction models...",
    "authors": [{"name": "Ashish Vaswani"}],
    "published": "2017-06-12T00:00:00Z",
    "aiSummary": "This paper introduces the Transformer architecture...",
    "toc": [
      {"section": "1", "nodeId": "sec:1", "title": "Introduction", "depth": 1},
      {"section": "3.2", "nodeId": "sec:3.2", "title": "Attention", "depth": 2}
    ],
    "sectionSummaries": [
      {"section": "1", "nodeId": "sec:1", "summary": "Introduces the motivation for..."}
    ],
    "figures": [
      {"nodeId": "fig:1", "number": "1", "caption": "The Transformer model architecture"}
    ],
    "tables": [
      {"nodeId": "tab:1", "number": "1", "caption": "Comparison of model architectures"}
    ],
    "equations": [
      {"nodeId": "eq:1", "number": "1", "latex": "\\mathrm{Attention}(Q,K,V)=\\mathrm{softmax}(...)"}
    ],
    "mathEnvs": [
      {"nodeId": "sec:4:12", "type": "theorem", "number": "3.1", "title": "Main Result"}
    ]
  },
  "_version": "1.0.0"
}
```

#### `GET /papers/{id}/figures`

Get all figures with image URLs and dimensions.

**Response:**
```json
{
  "data": [
    {
      "nodeId": "fig:1",
      "number": "1",
      "caption": "The Transformer model architecture",
      "imageUrl": "https://storage.sciencestack.ai/papers/.../fig1.png",
      "width": 800,
      "height": 600
    }
  ],
  "_version": "1.0.0"
}
```

#### `GET /papers/{id}/equations`

Get all numbered equations with clean LaTeX.

**Response:**
```json
{
  "data": [
    {
      "nodeId": "eq:1",
      "number": "1",
      "latex": "\\mathrm{Attention}(Q, K, V) = \\mathrm{softmax}\\left(\\frac{QK^T}{\\sqrt{d_k}}\\right)V"
    }
  ],
  "_version": "1.0.0"
}
```

#### `GET /papers/{id}/tables`

Get all tables with captions.

**Response:**
```json
{
  "data": [
    {
      "nodeId": "tab:1",
      "number": "1",
      "caption": "Comparison of model architectures on WMT 2014"
    }
  ],
  "_version": "1.0.0"
}
```

#### `GET /papers/{id}/theorems`

Get all theorem-like environments (theorems, lemmas, proofs, definitions, corollaries, etc.).

**Response:**
```json
{
  "data": [
    {
      "nodeId": "sec:4:12",
      "type": "theorem",
      "number": "3.1",
      "title": "Main Convergence Result"
    }
  ],
  "_version": "1.0.0"
}
```

### Nodes

Paper content is stored as a tree of typed nodes: sections, equations, figures, tables, etc.

#### `GET /papers/{id}/nodes`

Get nodes with optional filtering.

| Param | Type | Description |
|-------|------|-------------|
| `types` | string | Comma-separated node types (e.g., `equation,figure`) |
| `nodeId` | string | Get specific node and its descendants |
| `descendants` | bool | Include descendants when nodeId set (default: `true`) |
| `context` | int | When nodeId set, include N nodes before/after (stops at section boundary) |
| `startIndex` | int | Filter by order_index range (inclusive) |
| `endIndex` | int | Filter by order_index range (exclusive) |
| `limit` | int | Max results (default: 100, max: 500) |
| `offset` | int | Pagination offset |
| `format` | string | `raw` (default) or `text` |

**Node Types:**
- `section` - Document sections with hierarchy
- `equation` - Numbered equations
- `equation_array` - Multi-line equation blocks
- `figure` - Figures with images and captions
- `table` - Tables with structured data
- `algorithm` - Algorithm pseudocode
- `code` - Code blocks
- `math_env` - Theorem, lemma, proof, etc.
- `list` - Bulleted/numbered lists
- `content_block` - Paragraphs and inline content

**Example - Get a lemma with formatted content:**
```bash
curl "https://sciencestack.ai/api/v1/papers/2512.19334/nodes?nodeId=lem:8&format=markdown"
```
```json
{
  "data": "Let  $\\bm{Y}$  follow the rectangular spiked model with  $\\bm{W}$  having i.i.d.  $\\mathcal{N}(0,1/N)$  entries, and let  $M,N\\to\\infty$  with  $M/N\\to\\delta\\in(0,1)$ . Then the master equation \n\n $\\Gamma(\\lambda)=1-\\theta^2\\mathcal{C}(\\lambda)=0$ \n\nadmits no real solution on  $[0,a_-)$ .",
  "_version": "1.0.0"
}
```

**Example - Get all equations:**
```bash
curl "https://sciencestack.ai/api/v1/papers/1706.03762/nodes?types=equation"
```

**Response:**
```json
{
  "data": [
    {
      "nodeId": "eq:1",
      "type": "equation",
      "orderIndex": 27,
      "depth": 4,
      "parentNodeId": "sec:3.2.1",
      "content": [{"type": "text", "content": "\\mathrm{Attention}(Q, K, V) = ..."}],
      "metadata": {
        "numbering": "1",
        "anchor": "eq-1",
        "envName": "equation"
      }
    }
  ],
  "pagination": {"offset": 0, "limit": 100, "hasMore": false},
  "_version": "1.0.0"
}
```

**Example - Get figure with images and caption:**
```bash
curl "https://sciencestack.ai/api/v1/papers/1706.03762/nodes?nodeId=fig:1"
```

Returns the figure node plus all children (images, caption).

**Example - Get just the figure metadata (no children):**
```bash
curl "https://sciencestack.ai/api/v1/papers/1706.03762/nodes?nodeId=fig:1&descendants=false"
```

**Example - Get equation with surrounding context (for LLM explanation):**
```bash
curl "https://sciencestack.ai/api/v1/papers/1706.03762/nodes?nodeId=eq:1&context=3"
```

Returns the equation plus up to 3 nodes before and after. Stops at section boundaries (nodes with lower depth than the target).

### References

#### `GET /papers/{id}/references`

Get bibliography entries with optional enrichment data.

| Param | Type | Description |
|-------|------|-------------|
| `limit` | int | Max results (default: 100, max: 500) |
| `offset` | int | Pagination offset |
| `format` | string | `raw` (default) or `text` |

**Response:**
```json
{
  "data": [
    {
      "citeKey": "vaswani2017attention",
      "orderIndex": 0,
      "arxivId": "1706.03762",
      "doi": "10.48550/arXiv.1706.03762",
      "content": [...],
      "metadata": {"format": "bibtex"},
      "enrichment": {
        "status": "enriched",
        "semanticScholar": {
          "paperId": "abc123",
          "title": "Attention Is All You Need",
          "authors": [{"name": "Ashish Vaswani"}],
          "publicationDate": "2017-06-12"
        }
      }
    }
  ],
  "pagination": {"offset": 0, "limit": 100, "hasMore": false},
  "_version": "1.0.0"
}
```

## Content Formats

### `/nodes` endpoint

| Format | Response | Use Case |
|--------|----------|----------|
| `raw` (default) | `{"data": [{nodeId, type, content: [...tokens]}]}` | Programmatic processing |
| `text` | `{"data": "Plain text string"}` | Search, embeddings |
| `markdown` | `{"data": "# Section\n\nContent..."}` | LLM context, display |
| `latex` | `{"data": "\\section{...}"}` | Academic tooling |

**Example - raw (default):**
```bash
curl ".../papers/1706.03762/nodes?nodeId=sec:3"
```
```json
{"data": [{"nodeId": "sec:3", "type": "section", "content": [...], "metadata": {...}}]}
```

**Example - text:**
```bash
curl ".../papers/1706.03762/nodes?nodeId=sec:3&format=text"
```
```json
{"data": "Model Architecture\n\nMost competitive neural sequence..."}
```

### `/content` endpoint

| Format | Response | Use Case |
|--------|----------|----------|
| `markdown` (default) | `{"data": {content: "# Title\n\n..."}}` | LLM context, display |
| `latex` | `{"data": {content: "\\section{...}"}}` | Academic tooling |
| `text` | `{"data": {content: "Plain text..."}}` | Embeddings, search |

**Example:**
```bash
curl ".../papers/1706.03762/content?format=markdown"
```
```json
{"data": {"id": "...", "format": "markdown", "content": "# Attention Is All You Need\n\n## Abstract\n\n..."}}
```

## Errors

```json
{
  "error": {
    "code": "PAPER_NOT_FOUND",
    "message": "Paper not found: 9999.99999"
  }
}
```

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid API key |
| `RATE_LIMITED` | 429 | Too many requests per minute |
| `PAPER_LIMIT_EXCEEDED` | 429 | Monthly paper limit reached |
| `PAPER_NOT_FOUND` | 404 | Paper doesn't exist or not accessible |
| `NODE_NOT_FOUND` | 404 | Node doesn't exist |
| `INVALID_QUERY` | 400 | Bad search query |
| `UNSUPPORTED_FORMAT` | 400 | Invalid format param |
| `INTERNAL_ERROR` | 500 | Server error |

## Pagination

List endpoints return pagination info:

```json
{
  "data": [...],
  "pagination": {
    "offset": 0,
    "limit": 100,
    "hasMore": true
  }
}
```

Use `offset` and `limit` params to paginate through results.

## Versioning

All responses include `_version: "1.0.0"`. Breaking changes will increment the major version and be announced in advance.
