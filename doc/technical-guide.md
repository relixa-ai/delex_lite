## `delex_lite` technical guide

This guide documents the `delex-lite` command, blocking programs, comparison functions, supported data formats and limits. The [tutorial](tutorial.md) shows them in use, and [How `delex_lite` works](how-it-works.md) explains the concepts behind them.

Contents:

- [Conventions and data shapes](#conventions-and-data-shapes)
- [The command line](#the-command-line)
- [Programs](#programs)
- [Reading and writing files](#reading-and-writing-files)
- [The Python functions](#the-python-functions)
- [Limits](#limits)

## Conventions and data shapes

The examples below use `import delex_lite as D`. Use `block` to run a program and `evaluate` to measure saved candidates. Use `program` and `from_json` to build or load programs. The comparison functions are `bm25`, `jaccard`, `cosine`, `overlap`, `exact`, `edit`, `jaro`, `jaro_winkler`, `smith_waterman` and `near`. The package also provides `LimitError`.

A run uses the following data structures.

| Data | Columns | Notes |
| --- | --- | --- |
| A table | an ID column, and the columns the program names | The ID column is `_id` unless `id_col` names another. Every row must have a unique, non-negative integer ID. |
| Candidate pairs | `id1`, `id2` | `id1` is the ID of a record from table A, and `id2` is the ID of a record from table B. |
| A gold set | two ID columns | Pairs known to match. Each row contains one record ID from table A and one from table B. See [gold-set column requirements](how-it-works.md#recall-precision-reduction-and-the-gold-set) for column names and ordering. |

In a deduplication, table A and table B are the same table, and each pair holds the smaller ID in `id1`.

## The command line

The installer makes `delex-lite` available in the `.venv` environment. `delex-lite --help` lists its options.

### `delex-lite`

```
delex-lite PROGRAM TABLE_A [TABLE_B] [options]
```

Runs the blocking rules on table A and table B, or compares table A with itself when `--dedup` is given. Top-K rules find candidates in table A for each record from table B. The command prints a summary and saves candidates when `--out` is given. It performs the same run as the Python function [`block()`](#block), and each option corresponds to an argument of `block`.

| Option | Python | Default | What it does |
|---|---|---|---|
| `PROGRAM` | `program` | required | The program, as a JSON file or as JSON text. See [The JSON form](#the-json-form). |
| `TABLE_A` | `table_a` | required | Table A, as a file. |
| `TABLE_B` | `table_b` | none | Table B, as a file. Left out with `--dedup`. |
| `--dedup` | `dedup=True` | off | Matches table A against itself, to find its duplicates. |
| `--gold FILE` | `gold` | none | The gold set. Adds the rows `\|GT\|`, `Found/GT`, `Recall` and `Precision` to the summary. |
| `--out FILE` | `out` | none | Saves candidates to this file. The extension selects the format. No extension selects Parquet. See [Output files](#output-files). |
| `--format NAME` | `format` | from the extension | `parquet`, `csv`, `tsv` or `jsonl`. Specifies the output format explicitly when the extension does not identify it. |
| `--id-col NAME` | `id_col` | `_id` | The ID column. Use `_rid` to assign row numbers starting at zero. |
| `--quiet` | `progress=False` | off | Hides the progress line. |
| `--check` | | off | Checks the program, prints its plan, and exits without blocking. No table is needed. |

The run summary reports the measures explained in [Part 1](tutorial.md#part-1-running-a-program-on-two-tables) and [Part 2](tutorial.md#part-2-measuring-recall-and-precision-against-a-gold-set) of the tutorial. During a run, a progress line shows the number of records processed and the estimated time remaining.

The command exits with status 0 when the run succeeds. If the input tables, program or options are invalid, it prints the message after `error: ` and exits with status 2. [Troubleshooting](troubleshooting.md) lists the messages.

**Example:**

```bash
delex-lite programs/papers.json data/dblp_acm/table_a.parquet data/dblp_acm/table_b.parquet \
    --gold data/dblp_acm/gold.parquet --out candidates.parquet
```

### `delex-lite smoke`

```
delex-lite smoke
```

Checks the installation by running a search on two records. It prints `OK — delex_lite runs on this machine.` and exits with status 0, or prints the error and exits with status 1. The installer runs it as its last step.

## Programs

A program describes which pairs to select and exclude. A *keep rule* selects candidates. An optional *drop rule* removes selected pairs. Each condition within a rule is a *predicate*, and a pair must satisfy all the conditions in that rule. [How blocking programs work](how-it-works.md#blocking-programs) explains these concepts with examples. You can write a program in Python or JSON and convert between the two forms.

### Writing a program in Python

A predicate is written as a function that names the two columns, followed by a top-K or a comparison. The examples below use hypothetical product tables with `name`, `description`, `model` and `price` columns. They illustrate the syntax and do not refer to a supplied dataset:

```python
D.bm25("name", "name").topk(10)                 # find up to 10 similar product names
D.cosine("description", "description") >= 0.5    # require similar descriptions
D.jaccard("name", "name", "qgram:2") >= 0.4      # compare names using 2-character sequences
~D.exact("model", "model")                      # check whether model numbers differ
D.near("price", "price", abs=20, pct=0.1)        # prices within 20 units or 10%
```

The first argument names a column from table A. The second names a column from table B. Supply both arguments even when the column names are identical. `&` joins predicates into a rule. A comparison inside a rule needs parentheses, since `&` is applied before `>=`, as in `D.bm25("name", "name").topk(10) & (D.cosine("description", "description") >= 0.5)`.

Comparisons support `>=`, `>`, `==`, `!=`, `<=` and `<`. The corresponding methods are `.ge`, `.gt`, `.eq`, `.ne`, `.le` and `.lt`, each taking a threshold argument. A threshold is a number from 0 to 1.

### The predicates

`column_a` and `column_b` stand for column names from table A and table B. Pass them as strings, such as `"name"`. `k` is the maximum candidates per record from table B, and `t` is a similarity threshold from 0 to 1. These are placeholders in the examples below, not additional options.

| Python | JSON `type` | Role | Parameters, with defaults |
|---|---|---|---|
| `bm25(column_a, column_b, tokenizer="word").topk(k=10)` | `bm25_topk` | source | `k` from 1 to 128 |
| `jaccard(column_a, column_b, tokenizer="qgram:3").topk(k=10, floor=0.0)` | `jaccard_topk` | source | `k` from 1 to 128, `floor` from 0 to 1 |
| `cosine(column_a, column_b, tokenizer="qgram:3").topk(k=10, floor=0.0)` | `cosine_topk` | source | as `jaccard_topk` |
| `exact(column_a, column_b, lower=False)` | `exact_match` | source and filter | `lower=True` compares the values in lower case |
| `~exact(column_a, column_b)`, or `exact(column_a, column_b, invert=True)` | `exact_match` with `"invert": true` | filter | |
| `jaccard(column_a, column_b, tokenizer="qgram:3") >= t` | `jaccard` | filter, or source with `>=` and `>` | in JSON, `op` (`">="`) and `val` (0.7) |
| `cosine(column_a, column_b, tokenizer="qgram:3") >= t` | `cosine` | filter, or source with `>=` and `>` | in JSON, `op` (`">="`) and `val` (0.7) |
| `overlap(column_a, column_b, tokenizer="qgram:3") >= t` | `overlap_coeff` | filter | in JSON, `op` (`">="`) and `val` (0.7) |
| `edit(column_a, column_b) >= t` | `edit_distance` | filter | in JSON, `op` (`">="`) and `val` (0.8) |
| `jaro(column_a, column_b) >= t` | `jaro` | filter | in JSON, `op` (`">="`) and `val` (0.8) |
| `jaro_winkler(column_a, column_b, prefix_weight=0.1) >= t` | `jaro_winkler` | filter | `prefix_weight` from 0 to 0.25 |
| `smith_waterman(column_a, column_b, gap_cost=1.0) >= t` | `smith_waterman` | filter | `gap_cost` of 0 or more |
| `near(column_a, column_b, abs=None, pct=None, mode="or", ref="max")` | `numeric_distance` | filter | at least one of `abs` and `pct`, each 0 or more |

See [The predicates](how-it-works.md#the-predicates) for an explanation of each comparison. The following details apply to their parameters and behavior.

- *BM25.* BM25 uses fixed parameters `k1 = 1.2` and `b = 0.75`. Use `.topk(k)` with `bm25(...)`. Threshold comparisons and `floor` are not supported.
- *Set similarity.* `jaccard`, `cosine` and `overlap` compare the sets of tokens of the two values. For a top-K predicate, `floor` sets the minimum accepted similarity.
- *Exact match.* The values are compared as text. If either value is missing or empty, neither `exact` nor `~exact` is satisfied.
- *String similarity.* `edit` is 1 − (number of edits) / (length of the longer value), `jaro` and `jaro_winkler` are the Jaro and Jaro-Winkler similarities, with `prefix_weight` as the weight of a shared start, and `smith_waterman` is the Smith-Waterman score divided by the length of the longer value, with `gap_cost` as the cost of a gap. The values are compared as written, without tokens.
- *Numeric nearness.* `near` keeps a pair whose values are within `abs` of each other, or within `pct` as a fraction, such as 0.1 for 10%. When both tolerances are set, `mode="or"` accepts either and `mode="and"` requires both. `ref` selects the reference value used to calculate the relative tolerance: `"max"`, `"min"` or `"mean"` of the two values, `"a"` for the value from table A, or `"b"` for the value from table B. Numeric values can be extracted from text such as `$19.99`. In JSON, `abs` is `abs_tol` and `pct` is `rel_tol`.

A missing value never satisfies an `exact`, a `~exact`, a string similarity or a `near`.

### Tokenizers

BM25 and the set similarities take one of the tokenizers `word`, `alnum`, `whitespace`, `num`, `qgram:N` and `stripped_qgram:N`, where N is a whole number of 1 or more, such as `qgram:3`. `stripped_whitespace` is another name for `word`, and `standard` for `alnum`. See [Tokens](how-it-works.md#tokens) for definitions and examples.

### How sources and filters are chosen

| Predicate | Source | Filter | In a drop rule |
|---|:---:|:---:|:---:|
| `bm25_topk`, `jaccard_topk`, `cosine_topk` | yes | no | no |
| `exact_match` | yes | yes | yes |
| `exact_match` with `"invert": true` | no | yes | yes |
| `jaccard`, `cosine` with `>=` or `>` | yes, when the rule has no top-K predicate | yes | yes |
| `jaccard`, `cosine` with `==`, `!=`, `<=` or `<` | no | yes | yes |
| `overlap_coeff`, `edit_distance`, `jaro`, `jaro_winkler`, `smith_waterman`, `numeric_distance` | no | yes | yes |

In a keep rule, the top-K predicates are the sources, and the rule keeps the pairs found by all of them. Without a top-K predicate, the first eligible predicate becomes the source. All remaining predicates are filters, applied in the order written. Every keep rule requires a source. Drop rules contain filters only.

### The JSON form

A program in JSON is an object with the lists `keep_rules` and `drop_rules`. Each rule has a list of `predicates`, and each predicate specifies its `type`, columns and parameters. This example uses the hypothetical product columns introduced above:

```json
{
  "keep_rules": [
    {"predicates": [
      {"type": "bm25_topk", "col": "name", "tokenizer": "word", "k": 10},
      {"type": "cosine", "col": "description", "tokenizer": "qgram:3", "op": ">=", "val": 0.5}
    ]}
  ],
  "drop_rules": [
    {"predicates": [{"type": "exact_match", "col": "model", "invert": true}]}
  ]
}
```

Use `col` when the column name is the same in both tables. Otherwise, use `index_col` for the column name in table A and `search_col` for the column name in table B. These are JSON field names. They do not ask you to perform a separate indexing step. Omitted parameters use the defaults listed under [The predicates](#the-predicates). `bm25_topk` also accepts `analyzer` as another name for `tokenizer`. `drop_rules` may be left out.

### `program()`

```python
def program(
    keep,                    # a predicate or rule, or a list of them
    drop=(),                 # the drop rules, in the same form
) -> Program
```

Builds and validates a blocking program from keep and drop rules.

**Parameters:**

- `keep`: the keep rules. Each is a predicate or predicates joined with `&`.
- `drop`: the drop rules, in the same form.

**Returns:** a program that can be passed to [`block()`](#block). Its `explain()` method returns the execution plan as text, corresponding to the command-line `--check` output. Its `to_json(as_string=False, indent=2)` method returns a `dict`. Set `as_string=True` to return JSON text.

**Raises:** `ValueError` when a keep rule has no source, or a drop rule includes a predicate that cannot act as a filter. `TypeError` when a predicate is incomplete, such as `D.cosine("description", "description")` without a comparison.

**Example:**

```python
prog = D.program(
    keep=[D.bm25("name", "name").topk(10) & (D.cosine("description", "description") >= 0.5)],
    drop=[~D.exact("model", "model")],
)
print(prog.explain())
```

This builds a program for the hypothetical product tables. It does not read any data. It finds similar names, filters by description similarity, and excludes pairs with different model numbers.

### `from_json()`

```python
def from_json(
    source,                  # a JSON file, JSON text, or a dict
) -> Program
```

Loads and validates a program from a JSON file, JSON string or `dict`.

**Returns:** a validated program with the methods described under [`program()`](#program).

**Raises:** `ValueError` when the JSON does not describe a valid program. See [Troubleshooting](troubleshooting.md#the-program). `FileNotFoundError` when the specified file does not exist.

**Example:**

```python
prog = D.from_json("programs/papers.json")
```

## Reading and writing files

### File formats

Input tables, gold sets and saved candidates can be supplied as files or supported Python objects. File extensions determine how files are read.

| Extension | Format | Notes |
|---|---|---|
| `.parquet`, `.pq` | Parquet | Read with the types it has. |
| `.csv`, `.txt` | CSV, separated by commas | The first line names the columns. |
| `.tsv` | TSV, separated by tabs | The first line names the columns. |
| `.json`, `.jsonl`, `.ndjson` | JSON Lines, or a JSON array of objects | An object containing the array under `data`, `records`, `rows` or `items` is also read. |

In Python, input tables may be pandas or Polars DataFrames, pyarrow Tables or RecordBatches. Gold sets may be pandas or Polars DataFrames, pyarrow Tables, or sets of `(id1, id2)` pairs, where each tuple contains one record ID from each input table.

BM25, set similarity, exact-match and string-similarity comparisons convert column values to text before comparing them.

### What your files need

- *An ID column.* Each table needs a unique ID for every record. `delex_lite` reads the IDs from `_id` by default, and `--id-col` (or `id_col` in Python) names another ID column. The ID column must be specified explicitly if it is not `_id`. IDs are checked before blocking begins. A table without such a column can use `--id-col _rid`, which numbers the rows 0, 1, 2 and so on. These numbers are row positions, so they identify the records only while the row order stays the same. IDs must be non-negative integers, such as 0, 1 or 2. See the next section if your tables use string keys.
- *The columns the program names.* Each predicate names a column of table A and a column of table B, and each must exist in its table. The names are case-sensitive.
- *A gold set (optional).* Required only to measure recall and precision, with `--gold` (`gold`). Each row contains the IDs of two records known to match, one from each table. See [gold-set column requirements](how-it-works.md#recall-precision-reduction-and-the-gold-set) for supported column names and ordering.

### Tables keyed by strings

`delex_lite` does not support string IDs. Use non-negative integer IDs instead.

To block a table keyed by strings, such as product codes, use `--id-col _rid`, which numbers the rows 0, 1, 2 and so on. The column of codes is read only if a predicate of the program names it:

```bash
delex-lite program.json table_a.csv table_b.csv --id-col _rid --out candidates.parquet
```

The resulting candidates contain row numbers. A matcher or gold set must use those same IDs to refer to the records. For a reusable mapping, convert the files once and use the converted files for subsequent runs.

The example below assumes a `code` column in each input table and `code_a` and `code_b` columns in the gold set. It adds an integer `_id` column to each table and replaces the gold-set codes with their corresponding IDs. Run it in your Python environment with pandas installed, using your own file paths and column names:

```python
import pandas as pd

table_a = pd.read_csv('table_a.csv')
table_b = pd.read_csv('table_b.csv')
gold = pd.read_csv('gold.csv')
table_a.insert(0, '_id', range(len(table_a)))
table_b.insert(0, '_id', range(len(table_b)))
id_a = dict(zip(table_a['code'], table_a['_id']))
id_b = dict(zip(table_b['code'], table_b['_id']))
gold = pd.DataFrame({'id1': gold['code_a'].map(id_a), 'id2': gold['code_b'].map(id_b)})
table_a.to_parquet('table_a.parquet', index=False)
table_b.to_parquet('table_b.parquet', index=False)
gold.to_parquet('gold.parquet', index=False)
```

If a gold-set code is absent from its input table, the mapping produces a missing ID. Correct that mapping before blocking. Otherwise, validation reports `the gold set has missing ids`.

Run your program on the converted files:

```bash
delex-lite program.json table_a.parquet table_b.parquet --gold gold.parquet --out candidates.parquet
```

For deduplication, create IDs for the single input table and apply that same mapping to both columns of the gold set.

### Output files

Use `--out` on the command line or `out` in Python to save candidate pairs. The file extension selects the format. Use `--format` or `format` to specify it explicitly when the extension does not identify a format.

| Format | Extension | Content |
|---|---|---|
| `parquet`, the default | `.parquet`, `.pq` or none | One row per pair, with `id1` and `id2` as 64-bit integers. |
| `csv` | `.csv` | One row per pair, after a header line. |
| `tsv` | `.tsv` | The same as `csv`, separated by tabs. |
| `jsonl` | `.jsonl` | One JSON object per pair, with the keys `id1` and `id2`. |

All output formats follow these rules:

- An unsupported extension or a conflict between the extension and explicit format, such as `--out x.csv --format parquet`, is rejected before blocking begins.
- Output is written to a temporary file named `<file>.partial-<n>` in the destination folder. The file is renamed to the requested name only after writing completes.
- For deduplication, self-pairs are excluded and each remaining pair appears once, with the smaller ID in `id1`.

## The Python functions

### `block()`

```python
def block(
    program,                 # a program, a dict, JSON text, or a JSON file
    table_a,                 # table A, as a file, a DataFrame or a pyarrow Table
    table_b=None,            # table B; None for a deduplication of table A
    *,
    gold=None,               # the gold set, to measure recall and precision
    out=None,                # a file to save the candidates to
    format=None,             # the format of out, when its extension names none
    id_col="_id",            # the id column of both tables
    dedup=False,             # True to find the duplicates in table A
    verbose=True,            # print the summary
    progress=True,           # show the progress line
) -> dict
```

Runs a blocking program and returns the run summary. Like the `delex-lite` command, it can save candidates and measure recall and precision against a gold set. By default, it also prints the summary.

**Parameters:**

- `program`: a program from [`program()`](#program) or [`from_json()`](#from_json), or its JSON form as a `dict`, as text, or as the path of a JSON file.
- `table_a`, `table_b`: the input tables (see [File formats](#file-formats)). With `table_b` left out, or with `dedup=True`, the run is a deduplication of `table_a`.
- `gold`: known matching pairs (see [File formats](#file-formats)).
- `out`: the destination for saved candidates (see [Output files](#output-files)).
- `format`: the output format when the file extension does not identify one.
- `id_col`: the ID column of both tables, `_id` by default, or `_rid` to number the rows.
- `dedup`: `True` finds the duplicates in `table_a`.
- `verbose`: `True` prints the summary when the run ends.
- `progress`: `False` hides the progress line.

This function:

1. Checks the program, and, when `out` is given, determines its format.
2. Reads record IDs and validates the row limit and ID requirements.
3. Executes the sources and filters for each keep rule, combines their results, and applies the drop rules. For deduplication, it also removes self-pairs and duplicate pair orderings. See [Blocking programs](how-it-works.md#blocking-programs) for details.
4. Saves the candidates to `out`, when it is given.
5. Measures recall and precision against `gold`, when it is given.
6. Prints the summary when `verbose` is `True`.

**Returns:** the summary, as a `dict` with the following keys.

| Key | What it holds |
|---|---|
| `n_candidates` | The number of candidate pairs. In a deduplication, the number of distinct pairs. |
| `reduction` | The share of the possible pairs left out, as a fraction, such as 0.999603. |
| `seconds` | The time from reading the tables to having the candidates. |
| `funnel` | The steps of the run, as a list of `(step, pairs, note)`, as the `Funnel` rows show them. |
| `n_a`, `n_b` | The number of rows of table A and of table B. In a deduplication both are the rows of the one table. |
| `dedup` | `True` for a deduplication. |
| `program` | The path of the program file, or `"(inline)"`. |
| `keep_rules`, `drop_rules` | The numbers of keep rules and drop rules. |
| `dataset` | The name of the directory that holds table A, or `None` for a table in memory. |
| `out` | The file the candidates were saved to, or `None`. |
| `n_gold`, `tp`, `fn`, `recall`, `precision` | Only with a gold set. The number of pairs in the gold set, the number of them among the candidates, the number missing, `tp / n_gold` and `tp / n_candidates`, as fractions. |

**Raises:** `ValueError` for an invalid program, file or ID, as [Troubleshooting](troubleshooting.md) lists. `TypeError` for an argument of the wrong kind. `FileNotFoundError` when a file does not exist. `LimitError` when a table has more than 3,000,000 rows or K is above 128.

**Example:**

```python
prog = D.from_json("programs/papers.json")
summary = D.block(prog, "data/dblp_acm/table_a.parquet", "data/dblp_acm/table_b.parquet",
                  gold="data/dblp_acm/gold.parquet", out="candidates.parquet", verbose=False)
print(summary["n_candidates"], round(summary["recall"], 4))   # 2382 0.9807
```

To inspect candidates in Python, read the saved file with `pyarrow`, which is installed with `delex_lite`:

```python
import pyarrow.parquet as pq

pairs = pq.read_table("candidates.parquet")
print(pairs.num_rows, pairs.column_names)   # 2382 ['id1', 'id2']
```

### `evaluate()`

```python
def evaluate(
    candidates,              # the candidate set, as a file or a table
    gold,                    # the gold set
    dedup=False,             # True for the candidates of a deduplication
) -> dict
```

Measures recall and precision for previously saved candidates against known matching pairs. It performs the same evaluation as `--gold` without running blocking again.

**Parameters:**

- `candidates`: a candidate file in any of the formats of [Output files](#output-files), or a DataFrame or pyarrow Table with the columns `id1` and `id2`. A Parquet file grouped by `id2`, with the columns `id2` and `id1_list`, as other blockers write it, is read too.
- `gold`: known matching pairs in any form accepted by `block`.
- `dedup`: `True` when the candidates come from a deduplication. A gold pair then counts as found in either order, and a pair of a record with itself is never found.

**Returns:** a `dict` with the keys `n_candidates`, `n_gold`, `tp`, `fn`, `recall` and `precision`, with the same meanings as in the `block` summary.

**Raises:** `FileNotFoundError` when a file does not exist. `ValueError` when the candidates have no columns `id1` and `id2`, when an ID of the gold set is missing or is not a whole number, 0 or more, or when the gold set has fewer than two columns.

**Example:**

```python
report = D.evaluate("candidates.parquet", "data/dblp_acm/gold.parquet")
print(report["tp"], round(report["recall"], 4))   # 2181 0.9807
```

## Limits

### The row limit and `LimitError`

```python
class LimitError(RuntimeError)
```

`delex_lite` handles up to 3,000,000 rows in each table. `block` counts the rows after reading the IDs and before any other work, and raises `LimitError` when a table has more. The limit is fixed, and no setting changes it. For larger tables, contact [our team](mailto:hello@relixa.ai).

The following limits apply to each run.

| Limit | Value | When it is checked |
|---|---|---|
| Rows in each table | 3,000,000 | Before any work. A larger table raises `LimitError`. |
| K of a top-K predicate | from 1 to 128 | When the program is built, for a K below 1, which raises `ValueError`, and before any work, for a K above 128, which raises `LimitError`. |
| Text of one column of one table | under 4 GB | When the text is prepared, before the predicate runs. A larger text raises `ValueError`. |

**Example:**

```python
try:
    D.block(prog, "big_table.parquet", dedup=True)
except D.LimitError as e:
    print(e)
```
