## Troubleshooting

Use the tables below to identify an error and find a suggested fix. Messages are quoted as they appear, with variable parts in `<angle brackets>`. Command-line errors appear in Terminal after `error: `, and in Python the message appears in the exception. Installation problems are covered in the [installation guide](installation.md), which ends with its own troubleshooting table.

### The command line

You may see one of these messages if the command is missing an argument or contains an invalid argument. Check the command against the suggested fix before running it again.

| What you see | What it means | What to do |
|---|---|---|
| `error: no table given (use --check to validate the program only).` | No table was named after the program. | Name table A and table B, or table A and `--dedup`. |
| `error: only one table given (<file>), so there is nothing to block it against. Add table B, or add --dedup to find the duplicates in table A.` | One table was named, without `--dedup`. | Add table B, or add `--dedup` to find the duplicates in table A. |
| `error: --dedup searches a single table against itself, but two tables were given (<file A> and <file B>).` | `--dedup` was given with two tables. | Leave out table B to find the duplicates in table A, or leave out `--dedup` to block the two tables. |
| `error: no such program file: <path>` | The program argument is neither an existing file path nor valid JSON text. | Check the path of the program file. A relative path starts from the current directory, so run the tutorial's commands from the `delex_lite` directory. |
| `delex-lite: error: argument --format: invalid choice: '<name>' (choose from parquet, csv, tsv, jsonl)` | The requested output format is not supported. | Choose `parquet`, `csv`, `tsv` or `jsonl`. |
| `dedup=True finds the duplicates in a single table, but two tables were given; leave out table_b, or leave out dedup` | From Python, `block` was given two tables and `dedup=True`. | Leave out `table_b` to find the duplicates in table A, or leave out `dedup`. |

### The program

If a program cannot be read or validated, you may see one of these messages. On the command line, they follow `error: invalid program spec: `. In Python, they may appear as exceptions from `program` or `from_json`.

| What you see | What it means | What to do |
|---|---|---|
| `blocking program must contain at least one keep rule` | The program has no keep rule. | Add a keep rule with a source, such as a `bm25_topk`. |
| `keep rule must contain at least one source predicate` | A keep rule has only filters. The lines that follow the message list the predicates that can be a source. | Add a source to the rule, such as `bm25(...).topk(k)` or a positive `exact(...)`. [How sources and filters are chosen](technical-guide.md#how-sources-and-filters-are-chosen) lists them. |
| `drop rules may contain only filter predicates; <predicate> cannot be a filter` | A drop rule holds a top-K predicate. | Remove the top-K predicate from the drop rule. |
| `rule needs a non-empty 'predicates' list: <rule>` | A rule in the JSON program has an empty or missing predicate list. | Give each rule at least one predicate. |
| `unknown predicate type '<type>'` | The predicate type is not supported. See [The predicates](technical-guide.md#the-predicates). | Use one of those names, such as `bm25_topk`. |
| `predicate missing 'type': <predicate>` | A predicate of the JSON has no `type`. | Add the `type`. |
| `predicate missing column key '<key>'; give 'col' or both 'index_col' and 'search_col': <predicate>` | A predicate in the JSON program does not specify its input columns. | Add `col`, or `index_col` and `search_col`. |
| `program spec must be a JSON object` | The JSON value is not an object describing a program. | Write the program as an object with `keep_rules`, as [The JSON form](technical-guide.md#the-json-form) shows. |
| `Expecting value: line <n> column <m> (char <c>)`, or another message of the JSON reader | The JSON text has a syntax error at the position given. | Correct the JSON, for example a missing quote, comma or bracket. |
| `<kind>: threshold must be in [0, 1], got <value>` | The similarity threshold is outside the supported range of 0 to 1. | Use a threshold from 0 to 1. |
| `<kind>: unknown comparison op '<op>'; one of ['>=', '>', '==', '!=', '<=', '<']` | The JSON predicate specifies an unsupported comparison operator. | Use one of the comparisons listed. |
| `<kind>: k must be >= 1, got <k>` | K is below 1. | Use a K from 1 to 128. |
| `jaro_winkler: prefix_weight must be in [0, 0.25], got <value>` | The `prefix_weight` of a Jaro-Winkler predicate is outside 0 to 0.25. | Use a value from 0 to 0.25. |
| `smith_waterman: gap_cost must be finite and >= 0, got <value>` | The Smith-Waterman gap cost is negative or not finite. | Use a finite value greater than or equal to zero. |
| `NumericDistance needs at least one of abs_tol / rel_tol (>= 0)` | The numeric comparison has no tolerance. Supply `abs` (`abs_tol` in JSON), `pct` (`rel_tol` in JSON), or both. | Give one or both. |
| `unknown mode '<mode>'; one of ['or', 'and']` | The `mode` of a `near` predicate is not one of the two. | Use `"or"` or `"and"`. |
| `unknown ref '<ref>'; one of ['max', 'min', 'mean', 'a', 'b']` | The `ref` of a `near` predicate is not one of these. | Use one of the names listed. |
| `unknown tokenizer '<name>' — did you mean '<name>'?. Valid: alnum, num, standard, stripped_whitespace, whitespace, word, qgram:N, stripped_qgram:N` | From Python, a predicate names an unknown tokenizer. | Use one of the tokenizers listed. |
| `unknown tokenizer spec: '<name>'` | The JSON program specifies an unsupported tokenizer. The run stops when it reaches the predicate. | Use one of the tokenizers of [Tokenizers](technical-guide.md#tokenizers), such as `qgram:3`. |

If a predicate is incomplete or uses an unsupported operation, Python may report a `TypeError` with one of these messages:

| What you see | What it means | What to do |
|---|---|---|
| ``keep rule entry <metric>(...) is incomplete — add `.topk(k)` or a threshold (e.g. `>= 0.7`). ...`` | A predicate has neither a top-K nor a comparison. | Finish it with `.topk(k)` or a comparison, such as `>= 0.7`. |
| `` `<metric>(...)` is incomplete — add `.topk(k)` or a threshold (e.g. `>= 0.7`). If you wrote `… & <metric>(...) >= 0.7`, parenthesize the threshold: ... `` | A comparison joined with `&` lacks parentheses, since `&` is applied before `>=`. | Put the comparison in parentheses, as in `& (D.cosine("authors", "authors") >= 0.5)`. |
| ``bm25() has no threshold form; use `.topk(k)` (BM25 returns the k best, not a similarity cutoff)`` | A `bm25(...)` was given a comparison. | Use `.topk(k)`. |
| ``<metric>() has no top-K source; use a threshold like `<metric>(...) >= 0.7` `` | A `.topk(k)` was given to a predicate that has none, such as `overlap`. | Use a comparison. |
| `both columns are required: metric(index_col, search_col[, tokenizer]). ...` | The comparison specifies only one input column. | Supply the column name from each input table, even if the names are identical. |

### Reading the tables

If `delex_lite` cannot read or validate an input table, gold set, or candidate set, you may see one of these messages.

| What you see | What it means | What to do |
|---|---|---|
| `no such file: <path>` | The specified file does not exist. | Check the path. A relative path starts from the current directory, so run the tutorial's commands, and start Jupyter, from the `delex_lite` directory. |
| `unrecognized extension '<ext>' for <file>. Supported: .parquet/.pq, .csv/.txt, .tsv, .json/.jsonl/.ndjson (or pass a pandas/Polars DataFrame or pyarrow Table).` | The file extension is not supported. | Convert the file to one of these formats, for example a spreadsheet to CSV. |
| `unsupported source type '<type>'; pass a file path, pandas/Polars DataFrame, or pyarrow Table` | From Python, a table was given as another kind of object. | Pass a path, a DataFrame or a pyarrow Table. |
| `no id column '<name>' in columns [...]. Name the id column with id_col=NAME (CLI: --id-col NAME), or pass id_col='_rid' (CLI: --id-col _rid) to key rows by row number 0..n-1.` | The table has no column of that name, `_id` by default. | Name the ID column with `--id-col`, or number the rows with `--id-col _rid`. |
| `<table>: id column '<name>' (<how>) has <count> out of <total> rows. Every row needs a unique, non-null id. ...` | Some records have missing IDs. `<count>` is the number of empty IDs, as in `1 null value`. | Fill the IDs, name a column of unique values with `--id-col`, or number the rows with `--id-col _rid`. |
| `<table>: id column '<name>' (<how>) has duplicated values: <n> distinct out of <total> rows (e.g. <value> ×<count>). Every row needs a unique, non-null id. ...` | Two or more rows have the same ID. The message gives up to three repeated values. | Fix the IDs, name a column of unique values with `--id-col`, or number the rows with `--id-col _rid`. |
| `<what> has ids that are not whole numbers, such as '<value>'. Ids are whole numbers, 0 or more. To number the rows of a table keyed by strings, and its gold set, see 'Tables keyed by strings' in the technical guide.` | The IDs of `<what>`, which is `table A`, `table B` or `the gold set`, include a value that is not a whole number, such as a product code. | Use `--id-col _rid` for the tables, or convert the tables and the gold set, as the section [Tables keyed by strings](technical-guide.md#tables-keyed-by-strings) of the technical guide describes. |
| `<what> has negative ids, such as <id>. Ids are whole numbers, 0 or more.` | Some IDs of `<what>` are below 0. | Renumber the IDs from 0, and map the gold set to the new IDs. |
| `<what> has missing ids (<n> of <total>)` | Some IDs of `<what>` are empty, as when a gold set converted from codes holds a code that is not in its table. | Remove the pairs with an empty ID, or correct their codes and convert the gold set again. |
| `<what> has ids above 9,223,372,036,854,775,807, such as '<value>'` | Some IDs of `<what>` are too large to be stored as 64-bit integers. | Renumber the IDs from 0, as the section [Tables keyed by strings](technical-guide.md#tables-keyed-by-strings) of the technical guide describes. |
| `the program names the column '<name>', which this table does not have; its columns are [...]` | A predicate names a column that the table does not have. | Correct the column name in the program. The names are case-sensitive. |
| `the text to block is <n> bytes, above the 4 GB limit per table side; split the table or block on fewer columns` | The text of one column of one table is 4 GB or more. | Block the table in parts. A run in parts gives slightly different BM25 scores, because each part of table A has its own statistics. |
| `gold must have at least two id columns, found [...]` | The gold set has fewer than two columns. | Give a gold set with the IDs from table A in one column and the IDs from table B in the other. |
| `candidates have no column '<name>'; candidate tables have the columns id1 and id2` | The candidate set given to `evaluate` has no columns `id1` and `id2`. | Give a file that `--out` or `out=` wrote, or a table with these two columns. |
| `<path>: unknown extension '<ext>'; candidate files are .parquet, .csv, .tsv or .jsonl` | The candidate file given to `evaluate` is not in a format of [Output files](technical-guide.md#output-files). | Give a file that `--out` or `out=` wrote. |

### The row limit

| What you see | What it means | What to do |
|---|---|---|
| `the <side> table has <n> rows, above the limit of 3,000,000 rows per side.` | A table has more than 3,000,000 rows, and the run raised `LimitError`. `<side>` is `reference (A)` for table A and `search (B)` for table B. | Block a sample or a part of the table. For larger tables, contact [our team](mailto:hello@relixa.ai). |
| `k=<k> is above the limit of 128 for this build.` | A top-K predicate has a K above 128, and the run raised `LimitError`. | Use a K from 1 to 128. |

### Saving the candidates

If you save candidates using `--out` or `out=`, you may encounter one of these messages. Invalid output formats are detected before blocking begins, and file or disk problems when the output is written.

| What you see | What it means | What to do |
|---|---|---|
| `<path>: unknown extension '<ext>'; candidates are written as .parquet, .csv, .tsv or .jsonl, or with format= one of [...]` | The extension of the output file names no format. | Use one of the four extensions, or name the format with `--format`. |
| `<path>: format='<format>' does not match the extension '<ext>'; name the file for the format, or leave out format=` | The extension and `--format` name different formats. | Rename the file, or leave out `--format`. |
| `format must be one of [...], got '<value>'` | From Python, `format` is not one of the four. | Use one of the names listed. |
| `output directory does not exist: <directory>` | The output folder does not exist. | Create the directory, or name a file in a directory that exists. |
| `blocking failed (<step>); check that the output folder can be written and that the disk has room` | A step of the run could not complete, most often because the disk is full. | Free some disk space, or check that the directory can be written, and run it again. |

### Starting `delex_lite`

If `delex_lite` cannot start, check whether the message you see is listed below.

| What you see | What it means | What to do |
|---|---|---|
| `delex_lite requires macOS on Apple Silicon (arm64); this is <system>/<machine>.` | The machine is not a Mac with Apple silicon. | Run `delex_lite` on a Mac with Apple silicon. |
| `delex_lite could not start on this machine; it needs an Apple Silicon Mac (M1 or newer) with macOS 11 or later` | The Mac does not meet the requirements. | Update macOS to version 11 or later, on a Mac with Apple silicon. |
| `delex_lite is not installed completely: <path> is missing. Run 'bash install.sh' again in the delex_lite folder.` | A file of the installation is missing. | Run `bash install.sh` again from the `delex_lite` directory. |
| `macOS did not allow delex_lite's library to load:` | macOS refused to load a file of `delex_lite`. This happens when an installed `delex_lite` directory is copied from another Mac. | Run `bash install.sh` again from the `delex_lite` directory, which installs fresh copies of the files. |

### Python and the notebook

| What you see | What it means | What to do |
|---|---|---|
| `AttributeError: module 'delex_lite' has no attribute '<name>'` | The requested attribute is not available in `delex_lite`. When a public name is close to it, the message ends with ` — did you mean '<name>'?`. | Use the name suggested. The [technical guide](technical-guide.md) lists every public name. |
| `zsh: command not found: jupyter` | The environment is not active, or Jupyter is not installed. | Run `source .venv/bin/activate`, then `pip install notebook`. |
| `ModuleNotFoundError: No module named 'delex_lite'` | The notebook is using a different Python environment from the installed `.venv`. | Stop Jupyter with Ctrl-C, run `source .venv/bin/activate`, and start Jupyter again with `jupyter notebook`. |
| `ModuleNotFoundError: No module named 'pandas'` | Your code reads the candidates with pandas, which `delex_lite` does not install. | Run `pip install pandas` with the environment active, or read the file with `pyarrow`, which is installed, as [`block()`](technical-guide.md#block) shows. |
