## How `delex_lite` works

This guide explains blocking programs, how rules select and exclude pairs, and how to assess the results. The [tutorial](tutorial.md) shows these ideas in use. The [technical guide](technical-guide.md) documents the commands and Python API. If you are familiar with entity matching, you can start with [The inputs](#the-inputs).

### The blocking step

We refer to the two input tables as table A and table B. The goal is to find pairs of records, one from each table, that refer to the same real-world entity. Such a pair is a *match*. Comparing every possible pair becomes expensive as the tables grow: two tables of one million rows each produce one trillion pairs. Entity matching therefore typically separates candidate selection from match decisions.

*Blocking* selects a smaller set of plausible matches. Its output is the *candidate set*. `delex_lite` performs this step.

*Matching* examines the candidate pairs and decides whether each refers to the same entity. A matcher, such as Iris, performs this step.

Two measures help assess a blocker. *Recall* is the proportion of true matches included in the candidates. A matcher cannot recover a match that blocking excludes, so preserving recall is important when choosing rules. *Reduction* is the proportion of all possible pairs excluded by blocking. It indicates how much work remains for the matcher. Given known matching pairs, `delex_lite` also reports *precision*, the proportion of candidates found in that reference set.

### The inputs

Each record needs a unique ID: a non-negative integer, such as 0, 1 or 2. IDs must be present on every row. By default, `delex_lite` reads them from `_id`. Use `id_col` in Python or `--id-col` on the command line to select another column. These requirements are checked before blocking begins.

If a table has no ID column, use `id_col="_rid"` or `--id-col _rid` to number its rows from zero. The row order must remain unchanged for these IDs to continue identifying the same records. For string keys such as product codes, follow [Tables keyed by strings](technical-guide.md#tables-keyed-by-strings).

Each comparison specifies a column from table A and a column from table B. The names may be the same, such as `name` in both tables, or different, such as `product_name` and `name`. Only columns referenced by the program are read for comparison.

Input tables can be files or supported Python objects. See [File formats](technical-guide.md#file-formats) for the available formats and types.

### Blocking programs

A *blocking program* describes which pairs to select and which to exclude. For example, when comparing products from two stores, you might select pairs with similar names, require similar descriptions, and exclude pairs with different model numbers. This helps separate products such as headphones with similar names but different models.

A rule that selects candidates is a *keep rule*. It contains one or more conditions, called *predicates*, joined with AND: a pair must satisfy every condition in that rule. A program can have several keep rules. A pair selected by any of them is included, and a pair selected by more than one is included only once.

A *drop rule* excludes selected pairs that satisfy all of its conditions. Drop rules apply to the combined results of the keep rules. For example, a drop rule can exclude two products with different model numbers. Drop rules are optional. Use this condition only if model numbers are reliable in both tables. Otherwise, a typo or a different way of writing the same model number could exclude a true match.

A condition that finds candidate pairs is a *source*. In a keep rule with one source, the remaining conditions are *filters*: they check the pairs already found and keep only those that pass. In the product example, name similarity finds the pairs and description similarity filters them. [Sources and filters](#sources-and-filters) explains how these roles are chosen, including rules with multiple top-K sources.

The candidate set combines the keep rules and applies the exclusions:

```text
candidate set = (pairs selected by any keep rule) minus (pairs selected by any drop rule)
```

You can build a program in Python using `program` and the comparison functions, or describe it in a JSON file for the `delex-lite` command. The Python and JSON forms represent the same rules and can be converted between formats. Before a run, `delex_lite` checks the program. Use `--check` on the command line, or `explain()` in Python, to see which condition finds pairs first, which conditions filter those pairs, and which rules exclude pairs. The [tutorial](tutorial.md#part-4-adding-a-filter) shows the command and its output.

### The funnel

The summary of a run shows a *funnel*: the number of pairs after each step. It helps you see how much each condition narrows down the candidates.

For example, a program comparing two product catalogs might narrow its candidates as follows. These counts are illustrative:

| Step | What happens | Pairs remaining |
|---|---|---:|
| Find similar product names | Select 1,000 possible matches | 1,000 |
| Require similar descriptions | Remove 700 pairs | 300 |
| Exclude different model numbers | Remove another 50 pairs | 250 |

The final 250 pairs are the candidates passed to a matcher. The counts alone do not tell you whether the excluded pairs contained true matches. Use a gold set to measure that when one is available.

In the command's output, each keep rule has a row for its source and each filter. When several keep rules are combined, a union row reports their combined pairs, with duplicates counted once. A drop-rule row reports how many pairs it removes. The final row gives the candidate count. [Parts 4 to 6 of the tutorial](tutorial.md#part-4-adding-a-filter) show actual output and how to compare programs.

### Sources and filters

The role of each predicate depends on the other conditions in its keep rule:

1. A *top-K predicate*, which is `bm25(...).topk(k)`, `jaccard(...).topk(k)` or `cosine(...).topk(k)`, is always a source. If a rule contains several top-K predicates, each runs separately and only pairs found by every source are retained.
2. When a rule has no top-K predicate, the first predicate that can produce pairs on its own is the source. Eligible predicates are `exact(...)` without negation, and `jaccard(...)` or `cosine(...)` using `>=` or `>`.
3. Every other predicate is a filter.

Every keep rule needs a source. Drop rules contain filters only, so they cannot include a top-K predicate. For example, `jaccard(...) >= t` acts as a filter when the rule also contains `bm25(...).topk(k)`. To use the Jaccard comparison as a source, omit top-K predicates and place it before any other eligible source in that rule. See [How sources and filters are chosen](technical-guide.md#how-sources-and-filters-are-chosen) for the full reference.

### The predicates

In the examples below, `column_a` and `column_b` are column names from table A and table B, respectively. `k` is the maximum number of candidates per record from table B, and `t` is a similarity threshold from 0 to 1.

| Predicate | What it compares | Role |
|---|---|---|
| `bm25(column_a, column_b).topk(k)` | The BM25 score of the text of `column_a` from table A and `column_b` from table B | Source |
| `jaccard(column_a, column_b).topk(k)`, `cosine(column_a, column_b).topk(k)` | The Jaccard or cosine similarity of the sets of tokens of the two columns, keeping the K most similar records | Source |
| `jaccard(column_a, column_b) >= t`, `cosine(column_a, column_b) >= t` | The Jaccard or cosine similarity of the sets of tokens, against a threshold from 0 to 1 | Filter, or source as described above |
| `overlap(column_a, column_b) >= t` | The overlap coefficient of the sets of tokens, against a threshold from 0 to 1 | Filter |
| `exact(column_a, column_b)` | Whether the two values are equal. `~exact(column_a, column_b)` keeps the pairs whose values differ. | Source and filter, or filter for `~exact` |
| `edit(column_a, column_b)`, `jaro(column_a, column_b)`, `jaro_winkler(column_a, column_b)`, `smith_waterman(column_a, column_b)` | The similarity of the two values as strings of characters, from 0 to 1, against a threshold | Filter |
| `near(column_a, column_b, abs=…, pct=…)` | Whether two numbers are within an absolute or a relative tolerance | Filter |

Set similarities compare the distinct tokens in the two values. Jaccard divides the number of shared tokens by the number of distinct tokens across both sets. Cosine divides the shared-token count by the square root of the product of the set sizes. The overlap coefficient divides it by the size of the smaller set.

String similarities compare the values directly rather than splitting them into tokens. Edit similarity is calculated as 1 − (number of edits) / (length of the longer value). If either value is missing, string similarity, `exact`, `~exact` and `near` conditions are not satisfied. Such a condition excludes the pair when used as a keep-rule filter. In a drop rule, it leaves the pair unchanged.

`near` compares numeric values, including numbers extracted from text such as `$19.99`. Set `abs` for an absolute tolerance, such as 20 price units, or `pct` for a relative tolerance, such as 0.1 for 10%. When both are supplied, `mode="or"` accepts either tolerance and `mode="and"` requires both.

### Tokens

BM25 and the set similarities split values into *tokens*, the units used for comparison. Each predicate accepts a tokenizer that determines how the text is split. The default is `word` for BM25 and `qgram:3` for the set similarities.

| Tokenizer | Tokens | Example |
|---|---|---|
| `word` | Each sequence of consecutive letters and digits | `Data-Mining, 2nd ed.` gives `data`, `mining`, `2nd`, `ed` |
| `alnum` | The same tokens as `word` | |
| `whitespace` | Each sequence of characters between spaces, punctuation included | `Data-Mining, 2nd` gives `data-mining,`, `2nd` |
| `num` | Each sequence of consecutive digits | `ISBN 0-262-03384` gives `0`, `262`, `03384` |
| `qgram:N` | Each sequence of N consecutive characters of the whole value, spaces and punctuation included | `qgram:3` on `ab-c` gives `ab-`, `b-c` |
| `stripped_qgram:N` | Each sequence of N consecutive characters, after removing every character other than letters, digits and `_` | `stripped_qgram:3` on `ab-c` gives `abc` |

All tokenizers except `num` convert the upper-case letters A to Z to lower case. A letter here is one of the letters A to Z, so an accented letter such as é separates `word` tokens and is not part of any. `stripped_whitespace` is another name for `word`, and `standard` for `alnum`.

Q-grams can preserve shared fragments when words are misspelled, abbreviated or joined together. The set similarities use `qgram:3` by default for this reason.

### The BM25 score

`bm25` scores a pair with the BM25 variation of TF/IDF. Each token in the value from table B that also occurs in the value from table A adds to the score of the pair. A shared token contributes more when:

- it is rare in the compared column of table A (the *IDF* component)
- it occurs more often in the value from table A, with diminishing benefit from repetition (the *TF* component)
- the value from table A is shorter than the column average, measured in tokens

BM25 uses fixed parameters: `k1 = 1.2` controls the diminishing contribution of repeated tokens, and `b = 0.75` controls the adjustment for text length.

Token rarity and average length are calculated from the compared column of table A, or from the single table in a deduplication run. Scores can therefore change when the input table changes, including when you use a sample.

### The top K

A top-K predicate selects, for each record from table B, up to K records from table A with the highest scores. For example, with K = 10, a product from the second store can receive up to 10 possible matches from the first store. K is at most 128.

If table B contained 100 products, a BM25 rule with K = 10 could return at most 100 × 10 = 1,000 pairs. A pair must share at least one token in the compared values. If only three product names in table A share tokens with a particular name in table B, that record receives only three candidates, even though K is 10.

`jaccard(...).topk(k)` and `cosine(...).topk(k)` rank candidates by set similarity. Their `floor` parameter sets the minimum accepted similarity and defaults to 0.

K sets the balance between recall and reduction. A smaller K gives fewer pairs and less work for the matcher, but it can leave out a true match. For example, if the same product from the other store ranks 12th by name similarity, K = 10 leaves it out. K = 20 includes it among the candidates from that source, though later filters or drop rules may still exclude it.

### Deduplication

*Deduplication* finds possible duplicate records within one table. Use `--dedup` on the command line, or `dedup=True` in Python. In Python, supplying only table A also selects this mode. The same table fills both input roles, with three adjustments:

1. Pairs containing the same record twice are removed.
2. A pair that is found from both of its records, as (x, y) and as (y, x), is kept once. Each pair is written with the smaller ID as `id1`.
3. For a table containing **n records**, there are n × (n − 1) / 2 distinct pairs. The reduction is measured against this number, rather than n × n.

The funnel reports these removals in a step labeled `collapse  mirrors + self-pairs`. To use deduplication, pass one table and `--dedup` on the command line, or `dedup=True` to Python's [`block()`](technical-guide.md#block).

Giving the same table as both table A and table B is a run on two tables, not a deduplication. It keeps the pairs of a record with itself and both orders of each pair, so a matcher would see each pair twice. Use `--dedup` instead.

### Recall, precision, reduction and the gold set

A *gold set* is a set of pairs known to be matches, each with the ID of a record from table A and the ID of a record from table B. Given a gold set (`gold`, `--gold`), `delex_lite` counts how many of its pairs are in the candidate set and reports the following:

- `|GT|`, the number of pairs in the gold set
- `Found/GT`, the number of them in the candidate set, out of `|GT|`
- `Recall`, which is `Found/GT` as a percentage
- `Precision`, the number found divided by the number of candidates, as a percentage

In a deduplication, a gold pair counts as found in either order. A gold pair referencing IDs absent from the input tables still counts toward `|GT|`, but cannot be found among the candidates. `delex_lite` reads the two ID columns of the gold set by name, as `id1` and `id2`, `a_id` and `b_id`, `ltable_id` and `rtable_id`, `left_id` and `right_id`, or `lid` and `rid`. Otherwise it uses the first two columns: the first contains record IDs from table A, and the second contains record IDs from table B.

`Reduction` is measured from the candidate count and the table sizes alone. It is 1 − candidates / ((number of rows in table A) × (number of rows in table B)), the share of all possible pairs between table A and table B that the candidate set leaves out, and in a deduplication 1 − candidates / (n × (n − 1) / 2).

Recall and precision describe performance against the supplied gold set. If it contains only some true matches, recall covers only those known pairs, and reported precision may understate the proportion of candidates that actually match.

`evaluate` measures the same recall and precision for a candidate set that was saved earlier, without blocking again. Given `dedup=True`, it counts a gold pair as found in either order, as a deduplication does.

### The candidate set

To save the candidate set to a file, pass the file name with `--out` on the command line, as in `--out candidates.parquet`, or with the argument `out` of `block` in Python, as in `out="candidates.parquet"`. The output contains one row per pair with the columns `id1` and `id2`. Its extension selects Parquet, CSV, TSV or JSON Lines. Parquet is the default. See [Output files](technical-guide.md#output-files) for details.

### The row limit

`delex_lite` handles up to 3,000,000 rows in each table. A run with more rows in either table is refused before any work, with a `LimitError`. The text of one column of one table must also be under 4 GB. For larger tables, contact [our team](mailto:hello@relixa.ai).

### What determines the result

If recall is lower than expected, check the following:

1. *The gold set.* Check that the gold set uses the same record IDs as the input tables and follows the supported column naming or ordering. Check whether the known matches cover the cases you want to evaluate.
2. *The filters and the drop rules.* The funnel shows how many pairs each one removes. Check whether the conditions exclude true matches because values are missing or written differently. Missing values fail a keep-rule filter, but they do not satisfy a drop rule, so their effect depends on where the condition is used.
3. *K.* A larger K keeps more of the true matches. Measure recall at a few values of K, as the notebook of the tutorial does, before settling on one.
4. *The columns.* A column that identifies an entity, such as a name or a title, makes a good source. A column that is mostly empty, or that holds the same value in most records, adds noise. A second keep rule on another column may recover matches missed by the first, as Part 6 of the tutorial illustrates.
5. *The tokenizer.* On text with misspellings, abbreviations or words run together, a `qgram:3` tokenizer can find pairs that `word` misses.
