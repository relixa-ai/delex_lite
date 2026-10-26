## `delex_lite` tutorial

Follow Parts 1 to 6 in Terminal to run the included blocking programs, save candidate pairs, and compare the effect of adding conditions and combining rules. Part 7 shows how to write and run a program in a Python notebook. Work through the parts in order. Each builds on the previous examples.

### Before you begin

First, install `delex_lite` by following the [installation guide](installation.md). Wait for `Installation complete.` before continuing.

Open Terminal from **Applications → Utilities → Terminal**, or keep using the Terminal window from installation. Go to the downloaded `delex_lite` folder, which contains `install.sh`, `data` and `programs`. If you are opening a new Terminal window, type `cd` followed by a space, drag the `delex_lite` folder from Finder into Terminal, and press Return.

Activate the installed Python environment. Run this command in Terminal:

```bash
source .venv/bin/activate
```

This makes the `delex-lite` command available in this Terminal window. Your prompt should now start with `(.venv)`.

Run all Terminal commands below in this same window, from the `delex_lite` folder. Copy each command block into Terminal and press Return. Blocks labeled **Example output** show what to look for after running a command. If you open a new Terminal window later, return to the folder and activate the environment again.

### The sample data and the programs

The `data` folder contains two sample datasets.

| Dataset | What it is | Size |
|---|---|---|
| `data/dblp_acm` | Two tables of computer-science papers, DBLP (2,616 rows) and ACM (2,294 rows), with the columns `title`, `authors`, `venue` and `year`, and a gold set of the 2,224 pairs known to be the same paper. | 380 KB |
| `data/abt_buy` | Two tables of electronics products as two stores list them, Abt (1,081 rows) and Buy (1,092 rows), with the columns `name`, `description` and `price`, and a gold set of the 1,097 pairs known to be the same product. | 244 KB |

The `programs` directory holds the blocking programs used below as JSON files. These files already exist: you do not need to create or edit them for the Terminal walkthrough. The JSON excerpts explain the rules in each file. Part 7 shows how to write a program in Python.

The example outputs below were recorded on a Mac with an M5 Pro chip and 48 GB of memory. Run times may vary depending on your machine.

### Part 1: Running a program on two tables

Generate candidates for the ACM papers using the DBLP papers as possible matches. Run this command in Terminal:

```bash
delex-lite programs/title_bm25.json data/dblp_acm/table_a.parquet data/dblp_acm/table_b.parquet
```

**Example output:**

```text
Dataset           dblp_acm
|A|               2,616
|B|               2,294
Program           programs/title_bm25.json
Rules             1 keep − 0 drop
Funnel            keep[0]  bm25(title,k=10)  22,702
                  result                     22,702
Candidates        22,702
Reduction         99.6217%
Time              184 ms
```

The command reads DBLP as table A and ACM as table B. It runs the rules already saved in `programs/title_bm25.json`. A rule that selects possible matches is called a *keep rule*. This program has one keep rule: select papers with similar titles.

The file contains the following JSON. This is shown for reference. You do not need to paste it anywhere:

```json
{
  "keep_rules": [
    {
      "predicates": [
        {
          "col": "title",
          "type": "bm25_topk",
          "tokenizer": "word",
          "k": 10
        }
      ]
    }
  ],
  "drop_rules": []
}
```

The title comparison is a *predicate*, or a single condition within the rule. For each ACM record, it selects up to 10 DBLP records whose titles have the highest BM25 scores. Check the `Candidates` row in your output: this example produces 22,702 pairs. This command prints a summary. Part 3 adds a file output for the pairs.

The table below explains each row of the output.

| Row | What it shows |
|---|---|
| `Dataset` | The name of the directory that holds table A, as a label for the run. |
| `\|A\|` | The number of rows of table A. |
| `\|B\|` | The number of rows of table B. |
| `Program` | The program file, or `(inline)` for a program given as text or from Python. |
| `Rules` | The numbers of keep rules and drop rules of the program. |
| `Funnel` | The number of pairs after each step of the program. Parts 4 to 6 add steps. |
| `Candidates` | The number of pairs returned by the blocker. It is below \|B\| × 10 = 22,940, because two titles must share at least one word to form a pair. |
| `Reduction` | The share of all possible pairs between table A and table B that the blocker left out. |
| `Time` | The time from reading the tables to having the candidates. |

The two tables contain 2,616 × 2,294 = 6,001,104 possible pairs. Blocking reduces that to 22,702 candidates for a matcher to examine. In Part 2, you will check how many known matches are included.

### Part 2: Measuring recall and precision against a gold set

Check how many known matches are included in the candidates. Run the command again in Terminal, adding the included reference file with `--gold`:

```bash
delex-lite programs/title_bm25.json data/dblp_acm/table_a.parquet data/dblp_acm/table_b.parquet \
    --gold data/dblp_acm/gold.parquet
```

**Example output:**

```text
Dataset           dblp_acm
|A|               2,616
|B|               2,294
|GT|              2,224
Program           programs/title_bm25.json
Rules             1 keep − 0 drop
Funnel            keep[0]  bm25(title,k=10)  22,702
                  result                     22,702
Candidates        22,702
Reduction         99.6217%
Found/GT          2,219/2,224
Recall            99.78%
Precision         9.77%
Time              186 ms
```

The reference file is a *gold set*. It lists 2,224 pairs known to be the same paper. Adding `--gold` compares the candidates with these known matches. Look for `Found/GT`, `Recall` and `Precision` in your output.

The output has four new rows.

| Row | What it shows |
|---|---|
| `\|GT\|` | The number of pairs in the gold set. GT stands for ground truth. |
| `Found/GT` | The number of gold pairs among the candidates, out of `\|GT\|`. |
| `Recall` | `Found/GT` as a percentage. |
| `Precision` | The number of gold pairs among the candidates, divided by the number of candidates, as a percentage. |

2,219 of the 2,224 known matches are among the 22,702 candidates, so the recall is 99.78%. A matcher can examine only the candidates it receives, so check recall whenever you change the rules. The measured precision is 9.77%: about one in ten candidates is listed in this gold set. A matcher still needs to decide which candidates are matches. Parts 4 and 5 show how a filter and drop rule affect these results. [How `delex_lite` works](how-it-works.md#recall-precision-reduction-and-the-gold-set) describes how the gold set is read.

### Part 3: Saving the candidate pairs

Save the candidate pairs so you can inspect them or pass them to a matcher. Run this command in Terminal, adding `--out candidates.csv`:

```bash
delex-lite programs/title_bm25.json data/dblp_acm/table_a.parquet data/dblp_acm/table_b.parquet \
    --gold data/dblp_acm/gold.parquet --out candidates.csv
```

The `--out` option writes `candidates.csv` into your current `delex_lite` folder. Check that the summary ends with a `Saved` row:

**Example output:**

```text
...
Time              231 ms
Saved             candidates.csv (22,702 pairs)
```

Inspect the header and first three pairs. Run this command in the same Terminal window:

```bash
head -4 candidates.csv
```

**Example output:**

```text
id1,id2
2123,0
973,0
2131,0
```

Each row contains one candidate pair: `id1` identifies the record from table A and `id2` identifies the record from table B. In this preview, all three pairs refer to the record with ID 0 in table B.

You can pass this file to a matcher for the next step. The `.csv` extension selects CSV output. An extension of `.parquet`, or no extension, selects Parquet, the default format for Relixa's matchers. See [Output files](technical-guide.md#output-files) for the supported formats.

### Part 4: Adding a filter

Keep only the candidates whose authors are also similar. Use the supplied file `programs/title_authors.json`, which already includes the title rule from Part 1 and the author condition below. You do not need to edit the file. This excerpt shows the added condition:

```json
        {
          "col": "authors",
          "type": "cosine",
          "tokenizer": "qgram:3",
          "op": ">=",
          "val": 0.5
        }
```

First, validate the program and inspect its execution plan. Run this command in Terminal:

```bash
delex-lite programs/title_authors.json --check
```

**Example output:**

```text
program OK: 1 keep rule(s), 0 drop rule(s)
output = (⋃ keep) − (⋃ drop)
  keep[0]  source: bm25(title,k=10)   filters: cosine(authors)≥0.5
```

The `--check` option validates the program without reading the input tables or generating candidates. The plan identifies the title comparison as the rule's *source*, which finds the initial pairs. The author comparison is a *filter*: it checks those pairs and keeps only those with an author similarity of at least 0.5. Here, cosine similarity compares the author text as sets of 3-character sequences.

Run the supplied program on the paper tables in Terminal:

```bash
delex-lite programs/title_authors.json data/dblp_acm/table_a.parquet data/dblp_acm/table_b.parquet \
    --gold data/dblp_acm/gold.parquet
```

**Example output:**

```text
Funnel            keep[0]  bm25(title,k=10)       22,702
                  keep[0]  ∧ cosine(authors)≥0.5   3,072
                  result                           3,072
Candidates        3,072
Reduction         99.9488%
Found/GT          2,181/2,224
Recall            98.07%
Precision         71.00%
Time              200 ms
```

The funnel shows that the source found 22,702 pairs, as in Part 1, and that the filter kept 3,072 of them. Compare the `Recall` and `Precision` rows with Part 2. The filter removed 19,630 pairs, of which 38 were known matches, so the recall fell from 99.78% to 98.07% and the precision rose from 9.77% to 71.00%. Differences in how author lists are written can cause true matches to fail this filter. [How `delex_lite` works](how-it-works.md#sources-and-filters) describes how `delex_lite` decides which predicate is the source.

### Part 5: Adding a drop rule

Remove candidates whose publication years differ. A rule that excludes selected pairs is called a *drop rule*. Use the supplied file `programs/papers.json`, which already includes the conditions from Part 4 and the drop rule below. This excerpt is for reference, and no editing is needed:

```json
  "drop_rules": [
    {
      "predicates": [
        {
          "col": "year",
          "type": "exact_match",
          "invert": true,
          "lower": false
        }
      ]
    }
  ]
```

Setting `"invert": true` makes the `exact_match` condition select pairs with different `year` values. The drop rule removes those pairs from the candidates. Run this command in Terminal:

```bash
delex-lite programs/papers.json data/dblp_acm/table_a.parquet data/dblp_acm/table_b.parquet \
    --gold data/dblp_acm/gold.parquet
```

**Example output:**

```text
Rules             1 keep − 1 drop
Funnel            keep[0]  bm25(title,k=10)       22,702
                  keep[0]  ∧ cosine(authors)≥0.5   3,072
                  drop[0]  exact(year)≠              690  removed
                  result                           2,382
Candidates        2,382
Reduction         99.9603%
Found/GT          2,181/2,224
Recall            98.07%
Precision         91.56%
Time              197 ms
```

In this dataset, the drop rule removed 690 pairs, none of which was listed in the gold set. Recall stayed at 98.07%, and precision rose to 91.56%. On your own data, a year difference could be a data error, so excluding these pairs could remove true matches. A drop rule applies to the combined results of all keep rules.

### Part 6: Combining two keep rules

Combine two ways of finding candidates in the product data: similar names and similar descriptions. The program `programs/products.json` has two keep rules, BM25 on the product names with K = 10 and BM25 on the product descriptions with K = 5. Run this command in Terminal:

```bash
delex-lite programs/products.json data/abt_buy/table_a.parquet data/abt_buy/table_b.parquet \
    --gold data/abt_buy/gold.parquet
```

**Example output:**

```text
Dataset           abt_buy
|A|               1,081
|B|               1,092
|GT|              1,097
Program           programs/products.json
Rules             2 keep − 0 drop
Funnel            keep[0]  bm25(name,k=10)        10,807
                  keep[1]  bm25(description,k=5)   3,228
                  keep     = keep[0] ∪ keep[1]    12,727
                  result                          12,727
Candidates        12,727
Reduction         98.9219%
Found/GT          1,055/1,097
Recall            96.17%
Precision         8.29%
Time              216 ms
```

Compare the two keep-rule rows with the combined `keep` row. The name rule found 10,807 pairs and the description rule found 3,228. Their union contains 12,727 distinct pairs: the description rule added 1,920 pairs absent from the name rule. With the first rule alone, the recall is 94.99%, and the second rule raises it to 96.17%. A product whose name is written differently in the two stores can still have a similar description.

### Part 7: The same work from Python

Open the included notebook to write a program in Python, run it, inspect saved pairs, and measure their recall. First install Jupyter by running this command in the same Terminal window, with `(.venv)` still shown in the prompt:

```bash
pip install notebook
```

This installs Jupyter into the `.venv` environment. Wait for the command to finish and the Terminal prompt to return. Then open the notebook by running this command in Terminal, from the same `delex_lite` folder:

```bash
jupyter notebook demo.ipynb
```

Jupyter starts in Terminal and opens `demo.ipynb` in your browser. If the browser does not open, copy the address printed by Jupyter into your browser. Keep this Terminal window open while you use the notebook.

In the browser, select the first code cell and press **Shift+Enter** to run it. Wait for its output, then run the remaining cells in order the same way. Read the text above each cell to see what to check in its output.

The notebook builds the Part 5 program in Python, runs it with `D.block`, reads the saved pairs, evaluates them with `D.evaluate`, and compares several values of K. Python code belongs in the notebook's code cells, and Terminal commands belong in Terminal.

The following calls are included in the notebook and shown here for reference:

```python
import pyarrow.parquet as pq
import delex_lite as D

prog = D.program(
    keep=[D.bm25("title", "title").topk(10) & (D.cosine("authors", "authors") >= 0.5)],
    drop=[~D.exact("year", "year")],
)
summary = D.block(prog, "data/dblp_acm/table_a.parquet", "data/dblp_acm/table_b.parquet",
                  gold="data/dblp_acm/gold.parquet")

D.block(prog, "data/dblp_acm/table_a.parquet", "data/dblp_acm/table_b.parquet",
        out="candidates.parquet")
pairs = pq.read_table("candidates.parquet")
report = D.evaluate("candidates.parquet", "data/dblp_acm/gold.parquet")
```

The table below explains each call.

| Call | Returns | What it does |
| --- | --- | --- |
| `D.program(keep=[...], drop=[...])` | a program | Builds the program used in Part 5. `&` joins the predicates of a rule, `>=` sets a threshold, and `~` inverts `exact`. |
| `D.block(prog, table_a, table_b, gold=...)` | a `dict` | Performs the same run as `delex-lite` and prints the same summary. The returned dictionary contains measures such as `n_candidates`, `reduction`, `recall` and `precision`. |
| `D.block(..., out="candidates.parquet")` | a `dict` | Saves the candidate pairs to a file, as `--out` does in Part 3. |
| `pq.read_table("candidates.parquet")` | a pyarrow Table | Reads the saved pairs into Python, with the columns `id1` and `id2`. `pq` is `pyarrow.parquet`, which is installed with `delex_lite`. |
| `D.evaluate("candidates.parquet", gold)` | a `dict` | Measures the recall and precision of the saved pairs against the gold set, without blocking again. |

When you have finished, save the notebook in the browser. To stop Jupyter, return to Terminal, press Ctrl-C, and follow any shutdown prompt. The [technical guide](technical-guide.md) describes `block`, `evaluate` and every predicate.

### Summary

You have now run and compared blocking programs, saved candidate pairs, measured recall and precision, and built a program in Python. The examples show how keep rules, filters and drop rules affect both the number of candidates and the known matches retained.

## Going further

### A workflow of your own

For your own data, copy a program from `programs` and update the column names for your tables. Run it as shown in [Parts 1 and 3](#part-1-running-a-program-on-two-tables). To compare records within one table, use the [deduplication options](technical-guide.md#block). If you have a gold set, compare recall and the number of candidates, as Parts 4 to 6 do. Without one, start with a simple rule, inspect candidates for a sample of records, and compare what additional rules include or exclude. Candidate counts show how much work remains for a matcher. Measuring recall requires known matches identified independently of the blocker. When recall is lower than expected, the usual causes are the gold set, the filters and drop rules, K, the columns and the tokenizer, which [How `delex_lite` works](how-it-works.md#what-determines-the-result) discusses in that order.

### What your files need

Each table needs unique record IDs and the columns referenced by the program. A gold set is needed only to measure recall and precision. See [What your files need](technical-guide.md#what-your-files-need) for the requirements and [File formats](technical-guide.md#file-formats) for supported inputs.

---

- [Technical guide](technical-guide.md). The `delex-lite` command, every predicate and every public function, with its arguments, steps and return value. It also covers the input and output file formats.
- [How `delex_lite` works](how-it-works.md). The concepts behind each step.
- [Troubleshooting](troubleshooting.md). The error messages, what they mean, and what to do.
