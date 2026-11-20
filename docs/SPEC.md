# promptshape specification

Plan only; no code exists yet. This file defines behaviour. Reasons are in `DECISIONS.md`, current state in `HANDOFF.md`.

## 1. Scope and layout

promptshape renders structured data into prompt text in four formats and reports what each format costs in tokens for a given Claude model. A Rust extension, `promptshape._bpe`, implements byte-pair encoding as a learning piece. Not in scope: prompt templates, building Messages requests, async, containers of several items, reproducing Claude's tokenizer.

```
pyproject.toml  Cargo.toml (workspace)  CHANGELOG.md  README.md  .gitattributes
python/promptshape/  __init__.py  types.py  errors.py  render.py  counting.py  bpe.py  cli.py  py.typed  _bpe.pyi
                     _scalars.py  _cache.py  _retry.py  _render/{xml,markdown,json,text}.py
rust/bpe/            src/{lib,pretok,train,encode,decode}.rs  benches/encode.rs    (pure Rust crate promptshape-bpe)
rust/py/src/lib.rs   (PyO3 bindings, module promptshape._bpe)
tests/               conftest.py  fakes.py  parsers.py  strategies.py  golden/  data/gpt2/  test_*.py
benches/compare.py   scripts/{live_count,gen_gpt2_golden}.py   examples/   .github/workflows/{ci,release}.yml
```

## 2. Public API (Python 3.10+)

Callers pass data and a format name and get a `str`. Tag names, separators and escaping are private and not importable.

### 2.1 Signatures (`promptshape/__init__.py` exports exactly these, plus `__version__`)

```python
Cell: TypeAlias = None | bool | int | float | str
JSONValue: TypeAlias = None | bool | int | float | str | Sequence["JSONValue"] | Mapping[str, "JSONValue"]
FormatName: TypeAlias = Literal["xml", "markdown", "json", "text"]
FORMATS: tuple[FormatName, ...] = ("xml", "markdown", "json", "text")

@dataclass(frozen=True, slots=True)
class Table:
    columns: Sequence[str]                 # stored as tuple[str, ...]
    rows: Sequence[Sequence[Cell]]         # stored as tuple[tuple[Cell, ...], ...]
    @classmethod
    def from_dicts(cls, records: Sequence[Mapping[str, Cell]]) -> "Table": ...  # columns in first-seen order, missing -> None
@dataclass(frozen=True, slots=True)
class Document: title: str; source: str; body: str
@dataclass(frozen=True, slots=True)
class ToolResult: name: str; input: JSONValue; output: JSONValue; is_error: bool = False
Renderable: TypeAlias = Table | Document | ToolResult | JSONValue  # not one of the three classes => treated as a JSON value
def render(data: Renderable, format: FormatName) -> str: ...

class CountTokensClient(Protocol):         # satisfied by anthropic.Anthropic and by tests/fakes.py
    @property
    def messages(self) -> _SupportsCountTokens: ...   # .count_tokens(*, model, messages) -> object with .input_tokens: int
@dataclass(frozen=True, slots=True)
class RetryPolicy: max_attempts: int = 6; base_delay: float = 1.0; max_delay: float = 30.0   # delays in seconds
@dataclass(frozen=True, slots=True)
class FormatCost:
    format: FormatName
    model: str
    tokens: int                            # with_data_tokens - baseline_tokens
    with_data_tokens: int
    baseline_tokens: int
    characters: int                        # len(rendered)
    from_cache: bool                       # True when no request was sent for this result
class TokenCounter:                        # also a context manager
    def __init__(self, *, model: str, client: CountTokensClient | None = None, api_key: str | None = None,
                 cache_path: Path | None = None, use_cache: bool = True, retry: RetryPolicy = RetryPolicy()) -> None: ...
    def cost(self, data: Renderable, format: FormatName) -> FormatCost: ...
    def compare(self, data: Renderable, formats: Sequence[FormatName] = FORMATS) -> list[FormatCost]: ...  # cheapest first
    def close(self) -> None: ...
def default_cache_path() -> Path: ...

class PromptshapeError(Exception): ...
class InvalidDataError(PromptshapeError, ValueError): ...      # message names the location, e.g. rows[2][1], input.q
class UnknownFormatError(PromptshapeError, ValueError): ...
class MissingCredentialsError(PromptshapeError): ...
class CacheError(PromptshapeError): ...
class CountError(PromptshapeError): ...                        # non-retryable API failure; .status_code: int | None
class RateLimitedError(CountError): ...                        # retries exhausted on 429; .retry_after: float | None
```

### 2.2 Usage

```python
from promptshape import Table, render                                   # 1. render
stock = Table(columns=("name", "qty"), rows=(("apple", 3), ("pear", None)))
prompt = f"Stock list:\n\n{render(stock, 'markdown')}\n\nWhich items have no quantity?"

import os, anthropic                                                    # 2. compare formats for one model
from promptshape import Document, TokenCounter
client = anthropic.Anthropic(api_key=os.environ["PROMPTSHAPE_ANTHROPIC_API_KEY"])   # the caller reads the key, not the library
doc = Document(title="Q3 report", source="https://example.com/q3", body=open("q3.txt").read())
with TokenCounter(model="claude-opus-5-5", client=client) as counter:
    for c in counter.compare(doc):
        print(f"{c.format:9} {c.tokens:6} tokens  {c.characters:7} chars")

from promptshape import ToolResult                                      # 3. pick the cheapest format, then render with it
result = ToolResult(name="search", input={"q": "bpe"}, output={"hits": ["a", "b"]})
with TokenCounter(model="claude-opus-5-5", api_key=my_key) as counter:
    best = counter.compare(result, formats=("xml", "json"))[0]
block = render(result, best.format)
```

### 2.3 CLI

`promptshape render --format F [FILE|-]`, `promptshape count --model M [--format F]... [--no-cache] [FILE|-]`, `promptshape cache {path,clear}`. Input is a JSON envelope: `{"type":"table","columns":[..],"rows":[[..]]}`, `{"type":"document","title":..,"source":..,"body":..}`, `{"type":"tool_result","name":..,"input":..,"output":..,"is_error":..}`, `{"type":"json","value":..}`. Exit codes: 0 ok, 2 usage or invalid data, 3 missing key, 4 API failure.

## 3. Data rules

- `Table`: at least one column; every row has exactly `len(columns)` cells; zero rows is valid.
- JSON values: mapping keys are `str`; floats are finite; `bool` is tested before `int`; lists, tuples and any `Mapping` are accepted; cycles and other types are rejected.
- Every string (cells, column names, fields, JSON strings and keys) must be UTF-8 encodable (no lone surrogates).
- Violations raise `InvalidDataError`. The three classes validate on construction; a bare JSON value is validated by `render`.

## 4. Formats and golden output

### 4.1 Shared building blocks (private, `_scalars.py`)

- `scalar_text(cell)`: `None` → empty string, `True`/`False` → `true`/`false`, `int` → decimal, `float` → `repr()`, `str` → itself.
- `compact_json(v)`: `json.dumps(v, ensure_ascii=False, separators=(",", ":"), allow_nan=False)`, keys in insertion order.
- `payload(v)`: a `str` → (the string, `text`); anything else → (`compact_json(v)`, `json`).
- `oneline(s)`: each `\r\n`, `\r` or `\n` becomes one space.
- Output lines are joined with `\n` (never `\r\n`); no trailing newline is added.

Golden inputs (outputs stored byte-exact as `tests/golden/{table,document,tool_result,value}.{xml,markdown,json,text}.txt`). In the golden blocks below, lines starting with `── ` are labels, not output.

```python
TABLE = Table(columns=("name", "qty", "note"), rows=(("apple", 3, "fresh"), ("pear", None, "a|b"), ("fig", 1.5, "two\nlines")))
DOC   = Document(title="Q3 report", source="https://example.com/q3", body="Revenue rose 4% & costs fell.\nSee </document> for details.")
TOOL  = ToolResult(name="search", input={"q": "llm & tokens", "limit": 2}, output="2 hits\n- a")
VALUE = {"user": "ada", "tags": ["x", "y"], "active": True, "score": None}
```

### 4.2 XML tags

Elements only: no attributes, no declaration. Leaf text is the escaped value with no added whitespace. A table row is one line.

```
── table
<table>
<columns><column>name</column><column>qty</column><column>note</column></columns>
<row><cell>apple</cell><cell>3</cell><cell>fresh</cell></row>
<row><cell>pear</cell><cell></cell><cell>a|b</cell></row>
<row><cell>fig</cell><cell>1.5</cell><cell>two
lines</cell></row>
</table>
── document
<document>
<title>Q3 report</title>
<source>https://example.com/q3</source>
<document_content>Revenue rose 4% &amp; costs fell.
See &lt;/document&gt; for details.</document_content>
</document>
── tool_result
<tool_result>
<name>search</name>
<is_error>false</is_error>
<input>{"q":"llm &amp; tokens","limit":2}</input>
<output>2 hits
- a</output>
</tool_result>
── value
<data>{"user":"ada","tags":["x","y"],"active":true,"score":null}</data>
```

### 4.3 Markdown

Table: GFM pipe table, row = `"| " + " | ".join(cells) + " |"`. Document and tool result are positional: fixed header lines, then the free text. Fence = N backticks + info, payload, N backticks, with N = max(3, longest backtick run in payload + 1). `Status:` is `ok` or `error`. The document body starts at line 5 and runs to the end.

````
── table
| name | qty | note |
| --- | --- | --- |
| apple | 3 | fresh |
| pear |  | a\|b |
| fig | 1.5 | two<br>lines |
── document
# Q3 report

Source: https://example.com/q3

Revenue rose 4% & costs fell.
See </document> for details.
── tool_result
### Tool result: search

Status: ok

Input:
```json
{"q":"llm & tokens","limit":2}
```

Output:
```text
2 hits
- a
```
── value
```json
{"user":"ada","tags":["x","y"],"active":true,"score":null}
```
````

### 4.4 JSON

`compact_json` of: table → `{"columns", "rows"}` (rows as arrays), document → `{"title", "source", "body"}`, tool result → `{"name", "input", "output", "is_error"}`, JSON value → itself. Always one line.

```
── table
{"columns":["name","qty","note"],"rows":[["apple",3,"fresh"],["pear",null,"a|b"],["fig",1.5,"two\nlines"]]}
── document
{"title":"Q3 report","source":"https://example.com/q3","body":"Revenue rose 4% & costs fell.\nSee </document> for details."}
── tool_result
{"name":"search","input":{"q":"llm & tokens","limit":2},"output":"2 hits\n- a","is_error":false}
── value
{"user":"ada","tags":["x","y"],"active":true,"score":null}
```

### 4.5 Plain text

Table: tab-separated, header line first (`⇥` below is one TAB byte). Document and tool result are positional. Tool input is always `compact_json` (one line); tool output and a bare JSON value use `payload` text.

```
── table
name⇥qty⇥note
apple⇥3⇥fresh
pear⇥⇥a|b
fig⇥1.5⇥two\nlines
── document
Title: Q3 report
Source: https://example.com/q3

Revenue rose 4% & costs fell.
See </document> for details.
── tool_result
Tool: search
Status: ok
Input: {"q":"llm & tokens","limit":2}
Output:
2 hits
- a
── value
{"user":"ada","tags":["x","y"],"active":true,"score":null}
```

## 5. Escaping: content can never break structure

| Format | Where | Rule, applied in this order |
|---|---|---|
| XML | every leaf text | chars illegal in XML 1.0 (U+0000–0008, 000B, 000C, 000E–001F, FFFE, FFFF) → U+FFFD; `&` → `&amp;`; `<` → `&lt;`; `>` → `&gt;`; CR → `&#13;` |
| Markdown | table cells and headers | trim spaces and tabs at both ends; backslash → two backslashes; pipe → backslash + pipe; CRLF, CR or LF → `<br>` |
| Markdown | title, source, tool name | `oneline` |
| Markdown | fenced payloads | none needed: the fence is longer than any backtick run inside |
| Markdown, text | document body, text tool output | none needed: positional tail, nothing follows it |
| JSON | everything | JSON string escaping from `json.dumps` |
| Text | table cells and headers | backslash → two backslashes; TAB → `\t`; LF → `\n`; CR → `\r` (each a two-character sequence) |
| Text | title, source, tool name | `oneline` |
| Text | tool input | `compact_json`, so it is always exactly one line |

Examples (input → output): XML `a < b && c > d` → `a &lt; b &amp;&amp; c &gt; d`; XML body `</document>` → `&lt;/document&gt;`; XML x, CR, y → `x&#13;y`; Markdown cell `a|b` → `a\|b`; Markdown cell `a\|b` → `a\\\|b`; Markdown cell x, CR, LF, y → `x<br>y`; a Markdown payload containing three backticks gets a four-backtick fence; text cell a, TAB, b → `a\tb`; text cell `C:\dir` → `C:\\dir`.

Lossy by design, structure still intact: `None` and `""` look the same outside JSON; XML-illegal chars become U+FFFD; Markdown cells lose outer whitespace and a literal `<br>` in a cell reads as a line break; `oneline` fields lose line breaks; string payloads are unquoted, so `"1"` and `1` look the same outside JSON. JSON is the only lossless format.

Property tests (hypothesis). Reference parsers live in `tests/parsers.py` and share no code with the renderers (stdlib `xml.etree` and `json`; hand-written pipe-table, TSV and positional parsers). `tests/strategies.py` mixes `st.text()` with hostile snippets: closing tags (`</document>`, `</cell></row>`), `]]>`, `&amp;`, pipes, backslashes, backslash + pipe, `<br>`, runs of 3 and 4 backticks, CRLF, TAB, NUL, U+2028, `# heading`, `Source: x`, and a table separator line.

| ID | Test name | Statement, for any generated data |
|---|---|---|
| P1 | `test_xml_is_well_formed` | `ElementTree.fromstring(render(x, "xml"))` never raises |
| P2 | `test_xml_table_cells_survive` | parsed shape equals the input shape; each cell text equals `xml_clean(scalar_text(cell))` |
| P3 | `test_xml_document_fields_survive` | root `document` has exactly `title`, `source`, `document_content` with the cleaned field texts |
| P4 | `test_xml_tool_result_fields_survive` | exactly four children; `input` and `output` texts equal the cleaned payload texts |
| P5 | `test_markdown_table_shape_survives` | `rows + 2` lines; every line splits into `len(columns)` cells; cells equal `md_norm(scalar_text(cell))` |
| P6 | `test_markdown_document_is_positional` | parser returns `(oneline(title), oneline(source), body)` with body unchanged |
| P7 | `test_markdown_fence_outlasts_backticks` | fenced payloads are recovered exactly for tool results and JSON values |
| P8 | `test_text_table_roundtrips` | parsed cells equal `scalar_text(cell)` exactly, including tabs, newlines and backslashes |
| P9 | `test_text_document_is_positional` | as P6 for the text format |
| P10 | `test_text_tool_result_roundtrips` | name, status, `json.loads(input line)` and the output tail are recovered |
| P11 | `test_json_roundtrips` | `json.loads(render(x, "json"))` equals the canonical structure of `x`, for all four types |
| P12 | `test_render_is_deterministic` | two calls return identical strings |
| P13 | `test_lone_surrogates_are_rejected` | any string field containing a lone surrogate raises `InvalidDataError` in every format |

## 6. Token counting

Endpoint: `client.messages.count_tokens(model=..., messages=[...])` → `.input_tokens` (`POST /v1/messages/count_tokens`). Facts from the docs (checked 2026-11-20; re-check in T11): free; rate-limited in requests per minute per usage tier, separately from Messages (Start 5,000, Build 10,000, Scale 20,000); the result is an estimate; it may include system-added tokens (not billed); counts are model-specific; prompt caching plays no part.

- Requests, with `CARRIER = "Data:"` (an empty user message is not assumed to be accepted): baseline `{"model": M, "messages": [{"role": "user", "content": [{"type": "text", "text": CARRIER}]}]}`; with data: the same plus a second block `{"type": "text", "text": rendered}`.
- Cost = `count(with data) - count(baseline)`, same model, same counter. The subtraction cancels message overhead and system-added tokens to the extent they are constant. Reported as is, never clamped. `rendered == ""` → all zero, no request.
- Cache: SQLite file `platformdirs.user_cache_dir("promptshape") / "token_counts.sqlite3"`, overridable with `cache_path=`. `CREATE TABLE token_counts (model TEXT NOT NULL, request_sha256 TEXT NOT NULL, input_tokens INTEGER NOT NULL, created_at TEXT NOT NULL, PRIMARY KEY (model, request_sha256)) WITHOUT ROWID`; `PRAGMA user_version = 1`; WAL; `busy_timeout = 5000`. Key = (model ID, sha256 hex of the full request as canonical JSON: `sort_keys=True`, `separators=(",", ":")`, `ensure_ascii=False`, UTF-8). The model is a key column and is also inside the hashed request. Only successful counts are stored; prompt text is never stored. No expiry: use `use_cache=False` or `promptshape cache clear`. A file that is not this schema raises `CacheError`. Cold `compare` over four formats = 5 requests; warm = 0.
- Retries: one layer, ours. SDK retries are switched off (`max_retries=0` on clients we build, `with_options(max_retries=0)` on a passed client when it has that method). Classification by duck typing: `status_code == 429` → rate limited; 408, 409, ≥ 500 → transient; `ConnectionError`, `TimeoutError`, `anthropic.APIConnectionError` → transient; anything else → `CountError` at once. Wait before retry n (0-based): the `retry-after` header in seconds when present (above 120 s: give up now), else full jitter `uniform(0, min(max_delay, base_delay * 2**n))`. After `max_attempts`: `RateLimitedError` or `CountError`. A 429 without `retry-after` may be a spend cap; it gets the same bounded retries. `sleep` and `rng` are injectable for tests. No client-side throttle (the lowest tier allows 5,000 RPM).

## 7. API key handling

- The library never reads `ANTHROPIC_API_KEY` or any other environment variable and never builds a client without an explicit key. `TokenCounter` needs exactly one of `client=` or `api_key=`; neither → `MissingCredentialsError`, both → `ValueError`.
- Reason: `ANTHROPIC_API_KEY` in the environment makes Claude Code bill the API account instead of the subscription, so nothing in this project may require exporting it.
- The CLI, `examples/` and `scripts/` read `PROMPTSHAPE_ANTHROPIC_API_KEY` and pass it as `api_key=`. Missing → exit 3.
- A client built by the caller is the caller's business (the SDK may read `ANTHROPIC_*` variables when given no explicit key).
- Keys never appear in logs, exception messages or the cache.
- Enforced by tests in T11 and T13, including a source scan: `ANTHROPIC_API_KEY` appears under `python/promptshape/` only as part of `PROMPTSHAPE_ANTHROPIC_API_KEY`.
- Ordering: no task needs a key to be written or tested. The key is needed only to run T18.

## 8. Test safety

- Tests never call the real API. They use `tests/fakes.py::FakeCountClient`: `input_tokens = 10 + UTF-8 bytes of all text blocks`, records every call, and raises scripted `FakeAPIStatusError(status_code, headers)` values.
- Autouse fixtures in `tests/conftest.py`, with no marker, option or variable to switch them off: `_no_network` replaces `socket.socket.connect`, `connect_ex`, `socket.create_connection` and `socket.getaddrinfo` with functions raising `NetworkBlockedError`; `_no_real_anthropic_client` patches `anthropic.Anthropic.__init__` and `anthropic.AsyncAnthropic.__init__` to raise `RealClientBlockedError`; `_clean_env` removes `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN` and `PROMPTSHAPE_ANTHROPIC_API_KEY` and points the default cache path at `tmp_path`.
- `tests/test_guards.py` proves each guard fires. pytest runs with `filterwarnings = error` and `testpaths = ["tests"]`.
- Live calls exist only in `scripts/live_count.py`. Without `--live` it exits 2 before building a client. It is never collected by pytest and never referenced from a workflow. CI defines no secrets.

## 9. Rust BPE extension (`promptshape._bpe`)

### 9.1 Python surface (`promptshape.bpe`, a thin typed wrapper)

```python
def train_merges(text: str, num_merges: int) -> list[tuple[bytes, bytes]]: ...
class Tokenizer:
    def __init__(self, merges: Sequence[tuple[bytes, bytes]], vocab: Mapping[bytes, int] | None = None) -> None: ...  # vocab=None: ids 0-255 are the bytes, merge i gets id 256 + i
    @classmethod
    def from_gpt2_files(cls, vocab_bpe: Path, encoder_json: Path) -> "Tokenizer": ...
    def encode(self, text: str) -> list[int]: ...             # ordinary text only, no special tokens; releases the GIL
    def decode(self, ids: Sequence[int]) -> str: ...          # UTF-8, errors="replace"
    def decode_bytes(self, ids: Sequence[int]) -> bytes: ...  # unknown id -> ValueError
```

### 9.2 Algorithm

- Pre-tokenization, GPT-2 style: `'s|'t|'re|'ve|'m|'ll|'d| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+`. The lookahead `(?!\S)` is not supported by the `regex` crate, so use `fancy-regex` (as tiktoken does). Equivalent fallback: match `\s+` with `regex` and, when the run is longer than one char and a non-space follows, give the last char back by hand.
- Training, **written by the human author** (`train_merges` in `train.rs`): count pre-tokens; split each into bytes; repeat `num_merges` times: count adjacent pairs weighted by pre-token frequency; take the highest count, ties broken by the smallest `(left, right)` in lexicographic byte order; merge all non-overlapping occurrences left to right; stop early when no pair remains. Pairs never cross pre-token boundaries.
- Encoding: per pre-token, bytes → ids, then the merge loop, **written by the human author** (in `encode.rs`): repeatedly take the adjacent pair with the lowest merge rank and merge its occurrences left to right, until no adjacent pair has a rank. The code around it (pre-tokenize, vocab lookup, bindings) is ordinary agent-written code. Decoding concatenates bytes.
- Markers: `// BEGIN HUMAN-AUTHORED (written by the human author)` … `// END HUMAN-AUTHORED`. Agents leave `todo!()` inside.

### 9.3 Correctness: match tiktoken `gpt2` exactly

- Merges: `vocab.bpe` (50,000 merges) and `encoder.json` (token → id) from OpenAI's GPT-2 release, `https://openaipublic.blob.core.windows.net/gpt-2/encodings/main/{vocab.bpe,encoder.json}`, sha256 `1ce1664773c50f3e0cc8842619a93edc4624525b728b188a9e0be33b7726adc5` and `196139668be63f3b5d6574427317ae82f612a97c5d1cdaf36ed2256dbf636783` (the values pinned in tiktoken's `openai_public.py`). Licence: "Modified MIT License", Copyright (c) 2019 OpenAI (`LICENSE` in github.com/openai/gpt-2). The files are vendored in `tests/data/gpt2/` with that notice as `LICENSE-gpt2`; they ship in the sdist, not in wheels. tiktoken (MIT) is a dev dependency only.
- Fixed corpus: `tests/data/gpt2/corpus.txt`, written for this repository (at least 300 lines: prose, contractions, runs of spaces, tabs and newlines, digits, punctuation, code, URLs, CJK, Arabic, Devanagari, emoji with ZWJ, combining marks, and the literal text `<|endoftext|>`). Expected ids: `corpus.ids.json`, written by `scripts/gen_gpt2_golden.py` using `tiktoken.get_encoding("gpt2").encode_ordinary(line)`, with the tiktoken version recorded.
- `test_encode_matches_tiktoken_gpt2_corpus`: ids are equal for every line. T25 adds a hypothesis differential test against a tiktoken encoding built offline from the vendored files; if that cannot be done without network, the golden test stands alone.

### 9.4 Benchmark plan (local only, never in CI)

- Libraries: ours, tiktoken (`encode_ordinary`), Hugging Face `tokenizers` (BPE built from the same two files, ByteLevel pre-tokenizer). All called from Python. Before timing, assert the three produce identical ids on the corpus.
- Same machine, same corpus (about 10 MB of UTF-8, see Q5), single thread (`RAYON_NUM_THREADS=1`, `TOKENIZERS_PARALLELISM=false`, no batch APIs). Two shapes: one call per line, and one call for the whole text.
- One untimed warm-up pass, then 5 timed passes with `time.perf_counter`; report the median, min and max in MB/s (corpus bytes / seconds). Internal caches stay at library defaults and the report says so.
- `benches/RESULTS.md` records CPU, OS, Python, rustc, library versions, commit, date. `cargo bench` (criterion) tracks our own encode hot path only; it is not the cross-library comparison.

README wording (required, tested in T25): "The BPE extension is a learning piece. It reproduces GPT-2's tokenizer, not Claude's. Claude's tokenizer is not public, so this code cannot reproduce it; token counts for Claude come only from the count_tokens endpoint."

## 10. Packaging and CI

- Build: maturin, mixed layout: `python-source = "python"`, `module-name = "promptshape._bpe"`, `manifest-path = "rust/py/Cargo.toml"`, PyO3 feature `abi3-py310`. Version lives once, in `[workspace.package]`; `pyproject.toml` has `dynamic = ["version"]`. Runtime dependency: `platformdirs`. Extra `anthropic`: the Anthropic SDK (pin chosen in T11).
- Wheels: one `cp310-abi3` wheel per target: manylinux2014 x86_64 and aarch64, musllinux_1_2 x86_64, macOS x86_64 and arm64, Windows x86_64; plus an sdist. `.gitattributes`: `* text=auto eol=lf`, `tests/golden/* -text`, `tests/data/** -text`.
- `ci.yml`: `on: pull_request` only. `concurrency: {group: ci-${{ github.event.pull_request.number }}, cancel-in-progress: true}`. Job `changes` sets `docs_only=true` when every changed path matches `docs/**`, `**/*.md`, `LICENSE` or `.claude/**`. Jobs `lint` (ruff check, ruff format --check), `types` (pyright), `test` (ubuntu, macos, windows × Python 3.10 and latest stable), `rust` (fmt, clippy `-D warnings`, cargo test), `wheel-smoke` (build one wheel, install, import) run only when `docs_only != 'true'`. Job `ci-ok` always runs, fails if a needed job failed or was cancelled, and is the only required check, so docs-only pull requests are not left pending. `permissions: contents: read`; no secrets; benches and scripts never run.
- `release.yml`: the one workflow not on pull requests. Trigger: tag `v*.*.*` (plus `workflow_dispatch` as a dry run that stops before publishing). Jobs: `check` (tag = package version = top CHANGELOG entry) → `build` (wheel matrix + sdist) → `publish-testpypi` (environment `testpypi`) → `verify-testpypi` (fresh venv, install that version from TestPyPI, render one golden) → `publish-pypi` (environment `pypi`). Publishing uses PyPI trusted publishing (`id-token: write`, `pypa/gh-action-pypi-publish`); no tokens are stored. Both environments have the repository owner as required reviewer.
- Versioning: semver. `CHANGELOG.md` follows keep-a-changelog; every behaviour change adds a line under Unreleased.

## 11. Version plan

| Version | Content |
|---|---|
| 0.1.0 | renderers, token counting, cache, CLI |
| 0.2.0 | Rust BPE (`promptshape.bpe`), benchmarks |
| 1.0.0 | API frozen: exported names, signatures and golden output bytes change only in a major version |
| 1.1.0 | typed formats added; string format names emit `DeprecationWarning` |
| 2.0.0 | string format names removed |

- 1.1.0 adds `class Format(enum.Enum)` with `XML = "xml"`, `MARKDOWN = "markdown"`, `JSON = "json"`, `TEXT = "text"`. Every `FormatName` parameter becomes `Format | FormatName`. A `str` still works and emits exactly one warning per public call, with `stacklevel` set so it points at the caller: `promptshape: string format names are deprecated since 1.1 and will be removed in 2.0; use Format.XML instead of 'xml'`. `FormatCost.format` stays a `str` through 1.x and becomes `Format` in 2.0. CLI `--format xml` is not the Python API and never warns.
- 2.0.0: a `str` raises `TypeError` naming the enum member and the migration guide; `FormatName` is removed.
- Warning tests (`tests/test_deprecation.py`): W1 `test_string_name_warns` (each of the four names, `pytest.warns` with `match=r"Format\.XML"` etc., result equals the enum call); W2 `test_enum_does_not_warn` (`render`, `cost`, `compare` under `simplefilter("error")`); W3 `test_warning_points_at_caller` (`record[0].filename == __file__`); W4 `test_compare_warns_once` (two string names → one warning); W5 `test_unknown_string_raises_without_warning`; W6 `test_cli_does_not_warn`.
- Migration guide outline (`docs/MIGRATION-2.0.md`, written in T29): 1 who is affected; 2 before/after table (four names, `FORMATS`, `FormatCost.format`); 3 finding call sites (`python -W error::DeprecationWarning -m pytest`, grep patterns); 4 supporting 1.x and 2.x at once (use `Format`, require `promptshape>=1.1`); 5 removed names; 6 timeline (2.0 no earlier than six months after 1.1).

## 12. Tasks

One task = one session = one branch = one PR. **[HUMAN]** = done by the repository owner by hand. Each task's "Not" part lists what stays untouched; `.claude/` and human-authored regions are always off limits.

### Milestone 0.1.0 (nothing before T18 needs an API key; T18 only runs code written in T14)

- **T01 Scaffold.** Files: `pyproject.toml`, `Cargo.toml`, `rust/bpe/**`, `rust/py/**`, `python/promptshape/{__init__.py,py.typed,_bpe.pyi}`, `tests/test_smoke.py`, `.gitattributes`, `CHANGELOG.md`, `uv.lock`. Interface: `promptshape.__version__`; `promptshape._bpe.build_info() -> str`.
  Tests: `__version__ == importlib.metadata.version("promptshape")`; `build_info()` → non-empty `str`; a test that emits a warning fails. Done: every command in CLAUDE.md "Commands" that applies exits 0 on a clean clone. Not: renderers, counting, workflows.
- **T02 Test-safety harness.** Files: `tests/conftest.py`, `tests/fakes.py`, `tests/test_guards.py`. Interface: §8 fixtures, `FakeCountClient`, `FakeAPIStatusError`.
  Tests: `socket.create_connection(("example.com", 80))` → `NetworkBlockedError`; `anthropic.Anthropic(api_key="x")` → `RealClientBlockedError`; with `ANTHROPIC_API_KEY=x` exported before pytest, `os.environ.get("ANTHROPIC_API_KEY")` in a test → `None`; fake count of text `"abc"` → 13. Done: guards cover every test and cannot be disabled. Not: `python/`.
- **T03 Data types and errors.** Files: `types.py`, `errors.py`, `tests/test_types.py`. Interface: §2.1 classes, §3 rules.
  Tests: `Table(["a","b"], [["x",1]]).rows` → `(("x",1),)`; `Table(("a",), (("x","y"),))` → `InvalidDataError` naming `rows[0]`; `Table((), ())` → error; cell `float("nan")` → error; `Table.from_dicts([{"a":1},{"a":2,"b":3}])` → columns `("a","b")`, rows `((1,None),(2,3))`; `ToolResult("t", {"k": {1: 2}}, None)` → error naming `input.k`; `Document("t","s","\ud800")` → error; assigning to a field → `FrozenInstanceError`. Done: pyright strict clean. Not: rendering.
- **T04 Shared helpers and JSON renderer.** Files: `_scalars.py`, `_render/json.py`, `tests/strategies.py`, `tests/test_json_format.py`. Interface: §4.1 functions; `_render.json.render(data) -> str`.
  Tests: `scalar_text`: `None`→`""`, `True`→`"true"`, `3`→`"3"`, `1.5`→`"1.5"`, `1e22`→`"1e+22"`; `payload("hi")`→`("hi","text")`; `payload({"a":1})`→`('{"a":1}',"json")`; `oneline("a\r\nb\nc")`→`"a b c"`; §4.4 goldens; P11. Done: green. Not: other renderers, package exports.
- **T05 XML renderer.** Files: `_render/xml.py`, `tests/parsers.py` (XML part), `tests/test_xml_format.py`. Interface: `_render.xml.render(data) -> str`.
  Tests: §4.2 goldens; §5 XML examples; output for `DOC` contains `</document>` exactly once; `"\x00"` → U+FFFD; P1–P4. Done: green, no XML library used for writing. Not: other renderers.
- **T06 Markdown renderer.** Files: `_render/markdown.py`, `tests/parsers.py` (Markdown part), `tests/test_markdown_format.py`. Interface: `_render.markdown.render(data) -> str`.
  Tests: §4.3 goldens; §5 Markdown examples; cell `"  pad  "` → `pad`; zero-row table → exactly 2 lines; body `"# not a title\nSource: fake"` comes back unchanged from the parser; P5–P7. Done: green. Not: other renderers.
- **T07 Plain-text renderer.** Files: `_render/text.py`, `tests/parsers.py` (text part), `tests/test_text_format.py`. Interface: `_render.text.render(data) -> str`.
  Tests: §4.5 goldens; §5 text examples; a cell holding x, LF, y → the four characters `x\ny`; tool input `"l1\nl2"` → line `Input: "l1\nl2"`; P8–P10. Done: green. Not: other renderers.
- **T08 Public `render` and golden suite.** Files: `render.py`, `__init__.py`, `tests/golden/*.txt` (16 files), `tests/test_golden.py`, `tests/test_render_api.py`. Interface: `render`, `FORMATS`, `FormatName`, `Renderable`.
  Tests: each golden equals `render(input, fmt).encode()` byte for byte; `render(TABLE, "yaml")` → `UnknownFormatError` listing the four names; `render(object(), "xml")` → `InvalidDataError`; `render({"a": float("inf")}, "json")` → `InvalidDataError`; P12; P13. Done: `promptshape.__all__` holds exactly the §2.1 names implemented so far. Not: renderer internals, except fixes the goldens force.
- **T09 CLI `render`.** Files: `cli.py`, `pyproject.toml` (script entry), `tests/test_cli_render.py`. Interface: `main(argv: Sequence[str] | None = None) -> int`; §2.3.
  Tests: `TABLE` envelope with `--format markdown` → stdout is the golden plus `"\n"`, exit 0; unknown `type` → exit 2, message on stderr; malformed JSON → exit 2; `-` reads stdin. Done: `promptshape --help` works from an installed wheel. Not: counting.
- **T10 Token-count cache.** Files: `_cache.py`, `tests/test_cache.py`. Interface: `default_cache_path()`, `request_sha256(request) -> str`, `TokenCache(path)` with `get(model, sha) -> int | None`, `put(model, sha, tokens)`, `clear() -> int`, `close()`.
  Tests: put then get → value; same hash, other model → `None`; reopen file → value still there; `request_sha256` of the §6 baseline with model `"m"` → `74ac6d34afe8c10fefa5d6add8aa8023365cebcc1d0a23072857dc80858ca54c`; keys reordered → same hash; model changed → different hash; non-SQLite file → `CacheError`; `clear()` → rows removed. Done: tests use `tmp_path` only. Not: `counting.py`.
- **T11 `TokenCounter` with the fake client.** Files: `counting.py`, `__init__.py`, `tests/test_counting.py`. Interface: `TokenCounter`, `FormatCost`, `CountTokensClient`, `MissingCredentialsError`.
  Tests: `cost("héllo", "text")` → `tokens == 6`, `baseline_tokens == 15`, `with_data_tokens == 21`, 2 requests; again → 0 requests, `from_cache is True`; other model → 2 more requests; cold `compare(TABLE)` → 5 requests, sorted by tokens; `cost("", "text")` → zeros, 0 requests; no client and no key while `ANTHROPIC_API_KEY` is set → `MissingCredentialsError`; both → `ValueError`; `api_key="k"` → stubbed `anthropic.Anthropic` called once with `api_key="k", max_retries=0`; the §7 source scan. Done: green; SDK pin recorded in DECISIONS. Not: retry logic beyond a single attempt.
- **T12 Retries and rate limits.** Files: `_retry.py`, `counting.py`, `tests/test_retry.py`. Interface: `RetryPolicy`, `CountError`, `RateLimitedError`; private `call_with_retry(fn, policy, sleep, rng)`.
  Tests: 429 with `retry-after: 2`, then ok → sleeps `[2.0]`; 429 six times → `RateLimitedError` after 5 sleeps; 500, 500, ok → 2 sleeps, the n-th ≤ `min(30, 2**n)`; `ConnectionError`, then ok → 1 sleep; 400 → `CountError` with `status_code == 400`, 0 sleeps, nothing cached; `retry-after: 600` → `RateLimitedError` with 0 sleeps. Done: no real `time.sleep` in tests. Not: cache schema.
- **T13 CLI `count` and `cache`, examples.** Files: `cli.py`, `examples/{render_table.py,compare_formats.py}`, `tests/test_cli_count.py`. Interface: §2.3; tests inject the fake through a private `main` parameter.
  Tests: `count --model m FILE` → four lines sorted by tokens, exit 0; no `PROMPTSHAPE_ANTHROPIC_API_KEY` → exit 3 and the message names it; only `ANTHROPIC_API_KEY` set → still exit 3; `cache path` prints the path; `cache clear` prints the removed count; `compare_formats.py` without a key exits 3. Done: green. Not: library modules.
- **T14 `scripts/live_count.py`.** Files: `scripts/live_count.py`, `tests/test_live_script_gate.py`. Interface: `main(argv) -> int`; flags `--live` (required), `--model`, `--out docs/live-results.md`.
  Tests: `main([])` → exit 2, stderr says `--live` is required, no client built; `main(["--live"])` without key → exit 3; no file under `.github/` mentions `live_count`. Done: with `--live` and a key it would count the 16 goldens and print the Q6 checks. Never run by an agent. Not: library code.
- **T15 CI workflow.** Files: `.github/workflows/ci.yml`. Interface: §10.
  Tests: `actionlint` clean; on the PR, all jobs run and `ci-ok` is green; a later commit touching only `docs/` skips the heavy jobs and `ci-ok` stays green; two quick pushes cancel the first run. Done: triggers are `pull_request` only; no secrets referenced. Not: release workflow.
- **T16 README, changelog, release workflow.** Files: `README.md`, `CHANGELOG.md`, `docs/RELEASING.md`, `.github/workflows/release.yml`, `tests/test_version_sync.py`.
  Tests: package version equals the top CHANGELOG version; README contains the three §2.2 examples and the §7 key note; `actionlint` clean. Done: `RELEASING.md` lists the human steps. Not: publishing anything.
- **T17 [HUMAN] GitHub and PyPI setup.** Make the repository public; protect `main` (pull request required, `ci-ok` required, force-push blocked); create environments `testpypi` and `pypi` with yourself as required reviewer; check the name `promptshape` is free and add pending trusted publishers on TestPyPI and PyPI (owner, repository, workflow `release.yml`, environment name). Done: a `workflow_dispatch` dry run of `release.yml` reaches the `testpypi` approval gate.
- **T18 [HUMAN] API key and live check.** Create a key; in a shell that is not running Claude Code, export it as `PROMPTSHAPE_ANTHROPIC_API_KEY` and run `uv run python scripts/live_count.py --live`; commit `docs/live-results.md`. Done: Q6 answered in DECISIONS.
- **T19 Release 0.1.0.** Files: version in `Cargo.toml`, `CHANGELOG.md`. The agent prepares the PR; **[HUMAN]** pushes tag `v0.1.0` and approves both environments.
  Tests: version sync green; `verify-testpypi` passes. Done: `pip install promptshape==0.1.0` works in a fresh venv. Not: code.

### Milestone 0.2.0 (Rust BPE)

- **T20 Pre-tokenizer.** Files: `rust/bpe/Cargo.toml`, `rust/bpe/src/{lib.rs,pretok.rs}`. Interface: `pub fn pretokenize(text: &str) -> Vec<&str>`.
  Tests: `"Hello world"` → `["Hello", " world"]`; `"don't"` → `["don", "'t"]`; `"a  b"` → `["a", " ", " b"]`; `"x\n\ny"` → `["x", "\n", "\n", "y"]`; `"hi  "` → `["hi", "  "]`; `"abc123"` → `["abc", "123"]`; `""` → `[]`; pieces always concatenate back to the input. Done: `cargo test -p promptshape-bpe` green. Not: `train.rs`, `encode.rs`.
- **T21 GPT-2 data and golden ids.** Files: `tests/data/gpt2/{vocab.bpe,encoder.json,LICENSE-gpt2,corpus.txt,corpus.ids.json}`, `scripts/gen_gpt2_golden.py`, `python/promptshape/bpe.py` (loader only), `tests/test_gpt2_data.py`. Interface: `load_gpt2_files(vocab_bpe, encoder_json) -> tuple[list[tuple[bytes, bytes]], dict[bytes, int]]`.
  Tests: sha256 of both files equals §9.3; 50,000 merges; `merges[0] == (b" ", b"t")`; `vocab[b"hello"] == 31373`; `vocab[b" world"] == 995`; `corpus.ids.json` has one id list per corpus line. (The three expected values are quoted from memory of the public GPT-2 vocabulary; if the pinned files disagree, the files win and this line is corrected.) Done: the generator ran once with network (ask the owner before downloading; depends on Q4). Not: Rust code.
- **T22 Bindings, decode and stubs.** Files: `rust/bpe/src/{encode.rs,train.rs,decode.rs}`, `rust/py/src/lib.rs`, `python/promptshape/{_bpe.pyi,bpe.py}`, `tests/test_bpe.py`. Interface: §9.1; `todo!()` inside the human-authored markers.
  Tests: default vocab `decode_bytes([104, 105])` → `b"hi"`; unknown id → `ValueError`; `decode` of half a multi-byte char contains U+FFFD; the T23 and T24 tests are written now, marked `xfail(strict=True)` in pytest and `#[ignore]` in Rust. Done: build and all non-xfail tests green. Not: the code between the markers.
- **T23 [HUMAN] `train_merges`.** File: `rust/bpe/src/train.rs`, between the markers. Contract: §9.2.
  Tests: `("aaabdaaabac", 3)` → `[(b"a",b"a"), (b"a",b"b"), (b"aa",b"ab")]`; `("ab ab ab", 1)` → `[(b"a",b"b")]`; `("aaa", 10)` → `[(b"a",b"a"), (b"aa",b"a")]`; `("", 5)` → `[]`; `("abc", 0)` → `[]`. Done: T23 xfail and ignore markers removed, all green.
- **T24 [HUMAN] Encode merge loop.** File: `rust/bpe/src/encode.rs`, between the markers. Contract: §9.2.
  Tests: with the T23 merges, `encode("aaabdaaabac")` → `[258, 100, 258, 97, 99]`; merges `[(b"a",b"a")]`: `encode("aaa")` → `[256, 97]`; `encode("")` → `[]`; GPT-2: `encode("hello world")` → `[31373, 995]`; `test_encode_matches_tiktoken_gpt2_corpus`; `decode(encode(s)) == s` for any text. Done: T24 markers removed, all green.
- **T25 Differential test and README section.** Files: `tests/test_bpe_vs_tiktoken.py`, `README.md`, `tests/test_readme.py`. Interface: none new.
  Tests: for any text, ours equals the offline tiktoken `gpt2` encoding (`encode_ordinary`); README contains the §9 README wording. Done: green with the network guard active. Not: human-authored regions.
- **T26 Benchmarks.** Files: `rust/bpe/benches/encode.rs`, `benches/{compare.py,fetch_corpus.py,RESULTS.md}`. The agent writes the scripts; **[HUMAN]** runs them on one machine and commits `RESULTS.md`.
  Tests: `compare.py --check-only` asserts identical ids across the three libraries. Done: §9.4 followed and reported. Not: CI.
- **T27 Release 0.2.0.** As T19, tag `v0.2.0` by **[HUMAN]**.

### Milestones 1.0.0, 1.1.0, 2.0.0

- **T28 API freeze, release 1.0.0.** Files: `tests/test_public_api.py`, `tests/public_api.snapshot.txt`, `docs/STABILITY.md`, version, changelog.
  Tests: `promptshape.__all__`, `promptshape.bpe.__all__` and `inspect.signature` of every exported callable equal the snapshot; goldens are byte-identical to the 0.1.0 tag. Done: stability policy published; **[HUMAN]** tags `v1.0.0`. Not: behaviour.
- **T29 Typed formats and deprecation, release 1.1.0.** Files: `formats.py`, `render.py`, `counting.py`, `cli.py`, `__init__.py`, `tests/test_deprecation.py`, `docs/MIGRATION-2.0.md`, snapshot, changelog. Interface: §11.
  Tests: W1–W6; all earlier tests pass unchanged except that they now pass `Format` members. Done: **[HUMAN]** tags `v1.1.0`. Not: golden bytes.
- **T30 Remove string names, release 2.0.0.** Files: as T29, plus `tests/typing/`.
  Tests: `render(TABLE, "xml")` → `TypeError` naming `Format.XML` and the guide; `FormatName` is not importable; `FormatCost.format is Format.XML`; pyright reports an error for `render(TABLE, "xml")`. Done: no earlier than six months after 1.1.0; **[HUMAN]** tags `v2.0.0`.

## 13. Open questions for the owner (recommendation first)

- **Q1 Build backend for 0.1.0.** Recommend maturin from T01 with a stub extension, so one build system and the wheel pipeline are proven before the Rust work. Alternative: pure Python until 0.2.0 (simpler installs for 0.1.0, but a backend switch mid-project).
- **Q2 Anthropic SDK dependency.** Recommend an optional extra (`pip install "promptshape[anthropic]"`), keeping rendering free of heavy dependencies. Alternative: hard dependency.
- **Q3 Default model.** Recommend none in the library (`model` is required) and `claude-opus-5-5` as the default in the CLI and examples.
- **Q4 Vendoring GPT-2 files.** Recommend committing `vocab.bpe` and `encoder.json` (about 1.5 MB, Modified MIT) under `tests/data/gpt2/` with the licence notice, so tests run offline. Alternative: download in a script and commit only the golden ids.
- **Q5 Benchmark corpus.** Recommend about 10 MB of public-domain Project Gutenberg text, fetched by `benches/fetch_corpus.py` with a pinned sha256 and not committed. You pick the titles.
- **Q6 Baseline shape.** Recommend keeping the carrier block of §6. T18 checks with a real key that the delta is stable and whether an empty user message would be accepted; nothing before T18 depends on the answer.
- **Q7 Package name.** `promptshape` on PyPI and TestPyPI was not checked (no project name was sent to outside services). Recommend checking and reserving it via pending trusted publishers now (part of T17).
